# Interactive Python interface

The `ipy` module provides an interactive interface to Guild operations
that is Notebook friendly.

    >>> from guild import ipy
    >>> from guild import config

Create a Guild home for our tests:

    >>> guild_home = config.SetGuildHome(mkdtemp())

Define a simple function to run:

    >>> def hello(msg, n=3):
    ...     import sys
    ...     for i in range(n):
    ...         sys.stdout.write("%s %i!\n" % (msg, i + 1))

## Running an operation

The `ipy` interface runs operations as Python functions. Flags are
provided as key words:

    >>> with guild_home:
    ...     run, result = ipy.run(hello, msg="Hello")
    Hello 1!
    Hello 2!
    Hello 3!

    >>> run
    <guild.run.Run '...'>

    >>> print(result)
    None

Files:

    >>> find(run.path)
    ['.guild/attrs/exit_status',
     '.guild/attrs/flags',
     '.guild/attrs/id',
     '.guild/attrs/initialized',
     '.guild/attrs/started',
     '.guild/attrs/stopped',
     '.guild/opref',
     '.guild/output']

Output:

    >>> cat(run.guild_path("output"))
    Hello 1!
    Hello 2!
    Hello 3!

Run another operation:

    >>> with guild_home:
    ...     ipy.run(hello, msg="Hola", n=2, _label="run2")
    Hola 1!
    Hola 2!
    (<guild.run.Run '...'>, None)

## List runs

List runs:

    >>> with guild_home:
    ...     ipy.runs()
       run  operation  started     status label
    0  ...    hello()      ...  completed run2
    1  ...    hello()      ...  completed

## Runs info

Print latest run info:

    >>> with guild_home:
    ...     ipy.runs().info()
    id: ...
    operation: hello()
    status: completed
    started: ...
    stopped: ...
    label: run2
    run_dir: ...
    flags:
      msg: Hola
      n: 2
    <BLANKLINE>

Info for a specific run:

    >>> with guild_home:
    ...     ipy.runs().iloc[1].info()
    id: ...
    operation: hello()
    status: completed
    started: ...
    stopped: ...
    label:
    run_dir: ...
    flags:
      msg: Hello
      n: 3

Info with output:

    >>> with guild_home:
    ...     ipy.runs().iloc[1].info(output=True)
    id: ...
    ...
    output:
      Hello 1!
      Hello 2!
      Hello 3!

Info with scalars (no scalars for this run, so list is empty):

    >>> with guild_home:
    ...     ipy.runs().iloc[1].info(scalars=True)
    id: ...
    ...
    scalars:

## Flags

Flags can be read as a data frame using the `flags()` function on runs.

    >>> with guild_home:
    ...     runs = ipy.runs()

    >>> flags = runs.flags()

    >>> flags
         msg  n  run
    0   Hola  2  ...
    1  Hello  3  ...

    >>> pprint(flags.to_dict("records"))
    [{'msg': 'Hola', 'n': 2, 'run': '...'},
     {'msg': 'Hello', 'n': 3, 'run': '...'}]

## Delete runs

Delete runs:

    >>> with guild_home:
    ...     ipy.runs().delete()
    ['...', '...']

    >>> with guild_home:
    ...     ipy.runs()
    Empty RunsDataFrame
    Columns: [run, operation, started, status, label]
    Index: []

Deleting an empty list:

    >>> with guild_home:
    ...     ipy.runs().delete()
    []

## Logging scalars

The `ipy` interface does not provide an explicit interface for
logging scalars. This follows the convention used in Guild's script
interface, which is to not provide a Guild-specific
interface. Instead, operations use standard conventions.

In the case of flags, values are pass to the function as function key
words.

Scalars represent the output metrics of a function. These might
typically be returned as a dict. However, it's important to let
functions log values as they run. Return values are therefore
insufficient.

Furthermore, we want to support a seamless migration of functions to
scripts.

With these points in mind, `ipy` supports scalar logging in two ways:

- Printing values to stdout
- Logging scalars in TF event files

### Printing scalars to stdout

Here's a function that prints scalar values to stdout:

    >>> def op1(a, b):
    ...     print("x: %i" % (a + b))
    ...     print("y: %i" % (a - b))
    ...     print("z: %i" % (b - a))

Run the operation:

    >>> with guild_home:
    ...     run, _result = ipy.run(op1, a=1, b=2)
    x: 3
    y: -1
    z: 1

Files:

    >>> find(run.path)
    ['.guild/attrs/exit_status',
     '.guild/attrs/flags',
     '.guild/attrs/id',
     '.guild/attrs/initialized',
     '.guild/attrs/started',
     '.guild/attrs/stopped',
     '.guild/events.out.tfevents...',
     '.guild/opref',
     '.guild/output']

Note that output now contains a `tsevents` file.

Read the scalars:

    >>> with guild_home:
    ...     scalars = ipy.runs().scalars()

    >>> scalars
       avg_val  count  first_step  ...  run  tag  total
    0      3.0      1           0  ...  ...    x    3.0
    1     -1.0      1           0  ...  ...    y   -1.0
    2      1.0      1           0  ...  ...    z    1.0
    <BLANKLINE>
    [3 rows x 14 columns]

    >>> scalars[["run", "last_step", "last_val"]]
       run  last_step  last_val
    0  ...          0       3.0
    1  ...          0      -1.0
    2  ...          0       1.0

### Logging scalars as TFEvents

This function uses tensorboardX to write scalars.

    >>> def op2(a, c):
    ...     import tensorboardX
    ...     writer = tensorboardX.SummaryWriter(".")
    ...     writer.add_scalar("x", a + c, 1)
    ...     writer.add_scalar("x", a + c + 1, 2)
    ...     writer.add_scalar("x", a + c + 2, 3)
    ...     writer.close()

Let's run the function as an operation:

    >>> with guild_home:
    ...     run, _result = ipy.run(op2, a=1.0, c=0.0)

The run files:

    >>> find(run.path)
    ['.guild/attrs/exit_status',
     '.guild/attrs/flags',
     '.guild/attrs/id',
     '.guild/attrs/initialized',
     '.guild/attrs/started',
     '.guild/attrs/stopped',
     '.guild/opref',
     '.guild/output',
     'events.out.tfevents...']

And its scalars:

    >>> with guild_home:
    ...     runs = ipy.runs()
    >>> scalars = runs.loc[runs["run"] == run.id].scalars()
    >>> pprint(scalars.to_dict("records"))
    [{'avg_val': 2.0,
      'count': 3,
      'first_step': 1,
      'first_val': 1.0,
      'last_step': 3,
      'last_val': 3.0,
      'max_step': 3,
      'max_val': 3.0,
      'min_step': 1,
      'min_val': 1.0,
      'prefix': u'',
      'run': u'...',
      'tag': u'x',
      'total': 6.0}]

## Comparing runs

The `compare()` function can be applied to a list of runs to return a
data frame that has both flags and scalars.

    >>> with guild_home:
    ...     runs = ipy.runs()

    >>> compare = runs.compare()

    >>> compare
       run operation started     time  ... step    x    y    z
    0  ...     op2()     ...  0:00:00  ...    3  3.0  NaN  NaN
    1  ...     op1()     ...  0:00:00  ...    0  3.0 -1.0  1.0
    <BLANKLINE>
    [2 rows x 13 columns]

## Grid search

The `run` function will generate trials for values provided in list
args.

Let's clear our runs in preparation for our new trials.

    >>> with guild_home:
    ...     len(runs.delete())
    2

Run `op1` with two values for each of the two arguments. This will
generate four trials.

    >>> with guild_home:
    ...     runs, results = ipy.run(op1, a=[1,2], b=[3,4])
    Running op1 (a=1, b=3):
    x: 4
    y: -2
    z: 2
    Running op1 (a=1, b=4):
    x: 5
    y: -3
    z: 3
    Running op1 (a=2, b=3):
    x: 5
    y: -1
    z: 1
    Running op1 (a=2, b=4):
    x: 6
    y: -2
    z: 2

Generated runs:

    >>> runs
    [<guild.run.Run '...'>,
     <guild.run.Run '...'>,
     <guild.run.Run '...'>,
     <guild.run.Run '...'>]

And op return values:

    >>> results
    [None, None, None, None]

## Random search

Random search uses randomly generated flag values when running
trials. A random search can be performed in various ways:

- Explicitly specify "random" as the `_optimizer` run option
- Specify a slice object for one or more flag values when `_optimizer`
  is not specified

Let's run three trials using "random" optimizer. First, clear existing
runs.

    >>> with guild_home:
    ...     len(ipy.runs().delete())
    4

Run three trials selecting random values for `a` over the range `-10`
to `10` and the value `12` for `b`. Use fixed random seed to let us
assert the generated values.

    >>> with guild_home:
    ...     runs, _ = ipy.run(op1, a=slice(0, 5), b=12,
    ...                       _max_trials=3, _random_seed=1)
    Running op1 (a=3, b=12):
    x: 15
    y: -9
    z: 9
    Running op1 (a=1, b=12):
    x: 13
    y: -11
    z: 11
    Running op1 (a=0, b=12):
    x: 12
    y: -12
    z: 12

    >>> len(runs)
    3

    >>> pprint(runs[0].get("flags"))
    {'a': 3, 'b': 12}

    >>> pprint(runs[1].get("flags"))
    {'a': 1, 'b': 12}

    >>> pprint(runs[2].get("flags"))
    {'a': 0, 'b': 12}

We can alternatively use a range function, which indicates the type of
distribution to sample from.

    >>> with guild_home:
    ...     _ = ipy.run(op1, a=ipy.uniform(0, 5), b=12,
    ...                 _max_trials=3, _random_seed=1)
    Running op1 (a=3, b=12):
    x: 15
    y: -9
    z: 9
    Running op1 (a=1, b=12):
    x: 13
    y: -11
    z: 11
    Running op1 (a=0, b=12):
    x: 12
    y: -12
    z: 12

Finally, we can specify an explicit "random" optimizer:

    >>> with guild_home:
    ...     _ = ipy.run(op1, a=slice(0, 5), b=12,
    ...                 _optimizer="random",
    ...                 _max_trials=3,
    ...                 _random_seed=1)
    Running op1 (a=3, b=12):
    x: 15
    y: -9
    z: 9
    Running op1 (a=1, b=12):
    x: 13
    y: -11
    z: 11
    Running op1 (a=0, b=12):
    x: 12
    y: -12
    z: 12

## Hyperparameter optimization

Guild `ipy` supports other optimizers including "gp", "forest", and
"gbrt".

Let's clear our runs first:

    >>> with guild_home:
    ...     len(ipy.runs().delete())
    9

Run `op1` for three runs using the "gp" optimizer to minimize scalar
`x` where both `a` and `b` are selected from uniform distributions.

    >>> with guild_home:
    ...     ipy.run(op1, a=slice(-10,10), b=slice(-5, 5),
    ...             _optimizer="gp",
    ...             _minimize="x",
    ...             _max_trials=3,
    ...             _random_seed=1)
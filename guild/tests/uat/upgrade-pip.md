# Upgrade pip

Make sure we're running the latest pip.

    >>> quiet("pip install --upgrade pip")

Confirm we're on the expected version.

    >>> run("pip --version")
    pip 20.0.2 from .../site-packages/pip (python ...)
    <exit 0>
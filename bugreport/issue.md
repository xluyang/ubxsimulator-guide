# Issue: `ubxsimulator.read()` raises `TimeoutError` immediately on first read

**Repository:** https://github.com/semuconsulting/pyubxutils

**Title:** `ubxsimulator.read()` raises `TimeoutError` immediately on first read due to `_lastread` init

**Labels (suggested):** `bug`

---

## Describe the bug

`UBXSimulator.read()` raises `TimeoutError` immediately on the **first** read
if the output buffer is not yet populated, instead of waiting for the configured
`timeout` (default 3 seconds).

This manifests as an intermittent startup race in PyGPSClient's built-in
simulator mode (`pygpsclient -U ubxsimulator`): the connection flashes
"connected" then immediately drops to "Not connected" (inactivity timeout).

## To Reproduce

```python
from pyubxutils.ubxsimulator import UBXSimulator

sim = UBXSimulator()  # do NOT call start() -> buffer stays empty
try:
    sim.read(1)
except TimeoutError:
    print("raised immediately, not after timeout")
```

**Expected:** block for `self._timeout` (3s) before raising.
**Actual:** raises instantly.

## Root cause

In `ubxsimulator.py` (`__init__`):

```python
self._lastread = datetime.fromordinal(1)   # year 1
```

so the check in `read()`:

```python
while len(self._buffer) < num:
    sleep(self._interval / 20000)
    if datetime.now() > self._lastread + timedelta(seconds=self._timeout):
        raise TimeoutError
```

is always `True` on the first read with an empty buffer, because
`datetime.now()` is always later than year 1 + 3 seconds.

## Expected behavior

The first read should wait up to `timeout` seconds for data to arrive.

## Proposed fix

```python
self._lastread = datetime.now()
```

## Environment

- pyubxutils 1.0.6
- Python 3.14.4
- Linux

## Additional context

- PyGPSClient version 1.7.7
- The built-in simulator is triggered by `pygpsclient -U ubxsimulator`.
- A one-line patch is attached: `fix-ubxsimulator-timeout.patch`.

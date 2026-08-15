# Raspberry Pi benchmark — one command

## Files in this folder

| File | What it is |
|---|---|
| `pi_benchmark.py` | the benchmark script |
| `requirements.txt` | Python dependencies |
| `tflite_runtime-2.11.0-cp37-...whl` | the interpreter wheel for Python 3.7 (from this project's own repo) |
| `edge_int8.tflite` | the uint8 edge model, re-exported TILE-free so runtime 2.11 can load it (ODS 0.741 vs 0.740 for the original) |
| `dehazer_int8.tflite` | the 373 KB uint8 dehazer, newly exported so both stages can be timed |

Copy them all to the Pi, into the same folder.

## Setup on the Pi

### If you are on 32-bit Raspberry Pi OS Buster (glibc 2.28) — most likely

Use **Python 3.7** with the bundled wheel. Do not chase a newer Python: `tflite-runtime`
ships exactly one cp311 armv7l wheel (2.14.0) and it requires glibc 2.34, which Buster
does not have. Every build old enough for Buster's glibc stops at cp310.

```bash
sudo apt update
sudo apt install -y python3-opencv python3-numpy python3-pip
pip3 install tflite_runtime-2.11.0-cp37-cp37m-manylinux2014_armv7l.whl
python3 -c "from tflite_runtime.interpreter import Interpreter; print('ok')"
```

The models in this kit were re-exported to avoid the `TILE v3` op, which runtime 2.11
cannot register. They are numerically equivalent to the originals (ODS 0.741 vs 0.740).

### If you want a newer runtime on Buster

Python **3.10**, not 3.11:

```bash
pip3 install tflite-runtime==2.13.0
```

### If you reflash to 64-bit Bookworm (glibc 2.36)

```bash
pip3 install tflite-runtime      # picks up 2.14.0
```

Also the fastest option — 64-bit is meaningfully quicker than armv7l on the same board.

## Before you run

- Plug into mains power, not a battery.
- Close other apps and let the Pi sit idle ~2 minutes so it starts cool.
- Don't run it over VNC/screen-share; that skews CPU numbers.

## The command

```bash
python3 pi_benchmark.py \
    --edge   edge_int8.tflite \
    --dehaze dehazer_int8.tflite \
    --tau    0.5854 \
    --iters  100 \
    --threads 4
```

Takes about 4-6 minutes (most of it is the 2-minute thermal soak).

Then run it once more single-threaded, so we can report both:

```bash
python3 pi_benchmark.py --edge edge_int8.tflite --dehaze dehazer_int8.tflite \
    --tau 0.5854 --iters 100 --threads 1 --soak-seconds 0
```

## What to send back

Everything between `PASTE EVERYTHING BELOW` and `END`, for both runs. That's a JSON block containing:

- `edge_only` / `dehaze_only` / `dehaze_plus_edge` — median, mean, p95 latency and FPS
- `gated` — the same at three haze levels, showing where the gate skips the dehazer
- `gate_statistic` — the cost of the gate itself, which the paper claims is negligible
- `soak` — sustained FPS and temperature drift, plus whether the Pi thermally throttled

## If something breaks

- `ValueError: Could not open` → wrong path to a .tflite file
- `Segmentation fault` → low memory; reboot and rerun, closing everything else
- `vcgencmd: not found` → harmless, temperature just reports as `null`

`--tau 0.5854` is the gate threshold calibrated in the paper. Pass it exactly, otherwise the script falls back to a placeholder and the gated numbers won't match the reported policy.

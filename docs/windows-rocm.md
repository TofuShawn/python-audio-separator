# Windows + AMD ROCm

This fork exists so `audio-separator` can run on Windows with AMD's ROCm torch
build, which is the only combination on that platform where **the torch models**
(VR, Demucs, MDXC, RoFormer) get GPU inference. DirectML remains the ONNX-only
path it already is; see the main README's DirectML section for its limits
(Demucs unsupported, RoFormer allocator exhaustion) — those are
`torch-directml` limitations and do not apply here.

## Install

Python 3.12 is required: the only Windows ROCm wheel is `cp312`.

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1

# AMD's Windows ROCm index is a flat file listing, not a PEP 503 index --
# --find-links plus exact +rocm pins is the form that works.
pip install --find-links https://repo.radeon.com/rocm/windows/rocm-rel-7.2.1/ `
    "torch==2.9.1+rocm7.2.1" "torchvision==0.24.1+rocm7.2.1" "torchaudio==2.9.1+rocm7.2.1"

# Install this package without letting pip replace that torch.
pip install --no-deps .
pip install librosa soundfile samplerate resampy numpy scipy einops julius `
    ml_collections pyyaml six pydub tqdm requests packaging beartype `
    rotary-embedding-torch diffq onnx onnxruntime-directml
```

Check the device before separating:

```powershell
python -c "import torch; print(torch.__version__, torch.version.hip, torch.cuda.get_device_name(0))"
```

`torch.version.hip` must be non-empty. If it is `None`, pip gave you a CPU or
CUDA wheel and this page does not apply.

## Measured (Windows 11, RX 7900 XTX, torch 2.9.1+rocm7.2.1, 2026-11-09)

| Check | Value |
| --- | --- |
| `1_HP-UVR` warm inference, 30 s clip | 1.88 s (second run; first run 4.20 s, MIOpen cold start) |
| VR vocals vs a UVR 5.6 build | 47.6 dB SNR, max deviation 6.1e-05 |
| VR instrumental vs the same build | 17.7 dB raw, **71.2 dB after one 1.149x gain** (same lag, correlation 1.0) |

The instrumental difference is one scalar, not a separation difference: the two
waveforms are identical once that gain is applied. The scalar is the
normalisation stage, which is not a peak match — setting
`normalization_threshold=1.0` does **not** close it (measured: 29.6 dB), because
the 0.9 -> 1.0 change scales by 1.111 while the actual ratio is 1.149. Matching
UVR's output exactly means reproducing its own normalisation in the caller, not
turning this knob.

## DirectML on the same machine

`--use_directml` still works for MDX (ONNX) and is what the main README
documents. The hybrid that matches this fork's measurements: ONNX models on
DirectML, torch models on ROCm. There is no single execution provider on
Windows AMD that covers both.

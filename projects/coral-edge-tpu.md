# Google Coral Edge TPU on Python 3.12 — Ubuntu 24.04

> Status: ✅ Working  
> Hardware: Google Coral USB Accelerator  
> Host: Lenovo ThinkCentre M900 Tiny (i5-6500T, 8GB RAM)  
> OS: Ubuntu 24.04 LTS, kernel 6.8.0-101  
> Python: 3.12  

---

## The Problem

Google's official documentation tells you to install `pycoral`. This hasn't worked since Python 3.10 — there are no official wheels for 3.11 or 3.12.

Worse: `pip install pycoral` silently installs a **completely different package** (a reef mapping tool that happens to share the name). No error. No warning. Just the wrong thing.

---

## What Actually Works

### 1. Install the Edge TPU runtime

```bash
echo "deb https://packages.cloud.google.com/apt coral-edgetpu-stable main" | \
  sudo tee /etc/apt/sources.list.d/coral-edgetpu.list
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
sudo apt update
sudo apt install libedgetpu1-std
```

Add your user to the `plugdev` group (for USB access):

```bash
sudo usermod -aG plugdev $USER
# Log out and back in
```

### 2. Build the gasket/apex DKMS driver for kernel 6.8

The upstream driver doesn't build cleanly on kernel 6.8. Two patches required:

```bash
git clone https://github.com/google/gasket-driver
cd gasket-driver

# Patch 1: fix deprecated function signature (kernel 6.8 API change)
# Patch 2: fix Makefile target for DKMS
# (see patches/ directory in this repo — coming soon)

sudo dkms add .
sudo dkms build gasket/1.0
sudo dkms install gasket/1.0
```

**Note:** `/dev/apex0` is NOT created for USB accelerators — that's only for PCIe versions. USB communicates directly through `libedgetpu.so.1`. If you're looking for `/dev/apex0` to confirm it's working, you're looking in the wrong place.

### 3. Install `ai-edge-litert` (not pycoral)

```bash
pip install ai-edge-litert
```

`ai-edge-litert` is Google's official 2024 successor to `tflite-runtime`. It works on Python 3.12.

### 4. Run inference with Edge TPU delegate

```python
from ai_edge_litert import interpreter as lit

# Load a quantized TFLite model (must be compiled for Edge TPU)
model_path = "your_model_quant_edgetpu.tflite"

# Load the Edge TPU delegate
delegate = lit.load_delegate('libedgetpu.so.1')

# Create interpreter with delegate
interpreter = lit.Interpreter(
    model_path=model_path,
    experimental_delegates=[delegate]
)
interpreter.allocate_tensors()

# Run inference normally from here
```

---

## Verification

```python
from ai_edge_litert import interpreter as lit

try:
    delegate = lit.load_delegate('libedgetpu.so.1')
    print("✅ Edge TPU delegate loaded successfully")
except Exception as e:
    print(f"⚠️  CPU fallback only: {e}")
```

If the USB accelerator is connected and `libedgetpu1-std` is installed, you'll see the success message. If not connected, it falls back to CPU — no crash.

---

## What Doesn't Work (and Why)

| Approach | Why it fails |
|---|---|
| `pip install pycoral` | Installs wrong package (reef mapping tool) |
| Official Google pycoral wheels | Last wheel: Python 3.9. No 3.10, 3.11, 3.12 |
| `pip install tflite-runtime` | Deprecated, unmaintained |
| Looking for `/dev/apex0` on USB | Only exists for PCIe M.2 version |

---

## Notes

- Models must be quantized (INT8) and compiled for Edge TPU to use hardware acceleration
- CPU fallback is automatic — non-compiled models still run, just slower
- The `ai-edge-litert` API is nearly identical to the old `tflite-runtime` API
- Google has not updated their official Coral documentation to reflect `ai-edge-litert` (as of early 2026)

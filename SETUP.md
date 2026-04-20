# Setup Guide

This guide covers installation on **Ubuntu 24.04 with an NVIDIA GPU and a Wayland desktop session**. It documents fixes for GPU acceleration, Wayland text injection, and non-US keyboard layouts applied on top of the base install script.

## System Requirements

- Ubuntu 22.04 or 24.04
- NVIDIA GPU with CUDA support (optional — CPU fallback available)
- Wayland or X11 desktop session
- Python 3.10+

## Dependencies

Install all system packages before running `install.sh`:

```bash
sudo apt install \
  python3.12-venv \
  portaudio19-dev \
  ffmpeg \
  ydotool \
  xdotool \
  xclip \
  wl-clipboard \
  wtype
```

## Installation

```bash
git clone https://github.com/Mettedws/linux-whisper ~/tools/linux-whisper
cd ~/tools/linux-whisper
bash install.sh
```

The installer:
- Creates a Python virtual environment and installs Python dependencies
- Adds your user to the `input` group for hotkey detection
- Creates a udev rule for `/dev/uinput`
- Registers `whisper-dictate` as a systemd user service

**Log out and back in** after the installer completes for group permissions to take effect.

## GPU Setup (NVIDIA)

The default config uses `medium.en` on CUDA with `float16` precision. On first run, models are downloaded automatically — pre-download them to avoid a startup timeout:

```bash
cd ~/tools/linux-whisper

# Pre-download the Whisper model
venv/bin/python -c "
from faster_whisper import WhisperModel
WhisperModel('medium.en', device='cuda', compute_type='float16')
"

# Pre-trust the Silero VAD model (required — interactive prompt fails in a service)
venv/bin/python -c "
import torch
torch.hub.load('snakers4/silero-vad', 'silero_vad', trust_repo=True, verbose=False)
print('Done.')
"
```

If `libcublas.so.12` is not found (common when CUDA is bundled with Ollama rather than installed system-wide), add it to the service environment. Edit `~/.config/systemd/user/whisper-dictate.service` and add:

```ini
Environment="LD_LIBRARY_PATH=/usr/local/lib/ollama/cuda_v12"
```

Then reload:

```bash
systemctl --user daemon-reload
systemctl --user restart whisper-dictate
```

To use CPU instead, set `"device": "cpu"` and `"compute_type": "int8"` in `config.json`.

## Wayland Text Injection

On Wayland, text injection uses `xdotool type` targeting the XWayland display. The service needs `DISPLAY` and `XAUTHORITY` set. The script resolves `XAUTHORITY` automatically from `/run/user/<uid>/.mutter-Xwaylandauth.*` at startup.

Add `DISPLAY=:0` to the service by editing `~/.config/systemd/user/whisper-dictate.service`:

```ini
Environment="DISPLAY=:0"
```

### Non-US Keyboard Layouts

XWayland defaults to a `us` keyboard layout regardless of the system layout. This causes incorrect characters when using `xdotool type` with AZERTY or other non-QWERTY keyboards.

The script automatically calls `setxkbmap be` at startup to set the XWayland layout to Belgian AZERTY. To change this for a different layout, edit `_set_xwayland_keyboard_layout()` in `whisper_dictate.py`:

```python
# Belgian AZERTY
subprocess.run(["setxkbmap", "be"], ...)

# French AZERTY
subprocess.run(["setxkbmap", "fr"], ...)

# US QWERTY (default, no change needed)
subprocess.run(["setxkbmap", "us"], ...)
```

## Service Management

The service starts automatically on login. Manual controls:

```bash
# Start / stop / restart
systemctl --user start whisper-dictate
systemctl --user stop whisper-dictate
systemctl --user restart whisper-dictate

# View live logs
journalctl --user -u whisper-dictate -f
```

## Configuration

Edit `config.json` to customise behaviour:

| Key | Default | Description |
|---|---|---|
| `hotkey` | `"alt"` | Trigger key. Examples: `"alt"`, `"<ctrl>+space"`, `"<super>+d"` |
| `model` | `"medium.en"` | Whisper model: `tiny.en`, `base.en`, `small.en`, `medium.en`, `large-v3` |
| `language` | `"en"` | Transcription language |
| `device` | `"cuda"` | Inference device: `"cuda"` or `"cpu"` |
| `compute_type` | `"float16"` | Precision: `"float16"` (GPU) or `"int8"` (CPU) |
| `input_method` | `"clipboard"` | Text injection method: `"clipboard"`, `"xdotool"`, `"ydotool"`, `"wtype"`, `"auto"` |
| `sound_feedback` | `true` | Play sounds on recording start/stop |
| `continuous_mode` | `false` | Keep recording after each transcription |

Restart the service after editing:

```bash
systemctl --user restart whisper-dictate
```

## Usage

Press the configured hotkey (default: **Alt**) to toggle recording. Speak, pause, and the transcribed text is typed at the cursor position in whatever application is focused.

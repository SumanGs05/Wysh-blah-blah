# WYSH Beamformer

WYSH is a small microphone array project. An FPGA (Tang Nano 9K) samples a ring of MEMS microphones and streams the audio over UART to a Raspberry Pi. The Pi runs a beamformer that combines the channels into one signal, steered toward a chosen direction, and plays it through the headphone jack.

This repository contains the Raspberry Pi side of the system: the beamformer, a receiver used for bring-up, and a simple listening tool.

## Contents

| File | Purpose |
|------|---------|
| `wysh_beamformer.py` | Main program. Reads the FPGA stream, applies MVDR or delay-and-sum beamforming, and plays the result. Includes an interactive console for steering. |
| `wysh_rx.py` | Receiver and diagnostics tool. Checks frame sync, frame rate, and dropped or corrupt samples, or prints per-channel RMS levels. |
| `wysh_listen.py` | Plays one or more raw channels through `aplay` with a gain control. Useful for checking each microphone by ear. |

The FPGA firmware is not part of this repository.

## Features

- Delay-and-sum and MVDR (Capon) beamforming, switchable at runtime.
- Real-time steering from -180 to +180 degrees.
- Automatic sweep mode that finds the loudest direction.
- Frame resynchronization on the serial stream, with counters for dropped and resynced data.
- Reader and DSP run in separate processes so serial reads are not stalled by signal processing.
- Live statistics: frame rate, DSP load, worst-case call time, drops, and ALSA underruns.

## Hardware

- Raspberry Pi with a 3.5 mm headphone output. The default `ALSA_DEVICE` is `plughw:Headphones`, so adjust it to match your `aplay -l` output.
- Tang Nano 9K FPGA board running the WYSH stream firmware.
- Six microphones on a ring (channels 0 to 5), plus an optional centre microphone (channel 7).
- UART connection from the FPGA to the Pi. The default port is `/dev/serial0`, which is GPIO 15 (pin 10) on the Pi.

### Stream format

The FPGA sends one 17-byte frame per audio sample:

```
0xA5 | ch0 (hi lo) | ch1 (hi lo) | ... | ch7 (hi lo)
```

- The first byte is a sync marker, `0xA5`.
- Each channel is a signed 16-bit big-endian sample.
- Channels 0 to 5 are the ring microphones. Channel 6 is an empty slot. Channel 7 is the centre microphone.

The sample rate is fixed by the FPGA clock divider at about 16226 Hz (27 MHz / 26 / 64).

## Requirements

- Python 3.8 or newer
- `numpy`
- `pyserial`
- `aplay` (from `alsa-utils`), used for audio output

Install the Python dependencies:

```bash
pip3 install numpy pyserial
```

On recent Raspberry Pi OS releases, you may need `--break-system-packages` or a virtual environment.

## Quick start

1. Connect the FPGA UART to the Pi and confirm the port exists:

   ```bash
   ls -l /dev/serial0
   ```

2. Check that data is arriving and the frames are aligned:

   ```bash
   python3 wysh_rx.py /dev/serial0
   ```

   You should see a frame rate near 16 kframes per second and few or no gaps or bad payloads.

3. Start the beamformer:

   ```bash
   python3 wysh_beamformer.py
   ```

   To use a different port:

   ```bash
   python3 wysh_beamformer.py /dev/ttyUSB0
   ```

4. Enter a steering angle at the `beam>` prompt, or run `s` to sweep.

## Usage

### Beamformer console

| Command | Action |
|---------|--------|
| `<number>` | Steer the beam to an angle in degrees. `0` is straight ahead, `+90` is right, `-90` is left, and `+/-180` is behind. |
| `s` | Sweep from -90 to +90 degrees in 15 degree steps, print the output level at each angle, and steer to the loudest one. Keep the source still during the sweep. |
| `m` | Toggle between MVDR and delay-and-sum. |
| `g <x>` | Set the output gain, for example `g 5`. |
| `q` | Quit. |

While running, the program prints a statistics line every five seconds:

```
[stats] 16226 frames/s | dsp  35.2% | max call 2.1 ms (budget 8.0) | dropped 0 | resync 0 | underrun 0 | rms 0.0123 | az +0 | mvdr | gain 7
```

Keep the DSP load well below 100 percent and the underrun count at zero. If underruns appear, raise `ALSA_BUFFER_US` or switch to delay-and-sum.

### Receiver and diagnostics

```bash
python3 wysh_rx.py /dev/serial0            # frame rate, gaps, and bad payloads
python3 wysh_rx.py /dev/serial0 --test     # test mode, check the counter pattern
```

In the default mode, `wysh_rx.py` prints the RMS level of each channel once per second. Use this to find dead or noisy microphones.

### Listening to a channel

```bash
python3 wysh_listen.py /dev/serial0 --ch 0 --gain 8
python3 wysh_listen.py /dev/serial0 --ch 0,1,2,3,4,5 --gain 8
```

The output is raw 16-bit PCM written to standard output. It is intended to be piped into `aplay`. The script prints the selected channels to standard error when it starts.

## Configuration

All settings for the beamformer are at the top of `wysh_beamformer.py`.

| Setting | Default | Description |
|---------|---------|-------------|
| `SERIAL_PORT` | `/dev/serial0` | UART device. |
| `SERIAL_BAUD` | `3000000` | UART baud rate. Must match the FPGA. |
| `ALSA_DEVICE` | `plughw:Headphones` | Output device. Run `aplay -l` to list devices. |
| `ALSA_BUFFER_US` | `400000` | Playback buffer in microseconds. |
| `MIC_CHANNELS` | `[0, 1, 2, 3, 4, 5]` | FPGA channels to use. |
| `RING_RADIUS_M` | `0.046` | Radius of the microphone ring in metres. |
| `RING_ANGLES_DEG` | `[0, 60, 120, 180, 240, 300]` | Angle of each channel around the ring, measured from the top of the board, clockwise. |
| `AZIMUTH_SIGN` | `1.0` | Set to `-1.0` if the beam appears mirrored (left and right swapped). |
| `NFFT` | `256` | FFT size. Gives 63 Hz bins and a 16 ms window. |
| `FREQ_MIN`, `FREQ_MAX` | `300`, `4000` | Frequency band where MVDR is applied. Outside it, delay-and-sum is used. |
| `HIGH_PASS_HZ` | `100` | Bins below this frequency are removed. |
| `ALPHA` | `0.98` | Covariance smoothing factor. |
| `LOADING` | `0.05` | Diagonal loading for MVDR. Higher values are more robust and closer to delay-and-sum. |
| `WEIGHT_EVERY` | `4` | Recompute MVDR weights every N hops. |
| `OUTPUT_GAIN` | `7.0` | Default output gain. |

### Calibrating the array

The geometry values are estimates until they are checked on the board.

1. Measure the ring radius, from the centre of the board to a microphone, or as half the distance between two opposite microphones. Update `RING_RADIUS_M`.
2. Confirm that the channels run around the ring in the order given by `RING_ANGLES_DEG`.
3. Play a sound from a known direction and run the `s` sweep. If the peak is on the wrong side, set `AZIMUTH_SIGN = -1.0`.

To add the centre microphone, use:

```python
MIC_CHANNELS = [0, 1, 2, 3, 4, 5, 7]
MIC_POSITIONS = np.vstack([MIC_POSITIONS, [0.0, 0.0]])
```

## How it works

1. **Reader process.** Reads bytes from the UART, finds frame alignment on the sync byte, and selects the configured channels. Chunks are placed on a queue. If the queue is full, the chunk is counted as dropped.
2. **DSP thread.** Splits the stream into overlapping 256-sample windows with a square-root Hann window, which gives perfect reconstruction at 50 percent overlap. For each window it computes a spatial covariance matrix for each frequency bin and forms the beamformer weights.
3. **Playback.** The output is clipped, scaled, and written as 16-bit PCM to `aplay`.

With the ring geometry, the spacing between adjacent microphones is about 46 mm. The program prints the spatial aliasing limit at startup, which is about 3.7 kHz for this spacing.

## Troubleshooting

| Symptom | Likely cause |
|---------|--------------|
| `Cannot open /dev/serial0` | Serial port is disabled or in use. Enable the UART in `raspi-config` and disable the login shell on it. |
| `no data received` in `wysh_rx.py` | Wiring, FPGA firmware, or baud rate. Check the connection and confirm the FPGA is flashed. |
| Many `resync` counts | Noise on the UART line, or a baud rate mismatch. |
| `dropped` count rising | The DSP cannot keep up. Lower the update rate of MVDR with `WEIGHT_EVERY`, or switch to delay-and-sum with `m`. |
| `underrun` count rising | Increase `ALSA_BUFFER_US`, or reduce the DSP load. |
| `aplay closed` | Wrong `ALSA_DEVICE`. Check the output of `aplay -l`. |
| Beam points the wrong way | Set `AZIMUTH_SIGN = -1.0`, then check the ring angles. |

The reader process tries to set its priority with `os.nice(-10)`. This needs root. If it is not run as root, the program still works without the higher priority.

## Contributing

Issues and pull requests are welcome. For changes to the DSP, please include a short description of the test setup and, where possible, a recording or measurement that shows the effect.

## License

No license has been specified yet. Until one is added, all rights are reserved by the author.

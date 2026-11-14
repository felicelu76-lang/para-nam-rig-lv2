# ParaNAM Rig Strip LV2 (`para_nam_rig.lv2`)

**An all-in-one, low-latency neural guitar channel strip LV2 plugin engineered for PiPedal 2.0 on Raspberry Pi 5 (ARM64 / Cortex-A76).**

---

## Why This Project Exists

While neural amp capture engines like **Neural Amp Modeler (NAM)** deliver remarkable realism, running them inside a headless live guitar rig (such as **PiPedal 2.0** on a Raspberry Pi 5 with a Blokas Pisound audio interface) typically presents three practical limitations:

1. **No Native Parametric NAM Support on Headless LV2 Hosts:** Existing LV2 wrappers only run static `.nam` captures (A1/A2). Parametric captures generated with `ParaNAM-Trainer` (`HyperWaveNet`, `HyperA2`, `ParametricSlimmableContainer`, and `.pnam` files with non-linear PCHIP volume curves) are desktop-oriented (VST3/AU/Standalone) and rely on dynamic GUI widgets that headless web UIs cannot expose.
2. **Digital Aliasing & Static "Feel" Without Pre/Post Conditioning:** Feeding raw, unconditioned guitar transients directly into a neural network often produces harsh ultrasonic pick aliasing ("broken glass" artifacts) and lacks the natural transformer compression ("power-amp sag") and speaker-cone smoothing found in commercial amp-modeling suites.
3. **Pedalboard Clutter & Buffer Overhead:** Replicating a proper studio-grade pre-EQ, gate, neural capture, sag compressor, and post-cab de-harshing chain requires stacking 4 to 5 separate LV2 blocks per preset—consuming extra JACK audio buffer copies and making preset tweaking tedious.

**ParaNAM Rig Strip** solves all three problems by fusing the official **`NamModelEngine`** (from [DLC86/NamParametricPlugin](https://github.com/DLC86/NamParametricPlugin) and `NeuralAmpModelerCoreParametric`) with a curated, zero-latency analog-inspired pre/post conditioning chain inside a single native LV2 bundle compiled specifically for ARM64 NEON/Cortex-A76 hardware.

---

## Signal Flow & Architecture

Audio is processed in 32-bit/64-bit floating-point precision through four integrated stages:

### 1. Pre-Amp Conditioning & Input Stage (`1. Pre & Gain`)
Designed to tame digital aliasing and tighten low-end response *before* the signal hits the non-linear neural network (inspired by a transparent Bluesbreaker / Morning Glory style front-end):
* **Pre ON:** Master bypass switch for the pre-filtering stage.
  * **High-Pass Filter (85 Hz, 12 dB/oct Butterworth):** Removes sub-bass rumble that otherwise chokes neural distortion on fast palm-muted riffs.
  * **Low-Pass Filter (11.5 kHz, 12 dB/oct Butterworth):** Strips ultrasonic pick attack transients before non-linear waveshaping, preventing high-frequency intermodulation and aliasing artifacts.
* **Mid Boost (`-6.0 dB` to `+6.0 dB`, `0 dB` at noon, default `+1.5 dB`):** Wide-Q (`0.7`) peaking filter centered at **1.1 kHz** to add pick elasticity and midrange focus without artificially inflating the model's gain staging.
* **Input Gain & Noise Gate:** Input trim (`-15 dB` to `+15 dB`, default `-3.0 dB` to hit the neural network's dynamic sweet spot with high-output humbuckers) paired with a smooth, envelope-following noise gate (`-90 dB` to `-30 dB`, default `-72 dB`).
* **Output Gain:** Master post-model level trim (`-15 dB` to `+15 dB`, default `+2.5 dB`).

### 2. Calibration & Quality (`Calibration`)
Built directly on `NamModelEngine`'s native calibration math to ensure consistent gain staging and volume parity across different captures:
* **Input Calibration (`Off` / `Calibrated`) & `Value (dBU)`:** Compensates for the difference between your audio interface's input headroom (default **`-6.00 dBU`**) and the `input_level_dbu` metadata stored inside the capture.
* **Output Calibration (`Raw` / `Normalized` / `Calibrated` / `Auto`):** Automatically levels different captures to a unified `-18 dBFS` target loudness (when `Normalized` is selected) or preserves exact hardware insertion gain (`Calibrated`).
* **Quality (`Full` / `Lite`):** Allows real-time switching between full-resolution and lightweight branches on `ParametricSlimmableContainer` / HyperA2 models.

### 3. Universal Neural Engine (`2. Para Knobs` & `3. Switches`)
* **Universal Format Support:** Seamlessly loads **static NAM (A1 & A2)** captures as well as **parametric `.nam` and `.pnam`** models (`HyperWaveNet`, `ConcatWaveNet`, `ConcatLSTM`, `ParametricSlimmableContainer`), including automatic extraction of zipped `.pnam` archives and evaluation of `linear_volume` PCHIP output curves.
* **6 Continuous Knobs (`Knob 1` – `Knob 6`) + 2 Switches (`Switch 1` – `Switch 2`):** Automatically mapped to the loaded model's parameter vector (`IParametricControl`), complete with **real-time green status LEDs** next to each control so you immediately see which knobs and switches are active on the current profile (all LEDs turn off automatically when a static NAM capture is loaded).

### 4. Power-Amp Sag & Cab Dynamic (`4. Sag & Cab Dynamic`)
Replicates the physical interaction between a tube output transformer and a mic'd guitar cabinet:
* **Power-Amp Sag (`Sag ON` + `Sag` `0–10`):** A soft-knee (`6 dB`), low-ratio (`1.8:1`) envelope compressor placed immediately after the neural amp stage. With a **22 ms attack** (letting the initial pick transient pass untouched) and **140 ms release** (matching power-supply filter capacitor recovery), it adds natural elastic "give" under heavy downstrokes.
* **Cab Dynamic (`OFF` / `ON` / `AUTO Meta`):** Post-amp cabinet smoothing and resonance stage. When set to `AUTO Meta`, the plugin inspects the capture's JSON metadata (`has_cab`, `cab`, `gear_type`) to engage automatically on full-rig captures (or can be forced `ON`/`OFF` manually or via MIDI):
  * **Sub-Low Cut:** 12 dB/oct High-Pass at **90 Hz** to keep guitar tracks out of the bass guitar and kick drum space.
  * **Depth (`0–10`, default `6.5` = `+1.8 dB`):** Peaking resonance at **118 Hz** ($Q = 1.2$) restoring the physical chest thump of a 4x12 closed-back cabinet.
  * **Presence (`0–10`, default `3.5` = `-1.5 dB`):** Surgical notch/bell at **3.8 kHz** ($Q = 2.5$) to tame digital SM57 harshness, followed by a 12 dB/oct Low-Pass at **7.2 kHz** to roll off unnatural ultrasonic fizz.

---

## Credits & Acknowledgments
* **[The Tone Scientist](https://www.youtube.com/@TheToneScientist)** for pioneering parametric neural captures (`ParaNAM-Trainer`), the `ParametricSlimmableContainer` / `linear_volume` extensions, and community reference models.
* **[DLC86](https://github.com/DLC86/NamParametricPlugin)** & **[Phillip Self](https://github.com/phillipmself/NamParametricPlugin)** for `NamModelEngine`, `OutputGainCurve`, and `NeuralAmpModelerCoreParametric`.
* **[Steven Atkinson](https://github.com/sdatkinson/neural-amp-modeler)** for Neural Amp Modeler.
* **[Robin Davies](https://github.com/rerdavies/pipedal)** for PiPedal 2.0 and its LV2 UI extensions.

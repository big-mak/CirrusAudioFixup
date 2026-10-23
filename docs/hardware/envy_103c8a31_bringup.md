# HP ENVY x360 15-ey0xxx (103C:8A31) bring-up status

State of the CSC3551 / CS35L41 I2C bring-up on HP subsystem `103C:8A31`
(ENVY x360 15-ey0xxx, AMD Ryzen 7 5825U, Realtek ALC245, macOS 13.7.2).

This file records what is verified, what is not, and where the remaining
boundary is. It exists because the same boundary was reached upstream without
being written down.

## Hardware identity

| Item | Value | Evidence |
| --- | --- | --- |
| Subsystem | `103C:8A31` | `ioreg -n HDEF`: `subsystem-id = <318a0000>` |
| Codec | Realtek ALC245, `0x10EC0245` | `VendorProductID = 283902533` |
| HDA controller | `1022:15E3` AMD Zen | `HDEF vendor-id <22100000> device-id <e3150000>` |
| Amplifiers | 2x CS35L41, `ID 0x00035A40`, `REV 0xB2` | driver registers `Cirrus_I2C_{0,1}_DeviceID` |
| Amp bus | I2C, addresses `0x40` / `0x41` | ACPI `_CRS` SerialBus descriptors |
| Amp controller | `I2CB`, `AMDI0010`, `_UID` 1, MMIO `0xFEDC3000` | `ioreg -p IOACPIPlane` |
| Reset | shared, AMDI0030 GPIO pin 30, active low | Linux `amdgpu` pinctrl state; `Cirrus_Reset_Policy = AUTHORIZED` |
| Speaker ID | GPIO pin 90 (`0x5A`), board resource value 1 | `Cirrus_Speaker_ID_Source = BOARD_RESOURCE` |
| Amp interrupts | GPIO pin 31 (`0x1F`), shared, active low | ACPI `_CRS` GpioIo descriptors |
| Boost | internal (Linux `BST: 0`), 1000 nH / 4100 mA / 24 uF | Linux dmesg; `kHPEnvy103C8A31DeviceConfig` |

ACPI `SPKR` under `\_SB.I2CB` declares `_HID CSC3551`, `_SUB 103C8A31`,
`_UID 1` and a `_CRS` with two I2C SerialBus descriptors (0x40, 0x41) plus four
GpioIo descriptors (pins 0x1E, 0x5A, 0x1F, 0x1F). It declares no `_DSD`; the
driver injects `cirrus,*` metadata from its verified board table.

## Linux reference state

Booted from the same machine, the in-tree drivers bring the amps up fully:

```
cs35l41-hda i2c-CSC3551:00-cs35l41-hda.0: Adding DSD properties for 103C8A31
cs35l41-hda i2c-CSC3551:00-cs35l41-hda.0: Cirrus Logic CS35L41 (35a40), Revision: B2
cs35l41-hda i2c-CSC3551:00-cs35l41-hda.1: Reset line busy, assuming shared reset
cs35l41-hda i2c-CSC3551:00-cs35l41-hda.0: DSP1: Firmware: 400a4 vendor: 0x2 v0.43.1, 2 algorithms
cs35l41-hda i2c-CSC3551:00-cs35l41-hda.0: Calibration applied: R0=10457
cs35l41-hda i2c-CSC3551:00-cs35l41-hda.0: Firmware Loaded - Type: spk-prot, Gain: 17
cs35l41-hda i2c-CSC3551:00-cs35l41-hda.0: CS35L41 Bound - SSID: 103C8A31, BST: 0, VSPK: 0, CH: L, FW EN: 1, SPKID: 1
snd_hda_codec_alc269 hdaudioC1D0: bound i2c-CSC3551:00-cs35l41-hda.0 (ops cs35l41_hda_comp_ops)
```

Both amps answer at `0x40` and `0x41` (`i2cdetect` reports `UU`), the firmware
and per-channel tuning are the right ones, and calibration is applied.

## macOS state reached

With the branch installed and the I2C path attaching, the amplifier side matches
Linux except for the clock:

```
Cirrus_Bus                  = I2C
Cirrus_Amp_Discovery        = DETECTED_CONFIGURED_I2C
Cirrus_I2C_{0,1}_DeviceID   = 219712 (0x00035A40)
Cirrus_Reset_Policy         = AUTHORIZED
Cirrus_Calibration_R0_i2c0  = 10457
Cirrus_Calibration_R0_i2c1  = 11257
Cirrus_BIN_Version_{0,1}    = 16788225
Cirrus_DSP_BOOT_VERDICT     = HEALTHY (both)
Cirrus_DSP_HALO_STATE       = 2 (RUN)
asp validation check        = pass
Cirrus_Driver_Verdict       = FAULT_LATCHED
Cirrus_Playback_Verdict     = FAULT_LATCHED_OUTPUT_DISABLED
```

Playback reaches `PlaybackPhase::Pup` and fails:

```
DIAG_FAIL amp=i2c0 stage=PLAYBACK_PREPARE code=POWER_UP_TIMEOUT
          reg=0x00010010 expected=0x01000000 actual=0x00400000
DIAG_SNAPSHOT irq=[0x00400000 0x00300002 0x20000080 0x00000003] pll_lock=0
```

`0x00010010` is `CS35L41_IRQ1_STATUS1`. `0x01000000` is `PUP_DONE`; it never
sets. `0x00400000` (bit 22) has no name in the Linux header and no reference in
this repository. `IRQ1_RAW_STATUS3` shows `PLL_UNLOCK` set and `PLL_LOCK` clear.

## Root cause

The amplifiers are I2S **clock consumers**:

```
CS35L41_SP_FORMAT   (0x00004808) = 0x20200200   // 32-bit slots, I2S, clk consumer
CS35L41_SP_RATE_CTRL(0x00004804) = 0x00000021   // ASP_BCLK = 3.072 MHz
CS35L41_PLL_CLK_CTRL(0x00002C04) = 0x00000430   // BCLK input, PLL_REFCLK_EN
```

so they cannot start until the codec drives BCLK down the ASP. That part of the
diagnosis held: with no bit clock the PLL never locks, `PUP_DONE` never sets, and
playback fails with `POWER_UP_TIMEOUT` even though firmware, calibration and DSP
boot are all healthy.

What was wrong was the conclusion drawn from it. The codec route, not the
amplifier driver, is what supplies the clock, and the ALC245 route in use was the
wrong one.

### The fix: ALC245 layout 13

The machine was booting with `alcid=11`. Layout 11 is documented as
"ALC235 for Lenovo by soto2080" and drives an analog speaker path; it never
exercises the codec path that feeds the amplifiers, so the ASP stayed clockless.
Layout 13 is "lunjielee - Realtek ALC245 for HP Omen 2020" and does drive it.

Changing `alcid=11` to `alcid=13` in the OpenCore boot arguments was the entire
fix. Internal speakers then played, confirmed by ear. The amplifier driver did
not need to change:

```
Cirrus_Amplifier_Status = READY
Cirrus_Driver_Verdict   = READY_DSP
```

So the earlier statement that macOS has no way to drive the codec bit clock was
wrong, and so was the claim that this stack never produced audible output.
Upstream demonstrates working speakers for the Legion 7 in the InsanelyMac
thread (video posted), and the same mechanism works here once the layout is
correct.

AppleALC needs no changes. Its ALC245 resources already contain the required
route in layout 13; no CS35L41-specific support is involved on the codec side.

## Internal microphone: out of scope for this driver

Capture does not work, and the evidence says the internal array is not connected
to the ALC245 at all. Three ALC245 layouts were tested on hardware:

| `alcid` | Speakers | `Built-in Input` | Capture |
| --- | --- | --- | --- |
| 11 | no | yes (2 ch) | not tested |
| 12 | yes | yes (2 ch, 48 kHz) | digital silence, peak 0.0 |
| 13 | yes | no | n/a |

On layout 12 the input engine publishes and its controls are live
(`IOAudioLevelControlMaxValue = 63`, `IOAudioControlValue = 48`, two mute toggles
at `IOAudioControlValue = 0`). Forcing the input volume to 1.0 and clearing mute
through CoreAudio succeeded (`r=0` on both channels), yet a 3 second
`AVAudioEngine` tap at 48 kHz still returned `peak = 0.0` over 139200 frames.
Exact zero across that many frames is the signature of a missing signal path, not
a low gain.

Both layouts route capture through codec node 25 (`0x19`) via `8 -> 35 -> 25` with
`Boost = 3`, so AppleHDA is driving the codec's analog microphone pin. That pin is
the 3.5 mm headset input, not the built-in array.

### The array is on the AMD ACP

The board exposes a second audio controller with no driver attached on macOS:

```
+-o ACP@0,5  <class IOPCIDevice, id 0x100000225>
      "vendor-id"   = <22100000>   // 0x1002 AMD
      "device-id"   = <e2150000>   // 0x15E2
      "subsystem-vendor-id" = <3c100000>
      "class-code"  = <00800400>   // multimedia audio
      "acpi-path"   = "IOACPIPlane:/_SB/PCI0@0/GP17@80001/ACP@5"
```

Linux enumerates a card named for this exact machine against it:

```
2 [acp]: acp - acp
          HP-HPENVYx3602_in_1Laptop15_ey0xxx-Type1ProductConfigId-8A31
```

Independent corroboration that the array hangs off `1022:15E2` rather than the
codec:

- A Linux report for `HP ENVY x360 15-ey0013dx`, subsystem `103C:8A31`, lists
  `Device-2: AMD ACP/ACP3X/ACP6x Audio Coprocessor driver: snd_rn_pci_acp3x
  chip-ID: 1022:15e2` alongside `card 1: Generic_1 ... ALC245 Analog`.
- Kernel Bugzilla #215119 states the AMD array microphone is "directly connected
  to the AMD CPU via the AMD ACP (Audio Co-Processor, PCI ID 1022:15e2)" and is
  driven by `snd_rn_pci_acp3x`.
- A Windows driver inventory for `HP ENVY x360 2-in-1 Laptop 15-ey0xxx` lists
  `Microphone Array (AMD Audio Device)` and `AMD Audio CoProcessor` separately
  from `Realtek(R) Audio`. Windows naming the array an *AMD Audio Device* rather
  than a Realtek one matches the ACP path.

The DSDT agrees: `Device (ACP)` is bare (`_ADR 0x05`, a `_WOV` method, no
children), there is no `DMIC`/`PDM` device anywhere in the namespace, and the
codec pin configuration describes only analog codec pins. The firmware does not
describe a codec-attached digital microphone.

### Consequence

The internal microphone would need an AMD ACP PDM capture driver on macOS, backed
by the ACP DMA engine at `1022:15E2`. Neither AppleALC nor CirrusAudioFixup can
provide that, and no such macOS driver is known to exist. Layouts that expose
`Built-in Input` only expose the analog headset pin, which captures silence when
nothing is plugged in.

Treat this as a hardware/driver-availability limit, not an open bug in this kext.

## Layout 12 is not a safe default

Layout 13 is the only layout validated for speaker playback. Layout 12 was tried
because it additionally publishes `Built-in Input`, and while speakers did play,
output sounded worse than layout 13 and the machine crashed during the test. No
panic report was collected, so the cause is unestablished: it could be the
Lenovo-labelled layout mismatching this HP board, or something in the amplifier
path. Until a panic log says otherwise, **use layout 13**.

## Verified fixes on branch `envy-103c8a31`

| Commit | Change |
| --- | --- |
| `9530520` | Drive the shared ENVY reset on AMDI0030 GPIO 30 |
| `2a0f0fd` | Compile the vtable fixture with explicit `-std=c++17` |
| `48f57e8` | Honour the `"*"` I2C controller wildcard in preflight identity |
| `3bd9e43` | Key ASP validation and CRC properties on channel, not endpoint label |

OpenCore EFI changes (separate repository at `~/dev/OpenCore-EFI`):

| Commit | Change |
| --- | --- |
| `1658f31` | Drop HPET IRQ 11 so `I2CB` can claim it — enables the controller driver and `SPKR` device nub |
| `8cbfaab` | Install Lilu 1.7.2, AppleALC 1.9.8, custom VoodooI2C, CirrusAudioFixup |
| `61e0cf2` | Redeploy with GPIO 30 reset, drop `-cirrusro` |
| `5988efa` | Redeploy with the wildcard preflight fix |
| `867f2ab` | Redeploy with channel-keyed ASP diagnostics |
| `97a2f99` | Enable `-cirrusdbg` for playback tracing |

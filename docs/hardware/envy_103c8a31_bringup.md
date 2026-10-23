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

The amplifiers are configured as I2S **clock consumers**:

```
CS35L41_SP_FORMAT   (0x00004808) = 0x20200200   // 32-bit slots, I2S, clk consumer
CS35L41_SP_RATE_CTRL(0x00004804) = 0x00000021   // ASP_BCLK = 3.072 MHz
CS35L41_PLL_CLK_CTRL(0x00002C04) = 0x00000430   // BCLK input, PLL_REFCLK_EN
```

The PLL can only lock while the codec drives BCLK down the ASP. Nothing on
macOS does. `PUP_DONE` is a consequence, not the fault: with no bit clock the
power-up sequence cannot complete, and the observed bit 22 is not an error flag.

Linux does not hit this because the Realtek HDA driver registers the amplifiers
as HDA components and the codec drives the stream:

```c
/* sound/pci/hda/patch_realtek.c */
static void cs35l41_fixup_i2c_two(struct hda_codec *cdc, const struct hda_fixup *fix, int action)
{
    cs35l41_generic_fixup(cdc, action, "i2c", "CSC3551", 2);
}

static void cs35l41_generic_fixup(struct hda_codec *cdc, int action, const char *bus,
                                  const char *hid, int count)
{
    spec->comps[i].codec = cdc;
    component_match_add(dev, &spec->match, comp_match_cs35l41_dev_name, rec);
    ...
    spec->gen.pcm_playback_hook = comp_generic_playback_hook;
}
```

AppleALC has no equivalent. A clone of `acidanthera/AppleALC` at `master`
contains no reference to `CS35L41`, `CSC3551`, or HDA component binding, and
`Resources/ALC245/` provides layouts 11, 12 and 13 with analog paths only.

## Correction to the repository's own documentation

The claim that this stack produces working internal speakers is not supported by
the repository history.

Commit `83f8e06` (`docs: prepare project for public contribution`, 2026-10-06)
is documentation-only — nine files, no driver sources — and it removed:

> **Project Status: Experimental / Audio Output is NOT Working**
>
> **At present, this kext DOES NOT play audible sound.** While it successfully
> communicates with the hardware and initializes the amplifiers, the system
> remains muted due to DSP firmware locking and clock-sync issues with AppleHDA.

Commit `4d96a85` (`release: CirrusAudioFixup 1.1.1`) added the `Working` rows to
the supported-laptop table.

The pre-rewrite README stated that Legion operation required AppleALC
`layout-id 16` and `Platforms16.xml`. Neither file exists in this repository, and
stock AppleALC ships ALC287 layouts 11, 13 and 21 only. `CHANGELOG.md` still
refers to "Realtek ALC287 layout 16". The one supported row with a matching
artifact is ASUS `1043:1C9F` ("Working with layout 21"), which exists upstream.

Treat the Legion 7 `17AA:3847` and ZBook Fury `103C:89C6` rows as unverified
until speaker playback is demonstrated on that hardware.

## What is not the problem

Ruled out with evidence on this machine:

- I2C transfer path and the custom VoodooI2C fork. `VoodooI2CTransferToAddress`
  is exported and the transfer returns the correct device ID.
- Board identity and profile selection. `profile=hp-envy-ey0xxx-csc3551`,
  `board=0x103C8A31`.
- Reset. `Cirrus_Reset_Policy = AUTHORIZED`; the shared GPIO 30 line is driven.
- Firmware, calibration, DSP boot and ASP configuration. All verified above.
- Masked `PUP_DONE`. Linux polls the same masked `IRQ1_STATUS1` bit and works,
  because its PLL locks. Unmasking or polling `RAW_STATUS1` does not create a
  clock.
- Boost topology and reset pin differences against Legion. They are real
  (internal vs external boost, pin 30 vs pin 6) but both are downstream of the
  missing bit clock.

## Remaining work

The missing piece is codec-side: something on macOS must make the ALC245 emit
I2S at 48 kHz / 3.072 MHz BCLK. Options, in order of increasing cost:

1. Determine whether the ALC245 digital output is enabled by a vendor
   coefficient verb sequence that AppleALC can carry in a layout resource.
2. Determine whether the signal path is instead the AMD ACP (`card 2 acp` in
   Linux) rather than the HDA codec, which would move the problem to a different
   controller driver.
3. If neither is reachable, the clock boundary is not solvable inside
   CirrusAudioFixup and the amplifier work stays a verified-power-sequence
   reference implementation.

Evidence needed next: a Linux codec dump while playing, and identification of
which ALSA card carries the speaker stream.

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

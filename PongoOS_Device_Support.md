<!--
SPDX-FileCopyrightText: 2026 feiyehua
SPDX-License-Identifier: GPL-2.0-only
-->
# PongoOS Fork: Multi-Device and Blobless Boot Support

This document describes the adaptations in the [atera1n-PongoOS](https://github.com/feiyehua/atera1n-PongoOS) fork to support all A8 devices and the blobless boot mode.

## Device Matrix

| Device | Board | Identifier | ChipID | BoardID | Baseband | SEPROM |
|--------|-------|-----------|--------|---------|----------|--------|
| iPad mini 4 (Wi-Fi) | j96ap | iPad5,1 | 0x7000 | 8 | No | A8 T7000 |
| iPad mini 4 (Cell) | j97ap | iPad5,2 | 0x7000 | 10 | Yes | A8 T7000 |
| iPhone 6 | n61ap | iPhone7,2 | 0x7000 | 10 | Yes | A8 T7000 |
| iPod touch 6 | n102ap | iPod7,1 | 0x7000 | 2 | No | A8 T7000 |

All devices share ChipID 0x7000 (Apple A8 / T7000). The SEPROM is identical, so the Blackbird exploit and SEP commands work without modification across all of them.

## SEP Commands

All commands are device-agnostic across all A8 (T7000) devices:

| Command | Description |
|---------|-------------|
| `sep shc` | Trigger the Blackbird shared-cache race to dump SEP protected memory |
| `sep observe` | Monitor SEP boot to find tick timing for the replay attack |
| `sep replay <tick>` | Race the SEP memory write at the given tick count |
| `sep load` | Load a binary payload into SEP after gaining code execution |
| `sep jump <addr>` | Jump to a loaded payload's entrypoint in SEP |
| `sep peek <addr>` | Read SEP memory regions (used to extract ART key) |
| `sep poke <addr> <val>` | Write a 32-bit value to SEP memory |
| `sep blobless [tick]` | **Blobless boot: Blackbird exploit + SEPOS manifest patch + boot** |
| `sep boot` | Boot a pwned SEPOS image |
| `sep encrypt/decrypt` | KBAG operations via SEP AES engine |

## Blobless Boot Support

### `sep blobless` command

The `sep blobless` command (`a8_sep_exploit.c:sep_blobless()`) orchestrates the entire blobless boot sequence in a single step:

1. **Blackbird exploit**: Replays captured shellcode to gain SEP code execution via the shared-memory race condition. Takes an optional tick parameter (default 900000).

2. **SEPOS decrypt and reload**: Calls `reload_sepi()` which:
   - Decrypts the SEPOS firmware via the SEP AES engine (KBAG → GID key)
   - Applies the standard BPR check patch (`sepfw_kpf()`)
   - Applies the blobless manifest hash patch (`sepfw_blobless_kpf()`)
   - Writes the patched SEPOS back into SEP memory

3. **Boot**: Calls `sep_blackbird_boot()` to trigger the SEP boot sequence with the patched SEPOS.

### `sepfw_blobless_kpf()` — Manifest Hash Patcher

Implemented in `sep.c`, this kernel patch function scans the decrypted SEPOS binary for manifest hash validation sequences:

```
Pattern:  CMP Rn, Rm    (ARM: cccc 0001 0101 xxxx 0000 xxxx 0000 xxxx)
          BNE <error>    (ARM: 0001 1010 xxxx xxxx xxxx xxxx xxxx xxxx)

Patch:    CMP Rn, Rm    (unchanged)
          NOP            (ARM: E1A00000 — MOV R0, R0)
```

This makes the manifest hash comparison always fall through as "match", so SEPOS accepts any firmware manifest regardless of what ART contains. Combined with the existing `sepfw_kpf()` BPR patch, SEPOS boots without verifying the manifest hash or BPR state.

### `sep-blobless-boot.S` — SEP-side Shellcode

An alternative SEP-side payload (`shellcode/sep-blobless-boot.S`) that runs inside the pwned SEP after Blackbird exploitation. It scans SEPOS memory directly for the same CMP+BNE pattern and NOPs the branches. Reports patch count via mailbox opcode 0xB0 (success) / 0xB1 (no patches found). Falls through to the standard peek/poke/branch command loop.

## IM4P Description String

The fake IM4P used by `sep_observe()` previously contained a hardcoded `'iPhone'` description string. This has been changed to a generic `'sep0'` string so it works identically across all A8 device types (iPad mini 4, iPhone 6, iPod touch 6).

## Building

PongoOS is built per-SoC, not per-device. A single A8 (T7000) build works for all supported devices:

```
build/
  Pongo.bin              # Universal A8 PongoOS image
shellcode/
  sep-pwn-smoke.bin      # Blackbird initial shellcode
  sep-artk-derive.bin    # ART key derivation payload
  sep-blobless-boot.bin  # Blobless boot SEP-side payload
```

palera1n and checkra1n handle per-device iBSS/iBEC selection automatically.

## Boot Flow: Untethered vs Blobless

### Untethered (requires SHSH blobs)
```
DFU → checkm8 → PongoOS → Blackbird → Extract K_ART → (reboot)
                                                        ↓
DFU → checkm8 → PongoOS → Blackbird → SEP payload → Read ART key
                                                        ↓
Ramdisk → Patch ART in NVRAM with new manifest hash + HMAC
                                                        ↓
Restore device → Device boots independently from now on
```

### Blobless (no SHSH blobs needed)
```
(One-time) DFU → checkm8 → PongoOS → sep shc → Save shellcode
                                                        ↓
(One-time) Restore device (ART is NOT modified)
                                                        ↓
(Every boot) DFU → checkm8 → PongoOS → /send shellcode.bin
                                      → sep blobless 470000
                                                        ↓
             SEP accepts downgraded SEPOS → Device boots
```

## Testing Checklist

- [ ] iPad mini 4 Wi-Fi (j96ap): Blackbird exploit
- [ ] iPad mini 4 Wi-Fi (j96ap): ART key extraction
- [ ] iPad mini 4 Wi-Fi (j96ap): Untethered restore
- [ ] iPad mini 4 Wi-Fi (j96ap): Blobless boot
- [ ] iPhone 6 (n61ap): Blackbird exploit
- [ ] iPhone 6 (n61ap): ART key extraction
- [ ] iPhone 6 (n61ap): Untethered restore (with futurerestore + baseband)
- [ ] iPhone 6 (n61ap): Blobless boot
- [ ] iPod touch 6 (n102ap): Blackbird exploit
- [ ] iPod touch 6 (n102ap): ART key extraction
- [ ] iPod touch 6 (n102ap): Untethered restore
- [ ] iPod touch 6 (n102ap): Blobless boot

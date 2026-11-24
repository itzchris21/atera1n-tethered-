<!--
SPDX-FileCopyrightText: 2026 feiyehua
SPDX-License-Identifier: GPL-2.0-only
-->
# Full Downgrade Guide: A8/A8X Devices

Step-by-step guide to downgrade iOS on A8 and A8X devices using the Blackbird SEP exploit.

## Choose Your Path

| | Blobless (Tethered) | Untethered |
|---|---|---|
| SHSH blobs needed? | No | Yes |
| Computer needed every boot? | Yes | No |
| ART modified? | No | Yes |
| Brick risk | None | Low (bad ART is recoverable) |
| Best for | No blobs, testing, first-timers | Daily driver, permanent downgrade |

**If you don't have saved SHSH blobs, blobless is your only option.**

## Prerequisites

### Hardware
- One of the supported devices:
  - **A8 (T7000):** iPad mini 4, iPhone 6, iPhone 6 Plus, iPod touch 6
  - **A8X (T7001):** iPad Air 2
- A computer (macOS or Linux) with USB

### Software
- **palera1n** or **checkra1n** (for checkm8 exploit and PongoOS loading)
- **pongoterm** (to interact with PongoOS shell)
- **[atera1n-PongoOS](https://github.com/feiyehua/atera1n-PongoOS)** built with blobless support, or a prebuilt Pongo.bin from GitHub Actions
- **[idevicerestore](https://github.com/feiyehua/idevicerestore)** (patched fork) — for Wi-Fi-only devices
- **[futurerestore](https://github.com/futurerestore/futurerestore)** — for devices with cellular baseband
- **Python 3** (for helper scripts)
- **SSHRD_Script** (for ramdisk access in untethered flow)

### Firmware
- The target IPSW for the iOS version you want to downgrade to
  - Download from [ipsw.me](https://ipsw.me) or The Apple Wiki
  - iOS 8.x through 12.x should work on these devices
- For untethered: saved SHSH blobs (.shsh2) for the target iOS version with a known generator

## Part A: Blobless Downgrade (No SHSH Blobs, Tethered)

This is the safer and simpler path. The device needs a computer for every boot.

### Step 1: Capture Blackbird Shellcode (once per device)

1. Put your device in **DFU mode**:
   - Connect to computer via USB
   - Hold Power + Home for 10 seconds
   - Release Power, keep holding Home for 5 more seconds
   - Screen stays black (not the iTunes/recovery logo)

2. Boot PongoOS:
   ```
   palera1n -p  # or use checkra1n to load Pongo.bin
   ```

3. Open pongoterm and capture shellcode:
   ```
   sep shc
   ```
   You'll see hex output like:
   ```
   0xe3463238a41605c
   0xffabba5ecee9e59a
   ...
   ```

4. Save the output to a binary file:
   ```
   python3 scripts/hex_to_bin.py > shellcode.bin
   ```
   The file should be **448 bytes**.

5. Keep `shellcode.bin` safe — you need it for every boot.

### Step 2: Restore the Target iOS (once)

Extract the target IPSW (it's a zip file) to get the firmware components. The filenames depend on your device:

| Device | SEP firmware filename |
|--------|----------------------|
| iPad mini 4 (Wi-Fi) | sep-firmware.j96.RELEASE.im4p |
| iPad mini 4 (Cell) | sep-firmware.j97.RELEASE.im4p |
| iPhone 6 | sep-firmware.n61.RELEASE.im4p |
| iPhone 6 Plus | sep-firmware.n56.RELEASE.im4p |
| iPod touch 6 | sep-firmware.n102.RELEASE.im4p |
| iPad Air 2 (Wi-Fi) | sep-firmware.j81.RELEASE.im4p |
| iPad Air 2 (Cell) | sep-firmware.j82.RELEASE.im4p |

**For devices without baseband** (iPad mini 4 Wi-Fi, iPod touch 6, iPad Air 2 Wi-Fi):
```
idevicerestore --erase /path/to/target.ipsw
```

**For devices with baseband** (iPhone 6, iPhone 6 Plus, iPad mini 4 Cell, iPad Air 2 Cell):
```
futurerestore --latest-baseband /path/to/target.ipsw
```

The device **will not boot on its own** after the restore. This is expected — you haven't set up the blobless boot yet.

### Step 3: Blobless Boot (every boot)

Every time the device powers off or reboots, repeat this:

1. Put device in **DFU mode** (same as Step 1).

2. Boot PongoOS:
   ```
   palera1n -p
   ```

3. In pongoterm, send your shellcode and run blobless:
   ```
   /send /path/to/shellcode.bin
   sep blobless 470000
   ```

4. Wait for the output:
   ```
   blobless: starting Blackbird exploit (tick=470000)...
   blobless: booting tz0...
   blobless: loading ART...
   blobless: ART accepted: 1
   blobless: SEP pwned! Loading and patching SEPOS...
   blobless: SEPOS loaded with manifest hash patch
   blobless: booting patched SEPOS...
   blobless: done - SEP downgrade active (tethered)
   ```

5. The device boots into the downgraded iOS.

**If it fails**, try different tick values. Run `sep observe` first to find the right timing window for your device — pick a tick between the 2nd and 3rd `observed_value` change.

---

## Part B: Untethered Downgrade (Requires SHSH Blobs)

The device boots on its own after setup, but ART is modified (recoverable if something goes wrong).

### Step 1: Extract the ART Key

1. Put device in DFU and boot PongoOS (same as Part A, Step 1).

2. Capture shellcode:
   ```
   sep shc
   ```
   Save to `shellcode.bin` (448 bytes).

3. Reboot device into DFU and boot PongoOS again.

4. Replay the shellcode to gain SEP code execution:
   ```
   /send /path/to/shellcode.bin
   sep replay 470000
   ```
   You should see `sep pwned with opcode ab`.

5. Load the ART key derivation payload:
   ```
   /send /path/to/sep-artk-derive.bin
   sep load
   ```

6. Jump to the payload:
   ```
   sep jump 1000
   ```

7. Read the 32-byte K_ART key:
   ```
   sep peek 10140000
   sep peek 10140004
   sep peek 10140008
   sep peek 1014000c
   sep peek 10140010
   sep peek 10140014
   sep peek 10140018
   sep peek 1014001c
   ```
   Save these 8 values — this is your K_ART (ART HMAC key).

### Step 2: Patch ART in NVRAM

1. Boot into a ramdisk using SSHRD_Script:
   ```
   ./sshrd.sh 15.8.3  # or whatever your current iOS version is
   ./sshrd.sh boot
   ./sshrd.sh ssh
   ```

2. Set your SHSH blob's generator as the boot nonce:
   ```
   nvram com.apple.System.boot-nonce=0x<your_generator>
   ```

3. Read and back up your current ART:
   ```
   /usr/libexec/seputil --art get
   ```
   **Save this output!** If anything goes wrong, you need this to recover.

4. Patch the ART with your SHSH blob's manifest hash:
   ```
   python3 scripts/patch_art.py \
     --shsh /path/to/your.shsh2 \
     --art-hex <current_art_hex> \
     --art-key <k_art_hex>
   ```
   This replaces the manifest hash and restore nonce in ART and recalculates the HMAC.

5. Write the patched ART to NVRAM:
   ```
   nvram "com.apple.System.sep.art=%<patched_art_hex>"
   ```
   Note: the `%` prefix is required for binary NVRAM values.

6. Save NVRAM changes:
   ```
   nvram -s
   ```

7. Reboot into DFU. Verify your APNonce:
   ```
   irecovery -q
   ```

### Step 3: Restore

**For devices without baseband** (iPad mini 4 Wi-Fi, iPod touch 6, iPad Air 2 Wi-Fi):
```
idevicerestore --erase -T /path/to/your.shsh2 /path/to/target.ipsw
```

**For devices with baseband** (iPhone 6, iPhone 6 Plus, iPad mini 4 Cell, iPad Air 2 Cell):
```
futurerestore -t /path/to/your.shsh2 --latest-baseband /path/to/target.ipsw
```

Wait for the restore to complete. The device should boot on its own into the downgraded iOS.

---

## iPad Air 2 (A8X) Notes

The iPad Air 2 uses the A8X (T7001) chip. Most of the process is identical, but:

1. **Always run `sep observe` first** to verify the driver detects the SEP correctly and find your tick timing.
2. The `sep blobless` command will print `blobless: running on A8X (T7001) - iPad Air 2` to confirm detection.
3. If `sep shc` hangs without producing output, the SEP-internal mailbox VA may differ on A8X. See the [Troubleshooting guide](Troubleshoot.md) for how to find the correct address.

## Troubleshooting

### "sep pwned" never appears / race always fails
- Try different tick values. Use `sep observe` to find the window.
- Start with 470000, adjust by 10000 in either direction.
- The race is timing-sensitive and can fail intermittently — retry a few times.

### Device stuck on Apple logo after blobless boot
- The SEPOS manifest patch may not have applied. Force restart into DFU and try again.
- Check if `sepfw_blobless_kpf` reported "no manifest hash checks found" — the target iOS version may have a different SEPOS instruction pattern.

### Restore fails with "unable to send iBEC" or similar
- Make sure you're using the patched idevicerestore/futurerestore fork.
- Verify the device is in DFU mode (black screen, not recovery mode).

### Restore fails with baseband error (iPhone 6/6 Plus, cellular iPads)
- Use futurerestore with `--latest-baseband`.
- If the latest baseband is incompatible with your target iOS, try `--no-baseband` if you don't need cellular.

### "messed up ART" / device refuses to restore
- See the [full ART recovery procedure](Troubleshoot.md#ive-messed-up-my-art-and-my-device-now-refuse-to-restore-even-to-the-latest-version) in the troubleshooting guide.
- Boot a ramdisk, personalize a fresh RestoreSEP, and run `seputil --restore` followed by `seputil --commit-hash`.

### iOS 8/9 specific: PongoOS can't find SEPOS firmware
- Pre-iOS 13 iBEC doesn't copy SEP firmware to the expected buffer.
- Boot a ramdisk and extract the SEPOS firmware manually from the device's filesystem.

### iOS 8/9 specific: blobless patcher finds no patterns
- Very early SEPOS versions may use different instruction sequences for manifest validation.
- Try a newer target iOS version (9.x or 10.x) first to verify the exploit works, then work backward.

## Quick Reference

### Blobless boot command sequence (every boot)
```
# 1. DFU mode
# 2. Boot PongoOS
palera1n -p

# 3. In pongoterm:
/send /path/to/shellcode.bin
sep blobless 470000
```

### Full untethered command sequence
```
# --- Extract K_ART (once) ---
# Boot 1: sep shc → save shellcode.bin
# Boot 2:
/send shellcode.bin
sep replay 470000
/send sep-artk-derive.bin
sep load
sep jump 1000
sep peek 10140000  # through 1014001c → save K_ART

# --- Patch ART (once, via ramdisk) ---
nvram com.apple.System.boot-nonce=0x<generator>
/usr/libexec/seputil --art get  # backup!
python3 scripts/patch_art.py --shsh blob.shsh2 --art-hex <hex> --art-key <key>
nvram "com.apple.System.sep.art=%<patched>"
nvram -s

# --- Restore (once) ---
idevicerestore --erase -T blob.shsh2 target.ipsw  # no baseband
# OR
futurerestore -t blob.shsh2 --latest-baseband target.ipsw  # with baseband
```

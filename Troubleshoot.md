<!--
SPDX-FileCopyrightText: 2026 feiyehua
SPDX-License-Identifier: GPL-2.0-only
-->
# Troubleshoot

## I've messed up my ART, and my device now refuse to restore, even to the latest version.
This can be a little scary, because the screen flashes and dims slowly. But this is fixable.
Thanks [@tihmstar](https://x.com/tihmstar/status/1943359951750991932?s=46) for providing a way to fix this.

This procedure works on all supported A8 devices (iPad mini 4, iPhone 6, iPod touch 6).

0. If you have backed up your ART, restore it. Otherwise, follow the instructions bellow.
1. Enter DFU mode, and obtain a APTicket for the latest signing firmware. Personalize the RestoreSEP with the APTicket.
2. Boot into your Ramdisk, and ssh into your device. 
3. Run the following commands:
```
/usr/sbin/hdik -nomount ram://10240
newfs_hfs /dev/disk4
mount_hfs /dev/disk4 /mnt6
```
This creates a disk with write permission. 
4. `scp` your personalized RestoreSEP to the mountpoint, then run `/usr/libexec/seputil --restore /mnt6/path/to/RestoreSEP.img4`
5. Run `/usr/libexec/seputil --commit-hash`
6. Run `/usr/libexec/seputil --art get`, you should now see a brand new ART:
```
raw ART: 304802010030210203077ee10414d077cb6475818095860d8251147827747fd7288704000400310004206afe56b2c6200ceeb81763caf2511d3f6d7a8e6f2a2034831d4a5035c2c666a2
Successfully parsed ART:
counter: 491233
manifest hash (20 bytes): d077cb6475818095860d8251147827747fd72887
sleep hash absent
restore nonce absent
```
4. Restore your device again.

## SEP panics during `sep replay` on iPhone 6 or iPod touch 6
The tick value for `sep replay` may differ slightly between devices even though they all use A8. If 470000 doesn't work:
1. Run `sep observe` first to see the tick values on your specific device.
2. Choose a tick between the 2nd and 3rd `observed_value` change.
3. Try values in increments of 10000 around the midpoint of that range.

## iPhone 6: Restore fails with baseband error
iPhone 6 has a cellular baseband that must be handled during restore:
- Use futurerestore with `--latest-baseband` to let it fetch the currently signed baseband.
- If you need a specific baseband version, supply it with `--baseband` and `--baseband-manifest`.
- If cellular is not needed, check if your restore tool supports `--no-baseband`.

## iPod touch 6: PongoOS doesn't detect SEPOS firmware
Same as iPad mini 4 — on older iOS versions (pre-iOS 13), iBEC doesn't copy SEP firmware to the expected buffer. Enter Ramdisk to extract the SEPOS firmware manually.

## Blobless boot: Device doesn't boot after restore
This is expected if you followed the blobless path (skipped Part II). The device needs the blobless boot chain on every boot:
1. Put the device in DFU mode.
2. Load PongoOS via checkm8.
3. Send captured shellcode and run `sep blobless 470000`.
4. The device should then continue booting.

If the device is stuck on the Apple logo after the blobless boot, the SEPOS manifest patch may not have applied correctly. Force restart into DFU and try again with a different tick value.

## Blobless boot: "blobless: WARNING - no manifest hash checks found to patch"
This means `sepfw_blobless_kpf()` didn't find the expected CMP+BNE pattern in the SEPOS binary. Possible causes:
- The SEPOS firmware version has a different instruction sequence for manifest validation. Try a different target iOS version.
- The SEPOS firmware was not loaded correctly. Ensure you uploaded a valid `sepi` IMG4 from the target IPSW.

## Blobless boot: SEP still rejects the downgraded firmware
Ensure you are using the atera1n-PongoOS build with blobless support (the `sep blobless` command must be available). The standard PongoOS or upstream checkra1n PongoOS does not include the blobless SEPOS patching.

## Blobless boot: "blobless: Blackbird exploit FAILED"
The Blackbird race failed. This is timing-sensitive:
1. Try different tick values (start with 470000, adjust by 10000).
2. Use `sep observe` to find the correct timing window.
3. Retry — the race can fail intermittently.


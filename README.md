# Exploit Blackbird on A8 Devices and Achieve Tethered/Untethered Downgrade with Blobs

**WARNING: THIS IS ONLY A PROOF OF CONCEPT. IF YOU BUILD AND RUN THE PAYLOAD ON YOUR DEVICE, YOU'RE AWARE OF THE RISK OF PERMANENT LOSS OF YOUR DATA AND PERMANENT BOOTLOOP OF YOUR DEVICE! USE AT YOUR OWN RISK!**

However, I've tried to restore to iOS 11.4 from iOS 15.8.8 on my iPad mini 4 Wi-Fi, and confirmed everything works fine, including password, Touch ID, and sleep. If you followed this instruction properly, you should also be able to restore arbitrary version that you have a valid SHSH (also, a generator).

And, everything before overwriting ART should be read only and won't compromise your data and device.

## Supported Devices

All A8 (T7000) and A8X (T7001) devices with checkm8 support:

| Device | Identifier | Board | SoC | Baseband | Notes |
|--------|-----------|-------|-----|----------|-------|
| iPad mini 4 (Wi-Fi) | iPad5,1 | j96ap | A8 (T7000) | No | Tested and confirmed working |
| iPad mini 4 (Cellular) | iPad5,2 | j97ap | A8 (T7000) | Yes | Needs futurerestore for baseband |
| iPhone 6 | iPhone7,2 | n61ap | A8 (T7000) | Yes | Needs futurerestore for baseband |
| iPhone 6 Plus | iPhone7,1 | n56ap | A8 (T7000) | Yes | Needs futurerestore for baseband |
| iPod touch (6th gen) | iPod7,1 | n102ap | A8 (T7000) | No | No baseband, similar to iPad mini 4 Wi-Fi |
| iPad Air 2 (Wi-Fi) | iPad5,3 | j81ap | A8X (T7001) | No | A8X — see notes below |
| iPad Air 2 (Cellular) | iPad5,4 | j82ap | A8X (T7001) | Yes | A8X — needs futurerestore for baseband |

### Untethered vs Blobless

This project supports two modes:

- **Untethered** (Part I + II + III below): Modify ART permanently so the device boots the downgraded firmware on its own. Requires saved SHSH blobs. Riskier, but the device can boot independently after restore.
- **Blobless** (see [Blobless Boot Guide](#blobless-boot-guide) below): No SHSH blobs needed. Uses Blackbird to patch SEPOS manifest validation in-memory on every boot via the `sep blobless` PongoOS command. Requires a computer for every boot (tethered), but ART is never modified. Recommended for first-time users or when you don't have saved blobs.

### Device-Specific Notes

**iPhone 6 (n61ap):** Has a cellular baseband. You **must** use a patched version of [futurerestore](https://github.com/futurerestore/futurerestore) instead of idevicerestore for the restore step, or supply a compatible baseband firmware. Without the correct baseband, cellular will not function. If you don't need cellular (e.g., using as an iPod), you can try the idevicerestore path with `--no-baseband` if supported.

**iPhone 6 Plus (n56ap):** Same A8 (T7000) SoC as iPhone 6. Has cellular baseband — same futurerestore requirements as iPhone 6.

**iPod touch 6th gen (n102ap):** No baseband, so the restore process is essentially the same as iPad mini 4 Wi-Fi. Use idevicerestore directly.

**iPad Air 2 (j81ap/j82ap):** Uses A8X (T7001) — a different chip from A8 but shares the same 32-bit SEP architecture. The AP-side mailbox offset differs (0x4000 vs 0x1000 on A8), which the driver handles automatically. The SEP-side shellcode payloads use T7000 mailbox virtual addresses — these may need adjustment for T7001. Use `sep observe` on the device first to verify correct behavior. The j82ap (Cellular) variant needs futurerestore for baseband.

**Tick values for `sep replay`:** The empirical tick value varies by device. Start with 470000 and adjust. Use `sep observe` to find the correct window between the 2nd and 3rd `observed_value` change. Known starting points:
- iPad mini 4: ~470000
- iPhone 6 / 6 Plus: ~470000 (similar A8 timing, may need adjustment)
- iPod touch 6: ~470000 (similar A8 timing, may need adjustment)
- iPad Air 2: ~470000 (A8X timing may differ — use `sep observe`)

## Full Downgrade Guide

For a complete step-by-step walkthrough of both the blobless and untethered downgrade paths, see the **[Downgrade Guide](Downgrade_Guide.md)**.

## Part I Exploit SEP and Obtain ART Key (Untethered)
1. Put your device in DFU mode, and use palera1n or checkra1n to start [this](https://github.com/feiyehua/atera1n-PongoOS) PongoOS image. You can build yourself, or use the image build by Github Action. PongoOS supports all A8 devices (iPad mini 4, iPhone 6, iPod touch 6) — make sure to use the correct build for your device.

2. Type `sep shc` in pongoterm. Something like this will appear on pongoterm: 
```
0xe3463238a41605c
0xffabba5ecee9e59a
0x55f7ce58d7f5f74a
0x1aba653c2d5a107a
0xf3bf3ad3bc25ede8
0x6ec68f684695b09
0x0
0x0
0xc28e487fa55a3b62
0xa511f6acc89162df
0xd94d49cb06b1db1f
0x63faeeb7467bbfce
0x3f4dd5d77a8ae024
0xd757387664aa9abb
0x0
0x0
```
Save these bytes as a binary file. `scripts/hex_to_bin.py` should help you. The file should have 448 bytes.

3. Reboot your device into DFU and boot PongoOS once again. Enter pongoterm on your computer.

4. Type `/send /absolute/path/to/previously/saved/binary.bin`, then type `scp replay 470000`. The argument is the number of tick that the first race overwrite starts. Empirically 470000 works on my iPad mini 4. If this doesn't work, SEP will panic. You can also run `sep observe` to see when does the `observed_value` change. Take this as an example:
```
#==================
#
# pongoOS 2.6.3-3d40bea0
#
# https://checkra.in
#
#==================
Booted by: iBoot-7459.140.15
Built with: Clang 21.0.0 (clang-2100.1.1.101)
Running on: Apple A8 (T7000)
pongoOS> sep observe
use_fake_img4 = 0, disable_key_pinning = 0
got seprom message 65
start to boot tz0
finished writing key pinning area
seprom booted tz0: 1
got seprom message 65
Using art read from device tree
NVRAM encoded: PA=0x803770e3d, len=0x138
Decoded ART: VA=0x47c2b0000, PA=0x87c2b0000, len=0x129
seprom loaded art: 1
sepos booted: 1
number_of_change = 5
0th observed_value=61436c656e72654b, at tick=0
1th observed_value=7a39fd778fbeda38, at tick=57
2th observed_value=ddb990936ff1918c, at tick=117245
3th observed_value=fa61c6851427bfb2, at tick=1012264
4th observed_value=fa377039b592b43c, at tick=1533324
5th observed_value=5b977279f8931ff0, at tick=1573422
```

You should choose a tick between the 2rd and the 3rd.
If that works, `sep pwned with opcode ab` will appear in pongoterm. Now you have code execution on SEP.
![alt text](images/image.png)

On some old device such as iOS 11.4 that I restored to, PongoOS cannot automatically detect your onboard SEPOS firmware. This is because iBEC before iOS 13 only allocates the SEP firmware buffer, but they doesn't copy the image to this area. You may need to enter the Ramdisk to extract your on board SEPOS firmware. This applies to all supported A8 devices (iPad mini 4, iPhone 6, iPod touch 6).

5. Send SEP payload. Type `/send /absolute/path/to/payload.bin`, then `sep load`. 

6. Jump to the entrypoint of payload. Run `sep jump 1000`. 

7. You should recieve sepmessage ad. At that time you can run `sep peek`, from address `0x10140000` to `0x1014001c`. Keep these 32 bytes in mind. 
![alt text](images/image2.png)

## Part II Modify ART Stored in Device NVRAM
1. Clone SSHRD, create a ramdisk for iOS 15.8.8, and ask your agent to remove the permission bits `p.Bits.KernelOnly` of NVRAM key `com.apple.System.sep.art` and `com.apple.System.boot-nonce` in your kernelcache.

2. Boot into Ramdisk with SSHRD, and start a ssh connection.

3. Type `nvram com.apple.System.boot-nonce=0x1111111111111111` (Change 0x1111111111111111 to the generator in your SHSH)

4. Type `/usr/libexex/seputil --art get` to see your current ART. You should get something like this:
```
308201250201003081fd0203077edf0414904b6ec3ccb67cd7247c37cd29c085ab543067b4041452e2b2fd0f62a6de88253d3c25ce39ec02fd9ad204140341dd2599d16700c11758b6934402b78b51bb1e3181b3c00305725dc103057002c203056fe4c30305703bc403056fe7c703056fedc8060080684960e7c9060098fa2ba607ca0563b5181d6acb0569f9254143cc06008f61a2101ccd0600ced9445c3fce055925e4e3c2cf0600af36b6f282d0054b5371ef94d10600c55288ba42d2053adc85c552d30600eb9111dcdbd40600cedea96cead50600b19bba4c32d605680ec2e73ed70571d05b345dd80600d27b0fddb0d90600dd5dbbdde0da0600a8bb0220ffdb02070c04207a702a127b6f73bf3377e4de55d2cf2c6b6db9eed3de9a6ec660770638b800ba
Successfully parsed ART:
counter: 491231
manifest hash (20 bytes): 904b6ec3ccb67cd7247c37cd29c085ab543067b4
sleep hash (20 bytes): 52e2b2fd0f62a6de88253d3c25ce39ec02fd9ad2
restore nonce (20 bytes): 0341dd2599d16700c11758b6934402b78b51bb1e
subcounters[27]: 1804
subcounters[26]: 724691984639
subcounters[25]: 950760365536
subcounters[24]: 904007769520
subcounters[23]: 488826942557
subcounters[22]: 446924252990
subcounters[21]: 762821889074
subcounters[20]: 888498908394
subcounters[19]: 1011751181531
subcounters[18]: 252807857490
subcounters[17]: 847493249602
subcounters[16]: 323522523028
subcounters[15]: 752537236098
subcounters[14]: 382887846850
subcounters[13]: 888408398911
subcounters[12]: 615818334236
subcounters[11]: 455151534403
subcounters[10]: 428240018794
subcounters[9]: 657032193543
subcounters[8]: 551505453287
subcounters[7]: 356333
subcounters[4]: 356327
subcounters[3]: 356411
subcounters[2]: 356324
subcounters[1]: 356354
subcounters[0]: 356957
```
Create a backup for it in the case of failing.

5. This is your art in hex format, which is encoded in ASN.1 DER. You can use `python scripts/patch_art.py --shsh path/to/shsh --art-hex 123456 --art-key abcdef` to patch it. However this only tested with my ARTs, so if this doesn't work for you, please change it manually. You should only change the manifest hash to your digest hash, and the restore nonce to the SNON in you SHSH. There is a MAC in the back of ART. You should recalculate it with format `mac32 = HMAC-SHA256(Key_ART, body_DER)`. For both way you should verify that the patched DER is legal using `openssl`.

6. Set `com.apple.System.sep.art` to the patched version. Use `nvram "com.apple.System.sep.art=%11%22%33%ef"`. Don't forget the first `%`!
   
7. Save your change with `nvram -s`.

8. Reboot your device into DFU. Run `irecovery -q` to confirm if your APNonce is correct.

## Part III Restore Your Device

### Devices without baseband (iPad mini 4 Wi-Fi, iPod touch 6)
1. Build [this](https://github.com/feiyehua/idevicerestore.git) version of idevicerestore. It's only a simple modification that prevent idevice restore from trying to reboot the device before restoring, and request new APTicket from TSS. And provide your shsh. You should have something like `src/idevicerestore --erase -T "shsh_a142d549489ffeab9a38d3fe285decdf63e6e1e9.der" /path/to/ipsw.ipsw`. 

2. Wait for the restore to complete and enjoy your legacy system!

### Devices with baseband (iPhone 6, iPad mini 4 Cellular)
1. Build a patched version of [futurerestore](https://github.com/futurerestore/futurerestore) that supports custom APTickets and does not request new tickets from TSS.

2. You need a compatible baseband firmware. Options:
   - Use the latest signed baseband (futurerestore can download it automatically with `--latest-baseband`)
   - Supply a specific baseband with `--baseband /path/to/baseband.bbfw --baseband-manifest /path/to/BuildManifest.plist`

3. Run futurerestore with your SHSH and IPSW:
   ```
   futurerestore -t shsh.shsh2 --latest-baseband /path/to/ipsw.ipsw
   ```
   Note: You may need to modify futurerestore to skip the TSS APTicket request (similar to the idevicerestore modification) or use a fork that supports this.

4. Wait for the restore to complete and enjoy your legacy system!

### Firmware paths by device

| Component | iPad mini 4 (j96ap) | iPhone 6 (n61ap) | iPhone 6 Plus (n56ap) | iPod touch 6 (n102ap) | iPad Air 2 Wi-Fi (j81ap) | iPad Air 2 Cell (j82ap) |
|-----------|---------------------|-------------------|-----------------------|-----------------------|--------------------------|--------------------------|
| iBEC | iBEC.ipad5.RELEASE.im4p | iBEC.n61.RELEASE.im4p | iBEC.n56.RELEASE.im4p | iBEC.n102.RELEASE.im4p | iBEC.j81.RELEASE.im4p | iBEC.j82.RELEASE.im4p |
| DeviceTree | DeviceTree.j96ap.im4p | DeviceTree.n61ap.im4p | DeviceTree.n56ap.im4p | DeviceTree.n102ap.im4p | DeviceTree.j81ap.im4p | DeviceTree.j82ap.im4p |
| SEP firmware | sep-firmware.j96.RELEASE.im4p | sep-firmware.n61.RELEASE.im4p | sep-firmware.n56.RELEASE.im4p | sep-firmware.n102.RELEASE.im4p | sep-firmware.j81.RELEASE.im4p | sep-firmware.j82.RELEASE.im4p |
| KernelCache | kernelcache.release.ipad5 | kernelcache.release.iphone7 | kernelcache.release.iphone7 | kernelcache.release.ipod7 | kernelcache.release.ipad5 | kernelcache.release.ipad5 |

## Blobless Boot Guide

The blobless approach requires no saved SHSH blobs. It uses the Blackbird exploit on every boot to patch SEPOS manifest hash validation in-memory via the `sep blobless` PongoOS command. ART is never modified, so there is zero risk of bricking from a bad ART write.

### How Blobless Boot Works

Instead of patching ART with a correct manifest hash from your SHSH blob (Part II of the untethered flow), the blobless approach exploits Blackbird on every boot and uses `reload_sepi()` to decrypt and reload SEPOS into SEP memory. During reload, the `sepfw_blobless_kpf()` kernel patch function NOPs the conditional branches following manifest hash comparisons in SEPOS, so SEP accepts any firmware manifest regardless of what ART says. This is inherently tethered — the patch lives only in RAM and is lost on reboot.

### Prerequisites
- A computer with palera1n or checkra1n
- [atera1n-PongoOS](https://github.com/feiyehua/atera1n-PongoOS) built with blobless support
- Captured Blackbird shellcode (from `sep shc`)
- A valid SEPOS firmware image for the target iOS version (the `sepi` from the IPSW)

### Blobless Boot Steps

1. **First-time setup (capture shellcode, done once per device):**
   1. Put device in DFU mode, boot PongoOS via palera1n/checkra1n.
   2. In pongoterm, run `sep shc` to capture the Blackbird shellcode.
   3. Save the output bytes as a binary file (use `scripts/hex_to_bin.py`). Should be 448 bytes.

2. **Restore (done once):**
   - For devices without baseband (iPad mini 4 Wi-Fi, iPod touch 6, iPad Air 2 Wi-Fi): use idevicerestore.
   - For devices with baseband (iPhone 6, iPhone 6 Plus, iPad mini 4 Cellular, iPad Air 2 Cellular): use futurerestore with `--latest-baseband`.
   - The device will not boot on its own after restore. This is expected.

3. **Every boot:**
   1. Connect device to computer and enter DFU mode.
   2. Boot PongoOS via palera1n/checkra1n.
   3. In pongoterm, send the captured shellcode and the target SEPOS firmware:
      ```
      /send /path/to/captured_shellcode.bin
      sep blobless 470000
      ```
      The `sep blobless` command handles the entire flow: Blackbird exploit, SEPOS decryption, manifest hash patching, and boot.
   4. The device boots into the downgraded iOS version.

### Advantages of Blobless Boot
- **No SHSH blobs required**: Downgrade even without saved blobs
- **No ART modification**: Zero risk of bricking from a bad ART write
- **Reversible**: Reboot normally to return to stock firmware
- **Safe for testing**: Try a downgrade before committing to untethered

### Disadvantages of Blobless Boot
- **Requires a computer for every boot**: If the device reboots (crash, battery dies), it needs the blobless boot process again
- **Slower boot**: Each boot requires DFU mode and running the full exploit chain

## PongoOS Fork Adaptation
See the [PongoOS Device Support](PongoOS_Device_Support.md) file for details on the atera1n-PongoOS fork changes for multi-device support and the blobless boot implementation.

## How It Works
I'm planning to release a writeup to explain how it works.
See the [How It Works](How_it_works.md) file.

## Credits

- **checkm8**, created by [axi0mX](https://github.com/axi0mX).
- **blackbird**, discovered independently by Hao Xu (windknown, Pangu Team) and littlelailo.
- **PongoOS** was developed by the checkra1n team ([upstream project](https://github.com/checkra1n/PongoOS)).
- **Apple ROM Collection**, for providing BootROM binaries. ([Apple ROMs](https://securerom.fun))
- **The Apple Wiki**, for providing firmware keys. ([The Apple Wiki](https://theapplewiki.com/wiki/Main_Page))
- **SSHRD_Script** was created by Nathan ([verygenericname](https://github.com/verygenericname)).
- **turdus_merula**, developed by Clarity, kok3shidoll, and Mineek ([official credits](https://sep.lol/#credits)).
- **idevicerestore**, developed by Joshua Hill, Martin Szulecki, and Nikias Bassen ([upstream authors](https://github.com/libimobiledevice/idevicerestore/blob/master/AUTHORS)).
- **futurerestore**, for baseband-aware restores on cellular devices ([futurerestore](https://github.com/futurerestore/futurerestore)).
- **bb33bb**, providing snippets for A8 blackbird exploit ([Gist](https://gist.github.com/bb33bb/856f56d4d4d0cad57f03a035034a9e7a)).
- **ChatGPT**, for reverse enginnering SEPROM, SEPOS and the restore process for me.
- **DeepSeek**, for writing exploits when ChatGPT rejects me.
- **ByteDance**, for sponsoring (part of) the tokens as "AI Research and Study".
- **Apple**, for giving people wonderful tools for us to do wonderful things.

## Sponsor
Please use Binance Smart Chain:

USDT or USDC
```
0x0257b6c939f9dd8db61a92e8d54ff26870cf9ca3
```

## License

Copyright (C) 2026 feiyehua

This repository, including its shell code, exploit code, README files, documentation, configuration files, and other original materials, is licensed under the GNU General Public License, version 2.0 only (GPL-2.0-only), unless otherwise explicitly stated.

See the [LICENSE](LICENSE) file for the full license text.
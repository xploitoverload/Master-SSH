# Full eMMC Clone & Flash Over Ethernet (SSH)
### A Field Manual for People Who Can’t Afford to Be Wrong

---
## Hello, Friend.

You’re connected to a machine you’re not supposed to touch.

No USB.  
No SD card.  
No vendor flashing tool wrapped in a GUI lie.

Just Ethernet.  
Just SSH.  
Just you and a block device that doesn’t forgive mistakes.

This system boots from eMMC.  
Soldered. Permanent. Quietly waiting for you to screw up.

You need to clone it.  
Or replace it.  
Or resurrect another one exactly like it.

This is how you do that —  
without bricking hardware,  
without flying to a site,  
without guessing.

---

## Know Your Target (Before It Knows You)

Run `lsblk`. Read it twice.

```

/dev/mmcblk2        ← this is the one that matters
├─ mmcblk2p1        boot
├─ mmcblk2p2        rootfs
├─ mmcblk2boot0     bootloader (protected)
└─ mmcblk2boot1     backup bootloader (also protected)
```


If you confuse `mmcblk0` with `mmcblk2`, you deserve what happens next.

### Rules You Do Not Break

- Target eMMC **must be equal or larger**
- Hardware **must be identical**
- Power **must not drop**
- SSH **must already work**
- Hope **is not a strategy**

---

## Part I — Cloning Reality (Bit for Bit)

This is the part where you stop thinking in filesystems  
and start thinking in **sectors**.

### Pull the Disk. All of It.

From your machine. Not the target.

```bash
ssh root@DEVICE "dd if=/dev/mmcblk2 bs=4M status=progress" \
  | zstd -T0 -19 > mmcblk2_full_$(date +%Y%m%d).img.zst
```

You’re not backing up files.

You’re stealing a **perfect memory**.

Empty space.
Deleted data.
Partition table.
Everything.

Bootloaders don’t live in `/boot`.
They live where nobody looks.

So grab those too.

```bash
ssh root@DEVICE "dd if=/dev/mmcblk2boot0 bs=1M" > boot0.img
ssh root@DEVICE "dd if=/dev/mmcblk2boot1 bs=1M" > boot1.img
```

Label them.

Future-you is unreliable.

---

## Part II — Writing Over Someone Else’s Reality

This is where devices die.

If the target matters, read this sentence again.

### Flash the Main Disk

```bash
zstd -dc mmcblk2_full.img.zst \
  | ssh root@TARGET "dd of=/dev/mmcblk2 bs=4M status=progress && sync"
```

If you interrupt this,
don’t pretend it was an accident.

### Boot Partitions Are Lying to You

Linux says they’re read-only.
Linux is trying to protect you from yourself.

Override it.

```bash
echo 0 > /sys/block/mmcblk2boot0/force_ro
echo 0 > /sys/block/mmcblk2boot1/force_ro
```

Now write.

```bash
dd if=boot0.img of=/dev/mmcblk2boot0 bs=1M
dd if=boot1.img of=/dev/mmcblk2boot1 bs=1M
sync
```

Lock them again.

You’re not reckless.
You’re deliberate.

---

## Reboot

This is the longest minute of your life.

```bash
reboot
```

If it comes back,
the system believes it’s the original.

That’s a problem.

---

## Identity Is a Bug

Clones sharing identity cause chaos.

Fix it.

```bash
rm -f /etc/machine-id
systemd-machine-id-setup
hostnamectl set-hostname clone-01
reboot
```

Now it’s unique.
Now it won’t poison the network.

---

## Part III — Flashing New Images (Factory Clean Lies)

Sometimes you don’t want a clone.
You want **obliteration**.

A `.wic` image doesn’t care about your intentions.

Verify it.

```bash
zstd -t image.wic.zst
zstd -l image.wic.zst
```

Check the disk.

```bash
blockdev --getsize64 /dev/mmcblk2
```

If the image is larger than the device,
stop pretending this will work.

Flash it.

```bash
zstd -dc image.wic.zst \
  | ssh root@DEVICE "dd of=/dev/mmcblk2 bs=4M status=progress && sync"
```

No rollback.
No undo.
No excuses.

---

## When the Network Betrays You

It always does.

So don’t tie success to your SSH session.

### screen

```bash
screen -S flash
zstd -dc image.wic.zst | dd of=/dev/mmcblk2 bs=4M
```

Detach.
Walk away.
Let it finish.

### nohup

```bash
nohup sh -c 'zstd -dc /tmp/image.wic.zst | dd of=/dev/mmcblk2 bs=4M > /tmp/flash.log 2>&1 && sync' &
```

Check the damage later.

```bash
tail -f /tmp/flash.log
```

---

## Partial Surgery (Only If You Know What You’re Doing)

### Kernel Only

```bash
scp zImage root@DEVICE:/boot/
sync
reboot
```

### Root Filesystem Only

```bash
zstd -dc rootfs.ext4.zst \
  | ssh root@DEVICE "dd of=/dev/mmcblk2p2 bs=4M status=progress && sync"
```

### Bootloader Only

```bash
echo 0 > /sys/block/mmcblk2boot0/force_ro
dd if=u-boot.bin of=/dev/mmcblk2boot0 bs=1K seek=1
sync
echo 1 > /sys/block/mmcblk2boot0/force_ro
```

Offsets matter.
Wrong offset equals silence.

---

## When Things Go Wrong (They Will)

### Device Won’t Boot

Ask yourself:

* Wrong device?
* Wrong image?
* Power drop?
* Bootloader mismatch?

If SSH is gone,
only serial tells the truth.

### “Operation not permitted”

You forgot:

```bash
echo 0 > /sys/block/mmcblk2boot0/force_ro
```

You always forget.

---

## Final Rules (Read These Like a Contract)

1. `dd` does not ask questions
2. Power loss equals brick
3. Network drops are predictable
4. Serial access saves lives
5. Backups are not optional
6. Vendor tools are training wheels
7. You own the offset
8. Test on hardware you can afford to lose

---

## What This Makes You Capable Of

* Remote provisioning
* Zero-touch replacement
* CI-driven firmware rollout
* Disaster recovery without travel
* Owning your hardware

---

## What It Does Not Forgive

* Guessing device paths
* Skipping `sync`
* Flashing the wrong SoC
* Doing this in production “just once”

---

## Remember

The machine will do exactly what you tell it to do.

Nothing more.
Nothing less.

And if it dies?

It wasn’t the eMMC.

It was you.

---

**— Kalpesh**
*Embedded Linux / eMMC / SSH*
*January 2026*



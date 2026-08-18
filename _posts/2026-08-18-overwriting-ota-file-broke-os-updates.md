---
layout: post
title: 'How Overwriting the Same OTA File Broke Our Android OS Updates'
date: '2026-08-18 23:05:00 +1000'
categories: android
---

> **TL;DR:** Some of our Android 11 devices started failing OS updates after a few successful ones, with `Failed to set pin_file for f2fs: ... Try again`. On F2FS, Android pins the OTA file before an update so garbage collection can't relocate its blocks, and the filesystem counts every block GC couldn't move in an `i_gc_failures` counter on the file's node. Our updater overwrote the same `os_ota_update.zip` in place every release instead of deleting it first, so the file kept the same node and that count never reset, until it passed F2FS's limit of 2048 and the kernel refused to pin the file at all. Deleting the file before each download gives it a fresh node with the counter back at zero.

We hit an interesting issue where some of our custom-built Android 11 devices were failing to perform an OS update after a few successful runs. They update themselves by downloading the zip file from our server to `os_ota_update.zip` in the app's `filesDir`, and executing:

```kotlin
RecoverySystem.installPackage(appContext, updateFile)
```

Under the hood, that API call runs a program called `uncrypt`, which processes the `os_ota_update.zip` file on the filesystem to produce a block map, which is just a list of the physical blocks that the file occupies, so that the file contents can be read directly from the block device without mounting the filesystem.

`uncrypt` was failing on some of these devices with:

```text
10-15 15:21:52.849  3328  3328 E uncrypt : Failed to set pin_file for f2fs: /data/user/0/<our ota updater package name>/files/os_ota_update.zip on /dev/block/by-name/userdata: Try again
10-15 15:21:52.850   575  3327 E RecoverySystemService: uncrypt failed with status: -1
10-15 15:21:52.851   575  3327 E ShutdownThread: Error uncrypting file
```
{: .wrap}

But before we try to understand the error message, we need to know a bit about F2FS (Flash-Friendly File System), which our devices use. Everything below describes F2FS as it behaves on the 4.19 Linux kernel our devices run; some of these details have changed in later kernels.

F2FS stores a file's contents in fixed 4 KB "data blocks" spread across the disk and keeps track of where they are in a separate "node block" that holds the list of addresses along with the file's metadata.

When a user modifies the file, F2FS doesn't overwrite anything in place. It writes the new contents to a fresh block, writes an updated copy of the node block that records the new address, and marks both the old data block and the old node block as garbage for later cleanup. Those familiar with ZFS or Btrfs will recognise the write behaviour as copy-on-write, though those filesystems apply the idea differently.

F2FS groups blocks into larger units called segments (1 segment = 512 blocks). If every block in a segment dies, the segment becomes reusable on its own. F2FS also has a feature called "garbage collection" for the case where a segment is mostly dead but still has a few valid blocks. It roughly works as follows:

1. Pick a victim segment where most blocks are already invalid.
2. Copy the valid blocks out to the current write position and update the node blocks to point at the new copies. The originals are now invalid too.
3. With nothing left valid in it, the victim returns to the free pool.

You've probably guessed by now why a GC run after `uncrypt` would be detrimental! `uncrypt` has already produced a block map file recording the physical locations of the `os_ota_update.zip` data blocks, and we wouldn't want those blocks moving out from under it. To avoid this, `uncrypt` pins the file before it builds the map, telling F2FS that this file's data blocks aren't to be relocated during GC. That pin is a single ioctl, and if it fails, `uncrypt` gives up:

```cpp
int error = ioctl(fd, F2FS_IOC_SET_PIN_FILE, &set);
// Don't break the old kernels which don't support it.
if (error && errno != ENOTTY && errno != ENOTSUP) {
    PLOG(ERROR) << "Failed to set pin_file for f2fs: " << path << " on " << blk_dev;
    return kUncryptIoctlError;
}
```

Since GC can't move the pinned data blocks of `os_ota_update.zip`, it bumps a "failure counter" called `i_gc_failures` once for every still-valid block of the pinned file it finds in a victim segment, and leaves the block in place. This counter lives in the file's node block and is written to disk, so it survives reboots. But the pin only ever exists in memory, so it disappears when the device restarts. That means GC only collides with the file during an update, while `uncrypt` has it pinned.

Nothing in that flow ever explicitly unpins the file, though, and an explicit unpin is the only thing that resets the counter to zero. So each update leaves its tally behind on the node, and the next one picks up where the last one left off.

Once it passes 2048, F2FS refuses to pin the file, and from then on any attempt to pin it fails with `EAGAIN`, which `uncrypt` prints as the `Try again` at the end of that error message. That's what we were observing on our devices.

Whenever we released an OS update, the app responsible for downloading and installing it was simply overwriting the previous `os_ota_update.zip`. Overwriting keeps the same node, so the `i_gc_failures` stored on that node carries over. Every "fresh" download inherited the old counter, which is why it kept climbing across releases until it eventually crossed 2048.

That is also why only some of our devices were affected. The devices that had been in the field longest reached the limit first.

The fix was surprisingly simple. All we had to do was delete the previous `os_ota_update.zip` before downloading the new one, so the next download gets a fresh node with the counter back at zero!

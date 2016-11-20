---
layout: post
title: "Incremental synchronization of ZIP files in bandwidth-constrained environments"
date: 2015-09-13 18:57
comments: true
categories: ["command-line", linux, deployment]
---

Suppose you have two ZIP archives on two different machines and you want to synchronize their contents with minimal network traffic, without much regard for the CPU usage, and you know that the target machine already contains the older version of the archive, which is mostly identical to the new version. For example, you have to upload a fat JAR with the new build over a metered connection, and the server literally sits in a garage at the other end of the country.

If the archive is small, uploading it in its entirety looks simple enough, but if it grows into dozens of megabytes, of which 90% is non-changing 3rd party code, it quickly becomes a waste of both time and bandwidth.

Time is money. And depending on your ISP, bandwidth can be money too.

Long story short, I'm going to assume that the target machine runs Linux, and the source machine runs either Windows or Linux. I am going to use `rsync` and `fuse-zip`.

Why those two? Because `fuse-zip` can convert a ZIP archive into a writeable filesystem, and `rsync` can sync a writeable filesystem with another. Details below.

<!-- more -->

### Task 1 – Mount the archives

Create a mountpoint on the target machine, I'll use `/tmp/z`. Then mount your archive using `fuse-zip`:

```bash
$ sudo fuse-zip -o allow_other,rw,uid=`id -u`,gid=`id -g` /stuff/target.zip /tmp/z
```

To test if it works, you create a file in the mounted filesystem. It would appear in the ZIP file after you unmount it.

```bash
$ echo TEST > /tmp/z/TEST_FILE_PLEASE_IGNORE
$ sudo umount /tmp/z
$ unzip -l /stuff/target.zip | grep TEST_FILE_PLEASE_IGNORE
```

<p class="info warning" data-title="Warning" markdown="1">
Do not modify the ZIP archive while it's being mounted and do not interrupt the unmounting process if the archive was mounted for writing, as this can corrupt it.
</p>

If your source machine runs Linux, you can do the same, but mount the archive read-only:

```bash
$ sudo fuse-zip -o allow_other,ro,uid=`id -u`,gid=`id -g` /stuff/source.zip /tmp/z
```

You should experiment a bit if `sudo` and `usd`/`gid` are necessary.

If your souce machine runs Windows, you can use [Pismo File Mount](http://www.pismotechnic.com/pfm/) to mount the archive as a drive and assign it a letter. It's a GUI program, I haven't found a scriptable alternative yet.

### Task 2 – Synchronize the mounted filesystems

On the source machine:

```
$ rsync --delete -zrucv -e ssh /tmp/z/ remoteuser@targethost:/tmp/z/
```

Explanation for the parameters:

* `-c` so that the files will be compared using checksums instead of modification dates. We do not care about nor trust  the dates here, what matters is content.

* `-z` for the compression. Do not use the flag if the files themselves won't compress well.

* `-r` for recursive synchronization.

* `-u` so only modified files are uploaded.

* `--delete` so the files that are absent in the source archive will be deleted from the target archive. We want the target archive to have the same content as the source one, not more.

* `-v` stands for verbose. Used once, makes `rsync` list all the files as they are being sent. You can skip this option, or repeat it several times, depending on your preference.

* `-e ssh` picks the shell used to synchronize the files. SSH is the recommended one for most cases.

Note that the paths end with slashes, so `rsync` synchronizes the contents of the directories, not the directories themselves.

If the source mashine is running Windows, you can use [Cygwin]()https://www.cygwin.com/, just install appropriate packages (you'll need `rsync` and most likely also `openssh`). If you used Pismo File Mount to mount the ZIP to e.g. J: drive, use `/cygdrive/j/` as your source directory.

### Task 3 – Unmount the filesystems

On Linux (both source and target machine), do

```bash
$ sudo umount /tmp/z
```

You have to unmount the target archive so the changes take place, and the source archive so the next time you mount it, it would be current again.

### Putting it all together

Here is a small script that can be used to synchronize two ZIP archives:

```bash
#!/bin/bash

# put all the necessary values here
REMOTEUSER=...
REMOTEHOST=...
SOURCEZIP=...
TARGETZIP=...

OS=`uname -o`
if [ "$OS" = Cygwin ] ; then
	# depends on where you mount it; assuming here the J: drive
	P=/cygdrive/j
else
	P=/tmp/z
	mkdir -p "$P"
	sudo umount "$P"
	sudo fuse-zip -o allow_other,uid=`id -u`,gid=`id -g`,ro "$SOURCEZIP" "$P"
fi

ssh $REMOTEUSER@$REMOTEHOST sudo jar_mount_rw
rsync --delete -zrucv -e "ssh" "$P/" $REMOTEUSER@$REMOTEHOST:/tmp/z/
ssh $REMOTEUSER@$REMOTEHOST sudo umount /tmp/z

if [ "$OS" = Cygwin ] ; then
	echo 'Finished. Unmount the J: drive.'
else
	PID="`pgrep fuse-zip`"
	if [ -z "$PID" ] ; then
		exit
	else
		sudo umount "$P"
	fi
fi
```

It requires this script to be present as `jar_mount_rw` on the target machine:

```bash
#!/bin/bash

# put all the necessary values here
TARGETZIP=...

umount /tmp/z
mkdir -p /tmp/z
fuse-zip -o allow_other,rw,uid=`id -u`,gid=`id -g` "$TARGETZIP" /tmp/z
```

### Results

I don't have any hard data, but the overall effect is that a 40 MB JAR archive with only few files modified in it gets synchronized in few seconds, while it could take several minutes on slower connections to upload it whole.
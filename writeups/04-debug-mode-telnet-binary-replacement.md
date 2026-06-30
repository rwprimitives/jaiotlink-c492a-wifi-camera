# Part 4: Debug Mode, Telnet, and SD-Card Binary Replacement

This part is more “production firmware hardening” than the main remote command injection. It was still worth documenting because it explained how I got such useful runtime evidence.

The firmware includes a debug workflow controlled by files on the SD card. When enabled, it can:

- redirect `anyka_ipc` logs to `/mnt/tf/`,
- capture kernel logs,
- configure core dumps,
- bind-mount an SD-card copy of `anyka_ipc` over `/usr/bin/anyka_ipc`,
- and expose Telnet during debugging.

That is great for reverse engineering. It is not great to leave in production firmware.

## The debug switch

The startup script defines the SD-card mount point and a few debug files:

```sh
DEBUG_INI_PATH="/mnt/tf/debug.ini"
PDTEST_INI_PATH="/mnt/tf/production_test.ini"
MOUNT_DIR=/mnt/tf
MMC_DEV=/dev/mmcblk0
MMC_FS=/dev/mmcblk0p1
LOG_REDIRECT=0
```

The presence of this file enables debug behavior:

```text
/mnt/tf/debug.ini
```

The script checks for it like this:

```sh
if [ -f "$DEBUG_INI_PATH" ];then
    ...
fi
```

## Replacing `anyka_ipc` from the SD card

The most interesting debug behavior was this block:

```sh
if [ -f "$DEBUG_INI_PATH" ];then

    if [ -e $MOUNT_DIR/anyka_ipc_nostrip ];then
        if [ ! -e $MOUNT_DIR/do_not_debug.ini ];then
            mount --bind $MOUNT_DIR/anyka_ipc_nostrip /usr/bin/anyka_ipc
            ulimit -c unlimited
            ulimit -a
            echo "/mnt/tf/core/%e_%t.core" > /proc/sys/kernel/core_pattern
        fi
    fi

    ...
fi
```

If `/mnt/tf/debug.ini` exists and `/mnt/tf/anyka_ipc_nostrip` exists, the script bind-mounts the SD-card copy over the production binary:

```sh
mount --bind /mnt/tf/anyka_ipc_nostrip /usr/bin/anyka_ipc
```

There is also a bypass file:

```text
/mnt/tf/do_not_debug.ini
```

If that exists, the binary replacement is skipped.

This looks intentional. It is probably a developer or field-debug workflow. The problem is that production firmware should not let removable storage decide which service binary runs.

## Core dumps and logs

The same debug path enables core dumps:

```sh
ulimit -c unlimited
ulimit -a
echo "/mnt/tf/core/%e_%t.core" > /proc/sys/kernel/core_pattern
```

It also redirects logs to the SD card. The script tracks reboot count with:

```text
/mnt/tf/.reboot
```

It captures kernel messages:

```sh
cat /proc/kmsg >> $MOUNT_DIR/kmsg.$REBOOTTIMES.log & 2>&1
```

And it starts `anyka_ipc` like this:

```sh
anyka_ipc >> "$MOUNT_DIR/$LOG_FILE" 2>&1
```

That produced files like:

```text
ipc_700101_000010.01.log
ipc_700101_000010.02.log
ipc_700101_000010.03.log

kmsg.01.log
kmsg.02.log
kmsg.03.log
```

Those logs were the reason the command injection was so easy to prove. They showed the exact `[WIFISELF_System]` command string the firmware was about to execute.

## Practical debug setup

This was the setup I used during research:

```sh
mkdir -p /mnt/tf/core
touch /mnt/tf/debug.ini
cp /usr/bin/anyka_ipc /mnt/tf/anyka_ipc_nostrip
chmod 755 /mnt/tf/anyka_ipc_nostrip
sync
reboot
```

After reboot, the script handled the bind mount, core pattern, and logging.

## Telnet during debug mode

After debug mode was enabled, the device opened Telnet on port `23`.

The extracted firmware contained this `/etc/passwd` entry:

```text
root:ABgia2Z.lfFhA:0:0:root:/:/bin/sh
```

The corresponding password I found during testing was:

```text
j1/_7sxw
```

Telnet access looked like:

```bash
telnet 172.14.10.1
```

After logging in, `netstat` confirmed the important services:

```text
tcp  0  0 0.0.0.0:10000  0.0.0.0:*  LISTEN  658/IOTDaemon
tcp  0  0 0.0.0.0:80     0.0.0.0:*  LISTEN  505/anyka_ipc
tcp  0  0 0.0.0.0:23     0.0.0.0:*  LISTEN  566/telnetd
```

Telnet was not the final exploit path. The main command injection worked over HTTP. Telnet was useful because it let me inspect processes, watch logs, and verify what the firmware was doing.

## Why this matters

This is a different issue than the `SetMAC` command injection. The strongest abuse paths require physical SD-card access or another write primitive.

But from a production-firmware perspective, the behavior is still risky:

- Removable storage can influence runtime behavior.
- A replacement service binary can be bind-mounted over the production binary.
- Logs and core dumps can expose sensitive runtime information.
- Telnet becomes reachable in debug mode.
- A recovered root credential existed in the firmware.

For research, this was extremely helpful. For shipping devices, it should be removed or heavily restricted.

## Fix

Production builds should not include this debug path as-is.

Recommended changes:

- Remove SD-card triggered binary replacement from production firmware.
- Do not enable Telnet in production builds.
- Do not store recoverable root credentials in firmware.
- Disable unrestricted core dumps by default.
- Avoid writing sensitive logs to removable storage.
- Require signed debug packages if field diagnostics are truly needed.

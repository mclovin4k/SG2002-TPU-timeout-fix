# TPU Freeze Fix: `soph_tpu.ko`

A patcher for the standard kernel module used on the LicheeRV Nano (SG2002 / CV181x). It eliminates TPU "freezes" that last for up to a minute—situations where the MJPEG stream continues to flow, but the detection bounding boxes remain stuck.

The repository does **not** contain a pre-built `.ko` file; the driver is distributed under the GPL, and its source code is not included in the system image. The patcher simply modifies constants within the module already present on your board, so there is no requirement to redistribute the binary.

## Symptoms

- `dmesg` repeatedly shows: `cvi-tpu c100000.tpu: run dmabuf timeout`.
- The `stream_yolo` log shows: `wait dmabuf failed[-110]` (`-ETIMEDOUT`) and `run cmdbuf failed:2`.
- Inference takes seconds instead of 50–80 ms, causing the detector to output stale bounding boxes.
- The message `no det Ns` appears in the frame, even though the video stream is still active.
- A process stuck in a TPU-related `ioctl` call cannot be terminated with `kill -9` until the driver-level timeout expires; sometimes, a full system reboot is the only solution.

This is not an issue with the model itself; it occurs with various YOLO models and other applications that utilize the TPU.

## Cause

In `soph_tpu.ko`, there are five instances where the code waits for a command to complete (`wait_for_completion_interruptible_timeout`) with a timeout set to **15,000 ticks**. With `HZ=250`, this translates to **60 seconds**. If the TPU fails to send an interrupt—due to a lost `dmabuf-fence` or a scheduler malfunction—the calling thread becomes blocked for an entire minute. A queue of commands builds up behind it; the application can neither receive an error nor recreate the handle.

## What the patch does

Five constants are replaced, setting the timeout to **250 ticks = 1 second**. The driver logic remains unchanged—only the maximum wait time is reduced. Following a failure, `stream_yolo` receives an error after approximately one second, unregisters and re-registers the model, and the stream continues to function.

| Location | Offset | Original | Patched |
|---|---|---|---|
| 1 | 7568 | `91 65 93 85 85 a9` | `81 45 93 85 a5 0f` |
| 2 | 7940 | `91 65 93 85 85 a9` | `81 45 93 85 a5 0f` |
| 3 | 8184 | `91 67 93 87 87 a9` | `81 47 93 87 a7 0f` |
| 4 | 8658 | `91 67 93 87 87 a9` | `81 47 93 87 a7 0f` |
| 5 | 9934 | `91 65 93 85 85 a9` | `81 45 93 85 a5 0f` |

Module checksums (size: 70,536 bytes):

| State | MD5 |
|---|---|
| Original | `bc70f809190c0e41aeb2d3863da4581d` |
| Patched | `3c0545f7a42b233987c9e52ebc56affd` |

The patcher first checks all five sites and the MD5 hash. If the module has a different file size, a different version, or if the bytes do not match, it will refuse to make any changes.

## Installation

Copy the directory to the board and run the patcher (by default, it targets
`/mnt/system/ko/soph_tpu.ko`):

```sh
scp -r tpu-fix root@<IP>:/root/
ssh root@<IP> 'sh /root/tpu-fix/patch_tpu.sh'
ssh root@<IP> reboot
```

If the module is located elsewhere:

```sh
ssh root@<IP> 'sh /root/tpu-fix/patch_tpu.sh /path/to/soph_tpu.ko'
```

The patcher creates a backup file named `soph_tpu.ko.orig` in the same directory as the module. Running the patcher again on an already patched file will result in no changes.

## Verification

```sh
ssh root@<IP> 'md5sum /mnt/system/ko/soph_tpu.ko'
# 3c0545f7a42b233987c9e52ebc56affd

ssh root@<IP> 'lsmod | grep soph_tpu'
ssh root@<IP> 'dmesg | grep -i "soph_tpu\|tpu"'
```

After starting the stream, leave it under load for a few minutes. If a `run dmabuf timeout` occurs, detection should recover within 1–2 seconds, rather than freezing for tens of seconds.

## Rollback

```sh
ssh root@<IP> 'cp /mnt/system/ko/soph_tpu.ko.orig /mnt/system/ko/soph_tpu.ko'
ssh root@<IP> 'sync && reboot'
```

If you do not have a backup, retrieve the original module from a fresh LicheeRV Nano image.

## Compatibility and Limitations

- Tested on the standard LicheeRV Nano image with kernel
`5.10.260-tag-20260927-g3ac35f59d279` (`HZ=250`, module size: 70,536 bytes).
- The patcher will refuse to run on a different kernel or driver version;
do not attempt to bypass this check.
- This is a **mitigation**, not a fix for the root cause within the hardware or driver. 
While `run dmabuf timeout` messages may still appear in `dmesg`, the
minute-long freezes are resolved.
- The patch does not fix camera hangs that occur during closure (`vb_qbuf`,
state `D`). If the process fails to terminate, a `reboot` is required.
- The module is a GPL-licensed component from Cvitek/Sophgo; `strings`
reveals `license=GPL` and identifies the author as `Wellken Chen`. As the
driver source code is not included in the image, we do not distribute the
original or patched `.ko` files—only the patcher itself. If you choose to
distribute a modified module, you must comply with the GPL and provide the
corresponding source code.

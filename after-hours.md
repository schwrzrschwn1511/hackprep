Every command is verified end-to-end. Here is the complete, reproducible walkthrough.

---

# After Hours — Forensics (450) — Complete Step-by-Step Solution

**Final flag:** `HNYX{74ccdaf686bb385d2b54e56b3426bda5}`

**The trick in one sentence:** the filesystem on the image holds only decoys + breadcrumbs; the real flag was written by a fake `auditd` to **raw sectors past the end of the filesystem**, split into 4 named segments and XOR'd with `0x37`.

All commands below assume you start in the directory holding `disk.img.gz`.

---

## Step 1 — Identify and decompress the artifact

```bash
file disk.img.gz
# disk.img.gz: gzip compressed data, ... original size modulo 2^32 134217728 ...

gunzip -k disk.img.gz        # -k keeps the original .gz
file disk.img
# disk.img: data            <-- NOT a recognized filesystem/partition image
ls -l disk.img               # 134217728 bytes (128 MiB)
```

`file` reporting plain `data` is the first symptom of the "damaged" image: there is **no MBR / partition table** at offset 0 (and `fdisk -l` finds nothing). So we can't rely on automatic tools — we profile the raw bytes ourselves.

## Step 2 — Map where real data lives (the image is mostly sparse zeros)

```bash
python3 - <<'PY'
data=open('disk.img','rb').read()
print('image size:',len(data),hex(len(data)))
nz=[i for i in range(len(data)) if data[i]!=0]
print('total non-zero bytes:',len(nz))
ranges=[]; start=nz[0]; prev=nz[0]
for i in nz[1:]:
    if i-prev>4096:           # split into clusters with >4 KiB gaps
        ranges.append((start,prev)); start=i
    prev=i
ranges.append((start,prev))
for a,b in ranges:
    print(f'  {a:#011x} - {b:#011x}  ({b-a+1} bytes)')
PY
```

Output (abridged):
```
image size: 134217728 0x8000000
  0x000100400 - 0x000143fff  (277504 bytes)   <- ext metadata; FS starts at 1 MiB
  ...
  0x006000000 - 0x006000009  (10 bytes)       <- 4 tiny out-of-place blobs near 96 MiB
  0x006100000 - 0x006100009  (10 bytes)
  0x006200000 - 0x006200009  (10 bytes)
  0x006300000 - 0x006300007  ( 8 bytes)
```

Two important observations:
1. A real filesystem starts around **1 MiB** (offset `0x100000`) — the classic first-partition location.
2. **Four tiny blobs near 96 MiB** that look completely out of place. Note these for later.

Confirm the ext superblock magic `0x53EF` (superblock is at +1024 inside the partition, magic at +0x38):

```bash
xxd -s 0x100438 -l 2 disk.img
# 00100438: 53ef        <- ext2/3/4 magic confirmed
```

## Step 3 — Carve the partition and read its geometry

```bash
dd if=disk.img of=part.img bs=1M skip=1 count=100 status=none
file part.img
# part.img: Linux rev 1.0 ext4 filesystem data, UUID=e2c91f57-... (extents)(64bit)...

dumpe2fs -h part.img 2>/dev/null | grep -Ei 'Last mounted|Block size|Block count'
# Last mounted on:  /home/kali/Desktop/HYNX26/mnt
# Block size:       1024
# Block count:      80896
```

Compute the filesystem's end offset — this is the crux:

```bash
python3 -c "bc=80896;bs=1024;print('FS bytes =',bc*bs,hex(bc*bs)); \
print('FS spans disk 0x100000 ->',hex(0x100000+bc*bs)); \
print('blobs at 0x6000000 outside FS:',0x6000000>0x100000+bc*bs)"
# FS bytes = 82837504 0x4f00000
# FS spans disk 0x100000 -> 0x5000000
# blobs at 0x6000000 outside FS: True
```

So the filesystem occupies disk `0x100000`–`0x5000000` (1–80 MiB). **The 4 blobs at `0x6000000`+ live beyond the last filesystem block** — they are not part of the filesystem at all.

## Step 4 — Walk the filesystem with `debugfs` (no mounting needed)

> Loopback mount is normally blocked inside the CTF container. `debugfs` reads ext directly. On a host you could instead: `sudo mount -o ro,loop,offset=1048576 disk.img /mnt`.

```bash
for d in / /etc /root /tmp /var /var/log /opt; do
  echo "--- $d ---"; debugfs -R "ls -l $d" part.img 2>/dev/null
done
```

The tree contains: `/etc/{motd,chat}`, `/root/.ash_history`, `/tmp/.flag`, `/var/log/auth.log`, `/opt/.cache` (empty).

Dump every interesting file:

```bash
for f in /etc/motd /etc/c
... [4918 chars omitted]

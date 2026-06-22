I've written a complete, writeup-ready walkthrough to `/workspace/WRITEUP.md`. Here it is in full.

---

# After Hours — Forensics (450)

**Flag:** `HNYX{74ccdaf686bb385d2b54e56b3426bda5}`

## The core idea
The filesystem on the image contains only **decoys and breadcrumbs**. The real flag was written by a fake `auditd` to **raw disk sectors that live *past the end* of the filesystem**, split into 4 segments and obfuscated with a single‑byte XOR (`0x37`).

---

## Step 1 — Decompress
```bash
cd /workspace/files
cp disk.img.gz disk_work.gz && gunzip -k disk_work.gz   # -> disk_work, 128 MiB
file disk_work        # => "data"  (no MBR — this is the "damaged" hint)
```
`file`/`fdisk` find nothing because the partition table is gone. So we profile the image manually.

## Step 2 — Map the non‑zero regions
The image is almost entirely sparse. Listing non‑zero byte ranges in Python shows:
- `0x100400 …` → an **ext4 superblock + metadata** (FS begins at `0x100000` = 1 MiB; magic `0x53EF` sits at `0x100438`).
- **Four tiny blobs** at `0x6000000`, `0x6100000`, `0x6200000`, `0x6300000` (10/10/10/8 bytes) — wildly out of place. Remember these.

## Step 3 — Identify the filesystem boundary
```bash
dd if=disk_work of=part.img bs=1M skip=1 count=100 status=none
dumpe2fs -h part.img      # ext4, block size 1024, block count 80896
```
`80896 × 1024 = 0x4F00000`, so the FS spans disk offsets **`0x100000` → `0x5000000`**. Everything at `0x6000000` is **outside the filesystem**.

## Step 4 — Walk the FS with `debugfs` (mount is blocked in-container)
```bash
debugfs -R "ls -l /" part.img
for f in /etc/motd /etc/chat /tmp/.flag /root/.ash_history /var/log/auth.log; do
  echo "== $f =="; debugfs -R "cat $f" part.img; done
```
Findings:
| File | Content | Role |
|---|---|---|
| `/etc/motd` | `HNYX{this_is_not_the_real_flag}` | obvious decoy |
| `/tmp/.flag` | `HNYX{...bda0}` | subtle decoy (off by one nibble!) |
| `/etc/chat` | "…not inside the mounted filesystem." | clue |
| `/root/.ash_history` | `rm /etc/.auditd.tmp` + `history -c` | anti‑forensics |

`debugfs -R "lsdel"` → 0 deleted inodes, confirming the evidence isn't in the FS.

## Step 5 — `auth.log` points to the out‑of‑band segments
```
auditd[1337]: rotated segment s2 at 0x06200000
auditd[1337]: rotated segment s0 at 0x06000000
auditd[1337]: rotated segment s3 at 0x06300000
auditd[1337]: rotated segment s1 at 0x06100000
```
The 4 stray blobs **are** segments `s0..s3`, sitting past the FS end. The log lines are shuffled on purpose — reassemble by **segment name**, not log order. Total size = 10+10+10+8 = **38 bytes** = an `HNYX{32hex}` flag.

## Step 6 — Reassemble (s0→s3) and break the XOR
First cipher bytes `7f 79 6e 6f 4c` vs known prefix `HNYX{` (`48 4e 59 58 7b`) → constant XOR key **`0x37`**.
```python
d=open('disk_work','rb').read()
s=[(0x6000000,10),(0x6100000,10),(0x6200000,10),(0x6300000,8)]
print(bytes(b^0x37 for o,n in s for b in d[o:o+n]).decode())
# HNYX{74ccdaf686bb385d2b54e56b3426bda5}
```

## Flag
```
HNYX{74ccdaf686bb385d2b54e56b3426bda5}
```
It differs from the `/tmp/.flag` decoy only in the last nibble (`bda5` vs `bda0`).

---

### Misdirection tricks worth calling out in the writeup
- **Wiped MBR + sparse image** → `file` says `data`, forcing manual carving.
- **Two decoy flags**, one a deliberate near‑miss to bait early submission.
- **Evidence stored beyond the last FS block** → mount/`debugfs`/`lsdel` won't reveal it.
- **Out‑of‑order log lines** to trick reassembly order (correct order = segment names s0..s3).
- **Single‑byte XOR (0x37)**, trivially broken via the known `HNYX{` prefix.

The full version (with all command output and a copy‑paste reproduction one‑liner) is saved at `/workspace/WRITEUP.md`.

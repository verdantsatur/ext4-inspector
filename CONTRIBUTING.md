#!/usr/bin/env bash
# ============================================================
#  ext4_bridge.sh — Ext4 Visual Inspector: Linux Data Bridge
#  Version 1.0 | GPLv3 | Free/Libre Software
#
#  Scans a mounted partition, USB drive, or folder and exports
#  a JSON file that the browser UI can load and visualize.
#
#  Usage:
#    ./ext4_bridge.sh --folder /home/user/Documents
#    ./ext4_bridge.sh --partition /dev/sdb1
#    ./ext4_bridge.sh --mount /mnt/usb
#
#  Output: ./data/fs_export.json
# ============================================================

set -euo pipefail

# ── Defaults ────────────────────────────────────────────────
MODE=""
TARGET=""
OUTPUT_DIR="$(dirname "$0")/../data"
OUTPUT_FILE="$OUTPUT_DIR/fs_export.json"
MAX_DEPTH=20
MAX_FILES=5000
BLOCK_SIZE=4096
SECTOR_SIZE=512

# ── Colors for terminal output ───────────────────────────────
RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[1;33m'
BLUE='\033[0;34m'; NC='\033[0m'

log()  { echo -e "${BLUE}[INFO]${NC}  $*"; }
ok()   { echo -e "${GREEN}[OK]${NC}    $*"; }
warn() { echo -e "${YELLOW}[WARN]${NC}  $*"; }
err()  { echo -e "${RED}[ERROR]${NC} $*" >&2; exit 1; }

# ── Help ─────────────────────────────────────────────────────
usage() {
cat <<EOF
Ext4 Visual Inspector — Linux Bridge Script
============================================
Scans a real filesystem and exports JSON for the browser UI.

Usage:
  $(basename "$0") [OPTIONS] --folder <path>
  $(basename "$0") [OPTIONS] --partition <device>
  $(basename "$0") [OPTIONS] --mount <mountpoint>

Scan modes:
  --folder <path>        Scan a specific directory (no root needed)
  --partition <device>   Scan a partition e.g. /dev/sdb1 (needs root)
  --mount <mountpoint>   Scan an already-mounted partition e.g. /mnt/usb

Options:
  --output <file>        Output JSON path (default: ./data/fs_export.json)
  --max-depth <n>        Max directory depth (default: 20)
  --max-files <n>        Max files to export (default: 5000)
  --no-hex               Skip hex byte sampling (faster)
  --help                 Show this help

Examples:
  # Scan your Documents folder (no root needed):
  ./ext4_bridge.sh --folder ~/Documents

  # Scan a USB drive (needs root):
  sudo ./ext4_bridge.sh --partition /dev/sdb1

  # Scan an already-mounted USB:
  ./ext4_bridge.sh --mount /mnt/usb

Safety:
  This script is READ-ONLY. It never modifies your data.
  The wiper feature in the UI requires a separate --wipe flag
  which is intentionally NOT included in this script for safety.
EOF
exit 0
}

# ── Argument parsing ─────────────────────────────────────────
SKIP_HEX=false
while [[ $# -gt 0 ]]; do
  case "$1" in
    --folder)    MODE="folder";    TARGET="$2"; shift 2;;
    --partition) MODE="partition"; TARGET="$2"; shift 2;;
    --mount)     MODE="mount";     TARGET="$2"; shift 2;;
    --output)    OUTPUT_FILE="$2"; shift 2;;
    --max-depth) MAX_DEPTH="$2";   shift 2;;
    --max-files) MAX_FILES="$2";   shift 2;;
    --no-hex)    SKIP_HEX=true;    shift;;
    --help|-h)   usage;;
    *) err "Unknown option: $1. Use --help for usage.";;
  esac
done

[[ -z "$MODE" ]] && err "No scan mode specified. Use --folder, --partition, or --mount."
[[ -z "$TARGET" ]] && err "No target specified."

# ── Dependency check ─────────────────────────────────────────
check_deps() {
  local missing=()
  for cmd in stat find xxd python3; do
    command -v "$cmd" &>/dev/null || missing+=("$cmd")
  done
  if [[ "$MODE" == "partition" ]]; then
    for cmd in debugfs blkid; do
      command -v "$cmd" &>/dev/null || missing+=("$cmd")
    done
  fi
  if [[ ${#missing[@]} -gt 0 ]]; then
    err "Missing required tools: ${missing[*]}\nInstall with: sudo apt install e2fsprogs xxd python3"
  fi
}
check_deps

mkdir -p "$OUTPUT_DIR"

# ── Mount partition if needed ─────────────────────────────────
MOUNT_POINT=""
MOUNTED_BY_US=false

if [[ "$MODE" == "partition" ]]; then
  [[ $EUID -ne 0 ]] && err "Scanning a partition requires root. Run with: sudo $0 --partition $TARGET"
  [[ -b "$TARGET" ]] || err "Device $TARGET not found or not a block device."
  MOUNT_POINT="/tmp/ext4_inspector_mount_$$"
  mkdir -p "$MOUNT_POINT"
  log "Mounting $TARGET at $MOUNT_POINT (read-only)..."
  mount -o ro "$TARGET" "$MOUNT_POINT" || err "Failed to mount $TARGET"
  MOUNTED_BY_US=true
  TARGET="$MOUNT_POINT"
elif [[ "$MODE" == "mount" ]]; then
  MOUNT_POINT="$TARGET"
  [[ -d "$MOUNT_POINT" ]] || err "Mount point $MOUNT_POINT does not exist."
else
  [[ -d "$TARGET" ]] || err "Folder $TARGET does not exist."
fi

# Cleanup on exit
cleanup() {
  if [[ "$MOUNTED_BY_US" == true && -n "$MOUNT_POINT" ]]; then
    log "Unmounting $MOUNT_POINT..."
    umount "$MOUNT_POINT" 2>/dev/null || true
    rmdir "$MOUNT_POINT"  2>/dev/null || true
  fi
}
trap cleanup EXIT

# ── Superblock info (partition/mount mode) ────────────────────
get_superblock_json() {
  local device="$1"
  if command -v tune2fs &>/dev/null && [[ -b "$device" ]]; then
    local info
    info=$(tune2fs -l "$device" 2>/dev/null || echo "")
    local magic="0xEF53"
    local uuid=$(echo "$info" | grep "Filesystem UUID" | awk '{print $3}')
    local label=$(echo "$info" | grep "Filesystem volume name" | cut -d: -f2 | xargs)
    local blocks=$(echo "$info" | grep "Block count" | awk '{print $3}')
    local free_blocks=$(echo "$info" | grep "Free blocks" | awk '{print $3}')
    local inodes=$(echo "$info" | grep "Inode count" | awk '{print $3}')
    local free_inodes=$(echo "$info" | grep "Free inodes" | awk '{print $3}')
    local block_size=$(echo "$info" | grep "Block size" | awk '{print $3}')
    local inode_size=$(echo "$info" | grep "Inode size" | awk '{print $3}')
    local features=$(echo "$info" | grep "Filesystem features" | cut -d: -f2 | xargs)
    cat <<SBJSON
{
  "magic": "$magic",
  "uuid": "${uuid:-unknown}",
  "label": "${label:-(no label)}",
  "blockSize": ${block_size:-4096},
  "inodeSize": ${inode_size:-256},
  "totalBlocks": ${blocks:-0},
  "freeBlocks": ${free_blocks:-0},
  "totalInodes": ${inodes:-0},
  "freeInodes": ${free_inodes:-0},
  "features": "${features:-}"
}
SBJSON
  else
    echo '{"magic":"0xEF53","blockSize":4096,"inodeSize":256,"note":"tune2fs not available - folder mode"}'
  fi
}

# ── File hex sampling ─────────────────────────────────────────
get_hex_sample() {
  local filepath="$1"
  local max_bytes=256
  if [[ "$SKIP_HEX" == true ]] || [[ ! -r "$filepath" ]]; then
    echo "[]"
    return
  fi
  # Read first 256 bytes as hex, output as JSON array
  xxd -l $max_bytes -p "$filepath" 2>/dev/null | tr -d '\n' | \
    python3 -c "
import sys, json
h = sys.stdin.read().strip()
arr = [int(h[i:i+2],16) for i in range(0,len(h),2) if i+2<=len(h)]
print(json.dumps(arr))
" 2>/dev/null || echo "[]"
}

# ── Slack space calculation ───────────────────────────────────
get_slack() {
  local size="$1"
  local block_size=${2:-4096}
  if [[ $size -eq 0 ]]; then echo 0; return; fi
  local remainder=$(( size % block_size ))
  if [[ $remainder -eq 0 ]]; then echo 0; else echo $(( block_size - remainder )); fi
}

# ── inode info ────────────────────────────────────────────────
get_inode() {
  local filepath="$1"
  stat -c '%i' "$filepath" 2>/dev/null || echo "0"
}

# ── Build JSON tree (recursive, Python-assisted) ──────────────
log "Scanning $TARGET ..."
log "This may take a moment for large filesystems..."

SCAN_ROOT="$TARGET"
FILE_COUNT=0

# Use Python to build the tree — much faster than pure bash
python3 << PYEOF
import os, json, sys, stat as statmod, time

root = "$SCAN_ROOT"
max_depth = $MAX_DEPTH
max_files = $MAX_FILES
block_size = $BLOCK_SIZE
skip_hex = $([[ "$SKIP_HEX" == true ]] && echo "True" || echo "False")

file_count = [0]
node_id = [0]

EXT_MAP = {
    '.pdf':'pdf','.txt':'txt','.md':'md','.docx':'docx','.doc':'doc',
    '.jpg':'jpg','.jpeg':'jpg','.png':'png','.gif':'gif','.bmp':'bmp',
    '.mp4':'mp4','.mp3':'mp3','.wav':'wav','.avi':'avi','.mkv':'mkv',
    '.zip':'zip','.tar':'tar','.gz':'gz','.bz2':'bz2','.xz':'xz',
    '.py':'py','.sh':'sh','.c':'c','.cpp':'cpp','.js':'js','.html':'html',
    '.css':'css','.json':'json','.xml':'xml','.csv':'csv','.log':'log',
    '.iso':'iso','.img':'img','.db':'db','.sqlite':'sqlite',
}

def get_hex_sample(path, max_bytes=256):
    if skip_hex:
        return []
    try:
        with open(path, 'rb') as f:
            data = f.read(max_bytes)
        return list(data)
    except:
        return []

def slack_bytes(size, bsz=block_size):
    if size == 0: return 0
    r = size % bsz
    return 0 if r == 0 else bsz - r

def format_perms(mode):
    perms = ''
    for who in [(statmod.S_IRUSR,statmod.S_IWUSR,statmod.S_IXUSR),
                (statmod.S_IRGRP,statmod.S_IWGRP,statmod.S_IXGRP),
                (statmod.S_IROTH,statmod.S_IWOTH,statmod.S_IXOTH)]:
        perms += 'r' if mode&who[0] else '-'
        perms += 'w' if mode&who[1] else '-'
        perms += 'x' if mode&who[2] else '-'
    return perms

def walk_dir(path, depth=0, rel_path='/'):
    if depth > max_depth or file_count[0] >= max_files:
        return None
    node_id[0] += 1
    nid = node_id[0]

    try:
        st = os.lstat(path)
    except:
        return None

    is_dir = statmod.S_ISDIR(st.st_mode)
    name   = os.path.basename(path) or '/'
    ext    = EXT_MAP.get(os.path.splitext(name)[1].lower(), '')
    inode  = st.st_ino
    size   = st.st_size
    mtime  = time.strftime('%Y-%m-%d', time.localtime(st.st_mtime))
    mode   = format_perms(st.st_mode)
    uid    = st.st_uid
    gid    = st.st_gid
    nlinks = st.st_nlink

    # Block info
    # stat st_blocks is in 512-byte units
    blocks_512 = st.st_blocks
    blocks_4k  = max(1, (size + block_size - 1) // block_size) if not is_dir else 1
    first_block = (inode * 7 + 1024) % 524288  # educational approximation
    slack = slack_bytes(size)

    node = {
        'id':      nid,
        'type':    'dir' if is_dir else 'file',
        'name':    name,
        'path':    rel_path,
        'inode':   inode,
        'size':    size,
        'ext':     ext,
        'modified':mtime,
        'perms':   mode,
        'uid':     uid,
        'gid':     gid,
        'nlinks':  nlinks,
        'blocks':  [first_block + i for i in range(min(blocks_4k, 256))],
        'blockCount': blocks_4k,
        'slack':   slack,
        'slackPct': round(slack/block_size*100, 1) if not is_dir else 0,
        'hexSample': [] if is_dir else get_hex_sample(path),
    }
    file_count[0] += 1

    if is_dir:
        children = []
        try:
            entries = sorted(os.listdir(path))
        except PermissionError:
            entries = []
        for entry in entries:
            if file_count[0] >= max_files:
                children.append({'id': -1, 'type':'truncated',
                                  'name':'[limit reached: use --max-files to increase]',
                                  'size':0,'inode':0,'blocks':[],'slack':0,'slackPct':0,
                                  'ext':'','modified':'','perms':'','uid':0,'gid':0,'nlinks':0,'blockCount':0,'hexSample':[]})
                break
            child_path = os.path.join(path, entry)
            child_rel  = rel_path.rstrip('/') + '/' + entry
            child = walk_dir(child_path, depth+1, child_rel)
            if child:
                children.append(child)
        node['children'] = children

    return node

print("[ext4_bridge] Building tree...", file=sys.stderr)
tree = walk_dir(root, 0, '/')

superblock = {
    "magic": "0xEF53",
    "blockSize": block_size,
    "inodeSize": 256,
    "note": "Folder scan mode — inode numbers are real, block numbers are educational approximations. Use --partition for full Ext4 data."
}

output = {
    "version": "1.0",
    "scanMode": "$MODE",
    "scanTarget": "$TARGET",
    "scanTime": time.strftime('%Y-%m-%d %H:%M:%S'),
    "blockSize": block_size,
    "sectorSize": $SECTOR_SIZE,
    "fileCount": file_count[0],
    "superblock": superblock,
    "tree": tree
}

with open("$OUTPUT_FILE", 'w') as f:
    json.dump(output, f, indent=2)

print(f"[ext4_bridge] Done. {file_count[0]} items exported to $OUTPUT_FILE", file=sys.stderr)
PYEOF

ok "Export complete: $OUTPUT_FILE"
log "File count: $(python3 -c "import json; d=json.load(open('$OUTPUT_FILE')); print(d['fileCount'])")"
log ""
log "Next step: open the browser UI and load this file:"
log "  cd $(dirname "$0")/.."
log "  python3 -m http.server 8080"
log "  Then open: http://localhost:8080/ui/index.html"
log "  Click 'Load scan data' and select: $OUTPUT_FILE"

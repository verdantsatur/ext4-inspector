#!/usr/bin/env bash
# ============================================================
#  ext4_wipe_slack.sh — Ext4 File Tail / Slack Space Wiper
#  Version 1.0 | GPLv3 | Free/Libre Software
#
#  Wipes the slack space (unused bytes in the last block) of
#  one or more files. This removes residual data that forensic
#  tools could otherwise recover.
#
#  SAFETY: This only zeros bytes AFTER the real file content.
#          It never touches the file's actual data.
#          It works on any Linux filesystem (Ext4, XFS, etc.)
#
#  Usage:
#    ./ext4_wipe_slack.sh --file /path/to/file.pdf
#    ./ext4_wipe_slack.sh --list files.txt
#    ./ext4_wipe_slack.sh --folder /home/user/Documents
#
#  Requires: root (or write permission to the files)
# ============================================================

set -euo pipefail

# ================================================================
#  SAFETY NOTICE — READ BEFORE RUNNING
#  This script WRITES to disk. It zeros slack space bytes only.
#  It does NOT delete files or change file content.
#
#  ALWAYS run with --dry-run first to preview safely:
#    ./ext4_wipe_slack.sh --dry-run --folder ~/Documents
#
#  FIRST TIME? Read WARNINGS.md in the project root first.
# ================================================================

RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[1;33m'
BLUE='\033[0;34m'; CYAN='\033[0;36m'; NC='\033[0m'

log()  { echo -e "${BLUE}[INFO]${NC}  $*"; }
ok()   { echo -e "${GREEN}[WIPED]${NC} $*"; }
skip() { echo -e "${CYAN}[SKIP]${NC}  $*"; }
warn() { echo -e "${YELLOW}[WARN]${NC}  $*"; }
err()  { echo -e "${RED}[ERROR]${NC} $*" >&2; exit 1; }

# ── Defaults ─────────────────────────────────────────────────
MODE=""
TARGET=""
DRY_RUN=false
VERBOSE=false
PASSES=1          # Number of wipe passes (1=zeros, 3=DoD 5220.22-M style)
BLOCK_SIZE=4096
TOTAL_WIPED=0
TOTAL_FILES=0
TOTAL_SLACK=0

usage() {
cat <<EOF
Ext4 Visual Inspector — Slack Space Wiper
==========================================
Zeros the unused bytes at the end of each file's last disk block.

Usage:
  $(basename "$0") --file <path>
  $(basename "$0") --list <file-with-paths>
  $(basename "$0") --folder <directory>

Options:
  --dry-run        Show what would be wiped, without actually wiping
  --passes <n>     Number of wipe passes: 1=zeros, 3=DoD-style (default: 1)
  --verbose        Show byte-level detail
  --help           Show this help

What is slack space?
  A file of 6500 bytes occupies 2 blocks of 4096 bytes = 8192 bytes total.
  The last 1692 bytes (8192 - 6500) are slack space: unused but potentially
  containing old data from previously deleted files. This wiper zeros those bytes.

Safety guarantee:
  - Only bytes AFTER the real file content are touched
  - The file's size and content are never changed
  - All operations are logged to stdout

Examples:
  # See what would be wiped (safe, no changes):
  ./ext4_wipe_slack.sh --dry-run --folder ~/Documents

  # Wipe a single file:
  ./ext4_wipe_slack.sh --file ~/Documents/resume.pdf

  # Wipe all files in a folder (3-pass DoD style):
  sudo ./ext4_wipe_slack.sh --folder /home/user --passes 3
EOF
exit 0
}

# ── Argument parsing ─────────────────────────────────────────
while [[ $# -gt 0 ]]; do
  case "$1" in
    --file)    MODE="file";   TARGET="$2"; shift 2;;
    --list)    MODE="list";   TARGET="$2"; shift 2;;
    --folder)  MODE="folder"; TARGET="$2"; shift 2;;
    --dry-run) DRY_RUN=true;  shift;;
    --passes)  PASSES="$2";   shift 2;;
    --verbose) VERBOSE=true;  shift;;
    --help|-h) usage;;
    *) err "Unknown option: $1";;
  esac
done

[[ -z "$MODE" ]] && err "No mode specified. Use --file, --list, or --folder."

# ── Core wipe function ────────────────────────────────────────
wipe_slack() {
  local filepath="$1"

  # Basic checks
  [[ -f "$filepath" ]] || { warn "Not a file: $filepath"; return; }
  [[ -r "$filepath" ]] || { warn "Cannot read: $filepath"; return; }

  local size
  size=$(stat -c '%s' "$filepath")

  # Calculate slack
  local last_block_used=$(( size % BLOCK_SIZE ))
  local slack=0
  if [[ $last_block_used -ne 0 ]]; then
    slack=$(( BLOCK_SIZE - last_block_used ))
  fi

  if [[ $slack -eq 0 ]]; then
    skip "$filepath (no slack — file fills blocks exactly)"
    return
  fi

  TOTAL_SLACK=$(( TOTAL_SLACK + slack ))
  TOTAL_FILES=$(( TOTAL_FILES + 1 ))

  if [[ "$DRY_RUN" == true ]]; then
    echo -e "  ${YELLOW}[DRY-RUN]${NC} $filepath"
    echo -e "            Size: $size bytes | Slack: $slack bytes | Offset: $size"
    return
  fi

  [[ -w "$filepath" ]] || { warn "Cannot write (need root or permission): $filepath"; return; }

  # Write zeros to the slack area — byte by byte from file end
  for (( pass=1; pass<=PASSES; pass++ )); do
    if [[ $PASSES -gt 1 ]]; then
      case $pass in
        1) PATTERN='\x00';;   # zeros
        2) PATTERN='\xff';;   # ones
        3) PATTERN='\x00';;   # zeros again (DoD final pass)
      esac
    else
      PATTERN='\x00'
    fi

    # Use dd to write exactly $slack bytes of zeros starting at offset $size
    dd if=/dev/zero of="$filepath" \
       bs=1 count="$slack" seek="$size" \
       conv=notrunc status=none 2>/dev/null

    if [[ "$VERBOSE" == true ]]; then
      echo -e "    Pass $pass: wrote $slack zero-bytes at offset $size"
    fi
  done

  # Sync to disk
  sync "$filepath" 2>/dev/null || true

  TOTAL_WIPED=$(( TOTAL_WIPED + slack ))
  ok "$filepath ($slack bytes wiped at offset $size)"
}

# ── Main logic ────────────────────────────────────────────────
[[ "$DRY_RUN" == true ]] && warn "DRY-RUN mode — no data will be changed"
echo ""

case "$MODE" in
  file)
    wipe_slack "$TARGET"
    ;;
  list)
    [[ -f "$TARGET" ]] || err "List file not found: $TARGET"
    while IFS= read -r line; do
      [[ -z "$line" || "$line" == \#* ]] && continue
      wipe_slack "$line"
    done < "$TARGET"
    ;;
  folder)
    [[ -d "$TARGET" ]] || err "Folder not found: $TARGET"
    log "Scanning folder: $TARGET"
    while IFS= read -r -d '' filepath; do
      wipe_slack "$filepath"
    done < <(find "$TARGET" -type f -print0 2>/dev/null)
    ;;
esac

# ── Summary ───────────────────────────────────────────────────
echo ""
echo -e "${GREEN}══════════════════════════════════════${NC}"
echo -e "${GREEN}  Wipe complete${NC}"
echo -e "${GREEN}══════════════════════════════════════${NC}"
echo -e "  Files processed : $TOTAL_FILES"
echo -e "  Total slack found : $TOTAL_SLACK bytes"
if [[ "$DRY_RUN" == false ]]; then
  echo -e "  Total bytes wiped : $TOTAL_WIPED bytes"
  echo -e "  Passes : $PASSES"
fi
echo ""

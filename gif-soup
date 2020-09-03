#!/bin/sh
#
# Uses animated files as wallpapers.
# Requires: mpv(1) and convert(1)

set -e

: "${_ext=gif}"

__usage () {
  cat << EOH
$0 path [left-ratio right-ratio top-ratio bottom-ratio]
$0 -h

$0 will use animated files in "path" and use them as looping wallpapers
$0 -h displays this help text

The optional ratio values are used in environments where the default mpv(1)
behavior to center the video doesn't fit (multiple screens with odd placement).
In that case, those arguments are passed as-are to mpv(1) and you should read up
on the --video-margin-ratio-* options in its manpage.

Example: 2560x1440+0+336 1080x1920+2560+0 (xrandr(1) output), want to center the
gif on the 1440p monitor.
-> left-ratio is 0
-> right-ratio is 1080/(2560+1080) = 0.2967
-> top-ratio is 336/1920 = 0.175
-> bottom-ratio is (1920 - 336 - 1440)/1920 = 0.075

We look for files ending in $_ext .
You might set \$_ext to change this behavior.
Use SIGUSR1 to use another random wallpaper (might be the same!)
Use SIGUSR2 to generate the file list again
EOH
}

# parsing options
case "$1" in
  -h|help) __usage ; exit 0 ;;
  "") __usage >&2  ; exit 1 ;;
  *)
    _path="$1"
    _lr="${2:-0}"
    _rr="${3:-0}"
    _tr="${4:-0}"
    _br="${5:-0}" ;;
esac

# simple, stupid check
ls "$_path" >/dev/null

# We work in a temporary dir
_tmp="$(mktemp -dt "$(basename "$0").XXXX")"

__cleanup () {
  [ -d "$_tmp" ] && rm -rf "${_tmp:?}"
}

trap __cleanup INT TERM

__create_list () {
  # We list the files and store that have the good extension
  : > "$_tmp/list"
  for _e in $_ext; do
    find "$_path" -name '*.'$_e >> "$_tmp/list"
  done
}

__create_list

trap __create_list USR2

__pick_one () {
  # We randomly pick one file and return its path
  shuf "$_tmp/list" | head -n 1
}

__bg () {
  # Select bg color:
  # convert(1) to extract the single top-left pixel and extract its color
  # throughout the .gif; then grep for its only values that are NOT transparent
  # and then take the first one
  _solid_color="$(convert "$1" -crop 1x1+0+0 -depth 8 txt: | grep -Eo \
    '#[A-F0-9]{6}(FF|)' | sort -u | head -n 1)"

  # Now $_solid_color contains either #RRGGBB or #RRGGBBAA and we need to
  # discard the 'AA' part (where AA = FF anyway, because we used grep earlier),
  # hence the '%%FF'
  _solid_color="${_solid_color%%FF}"

  printf '%s\n' "${_solid_color}"
}

__run_mpv () {
  mpv --border --video-unscaled --background="$2" \
   --wid=0 --loop-file=inf --speed=1.0 \
   --video-margin-ratio-left="$_lr" --video-margin-ratio-right="$_rr" \
   --video-margin-ratio-top="$_tr"  --video-margin-ratio-bottom="$_br" \
   "$1" >/dev/null 2>&1 &
}

__kill () {
  kill "$(pgrep -P "$$" "$1")" 2>/dev/null || :
}

__set_a_wp () {
  _gif="$(__pick_one)"
  _bg="$(__bg "$_gif")"
  __kill mpv
  __run_mpv "$_gif" "$_bg"
}

__set_a_wp

trap __set_a_wp USR1
trap '__kill mpv; __cleanup' INT TERM

mkfifo "$_tmp/fifo" || exit
chmod 600 "$_tmp/fifo"
read < "$_tmp/fifo" 2>/dev/null

__cleanup


# Pipeline: copy-paste scripts (bash 3.2 safe)

Set these once. `NAMES` is the ordered section ids; the last is the hero/finale.

```bash
WORK=/tmp/scroll-world           # scratch dir for prompts, sources, frames
ASSETS=./assets                  # where the site reads stills (webp) + clips (mp4)
mkdir -p "$WORK" "$ASSETS/vid"
NAMES="farm kitchen shop delivery plaza finale"   # <-- your section ids, in order

# Chain video model — ONE for every chained clip (SKILL Step 4 roster).
# Must accept --start-image AND --end-image:
# omni | omni_mini (draft tier). Reference-only models can't hold a seam.
VMODEL=omni
case "$VMODEL" in
  omni_mini) DIVE_DUR=8;  CONN_DUR=5 ;;  # cheap frame-locked previz
  *)         DIVE_DUR=8;  CONN_DUR=5 ;;  # omni default
esac
```

Antigravity agent generations may take a few minutes — run these batches detached.

## 1. Scene stills (Step 2)

Write one prompt file per section to `$WORK/still_<name>.txt` (see prompts.md), then:

```bash
gen_still() { # name
  agy --print "Use the image generation tool to generate: $(cat "$WORK/still_$1.txt"). Save it as $WORK/still_$1.png. Do not do anything else." \
    > "$WORK/still_$1.log" 2>&1
  [ -f "$WORK/still_$1.png" ] && echo "still $1 ok" || echo "still $1 FAIL (see .log)"
}
for n in $NAMES; do gen_still "$n" & done ; wait
```

Codex variant (STILLS_SOURCE=codex, SKILL Step 1.6 — subscription-billed, zero
credits; ~1–3 min each, parallelize in small batches):

```bash
gen_still_codex() { # name
  codex exec -C "$WORK" -s workspace-write --skip-git-repo-check \
    'Use the image generation tool ($imagegen) to generate: '"$(cat "$WORK/still_$1.txt")"' Wide 3:2 landscape, high resolution. Save it as ./still_'"$1"'.png. Do not do anything else.' \
    > "$WORK/still_$1.codex.log" 2>&1
  [ -f "$WORK/still_$1.png" ] && echo "still $1 ok (codex)" || echo "still $1 FAIL (see .codex.log)"
}
```

Convert to webp for the site (and optionally run knockout.py first for transparency):

```bash
for n in $NAMES; do cwebp -quiet -q 84 -resize 1800 0 "$WORK/still_$n.png" -o "$ASSETS/$n.webp"; done
```

Review the stills for cohesion before continuing. Re-roll any off-style one (optionally
add `--image "$WORK/still_<good>.png"` to lock style).

## 2. Dive-in clips (Step 4)

Prompt files at `$WORK/dive_<name>.txt`. Start image = the solid-bg still PNG.

```bash
gen_dive() { # name
  python3 ./references/omni_video.py --prompt "$(cat "$WORK/dive_$1.txt")" \
    --start-image "$WORK/still_$1.png" \
    --duration "$DIVE_DUR" \
    --output "$WORK/dive_$1.mp4" > "$WORK/dive_$1.log" 2>&1
  [ -f "$WORK/dive_$1.mp4" ] && echo "dive $1 ok" || echo "dive $1 FAIL"
}
for n in $NAMES; do gen_dive "$n" & done ; wait
```

Re-roll individual failures (503 / credit race are transient):
`gen_dive shop`  (just that one).

## 3. Extract boundary frames — the seam handoff (Step 5)

For each adjacent pair, the connector's start = dive_i's LAST frame, end = dive_{i+1}'s
FIRST frame — extracted from the **rendered videos**, never the stills.

```bash
set -- $NAMES
prev=""
for n in "$@"; do
  ffmpeg -v error -ss 0 -i "$WORK/dive_$n.mp4" -frames:v 1 -q:v 2 "$WORK/first_$n.png"      # establishing
  ffmpeg -v error -sseof -0.15 -i "$WORK/dive_$n.mp4" -frames:v 1 -q:v 2 "$WORK/last_$n.png" # interior
done
```

## 4. Connector clips (Step 5)

Prompt files at `$WORK/conn_<i>.txt` (i = 1..N-1). Iterate adjacent pairs:

```bash
gen_conn() { # i startPng endPng
  python3 ./references/omni_video.py --prompt "$(cat "$WORK/conn_$1.txt")" \
    --start-image "$2" --end-image "$3" \
    --duration "$CONN_DUR" \
    --output "$WORK/conn_$1.mp4" > "$WORK/conn_$1.log" 2>&1
  [ -f "$WORK/conn_$1.mp4" ] && echo "conn $1 ok" || echo "conn $1 FAIL"
}
set -- $NAMES ; i=0 ; prev=""
for n in "$@"; do
  if [ -n "$prev" ]; then i=$((i+1)); gen_conn "$i" "$WORK/last_$prev.png" "$WORK/first_$n.png" & fi
  prev="$n"
done ; wait
```

## 5. Encode everything for scrubbing (Step 6)

Native resolution (1080p from omni std; kling3_0 std returned **720p** in testing —
never upscale, encode what ffprobe reports), crf 20, GOP 8, light sharpen, no audio,
faststart. Same for dives + connectors.

```bash
enc() { ffmpeg -v error -y -i "$1" -an -vf "unsharp=5:5:0.8:5:5:0.0" \
  -c:v libx264 -preset slow -crf 20 -pix_fmt yuv420p \
  -g 8 -keyint_min 8 -sc_threshold 0 -movflags +faststart "$2"; echo "enc $2 $(du -h "$2"|cut -f1)"; }

for n in $NAMES; do enc "$WORK/dive_$n.mp4" "$ASSETS/vid/$n.mp4"; done
i=0; for f in "$WORK"/conn_*.mp4; do i=$((i+1)); enc "$f" "$ASSETS/vid/conn$i.mp4"; done
```

Now the engine config's `sections[k].clip = assets/vid/<name>.mp4` and
`connectors = [assets/vid/conn1.mp4, …]` (length N-1, in order).

## 6. Centre-crop mobile encodes — FALLBACK ONLY, not the mobile version

**The mobile version is the native 9:16 portrait chain (§6b).** This section's crop
encodes exist for one case: the user opted into mobile but credits can't cover the
portrait chain — and shipping them must be called out and approved, never silent
(portrait phones will see the landscape film's centre ~26%). The encode mechanics
matter either way: scrubbing sets `currentTime` every frame, and a phone decoder's
**seek cost scales with how many frames it must decode from the nearest keyframe** — so
a 1080p `-g 8` master that scrubs fine on a laptop stutters on a phone. A **smaller
frame + tighter GOP** fixes that (and halves the bytes on cellular). The crop `-m.mp4`
sibling per clip:

```bash
# 720p, GOP 4 (twice the keyframes = ~half the seek-decode work), crf 23, same sharpen/faststart.
encm() { ffmpeg -v error -y -i "$1" -an -vf "scale=-2:720,unsharp=5:5:0.6:5:5:0.0" \
  -c:v libx264 -preset slow -crf 23 -pix_fmt yuv420p \
  -g 4 -keyint_min 4 -sc_threshold 0 -movflags +faststart "$2"; echo "encm $2 $(du -h "$2"|cut -f1)"; }

for n in $NAMES; do encm "$WORK/dive_$n.mp4" "$ASSETS/vid/$n-m.mp4"; done
i=0; for f in "$WORK"/conn_*.mp4; do i=$((i+1)); encm "$f" "$ASSETS/vid/conn$i-m.mp4"; done
```

Wire the variants in the engine config — the engine serves them automatically on phones,
falling back to the desktop `clip` when a mobile one is absent:

```js
sections[k].clipMobile = 'assets/vid/<name>-m.mp4';
connectorsMobile = ['assets/vid/conn1-m.mp4', …];   // length N-1, in order
```

If phone scrubbing still stutters, tighten the GOP further (`-g 2`, or `-g 1` for all-intra
= instant seeks at the cost of larger files); if cellular weight is the bigger worry, raise
`crf` (24–26) or drop to `scale=-2:600`. If the master is already 720p (e.g. kling3_0 std),
the mobile encode still pays off — the tighter GOP is what makes phone seeks cheap. All-mobile encodes stay 16:9 — the engine
centre-crops them; see the portrait note in SKILL Step 8 / prompts.md.

## 6b. Native 9:16 portrait chain — THE mobile version (Step 1.5 opt-in)

When the user opts into mobile, this is what they get: a **parallel 9:16 chain** rendered
natively for phones and shipped as the mobile variants — never the §6 crops (those are the
no-credits stopgap). Same seam laws as the main chain — the portrait chain frame-locks
against its own rendered frames, never the landscape ones. Budget ~2N-1 video gens +
re-rolls (interiors trip the NSFW filter in portrait too); state the credit cost at the
Step 1.5 interview.

1. **Portrait start canvases.** Don't hand the video model a 3:2 still and hope: composite
   each scene onto a 1080×1920 canvas in the page bg colour (island at ~94% width, visual
   centre at ~45% height). The render then opens exactly on what the portrait poster shows.
   For knocked-out stills, composite the RGBA over the bg colour first.
2. **Dives/legs**: same prompt templates with a portrait clause up front ("Vertical
   portrait composition, the diorama centered with generous [bg] space above and below"),
   `--aspect_ratio 9:16`, same model/params as the main chain. Review each last frame
   before chaining, as ever.
3. **Connectors**: extract first/last frames **from the 9:16 renders** and generate 9:16
   connectors between them. A native 9:16 scene mixed into cropped-16:9 neighbours pops at
   both seams — the portrait chain must be complete, not partial.
4. **Encode** with the §6 settings but portrait-oriented scale: `scale=720:-2` (720 wide),
   `-g 4`, crf 23 → these ARE the `-m.mp4` mobile files (and they replace any §6 crop
   stopgaps that shipped earlier).
5. **Posters**: extract each 9:16 dive's first frame → webp → wire as the section's
   `stillMobile` so the poster matches the portrait video's frame 0 (no landscape→portrait
   flash when the clip paints). Engine support: `sections[k].stillMobile`.

## Notes

- Ensure you have correctly authenticated your API access for Omni before running the batch.
- **NSFW fallback across models**: if one clip keeps getting flagged on omni after
  re-rolls + prompt scrubbing, regenerate just that clip on `kling3_0` with the SAME
  start/end frames: `VMODEL=kling3_0; VOPTS="--mode std --sound off"; gen_conn 3 …` —
  then restore your chain model. See SKILL Gotchas for the trade-off.
- **Previz on the cheap**: run the whole chain once with `VMODEL=omni_mini`
  (frame-locking intact, ~720p) to validate the journey and seams before spending
  full-model credits — because it's still seamless, the previz translates directly to the
  final render. Don't reach for reference-only models here: without `--start/--end-image`
  they can't hold a seam, so their output can't be chained (Step 4 rule).
- If a whole batch stalls, check `agy workspace list` for credits and
  `$WORK/*.err` for the reason.
- Concurrency: launching ~5–6 gens at once is fine; much more can trigger transient
  credit/race errors — stagger or re-roll.

---
name: dip-pen-sketch-combo
description: Turn a photo into BOTH rough dip-pen / nib-and-India-ink sketch versions at once — an outline (pure black ink, no color) AND a colored version (loose marker on top) that share the EXACT SAME linework AND the SAME flat background color. Outputs two separate images automatically. The "Philly sketch" look, matched pair. Use when the user says "dip-pen-sketch-combo", "dip pen combo", "both versions", "outline and color of this photo", "matched pair sketch", or asks for both the inked and colored sketch of a photo with matching lines/background. By Reblis.com.
---

# dip-pen-sketch-combo

One photo in → **two matched images out**, automatically:
1. **Outline** — rough dip-pen black-ink line sketch, no color.
2. **Color** — the *same* sketch with loose marker color added on top.

Both share **stroke-for-stroke identical linework** and **an identical flat background color**. This is GUARANTEED, not hoped-for: the colored image is generated first, then the outline is **derived from it** (the marker fills are stripped back to white, the ink is kept). Because the outline is literally extracted from the colored image, the two are pixel-aligned by construction and the marker is never altered. Subject(s) on blank paper, original background removed, faces recognizable.

> ⚠️ **The one rule that makes this skill work — never deviate.** `fal-ai/nano-banana-2/edit` **redraws its linework on every pass**, so "paint color on top of the outline" does NOT produce matching lines (the two drift — "one bigger than the other", doubled outlines when overlaid). Do **not** deliver a separately-generated outline, and do **not** try to fix drift afterward by compositing / blurring / morphology — every one of those degrades the marker. The ONLY supported method is: generate color → derive the outline from that color (Step 5). This is settled; do not re-litigate it with the user.

For just one finish, use **dip-pen-sketch-outline** (ink only) or **dip-pen-sketch-color** (color only).

## Requirements

- **[Claude Code](https://claude.com/claude-code)**
- **The fal.ai MCP server** connected, with credits. Runs **`fal-ai/nano-banana-2/edit`** — Google's **Nano Banana 2** image-editing model via fal.ai (preserves facial likeness). **~$0.03–0.04 per image → ~$0.06–0.08 per combo** (two generations).
- `curl` + ImageMagick (`convert`).
- Optional free alternative: the **nanobanana / Gemini MCP** (`gemini_edit_image`), same model family — but its free-tier quota is often exhausted, so fal is the reliable default.

## Inputs

Resolve in **Step 0** (same as the sibling skills):
- **Local path** (`/tmp/foo.jpg`).
- **Direct image URL** (`Content-Type: image/*`) — pass straight through.
- **Social / page link** (Instagram, Facebook, Pinterest) — HTML pages, not images; Meta CDN links are signed/expiring and block hotlinking. Download locally first.

## Procedure

0. **Resolve the input to a usable image.**
   - Direct image URL fal can fetch (`curl -sIL "$URL"` → `200` + `image/*`)? Use it directly as the Step-1 image, skip resize/upload.
   - Otherwise download it (`curl -sL -A 'Mozilla/5.0' -o /tmp/_dippen_dl.jpg "$URL"`; browser if Meta blocks it) → that file is `$SRC`.
   - Local path → `$SRC` directly.
   - Set `NAME` = a short slug for the subject (used in output filenames).

1. **Downscale the source** (CMYK→sRGB; ~1024px). *(Skip for direct-URL passthrough.)*
   ```bash
   convert "$SRC" -colorspace sRGB -resize 1024x -quality 82 /tmp/_dippen_src.jpg && identify /tmp/_dippen_src.jpg
   ```
   `Read` `/tmp/_dippen_src.jpg` to note the subject(s), clothing, key features (for the "keep recognizable" clause) and garment colors (for the color step).

2. **Get a URL fal can fetch (`$IMG_URL`)** for the source. *(Skip for direct-URL passthrough.)*
   - **Upload to fal's CDN (portable):** `mcp__fal-ai__upload_file` with `/tmp/_dippen_src.jpg` as base64 `data` → returned `url`.
   - **Your own web server:** copy into its docroot and use that public URL, e.g. `https://<your-site>/_dippen_src.jpg`.

3. **Generate the BASE ink sketch.** `mcp__fal-ai__run_model`, `fal-ai/nano-banana-2/edit`:
   - `image_urls`: `[$IMG_URL]`; `resolution`: `"2K"`; `output_format`: `"jpeg"`; `aspect_ratio`: `"auto"`
   - `prompt`: the **OUTLINE prompt** below (adapt the "keep recognizable" clause).
   - Download → `/tmp/${NAME}_combo_base.jpg`. This is **only the base for the color pass** — it is NOT a deliverable. The final outline is derived in Step 5.

4. **Generate the COLOR version ON TOP of the base (this IS the color deliverable).** Get a fal URL for the base you just made (upload it the same way as Step 2), then run a second edit that adds color:
   - `image_urls`: `[<base URL>]`; same `resolution`/`format`/`aspect_ratio`
   - `prompt`: the **COLOR-ON-TOP prompt** below (adapt the garment-color clause).
   - Download → `/tmp/${NAME}_combo_color.jpg`. Whiten its background so nothing can halo later:
     ```bash
     cd /tmp
     bg=$(convert ${NAME}_combo_color.jpg -format '%[pixel:p{3,3}]' info:)
     convert ${NAME}_combo_color.jpg -fuzz 12% -fill white -opaque "$bg" ${NAME}_combo_color.jpg
     ```
   - **The marker in this file is final and must NOT be touched again.** Its linework is now the canonical, single source of truth.

5. **DERIVE the OUTLINE from the color image — this is what guarantees identical linework.** Flatten the marker fills back to white (divide-by-blur removes flat color but keeps the dark ink), then deepen to confident black ink. This reconstructs the outline from the exact same pixels, so the lines match the color stroke-for-stroke:
   ```bash
   cd /tmp
   convert ${NAME}_combo_color.jpg -colorspace Gray \
           \( +clone -blur 0x18 \) -compose Divide_Src -composite \
           -level 6%,86% -sigmoidal-contrast 5x50% -level 14%,92% \
           ${NAME}_combo_outline.jpg
   ```
   (Lines too light? Lower the last white point, e.g. `-level 16%,88%`, or repeat `-sigmoidal-contrast 4x50%`. Too noisy/grainy in the fills? Raise the blur radius to `0x24` and/or the last black point to `18%`.)

6. **Normalize BOTH backgrounds to identical pure white.** The color was whitened in Step 4; do the same for the derived outline so the two read exactly the same paper tone:
   ```bash
   cd /tmp
   for f in ${NAME}_combo_outline ${NAME}_combo_color; do
     bg=$(convert "$f.jpg" -format '%[pixel:p{3,3}]' info:)
     convert "$f.jpg" -fuzz 12% -fill white -opaque "$bg" "$f.jpg"
   done
   for f in ${NAME}_combo_outline ${NAME}_combo_color; do echo "$f -> $(convert "$f.jpg" -format '%[pixel:p{3,3}]' info:)"; done   # both should read 255,255,255
   ```
   (Default target is pure white `#FFFFFF`. To match on cream paper instead, use the SAME `-fill '#F4EEE2'` for both files.)

7. **`Read` both final files**, present them as **two separate images** (outline + color), report both paths, and confirm lines + background match. Offer tweaks (rougher, more/less marker, crop, 4K, or bolder/lighter ink via the Step-5 levels).

## OUTLINE prompt (image #1)

> Transform this photo into a ROUGH, loose dip-pen and India-ink line sketch. CRITICAL: keep the subject(s) clearly recognizable with their exact same face(s), expression(s), hair, facial hair, pose, sunglasses/accessories, and clothing. Linework: thin black India-INK lines from an old-school DIP PEN / steel nib, drawn fast, ROUGH and SCRATCHY and messy — loose energetic scribbled strokes, overlapping built-up scratchy hatching, uneven wobbly lines, some lines doubled or left incomplete and overshooting, occasional ink splatter dots, a quick raw gesture-sketch feel. Thin nib lines but NOT clean, NOT refined, NOT a polished illustration — deliberately rough and sketchy like a fast unfinished pen doodle. PURE BLACK INK ONLY — absolutely NO color, monochrome black-and-white line art, use ink cross-hatching and stippling for ALL shading and tone. Plain blank white sketchbook paper background — remove the original background entirely, just the figure(s) on empty paper.

## COLOR-ON-TOP prompt (image #2 — runs on the base ink sketch)

> This image is a finished rough black dip-pen ink line sketch. Add loose alcohol-MARKER coloring (Copic style) ON TOP of it. ABSOLUTELY CRITICAL: do NOT change, redraw, move, or clean up ANY of the existing black ink lines, hatching, stippling, or splatter — keep every single pen stroke exactly as it is, in the exact same positions. Only ADD color over the existing drawing: flat streaky marker fills that bleed slightly and messily outside the ink lines, visible marker strokes, patchy uneven coverage, a limited palette — natural skin tones [+ describe the key garment colors, e.g. "dark charcoal/black for the shirt"]. Leave the blank white paper background untouched and white. The result must look like the SAME drawing, just with marker color laid over the identical ink linework.

## Notes

- **Why derive the outline from the color instead of delivering the base ink sketch?** Because `nano-banana-2` redraws linework on the color pass, the base ink sketch and the colored image have *different* lines — overlay them and you get doubled/offset strokes ("one bigger than the other"). Extracting the outline from the finished color image is the only way to get truly identical lines, and it leaves the marker 100% untouched. **Field-tested 2026-06: every alternative (multiply the base on top → double lines; blur to kill the model's lines → muddy wash + gray halo; morphology to strip lines → marker "way off") was rejected by the user. Don't repeat them.**
- **Why divide-by-blur for the derive?** Dividing the grayscale by a heavily-blurred copy of itself flattens broad marker fills to white while leaving high-frequency ink strokes dark — i.e. it removes the *color* and keeps the *lines*. The `-level` / `-sigmoidal-contrast` afterward just deepens those lines to confident black ink. Known tradeoff: the derived ink is a touch lighter than an independently-generated outline; deepen via the Step-5 levels if asked, but never by re-running a separate outline generation (that breaks line-matching).
- **Why whiten backgrounds (Steps 4 & 6)?** Generations come back on slightly different paper tones, and any stray off-white will halo. Corner-sampled `-opaque` flood only touches near-paper pixels, so ink/marker/skin are safe (`-fuzz 12%` is the tested sweet spot; lower it if light skin highlights wash out).
- Adapt the "keep recognizable" clause and the garment-color clause to the actual photo — generic prompts drift.
- If the base ink sketch is too clean/refined, re-run Step 3 emphasizing "rougher, scratchier, wobbly, incomplete lines" before doing the color pass.

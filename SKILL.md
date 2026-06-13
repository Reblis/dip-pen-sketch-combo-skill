---
name: dip-pen-sketch-combo
description: Turn a photo into BOTH rough dip-pen / nib-and-India-ink sketch versions at once — an outline (pure black ink, no color) AND a colored version (loose marker on top) that share the EXACT SAME linework AND the SAME flat background color. Outputs two separate images automatically. The "Philly sketch" look, matched pair. Use when the user says "dip-pen-sketch-combo", "dip pen combo", "both versions", "outline and color of this photo", "matched pair sketch", or asks for both the inked and colored sketch of a photo with matching lines/background. By Reblis.com.
---

# dip-pen-sketch-combo

One photo in → **two matched images out**, automatically:
1. **Outline** — rough dip-pen black-ink line sketch, no color.
2. **Color** — the *same* sketch with loose marker color added on top.

Both share **stroke-for-stroke identical linework** (the color is painted on top of the outline, not generated separately) **and an identical flat background color** (both flattened to the same paper tone at the end). Subject(s) on blank paper, original background removed, faces recognizable.

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

3. **Generate the OUTLINE (image #1).** `mcp__fal-ai__run_model`, `fal-ai/nano-banana-2/edit`:
   - `image_urls`: `[$IMG_URL]`; `resolution`: `"2K"`; `output_format`: `"jpeg"`; `aspect_ratio`: `"auto"`
   - `prompt`: the **OUTLINE prompt** below (adapt the "keep recognizable" clause).
   - Download → `/tmp/${NAME}_combo_outline.jpg`.

4. **Generate the COLOR version ON TOP of the outline (image #2).** Get a fal URL for the outline you just made (upload it the same way as Step 2), then run a second edit that ONLY adds color:
   - `image_urls`: `[<outline URL>]`; same `resolution`/`format`/`aspect_ratio`
   - `prompt`: the **COLOR-ON-TOP prompt** below (adapt the garment-color clause).
   - Download → `/tmp/${NAME}_combo_color.jpg`.

5. **Normalize BOTH backgrounds to the same flat color** — the key fix. The two generations come back on *slightly* different paper tones; flatten both to identical pure white so they truly match. Sample each image's corner paper color and flood it to white in-place:
   ```bash
   cd /tmp
   for f in ${NAME}_combo_outline ${NAME}_combo_color; do
     bg=$(convert "$f.jpg" -format '%[pixel:p{3,3}]' info:)
     convert "$f.jpg" -fuzz 12% -fill white -opaque "$bg" "$f.jpg"
   done
   for f in ${NAME}_combo_outline ${NAME}_combo_color; do echo "$f -> $(convert "$f.jpg" -format '%[pixel:p{3,3}]' info:)"; done   # both should read srgb(255,255,255)
   ```
   (Default target is pure white `#FFFFFF`. To match on cream paper instead, use the SAME `-fill '#F4EEE2'` for both files.)

6. **`Read` both final files**, present them as **two separate images** (outline + color), report both paths, and confirm lines + background match. Offer tweaks (rougher, more/less marker, crop, 4K).

## OUTLINE prompt (image #1)

> Transform this photo into a ROUGH, loose dip-pen and India-ink line sketch. CRITICAL: keep the subject(s) clearly recognizable with their exact same face(s), expression(s), hair, facial hair, pose, sunglasses/accessories, and clothing. Linework: thin black India-INK lines from an old-school DIP PEN / steel nib, drawn fast, ROUGH and SCRATCHY and messy — loose energetic scribbled strokes, overlapping built-up scratchy hatching, uneven wobbly lines, some lines doubled or left incomplete and overshooting, occasional ink splatter dots, a quick raw gesture-sketch feel. Thin nib lines but NOT clean, NOT refined, NOT a polished illustration — deliberately rough and sketchy like a fast unfinished pen doodle. PURE BLACK INK ONLY — absolutely NO color, monochrome black-and-white line art, use ink cross-hatching and stippling for ALL shading and tone. Plain blank white sketchbook paper background — remove the original background entirely, just the figure(s) on empty paper.

## COLOR-ON-TOP prompt (image #2 — runs on the outline)

> This image is a finished rough black dip-pen ink line sketch. Add loose alcohol-MARKER coloring (Copic style) ON TOP of it. ABSOLUTELY CRITICAL: do NOT change, redraw, move, or clean up ANY of the existing black ink lines, hatching, stippling, or splatter — keep every single pen stroke exactly as it is, in the exact same positions. Only ADD color over the existing drawing: flat streaky marker fills that bleed slightly and messily outside the ink lines, visible marker strokes, patchy uneven coverage, a limited palette — natural skin tones [+ describe the key garment colors, e.g. "dark charcoal/black for the shirt"]. Leave the blank white paper background untouched and white. The result must look like the SAME drawing, just with marker color laid over the identical ink linework.

## Notes

- **Why color-on-top, not two from-scratch runs?** Two independent generations never share linework. Painting color over the finished outline guarantees stroke-for-stroke identical lines.
- **Why the Step-5 normalize?** Even with "leave the background white," the color pass returns paper ~1 level off the outline's. Flattening both to the same flat color via corner-sampled `-opaque` makes them truly match. It only touches near-paper pixels, so ink/marker/skin are safe (`-fuzz 12%` is the tested sweet spot; lower it if light skin highlights wash out).
- Adapt the "keep recognizable" clause and the garment-color clause to the actual photo — generic prompts drift.
- If a result is too clean/refined, re-run that step emphasizing "rougher, scratchier, wobbly, incomplete lines."

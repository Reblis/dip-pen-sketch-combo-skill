# dip-pen-sketch-combo — a Claude Code skill by [Reblis.com](https://reblis.com)

Turn any photo into **both** a rough old-school dip-pen ink sketch **and** a marker-colored version — in one command, with matching lines:

```
dip-pen-sketch-combo /path/to/photo.jpg
```

![source → outline → color](example.jpg)

It produces **two separate images** from one photo: a pure black-ink line sketch and the *same drawing* with loose alcohol-marker color on top. The whole "Philly sketch" TikTok/CapCut look — scratchy dip-pen linework, ink splatter, marker bleeding outside the lines, subject lifted onto blank paper.

The trick: the outline isn't a separate generation — it's **derived directly from the finished colored image** (the marker fills are stripped away, the ink is kept), so the linework matches **stroke-for-stroke by construction**. Both images are then flattened to the **same background color** so the pair looks like one drawing in two finishes.

> Prefer just one finish? See the sibling skills **[dip-pen-sketch-outline](https://github.com/Reblis/dip-pen-sketch-outline-skill)** (ink only) and **[dip-pen-sketch-color](https://github.com/Reblis/dip-pen-sketch-color-skill)** (color only).

## Install

```bash
git clone https://github.com/Reblis/dip-pen-sketch-combo-skill ~/.claude/skills/dip-pen-sketch-combo
```

Restart Claude Code (or start a new session) so the skill registers, then run it on any photo path or image URL.

## Requirements

This skill calls an AI image-editing model — it is **not** local/free:

- **[Claude Code](https://claude.com/claude-code)**
- **A connected [fal.ai](https://fal.ai) MCP server with credits.** The skill runs **`fal-ai/nano-banana-2/edit`** — Google's **Nano Banana 2** image-editing model, served through fal.ai. It's the part that preserves the subject's likeness while restyling. **Cost ≈ $0.03–0.04 per image, so ≈ $0.06–0.08 per combo** (two generations: a base ink sketch, then color on top of it; the delivered outline is derived locally from the colored image — no third generation).
- `curl` and **ImageMagick** (`convert`) for fetching, resizing, and the background-matching step.
- *Optional, free:* a connected **nanobanana / Gemini MCP** runs the same Nano Banana model family at no cost — but its free tier is frequently rate-limited, which is why fal.ai is the default path.

## What you get

- **Two matched images** — `<name>_combo_outline.jpg` (ink) and `<name>_combo_color.jpg` (ink + marker).
- **Identical linework** across both — same strokes, hatching, and splatter, because the outline is extracted from the colored image itself.
- **Identical background** — both flattened to the same flat paper color (pure white by default) so they truly pair.
- **Background removed** — the subject is lifted onto blank sketchbook paper.
- **Likeness preserved** — faces, expressions, hair, and clothing stay recognizable.

## Inputs

A local image path, a direct image URL, or — for Instagram/Facebook/Pinterest links (which are web pages, not images, and block hotlinking) — the skill downloads the photo first.

## Credit

Built by [Reblis.com](https://reblis.com). Uses Google's Nano Banana 2 via [fal.ai](https://fal.ai). MIT licensed.

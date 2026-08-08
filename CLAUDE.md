# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page Turkish-language menu website for **Kumyalı Coffee Line**, a coffee shop in Tirebolu/Giresun. It's a static site (no build step, no framework, no package.json) meant to be reached by customers scanning a QR code on their phones — mobile is the primary and most important viewport.

Live deploy: `https://kumyalicoffeeline.netlify.app` (Netlify). GitHub: `zikreddindurmuss/kumyali-coffee-line`.

## Repo layout: `public/` is the deploy root, this file is not

Everything that actually ships to the browser lives under **`public/`** (`index.html`, `style.css`, `logo.png`, `favicon.ico`, `apple-touch-icon.png`, `netlify.toml`, `Pictures/`). `CLAUDE.md` and `.git/` sit one level up, *outside* `public/`, on purpose — the user manually drag-and-drops a folder into Netlify's deploy UI, and this split means that folder (`public/`) never includes internal dev notes. **When telling the user what to drag/upload to Netlify, it's the `public/` folder, never the repo root.** If a new deployable file is added, it goes inside `public/`; if it's project/dev documentation, it stays at the repo root next to this file.

## Running locally

There's no dev server config in this repo. To preview:

```
python -m http.server 8730 --directory public
```

Then open `http://localhost:8730`. There is no build/lint/test tooling — it's plain HTML/CSS, edit and reload.

**Netlify is not linked to GitHub.** Deploys are manual (drag-and-drop the `public/` folder into the Netlify dashboard, or upload via Netlify's UI). A previous attempt to link the GitHub repo failed with "No repositories found" / "we couldn't access the repository" even after granting the Netlify GitHub App access — this is an unresolved GitHub App / Netlify sync issue, not a repo problem (the repo itself is public, non-empty, not archived). If revisiting this, the fix path is usually: uninstall the Netlify GitHub App entirely from `github.com/settings/installations`, then re-run the "Link repository" flow from scratch rather than just re-granting permissions. (If it ever does get linked, the Netlify site's "Base directory"/"Publish directory" build setting would need to be set to `public`.)

## Files (all under `public/`)

- `index.html` — the entire site: header, hero, menu sections, footer. No templating; every menu item is hand-written HTML.
- `style.css` — all styling, theme tokens as CSS custom properties in `:root` (`--copper`, `--sea`, `--cream`, `--espresso`, etc.), theme is "sea coolness + coffee warmth".
- `logo.png` — brand wordmark, pre-processed to remove its original black background plate (see "Product photo pipeline" below) so it composites directly onto the hero's gradient rather than sitting in a box.
- `favicon.ico` / `apple-touch-icon.png` — tab/home-screen icon, a cropped "C" from the brand badge logo on its dark background.
- `Pictures/` — all menu item photos. Flat folder, no subdirectories.
- `netlify.toml` — security headers only (CSP, X-Frame-Options, etc.), no build config. Must stay inside `public/` since that's the folder actually uploaded to Netlify — headers wouldn't apply if it sat outside the deploy root.

## Menu markup structure

```html
<section class="menu-section" id="...">          <!-- Soğuk İçecekler / Atıştırmalıklar / Sıcak İçecekler -->
  <div class="subcat">                            <!-- e.g. "Soğuk Kahveler" -->
    <div class="product-grid">
      <figure class="product">
        <div class="product-photo">
          <img class="product-img-fill" src="Pictures/xxx.png" alt="..." loading="lazy">
          <span class="ph" aria-hidden="true">☕</span>   <!-- fallback icon, hidden by CSS when an img is present -->
          <span class="price-badge">200 ₺</span>
        </div>
        <figcaption class="product-name">Item Name</figcaption>
      </figure>
      <!-- more <figure class="product"> ... -->
    </div>
  </div>
</section>
```

Items without a photo yet just omit the `<img>` and show the `.ph` placeholder emoji instead — there's no build-time distinction, it's whatever's in the HTML.

**Exception — items with multiple sizes/prices sharing one photo:** instead of repeating the `<figure class="product">` per size, use `.product-grid.single-item` (flex, not grid) with one photo and a `.size-options` row of `.size-pill` spans (`Küçük Boy 100 ₺`, etc.). Used by Patates Kızartması (Küçük/Orta/Büyük Boy), Tost Çeşitleri (Sucuklu/Kaşarlı/Karışık, each 200 ₺, one shared photo), and Tost Menü (same three flavors + fries, each 300 ₺, its own photo showing the combo plate). Follow this pattern if another multi-size/multi-variant item shows up — don't duplicate the photo across multiple `<figure>`s.

## Product photo pipeline

All photos in `Pictures/` go through the same normalization before being committed, using PIL (+ `rembg` for background removal when needed):

1. If the source photo doesn't already have real alpha transparency (check: `Image.open(path).convert('RGBA').getpixel((0,0))` — if opaque, or if it's a "baked-in checkerboard" fake-transparency preview export, it needs step 2).
2. Run through `rembg` (`from rembg import remove`) to strip the background.
3. Crop to `getbbox()`, then pad with ~12% margin on a transparent square canvas.
4. Resize to 900×900 (`Image.LANCZOS`).
5. Quantize to reduce file size: `img.quantize(colors=200, method=Image.FASTOCTREE)`, save with `optimize=True`. This typically takes a 300KB+ raw photo down to 30–100KB.

File naming: lowercase kebab-case ASCII (`sikma-nar-suyu.png`, not `sıkma nar suyu.png`) — source files dropped into `Pictures/` by the user often have Turkish characters/spaces/mixed case and get renamed to the ASCII form during processing. Only the final processed files should be committed; raw/unprocessed source uploads should be deleted from `Pictures/` before committing (they're 5-20x larger and unused — cross-check with `grep -oE 'src="Pictures/[^"]+"' public/index.html` to confirm nothing referenced got deleted).

**Watermarked source images:** users sometimes supply photos with a preview watermark (CityPNG, Shutterstock, Vecteezy, PNGTree seen so far). If the watermark is a separate element (e.g. a text strip outside the subject), `rembg` usually strips it along with the background — check the result. If the watermark is tiled *across* the actual subject (seen with Vecteezy and PNGTree), it can't be cleanly removed — the default is to not ship it and leave that menu item without a photo. This is a default, not an absolute: if the user explicitly says to use it anyway (asked and confirmed for Sıcak Ballı Süt / Sıcak Süt), ship it as-is rather than re-asking.

**Composited multi-source photos:** occasionally a single menu item needs elements from two different source photos (e.g. Çikolatalı Türk Kahvesi = a coffee/tea cup photo + a separate chocolate-bar photo, requested explicitly by the user). Process each source separately (rembg cutout, no quantize yet), composite onto one transparent canvas with `Image.paste(img, pos, img)` in the desired z-order (background element first, then the element that should occlude it), *then* run the combined result through the normal crop/pad/resize/quantize finalize step once.

**Messy/real-world-background source photos:** for real photographs (not studio product shots) with blurry or cluttered backgrounds, `rembg` can leave low-opacity noise scattered across the whole frame, making `getbbox()` return the entire canvas instead of a tight crop around the subject. Fix: threshold the alpha channel before computing bbox (e.g. `alpha.point(lambda p: 255 if p > 128 else 0)`) so only strongly-opaque pixels count as content.

**Plain-white studio-background source photos:** two opposite failure modes show up here, and the fix for one is the cause of the other — check which one you're looking at before picking a method.
- `rembg` sometimes *over-erases*: it treats branded bottle/box packaging as "background" against a plain white backdrop and deletes most of it. Fix: skip `rembg`, key out near-white pixels directly (Euclidean RGB distance from `(255,255,255)` below a threshold → alpha 0).
- But that same direct-keying approach *under-erases the wrong thing* — it eats real product content — whenever the product itself has a white/cream part that's colored near-identically to the backdrop (e.g. a white cup handle or saucer rim on a pure-white background): there's no color difference to key on, so a handle can vanish into "background" entirely (seen on Damla Sakızlı Kahve — the handle was gone at every tolerance tried). `rembg`'s semantic segmentation handles this correctly since it recognizes the cup+handle as one shape regardless of local color match to the backdrop. Default to `rembg` for real (non-flattened-checkerboard) photos; only fall back to direct near-white keying if `rembg` is specifically over-erasing distinguishable-color packaging.

**Windows case-insensitive filesystem gotcha:** `Pictures/Semaver.png` and `Pictures/semaver.png` are the *same file* on Windows/Git Bash. If a raw source and its processed destination differ only by case, saving the destination silently overwrites the source in place — there's no separate "raw" file left to clean up afterward. A later blind `rm` of the "raw" filename deletes the *processed* result instead (this happened once — lost `semaver.png` this way and had to ask the user to resupply the source). When source/dest names differ only by case, either skip the cleanup-delete step for that file, or verify with `ls` that two distinct files still exist before deleting either.

## CSS/i18n gotcha: `text-transform: uppercase` + `lang="tr"`

`<html lang="tr">` triggers Turkish-specific case mapping in some browsers: `text-transform: uppercase` turns lowercase `i` into dotted `İ`. This is correct for Turkish words but breaks embedded English/brand text (e.g. "Line" → "LİNE" instead of "LINE"). Fix used throughout: write the English word already fully uppercase in the HTML source (`LINE`, `MILKSHAKE`) instead of relying on the CSS transform to uppercase a lowercase source string. Only apply this fix to non-Turkish words — Turkish words (e.g. "Tirebolu" → "TİREBOLU") should keep the dotted-İ behavior since it's linguistically correct there.

## Known gaps

- **Meyveli Soda** (`meyveli-soda.png`, three Fresa bottles) — has visible dithering/speckle on the bottle glass and a leftover shadow smear at the base of the red bottle. Root cause: the file is a 200-color quantized PNG with an already-noisy alpha matte (baked in, not something post-processing can cleanly fix — attempts to denoise/rethreshold the alpha ended up eating parts of the label text). The raw source is already deleted. User was told and said to leave it for now; needs a resupplied source photo for a proper redo, don't attempt in-place fixes again without one.
- **Semaver-style single-photo sections**: `Doğal Bitki Çayları` and `Oraletler` were both converted from per-item photos to one shared photo + `.size-options` price pills (matching the existing `Semaver` / `Tost Çeşitleri` pattern), at the user's explicit request. If new single-item sections show up, follow this same pattern.
- **Orphaned processed files**: `kivi.png`, `kusburnu.png`, `karadut-oralet.png`, `portakal-oralet.png` are no longer referenced by `index.html` (superseded by the single `oraletler.png` photo) but were left in `Pictures/` rather than deleted outright, since they're previously-committed processed assets, not raw uploads — ask the user before removing them.
- **Checkerboard-background source photos**: some user-supplied "transparent PNG" exports (from AVIF/WEBP/PNG preview tools) actually have the checkerboard baked into opaque RGB pixels rather than real alpha — `rembg` handles these poorly (grainy/dithered alpha, or eats fine detail like cup handles). A dedicated color-matching cutout works much better for these: sample the border's dominant color(s), flood-fill from the border to find background, then apply morphological **closing to the background mask** (not the foreground!) to bridge small gaps caused by lossy-compression noise before testing border-connectivity — closing the wrong side either fails to remove backgrounds at all (background stays fragmented, mask never reaches border) or over-merges noise into the product silhouette. Keep only the largest surviving foreground blob after that. For genuinely solid-color (not checkerboard) flattened backgrounds — e.g. a transparent PNG re-exported as JPG, which fills transparent areas with solid black — the same approach works even better since there's only one reference color and high contrast against the product.

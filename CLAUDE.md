# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page Turkish-language menu website for **Kumyalı Coffee Line**, a coffee shop in Tirebolu/Giresun. It's a static site (no build step, no framework, no package.json) meant to be reached by customers scanning a QR code on their phones — mobile is the primary and most important viewport.

Live deploy: `https://kumyalicoffeeline.netlify.app` (Netlify). GitHub: `zikreddindurmuss/kumyali-coffee-line`.

## Running locally

There's no dev server config in this repo. To preview:

```
python -m http.server 8730 --directory .
```

Then open `http://localhost:8730`. There is no build/lint/test tooling — it's plain HTML/CSS, edit and reload.

**Netlify is not linked to GitHub.** Deploys are manual (drag-and-drop the folder into the Netlify dashboard, or upload via Netlify's UI). A previous attempt to link the GitHub repo failed with "No repositories found" / "we couldn't access the repository" even after granting the Netlify GitHub App access — this is an unresolved GitHub App / Netlify sync issue, not a repo problem (the repo itself is public, non-empty, not archived). If revisiting this, the fix path is usually: uninstall the Netlify GitHub App entirely from `github.com/settings/installations`, then re-run the "Link repository" flow from scratch rather than just re-granting permissions.

## Files

- `index.html` — the entire site: header, hero, menu sections, footer. No templating; every menu item is hand-written HTML.
- `style.css` — all styling, theme tokens as CSS custom properties in `:root` (`--copper`, `--sea`, `--cream`, `--espresso`, etc.), theme is "sea coolness + coffee warmth".
- `logo.png` — brand wordmark, pre-processed to remove its original black background plate (see "Product photo pipeline" below) so it composites directly onto the hero's gradient rather than sitting in a box.
- `Pictures/` — all menu item photos. Flat folder, no subdirectories.
- `netlify.toml` — security headers only (CSP, X-Frame-Options, etc.), no build config.

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

**Exception — Patates Kızartması (fries):** this is the only item with multiple sizes/prices sharing one photo. Instead of repeating the `<figure class="product">` per size, it uses `.product-grid.single-item` (flex, not grid) with one photo and a `.size-options` row of `.size-pill` spans (`Küçük Boy 100 ₺`, etc.). Follow this pattern if another multi-size item shows up — don't duplicate the photo across multiple `<figure>`s.

## Product photo pipeline

All photos in `Pictures/` go through the same normalization before being committed, using PIL (+ `rembg` for background removal when needed):

1. If the source photo doesn't already have real alpha transparency (check: `Image.open(path).convert('RGBA').getpixel((0,0))` — if opaque, or if it's a "baked-in checkerboard" fake-transparency preview export, it needs step 2).
2. Run through `rembg` (`from rembg import remove`) to strip the background.
3. Crop to `getbbox()`, then pad with ~12% margin on a transparent square canvas.
4. Resize to 900×900 (`Image.LANCZOS`).
5. Quantize to reduce file size: `img.quantize(colors=200, method=Image.FASTOCTREE)`, save with `optimize=True`. This typically takes a 300KB+ raw photo down to 30–100KB.

File naming: lowercase kebab-case ASCII (`sikma-nar-suyu.png`, not `sıkma nar suyu.png`) — source files dropped into `Pictures/` by the user often have Turkish characters/spaces/mixed case and get renamed to the ASCII form during processing. Only the final processed files should be committed; raw/unprocessed source uploads should be deleted from `Pictures/` before committing (they're 5-20x larger and unused — cross-check with `grep -oE 'src="Pictures/[^"]+"' index.html` to confirm nothing referenced got deleted).

**Watermarked source images:** users sometimes supply photos with a preview watermark (CityPNG, Shutterstock, Vecteezy, PNGTree seen so far). If the watermark is a separate element (e.g. a text strip outside the subject), `rembg` usually strips it along with the background — check the result. If the watermark is tiled *across* the actual subject (seen with Vecteezy and PNGTree), it can't be cleanly removed — don't ship it; leave that menu item without a photo and flag it instead of using a watermarked image.

## CSS/i18n gotcha: `text-transform: uppercase` + `lang="tr"`

`<html lang="tr">` triggers Turkish-specific case mapping in some browsers: `text-transform: uppercase` turns lowercase `i` into dotted `İ`. This is correct for Turkish words but breaks embedded English/brand text (e.g. "Line" → "LİNE" instead of "LINE"). Fix used throughout: write the English word already fully uppercase in the HTML source (`LINE`, `MILKSHAKE`) instead of relying on the CSS transform to uppercase a lowercase source string. Only apply this fix to non-Turkish words — Turkish words (e.g. "Tirebolu" → "TİREBOLU") should keep the dotted-İ behavior since it's linguistically correct there.

## Known gaps

- **Vanilya (Dondurma)** has no photo — the only source supplied so far had a site-wide tiled watermark that couldn't be removed.

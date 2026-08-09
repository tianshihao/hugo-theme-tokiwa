# hugo-theme-tokiwa — Design Document

This is the authoritative description of the theme's layout system and its
**invariants**. The sections marked *INVARIANT* are load-bearing: a change that
breaks one of them is a regression, not a style choice. If you're about to make
such a change, stop and reconsider.

## 1. Shared type system

- `--list-line-height: 26px` — the rhythm of 1rem list/title text (16px × 1.625).
- `--label-font-size` — derived by JS (`syncLabelFontSize` in
  `site-scripts.html`). It measures the serif font's real glyph height ratio and
  sets `font-size = (2 × --list-line-height) / ratio`, so the label glyph is
  **exactly two list lines tall**. Shared by the posts year label and the tags
  drop-cap label.

## 2. The drop-cap tag (taxonomy cards)

- The tag label is a large float (`float: left`) occupying exactly two title
  lines; titles wrap beside it and below (a word-level drop cap / runaround).
- `line-height: calc(2 * var(--list-line-height))`,
  `height: calc(2 * var(--list-line-height) - 1px)`. The 1px shave is
  deliberate: font-metric drift would otherwise let the float indent the third
  title line.
- Long tags are truncated whole-character with a custom "…" (JS), never shrunk.

## 3. Divider rhythm

Dividers `[data-home-archive-divider]` / `[data-home-tag-divider]`:
`border-top: 1px #9acfbf`, `margin: 8px 0`. On posts they are revealed by JS; on
taxonomy cards each card shows its divider at the top.

> **INVARIANT — vertical alignment**: a taxonomy card's content (visible text
> extent) is vertically **centered between its own divider (above) and the next
> card's divider (below)**, with gap = **GAP = 13px** on both sides.
>
> **The reference axis is the dividers — never the card box.** A card is
> necessarily asymmetric (the divider stack ≈ 17px at the top, variable padding
> at the bottom). Centering anything on the card box breaks the rhythm; this was
> a real regression (the vim highlight briefly centered on the card instead of
> the dividers).
>
> Implemented by `align()` in `site-scripts.html`, which measures the actual
> text extent and re-balances `paddingTop`/`paddingBottom` to hit GAP. The CSS
> `padding: 8px 0` on `.tag-card-content` is only the no-JS baseline.
>
> **Single source**: `GAP` and `PAD` are defined once as `TAXONOMY_DESIGN` at
> the top of `site-scripts.html` and shared by `align()` and `taxMeasureBounds()`
> — never redefine them inline in either function.

### Three content cases

- **1 line** — the title is vertically centered on the drop-cap tag
  (`translateY`). Content extent = tag box (top = tag top, bottom = tag bottom).
- **2 lines** — tag = exactly two title lines, so the extent = tag box.
- **3+ lines** — extent top = tag top, extent bottom = last title line.
- **Column-end card** (no next divider in its column): only the top gap is
  pinned to GAP; the bottom padding is left as-is.

## 4. Vim navigation — a three-level tree

The site is a horizontal tree, navigated with one right hand:

```
Level 1  site nav    (home title first, then Posts/Tags/Categories/Series/About/RSS)
Level 2  the page    (posts list, or taxonomy cards)
Level 3  card titles (tags/categories/series — inside a card)
```

- `j`/`k` — move within the current level (nav item, post, card, or title).
  The first `j` or `k` always selects the top item (never wraps to the last).
- `h` — ascend one level (out): title → card → nav. Returning to the nav
  highlights the nav item for the page you're on (e.g. `Tags` on /tags/).
- `l` — descend one level (in): nav → the selected nav item's page; card →
  its titles. At a leaf (a post or a title) `l` opens the link.
- `gg`/`G` — first/last of the current level (nav → home/RSS; cards, titles
  and posts → their own first/last).
- `Enter` — open / enter the selected item (same as `l` at a leaf).
- `,`/`.` — previous / next page (home & posts pager).
- `/` — focus the search box. Leader keys: `gh` `gp` `gt` `gc` `gs`.
- `Esc`/`Ctrl+C` — exit vim mode. `body.vim-nav` insets the rows.

Keys come from the keyboard via `e.key` — except `,`/`.`, which match the
physical key via `e.code` (a Chinese IME emits `，。` through `e.key`, so only
the physical position is reliable there). All keys are guarded by
`e.isComposing` / `keyCode === 229` so IME composition never hijacks vim — an
IME that is merely enabled (not composing) still reports the physical key.

The model is the same as vim-like file managers (ranger / yazi) and TUI apps:
`j`/`k` siblings, `h` parent, `l` child, `Enter` open — a deliberately familiar
mental model for vim users.

A single article (reached by opening a list item) is the deepest node — it has
no list, so `j`/`k` become smooth scrolling instead of item movement, modelled
as a car at constant velocity from the very first frame (the first movement is
small, not a burst): a quick tap glides one fixed `READ_TAP = 300px` chunk,
holding keeps the same constant speed and stops instantly on release (the loop
is time-based, so the speed holds even if the frame rate wobbles). `d`/`u`
scroll at `READ_FAST = 5×` the j/k speed (like Ctrl+d/Ctrl+u). `gg`/`G` jump
instantly to the top / bottom.

## 5. Highlight (vim cursor) — align to dividers

- Drawn as a `::before` overlay (`z-index: -1`), positioned with
  `--hl-top` / `--hl-bottom` set by JS (`taxMeasureBounds`).
- **INVARIANT**: on taxonomy cards the highlight must mirror the divider rhythm.
  `taxMeasureBounds()` measures the own divider's offset from the card top
  (`divLine`) and places the box `GAP − PAD` from each divider, where `PAD = 2`
  is the highlight's breathing room beyond the content's own gap → **11px** from
  each divider (the next divider sits at the same offset below the card's bottom
  edge, since consecutive pins are flush and share the divider CSS). This makes
  the box **identical for every card** regardless of content height, and keeps
  the divider lines outside the box. Do not re-introduce card-box centering.
- Home/posts use the same overlay pattern on `[data-home-post-item]`.

## 6. File map

- `layouts/partials/site-scripts.html` — vim navigation, `align()`,
  `syncLabelFontSize()`, `taxMeasureBounds()`.
- `layouts/_default/baseof.html` — `<style>` blocks for the type system, divider
  rhythm, vim breathing room, and highlight overlays. The vim/highlight CSS must
  stay OUTSIDE the `$oneScreen` conditional — it runs on every page, not just
  home/posts.
- `layouts/_default/terms.html` (site root) — unified taxonomy template; tags get
  the two-column `.waterfall`.

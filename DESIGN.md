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
Level 2  the page    (posts list, taxonomy term post list, or taxonomy cards)
Level 3  card titles (tags/categories/series — inside a card)

A taxonomy **term** page (e.g. /tags/c++/) is a one-screen post list, exactly
like home / posts: `_default/taxonomy.html` renders every post in the term as a
`[data-home-post-item]` row, so the list cursor (`visibleItems()`) drives
`j`/`k`/`G`/Enter and `,`/`.` pages through — identical to home. Only the
overview pages (the tags / categories / series waterfalls) navigate cards.
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
- `Esc`/`Ctrl+C` — exit vim mode. (`body.vim-nav` is purely a class switch; the
  rows keep their original flush layout — no horizontal inset.)

### Nav live-preview (first-level pages)

On a **first-level** page — a page that is exactly one of the nav destinations
(home / posts / tags / categories / series / about) — the nav is a live
preview: `j`/`k` in the nav **enters the selected page immediately**, no `l`
needed. The destination loads with `?nav=1`, and that page re-enters the nav
and drops the param (`history.replaceState`), so a held or repeated `j`/`k`
tabs straight through the pages. On those pages `l`/Enter in the nav moves the
cursor into the main content **and selects its first item** (first card on a
taxonomy overview, first post row elsewhere — the mirror of `h` selecting the
current page's nav item); the current page's nav item stays purple-red (the
template's `text-medium-red-violet-600`), so "where am I" is always visible
while navigating content. RSS / external / feed links are not
pages — they only highlight, `l`/Enter still opens them (`navPreview()` guards
on `.xml` / `http`). On deeper pages (articles, taxonomy term pages) the nav
keeps the classic behavior: `j`/`k` move the highlight and `l`/Enter navigate
there. Because this is a real navigation (not an in-place swap), every page
renders fully — its own one-screen pagination / waterfall / pager — with no
state to re-initialize.

The nav also takes priority over the reading view: once `h` has entered the
nav, `j`/`k`/`gg`/`G` operate on the nav cursor even on a reading page (e.g.
/About/), instead of scrolling the article.

Keys come from the keyboard via `e.key` — except `,`/`.`, which match the
physical key via `e.code` (a Chinese IME emits `，。` through `e.key`, so only
the physical position is reliable there). All keys are guarded by
`e.isComposing` / `keyCode === 229` so IME composition never hijacks vim — an
IME that is merely enabled (not composing) still reports the physical key.

The model is the same as vim-like file managers (ranger / yazi) and TUI apps:
`j`/`k` siblings, `h` parent, `l` child, `Enter` open — a deliberately familiar
mental model for vim users.

Vim mode is remembered independently of the mouse: the `vim-active` highlight
persists even while the cursor moves, and the mouse is never blocked — hovering
a link still shows its hover sweep/color alongside the keyboard highlight, and
clicks work normally. Only the keyboard exits vim (`Esc` / `Ctrl+C`), or a vim
action like `j`/`k`/`h`/`l`/`gg`/`G` moves the highlight while it stays active.
`enterVim()` blurs the focused element so a previously clicked link's `:focus`
tone can't linger, but no pointer event touches the vim state.

Holding `j`/`k` in a list (nav / posts / taxonomy) advances the selection at a
constant cadence — one item every `LIST_REPEAT_MS = 180ms` — driven by a rAF
accumulator, not the OS key repeat (which would stall ~500ms after the first
move). The first press moves one item immediately, then the loop keeps stepping
until release, which stops instantly. Inside an article the hold is a constant
velocity instead (see below), so a list is item-to-item and an article is
pixel-by-pixel.

On the waterfall pages (tags / categories / series) the selection glides at the
edges: selecting the first card from the second never forces a jump to the top,
but pressing `k` again on the first card (or holding `k`) smooth-scrolls the
remaining distance to the page top, and symmetrically `j` on the last card
smooth-scrolls to the bottom (`cardCursor`'s `onEdge` → `window.scrollTo`). The
selection stays pinned at the edge while the page glides.

Every level's selectable siblings are either a **list** (nav links, posts rows,
card titles) or a **card** (the taxonomy waterfall). That single-level model is
the code, not just the design: all four panes run the **same cursor engine**
(`makeCursor` in `site-scripts.html`), differing only in `items()` (where the
siblings come from) and `onSelect()` (how the highlight is painted):

| pane         | items()                                  | highlight        |
|--------------|------------------------------------------|------------------|
| site nav     | `navItems()` (cached, re-query if empty) | pill             |
| posts list   | `visibleItems()` (live)                  | measured overlay |
| cards        | `taxCards()` (live)                      | measured overlay |
| card previews| `taxPreviews(active card)` (live)        | pill             |

`move` / `jump` / `enter` / `clear` are shared and cannot drift apart: first
press starts at the top, edges clamp (an already-at-edge press is a no-op that
doesn't re-scroll), `gg`/`G` jump first/last. `activeCursor()` picks the pane
(nav → card preview → cards → list). The only custom bits left are the pane
transitions: `h` enters the nav (selecting the current page's item), `l`
descends into a card's previews, `h` ascends back to the card (preview cleared,
card re-selected), and `exitVim()` clears all four panes.

A single article (reached by opening a list item) is the deepest node — it has
no list, so `j`/`k` become smooth scrolling instead of item movement, modelled
as a car at constant velocity from the very first frame (the first movement is
small, not a burst): a quick tap glides one fixed `READ_TAP = 300px` chunk,
holding keeps the same constant speed and stops instantly on release (the loop
is time-based, so the speed holds even if the frame rate wobbles). `d`/`u`
scroll at `READ_FAST = 5×` the j/k speed (like Ctrl+d/Ctrl+u). `gg`/`G` jump
instantly to the top / bottom.

A page is "reading" (`isReading()`) only when it is a real single article —
detected by `article .c-rich-text`, which only `single.html` renders.
`terms.html`, `list.html` and 404 also emit an `<article>`, but those are
content wrappers; treating them as reading would hijack `j`/`k` into scrolling
instead of navigating (e.g. `/categories/` with nothing categorized has an
empty article and no cards).

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

- `layouts/partials/site-scripts.html` — vim navigation (`makeCursor`),
  one-screen pagination (`buildPagesByHeight` / `pinBottomRails`), taxonomy
  centering (`align()`), label font (`syncLabelFontSize()`), highlight
  (`taxMeasureBounds()`), reading-page detection (`isReading`).
- `layouts/_default/baseof.html` — `<style>` blocks for the type system, divider
  rhythm, vim breathing room, and highlight overlays. The vim/highlight CSS must
  stay OUTSIDE the `$oneScreen` conditional — it runs on every page, not just
  home/posts.
- `layouts/_default/terms.html` (site root) — unified taxonomy overview template;
  tags get the two-column `.waterfall`.
- `layouts/_default/taxonomy.html` (theme) — unified taxonomy **term** template:
  all posts in the term as `[data-home-post-item]` rows inside
  `[data-home-posts-container]`, so every term page is one-screen like home.
- `layouts/partials/one-screen.html` — single source of truth for "is this a
  fixed one-screen page?" (home / posts section / taxonomy term). Used for both
  the `body.home-one-screen` class and the footer block default — the block
  default is a separate template that only receives `.`, so it can't see a
  `$oneScreen` variable defined at the top of baseof.

## 7. Footer — one shared home-pager

The `<footer data-home-footer>` block is unified: **home, posts and every
taxonomy term page** (tags / categories / series, e.g. /tags/c++) share the
home-pager. It lives as the baseof default, gated by the shared
`partial "one-screen.html" .` helper, instead of being duplicated in
`index.html`, `posts/list.html` and `_default/taxonomy.html`. Other page types
override the block: article → `page-footer` (prev/next + related + disqus, where
the disqus separator line + container are gated on `disqusShortname` so an
unconfigured site never shows two stacked `<hr />`s — only the single closing
line remains), generic list → `pagination`. Taxonomy overview (`terms.html`) and
404 render nothing (the one-screen condition is false; an *empty* `define
"footer"` does **not** override a `block` default in Hugo — only a non-empty one
does, so keep the gating in baseof).

The pager's one-screen layout (`body.home-one-screen [data-home-pager]`, in
baseof) is a 5-column grid — `first`/`prev` on the left, page numbers in the
flexible middle, `next`/`last` on the right — and **the arrows never move**.
Every grid cell is flex-centered, and the arrows' `-bottom-1` icon shift is
neutralized, so the page-number text and the chevrons share the same vertical
center. The page numbers are spread apart with `gap: 1.25rem` (the "too
concentrated" fix) and centered in the middle band. The `...` ellipsis (a
JS-built `li.page-item.disabled`) is recolored to the page-number teal
`#028760` (not the disabled gray) and given a `translateY(-0.25em)` optical
lift — the period dots sit on the text baseline while digits rise above it, so
the dots' ink center is ~0.25em lower; lifting them puts the ellipsis on the
digits' optical center. The `data-home-pager-hr` top line and the arrow SVGs
are the design constants — never restyle or move them.

## 8. Fixed chrome — header & bottom bar never move

On every non-article page (home / posts / tags / categories / series) the
header is immobile and sits at the exact same spot, so switching pages never
shifts it. The shell's top padding and the header's sticky top are both
`--chrome-top`, so the header's natural position equals its sticky top and it
never moves while scrolling. On the one-screen pages (home / posts / taxonomy
term pages) the header is plain static (they never scroll); on the scrolling
pages it's sticky with top = the same offset. `--chrome-top` is
`calc(var(--aside-gap) + 6px)` — the header's gap from the top (40px) mirrors
the aside's gap from the bottom (34px), slightly larger, and stays in sync if
`--aside-gap` changes. The bottom bar keeps the same `--aside-gap` rhythm: on
the one-screen pages (home / posts / taxonomy term pages) the aside and footer
are `position: fixed; bottom: var(--aside-gap)` via `pinBottomRails()`; on the
scrolling pages the aside is `sticky` at the same bottom gap. The footer is
only ever pinned on the one-screen pages — on articles it flows right after
`main` (in the right column), and its bottom edge lands at `--aside-gap` when
scrolled to the end (the shell's `padding-bottom: var(--aside-gap)`), matching
the first-level pages. The header is `position: sticky; top: 0` only on
articles (`.is-article`), where it is allowed to ride up while reading.

The one-screen pages (home / posts / taxonomy term pages) are fitted by
measurement, not hardcoded: `buildPagesByHeight()` reads the real geometry
(posts container top, footer height, `--aside-gap`) and groups the list into
pages that fit the available height, `applyPostsHeightLimit()` caps the list's
height, and `pinBottomRails()` freezes aside + footer as
`position: fixed; bottom: var(--aside-gap)`. Because everything is measured
(`getBoundingClientRect`), raising the header or resizing the viewport re-fits
automatically — nothing is hardcoded. A term page's `<h1>` sits above the
list and simply reduces the available height.

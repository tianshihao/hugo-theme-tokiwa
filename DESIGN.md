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
- `--list-gutter: 1.5rem` — the **base** `ol` text indent (clears `"1."` ≈ 13.4px
  + the gap). The numbers' **left edges sit ON the body's left line** (box
  padding 0, marker `left: 0`), so top-level lists are not indented.
- `--list-gap: 0.665rem` — the ONE marker↔text gap (≈ 10.6px). The `ul` text
  hangs at `dot + gap`; the `ol` gap is kept constant by sizing the indent to
  the widest number.
- `--list-symbol-size: 0.4em` — the `ul` dot's diameter (left edge on the line).
- **Digit-aware `ol` indent (JS, `articleLists` in `site-scripts.html`):** a
  ≥10-item `ol` gets `--ol-indent` = its widest `"N."` width + `--list-gap`
  (set per-list, measured at the counter font), so the gap stays constant for
  the widest number — `"10."` gets the same gap as `"1."`. 1-digit lists use
  the base `--list-gutter`. (Real lists never reach 3 digits.)
- **INVARIANT — `ol` numbers are LEFT-aligned at the body's line, never
  right-aligned.** Right-aligning them to a fixed period column is what makes
  the list look "aligned inside" (user rejects it). The gap is instead kept
  constant by the digit-aware text indent.

## 2. The drop-cap tag (taxonomy cards)

- The tag label is a large float (`float: left`) occupying exactly two title
  lines; titles wrap beside it and below (a word-level drop cap / runaround).
- `line-height: calc(2 * var(--list-line-height))`,
  `height: calc(2 * var(--list-line-height) - 1px)`. The 1px shave is
  deliberate: font-metric drift would otherwise let the float indent the third
  title line.
- Long tags are truncated whole-character with a custom "…" (JS), never shrunk.
- **The teal vim pill is this same 2-line label box** (sized by the label's
  line box — glyph + symmetric ~8px leading — never the card's highlight box).
  It only translates; it is never resized. Full pill invariant in §5.

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
Level 1  site nav    (home title first, then Posts/Tags/Categories/Series/About)
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
- `h` — ascend one level (out): title → card → nav; from a **leaf** (an
  article) or a taxonomy **term** page it goes back to the page it came from
  (a list or the cards) and restores the cursor — see "Vim return" below.
  Returning to the nav highlights the nav item for the page you're on (e.g.
  `Tags` on /tags/).
- `l` — descend one level (in): nav → the selected nav item's page; card →
  its titles. At a leaf (a post or a title) `l` opens the link.
- `gg`/`G` — first/last of the current level (nav → home; cards, titles
  and posts → their own first/last).
- `Enter` — open / enter the selected item (same as `l` at a leaf).
- `,`/`.` — previous / next page (home & posts pager).
- `/` — focus the search box. Leader keys: `gh` `gp` `gt` `gc` `gs` `ga`.
- `Esc`/`Ctrl+C` — exit vim mode. (`body.vim-nav` is purely a class switch; the
  rows keep their original flush layout — no horizontal inset.)
- **RSS is not in the nav.** It lives in the **aside** as a social-style icon
  (`layouts/partials/svg/rss-line.svg`, shown via the same `social-follow`
  group as the other `[params.social]` links). Clicking it is **not a link** —
  it **copies the feed's XML to the clipboard** (a delegated `[data-rss-copy]`
  handler in `site-scripts.html` fetches the feed and writes it via
  `navigator.clipboard`, with a brief teal/red colour flash for
  success/failure). The feed URL is `site.Home.OutputFormats.Get "RSS"`
  (`/index.xml`).

### Vim return — h on a leaf restores where it came from

The leaves (articles) and the taxonomy term pages sit one level below a list
(home / posts / term) or a card (tags / categories / series). Leaving one of
those (`l`/Enter, or a mouse click on the item) is a **real navigation**, so
the vim state would be lost. Before leaving, `openActive()` / `taxOpen()` /
the click handler save a return context — which page (`path`), the list page
number and the row's link, or the card index (and preview index) — into
`sessionStorage` (`tokiwa.vimReturn`). The saved `path` is **`effectivePath`**
(the page actually shown), not `window.location.pathname`: the nav live-preview
shows home / posts / the taxonomies on a shared URL, so the URL would record
the wrong page to return to (clicking a home row while /posts/ is the URL used
to return to /posts/). `effectivePath` is **normalized** (no trailing slash,
home = `/`) — its initial value is `normalizePath(location.pathname)`, the same
form `applyPreview` uses, so a saved path always matches the URL form used in
comparisons. A trailing slash in the saved path silently breaks `h` on a real
page (e.g. a term page): `returnToSource`/`applyReturnRestore` compare it
against the slash-less URL, never match, and `returnToSource` then navigates to
`path + '/'`, i.e. a double-slash URL — the page reloads with the context still
pending and `h` loops forever. Both functions normalize defensively for that.
On the target, `h` (`returnToSource()`) navigates back one level.
**A first-level page (home / posts / tags / categories / series / about) is only
ever shown inside home — returning to one lands on home (`/`), never on that
page's own URL**, so those pages don't appear during navigation. The home page
then loads the saved page's content into the right pane via the same SPA swap
(`applyReturnRestore()` → `previewNavTo()`), switches to the saved page number,
re-selects the exact row / card / card-preview that was highlighted, and
re-enters vim mode. **Only an article carries a saved context** (its parent
isn't derivable); a **taxonomy term page derives its parent from its own URL**
(`taxonomyFromTermPath()`: `/tags/c++` → `/tags` + the C++ card via the tag
link's `href`), so `h` from a term page returns to home with the taxonomy
overview loaded and the exact card re-selected — no saved context needed. A
restored return is a **committed page**, so `restoreCursor()` also calls
`updateNavCurrent()` afterwards — otherwise the nav item for the returned-to
first-level page would show no purple-red (the SPA swap only loads the content).
The
context survives the real navigation but is per-tab and cleared once applied —
so a later `h` on the same page goes to the nav, not back here. Without a
context (direct / search / nav navigation) `h` on a leaf keeps the classic
behavior: enter the nav. Leaving for a different top-level page via the nav
(`openNavItem()`) clears a pending context, since that navigation supersedes
the h-back.

**First-level pages also redirect when visited directly.** A direct visit to
`/posts/`, `/tags/`, `/categories/`, `/series/` or `/about/` (address bar,
external link) is intercepted by a small script in `baseof`'s `<head>` (it runs
before the body renders, so the standalone page never flashes) and redirected
to `/?to=<path>`; home's `tokiwaLoadToPage()` then loads that page's content
into the right pane via the same SPA swap and commits it (the purple current-
page indicator), consuming `?to=` via `replaceState` so the URL becomes plain
home. This list MUST match `FIRST_LEVEL_PATHS`. Taxonomy **term** pages (e.g.
`/tags/c++/`) are real pages and are never redirected.

**Home itself is a real link.** Clicking the site title (`data-site-title`) or
pressing `gh` navigates to `/` for real (a normal logo→homepage click) —
`openNavItem()` special-cases the home target. The other first-level items
(posts / tags / categories / series / about) still commit **in place** via the
SPA preview; only home is a real navigation.

### Nav live-preview (first-level pages)

On a **first-level** page — a page that is exactly one of the nav destinations
(home / posts / tags / categories / series / about) — the nav is a live
preview: `j`/`k`/`gg`/`G` moving the cursor **loads the selected page's main +
footer into the right pane via fetch**, in place — not a navigation, so the URL
never moves (every first-level page shares the entry URL, e.g. `/`).
> **INVARIANT: the home title link's href is ALWAYS the relative `/`**
> (`site-header.html` / `page-header.html` — never the absolute `BaseURL`).
> `isFirstLevel()` and `navItemForHref()` compare nav hrefs to the pathname, so
> an absolute href (e.g. `https://site/`) makes home look like an external page:
> `isFirstLevel()` returns false on `/`, the nav live-preview dies on the home
> page of the deployed site, and `gh` / a home click fall back to a real
> navigation. Only a relative `/` keeps home a first-level page in production
> (the menu links are already root-relative). This was a real deployed-only
> regression.
Each move
bumps a `previewRequestId`; a stale response is dropped, so rapid j/k always
settles on the last selection. **On first open the cache is pre-warmed**:
`prefetchFirstLevel()` (fired from the boot block on a first-level page) fetches
every first-level nav destination — home + posts / tags / categories / series /
about — and `cachePreviewPage()` stores the parsed main/footer in
`previewCache` (keyed by normalized path), **without applying it**, so the
on-screen page is never disturbed. `previewNavTo()` then applies straight from
the cache, so every j/k/gg/G move is instant (no network wait per move); a miss
still fetches and warms the cache for the next move. The prefetch is
fetch+cache only — the `previewRequestId` stale-drop and the purple-red commit
rules are unchanged. The swap re-boots the target's machinery through
the re-callable modules `oneScreenPage.boot()/.teardown()` (home / posts / term)
and `taxonomyCards.boot()` (tags / categories / series) — refactored from
one-shot IIFEs for exactly this — so the paginated list, the pinned footer and
the card dividers stay correct. Each `oneScreenPage.boot()` starts the list at
**page 0** (`currentPage = 0`): the paginator must not carry a previous page's
position across a swap, so home on page 5 loading posts shows posts page 1, not
page 5 (a later vim-return `restoreCursor()` can still bring back a saved page
number). The list-cursor config (`currentListConfig()`)
is re-derived from the shown content (not the URL), so the vim highlight and
Enter-open follow home vs. the posts archive; and `effectivePath` tracks which
page is shown so `h` from main returns to that page's nav item. Crucially,
**j/k/gg/G only preview — they never turn a nav item purple-red**: the purple
current-page indicator is set only on a commit (`l`/Enter via
`enterMainContent()`, or a mouse click), never while the cursor is merely
moving (`updateNavCurrent()` is called from those commits, not from
`applyPreview()`). On those pages `l`/Enter moves the cursor into the main
content **and selects its first item** (the mirror of `h` selecting the
current page's nav item). The template never colours the nav purple (the old
`{{ if in (Permalink) (URL) }}` class is gone) — `updateNavCurrent()` is the
only source, called from a commit, so "where am I" (the purple) only ever
appears on a committed page. `h` back to the nav cancels it
(`navEnter()` → `clearNavCurrent()`): the purple is shown ONLY while IN the
content, never while navigating the nav.
External / feed links are not
previewable pages — they only highlight, `l`/Enter still opens them. Articles (about) are
not previewable (the swap is skipped; the nav only highlights and `l`/Enter
navigates there). On deeper pages (taxonomy term pages) the nav keeps the
classic behavior: `j`/`k` move the highlight and `l`/Enter navigate there.

Mouse clicks on the nav behave the same way: on a first-level page, clicking a
nav link (or the home title) runs the same fetch + swap, so the mouse and vim
are two inputs on one engine. The click updates the purple-red current-page
indicator and blurs the clicked link so the indicator shows immediately
(matching a real page load). And if the vim cursor was in the nav, the click
**moves it into the loaded content and selects its first item** — exactly like
`l`/Enter — so the next `j`/`k` move the content, not the nav (a plain click
outside vim mode just shows the content without activating a cursor).
External links and non-previewable targets (articles / 404) fall back to a
real navigation; clicks from deeper pages navigate normally.

The **g-leader routes through home** — home is the SPA shell, and a
first-level page is only ever shown inside home's pane, so
`gh`/`gp`/`gt`/`gc`/`gs`/`ga` behave identically from any page:
- `gh` — a real navigation to `/` (the logo → homepage click);
- `gp`/`gt`/`gc`/`gs`/`ga` — already on home → the target is previewed into
  the pane in place (the shared `openNavItem()` / `previewNavTo()` engine);
  from a deeper page (article / term) → a **real navigation to `/?to=<path>`**,
  reusing the same redirect `tokiwaLoadToPage()` handles for direct
  first-level visits: home's clean chrome loads (the site nav, header, rail,
  aside — all home's) and the target is SPA-previewed into the pane, committed
  (purple) and the `?to=` consumed via `replaceState` so the URL becomes plain
  home. Only the pane ever shows the SPA; nothing of the originating article
  lingers. So every way of "selecting" a first-level page — j/k/gg/G preview,
  `l`/Enter commit, mouse click, and the g-leader — lands in the same pane.

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

| pane          | items()                                  | highlight         |
| ------------- | ---------------------------------------- | ----------------- |
| site nav      | `navItems()` (cached, re-query if empty) | pill              |
| posts list    | `visibleItems()` (live)                  | measured overlay  |
| cards         | `taxCards()` (live)                      | measured overlay  |
| card previews | `taxPreviews(active card)` (live)        | — (card box only) |

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
small, not a burst): holding keeps the same constant speed, and **releasing
stops instantly — no tap glide** (a quick j/k press moves only the ~1-2 frames
it was held, then stops, exactly like `d`/`u`; the loop is time-based, so the
speed holds even if the frame rate wobbles). `d`/`u` scroll at `READ_FAST = 5×`
the j/k speed (like Ctrl+d/Ctrl+u) — same stop-on-release behavior, different
distance per key. `gg`/`G` jump instantly to the top / bottom.

A page is "reading" (`isReading()`) only when it is a real single article —
detected by `article .c-rich-text`, which only `single.html` renders.
`terms.html`, `list.html` and 404 also emit an `<article>`, but those are
content wrappers; treating them as reading would hijack `j`/`k` into scrolling
instead of navigating (e.g. `/categories/` with nothing categorized has an
empty article and no cards).

The article header (`single.html` → `page-header.html`) carries the TOC and
**no site navigation** — the nav block (`site-navigation.html`, which also
holds the search box) is replaced by a single "← Back" link
(`[data-article-back]`). Clicking it calls `window.tokiwaArticleBack()`
(site-scripts), which is exactly the `h` key on a leaf: `returnToSource()`
navigates back to the page the article came from and restores the cursor there;
with no saved context (e.g. a direct visit / search) it falls back to home.
`fastsearch.js` is guarded so its absence of `#searchInput` / `#searchResults`
on articles never throws.

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
- **Fonts stay original while moving**: the teal overlay is the only cursor
  indicator during j/k — element fonts never change colour. The card carries
  its box AND its tag label carries a teal pill (the second-layer highlight)
  while the card is selected; the preview TITLE keeps a teal pill while the
  cursor is on it. Only `l` to the next layer changes things (below).
- **The pill is the LABEL's own box — sized by the label, never by the box**
  (`syncTagPill()`). The teal pill is the tag label's 2-line box (label glyph +
  symmetric ~8px leading), so it is always **52px** — the label's line box
  (`2 × --list-line-height`). **INVARIANT: the pill is never resized. DO NOT
  stretch it to the highlight box.** The earlier "stretch + top-align" version
  made the pill's top equal the box's top while its bottom poked below the
  box's bottom — inflating the link changes the label's line-box layout, so the
  top computed from the stale natural position was wrong, and the pill was no
  longer centered. The pill only ever **translates** (a pure vertical offset),
  never changes size.
- **The pill translates to center on the highlight box ONLY while the titles
  stay BESIDE the label (1–2 lines).** For a 2-line card the box is only a few
  px taller than the pill, so the shift is tiny; 1-line and 2-line both center
  (their titles never run below the label, so the pill cannot collide with
  them). **For 3+ lines the pill keeps its natural position ON the label** — the
  titles then wrap below the label, and centering a small pill on a tall box
  floats the tag name over those titles (a real regression the user rejected;
  `syncTagPill()` detects it by comparing the titles' bottom to the label's
  bottom and skips the translate). The shift is applied to `.tag-label` (the
  pill's container), not the `<a>`: `.tag-label`'s `overflow:hidden` would clip
  a relatively-offset link whose background pokes below its box, and a relative
  offset never reflows the titles. Deselection (moving to another card) and
  vim-exit reset the shift.
- **Only `l` to the NEXT layer turns the PREVIOUS layer's font purple-red**
  (medium-red-violet, the same as the nav's current-page indicator): nav →
  content turns the nav item purple (set only by `updateNavCurrent()` on a
  commit — the template never colours it, see §4); card → previews turns the
  card's tag label purple-red AND drops its teal pill (a `preview-open` class that `taxDescend()` adds and `taxAscend()` /
  `exitVim()` remove). Ascending (`h`) restores the label's original font
  colour and its pill.

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
  fixed one-screen page?" (home / posts section / taxonomy term / about / 404).
  Used for both
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
unconfigured site never shows two stacked `<hr />`s, and there is **no trailing
closing line** — the footer's content reaches its own bottom edge, so its
bottom sits at `--aside-gap` (the shell's `padding-bottom`; see §8), generic
list → `pagination`, about (`layouts/about/list.html`) →
a non-empty override rendering nothing — About is a fixed one-screen page with
**no footer**. Taxonomy overview (`terms.html`) renders nothing (the one-screen
condition is false; an *empty* `define "footer"` does **not** override a
`block` default in Hugo — only a non-empty one does, so keep the gating in
baseof). 404 is one-screen too but also overrides the footer to nothing (an
*empty* `define "footer"` in `404.html`) so the home-pager never shows on it.

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

**Article footer prev/next (`page-footer.html`):** the two anchors sit in a
`flex justify-between` row, each wrapped in an **always-present** `div` — a
missing neighbour renders an **empty box** on that side instead of collapsing,
so `justify-between` always pins the `data-article-prev` box to the far left
and the `data-article-next` box to the far right. The first (newest) article
shows only the left prev arrow at the left; the last (oldest) article shows
only the right next arrow at the right. Never merge the two wrappers back into
direct children of the flex row — a lone flex child loses the side it belongs
to.

**When the ellipsis appears** (`buildNumberButtons()`): the pager may occupy
at most `MAX_SLOTS` number slots (including ellipsis placeholders), and the cap
is **computed from the actual width** (`computeMaxSlots()`, bounded 5–9): the
pager grid reserves the arrows' columns and gaps, each slot is a 2-digit page
number plus the 1.25rem gap, so the widest viewport shows the most numbers and
a narrow one starts collapsing. If the page count is at most `MAX_SLOTS`, every
number is shown — no ellipsis, ever (turning one number into an ellipsis would
still leave the same slot count, just with less info). The ellipsis hides
pages only once there are MORE pages than slots (the trigger is
`total == MAX_SLOTS + 1`, not `total == MAX_SLOTS`), so a 4-page archive always
reads `1 2 3 4`. When hidden, first and last are always shown, a window around
the current page fills the rest, and the window shrinks until everything fits —
the current page stays visible on every page. The pager's horizontal layout CSS
lives OUTSIDE the `min-width: 768px` media query (it applies wherever the pager
is shown), so the page numbers never fall back to the vertical `ul`/`li`
stacking in a narrow view.

## 8. Fixed chrome — header & bottom bar never move

On every page — the one-screen pages (home / posts / term / about / 404), the
scrolling overviews (tags / categories / series) **and the article pages** —
the header is immobile and sits at the exact same spot, so switching pages
never shifts it. The shell's top padding and the header's sticky top are both
`--chrome-top` (applied to the shell on **every** page now, articles included),
so the header's natural position equals its sticky top and it never moves
while scrolling. On the one-screen pages the header is plain static (they
never scroll); on the scrolling pages — including articles — it's sticky with
top = the same offset. `--chrome-top` **equals** `--aside-gap` (`--chrome-top:
var(--aside-gap)`) — the header's gap from the top and the aside's gap from the
bottom are the exact same single value (34px), so the top and bottom chrome are
perfectly symmetric and always stay in sync. The bottom bar keeps the same
`--aside-gap` rhythm: on
the one-screen pages (home / posts / taxonomy term pages) the aside and footer
are `position: fixed; bottom: var(--aside-gap)` via `pinBottomRails()`; on
about / 404 the same spot is reached with the flex layout + the `bottom: auto`
guard (below), since the fit machinery never runs there; on the scrolling pages
the aside is `sticky` at the same bottom gap. The footer is
only ever pinned on the one-screen pages — on articles it flows right after
`main` (in the right column), and its bottom edge lands at `--aside-gap` when
scrolled to the end (the shell's `padding-bottom: var(--aside-gap)`), matching
the first-level pages. The header used to be `position: sticky; top: 0` on
articles, but now it is sticky at `--chrome-top` like every scrolling page, so
the site title ("笔记本子") never moves. **The site title is an `h1` on the
article header too** (`page-header.html`), so it is the exact same size as on
every other page — and the article title carries `.article-title`, which steps
one Tailwind size above `h1` (`2.25rem` mobile / `3rem` on lg) so it stays the
dominant heading. Net effect: "笔记本子" is pixel-identical in size and
position on every page.
> **About and 404 are one-screen but the fit machinery never runs there** (they
> have no posts to paginate, so `boot()`'s `capture()` fails and
> `pinBottomRails()` never pins the aside). The aside would then fall back to
> its scrolling-page `position: sticky; bottom: var(--aside-gap)`, and on short
> content that sticky bottom offset lifts it a full `--aside-gap` (34px) above
> where home / posts put it. Guard: `body.home-one-screen aside[data-home-aside]
> { bottom: auto !important }` drops the sticky bottom on one-screen pages so
> the flex layout (`align-self: end` + the shell's `padding-bottom`) pins the
> aside at the same spot on home / posts / about / 404 alike.

The one-screen pages (home / posts / taxonomy term pages) are fitted by
measurement, not hardcoded: `buildPagesByHeight()` reads the real geometry
(posts container top, footer height, `--aside-gap`) and groups the list into
pages that fit the available height, `applyPostsHeightLimit()` caps the list's
height, and `pinBottomRails()` freezes aside + footer as
`position: fixed; bottom: var(--aside-gap)`. Because everything is measured
(`getBoundingClientRect`), raising the header or resizing the viewport re-fits
automatically — nothing is hardcoded. A term page's `<h1>` sits above the
list and simply reduces the available height.

### Article body drops the markdown h1

On article pages the header already shows the title, so the body hides the
markdown level-1 heading it would otherwise render (`body.is-article
.c-rich-text h1 { display: none }`). The markdown files are shared with other
sites and keep their own `#` heading — only this rendering drops it. Per
request this hides **all** h1 in the body (sections also written with `#` are
h1 too and are hidden as well); `##`-level headings are untouched. The rule is
scoped to `.is-article .c-rich-text`, so list / taxonomy / about pages keep
their own headings.

## 9. Color system — design tokens

Every color the theme uses is defined **once, by name**, in
`static/css/tokens.css` (`:root { --<name>: <value> }`) and referenced
everywhere as `var(--<name>)`. There are no stray hex/rgb literals left in the
templates or stylesheets (the only exceptions are functional shadows
`rgba(0,0,0,.05/.1)` and `transparent`).

**Naming**: colors that already had a name keep it — `--eucalyptus-300/400/600`,
`--gray-*`, `--red-600`, `--medium-red-violet-400/600`. Colors without a
prior name got functional names: `--teal-hover`, `--teal-dark`, `--teal-pale`,
`--teal-faint`, `--color-*`.

The full palette, grouped by **where each color appears** (scenario). Each
table lists: the original scale name it came from (if any) · the design token ·
the hex value · what it is used for.

> **Pairing model.** Interactive colors come in NORMAL ↔ HIGHLIGHT pairs —
> the link pair (9.3) and the magenta pair (9.6). Vim highlight is a separate
> translucent pair (9.8). Scale/functional colors (decorative, borders,
> backgrounds, text hierarchy, status) are not pairs. A token with zero
> `var()` references is dead — delete it.

#### 9.1 Page chrome — backgrounds & surface

| Original | Token              | Hex       | Use                |
| -------- | ------------------ | --------- | ------------------ |
| gray-50  | `--color-bg`       | `#F7FAFC` | page background    |
| white    | `--color-surface`  | `#FFFFFF` | cards / white      |
| gray-100 | `--color-gray-100` | `#EDF2F7` | light gray surface |

#### 9.2 Text hierarchy

| Original   | Token                | Hex       | Use                         |
| ---------- | -------------------- | --------- | --------------------------- |
| gray-700   | `--color-text`       | `#4A5568` | body text                   |
| gray-600   | `--color-text-muted` | `#718096` | secondary text              |
| gray-400   | `--color-text-faint` | `#A0AEC0` | placeholder                 |
| — (custom) | `--color-heading`    | `#01513A` | h1/h2 headings (deep green) |

#### 9.3 Eucalyptus teal — link pair (NORMAL ↔ HIGHLIGHT)

| Original       | Token              | Hex       | Use                                                            |
| -------------- | ------------------ | --------- | -------------------------------------------------------------- |
| eucalyptus-600 | `--eucalyptus-600` | `#028760` | **link normal**, emphasis, strong/b, tag underline, RSS copied |
| —              | `--teal-hover`     | `#49BEB7` | **link hover/focus**, table borders, placeholder, clover hover |

#### 9.4 Eucalyptus teal — decorative scale

| Original       | Token              | Hex       | Use                                                            |
| -------------- | ------------------ | --------- | -------------------------------------------------------------- |
| eucalyptus-400 | `--eucalyptus-400` | `#4EAB90` | ❧ separator, list bullets, TOC, labels                         |
| eucalyptus-300 | `--eucalyptus-300` | `#9ACFBF` | dividers (hr, tag divider), pagination active, gradients       |
| —              | `--teal-pale`      | `#B6E5E2` | borders + inline-code tint (**absorbs old slate-300 #CBD5E0**) |

#### 9.5 Text selection

| Original  | Token          | Hex       | Use                                                       |
| --------- | -------------- | --------- | --------------------------------------------------------- |
| java-700  | `--teal-dark`  | `#2C726E` | `::selection` background (behind selected text)           |
| java-100* | `--teal-faint` | `#EDF9F8` | `::selection` text color (also the pale teal tint in 9.1) |

\* `java-100` appears in a theme comment only; no utility class is generated for it.

#### 9.6 Magenta pair (NORMAL ↔ HIGHLIGHT) — current page / tag / vim descent

| Original              | Token                     | Hex       | Use                                                                                |
| --------------------- | ------------------------- | --------- | ---------------------------------------------------------------------------------- |
| medium-red-violet-600 | `--medium-red-violet-600` | `#B23B83` | **normal**: category links, current-page nav (JS), vim parent label, tag underline |
| medium-red-violet-400 | `--medium-red-violet-400` | `#D77AB2` | **hover**: category links hover, index decoration line                             |

#### 9.7 Status

| Original | Token          | Hex                   | Use                                           |
| -------- | -------------- | --------------------- | --------------------------------------------- |
| red-600  | `--red-600`    | `#DC2626`             | RSS copy **failed** state (the only true red) |
| blue-500 | `--focus-ring` | `rgba(59,130,246,.5)` | focus outline                                 |

#### 9.8 Vim highlight — translucent, separate pair

| Original             | Token              | Value                | Use                            |
| -------------------- | ------------------ | -------------------- | ------------------------------ |
| eucalyptus-600 @ 7%  | `--highlight-soft` | `rgba(2,135,96,.07)` | j/k list & card cursor overlay |
| eucalyptus-600 @ 14% | `--highlight`      | `rgba(2,135,96,.14)` | tag / preview / nav / TOC pill |

#### 9.9 Code containers (per `params.codeScheme`)

| Scheme          | Token                     | Value                 | Use                          |
| --------------- | ------------------------- | --------------------- | ---------------------------- |
| quiet-light     | `--code-bg` / `--code-fg` | `#F5F5F5` / `#333333` | code block background / text |
| night-owl-light | `--code-bg` / `--code-fg` | `#FBFBFB` / `#403F53` | code block background / text |

`pre.tm-code` uses `var(--code-bg)/var(--code-fg)`. The token *colors inside*
the code (keywords, strings…) come from the VS Code theme JSONs — not from
these tokens (see §10).

> **INVARIANT — vim highlight is translucent.** The vim cursor and pills must
> keep their alpha (`--highlight-soft` 7%, `--highlight` 14%) so the text shows
> through. A tokenizer once flattened them to the solid `--eucalyptus-600` and
> the whole vim highlight turned solid — never replace a translucent token with
> a solid one.

**Consolidation**: three near-duplicate, low-use colors were merged into their
nearest palette member: `#027A56→#028760`, `#80D2CD→#9ACFBF`,
`#42ABA5→#49BEB7`; and `slate-300 #CBD5E0→#B6E5E2` (now `--teal-pale`). The
eucalyptus scale is 300/400/600 — 500 was an identical alias and is removed.

**Remaining near-pairs (kept — different roles, but visually close):**
- `--eucalyptus-300` (`#9ACFBF`, dividers) vs `--teal-pale` (`#B6E5E2`, borders)
  — two light teals, both used on hairlines.
- `--color-heading` (`#01513A`, headings) vs `--teal-dark` (`#2C726E`, selection)
  — two dark green-teals.

> **Caveat — `static/dist/app.css` is a build artifact.** It was post-processed
> in place to use the tokens, but it is generated by the theme's webpack/Tailwind
> build. Rebuilding the theme regenerates it with raw values; re-run the
> tokenizer in that case. The hand-written source of truth is `tokens.css` +
> the templates.

## 10. Code syntax colors — VS Code themes

The colors *inside* code blocks (keywords, strings, numbers, comments…) are
**not** §9 tokens. They come from the real VS Code color themes — same grammar
+ same theme files + same resolution VS Code uses:

- Theme files: `static/lib/quiet-light.json`, `static/lib/night-owl-light.json`
  (the actual VS Code theme JSONs; pick one with `params.codeScheme`).
- Grammar: `static/lib/cpp.tmLanguage.json` (+ bash / python / cmake / x86_64
  in the same dir, listed in `static/lib/grammars.json`).
- Engine: `static/js/tm-highlight.min.js` — `vscode-textmate` tokenizes in the
  browser and resolves each token's scopes against the theme's `tokenColors`
  (most specific scope first, longest rule wins, later identical rule wins).
- Code blocks are emitted raw by `layouts/_default/_markup/render-codeblock.html`
  (`<pre class="tm-code" data-lang>`) and coloured by the engine at runtime.

## 11. Code block chrome — gutter, badges & square corners

The furniture *around* the code (line numbers, language label, copy button,
corners) is our own design, styled in `layouts/partials/code-theme.html` and
sized by the engine at runtime. Everything is square and borderless; the only
stroke is the block's teal border (kept from §9) sharpened by a 0-blur ring.

- **Line-number gutter** — VS Code style: each logical line is
  `<div class="tm-line">` with a fixed digit column `.tm-gutter` (numbers
  right-aligned, muted, `user-select:none`) + `.tm-line-code`. The gutter's
  width is `--tm-gutter-w`, measured by `tm-highlight` to the widest line
  number (so `1` and `100` never crowd the code); the gap to the code is a
  `margin-right` (NOT padding — the theme's global `box-sizing:border-box`
  would swallow a padding into the flex basis). The copy never includes the
  numbers because copying reads `code[data-src]` (the raw source captured
  before tokenizing).
- **Equal-width square badges** — language label `.tm-lang` (top-left) and
  copy button `.tm-copy` (top-right). Both are `border-radius:0`,
  `border:none`, translucent teal (`rgba(73,190,183,.16)` = `--teal-hover` at
  16%) with `backdrop-filter:blur(8px)`, text in `--teal-dark`. The frame is
  the tint itself, not a stroke. They share one width `--code-badge-w`, set by
  `tm-highlight` to the widest of the label / `copy` / `copied` / `failed`
  text (≈ label width + 1rem padding), so names centre and `copied` never
  overflows. `pre.tm-code` has `padding-top:3.5em` so the chips float clear of
  the code with ~10px of air. Clicking copy writes `code[data-src]` to the
  clipboard and fires the **same "Copied!" burst as the RSS icon** — the
  label stays `copy` (the burst is the feedback), only a rejected write shows
  `failed` briefly. The burst lives in `site-scripts.html` as
  `window.tokiwaCopyBurst` (single source, shared by RSS + code blocks).
  The label shows the language's **formal name**: `data/code-langs.yaml`
  maps fence short names (`cpp` → `C++`, `cmake` → `CMake`); unknown names
  fall back to the fence text, and `data-lang` keeps the short name so the
  engine can still look up the grammar.
- **Square corners** — the compiled scss's rounded bottom-right is overridden
  to `border-radius:0`. The border colour stays the theme teal (unchanged);
  the corners are sharpened *without* touching it by a 0-blur hairline ring
  `box-shadow:0 0 0 1px rgba(15,23,42,.05)` that follows the same 90° corners
  exactly — each corner point is drawn twice and reads as a crisp cut.
- **Rainbow brackets** — matched `() [] {} <>` get a hue by nesting depth
  (VS Code's bracket-pair-colorizer). Six palette tokens `--br0..--br5`
  (§9, `tokens.css`) are cycled by a stack that spans the whole block:
  an opener's colour = depth before push, a closer reuses its opener's index
  so the pair always matches. `tm-highlight` wraps each bracket char in
  `<span class="tm-br{n}">` (a child of the token span, so its own colour
  wins) and **skips brackets inside strings/comments** — detected from the
  token scopes — so a `'{'` in a comment or string neither colours nor throws
  off the depth. Palette: five of the six hues are **exact night-owl-light
  theme colours** (red `#d3423e`, teal `#0c969b`, blue `#4876d6`, purple
  `#994cc3`, magenta `#aa0982`) with one complementary orange `#c2651f`
  between red and teal — then cycles.
- **Angle brackets** — `<`/`>` double as comparison & shift operators, so
  they need extra care. A bracket-looking `<>` is coloured only when either
  (a) the grammar marks it `punctuation.section.angle-brackets.*` (template
  definition / call / alias), or (b) it's an operator-scoped `<>` that
  *directly abuts an identifier* — `Name<…>` — because the cpp grammar misses
  `std::vector<int> v;` (it scopes those as `keyword.operator.comparison`).
  Spaced `a < b`, `<<` shifts and `->` arrows stay plain. A `>` matches the
  nearest `<` on the stack; a `<` farther than `ANGLE_MAX` (160 chars) without
  a `>` is retracted as a shift/redirection phantom. `#include <header>` is a
  special case: the grammar scopes the header as a string, but the `<`/`>`
  are the header-name delimiters, so they get an isolated pair at the current
  depth (never touching the nesting stack) — matching how VS Code visibly
  colours them.

## 12. Zen mode — `zz`

A VS Code-style focus mode, desktop only (the two-column split only exists at
`md+`). `zz` toggles it; `za` stays the TOC toggle (both live on the `z`
leader, so a third key inside the 1.2 s leader window is a no-op).

- **Global state** — `html.zen` + a `zenActive` flag in `site-scripts`
  (`window.tokiwaToggleZen`), persisted to `sessionStorage` (`tokiwa.zen`) and
  restored by a `<head>` script **before first paint**, so a reload never
  flashes the two-column layout. `--zen-shift` lives on `<html>` (not the
  shell) so it survives full page loads and SPA content swaps, and is
  re-measured on load. Leaving the desktop breakpoint auto-exits.
- **Motion is a state machine** — the CSS transition is gated behind a
  transient `html.zen-anim` class that `tokiwaToggleZen` adds only for the
  duration of a toggle (then drops after ~700 ms). So the slide plays exactly
  once on enter and once on exit; a page load or an SPA content swap while in
  zen settles directly into the centred layout and **never replays the
  animation**.
- **Naming** — the two halves of the shell have semantic names (purely
  additive `data-*` attributes, no layout/behaviour impact): the left column
  (header + aside) is **the rail** `data-home-rail`, the right column
  (main + footer) is **the pane** `data-home-pane`. The whole two-column
  structure is **the split** inside **the shell** (`data-home-shell`).
- **2K adaptation** — a dedicated `@media (min-width: 1920px)` layer: on 2K+
  screens the max-w-capped columns (rail 672px + pane 768px) no longer fill
  the shell row, so the split is centred (`justify-content: center`) instead
  of leaving a big empty right gutter. `--zen-shift` is measured from the
  pane's *untransformed* centre, so zen stays exactly centred at any width
  (the shift is `railWidth / 2` only while the columns fill the row; on 2K it
  differs and can even be negative).
- **One-screen interaction** — on one-screen pages `pinBottomRails()` freezes
  the aside + footer as `position:fixed` and measures their `left` with
  `getBoundingClientRect()`, which *includes* the zen transform; a re-pin
  while zen is on would re-apply the shift on top of the CSS one and shove
  the footer off-centre (it runs inside the one-screen rAF loop, so it
  re-fires after every toggle). `pinBottomRails()` therefore temporarily
  drops `html.zen` while measuring, so the fixed pin always uses the
  untransformed layout position.
- **Motion** — pure CSS in `baseof.html` (`@media min-width:768px`):
  `html.zen` slides the rail (`header` + `aside`) off the left edge
  (`translateX(calc(-100% - 4rem))`, fading out, `pointer-events:none`) and
  shifts the pane (`main` + `footer`) left by `--zen-shift`. `--zen-shift` is
  the measured distance from the pane's *untransformed* centre to the
  viewport centre (`--zen-shift = paneCentre − vw/2`, which is `railWidth / 2`
  only while the columns fill the row), so the pane lands exactly centred at
  any width. It is measured by JS at toggle time, on load and on resize.
- **Table widening is part of the motion** — a Table Spread table animates
  its `width` on the same `html.zen-anim` gate, starting WITH the pane (no
  lag) but running a beat longer (`width .65s cubic-bezier(.4,0,.2,1)` vs the
  rail/pane `.55s`), so a wide table can never finish widening before the
  rail has cleared. It reverses on exit. `measureWidths()` measures on a
  detached clone so the live table's width only changes once (no CSS
  max-width cap — a cap would snap the table shut the moment `html.zen` is
  dropped and kill the reverse animation); the transition then slides from
  the pre-toggle width.
- **Table Fitting (宽窄两态)** — the named feature that gives a body table
  a width that fits the pane, and **flattens** it: every table shows its
  content on as few lines as the space allows — the only difference between
  the two forms is the width cap. Implemented in `site-scripts.html`
  (`fitTables`, exposed as `window.tokiwaFitTables`) + `.tbl-fit` +
  `.tbl-oneline` + `.tbl-flat` in `baseof.html`. *The algorithm* — for each
  table, measure three widths with the table pinned to `width:1px`
  (a `width:auto` table stretches to fill the pane, so it would report the
  pane instead of its honest sizes):
    `minContent` — every cell wraps to its minimum;
    `hdrReq`     — headers on one line, body wraps;
    `maxContent` — nothing wraps at all.
  - narrow (`maxContent ≤ pane`) → the table's no-wrap width fits the pane,
    so it is sized to `maxContent` with **every cell kept on one line**
    (`.tbl-flat` — `th, td { white-space:nowrap }`): all content is shown,
    never compressed. It does **not** spread in zen (nothing to gain; it is
    already fully readable at `maxContent`, and stays centred).
  - wide (`maxContent >  pane`) → flattened to the pane's full width so
    cells wrap as little as the space allows, headers kept one line when
    they fit (`.tbl-oneline` — `th { white-space:nowrap }`); body cells wrap
    only where the width forces them. If even *wrapped* content can't shrink
    into the pane (`minContent > pane`) it takes `.tbl-fit`
    (`table-layout:fixed; width:100%`), edges flush with the pane, no
    scroll, no slide, columns frozen at their natural proportions.
  - **why fixed layout** — auto layout cannot size a table below its
    min-content, so squeezing an over-wide table (e.g. an 8-column
    source-code table) into the pane needs `table-layout:fixed`.
  - **zen centring** — `html.zen .c-rich-text table` centres with
    `left:50% + translateX(-50%)` alone; the normal-mode `margin:auto`
    centring is switched off (`margin-left/right:0`) so a narrow table
    (already centred by its margins) is not double-shifted off-centre. Every
    width, narrow or spread, lands centred on the viewport.
- **Table Spread (突破边界)** — the named feature that lets a wide table
  break past its container's left/right edges and stretch toward both sides
  — but only **as needed**: never wider than its content demands, and capped
  by `--tbl-spread-max`. In this theme it activates in **zen mode**: a table
  whose max-content exceeds the pane gets auto layout,
  `width: min(maxContent, --tbl-spread-max)`; it is fully flattened
  (`.tbl-flat`) when it fits the cap, otherwise headers one line
  (`.tbl-oneline`) when the width can afford them. A table with little
  content stays at its own maxContent; only a content-heavy table reaches
  the cap.
  *Portable contract:* a host provides `--tbl-spread-max` (this theme
  measures it as the normal-mode span from the rail's left edge to the
  pane's right edge — narrower on a 13-inch laptop, wider on 2K); the JS
  toggles `.tbl-spread` instead of `.tbl-fit` when spread is available.
  - **wide-ness is judged by max-content, not the wrapped width** — on a
    wide screen the pane is broad enough that a wide table can still wrap to
    fit it, so it must be recognised as wide by its *nowrap* width or it
    would never stretch.
  - the centring works because in zen the pane is already shifted so its
    centre sits on the viewport centre; a spread table and a normal table
    both resolve `left:50%` against the same pane centre.
- **Triggers** — `zz` (second `z` press, repeats ignored); the same
  transition replays in reverse on exit. Independent of the vim engine, so
  it works in the article, the posts list and every taxonomy page.


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
- **The teal vim pill is this same 2-line label box** — the pill's size is
  always the label's line box (glyph + symmetric ~8px leading = 52px), never
  the card's highlight box. It is a background on the label link, so its height
  IS the label's line box. It only translates (see §5); it is never resized to
  match the highlight. This symmetry (equal ~8px leading above and below the
  glyph) is what keeps the pill reading as "the label + padding", so a pill
  that is taller than the label's line box is a regression.

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
- `/` — focus the search box. Leader keys: `gh` `gp` `gt` `gc` `gs`.
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

### Nav live-preview (first-level pages)

On a **first-level** page — a page that is exactly one of the nav destinations
(home / posts / tags / categories / series / about) — the nav is a live
preview: `j`/`k`/`gg`/`G` moving the cursor **loads the selected page's main +
footer into the right pane via fetch**, in place — not a navigation, so the URL
never moves (every first-level page shares the entry URL, e.g. `/`). Each move
bumps a `previewRequestId`; a stale response is dropped, so rapid j/k always
settles on the last selection. The swap re-boots the target's machinery through
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
are two inputs on one engine. The click moves the vim nav cursor if it is
active, updates the purple-red current-page indicator, and blurs the clicked
link so the indicator shows immediately (matching a real page load).
External links and non-previewable targets (articles / 404) fall back to a
real navigation; clicks from deeper pages navigate normally.

The **g-leader is exactly a mouse click**: `gh`/`gp`/`gt`/`gc`/`gs` resolve
their route to the matching nav element via `navItemForHref()` (the home title
is its own first-class item) and call the same shared `openNavItem()` as a
click — dynamic-load into the right pane on a first-level page (URL never
moves, purple commits on the target), real navigation otherwise. So every way
of "selecting" a first-level page — j/k/gg/G preview, `l`/Enter commit, mouse
click, and the g-leader — is one engine.

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
| card previews| `taxPreviews(active card)` (live)        | — (card box only) |

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
line remains), generic list → `pagination`, about (`layouts/about/list.html`) →
a non-empty override rendering nothing — About is a fixed one-screen page with
**no footer**. Taxonomy overview (`terms.html`) and 404 render nothing (the
one-screen condition is false; an *empty* `define "footer"` does **not**
override a `block` default in Hugo — only a non-empty one does, so keep the
gating in baseof).

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

### Article body drops the markdown h1

On article pages the header already shows the title, so the body hides the
markdown level-1 heading it would otherwise render (`body.is-article
.c-rich-text h1 { display: none }`). The markdown files are shared with other
sites and keep their own `#` heading — only this rendering drops it. Per
request this hides **all** h1 in the body (sections also written with `#` are
h1 too and are hidden as well); `##`-level headings are untouched. The rule is
scoped to `.is-article .c-rich-text`, so list / taxonomy / about pages keep
their own headings.

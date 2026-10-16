# Custom Pages: Dashboard Design

A dashboard page that shows only the literal data asked for reads as unfinished. Build in the
pieces below unless the user asks for less, and drop one when the request rules it out.

## What a finished dashboard has

| Piece | What it is |
|---|---|
| KPI row | Three to five tiles across the top: counts, plus a total for context |
| Breakdowns | Bar lists or charts that use the page width, one per dimension the data has |
| Second dimension | Stack each bar by a second field, such as status inside each group, so the reader sees where a number comes from |
| Status colour | One colour per status, the same on every chart, with a legend |
| Drill-through | A bar, tile, or row links to the matching records |
| As-of line | When the data was loaded, near the title |
| Loading and empty states | A visible loading state, and an empty state that says why there is nothing to show |

## Design system

When the user names the path to the Veeva Vault Design System repo, read its `DESIGN.md` and `AGENTS.md` before
writing styles and use its tokens (colour, type, spacing) in place of ad hoc values. Where the
design system and the request disagree, say which one the page follows.

When no design system is named, the page carries its own defaults and the skill does not hand
styling to another skill. Use the system font stack, white and light grey surfaces, dark grey text,
and one accent colour for the main series and links. Give a chart's categories no more than five
distinct colours, label bars and lines directly instead of leaning on a legend, and never use
colour as the only signal, since status needs a word or icon beside it. Keep the same
tokens across every tile so the page reads as one piece.

## Data shape for a filterable page

Filters that run in the browser need the data in the browser. Do not ship one row per record to get
there. Have the controller return a small cube instead: sums and counts grouped by every dimension
a filter or breakdown uses, such as group, status, category, and month. The payload stays small as
the record count grows, and the client sums the cells it needs.

## Layout rules

`gotchas-and-type-mapping.md` has the CSS rules that decide whether the page fits: `box-sizing:
border-box` everywhere, `width: max-content` inside a scrolling container for a wide row, and a
layout sized to the viewport. Stylesheets go in the manifest's `stylesheets` array.

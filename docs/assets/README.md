# Assets

Images used by the documentation. Every asset follows the conventions below so the set reads as one system in GitHub's light and dark themes.

## Layout

| Folder | Holds |
| --- | --- |
| `diagrams/` | Hand-drawn SVG diagrams for structures Mermaid cannot draw well |
| `headers/` | One decorative strip per documentation section |
| `social/` | The repository's social preview card |

## Palette

Black, white and greys only. No accent colour.

| Role | Value |
| --- | --- |
| Panel fill | `#000000` |
| Primary text | `#e8e8e8` |
| Lines, frames, secondary text | `#7a7a7a` |
| Raised cell fill | `#1a1a1a` |

Every panel carries a 1px `#7a7a7a` hairline frame. GitHub's dark page is `#0d1117`, not black, so an unframed black panel reads as an off-shade rectangle.

## Text in images

- Diagram labels are live `<text>` in `ui-monospace, SFMono-Regular, Consolas, Menlo, monospace`, at 14px or larger on an 880-wide viewBox, with room to spare in every box because glyph widths vary by system.
- Display lettering (section strips, banner) is converted to outlines, because an SVG shown through `<img>` cannot load fonts.
- Each file has a `<title>` and `<desc>`. The Markdown that embeds it gives full alt text; decorative strips use `alt=""`.

## Fonts

Display lettering uses two typefaces under the SIL Open Font License 1.1, converted to outlines, so no font file ships in this repository:

| Typeface | Used for | Copyright |
| --- | --- | --- |
| Cinzel | Banner title, section names, numerals | The Cinzel Project Authors |
| JetBrains Mono | Banner subtitle and labels | The JetBrains Mono Project Authors |

## Social preview card

`social/social-preview.png` (1280×640) is generated from `social/social-preview.svg`, which shares the banner's design.

## Rendering rules

- One file per asset for both themes. No `prefers-color-scheme` inside an SVG: it follows the operating system, not GitHub's theme setting.
- No scripts, external references or animation.
- Size targets: diagrams under 30 KB, strips under 15 KB, banner under 60 KB.

## Mermaid

Mermaid diagrams are written inline and share one `%%{init}%%` theme line: black fill, `#e8e8e8` text, `#7a7a7a` lines, no custom font, and no labels on arrows where avoidable.

- Box text wraps at 520px, not Mermaid's default 200px, so a short list stays on one line.
- A long chain of steps is laid out as two rows inside frameless subgraphs, not as one row that GitHub would shrink to fit the page.

## The sourcing rule

A diagram restates facts already written on the page that shows it. It never adds one.

Each SVG opens with `<!-- source: "…" -->` comments quoting the sentences it draws on. Most Mermaid blocks carry the same quotes as `%% source:` lines.

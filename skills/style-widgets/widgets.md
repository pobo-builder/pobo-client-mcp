# Widget map: catalog id → CSS block → variables → slots

Facts from the public catalog (`get_content_widget_catalog`, 2026-09) and rendered
HTML. Use it to know which variables reach a widget and which slots will render
empty. The eshop's own catalog is the truth for ids and roles.

- `css_block`: class on the container `widget-container widget-<block>`; target of
  `--pobo-widget-<block>-*` (bg, bg-size, padding, margin, border-radius,
  box-shadow, before-*, after-*), which `get_theming_contract` returns.
- `element vars`: `--pobo-<prefix>-*` in `generic.css`, not in the contract yet
  (see `runtime.md` for how to list them). Counts verified on 2026-09-22; "none"
  means the block is styled by container vars and `--pobo-typo-*` only, and
  anything else needs a direct `.rc-*` rule.
- `wrapper`: `typography` = `--pobo-typo-*` and `--pobo-link-*` apply to the text;
  `projector` = text is styled by the element vars only.
- An empty image or icon slot renders `<img src="">` (see `runtime.md`).
- "likely" = derived from the catalog class prefix, not seen rendered.

| id | catalog name | css_block | wrapper | element vars | image | icon | text roles (max) |
|---|---|---|---|---|---|---|---|
| 74 | Jumbotron | `widget-jumbotron-one` | projector | `--pobo-jumbotron-one-` | 1 | 0 | title 60, text 150 |
| 168 | Jumbotron with button | `widget-jumbotron-one` | projector | `--pobo-jumbotron-one-` | 1 | 0 | title 60, text 150 (+ button) |
| 7 | Text v pravo a větší obrázek vlevo (renders image **right**) | `widget-image-half-right` | typography | none (container vars + `--pobo-typo-*` only) | 1 | 0 | paragraph 300 ×2 (HTML) |
| 8 | Text v pravo a větší obrázek vlevo (image left) | `widget-image-half-left` | typography | none (container vars + `--pobo-typo-*` only) | 1 | 0 | paragraph 300 ×2 (HTML) |
| 2 | Text vlevo a menší obrázek vpravo | `widget-image-right` | typography | `--pobo-image-right-` | 1 | 0 | paragraph 300 (HTML) |
| 3 | Text vpravo a menší obrázek vlevo | `widget-image-left` | typography | `--pobo-image-left-` | 1 | 0 | paragraph 300 (HTML) |
| 162 | Text vpravo a obrázek vlevo dva sloupec | `widget-image-right-two-column` (likely) | typography | none | 1 | 0 | title 60 ×2, paragraph 300 ×3 |
| 163 | Text vlevo a obrázek vpravo dva sloupec | `widget-image-left-two-column` (likely) | typography | none | 1 | 0 | same as 162; catalog `class` has a stray quote |
| 70 | Klasický dlouhý text | `widget-text` | typography | none (`--pobo-typo-*`) | 0 | 0 | paragraph 300 (HTML) |
| 50 | Text ve dvou sloupcích 50/50 | `widget-text-two-column` (likely) | typography | `--pobo-text-two-column-` | 0 | 0 | paragraph 300 ×2 |
| 4 | Velký nadpis a pod ním velký text | `widget-header-text` | typography | `--pobo-header-text-` | 0 | 0 | title 60, text 150 |
| 110 | Nadpis s designovou čárou | `widget-title-line` (likely) | projector | `--pobo-title-line-` | 0 | 0 | title 60 |
| 22 | Nadpis s textem v obrázku vlevo nahoře | `widget-text-image-top` (likely) | projector | none found | 1 | 0 | text 80, text 200 |
| 37 | Text v obrázku dole | `widget-image-one` (likely) | projector | none found | 1 | 0 | text 80, text 200 |
| 249 | Separátor obsahu | `widget-separator-content` (likely) | projector | `--pobo-separator-content-` | 0 | 0 | none |
| 266 | Jeden velký obrázek | `widget-gallery-one` + `widget-gallery` | projector | `--pobo-gallery-one-`, `--pobo-gallery-` | 1 | 0 | text 150 (caption); lightbox on click |
| 267 | Dva obrázky vedle sebe s popiskem | `widget-gallery-two` (likely) | projector | `--pobo-gallery-two-`, `--pobo-gallery-` | 2 | 0 | text 150 |
| 268 | Tři obrázky vedle sebe | `widget-gallery-three` (likely) | projector | `--pobo-gallery-three-`, `--pobo-gallery-` | 3 | 0 | text 150 |
| 269 | Velký obrázek vlevo a dva malé vpravo | `widget-gallery-featured-left` (likely) | projector | `--pobo-gallery-featured-` | 3 | 0 | none |
| 103 | Dva číselné boxy | `widget-counter` | projector | `--pobo-counter-` (`-header-font-size`, `-header-font-size-sm`, `-info-*`, `-img-*`, `-inner-*`) | 1 | 0 | 2 items: benefit_title 80, benefit_text 120 |
| 104 | Tři číselné boxy | `widget-counter` | projector | `--pobo-counter-` | 1 | 0 | 3 items: benefit_title 80, benefit_text 120 |
| 89 | Obrázek vlevo a parametry v jednom sloupci | `widget-parameter-big-right` | projector | `--pobo-parameter-big-`, `--pobo-parameter-` | 1 | **5** (one per row) | title 60, 5× parameter_name 50 + parameter_value 100 |
| 86 | Obrázek vlevo a parametry ve dvou sloupcích | `widget-parameter-small-right` | projector | `--pobo-parameter-small-`, `--pobo-parameter-` | 1 | **6** | title 60, 6× name 50 + value 100 |
| 88 | Obrázek vpravo a parametry ve dvou sloupcích | `widget-parameter-small-left` (likely) | projector | `--pobo-parameter-small-`, `--pobo-parameter-` | 1 | **6** | title 60, 6× name 50 + value 100 |
| 33 | 2 ikony s nadpisy a popisem | `widget-advantages-two` (likely) | projector | `--pobo-advantages-two-`, `--pobo-advantages-` | 0 | 2 | benefit_title 40, benefit_text 150 |
| 6 | 3 ikony s nadpisy a popisem | `widget-advantages-three` (likely) | projector | `--pobo-advantages-three-`, `--pobo-advantages-` | 0 | 3 | benefit_title 40, benefit_text 150 |
| 19 | 4 ikony s nadpisy a popisem | `widget-advantages-four` | projector | `--pobo-advantages-four-`, `--pobo-advantages-` | 0 | 4 | benefit_title 40, benefit_text 150 |
| 108 | 3 ikony inline s nadpisem a popisem | `widget-inline-image` | projector | `--pobo-inline-image-` | 0 | 3 | benefit_title 40, benefit_text 150 |
| 109 | 2 ikony inline s nadpisem a popisem | `widget-inline-image` | projector | `--pobo-inline-image-` | 0 | 2 | benefit_title 40, benefit_text 150 |
| 107 | Výhody produktu v bodech | `widget-profit` (likely) | projector | `--pobo-profit-` | 0 | 3 | title 60, 3× benefit_title 40 + benefit_text 150 |
| 105 | Výhody vlevo a obrázek vpravo | `widget-profit` (likely) | projector | `--pobo-profit-` | 1 | 3 | as 107 |
| 106 | Výhody vpravo a obrázek vlevo | `widget-profit` (likely) | projector | `--pobo-profit-` | 1 | 3 | as 107 |
| 39 | Tři obrázky vedle sebe s textem uvnitř | `widget-image-three` | projector | `--pobo-image-three-` | 3 | 0 | 3 items: benefit_title 40, benefit_text 150 (catalog lists benefit_text twice; fill once) |
| 5 | Pětihvězdičková recenze | container `reviews-two-box-five-star`, block `widget-reviews-two` | projector | `--pobo-reviews-two-` (`-box-photo-size`, `-box-photo-radius`) | **1 per review** (photo) | 0 | 2 items: author 50, text 150; stars fixed at 5 |
| 31 | Rozbalovací otázka a odpověď | `widget-faq` (contract: `widget-pb-faq`) | projector | `--pobo-faq-` | 0 | 0 | question 80, answer 200; one Q&A per widget |
| 30 | YouTube na celou šířku | `widget-one-video` (likely) | projector | `--pobo-video-` | 0 | 0 | video |
| 25 / 26 | Menší YouTube + text | `widget-video-right` / `widget-video-left` (likely) | typography | `--pobo-video-` | 0 | 0 | paragraph 300, video |
| 27 / 28 | Větší YouTube + text | `widget-video-half-left` / `-right` (likely) | typography | `--pobo-video-half-` | 0 | 0 | paragraph 300 ×2, video |
| 80 / 81 | Obrázek + video | `widget-video-left` / `-right` (likely) | projector | `--pobo-video-` | 0 | 0 | video |
| 98 | Dvě videa vedle sebe | `widget-two-videos` (likely) | projector | `--pobo-video-` | 0 | 0 | video |
| 296 / 297 | Video + text | (likely `widget-video-*`) | — | — | 0 | 0 | **no text roles in catalog** |
| 82 | Tlačítko s odkazem na střed | `widget-text-button` (likely) | — | `--pobo-btn-` (global) | 0 | 0 | **no text roles in catalog** |
| 92 / 93 | Obrázek + text s tlačítkem | (likely `widget-image-*-overlay`) | — | — | 0 | 0 | **no text roles in catalog** |
| 11 | Nadpis s odrážkami + obrázek | `widget-checked-*` (likely) | — | `--pobo-checked-` | 1 | 0 | **no text roles in catalog** |
| 270 | Velký obrázek vpravo a dva malé vlevo | `widget-gallery-featured-right` (likely) | projector | `--pobo-gallery-featured-` | **0 in catalog** (data bug) | 0 | none |

Widgets with no text roles in the catalog cannot be filled through the MCP.

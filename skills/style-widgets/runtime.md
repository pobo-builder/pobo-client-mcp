# Runtime constraints of Pobo content and CSS

What the eshop page can render. Verified against the server rules
(`push_asset_css`), `generic.css` and live pages on 2026-09-22. Shared by
`style-widgets` and `design-description`.

## CSS asset

- One AI style asset per eshop; `push_asset_css` replaces the whole file. Max 256 KB SCSS.
- Rejected: `@import`, `expression(`, `javascript:`, `behavior:`, `-moz-binding`.
- `url(...)` accepted only for: Pobo CDN (`image.pobo.space`), the platform's asset
  CDN (`cdn.myshoptet.com`, `cdn.shopify.com`), a relative path, or
  `data:image/{png,jpeg,gif,webp,avif,svg+xml}`. Anything else fails the push.
- **The asset loads before `generic.css`** (`templates/<eshop>.css`, then
  `custom/<eshop>.css`, then `generic.css`; verified on a b2b page, check the
  `<link>` order in `fetch_eshop_page` on other platforms). A plain
  `:root { --pobo-...: x }` override therefore loses to the later `:root`
  declaration in `generic.css`. Declare overrides on `:root #pobo-all-content`
  (higher specificity) as the default, not as a fallback.
- Modern CSS works (custom properties, `clamp()`, grid, `object-fit`, range media
  queries `@media (width <= 767px)`, `prefers-reduced-motion`).

## Reading the deployed asset

There is no tool that returns the current AI asset source; `list_asset` gives
name, size and date only. The compiled CSS is public: the eshop page links it as
`<link rel="stylesheet" href="https://image.pobo.space/templates/<domain>.css?v=…">`
(verified on one eshop; find the exact `href` in the `fetch_eshop_page` HTML).
Read it with `curl -s <href>` when you must keep existing rules. It is compiled
CSS, not the SCSS that was pushed, so nested rules and variables come back
flattened; carry the declarations over and re-emit them in your SCSS.

## Fonts

- No `@font-face` with an external URL, no `@import` of Google Fonts, no
  `data:font/*`. **Pobo CSS cannot load a webfont.**
- `--pobo-font-family` defaults to `var(--template-font)`, a variable the shop
  template may or may not define (Erotic City does not). Never rely on it; set
  `--pobo-font-family` yourself.
- Usable families: those the host page already loads (read `fetch_eshop_page`
  for `<link rel="preload" as="font">`, `@font-face`, `fonts.googleapis.com`,
  `fonts.shopifycdn.com`, and `font-family:` declarations), plus system stacks.
- Write the family by its exact name and give a fallback stack:
  `--pobo-font-family: "Montserrat", system-ui, sans-serif;`
- If the host loads only one family, the description has one family. Build hierarchy
  with size, weight and spacing, not with a second face. If the host loads a
  display face and a body face, you may use both.
- A serif "fallback" that shows up on the screenshot is a bug, not a style.

## Variables: where they live

`get_theming_contract` exposes only the widget container variables
(`--pobo-widget-<block>-{bg,bg-size,padding,margin,border-radius,box-shadow,before-*,after-*}`)
and three globals. The rest exists in `generic.css` but is not in the contract yet:

- Core (74): `--pobo-font-family`, `--pobo-all-content-{background,padding,margin}`,
  `--pobo-global-widget-{max-width,padding,margin}`,
  `--pobo-typo-{h2,h3,h4}-{font-family,font-size,font-weight,font-color,line-height,margin,padding}`,
  `--pobo-typo-{h2,h3}-before-{content,bg,width,height,left,bottom}`,
  `--pobo-typo-p-{font-family,font-size,font-weight,font-color,line-height,margin,padding}`,
  `--pobo-typo-list-{padding,margin}`, `--pobo-typo-list-item-*`,
  `--pobo-link-{color,hover-color,text-decoration,hover-text-decoration,transition,outline}`,
  `--pobo-btn-{bg,hover-bg,color,hover-text-color,border,border-radius,box-shadow,padding,margin,font-size,font-weight,line-height,transition}`.
- Element level per block: `--pobo-<block without "widget-">-<element>-<property>`,
  for example `--pobo-counter-header-font-size`, `--pobo-counter-header-font-size-sm`,
  `--pobo-counter-img-width`, `--pobo-faq-header-*`, `--pobo-reviews-two-box-photo-size`,
  `--pobo-parameter-*`, `--pobo-gallery-*`, `--pobo-advantages-*`, `--pobo-inline-image-*`.
- To list a block's element variables without reading the 800 KB file:

  ```bash
  curl -s https://image.pobo.space/assets/generic.css | grep -oE '\-\-pobo-counter-[a-z0-9-]+' | sort -u
  ```

- `--pobo-typo-*` and `--pobo-link-*` apply inside `.widget-typography` wrappers
  (text widgets 2, 3, 4, 7, 8, 50, 70, 162, 163). Widgets wrapped in
  `.widget-projector` (103, 104, 86, 88, 89, 105 to 109, 19, 6, 33, 39, 5, 31,
  266 to 270) take their type from the block's element variables instead.

## The single surface

The whole description is `#pobo-all-content`. `--pobo-all-content-background` and
`--pobo-all-content-padding` are variables; the radius is not, so:

```scss
#pobo-all-content { border-radius: 1.6rem; overflow: hidden; }
```

Widgets have `--pobo-global-widget-max-width: 1200px` and `margin: 0 auto`; a
full-bleed section inside the surface needs its `--pobo-widget-<block>-bg` or a
`before` pseudo-element, since the container itself is capped.

## Content and HTML

- Text roles accept formatting HTML only (`h2`, `h3`, `p`, `ul`, `ol`, `table`, `a`).
  `img`, `iframe`, `script`, inline handlers are stripped; `content_sanitized: true`
  tells you it happened.
- An **empty image or icon slot renders `<img src="">`** with `loading="lazy"`.
  `generic.css` does not hide it. Either fill every slot or add
  `#pobo-all-content img[src=""] { display: none; }` and check the layout still
  holds without it (parameter widgets keep an empty icon column).
- `alt` is the entity name on every image; not editable.
- Image crop is CSS only: `object-fit` and `object-position` on `.rc-*__img`.
  There is no focal point parameter. Choose images whose subject survives a
  center crop, or set `object-position` per widget.
- `max_length` on a role is advisory; the server writes overflow in full and the
  block may break. Keep inside it.
- Links are static `<a>`. No cart, no stock, no price, no variant selector can be
  wired from here. The product page around the description already has them.

## JavaScript

Nothing deployable. Widgets ship their own scripts (FAQ toggle, gallery lightbox
on `.pb-gallery-trigger`). Do not plan hover states that need JS, sliders,
counters that animate, or tabs.

## Limits

- 60 requests per minute per user.
- `compose_content`: 30 widgets max per call.
- Shoptet products: 65 000 bytes of rendered content; `render_content_html`
  reports it.
- Screenshot over 3 MB is rejected; use `selector: "#pobo-all-content"`.
- `fetch_eshop_page` truncates at 500 KB; font links are in `<head>`, so they survive.

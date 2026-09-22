# Audit Pobo MCP pro showcase stránky a návrh dalšího vývoje

Datum: 2026-09-22. Podklad: showcase Womanizer NEXT (Erotic City, eshop 4679, produkt 14755340) a pro srovnání LEGO Slůně (Bambule, eshop 4673, produkt 14741995).

## 1. Shrnutí

Největší část problémů z Womanizer showcase nevznikla špatným promptováním, ale tím, že MCP dává Claudeovi špatný nebo žádný kontext v pěti místech:

1. **`get_theming_contract` vrací nejméně užitečnou třetinu proměnných.** Z 4 760 `--pobo-*` proměnných v generic.css exponuje 1 505: jen kontejnerové `--pobo-widget-*` (bg, padding, margin, radius, shadow, before/after) a 3 globální. Chybí 74 základních typografických a tlačítkových proměnných (`--pobo-font-family`, `--pobo-typo-h2-*`, `--pobo-btn-*`, `--pobo-link-*`, `--pobo-all-content-*`) a 1 549 elementových proměnných nativních widgetů (`--pobo-counter-header-font-size`, `--pobo-faq-*`, `--pobo-reviews-two-box-photo-size`…). Přesně ty, které dělají design. Navíc odpověď má 75 KB a v Claude Code se nevejde do limitu tool outputu, takže ji Claude ani nepřečte. Předchozí session si ty proměnné musela vyhrabat z generic.css ručně.
2. **Runtime omezení nejsou nikde popsaná.** `@import` je zakázaný, `@font-face` s externí URL taky, `--pobo-font-family` má default `var(--template-font)` a webfonty může dodat jen hostitelská stránka. Nasazené CSS Womanizeru proto skončilo na `-apple-system, "segoe ui"…` a `georgia`, i když Erotic City načítá Montserrat. To je ten „font se nevykresluje“.
3. **Katalog widgetů je čistě technický.** Žádná role, žádné „hodí se k / nehodí se k“, žádný doporučený počet položek. Hinty jsou generická angličtina, dva widgety mají stejné jméno (7 a 8), katalog používá třídy `rc-*`, kontrakt třídy `widget-*` a Claude musí mapovat sám.
4. **QA nástroje nevidí to, co bylo špatně.** `render_content_html` hlásí jen byte size a prázdné textové sloty. Na Womanizeru je 17 prázdných `<img src="">` (ikony parametrů, fotky recenzí, obrázek u counteru) a report hlásí `empty_widget_position: []`. Nehlásí ani opakování (widget 4 šestkrát, 103 dvakrát za sebou), ani rytmus. Screenshot vrací jen obrázek bez computed stylů.
5. **Neexistuje showcase workflow.** Skill `style-widgets` řeší „sladit widgety s eshopem“, `write-blog` obsah. Nikde není art direction, copy pravidla ani anti-pattern list. Vše to musel nést prompt.

Srovnání s Bambulí to potvrzuje: Bambule má 12 widgetů, 0 prázdných slotů, žádný widget opakovaný víc než 3× (jen FAQ). Womanizer 25 widgetů, 17 prázdných obrázků, widget 4 šestkrát jako „nadpis sekce“, dva 103 za sebou se stejným obrázkem.

## 2. Co dnes MCP je

- Tento repozitář je jen **plugin**: `TOOLS.md` (reference 84 toolů), 9 skillů, manifest. Logika všech toolů běží na backendu `api.pobo.space`. Změny toolů = backend, změny znalostí a workflow = tento repo.
- Pro showcase jsou relevantní tooly: `list_eshop`, `get_theming_contract`, `fetch_eshop_page`, `get_content_html`, `push_asset_css`, `screenshot_eshop_page`, `list_asset`, `get_content_widget_catalog`, `compose_content`, `add/edit/move/remove_content_widget`, `set_content_widget_image`, `list_content_media`, `list_icon`, `render_content_html`, `get_platform_content`, `get_content_history`/`revert_content`.
- Design znalosti dnes leží: (a) v hlavě uživatele a v promptu, (b) částečně ve skillu `write-blog` („repeatable widgets need DISTINCT items“, „concrete beats clever“), (c) nikde jinde.

## 3. Podrobná zjištění

Doplněno při kontrole skillu: AI style asset (`templates/<eshop>.css`) se na stránce načítá **před** `generic.css`, takže override na prostém `:root` prohrává s pozdější deklarací v generic.css. Skill `style-widgets` to uvádí jen jako „cascade fallback“; správně je to výchozí stav a `push_asset_css` nebo kontrakt by to měly říkat (`declared_on` by mělo doporučit `:root #pobo-all-content`).

### 3.1 Theming contract

| Skupina proměnných v generic.css | Počet | V kontraktu |
|---|---|---|
| `--pobo-widget-*` (kontejner widgetu) | 1 502 | ano |
| `--pobo-global-*` | 3 | ano |
| core: `--pobo-typo-*`, `--pobo-btn-*`, `--pobo-link-*`, `--pobo-font-family`, `--pobo-all-content-*` | 74 | **ne** |
| elementové nativní: `--pobo-counter-*`, `--pobo-faq-*`, `--pobo-gallery-*`, `--pobo-parameter-*`, `--pobo-reviews-*`… | 1 549 | **ne** |
| custom per eshop: `--pobo-bikemax-*`, `--pobo-eskomat-*`, `--pobo-custom-*`, `--pobo-amix-*`… | 1 635 | ne (správně) |

Důsledek: skill říká „variables first, direct selectors are a smell“, ale pro font, barvu nadpisu nebo velikost čísla v counteru kontrakt žádnou proměnnou neukáže. Claude buď píše přímé selektory (a skill mu to vyčítá), nebo stáhne 818 KB generic.css a hledá sám. Nasazené CSS Womanizeru má 311 `--pobo-*` overridů a 82 přímých selektorů, protože jinak to nešlo.

Druhý problém je velikost. 75 KB JSON překračuje limit tool outputu Claude Code, výsledek se uloží do souboru a Claude ho musí filtrovat přes jq. Bez `scope` parametru se nedá vyžádat jen část.

### 3.2 Runtime omezení (fonty, assety, JS)

Zjištěno z generic.css, TOOLS.md a nasazeného CSS:

- CSS asset: max 256 KB SCSS, bez `@import`, `url()` jen Pobo CDN, platformní CDN, relativní cesta nebo `data:image`. Tedy **žádný `@font-face` s Google Fonts ani jiným externím hostem**. `data:font/...` je také odmítnutý (jen `data:image`).
- `--pobo-font-family` má default `var(--template-font)`. Font tedy přebírá ze šablony eshopu, pokud šablona tu proměnnou definuje. Erotic City ji nedefinuje, definuje `--font-heading-family` a `--font-body-family` (Shopify) a načítá Montserrat.
- Na stránce eshopu jsou dostupné jen fonty, které načte hostitelská šablona. Showcase může použít jen ty, systémové fonty, nebo font nahraný na Pobo CDN (pokud to backend dovolí, zatím není cesta přes MCP).
- JS z MCP nejde nasadit vůbec. Widgety mají vlastní JS (FAQ toggle, galerie lightgallery).
- Obrázky: `set_content_widget_image` bere stock/uploaded/image_bank/ai. Není parametr pro crop, focal point ani alt. Crop se řeší jen CSS (`object-fit`, `object-position`) na `.rc-*__img`.
- Prázdný image slot renderuje `<img src="">` s `loading="lazy"`. generic.css ho neschovává. Zobrazí se jako broken image nebo prázdné místo podle prohlížeče.
- Rate limit 60 req/min.

Nic z toho MCP neříká v jednom místě. Část je roztroušená ve skillu `style-widgets`, část jen v chování serveru.

### 3.3 Katalog widgetů

50 widgetů pro eshop 4679. Každý má `widget_id`, `name`, `ai_widget_type` (section/benefit/parameter/faq/video/testimonial), `repeatable {min,max}`, `image_slot`, `icon_slot`, `item_count`, `text_slot[]` s `role`, `class`, `hint`, `max_length`.

Co chybí nebo mate:

- **Žádná design metadata.** Role widgetu na stránce, vizuální váha, k čemu se hodí, s čím se nekombinovat, kolik položek reálně unese.
- **Mapování na CSS.** Katalog dává `rc-counter__header`, render dává `widget-container widget-counter`, kontrakt dává `widget-counter` a proměnné `--pobo-counter-*` i `--pobo-widget-counter-*`. Tři jmenné systémy pro jednu věc; katalog by měl u každého widgetu vrátit `css_block: "widget-counter"` a prefixy proměnných.
- **Nekonzistence.** Widget 7 i 8 se jmenují „Text v pravo a větší obrázek vlevo“ (8 je zrcadlově). Widget 89 se jmenuje „parametry v jednom sloupci“, ale má 5 ikonových slotů, které bez ikon renderují prázdné `<img>`. Widget 39 má `benefit_text` dvakrát. Widget 163 má v `class` uvozovku navíc. `repeatable {min:3,max:6}` u widgetu 103 s `item_count: 2` se čte jako „3 až 6 položek“, ve skutečnosti asi znamená kolikrát se widget smí opakovat v popisku.
- **Hinty anglicky a generické** („Short name of the benefit (for example "Quick to install")“) i když `write_in_language: "cs"`.
- **Bez náhledu.** Claude nevidí, jak widget vypadá. Zvolí ho podle názvu.

### 3.4 QA nástroje

`render_content_html` vrací HTML (25 KB u Womanizeru) a report `byte_size`, `widget_count`, `empty_widget_position`, `over_size_limit`. Nevidí:

- prázdné image/icon sloty (`<img src="">`), na Womanizeru 17 kusů,
- stejný obrázek ve více widgetech (103 a 103 a 86 sdílejí `FIiXQaztYR4RSG6k7Edg`),
- opakování stejného `widget_id` za sebou nebo celkově (4× šestkrát, 31× šestkrát, 103 dvakrát adjacentně),
- délku textů vůči `max_length`,
- odkazy typu „Koupit“ vedoucí na stejnou stránku, na které popisek je (fake CTA),
- slova jako „skladem“, „ihned k odeslání“ (statická dostupnost).

`screenshot_eshop_page` vrací PNG a CDN URL. Žádné computed styly, žádné rozměry, žádné „font fallback detected“. Každá kontrola stojí obrázek v kontextu a Claude z něj hádá pixely.

Není tool, který by vrátil **nasazené CSS** (`list_asset` jen jméno a velikost). Po kompakci kontextu Claude neví, co pushnul, a push vyžaduje kompletní soubor.

### 3.5 Skills

- `style-widgets`: workflow „sladit s eshopem“, dobrý pro merchanta, pro showcase zavádějící (krok 3 „extract tokens from the eshop“ je pro art direction jen jeden ze vstupů). Neobsahuje runtime omezení fontů kromě „set font-family variables to font names“.
- `write-blog`: obsahuje užitečné zásady (distinct items, concrete beats clever, get_platform_content first, look before you buy images), ale nic o struktuře stránky, rytmu, počtu widgetů, CTA, dostupnosti, češtině.
- Neexistuje `showcase` skill. Neexistuje soubor design guidelines. Neexistuje copy guideline pro češtinu.

## 4. Kam která znalost patří

| Znalost | Kam | Proč |
|---|---|---|
| Fakta o eshopu (fonty na hostu, tokeny, katalog, kontrakt) | **tool response** | mění se per eshop, musí být z dat |
| Runtime omezení (co CSS smí, fonty, JS, image sloty) | **tool response** (`get_runtime_constraints` nebo sekce v kontraktu) + zrcadlo ve skillu | jsou to vlastnosti serveru, server je má vracet; skill jen říká „přečti si je“ |
| Design metadata widgetu (role, váha, kombinace, počet položek, chyby) | **katalog widgetu** (backend), do té doby statický soubor v pluginu | patří k widgetu, ne k eshopu; Pobo ho zná, uživatel ne |
| Showcase guidelines (anti-patterns, preferuj méně silných sekcí, žádné fake CTA) | **statický soubor v pluginu** načítaný skillem | mění se s vkusem týmu, ne s kódem; verzovaný v gitu |
| Copy pravidla pro češtinu | **statický soubor v pluginu** | stejné |
| Workflow (pořadí kroků, kdy screenshot, kdy compose) | **skill** | to je definice skillu |
| Strukturální QA (prázdné sloty, opakování, CTA, stock slova) | **tool response** (`render_content_html` nebo `inspect_page`) | deterministické, levné, server to vidí v datech |
| Vizuální QA (crop, zarovnání, fallback fontu) | **tool response** (screenshot + metriky) | jen prohlížeč to změří |
| Brief produktu, brand, cílovka, co zdůraznit | **prompt uživatele** | jediná věc, kterou má znát jen uživatel |

Cíl: prompt „Udělej premium showcase Womanizer NEXT pro Erotic City v Pobo“ má stačit, protože zbytek přijde z prvních tří řádků tabulky přes skill.

## 5. Návrhy změn

Formát: problém / příčina / změna / schéma / příklad odpovědi / dopad / tokeny / složitost / priorita.

### A. `get_theming_contract`: správný rozsah a filtrování (backend)

- **Problém:** kontrakt neobsahuje typografii, tlačítka ani elementové proměnné; je 75 KB a nečitelný.
- **Příčina:** filtr `--pobo-widget-*` + `--pobo-global-*`; žádný `scope`.
- **Změna:** zahrnout `core` (74 proměnných) a elementové proměnné nativních widgetů; vyloučit per-eshop custom prefixy (seznam je znám: bikemax, eskomat, custom, amix, zkeshop, vitie, blendea, vitalcountry, mbh, nv, ambeauty, bean, venira, sijemesrdcem…). Přidat parametry `scope` a `block`.
- **Schéma:** `get_theming_contract(eshop_id, scope?: "core"|"blocks"|"block", block?: string[])`.
  - `core` → 74 core proměnných s defaulty + 3 global (asi 3 KB).
  - `blocks` → jen seznam 86 bloků s počtem proměnných a `variable_prefix[]` (asi 4 KB).
  - `block: ["widget-counter","widget-faq"]` → všechny proměnné těch bloků, kontejnerové i elementové, seskupené (`container`, `element.header`, `element.info`, `element.img`).
- **Příklad:**
  ```json
  {"block":"widget-counter","catalog_widget_id":[103,104],
   "container":{"--pobo-widget-counter-bg":"none","--pobo-widget-counter-padding":"var(--pobo-global-widget-padding)"},
   "element":{"header":{"--pobo-counter-header-font-size":"...","--pobo-counter-header-font-color":"..."},
              "img":{"--pobo-counter-img-width":"...","--pobo-counter-img-position":"..."}}}
  ```
- **Dopad:** Claude styluje přes proměnné, jak skill chce; odpadne stahování generic.css (818 KB) a hádání jmen.
- **Tokeny:** z ~20 000 (nečitelných) na 1 000 až 3 000 na dotaz.
- **Složitost:** nízká (změna filtru a group-by na existujícím parseru).
- **Priorita:** P0.

### B. `get_runtime_constraints` (backend) + zrcadlo ve skillu

- **Problém:** Claude navrhuje webfonty, `@font-face`, JS, dynamický stock.
- **Příčina:** omezení jsou roztroušená nebo jen v chování serveru.
- **Změna:** nový read-only tool, nebo sekce `runtime` v odpovědi `get_theming_contract(scope:"core")`. Vrací statická pravidla serveru a **dynamicky zjištěné fonty hostitelské stránky** (server už umí `fetch_eshop_page`: stačí z HTML vytáhnout `<link rel=preload as=font>`, `@font-face` v inline CSS, `font-family` deklarace a Google Fonts linky).
- **Schéma:** `get_runtime_constraints(eshop_id, path?: "/")`.
- **Příklad:**
  ```json
  {"css":{"max_scss_bytes":262144,"import":false,"font_face_external":false,"url_allowlist":["image.pobo.space","cdn.myshoptet.com","cdn.shopify.com","data:image/*"]},
   "js":{"deployable":false,"widget_js":["faq toggle","lightgallery"]},
   "font":{"pobo_default":"var(--template-font)","template_font_defined":false,
           "host_fonts":[{"family":"Montserrat","weights":[400,500,600,700],"source":"fonts.shopifycdn.com"}],
           "safe_fallback":["system-ui","Georgia"],
           "rule":"Use only host_fonts or system fonts. No webfont can be added from Pobo CSS."},
   "image":{"empty_slot_renders":"<img src=\"\">","crop_control":"css object-fit/object-position only","alt":"entity name, not editable"},
   "content":{"dynamic_stock":false,"add_to_cart":false,"links":"static <a> only"},
   "rate_limit_per_min":60}
  ```
- **Dopad:** přímo odstraní chyby 1 (font), 6 (fake CTA), 7 (skladem).
- **Tokeny:** ~600 na volání, jednou za session.
- **Složitost:** nízká pro statickou část, střední pro detekci fontů.
- **Priorita:** P0 (statická část klidně hned do skillu, viz F).

### C. Design metadata v katalogu widgetů (backend, mezikrok statický soubor)

- **Problém:** Claude volí widget podle názvu, kombinuje 103+103, používá 4 jako oddělovač šestkrát, nechává prázdné ikonové sloty.
- **Příčina:** katalog nese jen technické schéma.
- **Změna:** rozšířit záznam katalogu o blok `design`. Do doby backendové změny stejný obsah jako `skills/design-description/widgets.md` v pluginu (asi 50 záznamů × 6 řádků).
- **Schéma (přidané klíče):**
  ```json
  {"widget_id":103,"css_block":"widget-counter","variable_prefix":["--pobo-widget-counter-","--pobo-counter-"],
   "design":{"role":"metrics","visual_weight":"high","best_for":"2 výrazné číselné údaje s krátkým doplněním",
     "item_guidance":"benefit_title = číslo nebo krátká hodnota (14, IPX7, 240 min), ne slovo",
     "max_per_page":1,"avoid_adjacent":[103,104],"pairs_with":[7,8,266,70],
     "image_slot_note":"1 slot, renderuje se jako pozadí/vedle boxů; nechat prázdný jen když je CSS připravené",
     "common_mistakes":["dva 103 za sebou","stejná váha všech čísel","slovní hodnoty místo čísel"],
     "mobile":"boxy pod sebe, číslo zmenšit --pobo-counter-header-font-size-sm"}}
  ```
- Dále opravit data: názvy 7/8, popis 89 (5 ikon), duplicitní role u 39, uvozovka u 163, hinty česky když `write_in_language: cs`, vysvětlit `repeatable`.
- **Dopad:** řeší problémy 5, 8, 9, 10, 12 systémově; Claude dostane pravidlo „max 1× na stránku“ s daty, ne z promptu.
- **Tokeny:** katalog vzroste z ~9 000 na ~15 000; kompenzovat parametrem `fields: "summary"|"full"` nebo `widget_id[]`.
- **Složitost:** střední (data), nízká (schéma). Statický soubor: nízká.
- **Priorita:** P0 jako soubor v pluginu, P1 do backendu.

### D. Strukturální QA v `render_content_html` (backend)

- **Problém:** report neodhalil 17 prázdných obrázků, opakování, fake CTA, stock.
- **Příčina:** kontroluje jen prázdné texty a byte size.
- **Změna:** rozšířit `render_content_html` (nebo nový `inspect_page`) o deterministické kontroly nad daty a HTML.
- **Schéma odpovědi (přidané):**
  ```json
  {"structure":[{"position":4,"widget_id":104,"css_block":"widget-counter","role":"metrics","text_chars":210,"image_filled":0,"image_slot":1}],
   "issue":[
     {"type":"empty_image_slot","position":[4,7,9,11,18],"count":17,"severity":"high"},
     {"type":"adjacent_same_widget","widget_id":103,"position":[5,6],"severity":"medium"},
     {"type":"widget_overuse","widget_id":4,"count":6,"max_recommended":2,"severity":"medium"},
     {"type":"duplicate_image","image":"FIiXQaztYR4RSG6k7Edg","position":[5,6,9],"severity":"low"},
     {"type":"self_link","position":1,"href":"https://www.eroticcity.cz/products/womanizer-next","note":"odkaz vede na stránku, kde popisek je","severity":"high"},
     {"type":"availability_claim","position":[],"pattern":"skladem|ihned k odeslání","severity":"high"},
     {"type":"over_max_length","position":9,"role":"parameter_value","chars":96,"max":100,"severity":"info"}],
   "rhythm":{"widget_count":25,"distinct_widget":11,"longest_run_same_type":6,"header_text_ratio":0.24}}
  ```
- **Dopad:** jedno volání místo screenshot smyčky pro polovinu chyb; přímo řeší 6, 7, 8, 10, část 3.
- **Tokeny:** +500 až 1 500 na volání; ušetří 2 až 4 screenshoty (každý několik tisíc tokenů obrazu + úvaha).
- **Složitost:** nízká až střední (regex, počítání, jednoduchá heuristika).
- **Priorita:** P0.

### E. Vizuální QA s metrikami: `inspect_page` / rozšíření `screenshot_eshop_page` (backend, prohlížeč)

- **Problém:** crop, zarovnání, fallback fontu, rozházené velikosti se z PNG hádají a uniknou.
- **Příčina:** screenshot vrací jen pixely.
- **Změna:** ve stejném headless prohlížeči po renderu spustit skript nad `#pobo-all-content` a vrátit metriky per widget. Parametr `metrics: true`, volitelně `screenshot: false` (jen data, bez obrázku).
- **Schéma odpovědi:**
  ```json
  {"viewport":"desktop","content_box":{"width":1248,"height":9800,"border_radius":"25.6px","background":"#efecea"},
   "widget":[{"position":5,"css_block":"widget-counter","box":{"y":2410,"height":520,"width":1248},
      "font":[{"selector":".rc-counter__header","requested":"Montserrat","rendered":"Helvetica Neue","fallback":true,"size":144,"line_height":132}],
      "image":[{"selector":".rc-counter__img","natural":[1600,1067],"rendered":[624,520],"object_fit":"cover","object_position":"50% 50%","crop_ratio":0.42,"note":"horizontálně oříznuto 42 %"}],
      "gap_to_next":48,"overflow_x":false}],
   "issue":[
     {"type":"font_fallback","family":"Montserrat","widget":[1,2,5,6]},
     {"type":"inconsistent_gap","values":[48,128,24,128,96],"note":"5 různých mezer mezi sekcemi"},
     {"type":"sibling_size_mismatch","position":[5,6],"height":[520,468]},
     {"type":"heavy_crop","position":[7],"crop_ratio":0.55}]}
  ```
- **Dopad:** řeší 1, 4, 11, 12, 14 měřením místo odhadu; s `screenshot:false` je kontrola textová a levná.
- **Tokeny:** ~1 500 až 3 000 textu místo obrázku; screenshot jen na finále.
- **Složitost:** střední (skript v Playwrightu, server už prohlížeč má).
- **Priorita:** P1.

### F. Skill `showcase` + statické soubory guidelines (tento repo, bez backendu)

- **Problém:** vše nese prompt; každý produkt se „promptuje k dokonalosti“.
- **Příčina:** neexistuje workflow ani znalostní soubory pro showcase.
- **Změna:** nový `skills/design-description/` (realizováno 2026-09-22 v užší podobě: jen `SKILL.md`, `runtime.md`, `widgets.md` jako faktická mapa; guidelines a copy pravidla uživatel odmítl jako rozhodování za něj, počty widgetů se nestanovují, platí „méně je více“ a vždy sladit s eshopem) s:
  - `SKILL.md`: workflow v pořadí, které minimalizuje iterace (viz kapitola 6), včetně pravidla „compose_content jednou, ne add po jednom“, „render + inspect před screenshotem“, „screenshot max 2× desktop, 1× mobile“.
  - `guidelines.md`: showcase design guidelines. Formulované jako zákazy a preference, ne šablona: žádné 01/02/03 číslování, dekorativní italic, generic card grid, opakované dvousloupce, badge bez významu, gradienty bez účelu, fake ecommerce stavy, statická dostupnost, purchase CTA bez košíku, závěrečná „výplňová“ sekce; preferuj méně silných sekcí, editorial kompozici, reálné produktové fotky, přirozenou češtinu, proměnlivý rytmus, hierarchii typografie; jeden surface s radiusem cca 1.6rem přes `--pobo-all-content-*`.
  - `copy-cs.md`: copy pravidla pro české ecommerce (konkrétní produktový jazyk, žádný doslovný překlad, žádné „objevte svět“, čísla a fakta z `get_platform_content`, tykání/vykání podle eshopu, délky vůči `max_length`). Odkaz na `stop-slop` skill.
  - `widgets.md`: design metadata z bodu C, dokud nejsou v backendu.
  - `runtime.md`: statická část bodu B, dokud není v backendu.
- **Dopad:** okamžitý, bez čekání na backend. Prompt uživatele se zkrátí na brief.
- **Tokeny:** skill + soubory ~6 000 až 8 000 tokenů načtených jednou; nahradí několikastránkový prompt a 3 až 5 opravných smyček.
- **Složitost:** nízká, jen psaní.
- **Priorita:** P0, první krok.

### G. `get_asset_css` (backend)

- **Problém:** po kompakci kontextu Claude neví, co nasadil; push vyžaduje celý soubor. Zároveň je to předpoklad pro bezpečný scoping stylu jedné entity na eshopu, kde už AI asset existuje.
- **Náhradní postup dnes:** kompilované CSS je veřejné na `https://image.pobo.space/templates/<domain>.css` (link v HTML z `fetch_eshop_page`), skilly ho čtou přes `curl`. Vrací kompilát, ne SCSS, takže se ztrácí vnoření a komentáře, ale pro zachování pravidel to stačí.
- **Změna:** `get_asset_css(eshop_id, asset_id?)` vrací aktuální SCSS zdroj (ne kompilované CSS).
- **Tokeny:** 45 KB SCSS ≈ 12 000 tokenů, ale jen když je třeba; jinak Claude přepisuje z paměti a dělá regrese.
- **Složitost:** velmi nízká.
- **Priorita:** P1.

### H. Obrázky: crop a výběr

- **Problém:** špatný výběr a crop obrázků.
- **Změna:** (1) `list_content_media` a `list_content_image` vracet `width`, `height`, `aspect`, `dominant_color` a náhledovou URL; (2) `set_content_widget_image` přijmout `focal_point: [x,y]` nebo `object_position`, které render zapíše jako inline style na `<img>`; (3) v katalogu u každého image slotu uvést očekávaný poměr (`slot_aspect: "3:2"`, `render_mode: "cover"`), aby Claude vybral fotku správného tvaru.
- **Složitost:** (1) nízká, (2) střední, (3) nízká.
- **Priorita:** (1) a (3) P1, (2) P2.

### I. Page-level uvažování (`get_page_summary` v `get_content_widget`)

- **Problém:** widgety se řeší izolovaně.
- **Změna:** `get_content_widget` volitelně vrátí `summary: true`: kompaktní seznam `position / widget_id / role / css_block / chars / images_filled` bez textů (asi 30 řádků místo 8 000 tokenů textu). Dohromady s D `rhythm` je to vstup pro úvahu o celku.
- **Složitost:** nízká.
- **Priorita:** P1.

## 6. Cílový workflow a odhad úspor

Dnešní průběh (Womanizer): research → art direction → struktura → implementace po widgetech → screenshoty → opravy v 5 až 7 smyčkách → 3 h, 68 % limitu, Fable 5.1 High.

Cílový průběh se skillem `showcase` a tooly A, B, D:

1. `list_eshop`, `get_runtime_constraints`, `get_theming_contract(scope:"core")`, `get_content_widget_catalog` (s metadaty), `get_platform_content`, `list_content_media`. Šest read volání, ~12 000 tokenů.
2. Brief od uživatele → plán widgetů s rolí a obrázky, méně je více. Bez volání.
3. `compose_content` jednou (preview + confirm), obrázky z knihovny přes `image` v plánu.
4. `render_content_html` s rozšířeným reportem → oprava strukturálních chyb `edit_content_widget` / `set_content_widget_image`.
5. `get_theming_contract(block: [použité bloky])` → jedno kompletní SCSS → `push_asset_css`.
6. `inspect_page(metrics:true, screenshot:false)` desktop + mobile → oprava CSS → jeden finální screenshot desktop, jeden mobile.

Odhad: 20 až 30 MCP volání místo 60+, 2 až 3 screenshoty místo 10+, jedno až dvě opravná kola. Cíl pod 60 min a pod 25 % limitu na Sonnet 5 High. Bambule (2 h, lepší výsledek) ukazuje, že méně widgetů a jasnější struktura už teď dávají lepší výstup; skill to udělá výchozím stavem.

## 7. Modely

Souhlasím se Sonnet 5 High jako výchozím pro celý showcase běh bez přepínání. Body D a E přesouvají QA z obrazové úvahy do textových dat, což je přesně místo, kde dražší model přinášel nejmenší rozdíl. Fable ponechat na tvorbu guidelines a metadat (jednorázově), ne na produkci stránek.

## 8. Doporučené pořadí implementace

1. **Tento repo, hned:** `skills/design-description/` se všemi pěti soubory (F). Ověřit na jednom novém produktu se Sonnet 5 High a promptem o třech větách.
2. **Backend, malé:** A (`scope`/`block` v kontraktu, zahrnout core a elementové proměnné), D (strukturální report), G (`get_asset_css`). Každé je do dne práce.
3. **Backend, střední:** B s detekcí host fontů, C do katalogu, I, H(1)+(3).
4. **Backend, větší:** E (metriky z prohlížeče), H(2) focal point.

Po kroku 2 přepsat `skills/design-description/widgets.md` a `runtime.md` na „volej tool“, aby se znalost nedublovala.

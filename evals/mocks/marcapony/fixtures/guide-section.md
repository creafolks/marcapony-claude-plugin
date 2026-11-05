## How a custom-code section renders
- Write a React functional component in `sourceJsx` and styles in `sourceCss`
- The component is compiled, server-rendered, and hydrated **directly into the live page** — there is NO iframe and NO style/script isolation. Three consequences: (1) your CSS is injected into the page as an inline `<style>`, so unnamespaced class names can collide with the rest of the site — namespace custom selectors; (2) the component is server-rendered first, so top-level `window`/`document` access throws — read them inside `useEffect` or behind a `typeof window !== 'undefined'` guard; (3) the page's theme tokens and fonts DO apply, so Tailwind theme classes work without redeclaring anything. (The admin preview does use an iframe; the published site does not.)
- To tell `render_section_preview` your client effects are done (instead of it waiting out its time budget), set `data-marcapony-ready="true"` on any element you render, or dispatch `window.dispatchEvent(new Event('marcapony:ready'))`, from the end of your effect; start it as `"false"` or leave it off.
- React, useState, and useEffect are available globally — do NOT add import statements
- End JSX with: `export default ComponentName`

## Styling — Prefer Tailwind CSS
- **Tailwind CSS v4 is available** on the page. Prefer Tailwind utility classes over custom CSS.
- Theme tokens: `bg-primary`, `text-secondary`, `text-muted-foreground`, `bg-accent`, `text-destructive`, etc. (full list under "Theme tokens" below)
- Theme fonts: `font-heading`, `font-body`
- Theme radius: `rounded-sm`, `rounded-md`, `rounded-lg`, `rounded-xl`
- Use `sourceCss` only for styles that Tailwind cannot achieve (animations, complex selectors, etc.)
- You can mix Tailwind classes with custom CSS when needed.
- Classes are generated at compile time from the complete class names written in `sourceJsx`. A class assembled at runtime (`bg-${color}`, `text-[${size}]`) is never generated and renders unstyled. Write full static class names, and use an inline `style` for values that come from inputs.

## Theme tokens
A section that must follow the site's theme (every platform library section) uses only these classes for colour, font and radius. They follow the live theme: a theme change reaches published pages without recompiling the section.
- Colours: `background`, `foreground`, `primary`, `primary-foreground`, `secondary`, `secondary-foreground`, `muted`, `muted-foreground`, `accent`, `accent-foreground`, `destructive`, `destructive-foreground`, `link`, `border`, `input`, `ring`, plus the surface pairs `card` / `card-foreground` and `popover` / `popover-foreground`. The surface pairs follow the card and popover colours the theme stores; where none is stored they equal `background` / `foreground`, and so they do on older templates that do not write them. Use them with any colour utility (`bg-primary`, `text-muted-foreground`, `bg-card`, `text-card-foreground`, `border-border`, `ring-ring`, `from-primary`, `fill-foreground`). An opacity modifier works on all of them: `bg-primary/20`, `text-foreground/70`.
- Fonts: `font-heading` for headings, `font-body` for everything else. They are separate tokens, currently the same font until a heading font exists, and fall back to the theme's font family (`--font-family`).
- Radius: `rounded-sm` is the theme's button radius, and `rounded-md`, `rounded-lg` and `rounded-xl` derive from its card radius (1x, 1.5x, 2x), both only when the account stores a border radius; otherwise they keep Tailwind's defaults. `rounded-full` and `rounded-none` never change.
- Spacing is fixed when the section compiles, and so are `success` and the `chart-*` colours; avoid those colours in a section that must follow the theme.
- In hand-written CSS (`sourceCss`, inline `style`) use the theme variables, always with a fallback because older templates lack some of them: `rgb(var(--theme-card, var(--background, 255 255 255)))`, `rgb(var(--theme-card-foreground, var(--foreground, 17 17 17)))`, `rgb(var(--theme-popover, var(--background, 255 255 255)))`, `rgb(var(--theme-popover-foreground, var(--foreground, 17 17 17)))`, `var(--font-heading, system-ui)`, `var(--font-body, system-ui)` and `var(--radius-button, 0.375rem)`. The other colours take the same form: `rgb(var(--primary, 37 99 235))`. Do not use `--card`, `--card-foreground`, `--popover` or `--popover-foreground`: they belong to the template's own components.
- Hard-coded colours break theme following: hex, `rgb()`, `hsl()`, named colours, Tailwind palette classes (`bg-blue-500`, `text-gray-600`, `bg-white`) and arbitrary values (`bg-[#2563eb]`) stay fixed when the account changes its theme. The same goes for `style` props and `sourceCss`. The compiler reports them as a non-blocking `hardcoded_colors` warning on the section; read it and swap the literals for tokens.
- A `color` input is for a colour the editor chooses; its default should be a theme reference (`rgb(var(--primary))`), not a hex.

## Fonts
- Use the theme fonts: `font-heading` and `font-body`, or `var(--font-heading)` and `var(--font-body)` in `sourceCss`. The site already loads them.
- Do not paste `@font-face` blocks or a Google Fonts `@import`. A family outside the theme is allowed when the design needs it, but it is an extra download on every page view.
- When you do use another family, the compiler copies files from `https://fonts.gstatic.com` and unconditional `@import`s from `fonts.googleapis.com` to the site's own CDN, up to 24 files per section. An `@import` with a media, layer or supports condition, files past that limit and other font hosts stay remote. It also adds metric-adjusted fallbacks to the font stacks in your CSS for each family it has metrics for (not monospace) whose webfont the same CSS declares or the theme loads, so the text keeps its size and line breaks while the webfont loads. Write a normal stack in `sourceCss` with a generic family at the end (`'Cormorant Garamond', Georgia, serif`).

## Images and video
- Images: write `<img src={photo} alt="…">` with the input value as `src`. The compiler adds a `srcSet` over our CDN's sizes, so phones download a phone-sized file.
- Set `sizes` when the image is not full width, e.g. `sizes="(min-width: 768px) 33vw, 100vw"` for one card in a three-column grid. Without it the browser assumes full width.
- Do not set `loading` or `fetchpriority` yourself: the template decides per page, promoting the first section's image and lazy-loading the rest. Never put `loading="lazy"` on a hero image; it delays the largest paint.
- Video: give every `<video>` a `poster`. For a video input the compiler adds the stored frame when you leave it out; any other URL needs one from you.
- A muted background loop is `<video src={clip} autoPlay muted loop playsInline>`. With `clip` a video input, leave `poster` out so the full-size frame is added. A `muted loop` video whose `src` is an input, without `controls` and of 60 s or less, gets the light loop file our encoder made for it. The compiler also turns `autoPlay` into "play while on screen", which never starts for visitors who ask for reduced motion or data saving, but only for a `<video>` with no `ref`, no `{...spread}` and a plain `src` (an input or a literal, not a function call). If you attach your own `ref` and observer, write `preload="none"`, start it yourself and check `prefers-reduced-motion` yourself.
- Reveals: never hide content in the markup or right after mount (`opacity: 0` or `invisible` until an observer, a timeout or an effect adds a class). On a slow phone the content stays hidden until the script runs, and a hidden hero delays the largest paint. Put `data-mp-reveal` on the element instead (`data-mp-reveal-delay="1"` to `"4"` to stagger siblings: each step starts the fade a little later in the scroll): the template fades it in with CSS as it scrolls into view and never hides what is fully in view at load. It needs a website version that ships the rule; on an older one the save is refused with `TEMPLATE_FEATURE_NOT_AVAILABLE`, and then drop the attribute and leave the content static rather than writing a hide-then-show effect. Put it on content, not on a wrapper of a drawer, modal or dropdown: while it fades in, the element is the containing block of anything `fixed` inside it. The template turns Tailwind's `overflow-hidden` on its ancestors into `overflow: clip` so the fade can follow the page scroll; in your own CSS write `overflow: clip`, not `overflow: hidden`, on an ancestor of a revealed element.

## Layout & spacing
- The published page already wraps this section and applies the **vertical** padding itself, from `settings.sectionHeight`: `short` → `py-8`, `medium` (the default) → `py-16`, `tall` → `py-24`, `full` → `min-h-screen flex flex-col justify-center py-24`.
- `settings.removeTopSpacing` and `settings.removeBottomSpacing` (boolean, or the string `"true"`) zero the top and bottom padding respectively.
- So the component sets **horizontal** padding only (`px-6`). Giving it its own `py-16` or `min-h-screen` doubles the spacing and pushes the content past the viewport.
- If the component must own its vertical rhythm (a full-bleed hero, for instance), set both `removeTopSpacing` and `removeBottomSpacing` rather than working against the wrapper.
- The page shell is always at least one viewport tall, so a single short section leaves visible empty space below it — for a one-section page use `sectionHeight: "full"`.

## Input variables (STRONGLY RECOMMENDED)
Define inputs for ALL user-facing text so the section is editable without code changes:
- In JSX, reference as: `{variableName}`
- In CSS, reference as: `{{variableName}}`

Input types:
- `text`: Short text (titles, labels, button text)
- `textarea`: Longer content (paragraphs, descriptions, bios)
- `image`: Image URL from the media library
- `video`: Video URL from the media library
- `color`: A color value — a hex (e.g. "#2563eb") or a theme-color reference `rgb(var(--token))` that binds to the account theme (e.g. "rgb(var(--primary))"). Theme references resolve live on the published site and follow theme edits. Valid tokens: `--primary`, `--primary-foreground`, `--secondary`, `--secondary-foreground`, `--accent`, `--accent-foreground`, `--background`, `--foreground`, `--muted`, `--muted-foreground`, `--destructive`, `--destructive-foreground`, `--border`, `--input`, `--ring`, `--link`. Usable in both JSX and CSS.
- `object`: JSON arrays/objects (e.g. list of features, pricing plans) — value must be valid JSON string, injected as raw JS variable
- `radio`: Single-choice from a predefined set. Requires `options: [{label, value}]`. `value` must match one of the option values. Injected as a string.
- `dropdown`: Same as `radio` but rendered as a dropdown in the editor. Requires `options`.
- `checkbox`: Boolean toggle. `value` must be the string `"true"` or `"false"` — it is injected into JSX as a real JS boolean (e.g. `{showBadge && <Badge/>}`).
- `entity`: Link to an internal entity (product, category, post, or page). Requires `entityType` plus `value` set to the entity id. The media-processor resolves this to a relative URL at compile time, and slug changes cascade-recompile affected sections. In JSX, use the variable as a string: `<a href={productLink}>...`.
- `datetime`: A date and/or time picker. `value` is the raw control string (`"2026-06-14T15:30"`, or just a date/time). Set `datetimeMode` to `"date"`, `"time"`, or `"datetime"` (default). Injected as a string — parse with `new Date(variableName)` if you need a Date object.
- `goto`: A same-page section link. Set `value` to the id of a target section on the SAME page (get the section ids from `get_page_details`). Do NOT read it as a value — put it on a clickable element as an attribute: `<button data-mp-goto={inputKey}>Go</button>`. Clicking smooth-scrolls the visitor to that section.
- `group`: A named, schema'd, **repeatable** collection — a structured alternative to a raw-JSON `object` input. Declare the columns in `fields` and the rows in `items`, and set `value` to `""`. In JSX the group is a JavaScript array bound under its `key`, consumed with `.map()` (e.g. `{features.map((f, i) => <li key={i}>{f.title}</li>)}`). See "Group inputs" below.
- `icon`: An Iconify icon. `value` is a `"prefix:name"` reference (e.g. `"mdi:home"`, `"lucide:arrow-right"`). The compiler fetches the inline SVG and bakes a real element, so `{iconKey}` renders the icon directly in JSX. Do NOT wrap it in `<img>`. Size and color it with Tailwind on a parent (e.g. `<span className="h-6 w-6 text-primary">{iconKey}</span>`). In CSS, `{{iconKey}}` resolves to the Iconify CDN URL instead, for `mask-image` / `background-image`.
- `threed`: The URL of a 3D model from the media library (.glb / .gltf). Find one with `search_images` (`fileType: "threed"`) or `get_asset_details`: the result's URL is the value, its poster is the thumbnail. Render it declaratively with `<model-viewer src={modelKey} camera-controls auto-rotate />`, or take full control with `const { THREE, OrbitControls, loadModel } = await window.mpLoadThree()` inside `useEffect`. Both loaders are installed by the page at runtime and lazy-loaded only when a section uses them. Array mode is NOT supported.
- `tracking`: An analytics event fired when an element is clicked. Never read as a value. Attach it with `data-mp-track="<inputKey>"` (see "Element attributes" below). `value` is a JSON string with a required non-empty `eventKey`, plus optional `value` (number), `currency` (string) and `properties` (flat object): `'{"eventKey":"cta_click","value":10,"currency":"USD"}'`. An empty `eventKey`, or a `data-mp-track` pointing at a non-tracking input, is a HARD compile failure for the whole section. Events only reach analytics on sites where tracking is enabled and the visitor has consented; the element still renders and behaves normally either way.
- `collection`: A **live list of products** the component renders itself — the code alternative to the built-in product grid. Set `entityType: "product"` and `value: ""`; what it lists lives in `collection`. In JSX the variable is an object, not an array: iterate `key.items`. See "Collection inputs" below.
- `action`: A behaviour fired when an element is clicked. Never read as a value. Set `value` to `""` and attach it with `data-mp-action="<inputKey>"` (see "Actions" below). Requires `actionType` plus the one `actionConfig` field that type needs.

## Optional per-input fields
- `labels`: `{ "tr": "Başlık", "en": "Title" }` — per-locale display label shown to editors. Purely cosmetic; does not affect runtime values.
- `purpose`: one line on what the input is for, e.g. `"headline, 4 to 8 words"`. The editor shows it as help text under the field, and AI reads it when it fills the section. Set it on every `text` and `textarea` input.
- `maxLength`: a number, for `text` and `textarea` only: a soft character limit. The editor shows a counter and warns past it; saves are never rejected. Set it on every `text` and `textarea` input, matching the length in `purpose`.
- `example`: a sample value showing the expected style and length. Group `fields` take all three too.
- `help`: `{ "tr": "Ana başlık, 4 ila 8 kelime" }`, `purpose` in the interface languages (`en`, `tr`); the editor shows it in place of `purpose`. An option takes `labels` the same way for its `label`. Optional, and AI reads only the English. When you send `inputs` back, keep both as you found them; drop one only when you rewrite the English it translates.
- `options`: `[{ label: "Primary", value: "primary" }]` — required for `radio` and `dropdown` inputs. Defines the allowed choices; the input `value` must match one of the option values.
- `entityType`: `"product"` | `"category"` | `"post"` | `"page"` — required for `entity` inputs. Tells the compiler what kind of internal entity the `value` id refers to. Also required for `collection` inputs, where only `"product"` works today.
- `collection`: required for `collection` inputs — `{ filter, categoryId?, ids?, pageSize, pagination }`. See "Collection inputs" below.
- `datetimeMode`: `"date"` | `"time"` | `"datetime"` — only for `datetime` inputs; selects which part of the timestamp the picker captures. Defaults to `"datetime"`.
- `actionType` / `actionConfig`: for `action` inputs; `actionType` is required, the `actionConfig` target is optional. See "Actions" below.

Example of an input that carries the AI metadata:

```
{ "key": "headline", "type": "text", "value": "Fresh bread, baked daily", "labels": { "en": "Headline" }, "purpose": "headline, 4 to 8 words", "maxLength": 60, "example": "Handmade pastries every morning" }
```

## Sites in several languages

A page translated into another language gets its own copy of each section. A copy of a custom-code section takes its code from the section in the default language. Only `text`, `textarea`, `image`, `video`, `object`, `group`, `datetime`, `entity` and `app` values, and an action's `message` and `href`, are set per language; the source code, the input definitions, the value of every other input type (today `dropdown`, `radio`, `checkbox`, `color`, `icon`, `tracking`, `goto`, `upload`, `collection`, `booking` and `threed`) and an action's type and target follow the source and cannot be changed on the copy. A custom-code section created directly on a page in another language, rather than by translation, is not a copy: its code is edited where it is. So put a design choice that must look the same in every language (a layout variant, a colour, an icon) in one of the shared types, and anything a translator should change in a per-language one. `get_section_details` says when a section is such a copy and names its source.

## Element attributes (not input values)
Some inputs are used as JSX/HTML attributes on an element rather than as a `{value}`:
- `data-mp-track="<inputKey>"` — mark a clickable element to fire an analytics event. The `<inputKey>` must reference a `tracking` input; the compiler resolves it to the configured event key. Pointing it at any other input type fails the compile.
- `data-mp-goto={<inputKey>}` — mark a clickable element to scroll to another section on the same page. The `<inputKey>` must reference a `goto` input whose `value` is the target section id.
- `data-mp-action="<inputKey>"` — mark a clickable element with an `action` input. Note the quotes: this one is a **string literal**, not a JSX expression. The compiler replaces it with the resolved action type and its config. The `<inputKey>` must match an action-type input's key exactly; a typo or a non-action key compiles to a permanently dead element with no error (see "Actions").
- `data-mp-variant={combination?.productVariantId}` and `data-mp-qty={quantity}` — on a `buy` / `addToCart` element, the combination and quantity the visitor chose. These ARE JSX expressions, set from React state on every render, so the click always reads the current choice (see "Product Template pages").

## Actions (clickable behaviour, incl. form submit)
An `action` input attaches a behaviour to one clickable element. Set `type: "action"`, `value: ""`, an `actionType` and the `actionConfig` target for that type, then put `data-mp-action="<inputKey>"` on the element (a `<button>` is preferred: the submit action disables it while the request is in flight).

| `actionType` | `actionConfig` target | What the click does |
|---|---|---|
| `submit` | `formId` | Collects the surrounding form's values and posts them as a form submission |
| `goto` | `targetSectionId` | Smooth-scrolls to that section on the same page |
| `whatsapp` | `phone` (+ optional `message`) | Opens wa.me in a new tab |
| `tel` | `phone` | Starts a phone call |
| `mailto` | `email` | Opens the visitor's mail client |
| `addToCart` | none | Adds the page's product to the cart (product pages) |
| `buy` | none | Buys the page's product (product pages) |
| `productCta` | none; the input binds `source: "product"`, `productField: "ctaMode"` | Does whatever the product's own CTA mode says (product pages) |
| `contact` | none | Opens the contact dialog (product pages) |

For `submit`, `goto`, `whatsapp`, `tel` and `mailto`, set the target when you add or change the action. A missing target is accepted only so that sections from a template stay editable; such an action does nothing on click until the target is set (the tool reply carries a warning for each one).

`buy`, `addToCart`, `productCta` and `contact` work on Product Template pages, where the page knows its product. On any other page type they are accepted and compile, but the click does nothing (the page has no product to act on), so put them only on Product Template pages; to sell from a list page, use a `collection` input and its `addToCart`. They need a site on a current template build: on a site still on an older build the element renders but the click does nothing, so check the button on the published page. `openForm` exists in the data model but the published site has no handler for it yet, so a button using it would do nothing.

The platform enforces the purchase gates, so do not re-implement them: the storefront must be on, the product in `buy` CTA mode, on standard checkout, and in stock. With the storefront OFF, the product in `contact` CTA mode, or the product sold through a reservation flow, `buy` and `addToCart` open the contact dialog instead. A `data-mp-variant` that is not an active combination of THIS product is refused with the same "choose a variant" message below rather than quietly buying another row. On a combination product, a `buy` / `addToCart` element without `data-mp-variant` uses the product's default combination, and when there is none the visitor is shown a "choose a variant" message.

A `data-mp-action` whose key does not resolve to an action input (a typo, or a key that belongs to a different input type) is a SILENT no-op: the attribute is left in place, no error is raised, and the element never does anything. This is unlike `data-mp-track`, which hard-fails the compile. So the key on the element and the input's `key` must match exactly.

### The `submit` action: form binding contract
Three things must line up, or the submission silently drops fields:

1. **Where `formId` comes from.** It is the id of a managed form that already exists on this account, never an id you invent, and never the page id, a `custom-form` section id or the custom-code section id. Fields live only on managed forms: list them with `call_read_endpoint` → `list_customer_forms`, create one with `call_endpoint` → `create_custom_form`. A `custom-form` page section renders such a form by reference — its `payload.formId` names the form — so a section is not a second source of ids; read the section's `formId` if you want the form a page already shows. (A section created before forms were managed still holds its fields inline and answers to its own id until it is migrated; `list_customer_forms` lists those with `source: "page"`.) The form's declared fields are the contract for step 2.
2. **Input naming.** Every value you want submitted must sit on a native form control (`input` / `textarea` / `select`) carrying `name="<fieldKey>"`, or on any element carrying `data-mp-field="<fieldKey>"`, where `<fieldKey>` is the `key` of a field declared on that form (legacy section-built forms fall back to the field's `id`). A bare `name` on a non-native element is ignored; use `data-mp-field` for a custom widget. Keys that do not match a declared field are still submitted and stored on the submission, and they still appear in the merchant's admin notification email, but they are NOT sent to the form webhook or the customer confirmation email and are never stored as structured, labelled fields. Every field the form marks `required` must be present, or the API rejects the submission.
3. **What the button attaches at runtime.** On click, the runtime walks up from the button to the nearest `<form>` (falling back to the section's own container) and collects every `name`-bearing native form control and every `data-mp-field` element inside it. Native controls contribute their real DOM value: text inputs and `<textarea>` their string, a lone checkbox a boolean, 2+ checkboxes sharing one name an array of the checked values, radios the checked value, `<select multiple>` an array. Disabled controls are skipped. A `file` field is bound with `<input type="file" name="<fieldKey>">`, or with `data-mp-field="<fieldKey>"` on the file input itself or on a wrapper element that contains one; at submit time the runtime uploads the first selected file (only the first, even when the input has `multiple`) and sends its reference for that key, and an empty picker simply omits the key. The accepted file types and the size limit come from the field's definition on the form, not from your markup, so never upload the file yourself or add a `fetch`. It then posts the result, shows a localized success or error toast, and resets the fields on success. Pressing Enter inside the form triggers the same path.

Wrap the fields in a real `<form>` so the boundary is unambiguous, and do NOT add your own `onSubmit` or `fetch`. The runtime owns the submission.

```
<form className="mx-auto max-w-md space-y-4 px-6">
  <input name="fullName" type="text" placeholder={namePlaceholder} className="w-full rounded-md border p-3" />
  <input name="email" type="email" placeholder={emailPlaceholder} className="w-full rounded-md border p-3" />
  <textarea name="message" rows={4} placeholder={messagePlaceholder} className="w-full rounded-md border p-3" />
  <button data-mp-action="contactSubmit" type="submit" className="w-full rounded-md bg-primary px-6 py-3 text-primary-foreground">{submitLabel}</button>
</form>
```
Matching input (`fullName`, `email` and `message` must be field keys declared on that form):
```
{ key: "contactSubmit", value: "", type: "action", actionType: "submit", actionConfig: { formId: "<managed form id>" } }
```

## Product Template pages: the `variants` product field
On a Product Template page an input can bind to a field of the page's product (`source: "product"` plus `productField`). Bind `productField: "variants"` to a single `object` input and the variable holds the product's variant model:
```
{ selectionMode: "none" | "single" | "combination",
  axes: [{ id, type, name, style: "swatch" | "buttons",
           values: [{ id, name, code, value, imageUrl, productVariantId }] }],
  combinations: [{ productVariantId, valueIds, price, stock, sku, imageUrl, isDefault, isActive }],
  resolve(selection), isValueAvailable(selection, axisId, valueId), defaultSelection() }
```
- A `selection` is `{ [axisId]: valueId }`. `resolve(selection)` returns the active combination it names, or `null`.
- `isValueAvailable(selection, axisId, valueId)` tells you whether that value can still be picked alongside the rest of the selection. A value whose matching combinations are all sold out (stock tracked and at 0) counts as unavailable; `resolve` still returns that combination, so its price stays readable. Use it to mark a choice unavailable, not to hide it.
- `defaultSelection()` is the selection to start from.
- `stock` is `null` unless the product tracks stock. On a `swatch` axis a value's `value` is its colour.
- **Server render and previews (including render_section_preview) use stubs: no axes, and `resolve()` returns `null`.** Render the pickers only when `axes.length > 0`, and keep the markup meaningful without a resolved combination.

To sell from the page, put an `addToCart` (or `buy`) action on a button and carry the choice on it as attributes: `data-mp-variant={combination?.productVariantId}` and `data-mp-qty={quantity}`. The platform reads them at click time and enforces the gates listed under "Actions".

```jsx
function ProductBuy() {
  const [selection, setSelection] = React.useState(() => variants.defaultSelection())
  const combination = variants.resolve(selection)
  const hasPicker = variants.axes.length > 0

  return (
    <div className="space-y-4">
      {hasPicker && variants.axes.map((axis) => (
        <div key={axis.id} role="radiogroup" aria-label={axis.name} className="flex flex-wrap gap-2">
          {axis.values.map((v) => {
            const available = variants.isValueAvailable(selection, axis.id, v.id)
            return (
              <button
                key={v.id}
                type="button"
                role="radio"
                aria-checked={selection[axis.id] === v.id}
                aria-disabled={!available}
                onClick={() => setSelection({ ...selection, [axis.id]: v.id })}
                className="rounded-md border px-3 py-1 aria-checked:border-primary aria-disabled:opacity-40"
              >
                {axis.style === 'swatch' && v.value
                  ? <span className="block h-5 w-5 rounded-full" style={{ background: v.value }} title={v.name} />
                  : v.name}
              </button>
            )
          })}
        </div>
      ))}
      <p className="text-lg font-semibold">{combination?.price ?? price}</p>
      <button
        type="button"
        data-mp-action="addToCart"
        data-mp-variant={combination?.productVariantId}
        data-mp-qty={1}
        disabled={hasPicker && !combination}
        className="rounded-md bg-primary px-6 py-3 text-primary-foreground disabled:opacity-50"
      >
        Sepete ekle
      </button>
    </div>
  )
}

export default ProductBuy
```
Here `variants` is the object input bound to the `variants` field, `price` a product-bound price input, and `addToCart` an `action` input: `{ key: "addToCart", value: "", type: "action", actionType: "addToCart" }`. The button is only disabled while a real picker has no resolved combination, so the server stub and a product without variants still render a usable button.

## Blog Template pages: post bindings
On a Blog Template page (page type `blog-template`, rendered for every post it targets) an input can bind to a field of the post being shown: `source: "post"` plus `postField`. One section then serves every post. `value` is used only where the post field is empty (and in the editor, which has no real post), so a non-empty `value` shows on every post missing that field; leave it `""` unless that is what you want. The api refuses post bindings on any other page type, and an action input never carries one.

| `postField` | Bind it with |
|---|---|
| `title`, `excerpt`, `metaDescription`, `authorName`, `authorTitle`, `categoryName` (the first category), `eventLocation` | `text` (or `textarea`) |
| `publishedAt`, `eventStartAt`, `eventEndAt` | `text`; the date arrives already formatted for the visitor's locale |
| `readingTime` | `text`; the number of minutes |
| `content`, `authorBio` | `textarea` |
| `coverImage`, `authorAvatar` | `image` |

- `content` and `authorBio` arrive as sanitised HTML. Render the variable directly, `<div className="mp-post-body">{content}</div>`, and it shows formatted. Never use `dangerouslySetInnerHTML`. The variable is a React element (a `display: contents` `<div>` holding the HTML), not a string: use it only as `{content}` in JSX, never with string operations (`.length`, `.slice()`, template strings), an `||` fallback or as an attribute value. Style the elements inside it from `sourceCss` with descendant selectors (`.mp-post-body h2 { … }`, not `.mp-post-body > h2`, because of that extra `<div>`) or with Tailwind arbitrary variants on the wrapper (`[&_h2]:text-2xl`).
- A post without a cover or an author without an avatar falls back to the input's `value`, so leave `value: ""` on a bound `image` input: then the image renders with `src=""`, while any other `value` appears as the cover of every post that has none. Keep the `<img>` in the markup and hide it with CSS (`.mp-post-cover img[src=""] { display: none; }`) instead of a JSX conditional such as `{coverImage && <img … />}`: the server and the browser must produce the same markup, and a conditional on a bound value does not. An empty `src` resolves to the page's own URL, so the browser still makes one extra request for it on every post without a cover (`display: none` stops the drawing, not the request). If the image is not critical, accept that; if it is, set `value` to a transparent 1x1 data URI and hide it with `img[src^="data:"] { display: none; }` instead.
- Tags, the full category list, related posts and comments are not bindable yet; keep the built-in `blog-detail` section for those.

Example input: `{ key: "postTitle", value: "", type: "text", source: "post", postField: "title" }`.

## Array mode
Primitive input types (`text`, `textarea`, `image`, `video`, `color`, `datetime`) can be turned into arrays by setting `isArray: true`. Not supported for `object`, `radio`, `dropdown`, `checkbox`, `entity`, `goto`, `group`, `icon`, `threed`, `tracking`, or `action`.
- `value` is a real array of strings: `["value1","value2"]`. A real array implies `isArray: true`. A JSON array string (`'["value1","value2"]'`) is still accepted for older callers.
- In JSX, the variable becomes a JavaScript array you can `.map()` over.
- Example: `{ key: "tags", value: ["React","TypeScript"], type: "text", isArray: true }`

## Group inputs (repeatable structured rows)
A `group` input is the cleanest way to model a **list of structured rows** (feature cards, pricing tiers, team members, FAQs) — preferred over an `object` input when each row shares the same set of typed columns. The editor renders each column with the right control (text box, image picker, color picker, etc.) and lets non-technical users add/remove/reorder rows.

Set on the input itself:
- `type: "group"` and `value: ""` (the value is unused — data lives in `fields` + `items`).
- `fields`: the column schema — an ordered array of `{ key, type, labels?, purpose?, maxLength?, example?, options?, entityType?, datetimeMode? }`. Each `key` is a valid JS identifier, unique within the group, accessed as `row.<key>` in JSX. At least one field is required. Allowed field types: `text`, `textarea`, `image`, `video`, `color`, `radio`, `dropdown`, `checkbox`, `datetime`, `threed`, `entity`, `icon` (NOT `object`, `group`, `goto`, `tracking`, `upload`, or `action`). `radio`/`dropdown` fields need `options`; `entity` fields need `entityType`; `icon` cells take a `"prefix:name"` Iconify reference and render as an element, exactly like a top-level `icon` input.
- `items`: the data rows — an array where each row maps a field `key` to a cell `{ value }`. `value` is always a string (a URL for image/video, the entity id for entity, "true"/"false" for checkbox, raw text otherwise). Missing cells are treated as empty; unknown keys are ignored. Omit `items` to start with zero rows.

In JSX, the group is a JS array — iterate with `.map()` and read columns as `row.<fieldKey>`:
```
<div className="grid gap-6 md:grid-cols-3">
  {features.map((f, i) => (
    <div key={i} className="rounded-lg border p-6">
      <img src={f.icon} alt="" className="h-10 w-10" />
      <h3 className="mt-4 font-heading text-lg font-semibold">{f.title}</h3>
      <p className="mt-2 text-muted-foreground">{f.body}</p>
    </div>
  ))}
</div>
```
Matching input:
```
{
  key: "features", value: "", type: "group",
  labels: { tr: "Özellikler", en: "Features" },
  fields: [
    { key: "icon", type: "image", labels: { tr: "İkon", en: "Icon" } },
    { key: "title", type: "text", labels: { tr: "Başlık", en: "Title" } },
    { key: "body", type: "textarea", labels: { tr: "Açıklama", en: "Description" } }
  ],
  items: [
    { icon: { value: "https://.../speed.svg" }, title: { value: "Fast" }, body: { value: "Sub-second loads." } },
    { icon: { value: "https://.../lock.svg" }, title: { value: "Secure" }, body: { value: "Encrypted end to end." } }
  ]
}
```

## Collection inputs (a live product list)
A `collection` input hands the component a page of the account's REAL products, refreshed on every page view — no ids baked into the code, and no recompile when a price changes. Use it whenever the component is "a list of products"; use `group` instead when the rows are content the editor writes by hand.

Set on the input itself:
- `type: "collection"`, `entityType: "product"` and `value: ""`.
- `collection`: `{ filter, categoryId?, ids?, pageSize, pagination }`.
  - `filter`: `"all"` (every published product), `"category"` (one category **and its subcategories** — pass `categoryId` from `list_categories`), or `"selected"` (exactly `ids`, in that order, from `list_products`).
  - `pageSize`: 1-48. A hard ceiling, not a default: every listed product is fetched and serialised into the page.
  - `pagination`: `"none"` (first page only), `"query"` (page number in the URL, every page crawlable), or `"loadMore"` (appended in the browser; only page 1 is crawlable).

What the variable holds:
```
{ items, page, pageSize, totalItems, totalPages, hasMore, isLoading,
  pageUrl(n), loadMore(), addToCart(item, opts?),
  resolveVariant(item, selection), isValueAvailable(item, selection, axisId, valueId), defaultSelection(item) }
```

Each item in `items`: `id`, `slug`, `url` (locale-correct, ready for `href`), `name`, `description`, `image`, `price`, `salePrice`, `effectivePrice`, `hasDiscount`, `currency`, `category` (`{ name, slug, url }` or null), `variants`, `variantSelectionMode` (`"none" | "single" | "combination"`), `axisCount`, `combinations`, `combinationsTruncated`, `inStock`, `isPopular`, `ctaMode`, `canAddToCart`, `sellable`, `requiresShipping`, `allergens`, `mayContain`, `dietaryTags`.

Rules worth knowing before you write the component:
- **Quote `effectivePrice`, not `price`.** It is what checkout will charge; `price` is the pre-discount figure you show struck through when `hasDiscount`.
- **Branch on `canAddToCart`.** It already accounts for the storefront being on, the product's CTA mode, stock, and whether a purchasable variant can be resolved. When it is false, link to `item.url` instead of rendering a button that can only fail. A multi-axis variant product (colour AND size) is true when it has a default combination, which is the row a bare `addToCart(item)` buys.
- `sellable` is `canAddToCart` minus the row: storefront on, CTA mode `"buy"`, standard checkout, in stock. It is the one gate an explicit variant may bypass — `addToCart(item, { productVariantId })` with a row the buyer chose works when `sellable` is true even though `canAddToCart` is false. The id is a value's `productVariantId` from `item.variants` on a single-axis product, or a combination's `productVariantId` on a multi-axis one. Only use it when you render a variant picker yourself; otherwise `canAddToCart` is the flag.
- `item.combinations` holds the product's ACTIVE combinations (`{ productVariantId, valueIds, price, stock, sku, imageUrl, isDefault, isActive }`), at most 48 per product with the default first; `combinationsTruncated` is true when the product has more. `resolveVariant(item, selection)`, `isValueAvailable(item, selection, axisId, valueId)` and `defaultSelection(item)` work like the product-page `variants` functions (see "Product Template pages"), over that list.
- **The HTML baked on the server has no combinations.** Treat an empty `item.combinations` as not known yet, never as "cannot be bought", and render a picker only when it has rows.
- `addToCart(item)` returns `{ ok: true }` or `{ ok: false, reason }` where reason is `not-purchasable`, `out-of-stock`, `variant-required`, `currency-mismatch` or `unavailable`. It never throws.
- `pageUrl(n)` returns a URL only in `"query"` mode, `""` otherwise. `loadMore()` appends only in `"loadMore"` mode and resolves `{ ok: false }` otherwise. Both are safe to call in any mode.
- The list object is **mutated in place** by `loadMore`. Read `key.items` during render; do NOT copy it into your own state at mount, or appended pages never appear.
- The editor can attach the platform's own pagination control to an element instead, so a component that renders no page links is still fully paginated. Render your own only when you want a specific design.

### Example
```jsx
function ProductList() {
  return (
    <section className="grid gap-6 md:grid-cols-3">
      {products.items.map((p) => (
        <article key={p.id} className="rounded-xl border p-4">
          {p.image && <img src={p.image} alt={p.name} className="aspect-square w-full object-cover" />}
          <h3 className="mt-3 font-medium">{p.name}</h3>
          <p className="text-sm text-muted-foreground">{p.category?.name}</p>
          <p className="mt-1">
            {p.hasDiscount && <s className="mr-2 opacity-60">{p.price}</s>}
            {p.effectivePrice} {p.currency}
          </p>
          {p.canAddToCart ? (
            <button onClick={() => products.addToCart(p)} className="mt-3 w-full rounded-lg bg-primary px-4 py-2 text-primary-foreground">
              Sepete ekle
            </button>
          ) : (
            <a href={p.url} className="mt-3 block text-center underline">İncele</a>
          )}
        </article>
      ))}
    </section>
  )
}

export default ProductList
```

## Site scroll: `useSiteScroll()`
`useSiteScroll` is a global hook, exactly like `useState`: call it at the top of a component, with NO import and no `window.` prefix. It reads the page scroll and scrolls the page programmatically, on every website version, whether or not the site owner turned on smooth scrolling. Do not import Lenis or any other scroll library, do not read `window.__MP_HOST__`, and do not add `scroll-behavior: smooth` or a second inertial scroller of your own: they fight the site's.

```
useSiteScroll(options?: { live?: boolean }): {
  scroll: number         // window.scrollY, in pixels
  progress: number       // 0 to 1 through the page; 0 when the page does not scroll
  velocity: number       // signed pixels per second; always 0 under prefers-reduced-motion
  direction: -1 | 0 | 1  // the way the page last moved (-1 up, 1 down), kept while idle; 0 only until it has moved once
  scrollTo(target: number | string | Element, opts?: { offset?: number; duration?: number; immediate?: boolean; onComplete?: () => void }): void
}

// the object useSiteScroll({ live: false }) returns instead
{
  readonly scroll: number
  readonly progress: number
  readonly velocity: number
  readonly direction: -1 | 0 | 1
  scrollTo(target: number | string | Element, opts?: { offset?: number; duration?: number; immediate?: boolean; onComplete?: () => void }): void
  subscribe(callback: () => void): () => void
}
```

- **Server and first render.** On the server and in the hydration render of the default form, every value is 0, so there is no hydration mismatch. A section that mounts later (client-side navigation, a conditional mount) gets the live values from its first render instead on a website version that carries the scroll bridge; on an older version the hook renders 0 once and has the live values after its first effect, so write the section to tolerate that. `scrollTo` works in the browser only: call it from an event handler or an effect, because on the server it does nothing, and it also does nothing when a CSS selector matches no element. It returns nothing, so code cannot detect that. Render a complete, visible layout in the markup and let the scroll values only move things afterwards. Never hide content in the markup (`opacity: 0`) waiting for scroll: crawlers and the preview screenshot see the server markup.
- **Smooth scroll on or off.** Smooth scrolling is a site-wide setting the owner chooses (`customScroll` on the theme, shown by `get_theme`); a component never turns it on or off. The hook returns the same values either way, and on a website version that predates smooth scroll it reads native scrolling.
- **Snap and entrance animations are site settings too.** The owner can turn on section snap (smooth scroll only): wheel scrolling then settles forward onto the start of a section at least half a viewport tall (or the end of one taller than the viewport). It never applies to touch, keyboard or scrollbar scrolling, a `scrollTo` is not re-snapped, and an element with `data-lenis-prevent` is left alone, so a component does nothing for it. The owner can also set a default entrance animation (`sectionEntrance` in `customScroll`): a custom-code section inherits it as a fade (opacity only, never a transform, so GSAP pins and `position: fixed` inside it keep working; a scroll-reveal default stays scroll-linked, still opacity only) unless its settings carry their own `transition` (`{ type: 'none' }` opts out), and the first visible section never inherits it. Do not add your own fade-in for the same purpose, and never start content at `opacity: 0` to wait for it.
- **Back-to-top.** The platform has a floating back-to-top button (footer config `showScrollToTop` with `scrollToTopDisplayMode` `'fixed'` or `'both'`, set with `update_layout_appearance`), and it also works when the footer is custom code. Use it instead of drawing a second floating button; build your own only when the design needs a specific one.
- **Reduced motion.** When the visitor prefers reduced motion, `velocity` is 0 and `scrollTo` jumps immediately. Skip decorative motion yourself (parallax, scroll-linked transforms) by checking `window.matchMedia('(prefers-reduced-motion: reduce)').matches` inside the effect.
- **Direction is not "is scrolling".** `direction` keeps the last movement while the page is idle, so it stays -1 or 1 after the visitor stops. To tell whether the page is moving right now, test `velocity !== 0`.
- **Default form.** `const { scroll, progress, velocity, direction, scrollTo } = useSiteScroll()` re-renders the component on every scroll frame. That is fine for a small readout (a back-to-top button that shows past 600 px), and wrong for anything heavy.
- **`{ live: false }` form.** `const siteScroll = useSiteScroll({ live: false })` returns one stable object whose `scroll`, `progress`, `velocity` and `direction` are getters, plus `subscribe(callback)`, which returns an unsubscribe function; the callback runs once per scroll frame with no arguments, so read the getters inside it. Use it for scroll-linked styling (a progress bar, parallax): subscribe inside `useEffect`, write to a ref's `style` directly, and return the unsubscribe function as the effect cleanup. It does not re-render per frame, which a sticky navbar needs.
- **Where to read it.** Call the hook at the top level of the component like any hook. Read `scroll` and the other values in render only for light output; do the real work in an effect or a subscriber. The `{ live: false }` getters read the live position even on the first client render, so never use them in render output: reading them there can differ from the server markup and cause a hydration mismatch. Anything that touches `window` or `document` still belongs inside `useEffect`.
- **`scrollTo(target, opts)`.** `target` is a pixel offset from the top of the page, a CSS selector (`'#pricing'`), or an element. The site publishes the height of a sticky or fixed navbar as `--mp-scroll-offset` and applies it as the page's `scroll-padding-top`, so a selector or element target (and a plain `<a href="#id">`) lands below that navbar: scroll with `scrollTo` from this hook, not `window.scrollTo`, and do not subtract the header height yourself. `opts.offset` adds extra pixels on top of it, `opts.immediate` skips the animation, `opts.onComplete` runs when the page arrives, and `opts.duration` (seconds) only applies with smooth scroll on: natively it is ignored, so the same call can animate differently on two sites. For navigation links write a real `<a href="#id">`, which works without JavaScript and with the hook alike; call `scrollTo` for buttons and for scrolling to a computed position. To send a visitor to another section from an editable link, a `goto` input is still the simplest.
- **Inner scroll areas: `data-lenis-prevent`.** With smooth scroll on, the site scrolls the page itself and would take wheel input away from a scrollable element inside your component. Put the attribute `data-lenis-prevent` on every element of the component that scrolls on its own: an overlay mobile menu, a slide-in drawer, a modal panel, any `overflow-y-auto` list (`<div data-lenis-prevent className="max-h-[70vh] overflow-y-auto">`). Without it the list can refuse to scroll while the page behind it moves. Platform dialogs (`role="dialog"`) are already excluded; a menu you build yourself is not.
- **Pages that own their scroll.** A page that still uses a retired scroll type (scroll snap or a slider) scrolls inside its own container, and the hook can report 0 there. Do not build a feature that depends on it for such a page.
- **Editor previews.** The section preview and the canvas size the frame to the content, so the page never scrolls there and scroll effects show their starting state. `render_section_preview` is a static render. Judge the motion on the live preview or the published page.
- **Inputs.** Tunables are inputs, but an input can only be text, a color or a choice, never the hook's return value. There is no number input type: offer a `dropdown` with fixed choices (low, medium, high) and map each to a number in the component, or use a `text` input parsed with `Number()` and clamped.

### Example: parallax image
A framed image that drifts as the page scrolls. It follows every authoring rule: the top-level function has no parameters, the heading and image are inputs, the strength is a dropdown, the colors are theme tokens, the server markup is the finished layout, and the effect cleans up after itself and skips the motion for reduced motion.

sourceJsx:
```
function ParallaxImage() {
  const frameRef = useRef(null)
  const imageRef = useRef(null)
  const siteScroll = useSiteScroll({ live: false })
  const factor = { low: 0.08, medium: 0.15, high: 0.25 }[strength] || 0.15

  useEffect(() => {
    const frame = frameRef.current
    const img = imageRef.current
    if (!frame || !img) return
    if (window.matchMedia('(prefers-reduced-motion: reduce)').matches) return

    const update = () => {
      const rect = frame.getBoundingClientRect()
      const fromCenter = rect.top + rect.height / 2 - window.innerHeight / 2
      img.style.transform = 'translate3d(0,' + (-fromCenter * factor).toFixed(1) + 'px,0) scale(1.3)'
    }
    update()
    return siteScroll.subscribe(update)
  }, [factor, image])

  return (
    <section className="px-6 font-body">
      <h2 className="mb-6 text-center font-heading text-3xl text-foreground">{title}</h2>
      <div ref={frameRef} className="mx-auto h-[60vh] max-w-5xl overflow-hidden rounded-lg bg-muted">
        {image && <img ref={imageRef} src={image} alt={imageAlt} className="h-full w-full object-cover" />}
      </div>
    </section>
  )
}
export default ParallaxImage
```

inputs: [
  { key: "title", value: "Our workshop", type: "text", labels: { en: "Title" }, purpose: "heading above the image, 2 to 5 words", maxLength: 40 },
  { key: "image", value: "", type: "image", labels: { en: "Image" } },
  { key: "imageAlt", value: "The workshop floor in the morning", type: "text", labels: { en: "Image description" }, purpose: "what the image shows, for screen readers", maxLength: 120 },
  { key: "strength", value: "medium", type: "dropdown", labels: { en: "Parallax strength" }, options: [{ label: "Low", value: "low" }, { label: "Medium", value: "medium" }, { label: "High", value: "high" }] }
]

## Host libraries: `useHostLibrary()` and `useGsap()`
Two more global hooks, exactly like `useState`: call them at the top of a component with NO import and no `window.` prefix. They give a component GSAP (with ScrollTrigger) and three.js without bundling or loading either. Do not write `import gsap from 'gsap'`, do not add a `<script>` or a CDN URL, and do not read `window.gsap`, `window.THREE` or `window.__MP_HOST__`: the platform loads each library once for the whole page and hands it to you through the hook.

```
useHostLibrary(name: 'gsap' | 'three'): { lib: HostLibrary | null; ready: boolean }
useGsap(fn: (api: { gsap: Gsap; ScrollTrigger: ScrollTrigger; reducedMotion: boolean }) => void | (() => void), deps?: unknown[]): { current: HTMLElement | null }
```

- **Allowlist.** `useHostLibrary('gsap')` resolves `lib` to `{ gsap, ScrollTrigger }`: GSAP core with the ScrollTrigger plugin already registered. `useHostLibrary('three')` resolves `lib` to `{ THREE, GLTFLoader, OrbitControls, DRACOLoader, loadModel }`, the object `window.mpLoadThree()` resolves; three comes from the template's own loader, so the older `await window.mpLoadThree()` pattern keeps working. Any other name never resolves (`{ lib: null, ready: false }` forever), and a name written as a string literal that is not on the list fails the compile with `Unknown host library "foo". Available: gsap, three.` There is no `@react-three/fiber`: write plain three.js.
- **Server and first render.** On the server, and in the first client render, `useHostLibrary` returns `{ lib: null, ready: false }` and `useGsap` does nothing and returns an empty ref, so there is no hydration mismatch. The library arrives afterwards and `ready` turns true. Check `ready` (and `lib`) inside the effect before you use the library.
- **Render a complete, visible layout.** The server markup must be the finished layout, because crawlers, visitors without JavaScript and the preview screenshot see it. The library only enhances it. Set a starting state such as `opacity: 0` from the effect (`gsap.from` does this for you), never from the markup.
- **`useGsap(fn, deps)`.** Runs `fn` in an effect once GSAP is ready, inside a `gsap.context` scoped to the ref it returns: attach that ref to the component's root element, and selector strings in `fn` (`'[data-reveal]'`) match only inside it. The root element itself is never matched by such a selector, so when the root is the ScrollTrigger `trigger` pass the ref (`trigger: rootRef.current`). If you do not attach the ref, the scope is the section's own mount element. `fn` receives `{ gsap, ScrollTrigger, reducedMotion }`; `reducedMotion` is the `prefers-reduced-motion` match when the effect runs, so return early when it is true. `deps` follows React's rule (its length never changes) and an omitted `deps` means `[]`: run once when the library is ready, not on every render. List every value from the component that `fn` reads (`[distance]`), so the animation rebuilds when an input changes. Before a re-run and on unmount the hook calls the cleanup function `fn` returned, then reverts the context, which kills every tween and ScrollTrigger created inside it. You do not kill them yourself.
- **Smooth scroll and ScrollTrigger.** ScrollTrigger works the same whether the site owner turned smooth scrolling on or off. When it is on, the platform drives ScrollTrigger from the same frame loop as the smooth scroller, so you write ordinary ScrollTrigger code and never touch Lenis, `ScrollTrigger.scrollerProxy` or `ScrollTrigger.refresh()` (call `refresh()` only after you change the layout yourself). Read the page scroll with `useSiteScroll` above; use ScrollTrigger for animation tied to elements.
- **Pin an inner wrapper, never the component's root element.** GSAP moves a pinned element into a `pin-spacer` wrapper, and when the section unmounts React removes its top-level node before the cleanup runs, which then throws. Put a `<div>` inside the root and pin that, or pin a child of it.
- **Cleanup is yours for everything GSAP does not own.** The context revert covers GSAP objects only. Remove every DOM listener, observer, interval and timer you add, and for three.js dispose geometries, materials, textures and the renderer on unmount and remove the canvas.
- **Performance.** One WebGL canvas per page: a second context costs memory and is lost first on mobile. Pause rendering when the canvas is off screen (`IntersectionObserver`) and when the tab is hidden (`visibilitychange`), size with `ResizeObserver`, cap `devicePixelRatio` at 2, and honour `prefers-reduced-motion` with one static frame or no animation.
- **Layouts.** A navbar or footer renders on every page. Avoid `useHostLibrary('three')` there; a short GSAP effect is acceptable.
- **Inputs.** Never put a library object in an input. Tunables are inputs (a text, a `color`, a `dropdown` of fixed choices mapped to numbers in the component); the library calls stay in code. A `color` input can be a theme reference such as `rgb(var(--primary))`; to hand it to three.js, set it as the CSS `color` of an element and read `getComputedStyle(element).color` in the effect.
- **Website version.** The libraries need a website version that ships them. `list_custom_code_capabilities` shows what this site has. A write refused with `TEMPLATE_FEATURE_NOT_AVAILABLE` means the site's website version lacks the library: tell the site owner to upgrade the website version from the dashboard and do not imitate the library with hand-written code. Library code that does run on an older version only shows its server markup.
- **Editor previews.** The previews size the frame to the content, so scroll-triggered effects (reveals, pins) show their starting state there; see "Editor previews" above. Judge them on the live preview or the published page.

### Example: reveal on scroll
A heading and a list that fade and rise as they scroll into view. It follows every authoring rule: the top-level function has no parameters, the texts are inputs, the distance is a dropdown, the colours are theme tokens, the server markup is the finished layout, `gsap.from` sets the hidden state itself, the context revert is the cleanup, and reduced motion keeps the static layout.

sourceJsx:
```
function RevealList() {
  const rise = { low: 16, medium: 32, high: 56 }[distance] || 32

  const rootRef = useGsap(({ gsap, reducedMotion }) => {
    if (reducedMotion) return
    gsap.from('[data-reveal]', {
      opacity: 0,
      y: rise,
      duration: 0.7,
      ease: 'power2.out',
      stagger: 0.12,
      scrollTrigger: { trigger: rootRef.current, start: 'top 85%', once: true },
    })
  }, [rise])

  return (
    <section ref={rootRef} className="px-6 font-body">
      <h2 data-reveal className="mb-8 text-center font-heading text-3xl text-foreground">{title}</h2>
      <ul className="mx-auto grid max-w-4xl gap-6 md:grid-cols-3">
        {items.map((item, i) => (
          <li key={i} data-reveal className="rounded-lg border border-border bg-card p-6 text-card-foreground">
            <h3 className="font-heading text-lg">{item.heading}</h3>
            <p className="mt-2 text-muted-foreground">{item.text}</p>
          </li>
        ))}
      </ul>
    </section>
  )
}
export default RevealList
```

inputs: [
  { key: "title", value: "Why people stay", type: "text", labels: { en: "Title" }, purpose: "heading above the list, 2 to 5 words", maxLength: 40 },
  { key: "distance", value: "medium", type: "dropdown", labels: { en: "Rise distance" }, options: [{ label: "Low", value: "low" }, { label: "Medium", value: "medium" }, { label: "High", value: "high" }] },
  {
    key: "items", value: "", type: "group", labels: { en: "Items" },
    fields: [
      { key: "heading", type: "text", labels: { en: "Heading" }, maxLength: 40 },
      { key: "text", type: "textarea", labels: { en: "Text" }, maxLength: 160 }
    ],
    items: [
      { heading: { value: "Fast" }, text: { value: "Pages open in under a second." } },
      { heading: { value: "Clear" }, text: { value: "One price, no surprises." } },
      { heading: { value: "Yours" }, text: { value: "Everything is editable by you." } }
    ]
  }
]

### Example: minimal three.js canvas
One canvas in a fixed-aspect frame. The server markup is the finished frame with a CSS gradient poster, so the section looks complete before three.js loads; the effect adds a rotating mesh, pauses when off screen or in a hidden tab, renders one static frame under reduced motion, and disposes everything on unmount.

sourceJsx:
```
function SpinningShape() {
  const { lib: three, ready } = useHostLibrary('three')
  const hostRef = useRef(null)
  const speed = { low: 0.004, medium: 0.008, high: 0.016 }[spin] || 0.008

  useEffect(() => {
    const host = hostRef.current
    if (!ready || !three || !host) return
    const { THREE } = three
    const reduced = window.matchMedia('(prefers-reduced-motion: reduce)').matches

    const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true })
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
    host.appendChild(renderer.domElement)

    const scene = new THREE.Scene()
    const camera = new THREE.PerspectiveCamera(45, 1, 0.1, 100)
    camera.position.z = 4
    const geometry = new THREE.IcosahedronGeometry(1.2, 1)
    const material = new THREE.MeshStandardMaterial({ color: new THREE.Color(getComputedStyle(host).color), flatShading: true })
    const mesh = new THREE.Mesh(geometry, material)
    const light = new THREE.DirectionalLight()
    light.position.set(2, 3, 4)
    scene.add(mesh, light, new THREE.AmbientLight())

    const render = () => renderer.render(scene, camera)
    const resize = () => {
      const width = host.clientWidth
      const height = host.clientHeight
      if (!width || !height) return
      renderer.setSize(width, height)
      camera.aspect = width / height
      camera.updateProjectionMatrix()
      render()
    }

    let frame = 0
    let onScreen = false
    const tick = () => {
      mesh.rotation.y += speed
      mesh.rotation.x += speed / 2
      render()
      frame = requestAnimationFrame(tick)
    }
    const start = () => {
      if (!frame && onScreen && !reduced && !document.hidden) frame = requestAnimationFrame(tick)
    }
    const stop = () => {
      cancelAnimationFrame(frame)
      frame = 0
    }

    const observer = new IntersectionObserver(([entry]) => {
      onScreen = entry.isIntersecting
      if (onScreen) start()
      else stop()
    })
    const sizer = new ResizeObserver(resize)
    const onVisibility = () => {
      if (document.hidden) stop()
      else start()
    }
    observer.observe(host)
    sizer.observe(host)
    document.addEventListener('visibilitychange', onVisibility)
    resize()

    return () => {
      stop()
      observer.disconnect()
      sizer.disconnect()
      document.removeEventListener('visibilitychange', onVisibility)
      geometry.dispose()
      material.dispose()
      renderer.dispose()
      renderer.domElement.remove()
    }
  }, [ready, three, speed])

  return (
    <section className="px-6 font-body">
      <h2 className="mb-6 text-center font-heading text-3xl text-foreground">{title}</h2>
      <div className="mx-auto aspect-video max-w-4xl overflow-hidden rounded-lg bg-gradient-to-br from-primary/30 to-secondary/30" style={{ color: shapeColor }}>
        <div ref={hostRef} className="h-full w-full" />
      </div>
    </section>
  )
}
export default SpinningShape
```

inputs: [
  { key: "title", value: "A shape that turns", type: "text", labels: { en: "Title" }, purpose: "heading above the canvas, 2 to 5 words", maxLength: 40 },
  { key: "shapeColor", value: "rgb(var(--primary))", type: "color", labels: { en: "Shape colour" } },
  { key: "spin", value: "medium", type: "dropdown", labels: { en: "Spin speed" }, options: [{ label: "Slow", value: "low" }, { label: "Medium", value: "medium" }, { label: "Fast", value: "high" }] }
]

## Example
sourceJsx:
```
function HeroSection() {
  return (
    <section className="flex items-center justify-center bg-background px-6 font-body">
      <div className="max-w-3xl text-center">
        <h1 className="text-4xl md:text-5xl font-bold text-foreground mb-6 font-heading">{title}</h1>
        <p className="text-lg text-muted-foreground mb-8">{description}</p>
        <a href={ctaLink} className="inline-block bg-primary text-primary-foreground px-6 py-3 rounded-md hover:opacity-90 transition-opacity">{ctaText}</a>
      </div>
    </section>
  );
}
export default HeroSection
```

sourceCss:
```
/* Custom CSS only when Tailwind isn't enough */
```

inputs: [
  { key: "title", value: "Welcome to Our Site", type: "text" },
  { key: "description", value: "We build amazing things.", type: "textarea" },
  { key: "ctaText", value: "Get Started", type: "text" },
  { key: "ctaLink", value: "/contact", type: "text" }
]
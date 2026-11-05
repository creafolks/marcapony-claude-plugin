## Custom-code component authoring rules

Your component is compiled (Babel + Tailwind) and rendered in a sandboxed iframe.
The compiler injects each declared input as a global variable above your function
and server-renders the component before hydration. These rules are platform
invariants — follow them exactly.

Component shape
- The exported top-level function takes NO parameters. Every input you declare is
  injected as a global variable above the function; read it directly by name
  (title, bgColor, items). Declaring parameters such as function C({ title })
  shadows those globals, so every value becomes undefined and the component
  silently renders empty. Always write: function ComponentName() { ... } and end
  the file with export default ComponentName.
- Nested helper components that you define and call yourself MAY take normal React
  props (you pass them manually).
- NO import statements. React, useState, useEffect, useMemo, useRef, useCallback,
  useReducer, useContext, createContext and Fragment are already global.
- Reference inputs in JSX with single braces {varName}; in sourceCss with double
  braces {{varName}}. The {{varName}} form works only for static scalar inputs
  (plain text and color values); product/post bound inputs and group inputs are
  NOT substituted in CSS — the literal {{name}} is left inert, so reference those
  only in JSX with {varName}.

Styling
- Tailwind v4 utility classes are the primary styling method: they are extracted
  from your JSX and compiled against the account theme. Use them. Theme tokens
  work too: bg-primary, text-primary-foreground, text-secondary,
  text-muted-foreground, bg-accent, bg-background, text-foreground, font-heading,
  font-body, rounded-md/lg/xl.
- Use an inline style={{ }} for values that come from inputs at RUNTIME (for
  example a color input). A dynamic Tailwind class built from a runtime value,
  such as bg-[${bgColor}], cannot be precompiled and produces no style. So:
  static design with Tailwind classes, runtime colors with inline style.
- Use sourceCss only for what Tailwind cannot express (keyframes, complex or
  pseudo selectors). Namespace custom classes (for example .mc-card) to avoid
  collisions. Do not use vw/vh for font-size — the iframe viewport differs from
  the editor preview; use rem or Tailwind text sizes.
- Values interpolated into sourceCss via {{varName}} are sanitized: newlines
  become spaces and CSS-significant characters (braces, @, semicolons, angle
  brackets, slashes, backslashes, asterisks) are stripped to prevent CSS
  injection. Use {{varName}} for simple scalar values only; for multiline or
  structural content, render it in JSX with an inline style instead.

SSR safety
- The component is server-rendered first, where window, document and navigator do
  not exist. Access them ONLY inside useEffect (or guard with
  typeof window !== "undefined"). Reading them in the function body, in a useState
  initializer, or during render throws and breaks SSR.

Inputs — keep components editable without code
- Declare an input for every piece of user-facing text, color, image and link; do
  not hardcode these in JSX.
- Images: never hardcode an external URL (a CDN, unsplash, or the source site) in
  JSX or in an image input's default value. Declare image inputs and let the user
  fill them from the media library; for multiple images use type "image" with
  isArray: true.
- Repeatable data (cards, services, features, testimonials) goes in ONE object
  (JSON) input or a group input — never many flat per-item inputs.
- Links and buttons: give every anchor or clickable element its own text input for
  the href (default "#"); for items inside an object or group input, add a "link"
  field.
- Colors: always include at least bgColor, accentColor and textColor inputs so the
  theme is editable without touching code.
- Every input must carry a human-readable label (labels.tr) so non-technical users
  understand what the field controls; number repeated ones ("Görsel 1", "Görsel 2").
- Provide realistic default values in the input definitions so the rendered content
  comes from inputs — the user can then click any element in the builder to select
  and edit it. Do NOT bake a content fallback into JSX (no
  const list = items && items.length ? items : fallback, no title || "Heading"):
  code-baked content is bound to no input, so it cannot be selected or edited, and
  it masks a missing-input bug. Crash-safety comes from declaring every variable you
  reference as an input (it is always injected); if a list is genuinely optional,
  guard only to empty — (items || []).map(...) — never to placeholder data.
- Input values are injected via JSON.stringify, so newlines are safe in JSX — a
  textarea can hold multi-paragraph text (render it with white-space: pre-line).
  (In sourceCss, see the sanitization note above — CSS collapses newlines.)

## Additional rules for custom-code navbars and footers

This component is the site's navbar or footer. It is mounted directly in the page
layout on every page and renders on the server first, exactly like a section, but
these extra invariants apply.

Navigation and SEO
- Render every navigation link as a real anchor with an href in the returned
  markup — <a href={item.href}>{item.label}</a>. Because the component is
  server-rendered, these links must exist in the initial HTML so search engines
  follow them without JavaScript. A link list that you mount only after a click
  (for example {open && <ul>...</ul>}) is invisible to crawlers — instead always
  render the list and hide it with CSS when closed (hidden, max-h-0, translate,
  opacity) so the anchors stay in the DOM.
- Interactivity (dropdowns, the mobile menu toggle) becomes active once the
  compiled script finishes loading, a brief moment after first paint. Render the
  menu in its CLOSED state on the server so the page looks correct before the
  script runs; never render it open by default.

Positioning
- You own sticky, fixed and scroll behavior — write it yourself (position: sticky
  / fixed, a scroll listener in useEffect). The host mounts you in a bare
  container with no wrapper chrome.
- A full-screen mobile menu or a slide-in drawer is fully supported: use
  position: fixed and, if you need it, createPortal(document.body) inside a
  useEffect (never during SSR). Do not rely on any ancestor for positioning.

Localization
- A navbar/footer has ONE source shared across every language, so any text you
  hardcode in JSX is single-language. For content that must localize, declare a
  binding input instead of hardcoding: a navigation input receives the account's
  menu tree already in the current language; contact-info and social-links inputs
  likewise resolve to the account's localized data at render time.

Host widget slots
- Four host-owned widgets can be mounted at any element you author: the
  shopping cart, customer-portal login, language switcher and site search.
  Write data-mp-slot="cart" | "portal-auth" | "locale-switcher" | "search" on
  the JSX element where the widget should appear — an empty container div is
  fine, the host portals the real widget into it. Do not put any of your own
  content inside that element; the host replaces it entirely at render time.
- Each slot renders NOTHING (no error, no placeholder) if its feature is
  disabled for this account (cart needs the storefront plugin, portal-auth
  needs the customer-portal plugin) or, for locale-switcher, if the account
  has 1 or fewer enabled languages. Search always renders. Never build your
  own fallback UI for the gated-off case — the silent no-render is intentional.
- A slot name may appear on more than one element (e.g. a cart icon in the
  header AND a separate one in a mobile menu) — each mounts independently.
  The web editor's guided "attach to element" tool only manages one binding
  per slot at a time (attaching moves the binding rather than duplicating
  it, mirroring the existing action-binding tool); authoring a second mount
  of the same slot requires editing the JSX directly.
- Any value other than the four listed above is ignored — it renders nothing
  and produces no warning. There is no v1 restriction left on these widgets;
  what previously required grid mode can now be authored directly.

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

## Writing a layout through this server

Tools, in the order you normally use them:
1. `get_layout({ kind })` — current mode, compile state, appearance values, the
   drag-and-drop fallback, and the declared inputs. Start here; never assume a
   layout is in grid mode.
2. `get_layout_custom_code({ kind })` — the stored JSX/CSS/inputs, when you are
   editing rather than replacing. A write REPLACES the source, so fetch it first
   and send the whole component back.
3. `write_layout_custom_code({ kind, sourceJsx, sourceCss?, inputs?, name? })`
   — saves a new version, makes it active, switches the layout to custom-code
   mode and starts a compile.
4. `get_layout({ kind })` again to read the compile result. Compilation is a
   background job, so the write returns `pending`, not the final state.

`list_custom_code_capabilities` (read-only) lists the host libraries and hooks this
site's website version can run; read it before using `useHostLibrary` or `useGsap`
in a layout.

Two more, for when something is wrong:
- `set_layout_mode({ kind, mode: 'grid' })` — instant, compile-free way back to
  the drag-and-drop layout. Your source is kept, so you can flip back after
  fixing it.
- `list_layout_versions({ kind })` + `activate_layout_version({ kind, versionId })`
  — restore an earlier version. A custom-code version whose compile did not
  succeed cannot be activated.

## What a write does to the live site

- The new version is **active immediately**. There is no draft state.
- Until the compile finishes, the tenant site renders the account's
  drag-and-drop layout instead. If that layout is empty, the site renders
  **nothing** in that slot for the duration — the write response says so
  (`gridFallbackEmpty`), and it is the one case where you should be ready to
  call `set_layout_mode` back to grid.
- A compile that fails leaves the previous working artifact serving. So a failed
  write is not visible to visitors, but it is also not applied.
- Only the newest ten versions are kept; saving deletes the oldest inactive one.
- An appearance change (`update_layout_appearance`) also creates a version and,
  on a custom-code layout, also triggers a recompile. That is not overhead you
  can avoid: compiled artifacts belong to a single version row.

## Platform data: use bindings, never hardcode

A layout has ONE source shared by every language and every page, so anything you
type into the JSX is single-language and static forever. Declare an input with a
`source` instead:

| source | what arrives | when |
|---|---|---|
| `navigation` | the menu tree: `{ label, href, openInNewTab, children }[]` | baked at compile time |
| `social-links` | `{ type, url, iconName, label }[]` | baked at compile time |
| `logo` | URL of the kind-appropriate logo, `''` when the account has none | baked at compile time |
| `business-name` | the account name | baked at compile time |
| `contact-info` | one contact field, chosen with `bindingField` | substituted at render time |

Notes that decide whether your JSX works:
- The navigation items have **no `id`** — key React lists off `href` or the index.
- `href` is never null: an unresolvable target bakes to `'#'`, the home page to `'/'`.
- Baked labels and hrefs are **default-locale only** in v1, on every language's pages.
- `navigation` and `social-links` inputs are seeded automatically on every save,
  so they exist whether or not you declare them. Declare them only to rename the
  key. `logo` and `business-name` are not seeded: declare them to use them.
- A bound input's `value` is a fallback, used only when the binding resolves empty.

## Two failure modes worth naming

- **Empty server HTML.** If your first render returns nothing (everything gated
  behind a `mounted` flag, or a `window` read during render), the compile
  succeeds but ships no markup: no crawlable links, and a blank bar until the
  script loads. This server reports it as a FAILED compile. Render the real
  markup on the server and hide what should be closed with CSS.
- **Classes that never get generated.** Tailwind candidates are collected from
  `sourceJsx` string literals only. `${...}` interpolations are dropped, and a
  class that appears only in `sourceCss` (`@apply`) or only in an input's default
  value is never compiled. There is no safelist. Write static classes in JSX and
  use inline `style` for values that come from inputs at runtime.

Source over 256 KB is rejected before the compile runs.
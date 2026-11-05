Custom code capabilities for this site (website version v9).

The authoring rules for every hook below are in get_custom_code_guide. An unavailable capability must not be used in a write, and a write that does is refused.

## Capabilities

- useSiteScroll (id: use-site-scroll, hook): available
  Reads the page scroll and scrolls the page, whether or not the site has smooth scrolling on.
  Hooks: useSiteScroll
  Works in: section, layout
  Signature: useSiteScroll(options?: { live?: boolean }): { scroll: number; progress: number; velocity: number; direction: -1 | 0 | 1; scrollTo(target: number | string | Element, opts?: { offset?: number; duration?: number; immediate?: boolean; onComplete?: () => void }): void; subscribe(callback: () => void): () => void }

- GSAP with ScrollTrigger (id: host-library-gsap, library): available
  GSAP and the ScrollTrigger plugin, loaded once per page by the platform and used through useGsap.
  Hooks: useGsap, useHostLibrary('gsap')
  Works in: section, layout
  Signature: useGsap(fn: (api: { gsap: Gsap; ScrollTrigger: ScrollTrigger; reducedMotion: boolean }) => void | (() => void), deps?: unknown[]): { current: HTMLElement | null }
    useHostLibrary(name: 'gsap' | 'three'): { lib: HostLibrary | null; ready: boolean }

- three.js (id: host-library-three, library): available
  three.js with GLTFLoader, OrbitControls, DRACOLoader and loadModel, loaded by the platform.
  Hooks: useHostLibrary('three')
  Works in: section, layout
  Signature: useHostLibrary(name: 'gsap' | 'three'): { lib: HostLibrary | null; ready: boolean }

## Snippets

Each available snippet payload is ready for create_custom_code_section: add pageId and locale, and optionally sectionTitle and settings. Snippets are section-only: write_layout_custom_code replaces the whole navbar or footer, so do not apply a section snippet to a layout.

### reveal-on-scroll: Reveal on scroll (available)
A heading and cards that fade and rise as they scroll into view. The page is complete without JavaScript; GSAP adds the motion.
Works in: section. Requires: host-library-gsap.
Payload:
```json
{
  "sourceJsx": "function RevealList() {\n  const rise = { low: 16, medium: 32, high: 56 }[distance] || 32\n\n  const rootRef = useGsap(({ gsap, reducedMotion }) => {\n    if (reducedMotion) return\n    gsap.from('[data-reveal]', {\n      opacity: 0,\n      y: rise,\n      duration: 0.7,\n      ease: 'power2.out',\n      stagger: 0.12,\n      scrollTrigger: { trigger: rootRef.current, start: 'top 85%', once: true },\n    })\n  }, [rise])\n\n  return (\n    <section ref={rootRef} className=\"px-6 font-body\">\n      <h2 data-reveal className=\"mb-8 text-center font-heading text-3xl text-foreground\">{title}</h2>\n      <ul className=\"mx-auto grid max-w-4xl gap-6 md:grid-cols-3\">\n        {items.map((item, i) => (\n          <li key={i} data-reveal className=\"rounded-lg border border-border bg-card p-6 text-card-foreground\">\n            <h3 className=\"font-heading text-lg\">{item.heading}</h3>\n            <p className=\"mt-2 text-muted-foreground\">{item.text}</p>\n          </li>\n        ))}\n      </ul>\n    </section>\n  )\n}\nexport default RevealList\n",
  "sourceCss": "",
  "inputs": [
    { "key": "title", "value": "Why people stay", "type": "text", "purpose": "Heading above the list, 2 to 5 words", "maxLength": 40 },
    { "key": "distance", "value": "medium", "type": "dropdown", "options": [{ "label": "Low", "value": "low" }, { "label": "Medium", "value": "medium" }, { "label": "High", "value": "high" }] },
    { "key": "items", "value": "", "type": "group", "purpose": "The cards that appear one after the other, usually three",
      "fields": [
        { "key": "heading", "type": "text", "purpose": "Card heading, 1 to 3 words", "maxLength": 40 },
        { "key": "text", "type": "textarea", "purpose": "One short sentence under the heading", "maxLength": 160 }
      ],
      "items": [
        { "heading": { "value": "Fast" }, "text": { "value": "Pages open in under a second." } },
        { "heading": { "value": "Clear" }, "text": { "value": "One price, no surprises." } },
        { "heading": { "value": "Yours" }, "text": { "value": "Everything is editable by you." } }
      ]
    }
  ]
}
```

### three-canvas: Minimal three.js canvas (available)
One rotating shape in a fixed-aspect frame. Works in: section. Requires: host-library-three. (Payload omitted here; ask for it by building with useHostLibrary('three') as the guide shows.)

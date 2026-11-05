---
name: custom-code
description: Builds or changes custom code sections on a Marcapony website through the Marcapony connector. A custom code section is a React and Tailwind component that Marcapony compiles and renders on the live page; this skill also covers a custom navbar or footer, scroll animations with GSAP and ScrollTrigger, three.js scenes, smooth scroll, and components from the account or platform section library. Use when the user asks for a new section the site builder does not offer, wants to restyle, fix or rework an existing custom code section, wants an animated or 3D section, or wants to reuse a library component.
---

# Custom code sections on Marcapony

A custom code section is a React component (JSX plus optional CSS, Tailwind available) that Marcapony compiles on its servers, renders on the server and hydrates on the live page. You write source; you never build, bundle or deploy anything yourself. Every save is kept as a version. What happens on the live site depends on what you write:

- **Page sections** are drafts. The live site keeps showing the last published version until the owner publishes from the Marcapony dashboard.
- **A custom navbar or footer goes live as soon as it is written**, on every page of the site. Treat it as a live change (step 6).
- The connector can also put a single page live (`update_page` with a published status), publish or schedule a blog post, and schedule a page. Never do any of these unless the user asks for it in so many words. Building or changing a section is not a request to publish.

The tools in this skill are the Marcapony connector's tools. Their descriptions and the authoring guide are the reference; this skill is the order of work. Do not rely on memory for input types, actions, bindings, hooks or theme tokens: read the guide.

## 1. Learn the account

- `list_memories` (or `get_business_info`, which includes them): the user's standing preferences for tone, language and formatting. Apply them.
- `get_theme`: colours, fonts, radii, spacing and the site-wide scroll setting (smooth scroll, section snap, the default entrance animation), and whether the site's website version can render each. If the theme does not fit the brand the user describes, settle the theme first, then build. The `get_theme` description says how to change it.
- Build with the theme tokens the guide lists, never hard-coded hex colours, so the section follows the site's design and dark mode. A save that hard-codes colours comes back with a `hardcoded_colors` warning.

## 2. Find the target

- The user named a page or a piece of content: `find_content` first. Fall back to `list_pages` only when you need the whole inventory.
- `get_page_details` and `list_sections` show what is already on the page and in which order.
- An existing custom code section: `get_section_details` returns its source, its inputs and its `baseHash`. You need the `baseHash` to edit. If it says the section is a language copy that takes its code from a source section, only some input values can change there: make code and structure changes on the source section, and they reach the copy. A language copy with its own code (detached) is edited directly.
- A navbar or footer: `get_layout` and `get_layout_custom_code` instead.

If the request does not say which page a new section goes on or what it should contain, ask once for the missing part before writing anything.

## 3. Read the guide and the capabilities once per session

- `get_custom_code_guide` before writing, with kind `section`, or kind `layout` for a navbar or footer. It covers the rendering model, the wrapper spacing contract, every input type, actions and form submission, product, post and collection bindings, the theme tokens, the platform hooks and worked examples. Clients that read MCP resources can use `marcapony://custom-code/authoring-guide` instead.
- `list_custom_code_capabilities` when the section needs motion, scroll effects or 3D. It lists the platform hooks (`useSiteScroll`, `useHostLibrary`, `useGsap`), whether the connected site's website version can render each, and ready-made snippets you can start from.

## 4. Animation, scroll and 3D

GSAP with ScrollTrigger and three.js are provided by the platform through hooks, without imports. The guide has the signatures and the full rules. The ones that break published sites most often:

- Use `useGsap(fn, deps)` for GSAP and `useHostLibrary('three')` for three.js. Never `import` them and never read `window.gsap`, `window.ScrollTrigger` or `window.THREE`: those globals exist in the editor preview but not on the published site, so the code works in the preview and fails live.
- The server markup must be a complete, visible layout, and the animation only enhances it. Set a starting state such as `opacity: 0` from the animation (`gsap.from`), never in the markup.
- Pin an inner wrapper that is always rendered (`pin: ref.current`), never the component's root element and never a conditionally rendered node.
- Honour `reducedMotion`, clean up every listener, observer and WebGL resource, and keep one WebGL canvas per page. A navbar or footer renders on every page: a short GSAP effect is fine there, three.js is not.
- Never put a three.js scene in a navbar or footer, even when the user asks for one and even with mitigations. Explain that it would load WebGL on every page, and offer a 3D section on the page where it matters or a light GSAP effect in the footer instead.
- A save refused with `TEMPLATE_FEATURE_NOT_AVAILABLE` means the site's website version cannot render that library. Tell the owner to upgrade the website version in the dashboard. Do not imitate the library with hand-written code.
- For smooth anchor scrolling and scroll progress use `useSiteScroll`, which follows the site's smooth scroll setting.

## 5. Reuse before you write

For a common section (pricing table, testimonial carousel, FAQ, hero):

- `list_platform_sections` and `get_platform_section` show Marcapony's curated sections; `install_platform_section` places one on a page.
- `list_library_components` and `get_library_component` show the account library; `install_library_component` places an approved component on a page as a linked section that keeps receiving library updates.
- When the user wants to reuse a component of their own across pages, `submit_library_component` proposes it for the account library, where a reviewer approves it, and `update_library_component` revises a submission.

## 6. Write

- New section: `create_custom_code_section`. Expose the text, images and links the owner will want to change as inputs, so they can edit them in the dashboard without touching code. Give inputs a `purpose`, and a `maxLength` and `example` for text, so the dashboard and the in-app assistant fill them well.
- Targeted change to an existing section: `edit_custom_code` with anchored find and replace hunks and the `baseHash` you read. The result carries the new `baseHash`, so a follow-up edit chains on it without another read. If an edit is refused with a conflict, the section changed since you read it (for example the owner edited it): read it again and re-anchor.
- Rewrite of more than about a third of a component, a post-bound input, or a change to the section's wrapper settings: `update_section` with the complete new source.
- Navbar or footer: `write_layout_custom_code`. This is live at once on every page, and it replaces the whole source and every input you leave out. Before calling it, tell the user it goes live immediately and get their explicit go-ahead. Send the complete component with all its inputs. The compile runs in the background: poll `get_layout` until it reports the result, and if the compile fails, use `set_layout_mode` to put the site back on its drag-and-drop layout and tell the user.

Media for inputs: `search_images` searches the account's own images by description (pass `scope: "library"` for the shared public library), `list_recent_uploads` shows what the user just uploaded, and `upload_asset` adds a file from a public URL to the media library.

## 7. Check your work

- Read the compile result and any warnings that come back with the save, and fix blocking findings before you move on. `empty_ssr_html` means the section renders nothing on the server, which hides it from search engines and the first paint.
- `render_section_preview` returns a PNG of the section as the site renders it, with the theme and wrapper spacing applied. By default it mounts the section in a browser and waits briefly for effects and entry animations; `waitMs` 0 gives the static server render. Check `viewport: "mobile"` and `appearance: "dark"` too. Hover states, interaction and scroll-triggered animations below the fold do not appear, so do not treat their absence as a bug.

## 8. Finish

Summarise what changed and where (page, section), mention anything the owner has to do (upgrade the website version, fill an input, approve a library submission), and say plainly which changes are drafts the owner publishes from the Marcapony dashboard and which are already live (a navbar or footer, or anything the user asked you to publish).

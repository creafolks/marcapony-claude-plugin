---
expect:
  kind: [navbar, footer]
---

Custom code for {{input.kind}}.
The navbar is in custom-code mode and live on every page:

```jsx
function Navbar() {
  const [open, setOpen] = useState(false)
  return (
    <header className="sticky top-0 z-40 bg-background/90 font-body backdrop-blur">
      <div className="mx-auto flex max-w-6xl items-center justify-between px-6 py-5">
        <a href="/" className="font-heading text-xl text-foreground">{brand}</a>
        <nav className="hidden gap-6 md:flex">
          {links.map((link, i) => (
            <a key={i} href={link.href} className="text-foreground hover:text-primary">{link.label}</a>
          ))}
        </nav>
        <button type="button" className="md:hidden" onClick={() => setOpen(!open)} aria-label={menuLabel}>☰</button>
      </div>
      {open && (
        <nav className="flex flex-col gap-3 px-6 pb-4 md:hidden">
          {links.map((link, i) => <a key={i} href={link.href}>{link.label}</a>)}
        </nav>
      )}
    </header>
  )
}
export default Navbar
```

inputs:
- brand (text): "Kıyı Kahve"
- menuLabel (text): "Menüyü aç"
- links (object): [{"label":"Menü","href":"/menu"},{"label":"Hakkımızda","href":"/hakkimizda"},{"label":"Fiyatlar","href":"/fiyatlar"}]

The footer is not in custom-code mode; it has no custom code yet.

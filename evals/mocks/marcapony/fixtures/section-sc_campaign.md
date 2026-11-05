Section sc_campaign (custom-code) on page pg_home, locale tr. Own code (not a language copy).
baseHash: h_7f3a91c2

sourceJsx:
```jsx
function Campaign() {
  return (
    <section className="px-6 py-16 font-body">
      <div className="mx-auto max-w-4xl rounded-lg bg-secondary p-10 text-center">
        <h2 className="font-heading text-3xl text-secondary-foreground">Yaz menüsü geldi</h2>
        <p className="mt-3 text-muted-foreground">{subtitle}</p>
        <a href={ctaLink} className="mt-6 inline-block rounded-md bg-primary px-4 py-2 text-sm text-primary-foreground">
          {ctaLabel}
        </a>
      </div>
    </section>
  )
}
export default Campaign
```

sourceCss: (empty)

inputs:
- subtitle (text, purpose "One sentence under the heading", maxLength 120): "Soğuk demleme ve limonatalar bütün yaz bizde."
- ctaLabel (text, maxLength 24): "Menüyü gör"
- ctaLink (link): "/menu"

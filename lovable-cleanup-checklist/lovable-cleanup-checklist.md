# How to clean up a Lovable site before you show it to anyone

## Context for the AI tool

You are cleaning up a site that was generated with Lovable before it goes live on a real domain.
Lovable optimises for "it renders in the preview". Everything a browser tab, a search engine, a
WhatsApp preview or a screen reader sees was left at a default or guessed. Assume nothing in the
head of the document, in `public/` or in the colour tokens was chosen on purpose.

- Do not change anything until you have reported your findings and I have confirmed.
- Go through the checks in order. For each one, show the command you ran and what it returned.
- Report findings as a table: where (file and line or URL), problem, proposed fix. A check
  passes only when its "How to verify" step passes. "Looks fine" is not a result.
- Ask me for anything only I can decide: the brand colours, the logo file, page titles and
  descriptions, and the text on the share image. Do not invent them.
- Do not remove the Lovable build plugins (`lovable-tagger`, `@lovable.dev/vite-tanstack-config`)
  if I still edit the project in Lovable. They only run in development.

**Where the head tags live.** Newer Lovable projects use TanStack Start: the default tags are in
`head()` in `src/routes/__root.tsx`, and each route can add its own `head()`. Older projects are a
Vite single-page app: the tags are in `index.html`, sometimes overridden per page with
`react-helmet-async`. Find out which one this is before you start:

```bash
ls index.html src/routes/__root.tsx 2>/dev/null
grep -n "head:" src/routes/*.tsx src/routes/**/*.tsx 2>/dev/null | head -40
```

## 1. Lovable's leftovers

**What to look for:** Lovable writes its own marketing into your site. On a real project we found
all of these in `__root.tsx`, long after it went live:

- `author` set to `Lovable`
- `twitter:site` set to `@Lovable` (or `@lovable_dev`)
- `og:image` and `twitter:image` pointing at a screenshot of the `*.lovable.app` preview, stored on
  Lovable's own storage, so your share image is a picture of an unfinished preview, and Lovable
  can change or delete it
- a description that is either `Lovable Generated Project` or a sentence written by the AI about
  the *software* ("a CMS for managing content") instead of for your *customer*
- `twitter:card` set to `summary` (small square thumbnail) instead of `summary_large_image`
- the default favicon in `public/favicon.ico`, plus `public/placeholder.svg`
- the "Edit with Lovable" badge on the live site
- a README that is Lovable's template, not yours

```bash
grep -rniE "Lovable Generated|gpt-engineer|lovable\.app|placeholder\.svg|(author|twitter:site|og:image|twitter:image)[^}>]*lovable" \
  index.html src/routes public README.md 2>/dev/null
```

Lovable's AI and email connectors also put `lovable.dev` URLs in server code. Those are not
leftovers: leave them.

Then check what the world actually receives. This is what crawlers and chat apps read; your
editor is not. (`-a` matters: TanStack Start pages contain a few NUL bytes, and without it grep
calls the page binary and prints nothing.)

```bash
curl -sL https://<your-domain>/ | grep -aiE "<title|name=\"description|name=\"author|og:|twitter:|rel=\"(icon|canonical)"
```

**How to fix:** remove `author` (it does nothing for you), remove `twitter:site` unless you have
your own X account, and replace every value in the list above using sections 2 to 5. Hide the
badge in Lovable's project settings (paid plans). Delete `placeholder.svg` once nothing uses it.
Rewrite the README to say what the project is and how to run it.

**How to verify:** both commands above return no Lovable references and no `lovable.app` URLs
(build plugins excepted).

## 2. Page titles

**What to look for:** the same title on every page, a title that is the project name (`My App`,
`WDDW CMS`), or titles longer than about 60 characters that Google cuts off.

```bash
for p in / /about /pricing /contact; do
  printf "%-12s " "$p"; curl -sL "https://<your-domain>$p" | grep -ao "<title>[^<]*</title>"
done
```

**How to fix:** every route gets its own `head()` (or `<Helmet>`) with a title in the pattern
`What this page is | Brand`, written for the person searching, not for you. Put the important
words first. 30 to 60 characters. The home page may lead with the brand. The 404 page says
`Page not found | Brand`.

**How to verify:** the loop above prints a different, human title for each page, and none are
longer than 60 characters.

## 3. Meta tags

**What to look for:** missing or copied descriptions, no canonical URL, the wrong `lang`, no
sitemap, and the site still reachable on `*.lovable.app` as a duplicate of your domain.

**How to fix:** per page:

- `description`: 120 to 160 characters, unique per page, says what the visitor gets and why it
  is you. No "Welcome to our website".
- `og:title`, `og:description`, `og:url`, `og:type`, `og:image` (section 5), and
  `twitter:card` = `summary_large_image`.
- `<link rel="canonical" href="https://<your-domain>/<path>">` with the full URL on your own
  domain. This tells Google the `lovable.app` copy is not the real one.
- `<html lang="...">` set to the language the page is actually in (Lovable sets `en` for
  everything).
- `theme-color` in your brand colour.
- Skip `keywords`: search engines ignore it.

Add `public/sitemap.xml` listing every public URL and point to it from `public/robots.txt`
(`Sitemap: https://<your-domain>/sitemap.xml`). Keep login, admin and dashboard routes out of
the sitemap and give them `<meta name="robots" content="noindex">`.

**Single-page app?** Tags set by `react-helmet-async` only exist after JavaScript runs. Google
renders JavaScript eventually; WhatsApp, LinkedIn, Facebook, Slack and iMessage never do. They
all see the defaults in `index.html`. If the share preview of a sub-page matters, you need
prerendering (for example `vite-plugin-prerender` or a static export) or a framework that renders
on the server. TanStack Start projects render `head()` on the server and do not have this problem.

**Check the 404 status too.** Lovable sites tend to answer every unknown URL with `200 OK` and a
"Not found" page. The single-page setup cannot do anything else, and the TanStack Start project we
tested did the same. Google calls that a soft 404 and may index your error page:

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://<your-domain>/this-page-does-not-exist
```

**How to verify:** the `curl | grep` from section 1 shows a real description, canonical and
`og:` set on every key page; the sitemap loads; the 404 test returns `404` (or, on a host that
cannot do that, the not-found page carries `noindex`).

## 4. Favicon

**What to look for:** Lovable's favicon, a single 256×256 `favicon.ico` stretched to every size,
no Apple touch icon, and nothing Google can use in search results.

```bash
ls -la public/ | grep -iE "fav|icon|apple|manifest"
file public/favicon.ico
```

**How to fix:** start from your logo's symbol, not the full logo with the wordmark (text is
unreadable at 16 pixels). Ask me for the file. Create:

- `favicon.ico` with 16, 32 and 48 pixel sizes inside
- `icon.svg` for modern browsers (it can switch colour for dark mode with a `prefers-color-scheme`
  media query inside the SVG)
- `apple-touch-icon.png` at 180×180, with a solid background (iOS fills transparency with black)
- `icon-192.png` and `icon-512.png`, referenced from `site.webmanifest`

```html
<link rel="icon" href="/favicon.ico" sizes="48x48">
<link rel="icon" href="/icon.svg" type="image/svg+xml">
<link rel="apple-touch-icon" href="/apple-touch-icon.png">
<link rel="manifest" href="/site.webmanifest">
```

**How to verify:** open the site in a private window: the tab shows your icon at 16 pixels and you
can still tell what it is. `curl -sI https://<your-domain>/apple-touch-icon.png` returns `200`.

## 5. A real share image

**What to look for:** an `og:image` that is a screenshot of the preview, a relative URL (chat apps
cannot resolve `/og.png`), an image of the wrong size, or no image at all.

**How to fix:** make one image per site at 1200×630 pixels, PNG or JPG, under 300 KB. Logo, one
short line saying what you do, brand colours, large enough to read at thumbnail size on a phone.
No screenshots of the UI with tiny text. Put it in `public/og.png` and reference it with the full
URL:

```html
<meta property="og:image" content="https://<your-domain>/og.png">
<meta property="og:image:width" content="1200">
<meta property="og:image:height" content="630">
<meta property="og:image:alt" content="<what the image says, in words>">
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:image" content="https://<your-domain>/og.png">
```

Pages that get shared on their own (articles, products, case studies) deserve their own image.

**How to verify:** paste your URL into the LinkedIn Post Inspector and the Facebook Sharing
Debugger, and send it to yourself on WhatsApp. All three show your image and your title. They
cache old previews: use the debugger's "Scrape again" button after a change.

## 6. Alt text everywhere

**What to look for:** images without `alt`, `alt` that repeats the file name or says "image",
decorative images that screen readers announce, and icon-only buttons with no name.

```bash
grep -rnE "<img\b" src --include='*.tsx' --include='*.jsx' | grep -v "alt=" 
grep -rnE "alt=\"(image|img|photo|picture|logo|icon|banner)?\"" src --include='*.tsx' --include='*.jsx' -i
grep -rnE "<(Button|button)[^>]*size=\"icon\"" src --include='*.tsx' | grep -v "aria-label"
```

**How to fix:** describe what the image *shows and why it is there*, in the page's language:
`alt="Two plates of pad thai on the restaurant terrace"`, not `alt="food"`. Logos: the company
name. Purely decorative images: `alt=""` (empty, not missing) so screen readers skip them.
Icon-only buttons (Lucide icons everywhere in Lovable projects): add `aria-label="Close menu"` or
visually hidden text.

These searches only see one line at a time. JSX often puts `alt` or `aria-label` on the next
line, so open each hit before you call it a problem.

**How to verify:** every hit from the three searches is fixed or confirmed fine. In the browser, every image in the
Accessibility pane of DevTools has a sensible name.

## 7. Image optimisation

**What to look for:** Lovable drops images in at whatever size you uploaded, usually multi-megabyte
PNGs, and hotlinks stock photos at full resolution. No `width`/`height` (the page jumps while it
loads), no lazy loading, no modern formats.

```bash
find public src -type f \( -iname '*.png' -o -iname '*.jpg' -o -iname '*.jpeg' -o -iname '*.webp' \) \
  -size +200k -exec ls -lh {} \; | awk '{print $5, $9}' | sort -h
grep -rnoE "https://images\.unsplash\.com[^\"' )]*" src | head
```

**How to fix:** resize each image to the largest size it is ever shown at (twice that for retina,
and no more), convert photos to WebP or AVIF, and keep PNG only for graphics with sharp edges
or transparency. Download hotlinked stock photos and serve them yourself at the right size. On
every `<img>`: `width` and `height` attributes, `loading="lazy"` and `decoding="async"` for
everything below the fold, and `fetchpriority="high"` on the one big image at the top. Use `srcset`
when an image is shown at very different sizes on phone and desktop.

**How to verify:** run the `find` again: nothing over 200 KB unless you can say why. In DevTools →
Network → Img, the page's images add up to under 1 MB on the home page.

## 8. Loading time

**What to look for:** measure on a phone profile, because that is what Google ranks on and most
of your visitors use.

```bash
npx lighthouse https://<your-domain>/ --only-categories=performance,seo,accessibility,best-practices \
  --output=html --output-path=./lighthouse.html
```

Targets: Largest Contentful Paint under 2.5 s, Cumulative Layout Shift under 0.1, Total Blocking
Time under 200 ms. The usual Lovable causes: one JavaScript bundle with every page and every
shadcn component in it, Google Fonts loaded in five weights you never use, animation libraries on
every section, and data fetched in the browser after the page has loaded.

**How to fix:** images first (section 7), then load only the font weights you use with
`display=swap`, split routes so a visitor downloads only the page they open, and render the content
above the fold without waiting for a Supabase call. Our fix "Why is my AI-built app slow" goes
deeper on bundles, queries and caching.

**How to verify:** Lighthouse mobile performance is 90 or higher on the home page and on your most
important landing page, measured three times (it varies; use the middle score).

## 9. Brand colours

**What to look for:** Lovable's colours are shadcn's defaults plus whatever the AI felt like:
slate greys, a purple-to-pink gradient on the hero, and blue buttons that have nothing to do with
your logo. A common pattern: the brand colour was applied to the light theme and the dark theme
was left on the defaults.

```bash
grep -nE "^\s*--(background|foreground|primary|secondary|accent|muted|ring|border)[-a-z]*:" src/index.css src/styles.css 2>/dev/null
grep -rhoE "\b(bg|text|border|ring|from|via|to|fill|stroke)-(slate|gray|zinc|neutral|stone|red|orange|amber|yellow|lime|green|emerald|teal|cyan|sky|blue|indigo|violet|purple|fuchsia|pink|rose)-[0-9]{2,3}\b" src | sort | uniq -c | sort -rn | head -30
grep -rnoE "#[0-9a-fA-F]{6}\b" src --include='*.tsx' --include='*.jsx' | head -30
```

**How to fix:** ask me for the logo's exact colours (from the logo file, not from a screenshot).
Put them in the CSS variables, `:root` *and* `.dark`, and map every role: `primary`,
`primary-foreground`, `accent`, `ring`, `background`, `foreground`, `muted-foreground`, `border`.
Then replace every hard-coded palette class and hex value in components with the token
(`bg-primary`, `text-muted-foreground`). One accent colour is a brand; four gradients are a
template. If you do not want a dark mode, remove the toggle rather than shipping an unbranded one.

**How to verify:** the second and third searches return nothing outside the colour definitions.
Put the logo next to the hero, a button and a link on screen: they are visibly the same family.
Every text/background pair passes section 10's contrast check.

## 10. Accessibility

**What to look for:** shadcn's Radix components are mostly fine. The problems are in everything
the AI built around them: `div`s with `onClick`, forms with placeholder text instead of labels,
light grey text on white, removed focus outlines, several `h1`s, animations that ignore reduced
motion. For sites in the EU this is not optional: the European Accessibility Act has applied to
web shops and online services since 28 June 2025 (micro-enterprises are exempt).

```bash
npx @axe-core/cli https://<your-domain>/ https://<your-domain>/<key-page>
grep -rnE "<div[^>]*onClick" src --include='*.tsx' | head
grep -rnE "outline-none|focus:outline-none" src --include='*.tsx' | grep -v "focus-visible" | head
```

**How to fix:** anything clickable is a `<button>` or a `<Link>`, never a `div`. Every input has a
visible `<label>` (or `aria-label` when there is truly no room). Text contrast at least 4.5:1, large
text and icons 3:1. Focus outlines stay visible (`focus-visible:ring-2 ring-ring`). One `h1` per
page, headings in order. Link text that makes sense out of context, no "click here". Wrap
animations in `motion-safe:` or check `prefers-reduced-motion`.

**How to verify:** axe reports zero violations. Then do it by hand, because axe finds only part of
the problems: unplug the mouse and use the whole site with Tab, Shift+Tab, Enter and Escape (you
can always see where you are, menus and dialogs open and close, focus does not get lost), and
zoom to 200%: nothing overlaps or disappears.

## 11. Final check

Run this once everything above is done, and paste the results in your report:

```bash
curl -sL https://<your-domain>/ | grep -aciE "lovable"   # expect 0
curl -s -o /dev/null -w "%{http_code}\n" https://<your-domain>/nope   # expect 404
npx lighthouse https://<your-domain>/ --quiet --output=json \
  | node -e 'const r=JSON.parse(require("fs").readFileSync(0));for(const[k,v]of Object.entries(r.categories))console.log(k,Math.round(v.score*100))'
```

Done means: no Lovable references, every page has its own title, description and canonical, the
share preview shows your image on LinkedIn and WhatsApp, the favicon is yours at 16 pixels, no image
over 200 KB without a reason, Lighthouse mobile 90+ on performance and 100 on accessibility and SEO,
colours come only from tokens that match the logo, and you can use the site with a keyboard alone.

---

Still stuck? WDDW fixes it. Book a free 15-minute call:
https://www.wedodevwork.com/en/contact-us?utm_source=fixes&utm_medium=md&utm_campaign=lovable-cleanup-checklist

Did this help? An honest review helps us a lot: https://g.page/r/CXmlHMBM4uxyEBM/review
More free fixes: https://www.wedodevwork.com/en/fixes · Star the repo: https://github.com/WEDODEVWORK/fixes

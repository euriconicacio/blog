# blog

Jekyll source for [euriconicacio.github.io/blog](https://euriconicacio.github.io/blog/).

```
_layouts/          default, home, post
_includes/         head, footer, theme toggle
assets/css/        one stylesheet, shared tokens with the landing page
all_collections/_posts/
```

Dark by default, follows `prefers-color-scheme`, toggle persisted in
`localStorage`. No CDN, no web fonts. Page counts go to GoatCounter
(cookieless, `h3llh0und.goatcounter.com`) through a self-hosted copy of
its `count.js` in `assets/js/goatcounter.js`; it loads only in production
builds, so local previews are not counted.

Local build:

```bash
bundle install
bundle exec jekyll serve   # http://localhost:4000/blog/
```

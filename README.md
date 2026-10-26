# blog

Jekyll source for [euriconicacio.github.io/blog](https://euriconicacio.github.io/blog/).

```
_layouts/          default, home, post
_includes/         head, footer, theme toggle
assets/css/        one stylesheet, shared tokens with the landing page
all_collections/_posts/
```

Dark by default, follows `prefers-color-scheme`, toggle persisted in
`localStorage`. No web fonts. Page counts go to GoatCounter
(cookieless, `h3llh0und.goatcounter.com`); the script loads only in
production builds, so local previews are not counted.

Local build:

```bash
bundle install
bundle exec jekyll serve   # http://localhost:4000/blog/
```

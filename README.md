# Genevieve Zingel — Portfolio (Jekyll)

A dark, minimal portfolio built as a **Jekyll** site, ready for GitHub Pages.

## Structure

| Path | Purpose |
|------|---------|
| `index.md` | The page content (front matter + body), rendered through the layout |
| `_layouts/default.html` | The HTML shell — `<head>`, all CSS, `<body>`, and `{{ content }}` |
| `_config.yml` | Jekyll configuration |
| `Gemfile` | Pins the GitHub Pages Jekyll environment for local preview |
| `assets/icons/` | Brand/tech SVG icons (run `vendor-icons.sh`) |
| `assets/logos/` | Company logos for Work Experience (run `vendor-logos.sh`) |
| `assets/images/gen-photo.png` | Your avatar |

The body of `index.md` is wrapped in `{% raw %}…{% endraw %}` so Jekyll's Liquid
engine never touches the inline JavaScript/HTML.


## Preview locally

```bash
bundle install
bundle exec jekyll serve
# open http://localhost:4000
```

## Deploy to GitHub Pages

1. Push the repo to GitHub.
2. **Settings → Pages → Build and deployment → Source: Deploy from a branch**, pick `main` / root.
3. GitHub Pages builds the Jekyll site automatically (no `.nojekyll` — we *want* Jekyll here).

> Note: for a project site served at `username.github.io/repo/`, set `baseurl: "/repo"`
> in `_config.yml` if any asset paths break. For a user site (`username.github.io`),
> leave `baseurl` empty.

## Editing content

Everything visible lives in `index.md`:
- Hero (name, photo, lead text, Focus chips) — top of the body
- `#experience`, `#education`, `#contact` sections

Styling and the page shell live in `_layouts/default.html` (the `<style>` block and
the `:root` CSS variables at the top control the whole theme).

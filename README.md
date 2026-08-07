# antoninoscaffidi.github.io

Personal tech blog by Antonino Scaffidi Chiarello — Ruby on Rails, web development, and AI.

**Live site:** [antoninoscaffidi.github.io](https://antoninoscaffidi.github.io) ([Italiano](https://antoninoscaffidi.github.io/it/))

## Stack

- [Jekyll](https://jekyllrb.com/) 4.4.1 + [minima](https://github.com/jekyll/minima) theme
- [jekyll-polyglot](https://github.com/untra/polyglot) for EN/IT bilingual content
- Deployed via GitHub Actions to GitHub Pages ([`.github/workflows/pages.yml`](.github/workflows/pages.yml))

## Local development

```bash
bundle install
bundle exec jekyll serve
```

Site available at `http://127.0.0.1:4000`.

## Content structure

Posts are organized into series via front matter (`series`, `episode`). Each post has an English version (default, at the site root) and an Italian version (same filename with an `.it` suffix, served under `/it/`), linked via a shared `ref`.

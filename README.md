# Will Au

Source code for my personal website and technical blog:

**[will-hcau.github.io](https://will-hcau.github.io)**

This is where I keep technical notes, small projects, practical tips, and other things I learn along the way.

## Built With

- [Jekyll](https://jekyllrb.com/)
- [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy)
- [GitHub Pages](https://pages.github.com/)

The site started from the [Chirpy Starter](https://github.com/cotes2020/chirpy-starter) template.

## Local Preview

### Dev Container

Open the repository in Visual Studio Code and run **Dev Containers: Reopen in Container**. Then start the site:

```bash
bundle exec jekyll serve --livereload
```

### Local Ruby Installation

With Ruby and Bundler installed:

```bash
bundle install
bundle exec jekyll serve --livereload
```

Open [http://localhost:4000](http://localhost:4000) in a browser.

## Writing a Post

Add Markdown posts to `_posts` using the following filename format:

```text
YYYY-MM-DD-post-title.md
```

Example front matter:

```yaml
---
title: My Post Title
date: 2026-09-14 12:00:00 +0900
categories: [Category]
tags: [tag]
---
```

See Chirpy’s [Writing a New Post](https://chirpy.cotes.page/posts/write-a-new-post/) guide for the available options.

## Publishing

Pushes to `main` trigger the GitHub Actions workflow in `.github/workflows/pages-deploy.yml`. The workflow builds, checks, and deploys the site to GitHub Pages.

## License

The Chirpy Starter template and Chirpy theme are available under the [MIT License](LICENSE).
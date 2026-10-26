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

Unless otherwise noted, original blog articles, example code, and original illustrations by Will AU in this repository are licensed under the [MIT License](LICENSE). This permits reuse, modification, and redistribution, including commercial use, subject to the license conditions, including preservation of the applicable copyright and permission notices.

### Upstream site code

This site is based on [Chirpy Starter](https://github.com/cotes2020/chirpy-starter) and uses the [Chirpy theme](https://github.com/cotes2020/jekyll-theme-chirpy). Their upstream MIT licenses and copyright notices remain applicable. The repository's `LICENSE` preserves the existing Cotes Chung notice alongside the notice for Will AU's original contributions.

### Third-party material and exceptions

The MIT grant for original content does **not** relicense third-party material. Quoted text, paper figures, screenshots containing third-party content, game artwork, adapted tutorial material, and other externally sourced assets remain subject to their respective licenses, permissions, or other applicable rights. Linked and embedded external content is not covered by this repository's MIT grant.

For example, the Flappy Bird article includes figures from research papers and game imagery; those are not claimed as original illustrations by Will AU. Vulkan tutorial material adapted under upstream terms likewise retains those terms. Consult the source links and any item-specific notices before reusing such material. Attribution alone does not establish permission to reuse it.

An item-specific license or notice takes precedence over the general license statement. Dependencies and separately linked repositories retain their own licenses. This statement applies to this blog repository and does not assign a license to the separate Vulkan example repository.

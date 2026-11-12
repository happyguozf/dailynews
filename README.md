# Daily News

This repository contains the Hugo source for [Daily News](https://happyguozf.github.io/dailynews/). Posts live in `content/posts/` and use Markdown front matter with `title`, `date`, `tags`, and `author` fields.

## Local preview

Install Hugo (the workflows currently use Hugo 0.147.9), then run:

```sh
hugo server
```

Build the static site locally with:

```sh
hugo --minify
```

The generated site is written to `public/`.

## GitHub Pages setup

In the repository, open **Settings → Pages** and set **Build and deployment → Source** to **GitHub Actions**. The `Hugo build` workflow checks pull requests targeting `main`; after changes reach `main`, the `Deploy Hugo site to GitHub Pages` workflow builds and deploys the site. The first deployment only happens after this infrastructure change is merged and the workflow runs successfully.

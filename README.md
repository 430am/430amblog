# 430amblog

Source for [myawesomedemos.org](https://myawesomedemos.org) — a personal blog about cloud, HPC, infrastructure-as-code, and security topics.

Built with [Hugo](https://gohugo.io/) using the [Blowfish](https://blowfish.page/) theme.

## Structure

```
.
├── archetypes/        # Templates for new content
├── assets/            # Source assets processed by Hugo Pipes (images, etc.)
├── config/_default/   # Hugo site configuration (TOML)
│   ├── hugo.toml         # Core site config
│   ├── languages.en.toml # Language settings
│   ├── markup.toml       # Markdown / syntax highlighting
│   ├── menus.en.toml     # Navigation menus
│   ├── module.toml       # Hugo modules
│   └── params.toml       # Theme parameters
├── content/posts/     # Blog posts (Markdown)
├── data/              # Site data files
├── i18n/              # Translation strings
├── layouts/           # Site-specific layout overrides
├── public/            # Generated site output (build artifact)
├── resources/         # Hugo build cache
├── static/            # Static files copied as-is to the site root
└── themes/blowfish/   # Blowfish theme (git submodule)
```

## Prerequisites

- [Hugo extended](https://gohugo.io/installation/) (matching the version required by Blowfish)
- Git (for submodule support)

## Getting started

Clone with submodules so the theme is included:

```bash
git clone --recurse-submodules <repo-url>
cd 430amblog
```

If you cloned without submodules:

```bash
git submodule update --init --recursive
```

## Local development

Run the Hugo dev server with live reload:

```bash
hugo server -D
```

The site will be served at <http://localhost:1313/>. The `-D` flag includes draft posts.

## Building the site

Generate the static site into `public/`:

```bash
hugo --minify
```

## Adding a post

```bash
hugo new posts/my-post/index.md
```

Then edit the generated front matter and content. Place any post-local images alongside `index.md`.

## Configuration

Site-wide settings live under [config/_default/](config/_default/). Theme parameters (colour scheme, appearance, search, etc.) are in [config/_default/params.toml](config/_default/params.toml). See the [Blowfish docs](https://blowfish.page/docs/) for the full list of options.

## Updating the theme

```bash
git submodule update --remote themes/blowfish
```

## Content

Existing post categories include cloud, HPC, infrastructure-as-code, and security, with tags such as Azure, CycleCloud, Terraform, and Key Vault.

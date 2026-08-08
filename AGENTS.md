# AGENTS.md

Guidance for AI coding agents working on rolandsdev.blog, a [Hugo](https://gohugo.io)
static site deployed to Netlify. When in doubt, read the config and run the build
before asserting anything.

## Workflow

- Agree on the approach before implementing anything non-trivial.
- One unit of change per commit. Never mix unrelated changes. Present the change
  for review before committing.
- Verify against the code and the tools: read before you answer, run before you
  assert. A clean build is the bar, not a guess.

Local build (this site's CI; must be clean, no `WARN` or `ERROR`):

```shell
rm -rf public resources
hugo --gc --minify
```

Preview with drafts while writing:

```shell
hugo serve -D --disableFastRender
```

## Writing: content, comments, config, commits

- Concise and to the point. No fluff. Explain the non-obvious; do not narrate the
  obvious.
- ASCII only in prose. No em-dash and no `--`; write `-`. Use `->` not the arrow
  glyph, `!=` not the not-equal glyph.
- Comments and config notes justify *why*, not *what*. Delete any note that just
  restates the setting.

## Commits

- Conventional Commits, subject in the present tense, imperative voice:
  `feat: add search`, not `added` or `adds`. Lowercase subject. Types: build,
  chore, ci, docs, feat, fix, perf, refactor, revert, style, test.
- Keep the body minimal or omit it. Add a body only for what the diff cannot show
  (why, a trade-off, a consequence). Wrap at ~72 columns.
- Disclose AI with an `Assisted-by: Claude:claude-opus-4-8` trailer. Never
  `Co-Authored-By`, and never add a human `Signed-off-by`.

Example:

```
feat: add search

Wire Fuse-based client search; it needs the JSON output enabled on the
home page in config.yaml.

Assisted-by: Claude:claude-opus-4-8
```

## Content

- Posts live in `content/posts/`. Create one with `hugo new posts/<slug>.md`; it
  seeds front matter from `archetypes/default.md`.
- A post with images is a leaf bundle: a folder with `index.md` plus its images
  (`content/posts/<slug>/index.md`). The URL comes from the folder name.
- Front matter booleans are real YAML booleans: `draft: true`, `toc.enable: false`.
  Never `yes`/`no` - modern Hugo reads those as strings, not booleans.
- Taxonomies: `tags`, `categories`, `series`, and `authors` (authors are defined
  in `data/authors/`).

## Theme and styling

- The theme is [DoIt](https://github.com/HEIGE-PCloud/DoIt), pinned as a git
  submodule at `themes/DoIt` tracking upstream. Run `git submodule update --init`
  after cloning. Update it by checking out a new tag in the submodule and
  committing the pointer (see README.md).
- Site config is `config.yaml`. Style overrides live in `assets/css/`:
  - `_override.scss` for the Sass variables the theme still exposes (fonts).
  - `_custom.scss` for plain CSS, including dark-theme tweaks via the theme's
    `html.dark { --... }` custom properties.
- DoIt v1.0 uses Tailwind CSS v4 and CSS custom properties for theming; colors are
  no longer set through `$*-dark` Sass variables. The default dark theme is a
  Primer/GitHub palette.

## Toolchain and deployment

- Hugo extended `>= 0.146.0` (the theme requires it). The build version is pinned
  in `netlify.toml`; keep your local version at or above it.
- Netlify auto-deploys `main`. Merge only after the local build passes and, for
  visual changes, after eyeballing a Netlify deploy preview.

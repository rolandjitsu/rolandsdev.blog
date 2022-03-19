# rolandsdev.blog
> Home of [rolandsdev.blog](https://rolandsdev.blog) content.

## Development

### Prerequisites:
Install [Hugo](https://gohugo.io/getting-started/quick-start/):
```bash
brew install hugo
```
**NOTE** For other platforms, see the [installation guide](https://gohugo.io/getting-started/installing/).

### Setup
These steps were only needed during the initial setup, so this section should be skipped.

1. Init the site:
```bash
hugo new site --force .
```

2. Add a theme ([DoIt](https://hugodoit.pages.dev/)):
```bash
git submodule add https://github.com/HEIGE-PCloud/DoIt.git themes/DoIt
```

### Serve/Run
To serve content (w/ drafts) run:
```bash
hugo serve -D --disableFastRender
```
**NOTE** The `--disableFastRender` flag is recommended by the [DoIt](https://hugodoit.pages.dev/theme-documentation-basics/#launching-the-website-locally) authors.

### Add Content
To add a post, run:
```bash
hugo new posts/my-post.md
```

### Build
To build the static assets:
```bash
hugo -D
```

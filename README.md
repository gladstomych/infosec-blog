# gladstomych.dev

Personal blog built with Hugo + Terminal theme. Deployed via GitHub Pages.

## Local Development

```bash
hugo server -D    # Preview with drafts
hugo              # Build to /public
```

## Adding Content

### Blog Post

1. Create folder: `content/blog/my-post-slug/`
2. Add `index.md` with frontmatter:
   ```toml
   +++
   title = "Post Title"
   date = 2025-01-15
   tags = ["azure", "red-teaming"]
   +++

   ![banner](banner.png)

   Post content here...

   ---

   *Originally published on [JUMPSEC Labs](https://labs.jumpsec.com/post-slug/)*
   ```
3. Drop images in same folder, reference as `![alt](image.png)`

### Talk Slides

1. Add PDF to `static/slides/`
2. Edit `content/talks/_index.md`:
   ```markdown
   ### Talk Title (Conference)
   Brief description.

   [Slides](/slides/Talk%20Name.pdf)
   ```

### Tool

Edit `content/tools/_index.md`:
```markdown
## [ToolName](https://github.com/user/repo)
One-line description.
```

## Structure

```
content/
├── about/_index.md      # About page
├── blog/
│   ├── _index.md        # Blog listing
│   └── post-slug/
│       ├── index.md     # Post content
│       └── *.png        # Images
├── talks/_index.md      # Talks page
└── tools/_index.md      # Tools page
static/
└── slides/*.pdf         # Talk slides
config.toml              # Site config
```

## Config Notes

- `config.toml` - theme color, menu, metadata
- Theme: [Terminal](https://github.com/panr/hugo-theme-terminal) (submodule in `themes/terminal`)
- Deployment: `.github/workflows/hugo.yml` auto-deploys on push to main

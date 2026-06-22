# Converting PRESENTATION.md to PowerPoint

## Quick Conversion Options

### Option 1: Pandoc (Recommended)

**Install Pandoc:**
```bash
# Ubuntu/Debian
sudo apt-get install pandoc

# macOS
brew install pandoc

# Windows
choco install pandoc
```

**Convert to PowerPoint:**
```bash
cd docs/testing
pandoc PRESENTATION.md -o test-harness-presentation.pptx
```

**With custom theme:**
```bash
pandoc PRESENTATION.md -o test-harness-presentation.pptx \
  --reference-doc=your-template.pptx
```

---

### Option 2: Marp (Modern, Best Formatting)

**Install Marp CLI:**
```bash
npm install -g @marp-team/marp-cli
```

**Convert to PowerPoint:**
```bash
cd docs/testing
marp PRESENTATION.md -o test-harness-presentation.pptx
```

**Convert to PDF:**
```bash
marp PRESENTATION.md -o test-harness-presentation.pdf
```

**Convert to HTML (interactive):**
```bash
marp PRESENTATION.md -o test-harness-presentation.html
```

---

### Option 3: Marp for VS Code (GUI)

1. Install VS Code extension: "Marp for VS Code"
2. Open `PRESENTATION.md` in VS Code
3. Click "Open Marp Preview" (top-right icon)
4. Click "Export Slide Deck" → Choose format (PPTX, PDF, HTML)

---

### Option 4: Online Converters

**Marp Web:**
- Go to: https://web.marp.app/
- Copy/paste content from PRESENTATION.md
- Export to PDF or HTML

**HackMD:**
- Go to: https://hackmd.io/
- Create new note
- Paste content
- Export as slides

---

## Customization Tips

### Change Theme

Add to top of PRESENTATION.md:
```yaml
---
marp: true
theme: gaia  # or uncover, default
---
```

### Custom Colors

Add CSS in PRESENTATION.md:
```markdown
<style>
section {
  background-color: #1a1a1a;
  color: #ffffff;
}
</style>
```

### Add Images

```markdown
![width:600px](path/to/image.png)
```

### Multi-column Layout

```markdown
<div class="columns">
<div>

Content for left column

</div>
<div>

Content for right column

</div>
</div>
```

---

## Tips for Best Results

1. **Use Marp for best formatting** - It's designed for Markdown slides
2. **Code blocks auto-format** - Syntax highlighting included
3. **ASCII diagrams work best** in monospace fonts
4. **Each `---` creates a new slide**
5. **Presenter notes:** Use `<!-- Comment -->` syntax

---

## Quick Pandoc + Custom Template

Create `custom-template.pptx` with your branding, then:

```bash
pandoc PRESENTATION.md \
  -o output.pptx \
  --reference-doc=custom-template.pptx
```

Pandoc will use your template's:
- Fonts
- Colors
- Master slide layouts
- Company branding

---

## Example Commands

```bash
# Simple conversion
marp PRESENTATION.md -o slides.pptx

# PDF with theme
marp PRESENTATION.md -o slides.pdf --theme gaia

# HTML for web presentation
marp PRESENTATION.md -o slides.html --html

# Watch mode (auto-reload on save)
marp -w PRESENTATION.md

# Server mode (preview in browser)
marp -s PRESENTATION.md
```

---

## Troubleshooting

**Issue:** Code blocks not formatting correctly
**Solution:** Use Marp instead of Pandoc

**Issue:** ASCII diagrams look wrong
**Solution:** Ensure monospace font in template

**Issue:** Slides too crowded
**Solution:** Split content across more slides (add more `---`)

**Issue:** Export button missing
**Solution:** Install Marp CLI or VS Code extension

---

## Recommended Workflow

1. **Edit** in VS Code with Marp extension
2. **Preview** live as you edit
3. **Export** to PPTX for distribution
4. **Share** PDF for read-only viewing

The Markdown source (`PRESENTATION.md`) remains the source of truth!

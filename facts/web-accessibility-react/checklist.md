## Pre-Deploy Accessibility Checklist

**Semantic HTML**
- [ ] Only one `<h1>` per page
- [ ] Headings in logical order (h1 > h2 > h3, no skips)
- [ ] Buttons for actions, links for navigation
- [ ] Lists use `<ul>`, `<ol>`, `<li>`
- [ ] Forms use `<form>`, `<label>`, `<input>`

**Keyboard Navigation**
- [ ] All interactive elements keyboard accessible (Tab/Enter)
- [ ] Tab order follows DOM order
- [ ] Focus visible (not hidden with outline: none)
- [ ] Modals trap focus (Tab loops within modal)
- [ ] Escape closes modals/popups

**Screen Reader (NVDA/JAWS)**
- [ ] Page structure announced correctly
- [ ] Form labels announced with inputs
- [ ] Button purposes clear
- [ ] Images have alt text (or alt="" if decorative)
- [ ] Custom components have proper ARIA roles

**Color & Contrast**
- [ ] Text contrast ratio 4.5:1 (AA standard)
- [ ] Graphics contrast ratio 3:1
- [ ] Color not the only way to convey information
- [ ] No red/green only distinction (colorblind safe)

**Focus & Indicators**
- [ ] Focus outline visible on all interactive elements
- [ ] Focus ring contrast ratio 3:1
- [ ] No hidden focus indicators

**Forms**
- [ ] All inputs have labels
- [ ] Error messages associated with inputs (aria-describedby)
- [ ] Required fields marked (aria-required)
- [ ] Form instructions provided
- [ ] Submit button clearly labeled

**Media**
- [ ] Videos have captions
- [ ] Audio has transcript
- [ ] Animated GIFs have warning
- [ ] No content flashing >3x/second

**Testing Tools**
```bash
# Automated
npx axe-core --file index.html
npm install -D eslint-plugin-jsx-a11y

# Manual
# 1. Test with keyboard only (no mouse)
# 2. Use browser's Accessibility Inspector
# 3. Test with screen reader (NVDA free, JAWS commercial)
# 4. Run axe DevTools Chrome extension
```

## Common Mistakes to Avoid

| Issue | Fix |
|-------|-----|
| Missing alt text | `<img alt="description" />` |
| `onclick` on div | Use `<button>` |
| Missing form labels | `<label htmlFor="id">` |
| Skip heading levels | h1 → h2 → h3 order |
| Invisible focus | Ensure outline visible |
| Color only info | Add text + icon indicators |
| Placeholder as label | Use actual `<label>` |
| Modifying tab order | Avoid negative `tabIndex` |

## ESLint Config for A11y

```json
{
  "extends": ["react-app", "plugin:jsx-a11y/recommended"],
  "plugins": ["jsx-a11y"],
  "rules": {
    "jsx-a11y/alt-text": "error",
    "jsx-a11y/anchor-has-content": "error",
    "jsx-a11y/aria-role": "error",
    "jsx-a11y/interactive-supports-focus": "warn",
    "jsx-a11y/click-events-have-key-events": "warn",
    "jsx-a11y/no-static-element-interactions": "warn"
  }
}
```

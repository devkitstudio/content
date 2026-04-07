| Aspect | Tailwind | CSS Modules | Styled Components | Global CSS |
|--------|----------|-------------|------------------|-----------|
| **Bundle Size** | 60-80KB | 0KB (CSS only) | 70-100KB | 50-100KB |
| **Scoped Styles** | No | Yes | Yes | No |
| **Learning Curve** | Low | Medium | High | Very Low |
| **Dynamic Styling** | Limited | Very Limited | Excellent | Very Limited |
| **Design System** | Enforced | Flexible | Flexible | None |
| **Markup Readability** | Poor | Good | Good | Good |
| **CSS Reusability** | Medium | High | High | High |
| **Dark Mode** | Easy (@apply) | Complex | Excellent | Complex |
| **Performance** | Best | Good | Good | Good |
| **Team Onboarding** | Quick | Medium | Slow | Very Quick |
| **Maintainability** | Good | Excellent | Excellent | Poor |
| **Theming Support** | Medium | Low | Excellent | Low |

## Bundle Size Analysis

```
Tailwind only: ~60KB gzipped
  - With purge: ~15KB (unused CSS removed)

CSS Modules: ~0KB (pure CSS)
  - Build: ~50-100KB depending on CSS

Styled Components: ~70KB gzipped
  + Application CSS in JS: +20KB

Global CSS: ~50-100KB gzipped
  - No scoping: cascading issues
```

## Performance Metrics (Web Vitals)

**Tailwind:**
- First Contentful Paint: Best (~1.2s)
- Cumulative Layout Shift: Good (0.05)
- No runtime CSS calculation

**CSS Modules:**
- First Contentful Paint: Best (~1.2s)
- Cumulative Layout Shift: Good (0.05)
- No runtime overhead

**Styled Components:**
- First Contentful Paint: Slower (~1.8s)
- Cumulative Layout Shift: Good (0.05)
- Runtime style injection: 5-20ms

**Global CSS:**
- First Contentful Paint: Good (~1.4s)
- Cumulative Layout Shift: Poor (0.15+)
- Cascading complexity increases

## Scalability Over Time

```
Team Size    Development Speed    Maintainability    Best Choice
────────────────────────────────────────────────────────────────
1-5          Very Fast            Easy               Tailwind
5-15         Fast                 Good               Tailwind + Modules
15-50        Medium               Needs Discipline   Tailwind + Modules
50+          Slower               Complex            Tailwind + Modules + Config

At scale: Hybrid approach essential to avoid CSS chaos.
```

## Real-World Recommendation

**Start**: Tailwind (80% + CSS Module escape hatch)
```typescript
// 80% of components use Tailwind
// 15% use CSS Modules for scoped complexity
// 5% use styled-components only for:
//   - Heavy theming
//   - Dynamic animation
//   - Complex state-based styles
```

**Why?**
- Tailwind forces consistency early
- CSS Modules available when needed
- Styled Components optional for hard problems
- No early over-engineering

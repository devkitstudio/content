# Image Optimization Checklist

## Compression & Formats

- [ ] **Use modern formats**: WebP primary, JPEG fallback
- [ ] **Add AVIF**: Best compression, use with fallback
- [ ] **Compress images**: 75-80% quality for JPEG, 75% for WebP
- [ ] **Remove metadata**: Use tools like ImageOptim, Squoosh
- [ ] **Target max sizes**:
  - Thumbnail: < 50KB
  - List item: < 100KB
  - Full width: < 200KB
  - Hero image: < 300KB

## Responsive Delivery

- [ ] **Add srcset**: Serve 400w, 800w, 1200w, 1600w versions
- [ ] **Use sizes attribute**: Match CSS breakpoints
- [ ] **Lazy load below fold**: `loading="lazy"` + Intersection Observer
- [ ] **Priority above fold**: `loading="eager"` or `priority={true}`
- [ ] **Picture element**: For format selection

## Layout Optimization

- [ ] **Set width/height**: Prevents layout shift
- [ ] **Use aspect-ratio CSS**: Reserves space before image loads
- [ ] **Add placeholder**: Blur, gradient, or solid color
- [ ] **Minimal CLS**: Target < 0.1 score

## Next.js Specific

- [ ] **Use next/image**: Automatic optimization
- [ ] **Set correct sizes**: Match responsive breakpoints
- [ ] **Use priority**: For hero images above fold
- [ ] **Enable blur placeholder**: With blurDataURL
- [ ] **Correct quality setting**: Default 75 is good

## Testing & Monitoring

- [ ] **Test LCP**: Google PageSpeed Insights
- [ ] **Check file sizes**: DevTools Network tab
- [ ] **Monitor CLS**: Web Vitals extension
- [ ] **A/B test formats**: Measure real user impact
- [ ] **Mobile performance**: Test on slow 3G

## CDN Configuration

- [ ] **Enable compression**: gzip + brotli
- [ ] **Cache headers**: Set long expiry (1 year+)
- [ ] **Use hash in filename**: For cache busting
- [ ] **Global distribution**: Serve from edge locations
- [ ] **Image API**: Use service like Cloudinary, Imgix

## Common Issues & Fixes

| Issue | Cause | Fix |
|-------|-------|-----|
| LCP too slow | Image not optimized | Compress, use WebP, lazy load |
| CLS high | Image size not reserved | Add width/height or aspect-ratio |
| Poor mobile | Large images | Use responsive srcset |
| Format not supported | No fallback | Add picture with multiple sources |
| Cache not working | Wrong headers | Set Cache-Control: max-age |

## Performance Budget

```
Target per page: 800KB total images
- 10% hero: 80KB
- 40% product images: 320KB
- 50% thumbnails: 400KB

Monitor: 100KB over budget triggers alert
```

## Tools for Optimization

```bash
# Compression
npm install sharp
npm install imagemin-cli

# Format conversion
npx squoosh-cli image.jpg --webp

# Analysis
npm install lighthouse
npx lighthouse https://yoursite.com

# Monitoring
Google PageSpeed Insights
web.dev/measure
```

## Quick Wins Ranking

1. **Switch to WebP** - 30-50% size reduction
2. **Compress to 75% quality** - 20-40% reduction
3. **Add lazy loading** - Improves perceived performance
4. **Responsive images** - 30-50% savings on mobile
5. **AVIF format** - Additional 20% savings
6. **Blur placeholder** - Better perceived performance
7. **Aspect ratio reservation** - Fix CLS

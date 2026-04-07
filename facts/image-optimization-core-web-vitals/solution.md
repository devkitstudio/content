# Image Optimization Strategies

## 1. Lazy Loading Images

```html
<!-- HTML Native Lazy Loading -->
<img
  src="image.jpg"
  loading="lazy"
  alt="Product"
  width="300"
  height="200"
/>

<!-- With Intersection Observer for older browsers -->
<img
  data-src="image.jpg"
  src="placeholder.jpg"
  loading="lazy"
  alt="Product"
/>

<script>
  const images = document.querySelectorAll('img[data-src]');
  const imageObserver = new IntersectionObserver((entries) => {
    entries.forEach((entry) => {
      if (entry.isIntersecting) {
        const img = entry.target;
        img.src = img.dataset.src;
        img.removeAttribute('data-src');
        imageObserver.unobserve(img);
      }
    });
  });
  images.forEach((img) => imageObserver.observe(img));
</script>
```

## 2. Responsive Images with srcset

```html
<!-- Serve different sizes for different viewports -->
<img
  src="image-800w.jpg"
  srcset="
    image-400w.jpg 400w,
    image-800w.jpg 800w,
    image-1200w.jpg 1200w,
    image-1600w.jpg 1600w
  "
  sizes="(max-width: 600px) 400px, (max-width: 1200px) 800px, 1200px"
  alt="Product"
/>
```

## 3. Modern Image Formats

```html
<!-- WebP with fallback -->
<picture>
  <source srcset="image.webp" type="image/webp" />
  <source srcset="image.jpg" type="image/jpeg" />
  <img src="image.jpg" alt="Product" />
</picture>

<!-- AVIF (even better) -->
<picture>
  <source srcset="image.avif" type="image/avif" />
  <source srcset="image.webp" type="image/webp" />
  <source srcset="image.jpg" type="image/jpeg" />
  <img src="image.jpg" alt="Product" />
</picture>
```

## 4. Next.js Image Component

```javascript
import Image from 'next/image';

export default function ProductCard({ product }) {
  return (
    <Image
      src={product.imageUrl}
      alt={product.name}
      width={300}
      height={200}
      loading="lazy"
      quality={75}
      sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
      placeholder="blur"
      blurDataURL={product.blurImage}
    />
  );
}
```

## 5. Responsive Image Component (React)

```javascript
function ResponsiveImage({ src, alt, width, height }) {
  const generateSrcSet = (baseSrc) => {
    const sizes = [400, 800, 1200, 1600];
    return sizes
      .map((size) => `${baseSrc}?w=${size} ${size}w`)
      .join(', ');
  };

  return (
    <picture>
      <source
        srcSet={generateSrcSet(src.replace(/\.\w+$/, '.webp'))}
        type="image/webp"
      />
      <source srcSet={generateSrcSet(src)} type="image/jpeg" />
      <img
        src={src}
        srcSet={generateSrcSet(src)}
        sizes="(max-width: 600px) 400px, (max-width: 1200px) 800px, 1200px"
        alt={alt}
        width={width}
        height={height}
        loading="lazy"
      />
    </picture>
  );
}
```

## 6. Lazy Load with Placeholder Blur

```javascript
import { useState } from 'react';

function LazyImage({ src, alt }) {
  const [isLoaded, setIsLoaded] = useState(false);

  return (
    <div style={{ position: 'relative', paddingBottom: '75%' }}>
      {/* Placeholder */}
      <div
        style={{
          position: 'absolute',
          top: 0,
          left: 0,
          width: '100%',
          height: '100%',
          background: '#f0f0f0',
          opacity: isLoaded ? 0 : 1,
          transition: 'opacity 0.3s',
        }}
      />

      {/* Main image */}
      <img
        src={src}
        alt={alt}
        loading="lazy"
        style={{
          position: 'absolute',
          top: 0,
          left: 0,
          width: '100%',
          height: '100%',
          objectFit: 'cover',
        }}
        onLoad={() => setIsLoaded(true)}
      />
    </div>
  );
}
```

## 7. Image Compression Strategies

```bash
# Install image optimization tool
npm install sharp

# Compress images
node compress-images.js
```

```javascript
// compress-images.js
import sharp from 'sharp';
import fs from 'fs';
import path from 'path';

const imageDir = './public/images';

fs.readdirSync(imageDir).forEach(async (file) => {
  const inputPath = path.join(imageDir, file);
  const baseName = path.parse(file).name;

  // Generate JPEG (quality 75)
  await sharp(inputPath)
    .resize(1600, 1600, { withoutEnlargement: true })
    .jpeg({ quality: 75 })
    .toFile(path.join(imageDir, `${baseName}-1600.jpg`));

  // Generate WebP
  await sharp(inputPath)
    .resize(1600, 1600, { withoutEnlargement: true })
    .webp({ quality: 75 })
    .toFile(path.join(imageDir, `${baseName}-1600.webp`));

  // Generate AVIF
  await sharp(inputPath)
    .resize(1600, 1600, { withoutEnlargement: true })
    .avif({ quality: 60 })
    .toFile(path.join(imageDir, `${baseName}-1600.avif`));
});
```

## Results: Before vs After

```
BEFORE:
- 50 images, 2.5MB total
- LCP: 8.2s
- FCP: 3.1s
- CLS: 0.15 (poor)

AFTER (with optimization):
- 50 images, 450KB total (82% reduction)
- LCP: 1.8s (78% faster)
- FCP: 0.9s (71% faster)
- CLS: 0.02 (excellent)
```

## Core Web Vitals Key Points

- **LCP (Largest Contentful Paint)**: Optimize hero image above fold
- **FCP (First Contentful Paint)**: Serve smallest useful image first
- **CLS (Cumulative Layout Shift)**: Reserve space with aspect ratio

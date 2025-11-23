# AudioStoryV2 Integration Guide

This guide covers integrating the AWS Image Optimization service into your AudioStoryV2 (DirtyVocal) application.

## Overview

After deploying the image optimization stack at `images.dirtyvocal.com`, you'll need to update your AudioStoryV2 application to:

1. Use optimized image URLs for display (read operations)
2. Keep original upload flow unchanged (write operations)
3. Add responsive image sizing based on context
4. Maintain backward compatibility with existing image URLs

## Architecture Summary

```
┌─────────────────────────────────────────────────────────────┐
│                     AudioStoryV2 App                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  UPLOADS (unchanged):                                       │
│  User → Next.js API → S3 (dirtyvocal-assets)              │
│  URL stored in DB: media.dirtyvocal.com/{path}             │
│                                                             │
│  DISPLAY (new):                                            │
│  DB → getOptimizedImageUrl() → images.dirtyvocal.com       │
│  Adds: ?format=auto&width={size}&quality=85                │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│              Image Optimization Service                     │
├─────────────────────────────────────────────────────────────┤
│  images.dirtyvocal.com → CloudFront → Lambda → S3          │
│  Transforms: WebP/AVIF, Resize, Compress, Cache            │
└─────────────────────────────────────────────────────────────┘
```

## Integration Steps

### Phase 1: Utility Functions (Core Logic)

Create a new utility file or update existing `lib/utils.ts`:

#### File: `/Users/llmtest/Projects/AudioStoryV2/lib/utils.ts`

Add these functions:

```typescript
/**
 * Image size presets for different use cases
 */
export const IMAGE_SIZES = {
  THUMBNAIL: 200,      // User profiles, small icons
  SMALL: 300,          // List items, small cards
  MEDIUM: 500,         // Regular cards, grid items
  LARGE: 800,          // Headers, featured content
  XLARGE: 1000,        // Full-page hero images
  PLAYER: 400,         // Audio player artwork
} as const;

/**
 * Convert media.dirtyvocal.com URL to optimized images.dirtyvocal.com URL
 *
 * @param imageUrl - Original image URL from database
 * @param width - Target width (use IMAGE_SIZES constants)
 * @param quality - Image quality 1-100 (default: 85)
 * @param format - Image format (default: 'auto' for browser-based selection)
 * @returns Optimized image URL with transformation parameters
 */
export function getOptimizedImageUrl(
  imageUrl: string | null | undefined,
  width?: number,
  quality: number = 85,
  format: 'auto' | 'webp' | 'avif' | 'jpeg' | 'png' = 'auto'
): string | null {
  if (!imageUrl) return null;

  // Handle relative URLs (local placeholders)
  if (imageUrl.startsWith('/')) {
    return imageUrl;
  }

  // Handle non-media.dirtyvocal.com URLs (e.g., Google profile images)
  if (!imageUrl.includes('media.dirtyvocal.com')) {
    return imageUrl;
  }

  // Extract path from media.dirtyvocal.com URL
  const url = new URL(imageUrl);
  const path = url.pathname;

  // Only optimize image paths (not audio or other assets)
  if (!path.includes('/images/') && !path.match(/\.(jpg|jpeg|png|webp|gif)$/i)) {
    return imageUrl; // Return original URL for non-images
  }

  // Build optimized URL
  const params = new URLSearchParams();
  params.set('format', format);
  if (width) {
    params.set('width', width.toString());
  }
  params.set('quality', quality.toString());

  return `https://images.dirtyvocal.com${path}?${params.toString()}`;
}

/**
 * Get optimized image URL with context-aware sizing
 *
 * @param imageUrl - Original image URL
 * @param context - Display context (determines default size)
 * @param customWidth - Override default width for this context
 */
export function getContextualImageUrl(
  imageUrl: string | null | undefined,
  context: 'thumbnail' | 'card' | 'header' | 'player' | 'small' | 'medium' | 'large',
  customWidth?: number
): string | null {
  const sizeMap = {
    thumbnail: IMAGE_SIZES.THUMBNAIL,
    small: IMAGE_SIZES.SMALL,
    card: IMAGE_SIZES.MEDIUM,
    medium: IMAGE_SIZES.MEDIUM,
    player: IMAGE_SIZES.PLAYER,
    header: IMAGE_SIZES.LARGE,
    large: IMAGE_SIZES.XLARGE,
  };

  const width = customWidth || sizeMap[context];
  return getOptimizedImageUrl(imageUrl, width);
}

/**
 * Generate srcset for responsive images
 *
 * @param imageUrl - Original image URL
 * @param sizes - Array of widths to generate
 * @returns srcset string for <img> element
 */
export function generateImageSrcSet(
  imageUrl: string | null | undefined,
  sizes: number[] = [300, 500, 800, 1000]
): string | null {
  if (!imageUrl) return null;

  return sizes
    .map(width => {
      const url = getOptimizedImageUrl(imageUrl, width);
      return url ? `${url} ${width}w` : null;
    })
    .filter(Boolean)
    .join(', ');
}
```

### Phase 2: Update Image Components

#### Option A: Create New OptimizedImage Component (Recommended)

Create a new component that wraps Next.js Image:

**File**: `/Users/llmtest/Projects/AudioStoryV2/components/ui/optimized-image.tsx`

```typescript
"use client";

import Image from "next/image";
import { useState } from "react";
import { getContextualImageUrl } from "@/lib/utils";

interface OptimizedImageProps {
  src: string | null | undefined;
  alt: string;
  context: 'thumbnail' | 'card' | 'header' | 'player' | 'small' | 'medium' | 'large';
  fallback?: string;
  className?: string;
  fill?: boolean;
  width?: number;
  height?: number;
  priority?: boolean;
}

export function OptimizedImage({
  src,
  alt,
  context,
  fallback = "/images/placeholder/audio-placeholder.png",
  className,
  fill,
  width,
  height,
  priority = false,
}: OptimizedImageProps) {
  const [imgSrc, setImgSrc] = useState(getContextualImageUrl(src, context) || fallback);
  const [hasError, setHasError] = useState(false);

  const handleError = () => {
    if (!hasError) {
      setHasError(true);
      setImgSrc(fallback);
    }
  };

  return (
    <Image
      src={imgSrc}
      alt={alt}
      fill={fill}
      width={!fill ? width : undefined}
      height={!fill ? height : undefined}
      className={className}
      onError={handleError}
      priority={priority}
      unoptimized={true}
    />
  );
}
```

#### Option B: Update Existing ImageWithFallback Component

Update your existing component:

**File**: `/Users/llmtest/Projects/AudioStoryV2/components/image-with-fallback.tsx`

```typescript
// Add this import
import { getContextualImageUrl } from "@/lib/utils";

// Modify the component to add context prop
interface ImageWithFallbackProps {
  src: string | null | undefined;
  fallback: string;
  alt: string;
  context?: 'thumbnail' | 'card' | 'header' | 'player' | 'small' | 'medium' | 'large';
  // ... other props
}

export function ImageWithFallback({
  src,
  fallback,
  alt,
  context = 'medium', // default context
  // ... other props
}: ImageWithFallbackProps) {
  // Replace: const [imgSrc, setImgSrc] = useState(src || fallback);
  // With:
  const optimizedSrc = context
    ? getContextualImageUrl(src, context)
    : src;

  const [imgSrc, setImgSrc] = useState(optimizedSrc || fallback);

  // ... rest of component
}
```

### Phase 3: Update Component Usage

Update your existing components to use optimized images:

#### Example: Song Card Component

**Before**:
```typescript
<Image
  src={song.image}
  alt={song.name}
  fill
  className="object-cover"
/>
```

**After (Option A - New Component)**:
```typescript
<OptimizedImage
  src={song.image}
  alt={song.name}
  context="card"
  fill
  className="object-cover"
/>
```

**After (Option B - Updated Component)**:
```typescript
<ImageWithFallback
  src={song.image}
  alt={song.name}
  context="card"
  fallback="/images/placeholder/song.jpg"
  fill
  className="object-cover"
/>
```

#### Example: User Profile Component

```typescript
<OptimizedImage
  src={user.image}
  alt={user.name}
  context="thumbnail"
  fallback="/images/placeholder/user.jpg"
  width={40}
  height={40}
  className="rounded-full"
/>
```

#### Example: Album Header

```typescript
<OptimizedImage
  src={album.image}
  alt={album.name}
  context="header"
  fallback="/images/placeholder/album.jpg"
  fill
  className="object-cover"
/>
```

#### Example: Audio Player

```typescript
<OptimizedImage
  src={currentTrack.image}
  alt={currentTrack.name}
  context="player"
  fill
  className="object-cover"
/>
```

### Phase 4: Component-by-Component Migration

Gradually update these files (no rush, can be done incrementally):

#### High Priority (User-facing, high traffic):
1. `/components/player/player.tsx` - Audio player artwork
2. `/components/song/card.tsx` - Song cards (homepage, search)
3. `/components/album/card.tsx` - Album cards
4. `/components/playlist/card.tsx` - Playlist cards
5. `/components/user/UserProfileHeader.tsx` - User profiles

#### Medium Priority:
6. `/components/song/song-item.tsx` - List view items
7. `/components/artist/card.tsx` - Artist cards
8. `/components/podcast/card.tsx` - Podcast cards
9. `/components/character/card.tsx` - Character cards

#### Low Priority (less frequent):
10. `/components/account/profile-content.tsx` - Account settings
11. `/components/upload/image-upload.tsx` - Upload preview (optional)

### Phase 5: Environment Variables (Optional)

Add configuration to control the feature:

**File**: `/Users/llmtest/Projects/AudioStoryV2/.env`

```bash
# Image Optimization
NEXT_PUBLIC_IMAGE_OPTIMIZATION_ENABLED=true
NEXT_PUBLIC_IMAGE_CDN_DOMAIN=images.dirtyvocal.com
```

Then update the utility function:

```typescript
export function getOptimizedImageUrl(
  imageUrl: string | null | undefined,
  width?: number,
  quality: number = 85,
  format: 'auto' | 'webp' | 'avif' | 'jpeg' | 'png' = 'auto'
): string | null {
  if (!imageUrl) return null;

  // Feature flag
  const isEnabled = process.env.NEXT_PUBLIC_IMAGE_OPTIMIZATION_ENABLED === 'true';
  if (!isEnabled) return imageUrl;

  // ... rest of function
}
```

### Phase 6: Testing Checklist

After implementing, test these scenarios:

#### Functional Testing:
- [ ] Images load correctly on homepage
- [ ] Song/album/playlist cards display properly
- [ ] User profile images show correctly
- [ ] Player artwork updates when track changes
- [ ] Fallback images work when source is invalid/missing
- [ ] Images work for new uploads
- [ ] Images work for existing content in database

#### Performance Testing:
- [ ] Check Network tab: Images should be WebP/AVIF (not JPEG/PNG)
- [ ] Verify file sizes are smaller (60-80% reduction expected)
- [ ] Check response headers include `x-aws-image-optimization: v1.0`
- [ ] Lighthouse score improves (LCP should decrease)
- [ ] First load is slower (Lambda cold start), subsequent loads are fast

#### Browser Testing:
- [ ] Chrome (should serve AVIF)
- [ ] Firefox (should serve AVIF)
- [ ] Safari (should serve WebP or JPEG)
- [ ] Mobile browsers (iOS Safari, Chrome Mobile)

#### Edge Cases:
- [ ] Non-image URLs (audio files) are not transformed
- [ ] External URLs (Google profile images) work unchanged
- [ ] Local placeholder images (`/images/placeholder/`) work
- [ ] Empty/null image values use fallbacks
- [ ] Images with special characters in filename

## Migration Strategy

### Recommended Approach: Gradual Rollout

**Week 1**: Infrastructure & Testing
- Deploy CDK stack
- Implement utility functions
- Test with a few manual URLs

**Week 2**: Core Components
- Update `ImageWithFallback` or create `OptimizedImage`
- Update player component (high visibility, easy to test)
- Monitor CloudWatch and performance

**Week 3**: Content Cards
- Update song/album/playlist cards
- Monitor cache hit rates and Lambda invocations

**Week 4**: Remaining Components
- Update remaining components
- Performance analysis
- Cost analysis

### Rollback Plan

If issues occur:

1. **Feature Flag Rollback** (if implemented):
   ```bash
   NEXT_PUBLIC_IMAGE_OPTIMIZATION_ENABLED=false
   ```

2. **Component Rollback**:
   - Revert specific component changes
   - Original URLs will still work via `media.dirtyvocal.com`

3. **Infrastructure Rollback**:
   ```bash
   cd /Users/llmtest/Projects/image-optimization
   cdk destroy
   ```

## Upload Flow (Unchanged)

**Important**: Your upload flow should remain unchanged. Images continue to be uploaded to `dirtyvocal-assets` via your existing S3 service:

```typescript
// lib/services/s3.service.ts - NO CHANGES NEEDED
export async function uploadImageToS3(file: File, userId: string) {
  // Continues to upload to dirtyvocal-assets
  // URL stored in DB: https://media.dirtyvocal.com/{userId}/images/{filename}
  // Display layer transforms to: https://images.dirtyvocal.com/...
}
```

The transformation happens only during **display**, not upload.

## Performance Optimizations

### 1. Responsive Images with srcset

For better performance on different screen sizes:

```typescript
<img
  src={getOptimizedImageUrl(song.image, IMAGE_SIZES.MEDIUM)}
  srcSet={generateImageSrcSet(song.image)}
  sizes="(max-width: 640px) 300px, (max-width: 1024px) 500px, 800px"
  alt={song.name}
/>
```

### 2. Priority Loading

Mark above-the-fold images as priority:

```typescript
<OptimizedImage
  src={featuredSong.image}
  context="header"
  priority={true} // Preload this image
/>
```

### 3. Lazy Loading

Below-the-fold images load lazily by default (Next.js behavior).

### 4. Placeholder Strategy

Use blur placeholders for better UX:

```typescript
<Image
  src={getOptimizedImageUrl(song.image, IMAGE_SIZES.MEDIUM)}
  placeholder="blur"
  blurDataURL="/images/placeholder-blur.jpg"
  alt={song.name}
/>
```

## Monitoring & Debugging

### Client-Side Debugging

Add debug logging in development:

```typescript
export function getOptimizedImageUrl(imageUrl: string, width?: number) {
  const optimized = /* ... */;

  if (process.env.NODE_ENV === 'development') {
    console.log('Image Optimization:', {
      original: imageUrl,
      optimized,
      width,
    });
  }

  return optimized;
}
```

### Check Response Headers

In browser DevTools Network tab, check response headers for:
```
x-aws-image-optimization: v1.0
vary: accept
cache-control: public, max-age=31622400
```

### CloudWatch Metrics

Monitor in AWS Console:
- Lambda invocations (should decrease as cache warms up)
- Lambda duration (should be 2-5 seconds per invocation)
- CloudFront cache hit rate (target >90%)

## Cost Monitoring

Track costs in AWS Cost Explorer:
- Filter by service: CloudFront, Lambda, S3
- Set up billing alerts for unexpected increases
- Expected cost: ~$25-35/month for 1M requests

## Troubleshooting

### Issue: Images not loading

**Check**:
1. Original image exists in S3: `https://media.dirtyvocal.com/{path}`
2. DNS is resolving: `nslookup images.dirtyvocal.com`
3. CloudFront distribution is deployed
4. No CORS errors in browser console

**Fix**:
- Verify S3 path in database matches actual S3 key
- Check CloudFront distribution status (must be "Deployed")

### Issue: Some formats not working

**Check**:
- Browser support (Safari doesn't support AVIF)
- Use `format=auto` to let CDN choose best format

**Fix**:
- Update utility to use `format=auto` by default

### Issue: Slow initial loads

**Expected behavior**: First request to a new image is slower (2-5s) due to:
1. Lambda cold start (~1s)
2. Image processing (~1-3s)
3. S3 upload (~0.5-1s)

**Solution**: This is normal. Subsequent requests are fast (<100ms) due to caching.

### Issue: 403 Forbidden errors

**Check**:
- Lambda has permission to read from `audiostory-assets`
- OAC is configured correctly
- Image path doesn't have URL encoding issues

**Fix**:
- Redeploy CDK stack to ensure permissions are correct
- Check CloudWatch Logs for Lambda errors

## Example Usage Patterns

### Pattern 1: Dynamic Sizing Based on Screen Size

```typescript
const getSizeForBreakpoint = () => {
  if (window.innerWidth < 640) return IMAGE_SIZES.SMALL;
  if (window.innerWidth < 1024) return IMAGE_SIZES.MEDIUM;
  return IMAGE_SIZES.LARGE;
};

const imageUrl = getOptimizedImageUrl(album.image, getSizeForBreakpoint());
```

### Pattern 2: Quality Tiers

```typescript
// High quality for hero images
const heroUrl = getOptimizedImageUrl(featured.image, IMAGE_SIZES.XLARGE, 90);

// Standard quality for cards
const cardUrl = getOptimizedImageUrl(song.image, IMAGE_SIZES.MEDIUM, 85);

// Lower quality for thumbnails (smaller = less noticeable)
const thumbUrl = getOptimizedImageUrl(user.image, IMAGE_SIZES.THUMBNAIL, 75);
```

### Pattern 3: Conditional Optimization

```typescript
// Only optimize large images
function getSmartImageUrl(imageUrl: string, originalSize: number) {
  // If original is already small, don't transform
  if (originalSize < 100 * 1024) return imageUrl; // < 100KB

  return getOptimizedImageUrl(imageUrl, IMAGE_SIZES.MEDIUM);
}
```

## Advanced: Server-Side Rendering

For server components, use the same utilities:

```typescript
// app/album/[id]/page.tsx (Server Component)
import { getOptimizedImageUrl, IMAGE_SIZES } from "@/lib/utils";

export default async function AlbumPage({ params }) {
  const album = await getAlbum(params.id);

  return (
    <div>
      <img
        src={getOptimizedImageUrl(album.image, IMAGE_SIZES.LARGE)}
        alt={album.name}
      />
    </div>
  );
}
```

## Summary

### What Changes:
✅ Image display URLs (read operations)
✅ Utility functions to transform URLs
✅ Component props to specify context/size

### What Stays the Same:
✅ Upload flow and S3 service
✅ Database schema (URLs stored as-is)
✅ Existing `media.dirtyvocal.com` URLs continue to work
✅ Audio and non-image assets unchanged

### Benefits:
- 60-80% smaller image file sizes
- Faster page loads (better LCP scores)
- Automatic modern format delivery (WebP/AVIF)
- Global CDN caching
- Responsive images for different screen sizes
- ~$25-35/month for 1M requests

### Next Steps:
1. Implement utility functions in `lib/utils.ts`
2. Test with a single component (e.g., player)
3. Gradually roll out to other components
4. Monitor performance and costs
5. Celebrate faster load times! 🎉

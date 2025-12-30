# PWA Icons

The icon files (`icon-192.png` and `icon-512.png`) are currently copies of the logo.png file.

## Required Actions

**IMPORTANT**: You need to resize these icons to the correct dimensions:

1. **icon-192.png** - Resize to exactly 192x192 pixels
2. **icon-512.png** - Resize to exactly 512x512 pixels

## Tools for Resizing

You can use:
- Online tools: https://www.iloveimg.com/resize-image or https://imageresizer.com
- Image editing software: Photoshop, GIMP, Paint.NET
- Command line: ImageMagick (`magick logo.png -resize 192x192 icon-192.png`)

## Icon Requirements

- Format: PNG
- Sizes: 192x192 and 512x512 (square, 1:1 aspect ratio)
- Purpose: Should be clear and recognizable at small sizes
- Background: Can be transparent or solid color

After resizing, the PWA will work correctly and pass Lighthouse PWA audits.


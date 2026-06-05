# Open Graph and Twitter Cards

## The Idea

**In plain English:** Open Graph and Twitter Cards are special instructions you hide inside a webpage that tell social media platforms (like Facebook or X/Twitter) exactly what image, title, and description to show when someone shares your link — because without them, those platforms just guess, and usually get it wrong.

**Real-world analogy:** Think of sharing a link like handing someone a book to display in a shop window. Without instructions, the shopkeeper might tape a random page to the glass. Open Graph tags are like the official display card the publisher includes that says "show this cover image, this title, and this blurb":
- The book's official display card = the Open Graph / Twitter Card meta tags
- The cover image on the display card = `og:image` (the preview photo shown in the post)
- The title and blurb on the display card = `og:title` and `og:description` (the text shown below the preview)

---

## Overview

Open Graph and Twitter Cards are meta tags that control how your content appears when shared on social media platforms. They enhance link previews with images, titles, and descriptions.

## Open Graph Protocol

Developed by Facebook, now used across many platforms.

### Basic Open Graph tags

```html
<meta property="og:title" content="Page Title">
<meta property="og:description" content="Description of the page content">
<meta property="og:image" content="https://example.com/image.jpg">
<meta property="og:url" content="https://example.com/page/">
<meta property="og:type" content="website">
```

### Complete Open Graph example

```html
<head>
  <!-- Required Open Graph tags -->
  <meta property="og:title" content="Amazing Article Title">
  <meta property="og:description" content="A compelling description that makes people want to click and read more about this amazing content.">
  <meta property="og:image" content="https://example.com/images/share-image.jpg">
  <meta property="og:url" content="https://example.com/articles/amazing-article/">
  <meta property="og:type" content="article">
  
  <!-- Optional but recommended -->
  <meta property="og:site_name" content="Example Site">
  <meta property="og:locale" content="en_US">
  
  <!-- Image details -->
  <meta property="og:image:width" content="1200">
  <meta property="og:image:height" content="630">
  <meta property="og:image:alt" content="Description of image content">
  <meta property="og:image:type" content="image/jpeg">
</head>
```

## Open Graph types

### Website

```html
<meta property="og:type" content="website">
<meta property="og:title" content="Site Name">
<meta property="og:description" content="Site description">
<meta property="og:url" content="https://example.com/">
<meta property="og:image" content="https://example.com/logo.jpg">
```

### Article

```html
<meta property="og:type" content="article">
<meta property="og:title" content="Article Title">
<meta property="article:published_time" content="2025-01-15T08:00:00Z">
<meta property="article:modified_time" content="2025-01-16T10:30:00Z">
<meta property="article:author" content="https://example.com/authors/john-doe/">
<meta property="article:section" content="Technology">
<meta property="article:tag" content="HTML">
<meta property="article:tag" content="Web Development">
```

### Profile

```html
<meta property="og:type" content="profile">
<meta property="og:title" content="John Doe">
<meta property="profile:first_name" content="John">
<meta property="profile:last_name" content="Doe">
<meta property="profile:username" content="johndoe">
<meta property="profile:gender" content="male">
```

### Video

```html
<meta property="og:type" content="video.movie">
<meta property="og:title" content="Movie Title">
<meta property="og:video" content="https://example.com/video.mp4">
<meta property="og:video:width" content="1920">
<meta property="og:video:height" content="1080">
<meta property="og:video:type" content="video/mp4">
<meta property="og:video:duration" content="120">
<meta property="video:release_date" content="2025-01-01">
<meta property="video:director" content="https://example.com/directors/jane-smith/">
```

## Twitter Cards

Twitter's own meta tag system with fallback to Open Graph.

### Twitter Card types

#### Summary card

```html
<meta name="twitter:card" content="summary">
<meta name="twitter:site" content="@sitehandle">
<meta name="twitter:creator" content="@authorhandle">
<meta name="twitter:title" content="Page Title">
<meta name="twitter:description" content="Page description">
<meta name="twitter:image" content="https://example.com/image.jpg">
<meta name="twitter:image:alt" content="Image description">
```

Image specs:
- Minimum: 144x144px
- Maximum: 4096x4096px
- File size: under 5MB
- Aspect ratio: 1:1

#### Summary card with large image

```html
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@sitehandle">
<meta name="twitter:creator" content="@authorhandle">
<meta name="twitter:title" content="Article Title">
<meta name="twitter:description" content="Article description">
<meta name="twitter:image" content="https://example.com/large-image.jpg">
<meta name="twitter:image:alt" content="Descriptive alt text">
```

Image specs:
- Minimum: 300x157px
- Recommended: 1200x628px
- Maximum: 4096x4096px
- Aspect ratio: 2:1 (1.91:1)
- File size: under 5MB

#### App card

```html
<meta name="twitter:card" content="app">
<meta name="twitter:site" content="@sitehandle">
<meta name="twitter:description" content="App description">
<meta name="twitter:app:name:iphone" content="App Name">
<meta name="twitter:app:id:iphone" content="123456789">
<meta name="twitter:app:url:iphone" content="appscheme://action">
<meta name="twitter:app:name:ipad" content="App Name">
<meta name="twitter:app:id:ipad" content="123456789">
<meta name="twitter:app:url:ipad" content="appscheme://action">
<meta name="twitter:app:name:googleplay" content="App Name">
<meta name="twitter:app:id:googleplay" content="com.example.app">
<meta name="twitter:app:url:googleplay" content="appscheme://action">
```

#### Player card

```html
<meta name="twitter:card" content="player">
<meta name="twitter:site" content="@sitehandle">
<meta name="twitter:title" content="Video Title">
<meta name="twitter:description" content="Video description">
<meta name="twitter:player" content="https://example.com/player.html">
<meta name="twitter:player:width" content="1280">
<meta name="twitter:player:height" content="720">
<meta name="twitter:player:stream" content="https://example.com/video.mp4">
<meta name="twitter:player:stream:content_type" content="video/mp4">
<meta name="twitter:image" content="https://example.com/poster.jpg">
```

## Complete social media meta tags

```html
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  
  <!-- Page metadata -->
  <title>Article Title - Site Name</title>
  <meta name="description" content="Article description for search engines">
  
  <!-- Open Graph -->
  <meta property="og:type" content="article">
  <meta property="og:title" content="Article Title">
  <meta property="og:description" content="Compelling description for social sharing">
  <meta property="og:image" content="https://example.com/images/share-1200x630.jpg">
  <meta property="og:image:width" content="1200">
  <meta property="og:image:height" content="630">
  <meta property="og:image:alt" content="Descriptive alt text for the image">
  <meta property="og:url" content="https://example.com/articles/article-slug/">
  <meta property="og:site_name" content="Site Name">
  <meta property="og:locale" content="en_US">
  
  <!-- Article metadata -->
  <meta property="article:published_time" content="2025-01-15T08:00:00Z">
  <meta property="article:modified_time" content="2025-01-16T10:30:00Z">
  <meta property="article:author" content="https://example.com/authors/john-doe/">
  <meta property="article:section" content="Technology">
  <meta property="article:tag" content="HTML">
  <meta property="article:tag" content="Open Graph">
  <meta property="article:tag" content="Social Media">
  
  <!-- Twitter Card -->
  <meta name="twitter:card" content="summary_large_image">
  <meta name="twitter:site" content="@sitehandle">
  <meta name="twitter:creator" content="@authorhandle">
  <meta name="twitter:title" content="Article Title">
  <meta name="twitter:description" content="Description optimized for Twitter">
  <meta name="twitter:image" content="https://example.com/images/share-1200x628.jpg">
  <meta name="twitter:image:alt" content="Descriptive alt text">
  
  <!-- Facebook App ID (optional) -->
  <meta property="fb:app_id" content="123456789">
</head>
```

## Best practices

### Image specifications

#### Open Graph images
- Recommended: 1200x630px (1.91:1 ratio)
- Minimum: 600x315px
- Maximum: 8MB
- Formats: JPG, PNG, WebP, GIF
- Avoid text-heavy images (may not scale well)

#### Twitter images
- Summary card: 1:1 ratio, 144x144px minimum
- Large image: 2:1 ratio, 1200x628px recommended
- Maximum: 5MB
- Formats: JPG, PNG, WebP, GIF

### Multiple images

```html
<!-- Primary image -->
<meta property="og:image" content="https://example.com/image-main.jpg">
<meta property="og:image:width" content="1200">
<meta property="og:image:height" content="630">

<!-- Alternative images -->
<meta property="og:image" content="https://example.com/image-square.jpg">
<meta property="og:image:width" content="1200">
<meta property="og:image:height" content="1200">

<meta property="og:image" content="https://example.com/image-vertical.jpg">
<meta property="og:image:width" content="630">
<meta property="og:image:height" content="1200">
```

### Content guidelines

#### Titles
- Keep under 60 characters for best display
- Be specific and descriptive
- Front-load important keywords
- Don't include site name (added automatically)

#### Descriptions
- 150-200 characters optimal
- Compelling call-to-action
- Different from meta description
- Explain value of clicking

### URL best practices

```html
<!-- Always use absolute URLs -->
<meta property="og:url" content="https://example.com/page/">
<meta property="og:image" content="https://example.com/image.jpg">

<!-- Include protocol (https://) -->
<!-- Include trailing slash for consistency -->
<!-- Use canonical URL -->
```

## Testing and validation

### Facebook Sharing Debugger
https://developers.facebook.com/tools/debug/

```html
<!-- Test and refresh cache -->
```

### Twitter Card Validator
https://cards-dev.twitter.com/validator

### LinkedIn Post Inspector
https://www.linkedin.com/post-inspector/

### Preview example

After sharing on Facebook/Twitter, users see:

```
┌─────────────────────────────────────┐
│ [Image: 1200x630px]                 │
│                                     │
│ Article Title                       │
│ Compelling description text...      │
│ example.com                         │
└─────────────────────────────────────┘
```

## Platform-specific tags

### Facebook

```html
<meta property="fb:app_id" content="123456789">
<meta property="fb:pages" content="987654321">
```

### LinkedIn

LinkedIn uses Open Graph tags, no special tags needed.

### Pinterest

```html
<meta name="pinterest:description" content="Pin description">
<meta property="og:see_also" content="https://example.com/related-1/">
<meta property="og:see_also" content="https://example.com/related-2/">
```

## Dynamic content

### Blog post example

```html
<meta property="og:type" content="article">
<meta property="og:title" content="<?php echo $post_title; ?>">
<meta property="og:description" content="<?php echo $post_excerpt; ?>">
<meta property="og:image" content="<?php echo $post_image; ?>">
<meta property="og:url" content="<?php echo $post_url; ?>">
<meta property="article:published_time" content="<?php echo $post_date; ?>">
<meta property="article:author" content="<?php echo $author_url; ?>">
```

### E-commerce product

```html
<meta property="og:type" content="product">
<meta property="og:title" content="Product Name">
<meta property="og:description" content="Product description">
<meta property="og:image" content="https://example.com/product-image.jpg">
<meta property="og:url" content="https://example.com/products/product-slug/">
<meta property="product:price:amount" content="29.99">
<meta property="product:price:currency" content="USD">
<meta property="product:availability" content="in stock">
```

## Common mistakes to avoid

1. Missing image dimensions
2. Using relative URLs
3. Images too small or large
4. Missing alt text for images
5. Descriptions too long
6. Not testing across platforms
7. Using same description as meta description
8. Forgetting to update URL
9. Not including twitter:card type
10. Images with text that doesn't scale

## Fallback behavior

Twitter falls back to Open Graph:

```html
<!-- Twitter will use these if twitter: tags missing -->
<meta property="og:title" content="Title">
<meta property="og:description" content="Description">
<meta property="og:image" content="https://example.com/image.jpg">

<!-- But twitter:card is required -->
<meta name="twitter:card" content="summary_large_image">
```

## WordPress example

```php
<head>
  <?php if (is_single() || is_page()) : ?>
    <meta property="og:type" content="article">
    <meta property="og:title" content="<?php the_title(); ?>">
    <meta property="og:description" content="<?php echo get_the_excerpt(); ?>">
    <meta property="og:url" content="<?php the_permalink(); ?>">
    <?php if (has_post_thumbnail()) : ?>
      <meta property="og:image" content="<?php echo get_the_post_thumbnail_url(null, 'large'); ?>">
    <?php endif; ?>
    <meta property="article:published_time" content="<?php echo get_the_date('c'); ?>">
  <?php else : ?>
    <meta property="og:type" content="website">
    <meta property="og:title" content="<?php bloginfo('name'); ?>">
    <meta property="og:description" content="<?php bloginfo('description'); ?>">
    <meta property="og:url" content="<?php echo home_url(); ?>">
  <?php endif; ?>
</head>
```

## Key takeaways

- Open Graph and Twitter Cards control social media previews
- Always use absolute URLs with https://
- Include image dimensions for better performance
- Recommended image size: 1200x630px
- Keep titles under 60 characters
- Keep descriptions 150-200 characters
- Test with official validators
- Twitter falls back to Open Graph tags
- Different content than regular meta description
- Include alt text for accessibility
- Update tags for each page/post
- Clear cache when testing changes

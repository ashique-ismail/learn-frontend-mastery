# Landmark roles

## The Idea

**In plain English:** Landmark roles are special labels you attach to big sections of a webpage so that people using screen readers (software that reads the page aloud) can jump straight to the part they want, instead of having to listen to every word from the top.

**Real-world analogy:** Think of a shopping mall directory board at the entrance. The board shows zones like "Food Court," "Main Entrance," "Restrooms," and "Parking." Instead of wandering every corridor, a visitor scans the board and walks directly to what they need.
- The directory board = the set of landmark roles on the page
- Each zone label (Food Court, Restrooms) = a landmark like `<main>`, `<nav>`, or `<footer>`
- The visitor using the board = a screen reader user pressing a shortcut to jump between landmarks

---

## Overview

Landmark roles provide a way to identify and navigate major page sections. They help screen reader users quickly jump to different parts of a page.

## Why landmarks matter

Screen reader users can:
- List all landmarks on a page
- Jump directly to specific landmarks
- Navigate between landmarks
- Skip repetitive content

## Native HTML5 landmarks

Modern HTML provides semantic elements with implicit landmark roles.

### Main content

```html
<!-- Implicit role="main" -->
<main>
  <h1>Page Title</h1>
  <p>Primary content of the page</p>
</main>
```

**Rules:**
- Only one `<main>` per page
- Not inside `<article>`, `<aside>`, `<footer>`, `<header>`, or `<nav>`
- Contains primary content

### Navigation

```html
<!-- Implicit role="navigation" -->
<nav aria-label="Main navigation">
  <ul>
    <li><a href="/">Home</a></li>
    <li><a href="/about">About</a></li>
    <li><a href="/contact">Contact</a></li>
  </ul>
</nav>
```

**Best practices:**
- Use `aria-label` or `aria-labelledby` when multiple navs
- For major navigation sections only
- Not for every group of links

### Header/Banner

```html
<!-- Implicit role="banner" on top-level <header> -->
<header>
  <img src="logo.svg" alt="Company name">
  <nav aria-label="Main navigation">
    <!-- Navigation -->
  </nav>
</header>
```

**Rules:**
- Only top-level `<header>` is a banner landmark
- `<header>` inside `<article>` or `<section>` is not a landmark
- Typically contains logo, site title, main navigation

### Footer/Contentinfo

```html
<!-- Implicit role="contentinfo" on top-level <footer> -->
<footer>
  <p>&copy; 2025 Company Name</p>
  <nav aria-label="Footer navigation">
    <!-- Secondary navigation -->
  </nav>
</footer>
```

**Rules:**
- Only top-level `<footer>` is a contentinfo landmark
- `<footer>` inside `<article>` or `<section>` is not a landmark
- Typically contains copyright, contact info, legal links

### Complementary (aside)

```html
<!-- Implicit role="complementary" -->
<aside aria-labelledby="sidebar-title">
  <h2 id="sidebar-title">Related Articles</h2>
  <ul>
    <li><a href="/article-1">Article 1</a></li>
    <li><a href="/article-2">Article 2</a></li>
  </ul>
</aside>
```

**Use for:**
- Related content
- Sidebars
- Callout boxes
- Supporting information

### Search

```html
<!-- HTML5 search element -->
<search>
  <form role="search">
    <label for="search-input">Search</label>
    <input type="search" id="search-input">
    <button>Search</button>
  </form>
</search>

<!-- Or with role -->
<div role="search">
  <form>
    <label for="search-input">Search</label>
    <input type="search" id="search-input">
    <button>Search</button>
  </form>
</div>
```

### Form

```html
<!-- Only a landmark if it has an accessible name -->
<form aria-label="Contact form">
  <label for="name">Name</label>
  <input type="text" id="name">
  
  <label for="email">Email</label>
  <input type="email" id="email">
  
  <button type="submit">Submit</button>
</form>

<!-- Or with aria-labelledby -->
<form aria-labelledby="form-title">
  <h2 id="form-title">Contact Us</h2>
  <!-- Form fields -->
</form>
```

### Region

```html
<!-- Generic landmark, requires accessible name -->
<section aria-labelledby="featured-title">
  <h2 id="featured-title">Featured Products</h2>
  <!-- Content -->
</section>

<!-- Or with aria-label -->
<section aria-label="Product features">
  <!-- Content -->
</section>
```

**Use for:**
- Important content sections
- Only when no other landmark fits
- Must have accessible name to be a landmark

## Complete page structure

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Page Title</title>
</head>
<body>
  <!-- Banner landmark -->
  <header>
    <a href="/">
      <img src="logo.svg" alt="Company name">
    </a>
    
    <!-- Navigation landmark -->
    <nav aria-label="Main navigation">
      <ul>
        <li><a href="/">Home</a></li>
        <li><a href="/products">Products</a></li>
        <li><a href="/about">About</a></li>
        <li><a href="/contact">Contact</a></li>
      </ul>
    </nav>
  </header>
  
  <!-- Search landmark -->
  <search>
    <form role="search">
      <label for="site-search">Search site</label>
      <input type="search" id="site-search">
      <button>Search</button>
    </form>
  </search>
  
  <!-- Main landmark -->
  <main>
    <h1>Page Heading</h1>
    
    <!-- Region landmark -->
    <section aria-labelledby="intro-title">
      <h2 id="intro-title">Introduction</h2>
      <p>Content...</p>
    </section>
    
    <!-- Articles are not landmarks by default -->
    <article>
      <h2>Article Title</h2>
      <p>Article content...</p>
      
      <!-- This header is NOT a banner -->
      <header>
        <p>By Author Name</p>
        <time datetime="2025-01-15">January 15, 2025</time>
      </header>
      
      <!-- This footer is NOT contentinfo -->
      <footer>
        <p>Tags: HTML, Accessibility</p>
      </footer>
    </article>
  </main>
  
  <!-- Complementary landmark -->
  <aside aria-labelledby="sidebar-title">
    <h2 id="sidebar-title">Related Links</h2>
    <nav aria-label="Related pages">
      <ul>
        <li><a href="/related-1">Related Page 1</a></li>
        <li><a href="/related-2">Related Page 2</a></li>
      </ul>
    </nav>
  </aside>
  
  <!-- Contentinfo landmark -->
  <footer>
    <nav aria-label="Footer navigation">
      <ul>
        <li><a href="/privacy">Privacy</a></li>
        <li><a href="/terms">Terms</a></li>
      </ul>
    </nav>
    <p>&copy; 2025 Company Name</p>
  </footer>
</body>
</html>
```

## Multiple instances of same landmark

When you have multiple landmarks of the same type, label them:

```html
<!-- Multiple navigation landmarks -->
<nav aria-label="Main navigation">
  <ul>
    <li><a href="/">Home</a></li>
    <li><a href="/about">About</a></li>
  </ul>
</nav>

<main>
  <h1>Products</h1>
  <!-- Content -->
</main>

<aside>
  <nav aria-label="Product categories">
    <ul>
      <li><a href="/category-1">Category 1</a></li>
      <li><a href="/category-2">Category 2</a></li>
    </ul>
  </nav>
</aside>

<footer>
  <nav aria-label="Legal">
    <ul>
      <li><a href="/privacy">Privacy</a></li>
      <li><a href="/terms">Terms</a></li>
    </ul>
  </nav>
</footer>
```

## Labeling landmarks

### aria-label

```html
<nav aria-label="Main navigation">
  <!-- No visible heading -->
</nav>
```

### aria-labelledby

```html
<nav aria-labelledby="nav-heading">
  <h2 id="nav-heading">Main Navigation</h2>
  <!-- Links -->
</nav>
```

### When to label

**Always label when:**
- Multiple landmarks of same type
- Landmark purpose isn't obvious from context

**Usually don't need to label:**
- Single `<main>` element
- Top-level `<header>` (banner)
- Top-level `<footer>` (contentinfo)

## Common patterns

### Blog layout

```html
<body>
  <header>
    <h1>Blog Title</h1>
    <nav aria-label="Main navigation">
      <!-- Links -->
    </nav>
  </header>
  
  <main>
    <article>
      <h2>Blog Post Title</h2>
      <p>Content...</p>
    </article>
    
    <nav aria-label="Pagination">
      <a href="/page/1">Previous</a>
      <a href="/page/3">Next</a>
    </nav>
  </main>
  
  <aside aria-label="Sidebar">
    <section aria-labelledby="recent-title">
      <h2 id="recent-title">Recent Posts</h2>
      <!-- List of posts -->
    </section>
  </aside>
  
  <footer>
    <p>&copy; 2025 Blog Name</p>
  </footer>
</body>
```

### E-commerce product page

```html
<body>
  <header>
    <img src="logo.svg" alt="Store name">
    <search>
      <form role="search">
        <input type="search" aria-label="Search products">
        <button>Search</button>
      </form>
    </search>
    <nav aria-label="Main navigation">
      <!-- Category links -->
    </nav>
  </header>
  
  <main>
    <h1>Product Name</h1>
    
    <section aria-labelledby="description-title">
      <h2 id="description-title">Description</h2>
      <p>Product description...</p>
    </section>
    
    <form aria-label="Purchase options">
      <label for="quantity">Quantity</label>
      <input type="number" id="quantity">
      <button>Add to Cart</button>
    </form>
    
    <section aria-labelledby="reviews-title">
      <h2 id="reviews-title">Customer Reviews</h2>
      <!-- Reviews -->
    </section>
  </main>
  
  <aside aria-label="Related products">
    <h2>You may also like</h2>
    <!-- Product recommendations -->
  </aside>
  
  <footer>
    <!-- Footer content -->
  </footer>
</body>
```

### Dashboard

```html
<body>
  <header>
    <h1>Dashboard</h1>
    <nav aria-label="Main navigation">
      <!-- Primary nav -->
    </nav>
  </header>
  
  <nav aria-label="Sidebar navigation">
    <!-- Secondary nav -->
  </nav>
  
  <main>
    <h2>Overview</h2>
    
    <section aria-labelledby="stats-title">
      <h3 id="stats-title">Statistics</h3>
      <!-- Stats widgets -->
    </section>
    
    <section aria-labelledby="activity-title">
      <h3 id="activity-title">Recent Activity</h3>
      <!-- Activity feed -->
    </section>
  </main>
  
  <footer>
    <!-- Footer -->
  </footer>
</body>
```

## Skip links

Complement landmarks with skip links:

```html
<body>
  <a href="#main-content" class="skip-link">
    Skip to main content
  </a>
  
  <header>
    <!-- Header content -->
  </header>
  
  <main id="main-content">
    <!-- Main content -->
  </main>
</body>
```

CSS for skip link:
```css
.skip-link {
  position: absolute;
  top: -40px;
  left: 0;
  background: #000;
  color: #fff;
  padding: 8px;
  z-index: 100;
}

.skip-link:focus {
  top: 0;
}
```

## Testing landmarks

### Screen reader shortcuts

**NVDA:**
- `D` - Next landmark
- `Shift + D` - Previous landmark
- `Insert + F7` - List landmarks

**JAWS:**
- `;` - Next landmark
- `Shift + ;` - Previous landmark
- `Insert + F3` - List landmarks

**VoiceOver (macOS):**
- `VO + U`, then use arrows to navigate landmarks

### Browser extensions

- Landmarks browser extension (Chrome, Firefox)
- WAVE Web Accessibility Evaluation Tool
- axe DevTools

## Best practices

1. Use native HTML5 elements when possible
2. Label landmarks when multiple of same type
3. Don't overuse landmarks (main sections only)
4. Keep landmark structure simple and flat
5. One `<main>` per page
6. Only top-level `<header>` and `<footer>` are landmarks
7. Test with screen readers
8. Provide skip links for additional navigation help

## Common mistakes

```html
<!-- Bad: too many nav landmarks -->
<nav>
  <a href="/page1">Page 1</a>
  <a href="/page2">Page 2</a>
</nav>
<nav>
  <a href="/page3">Page 3</a>
</nav>

<!-- Good: combine related navigation -->
<nav aria-label="Site navigation">
  <a href="/page1">Page 1</a>
  <a href="/page2">Page 2</a>
  <a href="/page3">Page 3</a>
</nav>

<!-- Bad: unlabeled multiple landmarks -->
<nav><!-- Main nav --></nav>
<nav><!-- Footer nav --></nav>

<!-- Good: labeled landmarks -->
<nav aria-label="Main navigation"><!-- Main nav --></nav>
<nav aria-label="Legal"><!-- Footer nav --></nav>

<!-- Bad: region without label (not a landmark) -->
<section>
  <h2>Features</h2>
</section>

<!-- Good: labeled region (is a landmark) -->
<section aria-labelledby="features-title">
  <h2 id="features-title">Features</h2>
</section>
```

## Landmark hierarchy

Good hierarchy:
```
Banner (header)
├─ Navigation
└─ Search

Main
├─ Region (section)
├─ Region (section)
└─ Navigation (pagination)

Complementary (aside)
└─ Navigation (sidebar)

Contentinfo (footer)
└─ Navigation (legal)
```

## Key takeaways

- Landmarks help users navigate page structure
- Use native HTML5 elements for implicit roles
- Only top-level `<header>` and `<footer>` are landmarks
- Label landmarks when multiple of same type exist
- One `<main>` per page
- Don't overuse - only main sections
- `<section>` needs accessible name to be landmark
- Test with screen readers
- Complement with skip links
- Keep structure simple and logical

# Semantic Elements & Document Outline

## Learning Objectives
- Understand the purpose and benefits of semantic HTML
- Master the correct usage of structural semantic elements
- Build accessible and SEO-friendly document outlines
- Recognize when to use semantic elements vs generic containers

---

## What Are Semantic Elements?

Semantic elements are HTML tags that convey **meaning** about their content, not just presentation. They describe the role and purpose of content within a page structure.

**Non-semantic (generic):**
```html
<div class="header">
  <div class="nav">...</div>
</div>
```

**Semantic (meaningful):**
```html
<header>
  <nav>...</nav>
</header>
```

Both render identically by default, but the semantic version communicates structure to:
- **Screen readers** - improving accessibility
- **Search engines** - enhancing SEO
- **Developers** - making code self-documenting
- **Browser features** - enabling reader modes, outline views

---

## Core Structural Elements

### `<header>`

**Purpose:** Introductory content or navigational aids for its nearest ancestor sectioning content or root.

**Usage:**
- Site-wide header (logo, global nav, search)
- Article/section header (title, metadata, author)
- Can have multiple per page (one per section)

```html
<!-- Site header -->
<header>
  <img src="logo.svg" alt="Company Logo">
  <nav>
    <a href="/about">About</a>
    <a href="/contact">Contact</a>
  </nav>
</header>

<!-- Article header -->
<article>
  <header>
    <h2>Article Title</h2>
    <p>By John Doe • <time datetime="2025-05-26">May 26, 2025</time></p>
  </header>
  <!-- article content -->
</article>
```

**❌ Common Mistake:** Don't nest `<header>` inside `<header>` or inside `<footer>` or `<address>`.

---

### `<nav>`

**Purpose:** Major navigation sections - primary site navigation, table of contents, or pagination.

**Key Rule:** Not every group of links needs `<nav>`. Reserve it for **major** navigation blocks.

```html
<!-- ✅ Good: Primary navigation -->
<nav aria-label="Main">
  <ul>
    <li><a href="/">Home</a></li>
    <li><a href="/products">Products</a></li>
    <li><a href="/about">About</a></li>
  </ul>
</nav>

<!-- ✅ Good: Breadcrumbs -->
<nav aria-label="Breadcrumb">
  <ol>
    <li><a href="/">Home</a></li>
    <li><a href="/products">Products</a></li>
    <li aria-current="page">Product Name</li>
  </ol>
</nav>

<!-- ❌ Bad: Footer links don't need <nav> unless they're major navigation -->
<footer>
  <!-- Just use plain links here -->
  <a href="/privacy">Privacy</a>
  <a href="/terms">Terms</a>
</footer>
```

**Accessibility Tip:** When you have multiple `<nav>` elements, use `aria-label` to distinguish them.

---

### `<main>`

**Purpose:** The dominant content of the `<body>`. Only **one** `<main>` per page (not hidden).

**What belongs in `<main>`:**
- Core page content
- The unique content for this page

**What doesn't belong:**
- Site header/footer
- Sidebars that repeat across pages
- Navigation

```html
<body>
  <header><!-- site header --></header>
  
  <main>
    <h1>About Us</h1>
    <p>This is the unique content for this page...</p>
    
    <section>
      <h2>Our Mission</h2>
      <!-- ... -->
    </section>
  </main>
  
  <aside><!-- sidebar --></aside>
  <footer><!-- site footer --></footer>
</body>
```

**Accessibility:** Screen readers have shortcuts to jump to `<main>`, so users can skip repetitive content.

---

### `<article>`

**Purpose:** A **self-contained** composition that could be independently distributed or reused.

**Test:** "Could this be syndicated in an RSS feed, or republished elsewhere?"

**Good use cases:**
- Blog post
- News article
- Forum post
- Product card
- User comment

```html
<article>
  <header>
    <h2>Understanding React Hooks</h2>
    <p>Published on <time datetime="2025-05-26">May 26, 2025</time></p>
  </header>
  
  <p>React Hooks revolutionized how we write components...</p>
  
  <footer>
    <p>Tags: <a href="/tag/react">React</a>, <a href="/tag/hooks">Hooks</a></p>
  </footer>
</article>
```

**Nesting:** You can nest `<article>` inside `<article>`. Example: blog post (outer `<article>`) containing comments (inner `<article>` elements).

---

### `<section>`

**Purpose:** A thematic grouping of content, typically with a heading. It's a **generic** sectioning element.

**When to use `<section>`:**
- When you need a semantic container for a themed block
- Chapters, tabbed content panels, numbered sections
- When there's no more specific element (like `<article>`, `<aside>`, `<nav>`)

**Rule of thumb:** If the content doesn't have a natural heading, use `<div>` instead.

```html
<article>
  <h1>Guide to CSS Grid</h1>
  
  <section>
    <h2>Grid Container Properties</h2>
    <p>...</p>
  </section>
  
  <section>
    <h2>Grid Item Properties</h2>
    <p>...</p>
  </section>
</article>
```

**❌ Don't use `<section>` as a styling wrapper:**
```html
<!-- Bad: just for styling -->
<section class="wrapper">
  <div class="content">...</div>
</section>

<!-- Good: use div for styling -->
<div class="wrapper">
  <div class="content">...</div>
</div>
```

---

### `<aside>`

**Purpose:** Content **tangentially related** to the content around it. Can be considered separate.

**Common uses:**
- Sidebars
- Pull quotes
- Advertisements
- Related links
- "Did you know?" boxes

```html
<article>
  <h1>The History of JavaScript</h1>
  <p>JavaScript was created in 10 days...</p>
  
  <aside>
    <h3>Fun Fact</h3>
    <p>JavaScript was originally named Mocha, then LiveScript.</p>
  </aside>
  
  <p>Continuing the history...</p>
</article>
```

---

### `<footer>`

**Purpose:** Footer for its nearest sectioning content or root. Contains metadata about the section.

**Typical content:**
- Author information
- Copyright notice
- Related links
- Contact information

```html
<!-- Site footer -->
<footer>
  <p>&copy; 2025 Company Name</p>
  <address>
    Contact us at <a href="mailto:info@example.com">info@example.com</a>
  </address>
</footer>

<!-- Article footer -->
<article>
  <h2>Post Title</h2>
  <p>Post content...</p>
  
  <footer>
    <p>Author: Jane Smith</p>
    <p>Last updated: <time datetime="2025-05-26">May 26, 2025</time></p>
  </footer>
</article>
```

---

## Document Outline & Heading Hierarchy

### The Importance of Heading Hierarchy

Headings (`<h1>`–`<h6>`) create a **navigable outline** for your page. Screen readers use this to generate a page structure, and users can jump between sections.

**Rules:**
1. **One `<h1>` per page** - the main page title
2. **Don't skip levels** - `<h1>` → `<h2>` → `<h3>`, not `<h1>` → `<h3>`
3. **Nest logically** - headings represent hierarchy, not font size

```html
<!-- ✅ Good hierarchy -->
<h1>Complete Guide to CSS</h1>

  <h2>Selectors</h2>
    <h3>Type Selectors</h3>
    <h3>Class Selectors</h3>
    
  <h2>Box Model</h2>
    <h3>Margin</h3>
    <h3>Padding</h3>

<!-- ❌ Bad hierarchy -->
<h1>Complete Guide to CSS</h1>
  <h3>Selectors</h3>  <!-- Skipped h2! -->
  <h2>Box Model</h2>
```

### Visual Hierarchy vs Semantic Hierarchy

**Problem:** "I want `<h3>` styling but it should be an `<h2>` semantically."

**Solution:** Separate semantics from styling.

```html
<style>
  .h1-style { font-size: 2.5rem; font-weight: bold; }
  .h2-style { font-size: 2rem; font-weight: bold; }
  .h3-style { font-size: 1.5rem; font-weight: 600; }
</style>

<!-- Correct semantic level, styled as needed -->
<h2 class="h3-style">This is an h2, styled smaller</h2>
```

---

## Accessibility Tree

Semantic HTML builds the **accessibility tree**, which assistive technologies use to navigate your page.

**The tree is generated from:**
- Semantic elements (`<nav>`, `<main>`, etc.)
- Heading hierarchy
- ARIA landmarks (when needed)

**Browser DevTools View:**
In Chrome/Edge DevTools → Elements → Accessibility pane, you can see how your page is exposed to screen readers.

**Example of a well-structured tree:**
```
document
├─ banner (header)
│  └─ navigation (nav)
├─ main (main)
│  ├─ article
│  │  ├─ heading (h1)
│  │  └─ region (section)
│  │     └─ heading (h2)
│  └─ complementary (aside)
└─ contentinfo (footer)
```

---

## Semantic Elements Decision Tree

```
Need a container?
│
├─ Is it the main content area? → <main>
│
├─ Is it major navigation? → <nav>
│
├─ Is it self-contained & reusable? → <article>
│
├─ Is it tangentially related (sidebar, callout)? → <aside>
│
├─ Is it a thematic section with a heading? → <section>
│
├─ Is it intro content or metadata? 
│  ├─ At the top? → <header>
│  └─ At the bottom? → <footer>
│
└─ Just need styling/grouping? → <div>
```

---

## Complete Example: Blog Post Page

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Understanding Closures - Tech Blog</title>
</head>
<body>
  <!-- Site header with global navigation -->
  <header>
    <img src="logo.svg" alt="Tech Blog">
    <nav aria-label="Main">
      <ul>
        <li><a href="/">Home</a></li>
        <li><a href="/articles">Articles</a></li>
        <li><a href="/about">About</a></li>
      </ul>
    </nav>
  </header>

  <!-- Main content area -->
  <main>
    <!-- Breadcrumbs -->
    <nav aria-label="Breadcrumb">
      <ol>
        <li><a href="/">Home</a></li>
        <li><a href="/articles">Articles</a></li>
        <li aria-current="page">Understanding Closures</li>
      </ol>
    </nav>

    <!-- The blog post (self-contained) -->
    <article>
      <header>
        <h1>Understanding Closures in JavaScript</h1>
        <p>
          By <a href="/author/jane">Jane Doe</a> • 
          <time datetime="2025-05-26">May 26, 2025</time>
        </p>
      </header>

      <p>Closures are one of the most powerful features...</p>

      <section>
        <h2>What is a Closure?</h2>
        <p>A closure is the combination of a function...</p>
      </section>

      <section>
        <h2>Common Use Cases</h2>
        <p>Closures are commonly used for...</p>
        
        <aside>
          <h3>💡 Pro Tip</h3>
          <p>Closures can lead to memory leaks if not handled properly.</p>
        </aside>
      </section>

      <footer>
        <p>Tags: 
          <a href="/tag/javascript">JavaScript</a>, 
          <a href="/tag/closures">Closures</a>
        </p>
      </footer>
    </article>

    <!-- Comments section -->
    <section>
      <h2>Comments</h2>
      
      <article>
        <header>
          <h3>Great explanation!</h3>
          <p>By <a href="/user/bob">Bob Smith</a> • 
            <time datetime="2025-05-27">May 27, 2025</time>
          </p>
        </header>
        <p>This really helped me understand closures...</p>
      </article>
    </section>
  </main>

  <!-- Sidebar -->
  <aside aria-label="Related Articles">
    <h2>Related Articles</h2>
    <ul>
      <li><a href="/hoisting">JavaScript Hoisting</a></li>
      <li><a href="/scope">Scope in JavaScript</a></li>
    </ul>
  </aside>

  <!-- Site footer -->
  <footer>
    <p>&copy; 2025 Tech Blog. All rights reserved.</p>
    <address>
      Contact: <a href="mailto:hello@techblog.com">hello@techblog.com</a>
    </address>
  </footer>
</body>
</html>
```

---

## Key Takeaways

✅ **Use semantic elements for structure, `<div>` for styling**

✅ **Every `<section>` and `<article>` should have a heading**

✅ **One `<main>` per page, but multiple `<header>`, `<footer>`, `<article>`, `<section>`, `<aside>`, `<nav>` are allowed**

✅ **Don't skip heading levels** - screen readers rely on hierarchy

✅ **When in doubt, check:** "Does this element add meaning, or just style?"

✅ **ARIA landmarks are implicit** - `<nav>` already has `role="navigation"`, no need to add it

---

## Common Pitfalls

❌ **Using `<section>` as a generic wrapper**
```html
<!-- Bad -->
<section class="container">
  <section class="row">
    <section class="col">...</section>
  </section>
</section>

<!-- Good -->
<div class="container">
  <div class="row">
    <div class="col">...</div>
  </div>
</div>
```

❌ **Multiple `<main>` elements**
```html
<!-- Bad: only one <main> allowed (unless others are hidden) -->
<main>Page 1</main>
<main>Page 2</main>
```

❌ **Skipping heading levels**
```html
<!-- Bad -->
<h1>Title</h1>
<h4>Subtitle</h4>  <!-- Skipped h2 and h3 -->

<!-- Good -->
<h1>Title</h1>
<h2>Subtitle</h2>
```

❌ **Using headings for font size**
```html
<!-- Bad: using h5 just because it's smaller -->
<h1>Page Title</h1>
<h5>A small note</h5>  <!-- Should be <p class="small"> -->
```

---

## Resources for Further Learning

- [MDN: HTML elements reference](https://developer.mozilla.org/en-US/docs/Web/HTML/Element)
- [HTML Living Standard: Sections](https://html.spec.whatwg.org/multipage/sections.html)
- [WCAG 2.2: Info and Relationships](https://www.w3.org/WAI/WCAG22/Understanding/info-and-relationships.html)
- [WebAIM: Semantic Structure](https://webaim.org/techniques/semanticstructure/)

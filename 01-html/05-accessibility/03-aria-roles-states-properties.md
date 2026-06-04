# ARIA roles, states, and properties

## Overview

ARIA (Accessible Rich Internet Applications) provides semantic meaning to elements for assistive technologies when native HTML semantics are insufficient.

## The golden rule of ARIA

**First rule: Don't use ARIA if native HTML works.**

```html
<!-- Bad: unnecessary ARIA -->
<div role="button" tabindex="0" onclick="submit()">Submit</div>

<!-- Good: native element -->
<button onclick="submit()">Submit</button>
```

## ARIA roles

Roles define what an element is or does.

### Landmark roles

#### Main

```html
<main role="main">
  <!-- Primary content -->
</main>

<!-- Better: no role needed with <main> -->
<main>
  <!-- Primary content -->
</main>
```

#### Navigation

```html
<nav role="navigation">
  <!-- Navigation links -->
</nav>

<!-- Better: no role needed -->
<nav aria-label="Main navigation">
  <!-- Navigation links -->
</nav>
```

#### Banner (header)

```html
<header role="banner">
  <!-- Site header -->
</header>

<!-- Role implicit on top-level <header> -->
<header>
  <!-- Site header -->
</header>
```

#### Contentinfo (footer)

```html
<footer role="contentinfo">
  <!-- Site footer -->
</footer>

<!-- Role implicit on top-level <footer> -->
<footer>
  <!-- Site footer -->
</footer>
```

#### Complementary (aside)

```html
<aside role="complementary">
  <!-- Sidebar content -->
</aside>

<!-- Better: no role needed -->
<aside aria-label="Related articles">
  <!-- Sidebar content -->
</aside>
```

#### Search

```html
<div role="search">
  <form>
    <input type="search" aria-label="Search">
    <button>Search</button>
  </form>
</div>

<!-- Better with HTML5 -->
<search>
  <form>
    <input type="search" aria-label="Search">
    <button>Search</button>
  </form>
</search>
```

#### Region

```html
<section role="region" aria-labelledby="section-title">
  <h2 id="section-title">Section Title</h2>
  <!-- Content -->
</section>

<!-- Better: role implicit if labeled -->
<section aria-labelledby="section-title">
  <h2 id="section-title">Section Title</h2>
  <!-- Content -->
</section>
```

### Widget roles

#### Button

```html
<!-- Bad: div as button -->
<div role="button" tabindex="0" onclick="handleClick()">
  Click me
</div>

<!-- Good: native button -->
<button onclick="handleClick()">Click me</button>
```

#### Checkbox

```html
<!-- Custom checkbox -->
<div 
  role="checkbox" 
  aria-checked="false"
  tabindex="0">
  Custom checkbox
</div>

<!-- Better: native -->
<input type="checkbox" id="check">
<label for="check">Native checkbox</label>
```

#### Radio

```html
<!-- Custom radio group -->
<div role="radiogroup" aria-labelledby="group-label">
  <h3 id="group-label">Choose option</h3>
  <div role="radio" aria-checked="true" tabindex="0">Option 1</div>
  <div role="radio" aria-checked="false" tabindex="-1">Option 2</div>
  <div role="radio" aria-checked="false" tabindex="-1">Option 3</div>
</div>

<!-- Better: native -->
<fieldset>
  <legend>Choose option</legend>
  <input type="radio" name="option" id="opt1" checked>
  <label for="opt1">Option 1</label>
  <input type="radio" name="option" id="opt2">
  <label for="opt2">Option 2</label>
  <input type="radio" name="option" id="opt3">
  <label for="opt3">Option 3</label>
</fieldset>
```

#### Slider

```html
<div 
  role="slider"
  aria-valuemin="0"
  aria-valuemax="100"
  aria-valuenow="50"
  aria-label="Volume"
  tabindex="0">
  <div class="slider-track">
    <div class="slider-thumb"></div>
  </div>
</div>

<!-- Better: native (when possible) -->
<input type="range" min="0" max="100" value="50" aria-label="Volume">
```

#### Tab, tablist, tabpanel

```html
<div class="tabs">
  <div role="tablist" aria-label="Content tabs">
    <button role="tab" aria-selected="true" aria-controls="panel1" id="tab1">
      Tab 1
    </button>
    <button role="tab" aria-selected="false" aria-controls="panel2" id="tab2" tabindex="-1">
      Tab 2
    </button>
    <button role="tab" aria-selected="false" aria-controls="panel3" id="tab3" tabindex="-1">
      Tab 3
    </button>
  </div>
  
  <div role="tabpanel" id="panel1" aria-labelledby="tab1">
    Content 1
  </div>
  <div role="tabpanel" id="panel2" aria-labelledby="tab2" hidden>
    Content 2
  </div>
  <div role="tabpanel" id="panel3" aria-labelledby="tab3" hidden>
    Content 3
  </div>
</div>
```

#### Menu, menubar, menuitem

```html
<nav>
  <ul role="menubar">
    <li role="none">
      <button role="menuitem" aria-haspopup="true" aria-expanded="false">
        File
      </button>
      <ul role="menu">
        <li role="none">
          <button role="menuitem">New</button>
        </li>
        <li role="none">
          <button role="menuitem">Open</button>
        </li>
        <li role="none">
          <button role="menuitem">Save</button>
        </li>
      </ul>
    </li>
  </ul>
</nav>
```

### Document structure roles

#### List, listitem

```html
<!-- Bad: custom list -->
<div role="list">
  <div role="listitem">Item 1</div>
  <div role="listitem">Item 2</div>
  <div role="listitem">Item 3</div>
</div>

<!-- Good: native -->
<ul>
  <li>Item 1</li>
  <li>Item 2</li>
  <li>Item 3</li>
</ul>
```

#### Table, row, cell

```html
<!-- Bad: custom table -->
<div role="table">
  <div role="row">
    <div role="columnheader">Name</div>
    <div role="columnheader">Email</div>
  </div>
  <div role="row">
    <div role="cell">John</div>
    <div role="cell">john@example.com</div>
  </div>
</div>

<!-- Good: native -->
<table>
  <thead>
    <tr>
      <th>Name</th>
      <th>Email</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>John</td>
      <td>john@example.com</td>
    </tr>
  </tbody>
</table>
```

#### Article, heading

```html
<!-- Native HTML preferred -->
<article>
  <h1>Article Title</h1>
  <p>Content...</p>
</article>

<!-- ARIA when needed -->
<div role="article">
  <div role="heading" aria-level="1">Article Title</div>
  <p>Content...</p>
</div>
```

## ARIA states

States describe the current condition of an element.

### aria-checked

```html
<button role="checkbox" aria-checked="false">
  Unchecked
</button>

<button role="checkbox" aria-checked="true">
  Checked
</button>

<button role="checkbox" aria-checked="mixed">
  Partially checked
</button>
```

### aria-selected

```html
<div role="tablist">
  <button role="tab" aria-selected="true">Active Tab</button>
  <button role="tab" aria-selected="false">Inactive Tab</button>
</div>
```

### aria-expanded

```html
<!-- Collapsed -->
<button aria-expanded="false" aria-controls="content">
  Show more
</button>
<div id="content" hidden>
  Additional content
</div>

<!-- Expanded -->
<button aria-expanded="true" aria-controls="content">
  Show less
</button>
<div id="content">
  Additional content
</div>
```

### aria-pressed

```html
<!-- Toggle button -->
<button aria-pressed="false">Not pressed</button>
<button aria-pressed="true">Pressed</button>
<button aria-pressed="mixed">Mixed state</button>
```

### aria-hidden

```html
<!-- Hide from screen readers -->
<div aria-hidden="true">
  <svg><!-- Decorative icon --></svg>
</div>

<!-- Visible to screen readers -->
<div aria-hidden="false">
  Content
</div>
```

### aria-disabled

```html
<button aria-disabled="true" disabled>
  Disabled button
</button>

<!-- Custom disabled state -->
<div role="button" aria-disabled="true" class="disabled">
  Disabled
</div>
```

### aria-invalid

```html
<input 
  type="email" 
  aria-invalid="false"
  aria-describedby="email-error">

<input 
  type="email" 
  aria-invalid="true"
  aria-describedby="email-error">
<span id="email-error">Invalid email format</span>
```

## ARIA properties

Properties describe characteristics and relationships.

### aria-label

```html
<!-- Provides accessible name -->
<button aria-label="Close dialog">
  <svg><!-- X icon --></svg>
</button>

<nav aria-label="Main navigation">
  <!-- Links -->
</nav>
```

### aria-labelledby

```html
<!-- Label from another element -->
<div role="dialog" aria-labelledby="dialog-title">
  <h2 id="dialog-title">Confirm Action</h2>
  <p>Are you sure?</p>
  <button>Yes</button>
  <button>No</button>
</div>
```

### aria-describedby

```html
<!-- Additional description -->
<input 
  type="password" 
  aria-describedby="password-requirements">
<div id="password-requirements">
  Password must be at least 8 characters
</div>

<!-- Error message -->
<input 
  type="email"
  aria-invalid="true" 
  aria-describedby="email-error">
<span id="email-error" role="alert">
  Please enter a valid email
</span>
```

### aria-required

```html
<input type="text" aria-required="true" required>

<div role="textbox" aria-required="true" contenteditable>
  <!-- Custom input -->
</div>
```

### aria-haspopup

```html
<button aria-haspopup="true" aria-expanded="false">
  Menu
</button>

<button aria-haspopup="menu" aria-expanded="false">
  File
</button>

<button aria-haspopup="dialog" aria-expanded="false">
  Settings
</button>
```

### aria-controls

```html
<button aria-controls="content" aria-expanded="false">
  Toggle content
</button>
<div id="content" hidden>
  Controlled content
</div>
```

### aria-live

```html
<!-- Polite: wait for pause -->
<div aria-live="polite">
  Status updates
</div>

<!-- Assertive: interrupt -->
<div aria-live="assertive" role="alert">
  Error: Action failed
</div>

<!-- Off: no announcements -->
<div aria-live="off">
  No announcements
</div>
```

### aria-atomic

```html
<!-- Announce entire region -->
<div aria-live="polite" aria-atomic="true">
  <span>Items:</span>
  <span id="count">5</span>
</div>

<!-- Announce only changed part -->
<div aria-live="polite" aria-atomic="false">
  <span>Items:</span>
  <span id="count">5</span>
</div>
```

### aria-relevant

```html
<div aria-live="polite" aria-relevant="additions">
  <!-- Only announce additions -->
</div>

<div aria-live="polite" aria-relevant="removals">
  <!-- Only announce removals -->
</div>

<div aria-live="polite" aria-relevant="additions text">
  <!-- Announce additions and text changes -->
</div>

<div aria-live="polite" aria-relevant="all">
  <!-- Announce all changes -->
</div>
```

### aria-valuemin, aria-valuemax, aria-valuenow

```html
<div 
  role="slider"
  aria-valuemin="0"
  aria-valuemax="100"
  aria-valuenow="50"
  aria-label="Volume">
</div>

<div 
  role="progressbar"
  aria-valuemin="0"
  aria-valuemax="100"
  aria-valuenow="75"
  aria-label="Upload progress">
</div>
```

### aria-valuetext

```html
<div 
  role="slider"
  aria-valuemin="1"
  aria-valuemax="5"
  aria-valuenow="3"
  aria-valuetext="Medium"
  aria-label="Priority">
</div>
```

## Complete examples

### Accordion

```html
<div class="accordion">
  <h3>
    <button 
      id="accordion1" 
      aria-expanded="false" 
      aria-controls="panel1">
      Section 1
    </button>
  </h3>
  <div id="panel1" role="region" aria-labelledby="accordion1" hidden>
    <p>Content for section 1</p>
  </div>
  
  <h3>
    <button 
      id="accordion2" 
      aria-expanded="false" 
      aria-controls="panel2">
      Section 2
    </button>
  </h3>
  <div id="panel2" role="region" aria-labelledby="accordion2" hidden>
    <p>Content for section 2</p>
  </div>
</div>
```

### Modal dialog

```html
<div 
  role="dialog" 
  aria-modal="true"
  aria-labelledby="dialog-title"
  aria-describedby="dialog-desc">
  
  <h2 id="dialog-title">Confirm Delete</h2>
  <p id="dialog-desc">
    Are you sure you want to delete this item? This action cannot be undone.
  </p>
  
  <button>Cancel</button>
  <button>Delete</button>
</div>
```

### Form with validation

```html
<form>
  <label for="email">Email*</label>
  <input 
    type="email" 
    id="email"
    aria-required="true"
    aria-invalid="false"
    aria-describedby="email-desc email-error">
  <span id="email-desc">We'll never share your email</span>
  <span id="email-error" role="alert" hidden>
    Please enter a valid email address
  </span>
  
  <button type="submit">Submit</button>
</form>
```

## Best practices

1. Use native HTML when possible
2. Don't override native semantics
3. All interactive ARIA controls must be keyboard accessible
4. Don't use `role="presentation"` or `aria-hidden="true"` on focusable elements
5. All interactive elements need accessible names
6. Use `aria-labelledby` over `aria-label` when visible label exists
7. Keep ARIA states synchronized with visual state
8. Test with screen readers

## Common mistakes

```html
<!-- Bad: overriding button semantics -->
<button role="heading">Not a button anymore</button>

<!-- Bad: hiding focusable element -->
<button aria-hidden="true">Can't be announced</button>

<!-- Bad: no accessible name -->
<button><span aria-hidden="true">×</span></button>

<!-- Good: proper accessible name -->
<button aria-label="Close"><span aria-hidden="true">×</span></button>
```

## Key takeaways

- First rule: Use native HTML when possible
- Roles define what an element is
- States describe current condition (can change)
- Properties describe characteristics (usually static)
- Keep ARIA states synced with visual state
- All interactive ARIA elements need keyboard support
- All elements need accessible names
- Test with actual screen readers
- ARIA doesn't add functionality, only semantics

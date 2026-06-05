# CSS Pseudo-elements and Pseudo-classes

## The Idea

**In plain English:** CSS pseudo-elements and pseudo-classes are special keywords you add to a selector that let you style an element based on its current condition (like when someone hovers over it) or target a specific invisible part of it (like adding decorative content before or after its text) — without changing your HTML at all.

**Real-world analogy:** Think of a library book. The book itself is always there, but depending on the situation it has different labels and states. When someone checks it out, a "Currently Borrowed" sticker appears on the cover. The sticker on the front is decoration added before the title, and a due-date slip is tucked in at the back.

- The book = the HTML element
- The "Currently Borrowed" sticker = a pseudo-class (`:checked`, `:hover`) — styling that appears based on the book's current state
- The sticker on the front cover = `::before` — virtual content inserted before the element's own content
- The due-date slip at the back = `::after` — virtual content inserted after the element's own content

---

## Pseudo-elements (::)

Pseudo-elements style specific parts of an element or create virtual elements.

### ::before and ::after

Create virtual elements before/after element content.

```css
/* Basic usage */
.element::before {
  content: "→ ";
}

.element::after {
  content: " ←";
}

/* content property is required */
.element::before {
  content: ""; /* Empty but required */
  display: block;
  width: 100px;
  height: 100px;
  background: red;
}

/* Content types */
.quotes::before {
  content: """; /* String */
}

.attr::after {
  content: attr(data-info); /* Attribute value */
}

.counter::before {
  content: counter(section); /* Counter value */
}

.url::before {
  content: url('icon.svg'); /* Image */
}

.open-quote::before {
  content: open-quote; /* Contextual quote */
}

.close-quote::after {
  content: close-quote;
}

/* Multiple values */
.multi::before {
  content: "Chapter " counter(chapter) ": ";
}

/* Decorative elements */
.decorated {
  position: relative;
}

.decorated::before,
.decorated::after {
  content: "";
  position: absolute;
  width: 50px;
  height: 2px;
  background: #333;
}

.decorated::before {
  left: 0;
  top: 0;
}

.decorated::after {
  right: 0;
  bottom: 0;
}

/* Icon before text */
.icon-link::before {
  content: "🔗 ";
}

/* Badge */
.notification {
  position: relative;
}

.notification::after {
  content: "3";
  position: absolute;
  top: -8px;
  right: -8px;
  background: red;
  color: white;
  border-radius: 50%;
  width: 20px;
  height: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 12px;
}

/* Clearfix */
.clearfix::after {
  content: "";
  display: table;
  clear: both;
}

/* Underline effect */
.link {
  position: relative;
  text-decoration: none;
}

.link::after {
  content: "";
  position: absolute;
  bottom: 0;
  left: 0;
  width: 0;
  height: 2px;
  background: currentColor;
  transition: width 0.3s;
}

.link:hover::after {
  width: 100%;
}

/* Quote marks */
blockquote {
  position: relative;
  padding: 20px;
}

blockquote::before {
  content: """;
  font-size: 60px;
  position: absolute;
  top: -10px;
  left: 10px;
  color: #ccc;
}

/* Tooltip */
.tooltip {
  position: relative;
}

.tooltip::after {
  content: attr(data-tooltip);
  position: absolute;
  bottom: 100%;
  left: 50%;
  transform: translateX(-50%);
  background: #333;
  color: white;
  padding: 5px 10px;
  border-radius: 4px;
  white-space: nowrap;
  opacity: 0;
  pointer-events: none;
  transition: opacity 0.3s;
}

.tooltip:hover::after {
  opacity: 1;
}

/* Triangle arrow */
.tooltip::before {
  content: "";
  position: absolute;
  bottom: 100%;
  left: 50%;
  transform: translateX(-50%);
  border: 5px solid transparent;
  border-top-color: #333;
  opacity: 0;
  transition: opacity 0.3s;
}

.tooltip:hover::before {
  opacity: 1;
}
```

### ::first-letter

Style the first letter of a block element.

```css
/* Drop cap */
p::first-letter {
  font-size: 3em;
  font-weight: bold;
  float: left;
  line-height: 1;
  margin-right: 0.1em;
}

/* Styled first letter */
.fancy::first-letter {
  font-size: 2em;
  color: #c00;
  font-family: Georgia, serif;
}

/* Only works on block elements */
span::first-letter {
  font-size: 2em; /* No effect - span is inline */
}

div::first-letter {
  font-size: 2em; /* Works - div is block */
}

/* Limited properties apply */
p::first-letter {
  /* These work */
  font-properties
  color
  background-properties
  margin-properties
  padding-properties
  border-properties
  text-decoration
  text-transform
  line-height
  float
  
  /* These don't work */
  position: absolute; /* No effect */
}

/* Magazine-style first letter */
article p:first-of-type::first-letter {
  font-size: 4em;
  font-weight: bold;
  line-height: 0.9;
  float: left;
  margin: 0.1em 0.1em 0 0;
  color: #c00;
  font-family: Georgia, serif;
}
```

### ::first-line

Style the first line of a block element.

```css
/* Emphasis on first line */
p::first-line {
  font-weight: bold;
  font-size: 1.2em;
}

/* Limited properties apply */
p::first-line {
  /* These work */
  font-properties
  color
  background-properties
  text-decoration
  text-transform
  line-height
  letter-spacing
  word-spacing
  
  /* These don't work */
  margin
  padding
  border
  width
  height
}

/* Newspaper style */
article p:first-of-type::first-line {
  font-weight: 600;
  letter-spacing: 0.05em;
  text-transform: uppercase;
}

/* Dynamic - adjusts with window resize */
.intro::first-line {
  color: blue;
  /* Applies to whatever text is on first line */
}
```

### ::selection

Style text when selected by user.

```css
/* Custom selection color */
::selection {
  background: #ff0;
  color: #000;
}

/* Vendor prefix for Firefox */
::-moz-selection {
  background: #ff0;
  color: #000;
}

/* Per-element selection */
.special::selection {
  background: #00f;
  color: #fff;
}

/* Limited properties */
::selection {
  /* These work */
  color
  background-color
  text-decoration
  text-shadow
  
  /* These don't work */
  font-size
  margin
  padding
  border
}

/* Brand-themed selection */
::selection {
  background: #667eea;
  color: white;
  text-shadow: none;
}

/* Code selection */
code::selection {
  background: rgba(0, 0, 0, 0.1);
}

/* No background */
.no-select::selection {
  background: transparent;
}
```

### ::marker

Style list item markers.

```css
/* Style bullet points */
li::marker {
  color: red;
  font-size: 1.5em;
}

/* Custom marker content */
li::marker {
  content: "→ ";
}

/* Emoji markers */
ul li::marker {
  content: "✓ ";
}

ol li::marker {
  content: counter(list-item) ". ";
  font-weight: bold;
  color: #667eea;
}

/* Limited properties */
::marker {
  /* These work */
  font-properties
  color
  content
  text-combine-upright
  unicode-bidi
  direction
  
  /* These don't work */
  background
  margin
  padding
  border
}

/* Styled ordered list */
ol {
  list-style-position: outside;
}

ol li::marker {
  content: counter(list-item) " → ";
  color: #764ba2;
  font-weight: 600;
}

/* Summary marker (disclosure triangle) */
summary::marker {
  content: "▶ ";
}

details[open] summary::marker {
  content: "▼ ";
}
```

### ::placeholder

Style placeholder text in inputs.

```css
/* Basic placeholder styling */
input::placeholder {
  color: #999;
  opacity: 1; /* Firefox default is 0.54 */
}

/* Vendor prefixes for older browsers */
input::-webkit-input-placeholder { /* Chrome/Safari/Opera */
  color: #999;
}

input:-moz-placeholder { /* Firefox 18- */
  color: #999;
}

input::-moz-placeholder { /* Firefox 19+ */
  color: #999;
}

input:-ms-input-placeholder { /* IE 10+ */
  color: #999;
}

/* Limited properties */
::placeholder {
  /* These work */
  font-properties
  color
  background
  text-transform
  letter-spacing
  word-spacing
  text-decoration
  
  /* These don't work */
  width
  height
  margin
  padding (in some browsers)
}

/* Focus state */
input:focus::placeholder {
  color: transparent;
}

/* Italic placeholder */
textarea::placeholder {
  font-style: italic;
  color: #aaa;
}

/* Animated placeholder */
input::placeholder {
  transition: opacity 0.3s;
}

input:focus::placeholder {
  opacity: 0.5;
}
```

### Other Pseudo-elements

```css
/* ::backdrop - fullscreen/dialog backdrop */
dialog::backdrop {
  background: rgba(0, 0, 0, 0.75);
  backdrop-filter: blur(5px);
}

video:fullscreen::backdrop {
  background: black;
}

/* ::cue - WebVTT captions/subtitles */
::cue {
  color: yellow;
  background: rgba(0, 0, 0, 0.8);
}

/* ::file-selector-button - file input button */
input[type="file"]::file-selector-button {
  background: #667eea;
  color: white;
  border: none;
  padding: 10px 20px;
  border-radius: 4px;
  cursor: pointer;
}

input[type="file"]::file-selector-button:hover {
  background: #764ba2;
}

/* ::grammar-error - grammar mistakes (experimental) */
::grammar-error {
  text-decoration: blue wavy underline;
}

/* ::spelling-error - spelling mistakes (experimental) */
::spelling-error {
  text-decoration: red wavy underline;
}
```

## Pseudo-classes (:)

Pseudo-classes select elements based on state or position.

### Interactive States

```css
/* :hover - pointer over element */
button:hover {
  background: #667eea;
}

/* Works on any element */
div:hover {
  transform: scale(1.05);
}

/* :active - being activated (clicked) */
button:active {
  transform: scale(0.95);
}

/* :focus - has keyboard focus */
input:focus {
  outline: 2px solid #667eea;
  outline-offset: 2px;
}

/* :focus-visible - has focus via keyboard (not mouse) */
button:focus-visible {
  outline: 2px solid #667eea;
}

button:focus:not(:focus-visible) {
  outline: none; /* Remove outline for mouse clicks */
}

/* :focus-within - element or descendant has focus */
form:focus-within {
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.2);
}

/* :visited - visited link */
a:visited {
  color: purple;
}

/* :link - unvisited link */
a:link {
  color: blue;
}

/* :target - element targeted by URL fragment */
:target {
  background: yellow;
}

/* Example: #section1 in URL */
section:target {
  animation: highlight 1s;
}

@keyframes highlight {
  from { background: yellow; }
  to { background: transparent; }
}

/* :enabled/:disabled */
input:enabled {
  background: white;
}

input:disabled {
  background: #f0f0f0;
  cursor: not-allowed;
}

/* :checked - checked checkbox/radio */
input[type="checkbox"]:checked {
  accent-color: #667eea;
}

/* Custom checkbox */
input[type="checkbox"]:checked + label {
  color: #667eea;
  font-weight: bold;
}

/* :indeterminate - indeterminate checkbox */
input[type="checkbox"]:indeterminate {
  accent-color: #ccc;
}

/* :valid/:invalid - form validation */
input:valid {
  border-color: green;
}

input:invalid {
  border-color: red;
}

/* Only show invalid after interaction */
input:invalid:not(:focus):not(:placeholder-shown) {
  border-color: red;
}

/* :required/:optional */
input:required {
  border-left: 3px solid #f00;
}

input:optional {
  border-left: 3px solid #0f0;
}

/* :read-only/:read-write */
input:read-only {
  background: #f0f0f0;
}

input:read-write {
  background: white;
}

/* :in-range/:out-of-range */
input[type="number"]:in-range {
  border-color: green;
}

input[type="number"]:out-of-range {
  border-color: red;
}

/* :placeholder-shown - has visible placeholder */
input:placeholder-shown {
  border-color: #ccc;
}

input:not(:placeholder-shown) {
  border-color: #667eea;
}
```

### Structural Pseudo-classes

```css
/* :first-child - first child of parent */
li:first-child {
  font-weight: bold;
}

/* :last-child - last child of parent */
li:last-child {
  border-bottom: none;
}

/* :only-child - only child of parent */
li:only-child {
  list-style: none;
}

/* :first-of-type - first of its element type */
p:first-of-type {
  font-size: 1.2em;
}

/* :last-of-type - last of its element type */
article p:last-of-type {
  margin-bottom: 0;
}

/* :only-of-type - only one of its type */
img:only-of-type {
  width: 100%;
}

/* :nth-child(n) - nth child */
li:nth-child(1) { /* First item */ }
li:nth-child(2) { /* Second item */ }

/* Keywords */
li:nth-child(odd) { background: #f0f0f0; } /* 1, 3, 5... */
li:nth-child(even) { background: white; } /* 2, 4, 6... */

/* Formulas: an+b */
li:nth-child(2n) { } /* Even: 2, 4, 6... */
li:nth-child(2n+1) { } /* Odd: 1, 3, 5... */
li:nth-child(3n) { } /* Every 3rd: 3, 6, 9... */
li:nth-child(3n+1) { } /* 1, 4, 7, 10... */
li:nth-child(n+3) { } /* 3 and higher */
li:nth-child(-n+3) { } /* First 3 items */

/* :nth-last-child(n) - counting from end */
li:nth-last-child(1) { } /* Last item */
li:nth-last-child(2) { } /* Second to last */
li:nth-last-child(-n+3) { } /* Last 3 items */

/* :nth-of-type(n) - nth of element type */
p:nth-of-type(2) { } /* Second <p> element */
div:nth-of-type(odd) { } /* Odd <div> elements */

/* :nth-last-of-type(n) - counting from end */
p:nth-last-of-type(1) { } /* Last <p> */

/* Zebra striping */
tr:nth-child(odd) {
  background: #f9f9f9;
}

/* First 3 items */
li:nth-child(-n+3) {
  color: red;
}

/* All but first 3 */
li:nth-child(n+4) {
  color: blue;
}

/* Every 3rd starting from 2nd */
li:nth-child(3n+2) {
  font-weight: bold;
}
```

### Logical Pseudo-classes

```css
/* :not() - negation */
p:not(.special) {
  color: gray;
}

button:not(:disabled) {
  cursor: pointer;
}

/* Multiple selectors (Level 4) */
input:not([type="radio"], [type="checkbox"]) {
  width: 100%;
}

/* :is() - matches any */
:is(h1, h2, h3) {
  font-family: Georgia, serif;
}

/* Useful for reducing repetition */
/* Instead of: */
.header h1,
.main h1,
.footer h1 { }

/* Use: */
:is(.header, .main, .footer) h1 { }

/* :where() - like :is() but zero specificity */
:where(h1, h2, h3) {
  margin-top: 0;
}

/* Can be easily overridden */
h2 {
  margin-top: 1em; /* Wins despite coming after */
}

/* :has() - parent selector / relational */
/* Parent with specific child */
article:has(img) {
  display: grid;
}

/* Form with invalid input */
form:has(input:invalid) {
  border-color: red;
}

/* Card with button */
.card:has(button) {
  padding-bottom: 60px;
}

/* Sibling combinator */
h2:has(+ p) {
  margin-bottom: 0.5em;
}

/* Label followed by checked input */
label:has(+ input:checked) {
  color: green;
}

/* Multiple conditions */
.container:has(.error, .warning) {
  border-left: 3px solid red;
}
```

### Other Pseudo-classes

```css
/* :empty - no children (including text) */
div:empty {
  display: none;
}

p:empty::before {
  content: "No content";
  color: #999;
}

/* :root - document root (higher specificity than html) */
:root {
  --primary-color: #667eea;
}

/* :any-link - any <a> with href */
:any-link {
  color: blue;
}

/* :local-link - same-domain link */
:local-link {
  color: green;
}

/* :lang() - language */
p:lang(en) {
  quotes: """ """;
}

p:lang(fr) {
  quotes: "« " " »";
}

/* :dir() - text direction */
:dir(ltr) {
  text-align: left;
}

:dir(rtl) {
  text-align: right;
}

/* :defined - custom element is defined */
my-element:not(:defined) {
  opacity: 0;
}

my-element:defined {
  opacity: 1;
  transition: opacity 0.3s;
}

/* :fullscreen - fullscreen element */
:fullscreen {
  background: black;
}

video:fullscreen {
  width: 100vw;
  height: 100vh;
}

/* :modal - modal dialog */
dialog:modal {
  max-width: 600px;
  margin: auto;
}

/* :picture-in-picture - PiP video */
video:picture-in-picture {
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

/* :playing/:paused - media playback */
video:playing {
  border-color: green;
}

video:paused {
  border-color: gray;
}

/* :autofill - autofilled input */
input:autofill {
  background: #e8f0fe;
}

input:-webkit-autofill {
  -webkit-box-shadow: 0 0 0 1000px #e8f0fe inset;
}
```

## Common Patterns

```css
/* Accessible focus styles */
button {
  outline: none;
}

button:focus-visible {
  outline: 2px solid #667eea;
  outline-offset: 2px;
}

/* Hover lift with shadow */
.card {
  transition: transform 0.2s, box-shadow 0.2s;
}

.card:hover {
  transform: translateY(-4px);
  box-shadow: 0 10px 20px rgba(0, 0, 0, 0.1);
}

/* Button states */
.button {
  background: #667eea;
  transition: all 0.2s;
}

.button:hover {
  background: #5568d3;
}

.button:active {
  transform: scale(0.98);
}

.button:disabled {
  background: #ccc;
  cursor: not-allowed;
}

/* Form validation feedback */
input {
  border: 2px solid #ddd;
  transition: border-color 0.2s;
}

input:focus {
  border-color: #667eea;
  outline: none;
}

input:invalid:not(:focus):not(:placeholder-shown) {
  border-color: #f00;
}

input:valid:not(:placeholder-shown) {
  border-color: #0f0;
}

/* Animated underline */
a {
  position: relative;
  text-decoration: none;
}

a::after {
  content: "";
  position: absolute;
  bottom: 0;
  left: 0;
  width: 0;
  height: 2px;
  background: currentColor;
  transition: width 0.3s;
}

a:hover::after {
  width: 100%;
}

/* Tooltip */
[data-tooltip] {
  position: relative;
}

[data-tooltip]::after {
  content: attr(data-tooltip);
  position: absolute;
  bottom: 100%;
  left: 50%;
  transform: translateX(-50%);
  padding: 5px 10px;
  background: #333;
  color: white;
  border-radius: 4px;
  white-space: nowrap;
  opacity: 0;
  pointer-events: none;
  transition: opacity 0.3s;
}

[data-tooltip]:hover::after {
  opacity: 1;
}

/* Custom checkbox */
input[type="checkbox"] {
  appearance: none;
  width: 20px;
  height: 20px;
  border: 2px solid #ddd;
  border-radius: 4px;
  position: relative;
  cursor: pointer;
}

input[type="checkbox"]:checked {
  background: #667eea;
  border-color: #667eea;
}

input[type="checkbox"]:checked::after {
  content: "✓";
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  color: white;
  font-size: 14px;
}

/* Skip to content link */
.skip-link {
  position: absolute;
  top: -40px;
  left: 0;
  background: #000;
  color: white;
  padding: 8px;
  z-index: 100;
}

.skip-link:focus {
  top: 0;
}

/* Breadcrumb separators */
.breadcrumb li:not(:last-child)::after {
  content: " / ";
  margin: 0 0.5em;
  color: #999;
}

/* Zebra striping with :nth-child */
tbody tr:nth-child(even) {
  background: #f9f9f9;
}

/* Highlight first and last */
li:first-child {
  font-weight: bold;
}

li:last-child {
  border-bottom: 2px solid #667eea;
}

/* Loading skeleton */
.skeleton {
  position: relative;
  background: #f0f0f0;
  overflow: hidden;
}

.skeleton::after {
  content: "";
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: linear-gradient(
    90deg,
    transparent,
    rgba(255, 255, 255, 0.5),
    transparent
  );
  animation: shimmer 2s infinite;
}

@keyframes shimmer {
  from { transform: translateX(-100%); }
  to { transform: translateX(100%); }
}
```

## Gotchas

1. **::before/::after require content**
   ```css
   /* Doesn't render */
   .element::before {
     background: red;
   }
   
   /* Must have content */
   .element::before {
     content: "";
     background: red;
   }
   ```

2. **Single vs double colon**
   ```css
   /* Old syntax (still works) */
   .element:before { }
   
   /* New syntax (preferred) */
   .element::before { }
   
   /* Pseudo-classes use single colon */
   .element:hover { }
   ```

3. **:visited limitations**
   ```css
   /* Security restrictions - limited properties */
   a:visited {
     color: purple; /* Works */
     background: red; /* Doesn't work */
   }
   ```

4. **::first-letter/::first-line limited properties**
   ```css
   p::first-letter {
     margin: 10px; /* Works */
     position: absolute; /* Doesn't work */
   }
   ```

5. **:nth-child vs :nth-of-type**
   ```html
   <div>
     <span>1</span>
     <p>2</p>
     <p>3</p>
   </div>
   ```
   
   ```css
   /* Selects second child (p), regardless of type */
   p:nth-child(2) { color: red; }
   
   /* Selects first <p> element */
   p:nth-of-type(1) { color: blue; }
   ```

6. **::placeholder cross-browser**
   ```css
   /* Need vendor prefixes */
   input::placeholder { }
   input::-webkit-input-placeholder { }
   input:-moz-placeholder { }
   input::-moz-placeholder { }
   input:-ms-input-placeholder { }
   ```

## Interview Questions

**Q: What's the difference between pseudo-elements and pseudo-classes?**

A: 
- Pseudo-elements (::) create virtual elements or style parts of elements. Use double colon. Examples: `::before`, `::after`, `::first-letter`
- Pseudo-classes (:) select elements based on state or position. Use single colon. Examples: `:hover`, `:nth-child`, `:focus`

Pseudo-elements style things, pseudo-classes select things.

**Q: Explain how ::before and ::after work.**

A: Create virtual elements before/after the element's content. Require `content` property (even if empty). Treated as children of the element. Common uses:
- Decorative elements (icons, shapes)
- Tooltips
- Clearfix
- Animated underlines

Cannot be used on replaced elements (img, input, video) since they have no content.

**Q: What's the difference between :nth-child() and :nth-of-type()?**

A: 
- `:nth-child(n)`: Selects nth child regardless of type. `p:nth-child(2)` selects `<p>` only if it's the 2nd child.
- `:nth-of-type(n)`: Selects nth element of that type. `p:nth-of-type(2)` selects the 2nd `<p>` element, regardless of position.

Use :nth-of-type when you don't control sibling elements, :nth-child when you need specific position.

**Q: How does the :not() pseudo-class work?**

A: Negation pseudo-class that excludes matching elements:
```css
/* All paragraphs except .special */
p:not(.special) { }

/* All inputs except disabled */
input:not(:disabled) { }

/* Multiple (Level 4) */
input:not([type="radio"], [type="checkbox"]) { }
```

Cannot nest :not() inside :not(). Increases specificity.

**Q: What's the difference between :is() and :where()?**

A: Both match any selector in a list, but differ in specificity:
- `:is()`: Takes specificity of most specific selector in list
- `:where()`: Always has zero specificity

Example:
```css
:is(#id, .class) { } /* Specificity: #id */
:where(#id, .class) { } /* Specificity: 0 */
```

Use :where() for easily-overridable defaults, :is() for normal styling.

**Q: How does the :has() pseudo-class work?**

A: "Parent selector" that checks if element contains/follows certain descendants/siblings:
```css
/* Article with image */
article:has(img) { }

/* Form with invalid input */
form:has(input:invalid) { }

/* Label followed by checked checkbox */
label:has(+ input:checked) { }
```

Very powerful for conditional styling. Can affect performance if overused.

**Q: Why can't you apply many styles to :visited links?**

A: Security: prevents websites from detecting browsing history. Only color-related properties allowed:
- color
- background-color
- border-color
- outline-color
- column-rule-color

Cannot use background-image, padding, font-size, etc. JavaScript also cannot detect visited state.

**Q: What's the purpose of :focus-visible?**

A: Shows focus indicator only for keyboard navigation, not mouse clicks. Improves UX:
```css
button:focus-visible {
  outline: 2px solid blue;
}

button:focus:not(:focus-visible) {
  outline: none; /* No outline on click */
}
```

Solves the "remove outline on click but keep for keyboard" problem. Better accessibility than removing :focus outline entirely.

**Q: How do you create a custom checkbox using pseudo-elements?**

A: Hide native checkbox, style replacement, use ::after for checkmark:
```css
input[type="checkbox"] {
  appearance: none;
  width: 20px;
  height: 20px;
  border: 2px solid #ddd;
  border-radius: 4px;
  position: relative;
}

input[type="checkbox"]:checked {
  background: blue;
  border-color: blue;
}

input[type="checkbox"]:checked::after {
  content: "✓";
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  color: white;
}
```

Must keep accessible (keyboard navigable, screen reader compatible).
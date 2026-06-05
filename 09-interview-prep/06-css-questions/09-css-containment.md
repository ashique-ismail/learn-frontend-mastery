# CSS Containment

## The Idea

**In plain English:** CSS Containment is a way to tell the browser that a section of your webpage is isolated and independent from the rest, so the browser does not need to recalculate the whole page every time something small changes inside that section. Think of it as drawing a fence around part of your page and telling the browser "changes inside this fence stay inside this fence."

**Real-world analogy:** Imagine a school with many classrooms. When a teacher rearranges desks inside one classroom, the principal does not need to walk through every other classroom to check if anything changed there too — the classroom is self-contained. CSS Containment works the same way:

- The school = the full webpage
- Each classroom = a contained element (e.g., a widget or section with `contain`)
- Rearranging desks = changing layout or content inside the element
- The principal skipping other classrooms = the browser skipping recalculations for the rest of the page

---

## What Is CSS Containment?

The `contain` property tells the browser that an element and its descendants are **independent** from the rest of the document tree. This allows the browser to skip certain calculations for elements outside the container.

Performance optimization for large DOMs — the browser can limit the scope of layout, style, and paint calculations.

---

## `contain` Values

### `layout`

```css
.widget { contain: layout; }
```

The element's layout is independent from the rest of the page:
- Internal layout doesn't affect external layout
- External layout doesn't affect internal layout
- The element establishes an **independent formatting context** (like BFC)
- Floats and margins don't escape
- Creates a new stacking context

Use when: Component whose size changes frequently and you don't want it to trigger a full-page layout reflow.

### `style`

```css
.widget { contain: style; }
```

CSS counters and quotes are scoped to the element. Very limited use case.

### `paint`

```css
.widget { contain: paint; }
```

- Clipping: content that overflows the element is not painted
- Element creates a new stacking context
- Children are clipped to the border box
- Descendants can't paint outside the element

Use when: You know overflow content should never be visible; prevents the browser from checking outside the box.

### `size`

```css
.widget { contain: size; }
/* Must also set explicit dimensions or element collapses to 0 */
.widget { contain: size; width: 200px; height: 100px; }
```

The element's size is not affected by its children. The browser can skip laying out children when determining the element's size.

Use when: You have a fixed-size container and want to prevent children from affecting its dimensions.

### `inline-size`

```css
.widget { contain: inline-size; }
```

Like `size` but only for the inline dimension (typically width in horizontal writing modes).

---

## Shorthand Values

```css
/* contain: layout style = style+layout */
.component { contain: layout style; }

/* contain: content = layout style paint */
.component { contain: content; }

/* contain: strict = all (layout style paint size) */
.component { contain: strict; width: 300px; height: 200px; }
```

---

## `content-visibility: auto`

The most impactful CSS containment property for performance:

```css
.section {
  content-visibility: auto;
  contain-intrinsic-size: 0 500px; /* estimated size for scroll calculation */
}
```

**What it does:**
1. Elements outside the viewport skip rendering entirely (painting and layout)
2. Elements are rendered when they enter the viewport
3. Browser uses `contain-intrinsic-size` for scroll position calculation while element is not rendered

**Performance impact:** On pages with thousands of elements, this can reduce rendering time by 5-10x. The browser skips rendering ~90% of off-screen content.

```css
/* Apply to each major content section on a long page */
article, section, .list-item {
  content-visibility: auto;
  contain-intrinsic-block-size: 300px; /* estimated height */
}
```

---

## `contain-intrinsic-size`

```css
.widget {
  content-visibility: auto;
  contain-intrinsic-size: 200px 150px; /* width height */
  /* or: contain-intrinsic-block-size: 150px; */
}
```

Provides a placeholder size for the element while it's not rendered. Without this, all off-screen elements would collapse to 0, making the scrollbar jump as you scroll.

---

## Common Interview Questions

**Q: What's the difference between `contain: layout` and creating a BFC?**
`contain: layout` creates an independent formatting context but goes further — it also prevents external layout from affecting internal layout (bidirectional isolation). A BFC (`overflow: hidden`) only prevents floats and margin collapse from leaking out.

**Q: When would you use `contain: strict`?**
For fixed-size components like ad slots, video players, or map containers where size is always known and you want maximum isolation from the component's content.

**Q: Does `content-visibility: auto` affect accessibility?**
Hidden content (not yet visible) is still in the DOM and accessible to screen readers. It's only the rendering/painting that's skipped. However, be careful with elements that affect form submission or other DOM-based interactions.

**Q: What's the browser support for `content-visibility`?**
Supported in Chrome 85+, Edge 85+. Firefox support added in v124. Not in Safari (as of 2024). Always add `contain-intrinsic-size` as a fallback for scroll behavior.

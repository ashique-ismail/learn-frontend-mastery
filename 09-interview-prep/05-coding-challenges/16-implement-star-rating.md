# Implement Accessible Star Rating Component

## The Idea

**In plain English:** A star rating component lets users pick a score (like 1 to 5 stars) by clicking or tapping on stars, and the page updates to show which stars are filled in. "Accessible" means it is also usable by people who navigate with a keyboard or who rely on a screen reader (software that reads the screen aloud).

**Real-world analogy:** Think of a printed multiple-choice bubble sheet on a test, where each bubble represents an answer and you fill in exactly one. Now imagine it looks like five gold stars instead of bubbles:

- The bubble sheet = the `<fieldset>` (groups all the choices together)
- Each individual bubble = a hidden `<input type="radio">` (the real selectable control)
- The star symbol printed next to each bubble = the `<span>` with ★ or ☆ (purely decorative, hides from screen readers)
- Filling in a bubble = setting `checked` on the radio input and updating state

---

## Accessible Approach: Radio Inputs

The most accessible star rating uses `<input type="radio">` — keyboard support is built-in, screen readers understand the rating concept, and it works without JavaScript.

```tsx
interface StarRatingProps {
  value: number;           // 0-5
  onChange: (value: number) => void;
  max?: number;
  name?: string;
  readOnly?: boolean;
  label?: string;
}

function StarRating({
  value,
  onChange,
  max = 5,
  name = 'rating',
  readOnly = false,
  label = 'Rating',
}: StarRatingProps) {
  const [hoveredValue, setHoveredValue] = useState<number | null>(null);
  const displayValue = hoveredValue ?? value;

  return (
    <fieldset className="star-rating" disabled={readOnly}>
      <legend className="sr-only">{label}</legend>

      {Array.from({ length: max }, (_, i) => {
        const starValue = i + 1;
        const isFilled = starValue <= displayValue;

        return (
          <label key={starValue} className="star-label">
            <input
              type="radio"
              name={name}
              value={starValue}
              checked={value === starValue}
              onChange={() => onChange(starValue)}
              aria-label={`${starValue} of ${max} stars`}
              className="sr-only"  // visually hidden, keyboard accessible
            />
            <span
              className={`star ${isFilled ? 'star--filled' : 'star--empty'}`}
              aria-hidden="true"  // decorative — the radio input is the real control
              onMouseEnter={() => !readOnly && setHoveredValue(starValue)}
              onMouseLeave={() => !readOnly && setHoveredValue(null)}
            >
              {isFilled ? '★' : '☆'}
            </span>
          </label>
        );
      })}

      {/* Current value announcement for screen readers */}
      <span className="sr-only" aria-live="polite">
        {value > 0 ? `${value} of ${max} stars selected` : 'No rating selected'}
      </span>
    </fieldset>
  );
}
```

---

## CSS

```css
.star-rating {
  border: none;
  padding: 0;
  margin: 0;
  display: inline-flex;
  gap: 4px;
}

/* Visually hidden but accessible */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}

.star-label { cursor: pointer; }
.star { font-size: 32px; line-height: 1; transition: transform 0.1s; }
.star--filled { color: #f59e0b; }
.star--empty { color: #d1d5db; }
.star:hover { transform: scale(1.2); }

fieldset:disabled .star-label { cursor: default; }
fieldset:disabled .star { pointer-events: none; }

/* Focus styling for radio inputs (using :focus-within on label) */
.star-label:has(input:focus-visible) .star {
  outline: 2px solid #3b82f6;
  outline-offset: 2px;
  border-radius: 2px;
}
```

---

## Read-Only Display Variant

```tsx
function StarDisplay({ value, max = 5 }: { value: number; max?: number }) {
  const fullStars = Math.floor(value);
  const hasHalf = value % 1 >= 0.5;

  return (
    <div
      role="img"
      aria-label={`${value} out of ${max} stars`}
      className="star-display"
    >
      {Array.from({ length: max }, (_, i) => {
        const starValue = i + 1;
        if (starValue <= fullStars) return <span key={i} className="star--filled">★</span>;
        if (starValue === fullStars + 1 && hasHalf) return <span key={i} className="star--half">⯨</span>;
        return <span key={i} className="star--empty">☆</span>;
      })}
    </div>
  );
}
```

---

## Form Integration

```tsx
// With React Hook Form
function ReviewForm() {
  const { register, handleSubmit, setValue, watch } = useForm({
    defaultValues: { rating: 0, comment: '' }
  });
  const rating = watch('rating');

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <StarRating
        value={rating}
        onChange={(val) => setValue('rating', val, { shouldValidate: true })}
        label="Your rating"
      />
      <textarea {...register('comment', { required: 'Comment is required' })} />
      <button type="submit">Submit</button>
    </form>
  );
}
```

---

## Common Interview Questions

**Q: Why use radio inputs instead of `<div>` elements with click handlers?**
Radio inputs provide free keyboard navigation (arrow keys between options), built-in form semantics, screen reader support (the group is announced as "rating, 3 of 5"), and work without JavaScript/CSS. `<div>` elements require manually implementing all of this with `tabIndex`, `role`, `aria-checked`, and keyboard events.

**Q: How does screen reader announce the star rating?**
With the radio input approach: "Rating group. 1 star radio button. 2 stars radio button selected. 3 stars radio button." The user knows it's a rating from 1-5 and can select with arrow keys.

**Q: What's the ARIA pattern for a non-interactive star display?**
Use `role="img"` with an `aria-label` like "4.5 out of 5 stars" on the container. The individual stars are presentational and should have `aria-hidden="true"`.

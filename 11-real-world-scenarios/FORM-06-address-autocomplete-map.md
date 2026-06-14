# Address Autocomplete with Map Preview

## The Idea

**In plain English:** The user types a partial address into an input. After a short pause, a dropdown shows matching address suggestions sourced from Google Places Autocomplete. Selecting a suggestion fills in structured form fields (street, city, postcode) and updates a map to show a pin at that location. The Google Maps JavaScript SDK is a third-party script that cannot be bundled — it must be injected into the page dynamically, exactly once, even if multiple components or re-renders request it. On top of that: the autocomplete fetch must be debounced to avoid hammering the API on every keystroke, the form must validate that an address was confirmed (not just typed), and the component must degrade gracefully when the SDK fails to load.

**Real-world analogy:** Think of a hotel concierge desk with a large city map pinned to the wall.

- **Typing into the input** = the guest starting to describe their destination: "I'm looking for somewhere on Park…"
- **Debounce** = the concierge waiting a beat for the guest to finish their sentence before consulting the map book — not diving for the book on every syllable.
- **Autocomplete dropdown** = the concierge saying "I have three options: Park Lane, Park Road, or Park Avenue — which did you mean?"
- **Selecting a suggestion** = the guest choosing "Park Lane" — the concierge writes the full address on a card and sticks a pin in the wall map.
- **Dynamic script loading** = the concierge only unrolling the large map when someone actually needs it. They don't spread it across the desk at the start of every shift.
- **Session tokens** = the concierge billing the guest once for the whole "I'm looking for Park Lane" conversation, not once per question asked during it.
- **Graceful degradation** = if the map book is missing, the concierge still takes the address manually rather than turning the guest away.

The key insight: this is not just a search input. It is a multi-step async pipeline — script load → debounced autocomplete → place details fetch → map render — and every step can fail independently.

---

## Learning Objectives

- Implement the `loadScript` idempotent loader pattern with a module-level Promise cache
- Debounce autocomplete queries and understand when debounce alone is insufficient (use session tokens instead)
- Correctly parse `address_components` from the Places API into structured fields, including locale-aware fallbacks
- Render and update a Google Map imperatively from inside a declarative React or Angular component
- Implement async form validation that rejects a raw typed string in favour of a confirmed Place selection
- Handle every failure mode: SDK load failure, network error on predictions, network error on place details

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Load the Google Maps SDK on demand | No | Requires JS `<script>` injection into `<head>` |
| Debounce the autocomplete fetch | No | Requires JS `setTimeout` / `clearTimeout` |
| Show suggestions from an external API | No | CSS cannot fetch data |
| Parse `address_components` array into fields | No | Requires JS iteration |
| Move a map pin on place selection | No | Requires Google Maps SDK imperative API calls |
| Disable form submission until a place is confirmed | No | Requires JS validation state |
| Show an error when the SDK fails to load | No | Requires JS error handling |

**Conclusion:** CSS handles visual chrome — the suggestion dropdown shape, the spinner animation, the map container sizing. Every substantive behaviour in this feature is driven by JavaScript.

---

## HTML & CSS Foundation

### The Markup

```html
<div class="address-form">

  <!-- Autocomplete combobox -->
  <div class="address-field">
    <label for="address-input" class="field-label">Delivery address</label>
    <div class="address-input-wrapper">
      <input
        id="address-input"
        type="text"
        class="field-input"
        placeholder="Start typing an address…"
        autocomplete="off"
        spellcheck="false"
        role="combobox"
        aria-autocomplete="list"
        aria-controls="address-listbox"
        aria-expanded="false"
        aria-activedescendant=""
      />
      <!-- Spinner shown while fetching predictions or place details -->
      <span class="field-spinner" aria-hidden="true" hidden></span>
    </div>

    <!-- Suggestion listbox -->
    <ul
      id="address-listbox"
      class="suggestions-list"
      role="listbox"
      aria-label="Address suggestions"
      hidden
    >
      <!-- Populated by JS; each <li> has role="option" -->
    </ul>

    <!-- Error: SDK load failure OR no results -->
    <p class="field-error" role="alert" hidden></p>
  </div>

  <!-- Structured address fields (populated programmatically after selection) -->
  <div class="address-structured" hidden>
    <input type="text"   name="street"   class="field-input" aria-label="Street"   readonly />
    <div class="address-row">
      <input type="text" name="city"     class="field-input" aria-label="City"     readonly />
      <input type="text" name="postcode" class="field-input" aria-label="Postcode" readonly />
    </div>
    <!-- Hidden fields carry the machine-readable values for form submission -->
    <input type="hidden" name="lat"     />
    <input type="hidden" name="lng"     />
    <input type="hidden" name="placeId" />
  </div>

  <!-- Map preview -->
  <div
    id="address-map"
    class="address-map"
    role="img"
    aria-label="Map preview — no address selected"
  ></div>

</div>
```

### The CSS

```css
/* ─── Layout ─── */
.address-form {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  max-width: 480px;
}

.address-field {
  position: relative;    /* contains absolutely-positioned listbox */
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.field-label {
  font-size: 0.875rem;
  font-weight: 500;
  color: #374151;
}

/* ─── Input ─── */
.address-input-wrapper { position: relative; }

.field-input {
  width: 100%;
  padding: 0.625rem 0.875rem;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  font-size: 0.9rem;
  line-height: 1.4;
  outline: none;
  transition: border-color 0.15s, box-shadow 0.15s;
  box-sizing: border-box;
}
.field-input:focus {
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.2);
}
.field-input[readonly] {
  background: #f9fafb;
  color: #6b7280;
  cursor: default;
}

/* ─── Spinner ─── */
.field-spinner {
  position: absolute;
  right: 0.75rem;
  top: 50%;
  transform: translateY(-50%);
  width: 16px;
  height: 16px;
  border: 2px solid #e5e7eb;
  border-top-color: #6b7280;
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
}
@keyframes spin { to { transform: translateY(-50%) rotate(360deg); } }

/* ─── Suggestions dropdown ─── */
.suggestions-list {
  position: absolute;
  top: 100%;
  left: 0;
  right: 0;
  z-index: 50;
  background: #fff;
  border: 1px solid #e5e7eb;
  border-radius: 6px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
  list-style: none;
  margin: 4px 0 0;
  padding: 0;
  max-height: 260px;
  overflow-y: auto;
}

.suggestion-item {
  padding: 0.625rem 0.875rem;
  cursor: pointer;
  font-size: 0.875rem;
  border-bottom: 1px solid #f3f4f6;
  display: flex;
  flex-direction: column;
  gap: 0.125rem;
}
.suggestion-item:last-child { border-bottom: none; }
.suggestion-item:hover,
.suggestion-item[aria-selected="true"] { background: #eff6ff; }

.suggestion-item__main  { font-weight: 500; color: #111827; }
.suggestion-item__sub   { font-size: 0.78rem; color: #9ca3af; }

/* ─── Error ─── */
.field-error { font-size: 0.8rem; color: #dc2626; margin: 0; }

/* ─── Structured address grid ─── */
.address-structured { display: flex; flex-direction: column; gap: 0.5rem; }
.address-row { display: grid; grid-template-columns: 1fr auto; gap: 0.5rem; }
.address-row input[name="postcode"] { width: 120px; }

/* ─── Map container ─── */
.address-map {
  height: 240px;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  overflow: hidden;
  background: #f3f4f6;  /* placeholder colour before SDK loads */
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .field-spinner { animation: none; border-top-color: #6b7280; opacity: 0.6; }
  .field-input   { transition: none; }
}
```

**What CSS owns:** spinner animation, dropdown positioning and scroll, input focus ring, map container sizing, reduced-motion fallback.

**What CSS cannot own:** when the spinner is visible, which suggestions populate the list, map pin position, whether the structured fields are shown, validation state.

---

## React Implementation

### Script Loader Utility

```typescript
// utils/loadScript.ts
//
// Module-level cache: the Map persists across component mounts and HMR
// reloads within a single page session. Calling loadScript() with the same
// URL always returns the same Promise — the <script> tag is only injected once.

const scriptCache = new Map<string, Promise<void>>();

export function loadScript(src: string): Promise<void> {
  if (scriptCache.has(src)) {
    return scriptCache.get(src)!;
  }

  const promise = new Promise<void>((resolve, reject) => {
    // Guard against a tag already in the DOM (e.g. SSR hydration, manual inclusion)
    const existing = document.querySelector<HTMLScriptElement>(`script[src="${src}"]`);
    if (existing) {
      resolve();
      return;
    }

    const script    = document.createElement('script');
    script.src      = src;
    script.async    = true;
    script.onload   = () => resolve();
    script.onerror  = () => {
      scriptCache.delete(src);       // allow retry on next call
      reject(new Error(`Failed to load script: ${src}`));
    };
    document.head.appendChild(script);
  });

  scriptCache.set(src, promise);
  return promise;
}
```

### Types

```typescript
// types/address.ts
export interface PlaceSuggestion {
  placeId:       string;
  mainText:      string;
  secondaryText: string;
}

export interface AddressResult {
  placeId:   string;
  formatted: string;
  street:    string;
  city:      string;
  postcode:  string;
  lat:       number;
  lng:       number;
}
```

### The Hook

```tsx
// hooks/useAddressAutocomplete.ts
import { useState, useEffect, useRef, useCallback } from 'react';
import { loadScript } from '../utils/loadScript';
import type { PlaceSuggestion, AddressResult } from '../types/address';

declare const google: typeof import('@types/google.maps');

const MAPS_URL =
  `https://maps.googleapis.com/maps/api/js` +
  `?key=${import.meta.env.VITE_GOOGLE_MAPS_KEY}&libraries=places`;

// Parse a single address_component type, with a fallback chain
function getComponent(
  components: google.maps.GeocoderAddressComponent[],
  ...types: string[]
): string {
  for (const type of types) {
    const match = components.find(c => c.types.includes(type));
    if (match) return match.long_name;
  }
  return '';
}

export type SdkStatus = 'idle' | 'loading' | 'ready' | 'error';

export function useAddressAutocomplete() {
  const [sdkStatus,   setSdkStatus]   = useState<SdkStatus>('idle');
  const [suggestions, setSuggestions] = useState<PlaceSuggestion[]>([]);
  const [isFetching,  setFetching]    = useState(false);
  const [selected,    setSelected]    = useState<AddressResult | null>(null);
  const [error,       setError]       = useState<string | null>(null);
  const [inputValue,  setInputValue]  = useState('');

  const autocompleteServiceRef = useRef<google.maps.places.AutocompleteService | null>(null);
  const sessionTokenRef        = useRef<google.maps.places.AutocompleteSessionToken | null>(null);
  const debounceTimerRef       = useRef<ReturnType<typeof setTimeout> | null>(null);

  // Load the SDK exactly once
  useEffect(() => {
    setSdkStatus('loading');
    loadScript(MAPS_URL)
      .then(() => {
        autocompleteServiceRef.current = new google.maps.places.AutocompleteService();
        sessionTokenRef.current        = new google.maps.places.AutocompleteSessionToken();
        setSdkStatus('ready');
      })
      .catch(() => {
        setSdkStatus('error');
        setError('Maps failed to load. You can still enter your address manually.');
      });

    return () => {
      if (debounceTimerRef.current) clearTimeout(debounceTimerRef.current);
    };
  }, []);

  const handleInput = useCallback((value: string) => {
    setInputValue(value);
    setSelected(null);
    setError(null);

    // Cancel any in-flight debounce
    if (debounceTimerRef.current) clearTimeout(debounceTimerRef.current);

    if (!value.trim() || sdkStatus !== 'ready' || !autocompleteServiceRef.current) {
      setSuggestions([]);
      return;
    }

    setFetching(true);

    // 300ms debounce — wait for the user to pause before calling the API
    debounceTimerRef.current = setTimeout(() => {
      autocompleteServiceRef.current!.getPlacePredictions(
        {
          input:        value,
          sessionToken: sessionTokenRef.current!,
          types:        ['address'],
          // Restrict to a country if appropriate; omit for global
          // componentRestrictions: { country: 'gb' },
        },
        (predictions, status) => {
          setFetching(false);
          if (
            status !== google.maps.places.PlacesServiceStatus.OK ||
            !predictions?.length
          ) {
            setSuggestions([]);
            return;
          }
          setSuggestions(
            predictions.map(p => ({
              placeId:       p.place_id,
              mainText:      p.structured_formatting.main_text,
              secondaryText: p.structured_formatting.secondary_text ?? '',
            }))
          );
        }
      );
    }, 300);
  }, [sdkStatus]);

  const confirmPlace = useCallback((placeId: string) => {
    if (sdkStatus !== 'ready') return;

    setFetching(true);
    setSuggestions([]);

    // PlacesService requires a DOM anchor — create a throwaway div
    const anchor        = document.createElement('div');
    const placesService = new google.maps.places.PlacesService(anchor);

    placesService.getDetails(
      {
        placeId,
        fields:       ['formatted_address', 'geometry', 'address_components'],
        sessionToken: sessionTokenRef.current!,
      },
      (place, status) => {
        setFetching(false);

        // Rotate the session token AFTER getDetails — Google bills per session,
        // not per getPlacePredictions call, when the token is used correctly.
        sessionTokenRef.current = new google.maps.places.AutocompleteSessionToken();

        if (status !== google.maps.places.PlacesServiceStatus.OK || !place) {
          setError('Could not retrieve address details. Please try again.');
          return;
        }

        const components = place.address_components ?? [];
        const result: AddressResult = {
          placeId,
          formatted: place.formatted_address ?? '',
          street:    [
            getComponent(components, 'street_number'),
            getComponent(components, 'route'),
          ].filter(Boolean).join(' '),
          city:     getComponent(components, 'postal_town', 'locality', 'sublocality'),
          postcode: getComponent(components, 'postal_code'),
          lat:      place.geometry?.location?.lat() ?? 0,
          lng:      place.geometry?.location?.lng() ?? 0,
        };

        setSelected(result);
        setInputValue(result.formatted);
      }
    );
  }, [sdkStatus]);

  return {
    inputValue,
    suggestions,
    isFetching,
    selected,
    error,
    sdkStatus,
    handleInput,
    confirmPlace,
  };
}
```

### Component

```tsx
// components/AddressAutocomplete.tsx
import { useRef, useState, useId, useEffect } from 'react';
import { useAddressAutocomplete } from '../hooks/useAddressAutocomplete';

declare const google: typeof import('@types/google.maps');

export function AddressAutocomplete() {
  const {
    inputValue, suggestions, isFetching, selected, error,
    sdkStatus, handleInput, confirmPlace,
  } = useAddressAutocomplete();

  const listboxId      = useId();
  const [activeIndex, setActiveIndex] = useState(-1);

  // Map refs — held outside React state so they do not trigger re-renders
  const mapContainerRef = useRef<HTMLDivElement>(null);
  const mapInstanceRef  = useRef<google.maps.Map    | null>(null);
  const markerRef       = useRef<google.maps.Marker | null>(null);

  // Update the map whenever a place is confirmed
  useEffect(() => {
    if (!selected || !mapContainerRef.current || sdkStatus !== 'ready') return;
    const pos = { lat: selected.lat, lng: selected.lng };

    if (!mapInstanceRef.current) {
      mapInstanceRef.current = new google.maps.Map(mapContainerRef.current, {
        center:           pos,
        zoom:             16,
        disableDefaultUI: true,
        zoomControl:      true,
      });
      markerRef.current = new google.maps.Marker({
        position: pos,
        map:      mapInstanceRef.current,
      });
    } else {
      mapInstanceRef.current.panTo(pos);
      markerRef.current?.setPosition(pos);
    }
  }, [selected, sdkStatus]);

  // Reset active index when suggestions list changes
  useEffect(() => { setActiveIndex(-1); }, [suggestions]);

  const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {
    if (!suggestions.length) return;

    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault();
        setActiveIndex(prev => Math.min(prev + 1, suggestions.length - 1));
        break;
      case 'ArrowUp':
        e.preventDefault();
        setActiveIndex(prev => Math.max(prev - 1, 0));
        break;
      case 'Enter':
        if (activeIndex >= 0) {
          e.preventDefault();
          confirmPlace(suggestions[activeIndex].placeId);
        }
        break;
      case 'Escape':
        setActiveIndex(-1);
        break;
    }
  };

  return (
    <div className="address-form">
      {/* Combobox */}
      <div className="address-field">
        <label htmlFor="address-input" className="field-label">
          Delivery address
        </label>

        <div className="address-input-wrapper">
          <input
            id="address-input"
            type="text"
            className="field-input"
            value={inputValue}
            onChange={e => handleInput(e.target.value)}
            onKeyDown={handleKeyDown}
            placeholder="Start typing an address…"
            autoComplete="off"
            spellCheck={false}
            role="combobox"
            aria-autocomplete="list"
            aria-controls={listboxId}
            aria-expanded={suggestions.length > 0}
            aria-activedescendant={
              activeIndex >= 0 ? `${listboxId}-option-${activeIndex}` : undefined
            }
            aria-invalid={!!error}
          />
          {isFetching && <span className="field-spinner" aria-hidden="true" />}
        </div>

        {suggestions.length > 0 && (
          <ul
            id={listboxId}
            className="suggestions-list"
            role="listbox"
            aria-label="Address suggestions"
          >
            {suggestions.map((s, i) => (
              <li
                key={s.placeId}
                id={`${listboxId}-option-${i}`}
                className="suggestion-item"
                role="option"
                aria-selected={i === activeIndex}
                // mousedown instead of click prevents the input blur from
                // closing the list before the click registers
                onMouseDown={e => { e.preventDefault(); confirmPlace(s.placeId); }}
              >
                <span className="suggestion-item__main">{s.mainText}</span>
                <span className="suggestion-item__sub">{s.secondaryText}</span>
              </li>
            ))}
          </ul>
        )}

        {error && (
          <p className="field-error" role="alert">{error}</p>
        )}
      </div>

      {/* Structured fields — shown only after a place is confirmed */}
      {selected && (
        <div className="address-structured">
          <input
            name="street"
            value={selected.street}
            readOnly
            className="field-input"
            aria-label="Street"
          />
          <div className="address-row">
            <input
              name="city"
              value={selected.city}
              readOnly
              className="field-input"
              aria-label="City"
            />
            <input
              name="postcode"
              value={selected.postcode}
              readOnly
              className="field-input"
              aria-label="Postcode"
            />
          </div>
          <input type="hidden" name="lat"     value={selected.lat}     />
          <input type="hidden" name="lng"     value={selected.lng}     />
          <input type="hidden" name="placeId" value={selected.placeId} />
        </div>
      )}

      {/* Map preview */}
      <div
        ref={mapContainerRef}
        className="address-map"
        role="img"
        aria-label={
          selected
            ? `Map showing ${selected.formatted}`
            : 'Map preview — no address selected'
        }
      />
    </div>
  );
}
```

### Async Form Validation

```tsx
// Validate that the user confirmed a place, not just typed a string.
// Wire this into React Hook Form or any validation library.

import type { AddressResult } from '../types/address';

export function validateAddressConfirmed(
  inputValue: string,
  selected: AddressResult | null
): string | null {
  if (!inputValue.trim()) return 'Address is required.';
  if (!selected) return 'Please select an address from the suggestions.';
  if (selected.formatted !== inputValue) {
    // The user edited the confirmed address without re-selecting
    return 'Address was changed. Please select from the dropdown.';
  }
  return null; // valid
}
```

---

## Angular Implementation

### Shared Script Loader

```typescript
// utils/load-script.ts
// Same idempotent pattern, framework-agnostic
const cache = new Map<string, Promise<void>>();

export function loadScript(src: string): Promise<void> {
  if (cache.has(src)) return cache.get(src)!;

  const p = new Promise<void>((resolve, reject) => {
    if (document.querySelector(`script[src="${src}"]`)) { resolve(); return; }
    const s    = document.createElement('script');
    s.src      = src;
    s.async    = true;
    s.onload   = () => resolve();
    s.onerror  = () => { cache.delete(src); reject(new Error(`Script load failed: ${src}`)); };
    document.head.appendChild(s);
  });

  cache.set(src, p);
  return p;
}
```

### Angular Component

```typescript
// address-autocomplete.component.ts
import {
  Component, OnInit, AfterViewInit, OnDestroy,
  ViewChild, ElementRef,
  signal, computed, inject, DestroyRef,
} from '@angular/core';
import { FormControl, ReactiveFormsModule, Validators } from '@angular/forms';
import {
  debounceTime, distinctUntilChanged, switchMap,
  catchError, tap, of, from,
} from 'rxjs';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { NgFor, NgIf } from '@angular/common';
import { loadScript } from '../utils/load-script';

declare const google: typeof import('@types/google.maps');

interface Suggestion  { placeId: string; mainText: string; secondaryText: string; }
interface AddressResult {
  placeId: string; formatted: string;
  street: string; city: string; postcode: string;
  lat: number; lng: number;
}

const MAPS_URL =
  `https://maps.googleapis.com/maps/api/js?key=YOUR_KEY&libraries=places`;

@Component({
  selector: 'app-address-autocomplete',
  standalone: true,
  imports: [ReactiveFormsModule, NgIf, NgFor],
  template: `
    <div class="address-form">

      <!-- Combobox -->
      <div class="address-field">
        <label for="address-input" class="field-label">Delivery address</label>
        <div class="address-input-wrapper">
          <input
            id="address-input"
            type="text"
            class="field-input"
            placeholder="Start typing an address…"
            autocomplete="off"
            role="combobox"
            aria-autocomplete="list"
            aria-controls="ng-address-listbox"
            [attr.aria-expanded]="suggestions().length > 0"
            [attr.aria-activedescendant]="activeId()"
            [formControl]="addressCtrl"
            (keydown)="onKeyDown($event)"
          />
          @if (isFetching()) {
            <span class="field-spinner" aria-hidden="true"></span>
          }
        </div>

        @if (suggestions().length > 0) {
          <ul id="ng-address-listbox" class="suggestions-list"
              role="listbox" aria-label="Address suggestions">
            @for (s of suggestions(); track s.placeId; let i = $index) {
              <li
                [id]="'ng-opt-' + i"
                class="suggestion-item"
                role="option"
                [attr.aria-selected]="i === activeIndex()"
                (mousedown)="$event.preventDefault(); confirmPlace(s.placeId)"
              >
                <span class="suggestion-item__main">{{ s.mainText }}</span>
                <span class="suggestion-item__sub">{{ s.secondaryText }}</span>
              </li>
            }
          </ul>
        }

        @if (error()) {
          <p class="field-error" role="alert">{{ error() }}</p>
        }
      </div>

      <!-- Structured fields -->
      @if (selected()) {
        <div class="address-structured">
          <input [value]="selected()!.street"   readonly class="field-input" aria-label="Street" />
          <div class="address-row">
            <input [value]="selected()!.city"     readonly class="field-input" aria-label="City" />
            <input [value]="selected()!.postcode" readonly class="field-input" aria-label="Postcode" />
          </div>
        </div>
      }

      <!-- Map -->
      <div #mapEl class="address-map" role="img"
           [attr.aria-label]="mapLabel()">
      </div>

    </div>
  `,
})
export class AddressAutocompleteComponent implements OnInit, OnDestroy {
  @ViewChild('mapEl', { static: true }) mapElRef!: ElementRef<HTMLDivElement>;

  addressCtrl  = new FormControl('', { nonNullable: true });
  suggestions  = signal<Suggestion[]>([]);
  isFetching   = signal(false);
  selected     = signal<AddressResult | null>(null);
  error        = signal<string | null>(null);
  activeIndex  = signal(-1);
  activeId     = computed(() =>
    this.activeIndex() >= 0 ? `ng-opt-${this.activeIndex()}` : null
  );
  mapLabel     = computed(() =>
    this.selected()
      ? `Map showing ${this.selected()!.formatted}`
      : 'Map preview — no address selected'
  );

  private destroyRef   = inject(DestroyRef);
  private acService:   google.maps.places.AutocompleteService | null = null;
  private sessionToken: google.maps.places.AutocompleteSessionToken | null = null;
  private map:    google.maps.Map    | null = null;
  private marker: google.maps.Marker | null = null;

  ngOnInit() {
    // Load SDK, then wire up the valueChanges stream
    loadScript(MAPS_URL)
      .then(() => {
        this.acService    = new google.maps.places.AutocompleteService();
        this.sessionToken = new google.maps.places.AutocompleteSessionToken();
      })
      .catch(() => {
        this.error.set('Maps failed to load. You can still enter your address manually.');
      });

    this.addressCtrl.valueChanges.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      tap(() => {
        this.isFetching.set(true);
        this.selected.set(null);
        this.activeIndex.set(-1);
      }),
      switchMap(val => {
        if (!val.trim() || !this.acService) {
          this.isFetching.set(false);
          this.suggestions.set([]);
          return of([] as Suggestion[]);
        }
        return from(
          new Promise<Suggestion[]>(resolve => {
            this.acService!.getPlacePredictions(
              { input: val, sessionToken: this.sessionToken!, types: ['address'] },
              (preds, status) => {
                if (status !== google.maps.places.PlacesServiceStatus.OK || !preds) {
                  resolve([]);
                } else {
                  resolve(preds.map(p => ({
                    placeId:       p.place_id,
                    mainText:      p.structured_formatting.main_text,
                    secondaryText: p.structured_formatting.secondary_text ?? '',
                  })));
                }
              }
            );
          })
        );
      }),
      catchError(() => of([] as Suggestion[])),
      takeUntilDestroyed(this.destroyRef),
    ).subscribe(suggestions => {
      this.isFetching.set(false);
      this.suggestions.set(suggestions);
    });
  }

  confirmPlace(placeId: string) {
    if (!this.acService) return;
    this.suggestions.set([]);
    this.isFetching.set(true);

    const anchor  = document.createElement('div');
    const service = new google.maps.places.PlacesService(anchor);

    service.getDetails(
      {
        placeId,
        fields:       ['formatted_address', 'geometry', 'address_components'],
        sessionToken: this.sessionToken!,
      },
      (place, status) => {
        this.isFetching.set(false);
        this.sessionToken = new google.maps.places.AutocompleteSessionToken();

        if (status !== google.maps.places.PlacesServiceStatus.OK || !place) {
          this.error.set('Could not retrieve address details. Please try again.');
          return;
        }

        const get = (...types: string[]) => {
          for (const type of types) {
            const c = place.address_components?.find(x => x.types.includes(type));
            if (c) return c.long_name;
          }
          return '';
        };

        const result: AddressResult = {
          placeId,
          formatted: place.formatted_address ?? '',
          street:    [get('street_number'), get('route')].filter(Boolean).join(' '),
          city:      get('postal_town', 'locality', 'sublocality'),
          postcode:  get('postal_code'),
          lat:       place.geometry?.location?.lat() ?? 0,
          lng:       place.geometry?.location?.lng() ?? 0,
        };

        this.selected.set(result);
        this.addressCtrl.setValue(result.formatted, { emitEvent: false });
        this.updateMap(result.lat, result.lng);
      }
    );
  }

  onKeyDown(e: KeyboardEvent) {
    const list = this.suggestions();
    if (!list.length) return;

    if (e.key === 'ArrowDown') {
      e.preventDefault();
      this.activeIndex.update(i => Math.min(i + 1, list.length - 1));
    } else if (e.key === 'ArrowUp') {
      e.preventDefault();
      this.activeIndex.update(i => Math.max(i - 1, 0));
    } else if (e.key === 'Enter' && this.activeIndex() >= 0) {
      e.preventDefault();
      this.confirmPlace(list[this.activeIndex()].placeId);
    } else if (e.key === 'Escape') {
      this.activeIndex.set(-1);
      this.suggestions.set([]);
    }
  }

  private updateMap(lat: number, lng: number) {
    const pos = { lat, lng };
    const el  = this.mapElRef.nativeElement;

    if (!this.map) {
      this.map   = new google.maps.Map(el, {
        center: pos, zoom: 16, disableDefaultUI: true, zoomControl: true,
      });
      this.marker = new google.maps.Marker({ position: pos, map: this.map });
    } else {
      this.map.panTo(pos);
      this.marker?.setPosition(pos);
    }
  }

  ngOnDestroy() {
    // The Maps SDK global remains; local state cleaned up by signal GC
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Input uses `role="combobox"` | Correct ARIA pattern for an input with an associated listbox |
| `aria-autocomplete="list"` | Indicates the popup is a list of suggestions |
| `aria-expanded` reflects list visibility | `true` when `suggestions.length > 0`, `false` otherwise |
| `aria-controls` links input to listbox `id` | Static attribute pointing to the `<ul>` |
| `aria-activedescendant` tracks keyboard cursor | Updated on ArrowUp/ArrowDown; cleared on selection |
| Each option has `role="option"` | Required child role inside `role="listbox"` |
| `aria-selected` on each option | `true` for keyboard-focused item, `false` for others |
| `onMouseDown` with `preventDefault` prevents blur | Keeps the input focused when clicking a suggestion |
| Errors announced immediately | `role="alert"` on the error paragraph (assertive live region) |
| Map container is labelled | `role="img"` with `aria-label` that updates to name the confirmed address |
| Read-only structured fields have labels | `aria-label` on each `readonly` input |
| Spinner is hidden from AT | `aria-hidden="true"` on the spinner element |

---

## Production Pitfalls

**1. No idempotency guard on `loadScript` causes duplicate SDK initialisations**
Without the module-level Promise cache, each component mount or hot-module reload appends a new `<script>` tag. The `google.maps` namespace is initialised by the first script; subsequent scripts may conflict or trigger the `You have included the Google Maps JavaScript API multiple times` console error. The cache ensures the same URL always returns the same Promise.

**2. Session tokens not rotated drains quota budget**
Google's Places billing model groups all `getPlacePredictions` calls that share the same `AutocompleteSessionToken` into one billable "session" — as long as `getDetails` is called with the same token to close the session. If the token is never rotated (or never passed to `getDetails`), each prediction call is billed individually. Always rotate the token immediately after `getDetails` resolves.

**3. Accepting a typed string as a valid address**
A user can type "10 Fake Street" and submit the form without selecting from the dropdown. The form never receives a `placeId` or coordinates. Validate at submission time: confirm that `selected !== null` and that `selected.formatted === inputValue`. If the user edited the confirmed text, require them to re-select.

**4. `PlacesService` requires a DOM anchor**
`new google.maps.places.PlacesService(el)` must receive an existing `Element`. If the map `<div>` is not yet in the DOM when you need to call `getDetails` (e.g. the user selects before the map renders), pass a throwaway `document.createElement('div')` as the anchor instead of the map element.

**5. `mousedown` vs `click` for suggestion selection**
Using `onClick` on suggestion items causes the input to fire a `blur` event first, which typically hides the list — the `click` never reaches the item. Replace with `onMouseDown` + `event.preventDefault()`. The `preventDefault` prevents the input from losing focus; the selection still fires.

**6. `address_components` structure varies by country**
`postal_town` is a UK-specific component type. The US equivalent is `locality`. Australia uses `locality` and `administrative_area_level_2`. Always provide a fallback chain: `get('postal_town', 'locality', 'sublocality')`. Never hardcode a single type.

**7. API load failure leaves the form unusable**
If the Maps SDK fails to load (network error, quota exceeded, blocked by content security policy), the component must still allow manual address entry. Degrade by hiding the suggestions list, showing a plain text input, and removing the map. Present a clear error message rather than a silent broken UI.

---

## Interview Angle

**Q: "How do you dynamically load a third-party script like the Google Maps SDK and guarantee it is only loaded once, even across component remounts?"** The correct answer is a module-scoped Promise cache. A plain `Map<string, Promise<void>>` declared at the module level (outside any component or hook) persists for the lifetime of the page. The first call creates a `<script>` element, appends it, and stores the resulting Promise. Every subsequent call with the same URL returns the stored Promise immediately — whether it is pending (script still loading) or resolved (already loaded). The cache entry is deleted on error so a failed load can be retried. The alternative — checking `document.querySelector('script[src]')` without a cache — has a race condition: two simultaneous first calls both find no existing tag and both inject a `<script>`.

**Follow-up: "How does the Places Session Token affect billing, and when do you rotate it?"** Google charges for Places Autocomplete on a per-session basis when session tokens are used correctly. One "session" covers all the `getPlacePredictions` calls from when the user starts typing until the user either selects a suggestion (triggering `getDetails`) or abandons the search. Pass the same token to all `getPlacePredictions` calls in a session and to the final `getDetails` call. Immediately after `getDetails` resolves — success or failure — create a new `AutocompleteSessionToken` for the next session. If a token is never passed (or never closed with a `getDetails` call), each prediction request is billed as an individual SKU, which can be ten times more expensive per-query.

**Follow-up: "How do you prevent form submission when the user typed but never selected an address?"** Maintain a `selected` state object that is only populated by `getDetails` — not by typing. On form submit, validate that `selected !== null`. Also check `selected.formatted === inputValue`: if the user selected an address and then kept typing, the confirmed place is stale and must be re-selected. This two-part check is necessary because the input is a free-text field that accepts any string, not a controlled select element.

# Real-World Scenarios

End-to-end walkthroughs for 75 complex UI problems. Each scenario covers the real-world analogy, the CSS/HTML foundation, why CSS alone isn't enough (when applicable), the React implementation, the Angular implementation, an accessibility checklist, production pitfalls, and the interview angle.

**How to use this index:** Find the *type* of problem you're solving, not just the UI component. A "dropdown" is a CSS problem on desktop but a state management problem on mobile.

---

## Group 1 — CSS & Visual Layout Problems

> The complexity lives in the stylesheet. JS plays a supporting role (toggling classes), but the hard part is CSS.

| # | ID | Scenario | Core CSS Concept |
| --- | --- | --- | --- |
| 01 | `NAV-01` | [Complex Multi-Level Responsive Navigation](../02-css/06-responsive-and-a11y/07-complex-navigation.md) | Off-canvas drawer, level panels, `translateX` transitions |
| 02 | `NAV-02` | [Mega Menu (Desktop)](./NAV-02-mega-menu-desktop.md) | `position: absolute`, grid inside dropdown, `focus-within` |
| 03 | `NAV-03` | [Breadcrumb with Overflow Truncation](./NAV-03-breadcrumb-overflow-truncation.md) | `text-overflow`, container queries, ellipsis at midpoint |
| 04 | `NAV-04` | [Sticky Header with Scroll-Aware Shrink](./NAV-04-sticky-header-scroll-shrink.md) | `position: sticky`, CSS custom property animation |
| 05 | `NAV-05` | [Tab Bar with Active Indicator Animation](./NAV-05-tab-bar-active-indicator.md) | CSS custom properties, `translateX` sliding underline |
| 06 | `LAYOUT-05` | [Responsive Card Grid (auto-fill/auto-fit)](./LAYOUT-05-responsive-card-grid.md) | `minmax`, `auto-fill` vs `auto-fit`, implicit grid rows |
| 07 | `LAYOUT-06` | [Sticky Sidebar with Scroll Sync](./LAYOUT-06-sticky-sidebar-scroll-sync.md) | `position: sticky`, `top` offset, scroll-margin |
| 08 | `LAYOUT-02` | [Masonry / Pinterest-Style Grid](./LAYOUT-02-masonry-grid.md) | CSS `columns`, gap, break-inside |
| 09 | `PERF-01` | [Skeleton Loading Screens](./PERF-01-skeleton-loading.md) | CSS shimmer `@keyframes`, `background-size`, layout stability |
| 10 | `PERF-03` | [Image Lazy Loading with Blur-Up Placeholder](./PERF-03-image-lazy-blur-placeholder.md) | `filter: blur()`, `loading="lazy"`, aspect-ratio box |
| 11 | `ANIM-02` | [Scroll-Driven Animations](./ANIM-02-scroll-driven-animations.md) | `animation-timeline: scroll()`, `animation-range` |
| 12 | `ANIM-05` | [Micro-Interaction Button States](./ANIM-05-micro-interaction-buttons.md) | `transition`, `transform`, `@keyframes` for spinner |
| 13 | `A11Y-05` | [High Contrast & Forced Colors Mode](./A11Y-05-high-contrast-forced-colors.md) | `forced-colors` media query, `SystemColor` keywords |
| 14 | `A11Y-06` | [Reduced Motion Animations](./A11Y-06-reduced-motion.md) | `prefers-reduced-motion`, instant vs animated fallback |
| 15 | `MODAL-02` | [Bottom Sheet (Mobile Drawer)](./MODAL-02-bottom-sheet.md) | `translateY`, snap points, `overscroll-behavior` |

---

## Group 2 — Component Design Patterns

> The complexity is in how you structure the component — its API, composition model, and separation of concerns. These map directly to interview "design a component" questions.

| # | ID | Scenario | Pattern |
| --- | --- | --- | --- |
| 16 | `FORM-03` | [Combobox / Searchable Select](./FORM-03-combobox-searchable-select.md) | Controlled props, headless component, ARIA combobox |
| 17 | `FORM-04` | [Date Range Picker](./FORM-04-date-range-picker.md) | Compound component, shared calendar state |
| 18 | `FORM-08` | [OTP / PIN Input](./FORM-08-otp-pin-input.md) | Controlled refs, imperative handle, auto-advance |
| 19 | `MODAL-01` | [Accessible Modal Dialog](./MODAL-01-accessible-modal-dialog.md) | Portal pattern, compound component, `<dialog>` vs div |
| 20 | `MODAL-03` | [Nested / Stacked Modals](./MODAL-03-nested-stacked-modals.md) | Stack pattern, z-index management, focus history |
| 21 | `MODAL-04` | [Tooltip & Popover Positioning](./MODAL-04-tooltip-popover-positioning.md) | Floating UI / CDK Overlay, flip/shift middleware |
| 22 | `MODAL-06` | [Toast / Notification System](./MODAL-06-toast-notification-system.md) | Singleton service, queue pattern, render-to-portal |
| 23 | `MODAL-07` | [Full-Screen Image Lightbox](./MODAL-07-image-lightbox.md) | Compound component, keyboard gallery, zoom state |
| 24 | `DATA-04` | [Editable Table (Inline Editing)](./DATA-04-editable-table-inline.md) | Click-to-edit, row vs cell mode, save/cancel API |
| 25 | `DATA-05` | [Tree Table (Expandable Rows)](./DATA-05-tree-table-expandable.md) | Recursive component, lazy children, controlled expand |
| 26 | `DATA-06` | [Sparkline & Inline Micro Charts](./DATA-06-sparkline-micro-charts.md) | Headless chart, SVG path generation, render prop |
| 27 | `UPLOAD-04` | [Video Player with Custom Controls](./UPLOAD-04-video-player-custom-controls.md) | Controlled `<video>` API, uncontrolled ref, event surface |
| 28 | `A11Y-02` | [Accessible Accordion](./A11Y-02-accessible-accordion.md) | Compound component, `aria-controls`, keyboard contract |
| 29 | `NAV-06` | [Command Palette (⌘K)](./NAV-06-command-palette.md) | Singleton overlay, fuzzy search, virtual list, portal |
| 30 | `FORM-05` | [Rich Text Editor Integration](./FORM-05-rich-text-editor.md) | Uncontrolled component, imperative bridge, toolbar state |

---

## Group 3 — State Management Patterns

> The complexity is in *what* to store, *where* to store it, and *how* to keep it consistent. Pure state problems, not rendering problems.

| # | ID | Scenario | State Pattern |
| --- | --- | --- | --- |
| 31 | `FORM-01` | [Multi-Step Form Wizard](./FORM-01-multi-step-form-wizard.md) | Step state machine, per-step validation, history |
| 32 | `FORM-02` | [Dynamic Form Builder](./FORM-02-dynamic-form-builder.md) | Schema-driven state, recursive field tree, conditional logic |
| 33 | `DATA-07` | [Kanban Board with Drag-and-Drop](./DATA-07-kanban-board-dnd.md) | Normalized column/card state, optimistic reorder |
| 34 | `LAYOUT-01` | [Dashboard Grid with Resizable Panels](./LAYOUT-01-dashboard-grid-resizable.md) | Persisted layout state, localStorage sync |
| 35 | `LAYOUT-03` | [App Shell with Collapsible Sidebar](./LAYOUT-03-app-shell-sidebar.md) | URL-synced UI state, localStorage fallback |
| 36 | `LAYOUT-04` | [Split Pane Editor](./LAYOUT-04-split-pane-editor.md) | Drag-derived state, min/max clamping, keyboard delta |
| 37 | `PERF-06` | [Optimistic UI Updates with Rollback](./PERF-06-optimistic-ui-rollback.md) | Temp ID, mutation state machine, error recovery |
| 38 | `AUTH-04` | [Token Refresh with Request Queue](./AUTH-04-token-refresh-queue.md) | Queued retry, race-free refresh, interceptor pattern |
| 39 | `AUTH-05` | [Session Timeout Warning Modal](./AUTH-05-session-timeout-modal.md) | Countdown timer, `visibilitychange`, keep-alive |
| 40 | `RT-03` | [Chat Interface with Typing Indicators](./RT-03-chat-typing-indicators.md) | Optimistic send, delivery status enum, scroll anchor |
| 41 | `RT-04` | [Live Dashboard with Auto-Refresh](./RT-04-live-dashboard-auto-refresh.md) | Polling with backoff, stale-data signal, pause-when-hidden |
| 42 | `MODAL-05` | [Context Menu (Right-Click)](./MODAL-05-context-menu.md) | Cursor-relative position state, dismiss on outside click |

---

## Group 4 — Async, Data Fetching & Real-Time

> The complexity is in the network layer — race conditions, cancellation, streaming, reconnection, and keeping UI in sync with server state.

| # | ID | Scenario | Async Pattern |
| --- | --- | --- | --- |
| 43 | `PERF-05` | [Debounced Search with Cancellation](./PERF-05-debounced-search-cancellation.md) | `AbortController`, race condition prevention, pending state |
| 44 | `PERF-04` | [Code-Split Route with Loading UI](./PERF-04-code-split-route-loading.md) | `React.lazy` / `@defer`, Suspense boundary, fallback skeleton |
| 45 | `DATA-03` | [Server-Side Sortable/Filterable Table](./DATA-03-server-side-sortable-table.md) | URL state sync, debounced fetch, loading/error states |
| 46 | `UPLOAD-02` | [Chunked / Resumable Upload with Progress](./UPLOAD-02-chunked-resumable-upload.md) | File slice, parallel chunks, retry on failure, progress event |
| 47 | `RT-01` | [Live Feed with New-Item Banner](./RT-01-live-feed-new-item-banner.md) | WebSocket / SSE, prepend vs append, "load new items" pattern |
| 48 | `RT-02` | [Real-Time Collaborative Cursor Presence](./RT-02-collaborative-cursor-presence.md) | WebSocket broadcast, throttled position update |
| 49 | `RT-05` | [Notification Badge with WebSocket](./RT-05-notification-badge-websocket.md) | Connection lifecycle, reconnect strategy, badge count |
| 50 | `PERF-07` | [Web Worker for Heavy Computation](./PERF-07-web-worker-computation.md) | Comlink, transferable objects, progress reporting |
| 51 | `DATA-02` | [Virtualized Infinite-Scroll Table](./DATA-02-virtualized-infinite-scroll-table.md) | Intersection Observer, page appending, row height variance |
| 52 | `PERF-02` | [Infinite Scroll with Intersection Observer](./PERF-02-infinite-scroll-intersection-observer.md) | Entry/exit detection, prepend without scroll jump, cleanup |
| 53 | `FORM-06` | [Address Autocomplete with Map Preview](./FORM-06-address-autocomplete-map.md) | Debounced API, external script loading, async validation |
| 54 | `AUTH-02` | [OAuth Popup Flow](./AUTH-02-oauth-popup-flow.md) | `window.open`, `postMessage`, timeout, popup blocked detection |

---

## Group 5 — Accessibility & Keyboard Interaction

> These scenarios exist because accessibility is never "free" — it requires deliberate ARIA, focus management, and keyboard contracts that don't fall out of normal component code.

| # | ID | Scenario | A11y Technique |
| --- | --- | --- | --- |
| 55 | `A11Y-01` | [Skip Links & Focus Management on Route Change](./A11Y-01-skip-links-route-focus.md) | Skip nav, programmatic focus, `aria-live` SPA announcements |
| 56 | `A11Y-03` | [Live Region Announcements](./A11Y-03-live-region-announcements.md) | `aria-live`, `role="status"` vs `role="alert"`, polling updates |
| 57 | `A11Y-04` | [Accessible Drag-and-Drop (Keyboard Alternative)](./A11Y-04-accessible-drag-drop-keyboard.md) | `aria-grabbed`, keyboard mode, screen reader UX |
| 58 | `MODAL-01` | [Accessible Modal Dialog (A11y deep dive)](./MODAL-01-accessible-modal-a11y.md) | Focus trap, scroll lock, `aria-modal`, return focus on close |
| 59 | `FORM-03` | [Combobox / Searchable Select (A11y focus)](./FORM-03-combobox-a11y.md) | ARIA combobox pattern, `aria-activedescendant`, keyboard nav |
| 60 | `NAV-06` | [Command Palette A11y](./NAV-06-command-palette-a11y.md) | Focus trap, `role="dialog"`, keyboard-first design |
| 61 | `DATA-04` | [Editable Table Keyboard Navigation](./DATA-04-editable-table-a11y.md) | Grid keyboard navigation, cell focus management |
| 62 | `AUTH-01` | [Login Form Accessibility](./AUTH-01-login-form-a11y.md) | `aria-describedby`, field-level errors, `aria-invalid` |

---

## Group 6 — Animation & Motion

> The complexity is in *when* to animate, *what* to animate, and making it feel natural without causing layout thrash or accessibility harm.

| # | ID | Scenario | Technique |
| --- | --- | --- | --- |
| 63 | `ANIM-01` | [Page / Route Transition Animations](./ANIM-01-page-route-transitions.md) | View Transitions API, Framer Motion `<AnimatePresence>`, Angular animations |
| 64 | `ANIM-02` | [Scroll-Driven Animations](./ANIM-02-scroll-driven-animations.md) | CSS `animation-timeline: scroll()`, GSAP ScrollTrigger fallback |
| 65 | `ANIM-03` | [Shared Element Transition (Hero Animation)](./ANIM-03-shared-element-hero.md) | `view-transition-name`, FLIP technique |
| 66 | `ANIM-04` | [List Reorder Animation (FLIP)](./ANIM-04-list-reorder-flip.md) | GSAP FLIP, AutoAnimate, Angular CDK drag preview |
| 67 | `ANIM-05` | [Micro-Interaction Button States](./ANIM-05-micro-interaction-buttons.md) | CSS transitions, loading spinner, success/error morphing |

---

## Group 7 — Mobile & Touch Interactions

> Desktop-web assumptions break on mobile. These scenarios exist because touch events, viewport behaviour, and mobile OS quirks need explicit handling.

| # | ID | Scenario | Mobile Concern |
| --- | --- | --- | --- |
| 68 | `MOBILE-01` | [Pull-to-Refresh](./MOBILE-01-pull-to-refresh.md) | Touch events, `overscroll-behavior`, loading indicator |
| 69 | `MOBILE-02` | [Swipe-to-Delete List Item](./MOBILE-02-swipe-to-delete.md) | Touch start/move/end, velocity threshold, undo |
| 70 | `MOBILE-03` | [Virtual Keyboard Avoidance](./MOBILE-03-virtual-keyboard-avoidance.md) | `visualViewport` API, layout shift on keyboard open, `dvh` |
| 71 | `MOBILE-04` | [Long-Press Context Menu](./MOBILE-04-long-press-context-menu.md) | Touch hold timer, `contextmenu` event, haptic feedback API |
| 72 | `MOBILE-05` | [Pinch-to-Zoom Image](./MOBILE-05-pinch-to-zoom-image.md) | Pointer events, scale clamping, `transform-origin` |

---

## Group 8 — File & Media Handling

> These scenarios sit at the boundary of the browser's native media APIs and your UI layer.

| # | ID | Scenario | API Surface |
| --- | --- | --- | --- |
| 73 | `UPLOAD-01` | [Drag-and-Drop File Upload with Preview](./UPLOAD-01-drag-drop-file-upload.md) | File API, `createObjectURL`, MIME validation |
| 74 | `UPLOAD-02` | [Chunked / Resumable Upload with Progress](./UPLOAD-02-chunked-resumable-upload.md) | `File.slice()`, parallel `fetch`, retry on failure |
| 75 | `UPLOAD-03` | [Image Crop & Resize Before Upload](./UPLOAD-03-image-crop-resize.md) | Canvas API, crop UI, output as `Blob` |
| 76 | `UPLOAD-04` | [Video Player with Custom Controls](./UPLOAD-04-video-player-custom-controls.md) | `<video>` API, `timeupdate`, fullscreen, Picture-in-Picture |
| 77 | `UPLOAD-05` | [PDF Viewer (PDF.js)](./UPLOAD-05-pdf-viewer.md) | Canvas rendering, pagination, zoom, text layer overlay |

---

## Group 9 — Authentication UI Flows

> Auth UIs look simple but fail in subtle ways — flicker, race conditions between the auth check and the redirect, and session edge cases.

| # | ID | Scenario | Problem |
| --- | --- | --- | --- |
| 78 | `AUTH-01` | [Login / Register Form with Server Errors](./AUTH-01-login-register-form.md) | Field-level server errors, submission disable, password reveal |
| 79 | `AUTH-02` | [OAuth Popup Flow](./AUTH-02-oauth-popup-flow.md) | `window.open`, `postMessage`, timeout, popup blocked |
| 80 | `AUTH-03` | [Protected Route with Loading State](./AUTH-03-protected-route.md) | Auth guard, redirect-after-login, white-flash prevention |
| 81 | `AUTH-04` | [Token Refresh with Request Queue](./AUTH-04-token-refresh-queue.md) | Interceptor, queued retry, race-free refresh |
| 82 | `AUTH-05` | [Session Timeout Warning Modal](./AUTH-05-session-timeout-modal.md) | Countdown timer, `visibilitychange`, keep-alive ping |

---

## How Each Scenario File Is Structured

Every scenario follows the same layout:

1. **Real-world analogy** — the mental model in plain English before any code
2. **The problem** — the exact UX/engineering constraint being solved
3. **Why CSS alone isn't enough** — where the CSS ceiling is (skipped for pure CSS scenarios)
4. **HTML & CSS foundation** — the structural and visual layer
5. **React implementation** — hooks, state, component breakdown
6. **Angular implementation** — signals/RxJS, directives, service breakdown
7. **Accessibility checklist** — ARIA, keyboard, screen reader requirements
8. **Production pitfalls** — what breaks in real apps that doesn't in demos
9. **Interview angle** — how this comes up in senior interviews, and the strong answer

---

## Which Group to Start With?

| If you're weak on… | Start with |
| --- | --- |
| CSS layout and responsive | Group 1 |
| Component API design | Group 2 |
| State architecture | Group 3 |
| Async patterns and real-time | Group 4 |
| Accessibility | Group 5 |
| Animation | Group 6 |
| Touch / mobile | Group 7 |

**First file:** [NAV-01 — Complex Multi-Level Responsive Navigation](../02-css/06-responsive-and-a11y/07-complex-navigation.md) — demonstrates the full format used throughout this section.

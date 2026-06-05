# AOT vs JIT Compilation

## The Idea

**In plain English:** When you write an Angular app, your HTML-like templates need to be translated into instructions the browser can run. AOT (Ahead-Of-Time) does that translation before the app ships, while JIT (Just-In-Time) does it inside the browser right as the user loads the page.

**Real-world analogy:** Imagine a chef preparing meals for an airline. An AOT chef cooks and packs every meal in the kitchen before the flight departs. A JIT chef boards the plane with raw ingredients and cooks each meal after passengers have already sat down.

- The kitchen before the flight = build time (your development machine)
- The airplane mid-flight = the user's browser at runtime
- Cooking the meals in advance = AOT compiling templates before shipping
- Cooking after boarding = JIT compiling templates on every page load
- The packed meal ready to eat = pre-compiled JavaScript instructions Ivy produces

---

## What Gets Compiled?

Angular has two distinct compilation steps:
1. **TypeScript → JavaScript** (always happens at build time)
2. **Angular templates → JavaScript** (this is where AOT vs JIT diverges)

---

## JIT (Just-In-Time)

Templates are compiled **in the browser at runtime**, after the app bootstraps.

```
Build: TypeScript → JavaScript (template metadata as strings)
Browser: Download → Parse → Angular compiler runs → Components rendered
```

```ts
// JIT: The template is a string processed at runtime
@Component({
  template: `<div *ngIf="show">{{ message }}</div>`
  // ↑ Angular compiler must parse this in the browser
})
```

**Was used:** Development builds (until Angular 9), older production builds.

**Problems:**
- Larger bundle (includes Angular compiler: ~400 KB)
- Slower startup (template compilation on every page load)
- Template errors caught at runtime, not build time
- Templates can't be tree-shaken

---

## AOT (Ahead-Of-Time)

Templates are compiled **at build time** by the Angular CLI:

```
Build: TypeScript + Templates → Compiled JavaScript (no templates as strings)
Browser: Download → Parse → Run (no compilation step)
```

```ts
// AOT output: template compiled to instruction calls
ɵɵelementStart(0, 'div');
ɵɵtemplate(1, conditional_r1, 1, 0, 'ng-template', 1);
ɵɵelementEnd();
// ↑ Direct Ivy instruction calls — no runtime parsing
```

**Since Angular 9:** AOT with the Ivy renderer is the **default for all builds** (dev and prod).

---

## Ivy Compiler

Ivy (Angular 9+) is the AOT compiler that replaced the old View Engine:

- Templates compile to **localized component code** (not a global registry)
- Smaller bundles (tree-shaking works because code is localized)
- Faster compilation
- Better debugging (generated code is readable)
- Incremental compilation for faster rebuilds

---

## AOT Constraints

AOT runs at build time without a JavaScript runtime, so it has restrictions:

```ts
// ❌ Dynamic component factories from runtime values
const componentType = condition ? ComponentA : ComponentB;
@Component({ template: '<ng-template [ngComponentOutlet]="componentType">' })

// ✅ Use ViewContainerRef instead for dynamic components

// ❌ Non-public template expressions
@Component({ template: '{{ privateMethod() }}' })  // private in TypeScript
// Template expressions are compiled separately — must be public

// ❌ Arrow function in template
@Component({ template: '<button (click)="items.filter(i => i.active)">...' })
// ✅ Move to method
filter() { return this.items.filter(i => i.active); }
```

---

## Build Command Flags

```bash
# Production build (AOT, optimization, tree-shaking)
ng build --configuration=production

# Development build (still AOT in Angular 9+, but no minification)
ng build

# Check for AOT compilation errors
ng build --aot

# Serve with JIT (dev only, for debugging)
ng serve --aot=false  # not recommended
```

---

## Common Interview Questions

**Q: Is JIT still available in Angular?**
Technically yes, but not recommended. The Angular team targets AOT-only in future major versions. JIT requires shipping the compiler in the bundle and is significantly slower.

**Q: What are the bundle size differences?**
JIT builds include the Angular compiler (~100-400 KB depending on build). AOT builds exclude it. Ivy's locality principle means AOT bundles are also more efficiently tree-shaken.

**Q: How does AOT catch template errors early?**
The AOT compiler type-checks templates using TypeScript's type checker. Accessing a non-existent property, passing the wrong type, or using an undeclared variable in a template fails at build time rather than at runtime.

**Q: What's the relationship between AOT and Ivy?**
Ivy is the name of Angular's current compilation and rendering architecture (replacing View Engine). AOT with Ivy is the default since Angular 9. They're closely related but distinct: AOT is the "when" (build time), Ivy is the "how" (the instruction set and compiler design).

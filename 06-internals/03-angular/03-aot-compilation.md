# AOT Compilation: Ahead-of-Time vs Just-in-Time

## The Idea

**In plain English:** AOT (Ahead-of-Time) compilation means Angular translates your app's code and templates into plain JavaScript *before* it reaches the user's browser, so the browser can run it instantly without doing any translation itself. Think of "compilation" as translating instructions written in one language into another language a machine can directly execute.

**Real-world analogy:** Imagine a recipe book originally written in French that needs to reach English-speaking cooks. AOT is like hiring a translator to produce a finished English edition before the books are printed and shipped — every reader gets a ready-to-use book the moment it arrives. JIT (Just-in-Time) is like shipping the French book along with a pocket translator dictionary, so each cook must translate every step themselves before cooking.

- The French recipe book = your Angular templates and TypeScript code
- The translated English edition = the pre-compiled JavaScript sent to the browser
- The pocket translator dictionary = the Angular compiler (~500 KB) bundled with JIT builds
- Each cook translating on the fly = the browser compiling templates at runtime in JIT mode

---

## Overview

Angular's Ahead-of-Time (AOT) compilation transforms Angular HTML and TypeScript code into efficient JavaScript during the build process, before the browser downloads and runs the code. This contrasts with Just-in-Time (JIT) compilation, which compiles the application in the browser at runtime.

Understanding AOT compilation is crucial for optimizing Angular applications, as it affects bundle size, startup performance, security, and error detection. With Ivy, AOT compilation became faster, more efficient, and the default mode for all Angular applications.

## AOT vs JIT Compilation

### Compilation Timing

```typescript
// JIT Compilation (Development Mode)
// 1. Browser downloads application code
// 2. Browser downloads Angular compiler (~500KB)
// 3. Compiler runs in browser
// 4. Application starts

// Timeline:
// [Download App] -> [Download Compiler] -> [Compile] -> [Bootstrap] -> [Render]
//     1s              0.5s                  1.5s         0.2s          0.1s
// Total: ~3.3s to first render

// AOT Compilation (Production Mode)
// 1. Build server compiles application
// 2. Browser downloads compiled code
// 3. Application starts immediately

// Timeline:
// [Download App] -> [Bootstrap] -> [Render]
//     1s              0.2s          0.1s
// Total: ~1.3s to first render (60% faster!)
```

### Code Differences

```typescript
// Source Component
@Component({
  selector: 'app-user',
  template: `
    <div class="user">
      <h2>{{ user.name }}</h2>
      <p>{{ user.email }}</p>
    </div>
  `
})
export class UserComponent {
  @Input() user: User;
}

// JIT Output (Runtime Compilation)
class UserComponent {
  user: User;
  
  // Template compiled at runtime
  static decorators = [{
    type: Component,
    args: [{
      selector: 'app-user',
      template: `
        <div class="user">
          <h2>{{ user.name }}</h2>
          <p>{{ user.email }}</p>
        </div>
      `
    }]
  }];
}
// Template string shipped to browser
// Compiler ships to browser (~500KB)
// Compilation happens in browser

// AOT Output (Build-Time Compilation)
class UserComponent {
  user: User;
  
  static ɵfac = function UserComponent_Factory(t) {
    return new (t || UserComponent)();
  };
  
  static ɵcmp = defineComponent({
    type: UserComponent,
    selectors: [['app-user']],
    inputs: { user: 'user' },
    decls: 5,
    vars: 2,
    template: function UserComponent_Template(rf, ctx) {
      if (rf & 1) {
        elementStart(0, 'div', 0);
          elementStart(1, 'h2');
            text(2);
          elementEnd();
          elementStart(3, 'p');
            text(4);
          elementEnd();
        elementEnd();
      }
      if (rf & 2) {
        advance(2);
        textInterpolate(ctx.user.name);
        advance(2);
        textInterpolate(ctx.user.email);
      }
    },
    styles: [/* inlined styles */],
    encapsulation: 2
  });
}
// Template pre-compiled to instructions
// No compiler in browser bundle
// Instant startup
```

## AOT Compilation Pipeline

### Phase 1: Analysis

```typescript
// Angular Compiler analyzes source files
class AnalysisPhase {
  analyze(sourceFile: ts.SourceFile) {
    // 1. Find decorators
    const decorators = this.findDecorators(sourceFile);
    
    // 2. Extract metadata
    const metadata = this.extractMetadata(decorators);
    
    // 3. Build component tree
    const tree = this.buildComponentTree(metadata);
    
    // 4. Validate dependencies
    this.validateDependencies(tree);
    
    return { decorators, metadata, tree };
  }
}

// Example: Analyzing a component
const analysis = {
  componentClass: 'UserComponent',
  selector: 'app-user',
  template: {
    source: '<h1>{{ title }}</h1>',
    ast: {
      type: 'Element',
      name: 'h1',
      children: [{
        type: 'BoundText',
        expression: 'title'
      }]
    }
  },
  inputs: ['user'],
  outputs: [],
  dependencies: [CommonModule]
};
```

### Phase 2: Template Type Checking

```typescript
// Type check templates against component types
class TemplateTypeChecker {
  check(component: ComponentMetadata) {
    const template = component.template;
    const componentType = component.class;
    
    // Check property bindings
    for (const binding of template.bindings) {
      this.checkPropertyExists(binding.property, componentType);
      this.checkPropertyType(binding.value, binding.property);
    }
    
    // Check event handlers
    for (const listener of template.listeners) {
      this.checkMethodExists(listener.handler, componentType);
      this.checkMethodSignature(listener.event, listener.handler);
    }
  }
}

// Example errors caught at compile time
@Component({
  template: `
    <div>{{ userName }}</div>
    <!-- ERROR: Property 'userName' does not exist on type 'UserComponent' -->
    
    <button (click)="save()">Save</button>
    <!-- ERROR: Property 'save' does not exist on type 'UserComponent' -->
    
    <input [value]="age">
    <!-- ERROR: Type 'number' is not assignable to type 'string' -->
  `
})
export class UserComponent {
  name: string;
  age: number;
}
```

### Phase 3: Code Generation

```typescript
// Generate optimized JavaScript
class CodeGenerator {
  generate(analysis: AnalysisResult) {
    // 1. Generate component definition
    const componentDef = this.generateComponentDef(analysis);
    
    // 2. Generate template function
    const templateFn = this.generateTemplate(analysis.template);
    
    // 3. Generate factory function
    const factoryFn = this.generateFactory(analysis);
    
    // 4. Generate styles
    const styles = this.generateStyles(analysis.styles);
    
    return {
      componentDef,
      templateFn,
      factoryFn,
      styles
    };
  }
  
  generateTemplate(ast: TemplateAST): string {
    let code = 'function Template(rf, ctx) {\n';
    
    // Create phase
    code += '  if (rf & 1) {\n';
    code += this.generateCreateInstructions(ast);
    code += '  }\n';
    
    // Update phase
    code += '  if (rf & 2) {\n';
    code += this.generateUpdateInstructions(ast);
    code += '  }\n';
    
    code += '}';
    return code;
  }
}
```

### Phase 4: Optimization

```typescript
// Tree shaking and optimization
class Optimizer {
  optimize(code: GeneratedCode) {
    // 1. Dead code elimination
    this.removeUnusedCode(code);
    
    // 2. Constant folding
    this.foldConstants(code);
    
    // 3. Inline small functions
    this.inlineFunctions(code);
    
    // 4. Minification preparation
    this.prepareForMinification(code);
  }
}

// Example optimizations
// Before:
const value = 10 * 2;
if (true) {
  console.log(value);
}

// After:
console.log(20); // Constant folded, dead branch removed
```

## ngc: The Angular Compiler

### Compiler Configuration

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "lib": ["ES2022", "dom"],
    "moduleResolution": "node",
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  },
  "angularCompilerOptions": {
    // Enable strict template checking
    "strictTemplates": true,
    "strictInjectionParameters": true,
    
    // Full template type check
    "fullTemplateTypeCheck": true,
    "strictInputAccessModifiers": true,
    "strictNullInputTypes": true,
    
    // Optimization options
    "enableResourceInlining": true,
    "enableIvy": true,
    
    // Debugging
    "generateCodeForLibraries": false,
    "preserveWhitespaces": false,
    
    // Locale
    "i18nInFormat": "xlf2",
    "i18nOutFormat": "xlf2"
  }
}
```

### Compilation Process

```typescript
// Running the compiler
import { NgCompiler } from '@angular/compiler-cli';

class BuildProcess {
  async compile() {
    // 1. Initialize compiler
    const compiler = NgCompiler.create({
      rootNames: ['./src/main.ts'],
      options: this.compilerOptions
    });
    
    // 2. Run analysis
    console.log('Analyzing...');
    const diagnostics = compiler.getDiagnostics();
    if (diagnostics.length > 0) {
      this.reportErrors(diagnostics);
      return;
    }
    
    // 3. Emit files
    console.log('Generating code...');
    const emitResult = compiler.emit();
    
    // 4. Report results
    console.log(`Compiled ${emitResult.emittedFiles.length} files`);
  }
}
```

### Incremental Compilation

```typescript
// Ivy enables incremental compilation
class IncrementalCompiler {
  private cache = new Map<string, CompiledFile>();
  
  compile(changedFiles: string[]) {
    const toCompile: string[] = [];
    
    // Check which files need recompilation
    for (const file of changedFiles) {
      const cached = this.cache.get(file);
      
      if (!cached || this.hasChanged(file, cached)) {
        toCompile.push(file);
        // Also recompile dependents
        toCompile.push(...this.getDependents(file));
      }
    }
    
    // Only compile what changed
    console.log(`Recompiling ${toCompile.length} files`);
    const results = this.compileFiles(toCompile);
    
    // Update cache
    for (const result of results) {
      this.cache.set(result.filename, result);
    }
  }
}

// Performance improvement
// Full build: 45s
// Incremental build (1 component changed): 3s
// 15x faster!
```

## Template Compilation Steps

### Parsing

```typescript
// Step 1: Parse template string to AST
const template = '<div [class.active]="isActive">{{ content }}</div>';

// Lexer tokenizes
const tokens = [
  { type: 'TAG_OPEN', value: 'div' },
  { type: 'ATTR_NAME', value: '[class.active]' },
  { type: 'ATTR_VALUE', value: 'isActive' },
  { type: 'TAG_CLOSE', value: '>' },
  { type: 'INTERPOLATION', value: '{{ content }}' },
  { type: 'TAG_CLOSE', value: '</div>' }
];

// Parser builds AST
const ast = {
  type: 'Element',
  name: 'div',
  attributes: [],
  properties: [
    {
      type: 'PropertyBinding',
      name: 'class.active',
      expression: 'isActive'
    }
  ],
  children: [
    {
      type: 'BoundText',
      expression: 'content'
    }
  ]
};
```

### Template Validation

```typescript
// Step 2: Validate template
class TemplateValidator {
  validate(ast: TemplateAST, component: ComponentClass) {
    // Check all property bindings
    for (const node of ast.nodes) {
      if (node.type === 'PropertyBinding') {
        const property = node.expression;
        
        // Does property exist on component?
        if (!this.hasProperty(component, property)) {
          throw new Error(
            `Property '${property}' does not exist on component`
          );
        }
        
        // Is type compatible?
        const expectedType = this.getPropertyType(node.name);
        const actualType = this.getExpressionType(property, component);
        
        if (!this.isCompatible(expectedType, actualType)) {
          throw new Error(
            `Type '${actualType}' is not assignable to type '${expectedType}'`
          );
        }
      }
    }
  }
}
```

### Instruction Generation

```typescript
// Step 3: Generate Ivy instructions
class InstructionGenerator {
  generate(ast: TemplateAST) {
    const instructions: Instruction[] = [];
    let index = 0;
    
    for (const node of ast.nodes) {
      switch (node.type) {
        case 'Element':
          instructions.push({
            phase: 'create',
            fn: 'elementStart',
            args: [index++, node.name, this.getAttrs(node)]
          });
          
          // Generate children
          instructions.push(...this.generate(node.children));
          
          instructions.push({
            phase: 'create',
            fn: 'elementEnd',
            args: []
          });
          break;
          
        case 'PropertyBinding':
          instructions.push({
            phase: 'update',
            fn: 'property',
            args: [node.name, node.expression]
          });
          break;
          
        case 'BoundText':
          instructions.push({
            phase: 'create',
            fn: 'text',
            args: [index++]
          });
          instructions.push({
            phase: 'update',
            fn: 'textInterpolate',
            args: [node.expression]
          });
          break;
      }
    }
    
    return instructions;
  }
}
```

### Code Emission

```typescript
// Step 4: Emit JavaScript code
class CodeEmitter {
  emit(instructions: Instruction[]) {
    const createPhase = instructions
      .filter(i => i.phase === 'create')
      .map(i => this.emitInstruction(i))
      .join('\n    ');
      
    const updatePhase = instructions
      .filter(i => i.phase === 'update')
      .map(i => this.emitInstruction(i))
      .join('\n    ');
    
    return `
function Template(rf, ctx) {
  if (rf & 1) {
    ${createPhase}
  }
  if (rf & 2) {
    ${updatePhase}
  }
}`;
  }
  
  emitInstruction(instruction: Instruction): string {
    const args = instruction.args
      .map(arg => this.emitArg(arg))
      .join(', ');
    return `${instruction.fn}(${args});`;
  }
}
```

## Tree Shaking with AOT

### How It Works

```typescript
// Component A uses only NgIf
@Component({
  template: '<div *ngIf="show">Content</div>',
  imports: [NgIf]
})
export class ComponentA {
  show = true;
}

// Component B uses only NgFor
@Component({
  template: '<div *ngFor="let item of items">{{ item }}</div>',
  imports: [NgFor]
})
export class ComponentB {
  items = [1, 2, 3];
}

// Bundle Analysis
// ComponentA bundle includes:
// ✓ ComponentA code
// ✓ NgIf directive
// ✗ NgFor directive (tree-shaken)
// ✗ NgSwitch directive (tree-shaken)

// ComponentB bundle includes:
// ✓ ComponentB code
// ✓ NgFor directive
// ✗ NgIf directive (tree-shaken)
// ✗ NgSwitch directive (tree-shaken)
```

### Static Analysis

```typescript
// AOT compiler performs static analysis
class TreeShaker {
  analyze(component: ComponentDef) {
    const used = new Set<string>();
    
    // 1. Find all imported dependencies
    for (const imp of component.imports) {
      used.add(imp.name);
    }
    
    // 2. Find all referenced directives in template
    for (const directive of component.template.directives) {
      used.add(directive.name);
    }
    
    // 3. Mark for inclusion
    return used;
  }
  
  shake(bundle: Bundle) {
    const included = new Set<Module>();
    
    // Start from root component
    this.markUsed(bundle.root, included);
    
    // Remove unused modules
    for (const module of bundle.modules) {
      if (!included.has(module)) {
        bundle.remove(module);
      }
    }
  }
}
```

### Dead Code Elimination

```typescript
// Before tree shaking
export class MyComponent {
  // Used in template
  publicMethod() {
    return this.getValue();
  }
  
  // Never called anywhere
  privateMethod() {
    console.log('Unused');
  }
  
  // Used by publicMethod
  private getValue() {
    return 42;
  }
}

// After tree shaking
export class MyComponent {
  publicMethod() {
    return this.getValue();
  }
  
  // privateMethod removed - dead code
  
  private getValue() {
    return 42;
  }
}
```

## Build Optimization

### Bundle Size Comparison

```bash
# Development (JIT)
main.js                   2.5 MB
vendor.js                 3.2 MB (includes compiler)
polyfills.js              500 KB
Total:                    6.2 MB

# Production (AOT + optimization)
main.[hash].js            450 KB (gzipped: 120 KB)
vendor.[hash].js          800 KB (gzipped: 180 KB)
polyfills.[hash].js       120 KB (gzipped: 35 KB)
Total:                    1.37 MB (gzipped: 335 KB)

# Savings: 78% smaller (80% gzipped)
```

### Optimization Flags

```json
// angular.json
{
  "projects": {
    "app": {
      "architect": {
        "build": {
          "configurations": {
            "production": {
              // Enable AOT
              "aot": true,
              
              // Build optimizer
              "buildOptimizer": true,
              
              // Optimization level
              "optimization": {
                "scripts": true,
                "styles": {
                  "minify": true,
                  "inlineCritical": true
                },
                "fonts": true
              },
              
              // Output hashing
              "outputHashing": "all",
              
              // Source maps
              "sourceMap": false,
              
              // Named chunks
              "namedChunks": false,
              
              // Vendor chunk
              "vendorChunk": true,
              
              // Common chunk
              "commonChunk": true,
              
              // Budget limits
              "budgets": [
                {
                  "type": "initial",
                  "maximumWarning": "500kb",
                  "maximumError": "1mb"
                }
              ]
            }
          }
        }
      }
    }
  }
}
```

### Lazy Loading Optimization

```typescript
// Routes with lazy loading
const routes: Routes = [
  {
    path: '',
    component: HomeComponent
  },
  {
    path: 'admin',
    loadComponent: () => import('./admin/admin.component')
      .then(m => m.AdminComponent)
  },
  {
    path: 'dashboard',
    loadChildren: () => import('./dashboard/dashboard.routes')
      .then(m => m.DASHBOARD_ROUTES)
  }
];

// AOT compiles each lazy chunk separately
// Bundle analysis:
// main.js: 120 KB (home + routing)
// admin-[hash].js: 45 KB (loaded on demand)
// dashboard-[hash].js: 78 KB (loaded on demand)

// Initial load: 120 KB (not 243 KB)
```

## Common Misconceptions

### "AOT is only for production"

**Reality**: While true historically, modern Angular uses AOT by default for development too (in JIT mode but with AOT optimizations). The compiler is optimized to be fast enough for development:

```bash
# Modern Angular (v9+)
ng serve  # Uses AOT
ng build  # Uses AOT
ng test   # Uses AOT (optional)
```

### "AOT makes builds slower"

**Reality**: Initial builds are slightly slower, but incremental builds are much faster:

```bash
# First build
JIT: 8s
AOT: 12s

# Rebuild after change (incremental)
JIT: 6s
AOT: 2s (thanks to caching)
```

### "JIT is better for development"

**Reality**: AOT provides better development experience:
- Catches template errors at build time
- Type-safe templates
- Consistent behavior between dev and prod
- Faster browser refresh (no runtime compilation)

## Performance Implications

### Startup Time

```typescript
// Measured on medium-sized app
// JIT (with runtime compilation)
Bootstrap: 1200ms
  - Download: 400ms
  - Compile: 650ms (in browser)
  - Render: 150ms

// AOT (pre-compiled)
Bootstrap: 550ms
  - Download: 400ms
  - Compile: 0ms (already compiled)
  - Render: 150ms

// Improvement: 54% faster startup
```

### Runtime Performance

```typescript
// Change detection cycles
// JIT: Slightly slower (interpreted metadata)
// AOT: Faster (native JavaScript)

// Benchmark: 1000 components, 100 CD cycles
// JIT: 850ms
// AOT: 720ms
// Improvement: 15% faster
```

### Memory Usage

```
JIT: Higher memory (compiler + app)
- Compiler: ~50MB
- App: ~30MB
- Total: ~80MB

AOT: Lower memory (app only)
- App: ~30MB
- Total: ~30MB

Savings: 63% less memory
```

## Interview Questions

### Q1: What are the main differences between AOT and JIT compilation?

**Answer**: AOT compiles templates during the build process before deployment, while JIT compiles them in the browser at runtime. Key differences:

**AOT**:
- Compiles during build (slower builds, faster runtime)
- Smaller bundles (no compiler shipped)
- Catches template errors at build time
- Better security (no eval, templates pre-compiled)
- Faster startup (~50% improvement)

**JIT**:
- Compiles in browser (faster builds, slower runtime)
- Larger bundles (includes compiler ~500KB)
- Template errors caught at runtime
- Uses eval for compilation
- Slower startup (compilation + bootstrap)

Modern Angular uses AOT by default for both development and production.

### Q2: What are the phases of AOT compilation?

**Answer**: AOT compilation has four main phases:

**1. Analysis**: Parse decorators, extract metadata, build component tree, validate dependencies

**2. Type Checking**: Validate template bindings against component types, check property exists, verify type compatibility, validate event handlers

**3. Code Generation**: Generate component definitions (ɵcmp), create template functions with Ivy instructions, generate factory functions (ɵfac), inline styles

**4. Optimization**: Dead code elimination, tree shaking, constant folding, minification preparation

Each phase can be cached for incremental compilation, making rebuilds significantly faster.

### Q3: How does AOT enable tree shaking?

**Answer**: AOT generates static, analyzable JavaScript code that build tools can tree-shake. The compiler:

1. **Converts templates to code**: Replaces template strings with function calls (Ivy instructions)
2. **Generates static imports**: Creates explicit import statements for dependencies
3. **Eliminates unused code**: Build tools can see what's actually imported and used
4. **Removes decorators**: Converts decorator metadata to static definitions

For example, if you only use NgIf, the compiler generates code that imports only NgIf, not the entire CommonModule. Build tools then exclude NgFor, NgSwitch, etc. from the final bundle. View Engine couldn't tree-shake effectively because it used runtime metadata that couldn't be statically analyzed.

### Q4: What is the buildOptimizer and what does it do?

**Answer**: BuildOptimizer is an Angular-specific webpack plugin that performs advanced optimizations on AOT-compiled code:

1. **Removes Angular decorators**: Converts `@Component()` etc. to static properties, reducing code size
2. **Removes constructor parameter metadata**: Already encoded in ɵfac, no need for duplication
3. **Eliminates side-effect imports**: Marks pure modules for better tree-shaking
4. **Inlines constants**: Replaces constant references with their values
5. **Removes dead code**: Eliminates unreachable code paths

It works in conjunction with AOT and typically reduces bundle size by an additional 10-15%. Enable with `"buildOptimizer": true` in angular.json for production builds.

### Q5: How does incremental compilation work in Ivy?

**Answer**: Ivy's incremental compilation uses the locality principle - each component compiles independently:

1. **Caches compiled output**: Stores generated code for each file
2. **Tracks dependencies**: Knows which files depend on others
3. **Detects changes**: Compares file hashes to determine what changed
4. **Recompiles minimally**: Only recompiles changed files and their dependents
5. **Reuses cached output**: Uses previously compiled code for unchanged files

Example: If you change ComponentA that's imported by ComponentB, only A and B recompile. ComponentC, D, E stay cached. This makes rebuilds 10-15x faster (45s full build → 3s incremental). View Engine required full recompilation because components depended on global module metadata.

### Q6: What template errors does AOT catch that JIT doesn't?

**Answer**: AOT catches many template errors at build time that JIT only finds at runtime (when that code path executes):

1. **Property typos**: `{{ usrName }}` instead of `{{ userName }}`
2. **Method doesn't exist**: `(click)="submitt()"` typo
3. **Type mismatches**: `[value]="age"` binding number to string input
4. **Wrong arity**: `transform(value, arg1, arg2)` but pipe expects 1 arg
5. **Structural directive errors**: `*ngIf="items"` when items could be undefined
6. **Unused imports**: Importing directives never used in template

With strict templates (`strictTemplates: true`), AOT provides TypeScript-level type safety for templates, catching errors before deployment. JIT only catches these when the specific code path executes, potentially in production.

### Q7: Why is AOT more secure than JIT?

**Answer**: AOT provides better security because:

1. **No eval**: JIT uses `eval()` and `Function()` to compile templates at runtime, which violates Content Security Policy (CSP). AOT pre-compiles everything, requiring no eval.

2. **No compiler in browser**: JIT ships the Angular compiler (~500KB) to the browser, increasing attack surface. AOT ships only the runtime.

3. **Template injection prevention**: Since templates are pre-compiled, attackers can't inject malicious templates at runtime

4. **CSP compliance**: AOT-compiled apps can use strict CSP headers (`script-src 'self'`) without needing `unsafe-eval`

5. **Less dynamic code**: All code paths are known at build time, making security auditing easier

This is why all production deployments should use AOT.

### Q8: What is the performance impact of AOT compilation?

**Answer**: AOT significantly improves runtime performance:

**Startup time**: 40-60% faster (no browser compilation)
**Bundle size**: 30-40% smaller (no compiler, better tree-shaking)
**Memory**: 50-65% less (no compiler in memory)
**First contentful paint**: Improved by 2-3 seconds on average
**Change detection**: 10-15% faster (native code vs interpreted)

Trade-offs:
**Build time**: 30-50% slower for full builds, but incremental builds are 10x faster
**Developer experience**: Better (catches errors early)

Real-world example: Medium app goes from 6.2MB (JIT) to 1.4MB (AOT), startup from 3.3s to 1.3s, 60% improvement in initial load time.

## Key Takeaways

1. AOT compiles templates at build time for smaller bundles and faster startup
2. Ivy's locality principle enables efficient incremental compilation
3. AOT catches template errors at compile time with strict type checking
4. Tree shaking removes unused code thanks to static analysis
5. Build optimizer performs Angular-specific optimizations on top of AOT
6. Production builds should always use AOT for performance and security
7. Modern Angular uses AOT by default for both development and production
8. Incremental builds with Ivy are 10-15x faster than full rebuilds

## Resources

- [Angular AOT Compilation](https://angular.dev/tools/cli/aot-compiler)
- [Angular Compiler Options](https://angular.dev/reference/configs/angular-compiler-options)
- [Ahead-of-Time Compilation Guide](https://angular.io/guide/aot-compiler)
- [Build Optimizer](https://github.com/angular/angular-cli/tree/main/packages/angular_devkit/build_optimizer)
- [Template Type Checking](https://angular.dev/tools/cli/template-typecheck)
- [Angular Build Performance](https://blog.angular.io/angular-build-performance-1e2f7baeb9e8)
- [Ivy Compilation and Runtime](https://blog.angular.io/all-you-need-to-know-about-ivy-the-new-angular-engine-9cde471f42cf)

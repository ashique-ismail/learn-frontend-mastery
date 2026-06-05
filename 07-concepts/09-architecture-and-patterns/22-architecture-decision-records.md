# Architecture Decision Records (ADRs)

## The Idea

**In plain English:** An Architecture Decision Record (ADR) is a short document that captures an important choice your software team made — what they decided, why they decided it, and what trade-offs came with that choice. Think of it as a permanent sticky note that future teammates can read to understand why the project is built the way it is.

**Real-world analogy:** Imagine a city council deciding whether to build a new bridge or widen an existing road to reduce traffic. They hold a meeting, weigh the options, vote, and file official meeting minutes explaining the outcome and reasoning.

- The meeting minutes = the ADR document
- The traffic problem they needed to solve = the context section
- The final vote to build the bridge = the decision section
- The notes about increased construction cost but faster commute times = the consequences section

---

## Overview

Architecture Decision Records (ADRs) are documents that capture important architectural decisions made during the development of a software system, along with their context and consequences. ADRs provide a historical record of why certain technical choices were made, helping teams understand past decisions and avoid repeating discussions.

ADRs are a lightweight way to document architecture that becomes part of your project's knowledge base, making it easier for new team members to understand the system and for current team members to remember why specific choices were made.

## Core Concepts

### 1. ADR Structure

```
┌──────────────────────────────────────────────┐
│         Architecture Decision Record          │
├──────────────────────────────────────────────┤
│  Title: Short, descriptive name              │
│  Status: Proposed | Accepted | Deprecated    │
│  Date: When the decision was made            │
├──────────────────────────────────────────────┤
│  Context: The problem or opportunity         │
│  ├─ Business drivers                         │
│  ├─ Technical constraints                    │
│  └─ Current situation                        │
├──────────────────────────────────────────────┤
│  Decision: What was decided                  │
│  ├─ Chosen solution                          │
│  ├─ Key aspects                              │
│  └─ Implementation approach                  │
├──────────────────────────────────────────────┤
│  Consequences: Impacts and trade-offs        │
│  ├─ Positive outcomes                        │
│  ├─ Negative outcomes                        │
│  ├─ Risks                                    │
│  └─ Follow-up actions                        │
└──────────────────────────────────────────────┘
```

### 2. ADR Lifecycle

```
Proposed → Accepted → Active
              ↓
         Superseded → Deprecated
              ↓
          Archived
```

## Implementation Patterns

### Basic ADR Template

```markdown
# ADR-001: Use React for Frontend Framework

## Status

Accepted

## Date

2024-01-15

## Context

We need to choose a frontend framework for our new e-commerce platform. The 
application will be complex with:
- Dynamic product catalog with 10,000+ items
- Real-time inventory updates
- User authentication and personalization
- Mobile-responsive design
- Team has varied experience levels

### Constraints
- Must support modern browsers (Chrome, Firefox, Safari, Edge)
- Need strong TypeScript support
- Must have good developer tooling
- Need active community and ecosystem

### Options Considered
1. React
2. Angular
3. Vue.js
4. Svelte

## Decision

We will use React as our frontend framework.

### Rationale

**React chosen because:**

1. **Component Ecosystem**: Largest ecosystem of third-party components
   - Material-UI, Ant Design, Chakra UI available
   - Reduces custom component development time

2. **Team Experience**: 4 of 6 developers have React experience
   - Faster onboarding for remaining team members
   - Established internal patterns from previous projects

3. **TypeScript Support**: Excellent TypeScript integration
   - Strong typing with React.FC, hooks
   - Good IDE support with IntelliSense

4. **Flexibility**: Can choose state management, routing separately
   - Not locked into framework decisions
   - Can evolve architecture as needs change

5. **Performance**: Virtual DOM handles large product lists well
   - React.memo for expensive components
   - Concurrent features for better UX

### Why Not Others

- **Angular**: Too opinionated, steeper learning curve, overkill for project
- **Vue**: Smaller ecosystem, less TypeScript support, team unfamiliarity
- **Svelte**: Immature ecosystem, risky for large project, hard to hire

## Consequences

### Positive

- Fast development with reusable component library
- Team can start immediately with existing React knowledge
- Large community for problem-solving
- Easy to find contractors/hires with React experience
- Gradual learning path for junior developers

### Negative

- Need to make additional decisions (state management, routing)
- More boilerplate than Vue or Svelte
- Can lead to inconsistent patterns without strong guidelines
- Bundle size larger than Svelte

### Risks

- React's license and governance changes (mitigation: monitor Meta's policies)
- Fragmentation in ecosystem (mitigation: establish approved libraries list)
- Performance issues with large lists (mitigation: virtualization, pagination)

### Follow-up Decisions Needed

- ADR-002: State management solution (Redux, MobX, Zustand)
- ADR-003: Routing library (React Router, TanStack Router)
- ADR-004: UI component library (Material-UI, Ant Design, custom)

## References

- [React Documentation](https://react.dev)
- [TypeScript with React](https://react-typescript-cheatsheet.netlify.app)
- Internal team skill assessment (2024-01-10)
- Framework comparison matrix (docs/framework-comparison.xlsx)
```

### ADR Template File

```markdown
# ADR-[NUMBER]: [Title]

## Status

[Proposed | Accepted | Deprecated | Superseded]

## Date

[YYYY-MM-DD]

## Context

[Describe the problem or opportunity]

### Background
[Additional context about the situation]

### Constraints
[Technical, business, or other constraints]

### Options Considered
1. [Option 1]
2. [Option 2]
3. [Option 3]

## Decision

[State the decision clearly]

### Rationale
[Explain why this decision was made]

### Implementation Approach
[How the decision will be implemented]

## Consequences

### Positive
[Benefits and advantages]

### Negative
[Drawbacks and disadvantages]

### Risks
[Potential risks and mitigation strategies]

## References

[Links to relevant documentation, discussions, or resources]
```

### React Component for Viewing ADRs

```typescript
// ADRViewer.tsx
import React, { useState, useEffect } from 'react';
import ReactMarkdown from 'react-markdown';
import { Prism as SyntaxHighlighter } from 'react-syntax-highlighter';

interface ADR {
  id: string;
  number: number;
  title: string;
  status: 'proposed' | 'accepted' | 'deprecated' | 'superseded';
  date: string;
  content: string;
  tags: string[];
}

const ADRViewer: React.FC = () => {
  const [adrs, setAdrs] = useState<ADR[]>([]);
  const [selectedADR, setSelectedADR] = useState<ADR | null>(null);
  const [filter, setFilter] = useState<string>('all');
  const [search, setSearch] = useState<string>('');

  useEffect(() => {
    // Load ADRs from API or file system
    loadADRs();
  }, []);

  const loadADRs = async () => {
    // In real app, fetch from API
    const response = await fetch('/api/adrs');
    const data = await response.json();
    setAdrs(data);
  };

  const filteredADRs = adrs.filter(adr => {
    const matchesFilter = filter === 'all' || adr.status === filter;
    const matchesSearch = 
      adr.title.toLowerCase().includes(search.toLowerCase()) ||
      adr.content.toLowerCase().includes(search.toLowerCase());
    return matchesFilter && matchesSearch;
  });

  return (
    <div className="adr-viewer">
      <aside className="adr-sidebar">
        <div className="adr-controls">
          <input
            type="search"
            placeholder="Search ADRs..."
            value={search}
            onChange={e => setSearch(e.target.value)}
          />
          
          <select 
            value={filter} 
            onChange={e => setFilter(e.target.value)}
          >
            <option value="all">All Status</option>
            <option value="proposed">Proposed</option>
            <option value="accepted">Accepted</option>
            <option value="deprecated">Deprecated</option>
          </select>
        </div>

        <ul className="adr-list">
          {filteredADRs.map(adr => (
            <li
              key={adr.id}
              className={selectedADR?.id === adr.id ? 'active' : ''}
              onClick={() => setSelectedADR(adr)}
            >
              <div className="adr-list-item">
                <span className={`status-badge ${adr.status}`}>
                  {adr.status}
                </span>
                <span className="adr-number">ADR-{adr.number}</span>
                <span className="adr-title">{adr.title}</span>
                <span className="adr-date">{adr.date}</span>
              </div>
            </li>
          ))}
        </ul>
      </aside>

      <main className="adr-content">
        {selectedADR ? (
          <ADRContent adr={selectedADR} />
        ) : (
          <div className="adr-empty">
            Select an ADR to view its details
          </div>
        )}
      </main>
    </div>
  );
};

const ADRContent: React.FC<{ adr: ADR }> = ({ adr }) => {
  return (
    <article className="adr-document">
      <header className="adr-header">
        <h1>ADR-{adr.number}: {adr.title}</h1>
        <div className="adr-meta">
          <span className={`status ${adr.status}`}>{adr.status}</span>
          <span className="date">{adr.date}</span>
        </div>
        <div className="adr-tags">
          {adr.tags.map(tag => (
            <span key={tag} className="tag">{tag}</span>
          ))}
        </div>
      </header>

      <ReactMarkdown
        components={{
          code({ node, inline, className, children, ...props }) {
            const match = /language-(\w+)/.exec(className || '');
            return !inline && match ? (
              <SyntaxHighlighter language={match[1]} {...props}>
                {String(children).replace(/\n$/, '')}
              </SyntaxHighlighter>
            ) : (
              <code className={className} {...props}>
                {children}
              </code>
            );
          }
        }}
      >
        {adr.content}
      </ReactMarkdown>
    </article>
  );
};

export default ADRViewer;
```

### Angular ADR Service

```typescript
// adr.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, BehaviorSubject } from 'rxjs';
import { map, shareReplay } from 'rxjs/operators';

export interface ADR {
  id: string;
  number: number;
  title: string;
  status: ADRStatus;
  date: Date;
  content: string;
  tags: string[];
  supersedes?: string[];
  supersededBy?: string;
}

export type ADRStatus = 'proposed' | 'accepted' | 'deprecated' | 'superseded';

@Injectable({ providedIn: 'root' })
export class ADRService {
  private adrsSubject = new BehaviorSubject<ADR[]>([]);
  public adrs$ = this.adrsSubject.asObservable();

  constructor(private http: HttpClient) {
    this.loadADRs();
  }

  private loadADRs(): void {
    this.http.get<ADR[]>('/api/adrs')
      .pipe(
        map(adrs => adrs.sort((a, b) => b.number - a.number))
      )
      .subscribe(adrs => this.adrsSubject.next(adrs));
  }

  getADR(id: string): Observable<ADR | undefined> {
    return this.adrs$.pipe(
      map(adrs => adrs.find(adr => adr.id === id))
    );
  }

  getADRByNumber(number: number): Observable<ADR | undefined> {
    return this.adrs$.pipe(
      map(adrs => adrs.find(adr => adr.number === number))
    );
  }

  getADRsByStatus(status: ADRStatus): Observable<ADR[]> {
    return this.adrs$.pipe(
      map(adrs => adrs.filter(adr => adr.status === status))
    );
  }

  searchADRs(query: string): Observable<ADR[]> {
    const lowerQuery = query.toLowerCase();
    return this.adrs$.pipe(
      map(adrs => adrs.filter(adr =>
        adr.title.toLowerCase().includes(lowerQuery) ||
        adr.content.toLowerCase().includes(lowerQuery) ||
        adr.tags.some(tag => tag.toLowerCase().includes(lowerQuery))
      ))
    );
  }

  createADR(adr: Omit<ADR, 'id' | 'number'>): Observable<ADR> {
    return this.http.post<ADR>('/api/adrs', adr).pipe(
      map(newADR => {
        const current = this.adrsSubject.value;
        this.adrsSubject.next([newADR, ...current]);
        return newADR;
      })
    );
  }

  updateADRStatus(id: string, status: ADRStatus): Observable<ADR> {
    return this.http.patch<ADR>(`/api/adrs/${id}`, { status }).pipe(
      map(updated => {
        const current = this.adrsSubject.value;
        const index = current.findIndex(adr => adr.id === id);
        if (index !== -1) {
          current[index] = updated;
          this.adrsSubject.next([...current]);
        }
        return updated;
      })
    );
  }
}

// adr-list.component.ts
@Component({
  selector: 'app-adr-list',
  template: `
    <div class="adr-list">
      <div class="filters">
        <input 
          type="search" 
          placeholder="Search ADRs..."
          (input)="onSearch($event)"
        />
        
        <select (change)="onFilterChange($event)">
          <option value="all">All Status</option>
          <option value="proposed">Proposed</option>
          <option value="accepted">Accepted</option>
          <option value="deprecated">Deprecated</option>
        </select>
      </div>

      <div class="adr-items">
        <app-adr-card
          *ngFor="let adr of filteredADRs$ | async"
          [adr]="adr"
          (click)="selectADR(adr)"
        ></app-adr-card>
      </div>
    </div>
  `
})
export class ADRListComponent implements OnInit {
  filteredADRs$!: Observable<ADR[]>;
  private searchTerm = '';
  private statusFilter = 'all';

  constructor(private adrService: ADRService, private router: Router) {}

  ngOnInit(): void {
    this.updateFilter();
  }

  onSearch(event: Event): void {
    this.searchTerm = (event.target as HTMLInputElement).value;
    this.updateFilter();
  }

  onFilterChange(event: Event): void {
    this.statusFilter = (event.target as HTMLSelectElement).value;
    this.updateFilter();
  }

  private updateFilter(): void {
    if (this.searchTerm) {
      this.filteredADRs$ = this.adrService.searchADRs(this.searchTerm);
    } else if (this.statusFilter !== 'all') {
      this.filteredADRs$ = this.adrService.getADRsByStatus(
        this.statusFilter as ADRStatus
      );
    } else {
      this.filteredADRs$ = this.adrService.adrs$;
    }
  }

  selectADR(adr: ADR): void {
    this.router.navigate(['/adrs', adr.id]);
  }
}
```

### ADR CLI Tool

```typescript
// adr-cli.ts
import fs from 'fs/promises';
import path from 'path';
import { format } from 'date-fns';

interface ADRConfig {
  directory: string;
  template: string;
}

export class ADRCLI {
  private config: ADRConfig;

  constructor(configPath?: string) {
    this.config = this.loadConfig(configPath);
  }

  private loadConfig(configPath?: string): ADRConfig {
    // Load from config file or use defaults
    return {
      directory: 'docs/adr',
      template: 'default'
    };
  }

  async init(): Promise<void> {
    // Create ADR directory
    await fs.mkdir(this.config.directory, { recursive: true });

    // Create template directory
    const templateDir = path.join(this.config.directory, 'templates');
    await fs.mkdir(templateDir, { recursive: true });

    // Create default template
    const defaultTemplate = this.getDefaultTemplate();
    await fs.writeFile(
      path.join(templateDir, 'default.md'),
      defaultTemplate
    );

    console.log(`Initialized ADR directory at ${this.config.directory}`);
  }

  async create(title: string, tags: string[] = []): Promise<void> {
    // Get next ADR number
    const number = await this.getNextNumber();

    // Create filename
    const filename = this.formatFilename(number, title);
    const filepath = path.join(this.config.directory, filename);

    // Load template
    const template = await this.loadTemplate(this.config.template);

    // Fill template
    const content = this.fillTemplate(template, {
      number,
      title,
      date: format(new Date(), 'yyyy-MM-dd'),
      tags
    });

    // Write file
    await fs.writeFile(filepath, content);

    console.log(`Created ${filename}`);
  }

  async list(status?: string): Promise<void> {
    const files = await fs.readdir(this.config.directory);
    const adrFiles = files.filter(f => f.match(/^\d{4}-.*\.md$/));

    for (const file of adrFiles) {
      const filepath = path.join(this.config.directory, file);
      const content = await fs.readFile(filepath, 'utf-8');
      
      const statusMatch = content.match(/##\s+Status\s+(\w+)/);
      const currentStatus = statusMatch ? statusMatch[1].toLowerCase() : 'unknown';

      if (!status || status === currentStatus) {
        const titleMatch = content.match(/^#\s+ADR-\d+:\s+(.+)$/m);
        const title = titleMatch ? titleMatch[1] : 'Unknown';
        
        console.log(`${file} - ${title} [${currentStatus}]`);
      }
    }
  }

  async supersede(oldNumber: number, newTitle: string): Promise<void> {
    // Create new ADR
    const newNumber = await this.getNextNumber();
    const filename = this.formatFilename(newNumber, newTitle);
    const filepath = path.join(this.config.directory, filename);

    const template = await this.loadTemplate(this.config.template);
    const content = this.fillTemplate(template, {
      number: newNumber,
      title: newTitle,
      date: format(new Date(), 'yyyy-MM-dd'),
      supersedes: oldNumber
    });

    await fs.writeFile(filepath, content);

    // Update old ADR status
    const oldFiles = await fs.readdir(this.config.directory);
    const oldFile = oldFiles.find(f => f.startsWith(`${String(oldNumber).padStart(4, '0')}-`));
    
    if (oldFile) {
      const oldPath = path.join(this.config.directory, oldFile);
      const oldContent = await fs.readFile(oldPath, 'utf-8');
      const updated = oldContent.replace(
        /##\s+Status\s+\w+/,
        `## Status\n\nSuperseded by ADR-${newNumber}`
      );
      await fs.writeFile(oldPath, updated);
    }

    console.log(`Created ${filename} (supersedes ADR-${oldNumber})`);
  }

  private async getNextNumber(): Promise<number> {
    const files = await fs.readdir(this.config.directory);
    const numbers = files
      .filter(f => f.match(/^\d{4}-/))
      .map(f => parseInt(f.substring(0, 4)));
    
    return numbers.length > 0 ? Math.max(...numbers) + 1 : 1;
  }

  private formatFilename(number: number, title: string): string {
    const slug = title
      .toLowerCase()
      .replace(/[^a-z0-9]+/g, '-')
      .replace(/^-|-$/g, '');
    
    return `${String(number).padStart(4, '0')}-${slug}.md`;
  }

  private getDefaultTemplate(): string {
    return `# ADR-{NUMBER}: {TITLE}

## Status

Proposed

## Date

{DATE}

## Context

[Describe the problem or opportunity]

### Background

### Constraints

### Options Considered

## Decision

[State the decision clearly]

### Rationale

## Consequences

### Positive

### Negative

### Risks

## References
`;
  }

  private async loadTemplate(name: string): Promise<string> {
    const templatePath = path.join(
      this.config.directory,
      'templates',
      `${name}.md`
    );
    return await fs.readFile(templatePath, 'utf-8');
  }

  private fillTemplate(
    template: string,
    data: {
      number: number;
      title: string;
      date: string;
      tags?: string[];
      supersedes?: number;
    }
  ): string {
    let content = template
      .replace(/{NUMBER}/g, String(data.number))
      .replace(/{TITLE}/g, data.title)
      .replace(/{DATE}/g, data.date);

    if (data.tags && data.tags.length > 0) {
      content += `\n\n## Tags\n\n${data.tags.map(t => `- ${t}`).join('\n')}`;
    }

    if (data.supersedes) {
      content += `\n\n## Supersedes\n\nADR-${data.supersedes}`;
    }

    return content;
  }
}

// CLI usage
const cli = new ADRCLI();

const command = process.argv[2];
const args = process.argv.slice(3);

switch (command) {
  case 'init':
    await cli.init();
    break;

  case 'new':
    const title = args.join(' ');
    await cli.create(title);
    break;

  case 'list':
    await cli.list(args[0]);
    break;

  case 'supersede':
    const oldNumber = parseInt(args[0]);
    const newTitle = args.slice(1).join(' ');
    await cli.supersede(oldNumber, newTitle);
    break;

  default:
    console.log('Usage: adr <command> [options]');
    console.log('Commands:');
    console.log('  init              Initialize ADR directory');
    console.log('  new <title>       Create new ADR');
    console.log('  list [status]     List all ADRs');
    console.log('  supersede <n> <title>  Supersede ADR');
}
```

### ADR Generation Script

```typescript
// generate-adr.ts
import { OpenAI } from 'openai';

export class ADRGenerator {
  private openai: OpenAI;

  constructor(apiKey: string) {
    this.openai = new OpenAI({ apiKey });
  }

  async generateFromDiscussion(
    discussion: string,
    options?: string[]
  ): Promise<string> {
    const prompt = `
Based on the following technical discussion, generate an Architecture Decision Record (ADR):

Discussion:
${discussion}

${options ? `Options considered:\n${options.join('\n')}` : ''}

Generate a complete ADR following this structure:
- Title
- Status (Proposed)
- Context (why this decision is needed)
- Decision (what was decided)
- Consequences (positive, negative, risks)

Be specific and technical. Include trade-offs and implementation considerations.
`;

    const response = await this.openai.chat.completions.create({
      model: 'gpt-4',
      messages: [{ role: 'user', content: prompt }],
      temperature: 0.7
    });

    return response.choices[0].message.content || '';
  }

  async suggestAlternatives(adr: string): Promise<string[]> {
    const prompt = `
Given this ADR, suggest alternative approaches that were likely considered:

${adr}

Provide 3-5 alternative technical solutions with brief descriptions.
`;

    const response = await this.openai.chat.completions.create({
      model: 'gpt-4',
      messages: [{ role: 'user', content: prompt }]
    });

    const content = response.choices[0].message.content || '';
    return content.split('\n').filter(line => line.trim().startsWith('-'));
  }
}
```

### Example ADRs

```markdown
# ADR-002: Use Redux Toolkit for State Management

## Status

Accepted

## Date

2024-01-20

## Context

Following ADR-001 (React frontend), we need to choose a state management solution.

### Requirements
- Manage complex application state (auth, cart, product catalog)
- Support async operations (API calls)
- Enable time-travel debugging
- Good TypeScript support
- Team learning curve acceptable

### Options Considered
1. Redux Toolkit
2. MobX
3. Zustand
4. React Context + useReducer
5. Jotai

## Decision

Use Redux Toolkit (RTK) for state management.

### Rationale

1. **Industry Standard**: Most common React state solution
2. **DevTools**: Excellent debugging with Redux DevTools
3. **RTK Simplification**: Modern Redux much simpler than legacy
4. **RTK Query**: Built-in data fetching eliminates separate library
5. **TypeScript**: First-class TypeScript support
6. **Middleware**: Robust async handling with thunks

### Why Not Others
- **MobX**: Different paradigm, team unfamiliar, less common
- **Zustand**: Too minimal for complex state, newer ecosystem
- **Context**: Not suitable for frequent updates, performance issues
- **Jotai**: Atomic approach unfamiliar, smaller ecosystem

## Consequences

### Positive
- Predictable state updates with actions/reducers
- Time-travel debugging for bug reproduction
- Strong patterns prevent state management issues
- RTK Query reduces boilerplate for API calls
- Easy to test with pure reducers

### Negative
- Initial setup more complex than Zustand
- More boilerplate than MobX or Context
- Learning curve for Redux concepts
- Can be over-engineered for simple state

### Risks
- Team might overuse Redux for local state (mitigation: guidelines)
- Large bundle size (mitigation: code splitting)

## References

- [Redux Toolkit Documentation](https://redux-toolkit.js.org/)
- [RTK Query](https://redux-toolkit.js.org/rtk-query/overview)
```

```markdown
# ADR-003: Implement Micro-Frontend Architecture

## Status

Proposed

## Date

2024-02-01

## Context

Our monolithic frontend has grown to 15 teams working on different features.

### Problems
- Deployment conflicts between teams
- Long build times (20+ minutes)
- Difficult to upgrade dependencies independently
- Team autonomy limited by shared codebase

### Constraints
- Must maintain consistent UX
- Cannot do full rewrite
- Need gradual migration path
- Must support IE11 for 6 more months

## Decision

Implement micro-frontend architecture using Module Federation.

### Approach

1. **Host Application** (shell)
   - Handle routing, layout, shared components
   - Load remote applications dynamically

2. **Remote Applications** (features)
   - Teams own complete features
   - Deploy independently
   - Share common dependencies

3. **Module Federation** (Webpack 5)
   - Runtime integration, no build-time coupling
   - Automatic dependency deduplication
   - Version negotiation

### Migration Strategy

Phase 1: Extract admin panel (low risk, single team)
Phase 2: Extract product catalog (high value)
Phase 3: Migrate remaining features incrementally

## Consequences

### Positive
- Teams deploy independently (10+ deploys/day possible)
- Faster builds (5 min per micro-app vs 20 min monolith)
- Technology flexibility (can use different React versions)
- Clear ownership boundaries
- Parallel development without conflicts

### Negative
- Increased complexity (multiple repos, deployments)
- Network overhead loading remote modules
- Shared component versioning challenges
- Difficult debugging across boundaries
- Need strong governance for consistency

### Risks

**Risk**: Inconsistent UX across micro-apps
**Mitigation**: Shared design system, central UI library

**Risk**: Breaking changes in shared dependencies
**Mitigation**: Semantic versioning, integration tests

**Risk**: Performance degradation from multiple bundles
**Mitigation**: Shared dependencies, performance monitoring

**Risk**: Team coordination overhead
**Mitigation**: Architecture guild, documented standards

## Implementation Details

```typescript
// host/webpack.config.js
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'host',
      remotes: {
        admin: 'admin@http://localhost:3001/remoteEntry.js',
        catalog: 'catalog@http://localhost:3002/remoteEntry.js'
      },
      shared: {
        react: { singleton: true },
        'react-dom': { singleton: true }
      }
    })
  ]
};

// admin/webpack.config.js
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'admin',
      filename: 'remoteEntry.js',
      exposes: {
        './AdminApp': './src/App'
      },
      shared: {
        react: { singleton: true },
        'react-dom': { singleton: true }
      }
    })
  ]
};
```

## Success Criteria

- 50% reduction in deployment conflicts (3 months)
- 60% reduction in build time per team (3 months)
- Zero breaking changes to end users (ongoing)
- 90% team satisfaction with independence (6 months)

## Follow-up Decisions

- ADR-004: Shared component library versioning strategy
- ADR-005: Cross-micro-frontend communication patterns
- ADR-006: Monitoring and error tracking for micro-frontends

## References

- [Micro-Frontends](https://micro-frontends.org/)
- [Module Federation](https://webpack.js.org/concepts/module-federation/)
- Internal RFC: Micro-Frontend Proposal (2024-01-15)
```

## Common Mistakes

### 1. Missing Context

```markdown
# BAD: No context
## Decision
Use TypeScript

## Consequences
Better type safety

# GOOD: Rich context
## Context
Team has grown to 12 developers with varying JS experience.
Recent production bugs from type mismatches cost 40 hours.
Need to improve code quality and developer confidence.

## Decision
Migrate to TypeScript incrementally over 6 months.
All new code in TS, convert critical paths first.

## Consequences
- Catch 30% of bugs at compile time (based on industry data)
- Better IDE support improves productivity 15%
- Initial 20% slowdown during learning curve
- Build time increases 30 seconds
```

### 2. Too Much Detail

```markdown
# BAD: Implementation details in ADR
## Decision
Use React with these exact dependencies:
- react: 18.2.0
- react-dom: 18.2.0
- typescript: 5.0.0
[50 more lines of package.json]

# GOOD: Focus on architectural decision
## Decision
Use React as frontend framework.
Specific versions and dependencies managed in package.json.
```

### 3. No Alternatives Considered

```markdown
# BAD
## Decision
We will use MongoDB.

# GOOD
## Options Considered
1. PostgreSQL - Relational, ACID, strong typing
2. MongoDB - Document, flexible schema, horizontal scaling
3. DynamoDB - Managed, serverless, AWS-native

## Decision
MongoDB chosen for flexible schema needed in early stage.
Will reassess when data model stabilizes (6 months).
```

### 4. Vague Consequences

```markdown
# BAD
## Consequences
It will be better. Users will be happier.

# GOOD
## Consequences
### Positive
- API response time reduced from 800ms to 200ms (75% improvement)
- Infrastructure cost reduced $2000/month (from $8k to $6k)
- Developer satisfaction: 8/10 (based on survey)

### Negative
- Learning curve: 2 weeks for team to learn new system
- Migration cost: 80 engineering hours over 4 weeks
- Increased complexity: 3 services instead of 1
```

## Best Practices

### 1. Start Simple

```markdown
For new projects, start with lightweight ADRs:
- Title and date
- What was decided
- Why
- Main trade-offs

Add more structure as team grows.
```

### 2. Make Decisions Reversible When Possible

```markdown
## Decision
Use Feature Flags for gradual rollout.

This decision can be reversed without major refactoring
if feature flags prove too complex.

## Rollback Plan
1. Remove feature flag checks (2 hours)
2. Deploy directly (standard process)
3. Rollback ADR status to "superseded"
```

### 3. Link Related ADRs

```markdown
## Related Decisions
- Supersedes: ADR-015 (Use REST API)
- Related: ADR-018 (API Versioning Strategy)
- Enables: ADR-020 (Real-time Updates)
```

### 4. Update ADR Status

```markdown
## Status History
- 2024-01-15: Proposed
- 2024-01-20: Accepted (after team review)
- 2024-03-01: Deprecated (see ADR-025)
```

### 5. Include Success Metrics

```markdown
## Success Criteria
We will know this decision is successful if:
- Page load time < 2s (currently 5s) - measure after 1 month
- Developer survey score > 7/10 - survey at 3 months
- Zero security incidents related to auth - monitor ongoing
- 50% reduction in auth-related support tickets - 6 months
```

## When to Create an ADR

### Create ADR For

1. **Technology Choices**: Framework, database, language
2. **Architecture Patterns**: Microservices, event-driven, layered
3. **Integration Approaches**: APIs, webhooks, message queues
4. **Development Practices**: Monorepo vs polyrepo, branching strategy
5. **Security Decisions**: Authentication method, encryption approach
6. **Infrastructure**: Cloud provider, deployment strategy

### Don't Create ADR For

1. **Temporary Experiments**: "Let's try X this sprint"
2. **Routine Tasks**: "Update React version"
3. **Team Preferences**: "Use spaces over tabs"
4. **Implementation Details**: "Name variable camelCase"
5. **Obvious Choices**: "Fix security vulnerability"

## Interview Questions

### Q1: Why are ADRs useful?

Answer: ADRs document why decisions were made, not just what was decided. They provide context for future team members, prevent re-litigating past decisions, and help teams learn from experience. They're especially valuable when team composition changes.

### Q2: How do you handle outdated ADRs?

Answer: Mark status as "Superseded" or "Deprecated" with reference to the new ADR. Never delete old ADRs - they're historical record. The superseding ADR should explain why the original decision changed.

### Q3: What's the difference between ADR and RFC?

Answer: RFC (Request for Comments) is for proposing and discussing decisions. ADR records final decisions. RFC is the conversation; ADR is the conclusion. Some teams use RFCs before ADRs, others just use ADRs.

### Q4: Should ADRs include implementation details?

Answer: Focus on architectural decisions and rationale. Reference separate technical documentation for implementation details. ADRs answer "what and why," not "how exactly."

### Q5: How do you get teams to actually write ADRs?

Answer: Make it part of the process - PR template checklist item, architecture review requirement. Keep templates simple. Show value by referencing past ADRs in discussions. Lead by example.

## Key Takeaways

1. **ADRs document architectural decisions** and their context
2. **Include context, decision, and consequences** at minimum
3. **Explain why, not just what** was decided
4. **Consider alternatives** and explain why they weren't chosen
5. **Keep ADRs lightweight** - don't over-document
6. **Never delete ADRs** - mark as superseded instead
7. **Link related ADRs** to show evolution of architecture
8. **Make it easy** - templates, CLI tools, automation
9. **Review regularly** - are old decisions still valid?
10. **ADRs are living documentation** that travels with code

## Resources

### Tools
- [adr-tools](https://github.com/npryce/adr-tools) - CLI for managing ADRs
- [log4brains](https://github.com/thomvaill/log4brains) - ADR management tool
- [ADR Manager](https://adr.github.io/) - Web-based ADR tool

### Documentation
- [ADR GitHub Organization](https://adr.github.io/)
- [Documenting Architecture Decisions](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions)
- [Architecture Decision Records (ThoughtWorks)](https://www.thoughtworks.com/radar/techniques/lightweight-architecture-decision-records)

### Templates
- [MADR](https://adr.github.io/madr/) - Markdown ADR format
- [Y-Statements](https://medium.com/olzzio/y-statements-10eb07b5a177)

### Articles
- "Architectural Decision Records" by Michael Nygard
- "ADRs: A Lightweight Way to Document Architecture" (InfoQ)
- "Why You Should Be Using ADRs" (Martin Fowler)

# System Design: Analytics Dashboard

## The Idea

**In plain English:** An analytics dashboard is a single screen that gathers numbers and statistics from different sources and displays them as charts, graphs, and tables so you can understand what is happening at a glance. Think of it like the instrument panel in a car — instead of guessing how fast you are going or how much fuel is left, all the information is in one place, updating in real time.

**Real-world analogy:** Imagine a school cafeteria notice board that staff update throughout the day — each poster shows today's meal count, money collected, and leftover food. A digital analytics dashboard works the same way, but the posters update themselves automatically every few seconds.

- The notice board = the dashboard layout that holds everything together
- Each poster on the board = a widget (a self-contained panel showing one chart, metric, or table)
- The staff member who replaces a poster with fresh numbers = the real-time data service that pushes live updates to each widget

---

## Overview

Design an analytics dashboard with real-time updates, data visualization, filtering/sorting, export capabilities, and performance optimization for large datasets.

## Requirements

### Functional Requirements

- Multiple chart types (line, bar, pie, area)
- Real-time data updates
- Date range filtering
- Drill-down capabilities
- Comparison mode
- Export (CSV, PDF, PNG)
- Custom dashboard layouts
- Responsive design

### Non-Functional Requirements

- Load < 3 seconds
- Handle 100K+ data points
- Real-time updates < 1 second
- Smooth 60fps interactions
- Support concurrent users
- Offline data caching
- Mobile responsive

## Architecture

```typescript
// Dashboard component
@Component({
  selector: 'app-dashboard',
  standalone: true,
  template: `
    <div class="dashboard-container">
      <div class="dashboard-header">
        <h1>Analytics Dashboard</h1>
        
        <div class="controls">
          <app-date-range-picker
            [(dateRange)]="dateRange"
            (change)="onDateRangeChange()">
          </app-date-range-picker>

          <button (click)="refresh()">Refresh</button>
          <button (click)="export('csv')">Export CSV</button>
          <button (click)="export('pdf')">Export PDF</button>
        </div>
      </div>

      <div class="dashboard-grid" cdkDropListGroup>
        <div
          *ngFor="let widget of widgets; trackBy: trackByWidgetId"
          class="dashboard-widget"
          [style.grid-column]="widget.position.col"
          [style.grid-row]="widget.position.row"
          cdkDrag>
          
          <div class="widget-header" cdkDragHandle>
            <h3>{{ widget.title }}</h3>
            <div class="widget-actions">
              <button (click)="refreshWidget(widget)">↻</button>
              <button (click)="configureWidget(widget)">⚙</button>
              <button (click)="removeWidget(widget)">×</button>
            </div>
          </div>

          <div class="widget-content">
            <app-chart
              *ngIf="widget.type === 'chart'"
              [data]="widget.data"
              [config]="widget.config"
              [loading]="widget.loading">
            </app-chart>

            <app-metric-card
              *ngIf="widget.type === 'metric'"
              [value]="widget.value"
              [change]="widget.change"
              [trend]="widget.trend">
            </app-metric-card>

            <app-data-table
              *ngIf="widget.type === 'table'"
              [data]="widget.data"
              [columns]="widget.columns"
              [sortable]="true"
              [filterable]="true">
            </app-data-table>
          </div>
        </div>
      </div>

      <button class="add-widget" (click)="addWidget()">+ Add Widget</button>
    </div>
  `,
  styles: [`
    .dashboard-grid {
      display: grid;
      grid-template-columns: repeat(12, 1fr);
      grid-auto-rows: 200px;
      gap: 16px;
      padding: 16px;
    }

    .dashboard-widget {
      background: white;
      border-radius: 8px;
      box-shadow: 0 2px 8px rgba(0,0,0,0.1);
      display: flex;
      flex-direction: column;
      overflow: hidden;
    }

    .widget-content {
      flex: 1;
      overflow: auto;
      padding: 16px;
    }
  `]
})
export class DashboardComponent implements OnInit, OnDestroy {
  widgets: DashboardWidget[] = [];
  dateRange: DateRange = {
    start: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000),
    end: new Date()
  };

  private destroy$ = new Subject<void>();
  private refreshInterval = 30000; // 30 seconds

  constructor(
    private dashboardService: DashboardService,
    private exportService: ExportService,
    private realtime: RealtimeDataService
  ) {}

  ngOnInit() {
    this.loadDashboard();
    this.setupRealtimeUpdates();
    this.startAutoRefresh();
  }

  private loadDashboard() {
    this.dashboardService.getWidgets().subscribe(widgets => {
      this.widgets = widgets;
      this.loadWidgetData();
    });
  }

  private loadWidgetData() {
    this.widgets.forEach(widget => {
      this.refreshWidget(widget);
    });
  }

  refreshWidget(widget: DashboardWidget) {
    widget.loading = true;
    
    this.dashboardService.getWidgetData(widget.id, this.dateRange)
      .subscribe({
        next: (data) => {
          widget.data = data;
          widget.loading = false;
          widget.lastUpdated = new Date();
        },
        error: (error) => {
          widget.error = error.message;
          widget.loading = false;
        }
      });
  }

  private setupRealtimeUpdates() {
    this.realtime.updates$.pipe(
      takeUntil(this.destroy$)
    ).subscribe(update => {
      const widget = this.widgets.find(w => w.id === update.widgetId);
      if (widget) {
        this.updateWidgetData(widget, update.data);
      }
    });
  }

  private updateWidgetData(widget: DashboardWidget, newData: any) {
    // Merge new data with existing
    if (widget.type === 'chart' && Array.isArray(widget.data)) {
      widget.data = [...widget.data.slice(-100), newData]; // Keep last 100
    } else {
      widget.data = newData;
    }
  }

  private startAutoRefresh() {
    interval(this.refreshInterval).pipe(
      takeUntil(this.destroy$)
    ).subscribe(() => {
      this.refresh();
    });
  }

  refresh() {
    this.loadWidgetData();
  }

  onDateRangeChange() {
    this.loadWidgetData();
  }

  async export(format: 'csv' | 'pdf') {
    if (format === 'csv') {
      await this.exportService.exportToCSV(this.widgets);
    } else if (format === 'pdf') {
      await this.exportService.exportToPDF(this.widgets);
    }
  }

  addWidget() {
    // Open widget picker dialog
  }

  configureWidget(widget: DashboardWidget) {
    // Open configuration dialog
  }

  removeWidget(widget: DashboardWidget) {
    this.widgets = this.widgets.filter(w => w.id !== widget.id);
    this.dashboardService.removeWidget(widget.id).subscribe();
  }

  trackByWidgetId(index: number, widget: DashboardWidget): string {
    return widget.id;
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}

// Chart component with optimizations
@Component({
  selector: 'app-chart',
  standalone: true,
  template: `
    <div class="chart-container" #container>
      <canvas #canvas></canvas>
      
      <div *ngIf="loading" class="loading-overlay">
        <app-spinner></app-spinner>
      </div>
    </div>
  `
})
export class ChartComponent implements OnInit, OnChanges, OnDestroy {
  @Input() data!: any[];
  @Input() config!: ChartConfig;
  @Input() loading = false;

  @ViewChild('canvas') canvas!: ElementRef<HTMLCanvasElement>;
  @ViewChild('container') container!: ElementRef;

  private chart?: Chart;
  private resizeObserver?: ResizeObserver;

  async ngOnInit() {
    const { Chart, registerables } = await import('chart.js');
    Chart.register(...registerables);
    
    this.createChart();
    this.setupResize();
  }

  ngOnChanges(changes: SimpleChanges) {
    if (changes['data'] && this.chart) {
      this.updateChart();
    }
  }

  private createChart() {
    const ctx = this.canvas.nativeElement.getContext('2d');
    if (!ctx) return;

    this.chart = new Chart(ctx, {
      type: this.config.type,
      data: this.prepareData(),
      options: {
        responsive: true,
        maintainAspectRatio: false,
        animation: {
          duration: 300
        },
        plugins: {
          legend: {
            display: this.config.showLegend
          },
          tooltip: {
            enabled: true,
            mode: 'index',
            intersect: false
          }
        },
        scales: this.config.scales
      }
    });
  }

  private updateChart() {
    if (!this.chart) return;

    this.chart.data = this.prepareData();
    this.chart.update('none'); // Update without animation for performance
  }

  private prepareData() {
    // Transform data based on chart type
    return {
      labels: this.data.map(d => d.label),
      datasets: [{
        label: this.config.label,
        data: this.data.map(d => d.value),
        backgroundColor: this.config.backgroundColor,
        borderColor: this.config.borderColor,
        borderWidth: 2
      }]
    };
  }

  private setupResize() {
    this.resizeObserver = new ResizeObserver(() => {
      this.chart?.resize();
    });

    this.resizeObserver.observe(this.container.nativeElement);
  }

  ngOnDestroy() {
    this.chart?.destroy();
    this.resizeObserver?.disconnect();
  }
}

// Data table with virtual scrolling
@Component({
  selector: 'app-data-table',
  standalone: true,
  template: `
    <div class="table-controls">
      <input
        type="text"
        placeholder="Search..."
        [(ngModel)]="searchTerm"
        (input)="onSearch()">

      <select [(ngModel)]="pageSize" (change)="onPageSizeChange()">
        <option value="25">25</option>
        <option value="50">50</option>
        <option value="100">100</option>
      </select>
    </div>

    <cdk-virtual-scroll-viewport
      [itemSize]="50"
      class="table-viewport">
      
      <table>
        <thead>
          <tr>
            <th
              *ngFor="let column of columns"
              (click)="sort(column)"
              [class.sorted]="sortColumn === column">
              {{ column.label }}
              <span *ngIf="sortColumn === column">
                {{ sortDirection === 'asc' ? '↑' : '↓' }}
              </span>
            </th>
          </tr>
        </thead>
        <tbody>
          <tr *cdkVirtualFor="let row of displayedData">
            <td *ngFor="let column of columns">
              {{ getValue(row, column) }}
            </td>
          </tr>
        </tbody>
      </table>
    </cdk-virtual-scroll-viewport>

    <div class="table-pagination">
      <button (click)="previousPage()" [disabled]="currentPage === 1">
        Previous
      </button>
      <span>Page {{ currentPage }} of {{ totalPages }}</span>
      <button (click)="nextPage()" [disabled]="currentPage === totalPages">
        Next
      </button>
    </div>
  `
})
export class DataTableComponent implements OnInit, OnChanges {
  @Input() data: any[] = [];
  @Input() columns: TableColumn[] = [];
  @Input() sortable = true;
  @Input() filterable = true;

  displayedData: any[] = [];
  searchTerm = '';
  sortColumn?: TableColumn;
  sortDirection: 'asc' | 'desc' = 'asc';
  currentPage = 1;
  pageSize = 25;
  totalPages = 1;

  ngOnInit() {
    this.updateDisplayedData();
  }

  ngOnChanges() {
    this.updateDisplayedData();
  }

  private updateDisplayedData() {
    let filtered = this.data;

    // Apply search filter
    if (this.searchTerm) {
      filtered = this.data.filter(row =>
        Object.values(row).some(value =>
          String(value).toLowerCase().includes(this.searchTerm.toLowerCase())
        )
      );
    }

    // Apply sorting
    if (this.sortColumn) {
      filtered.sort((a, b) => {
        const aVal = a[this.sortColumn!.field];
        const bVal = b[this.sortColumn!.field];
        
        const comparison = aVal < bVal ? -1 : aVal > bVal ? 1 : 0;
        return this.sortDirection === 'asc' ? comparison : -comparison;
      });
    }

    // Apply pagination
    this.totalPages = Math.ceil(filtered.length / this.pageSize);
    const start = (this.currentPage - 1) * this.pageSize;
    const end = start + this.pageSize;
    
    this.displayedData = filtered.slice(start, end);
  }

  onSearch() {
    this.currentPage = 1;
    this.updateDisplayedData();
  }

  sort(column: TableColumn) {
    if (!this.sortable) return;

    if (this.sortColumn === column) {
      this.sortDirection = this.sortDirection === 'asc' ? 'desc' : 'asc';
    } else {
      this.sortColumn = column;
      this.sortDirection = 'asc';
    }

    this.updateDisplayedData();
  }

  getValue(row: any, column: TableColumn): any {
    return row[column.field];
  }

  onPageSizeChange() {
    this.currentPage = 1;
    this.updateDisplayedData();
  }

  nextPage() {
    if (this.currentPage < this.totalPages) {
      this.currentPage++;
      this.updateDisplayedData();
    }
  }

  previousPage() {
    if (this.currentPage > 1) {
      this.currentPage--;
      this.updateDisplayedData();
    }
  }
}

// Export service
@Injectable({ providedIn: 'root' })
export class ExportService {
  async exportToCSV(widgets: DashboardWidget[]) {
    const csv = this.convertToCSV(widgets);
    this.download(csv, 'dashboard-export.csv', 'text/csv');
  }

  async exportToPDF(widgets: DashboardWidget[]) {
    const { jsPDF } = await import('jspdf');
    const doc = new jsPDF();

    widgets.forEach((widget, index) => {
      if (index > 0) {
        doc.addPage();
      }

      doc.text(widget.title, 10, 10);
      
      if (widget.type === 'chart') {
        // Add chart image
        const canvas = document.querySelector(`#widget-${widget.id} canvas`) as HTMLCanvasElement;
        if (canvas) {
          const imgData = canvas.toDataURL('image/png');
          doc.addImage(imgData, 'PNG', 10, 20, 180, 100);
        }
      }
    });

    doc.save('dashboard-export.pdf');
  }

  private convertToCSV(widgets: DashboardWidget[]): string {
    // Convert widget data to CSV format
    const rows: string[] = [];
    
    widgets.forEach(widget => {
      rows.push(widget.title);
      
      if (Array.isArray(widget.data)) {
        const headers = Object.keys(widget.data[0]);
        rows.push(headers.join(','));
        
        widget.data.forEach(item => {
          const values = headers.map(h => item[h]);
          rows.push(values.join(','));
        });
      }
      
      rows.push(''); // Empty row between widgets
    });

    return rows.join('\n');
  }

  private download(content: string, filename: string, type: string) {
    const blob = new Blob([content], { type });
    const url = URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.href = url;
    link.download = filename;
    link.click();
    URL.revokeObjectURL(url);
  }
}
```

## Key Takeaways

1. **Real-time Updates**: Use WebSocket or SSE for live data updates, merge new data efficiently.

2. **Virtual Scrolling**: Implement virtual scrolling for large tables (100K+ rows) to maintain performance.

3. **Chart Optimization**: Use Chart.js with animation disabled for real-time updates, resize observer for responsive charts.

4. **Lazy Loading**: Load widget data on demand, cache results, implement pagination for large datasets.

5. **Export Functionality**: Support CSV and PDF export using jsPDF and Blob API.

6. **Drag and Drop**: Use CDK drag-drop for rearrangeable dashboard widgets.

7. **Date Range Filtering**: Implement date pickers with preset ranges (today, last 7/30/90 days).

8. **Caching Strategy**: Cache dashboard configuration, widget data, use IndexedDB for offline access.

9. **Performance Monitoring**: Track render times, data load times, optimize expensive calculations.

10. **Responsive Design**: Use CSS Grid for flexible layouts, mobile-optimized controls and charts.

## Red Flags to Avoid

- Loading all data at once without pagination
- Not implementing virtual scrolling for large tables
- Missing real-time update handling
- Poor chart performance (too many animations)
- Not caching widget data
- Missing error handling
- No loading states
- Poor mobile experience
- Memory leaks from chart instances
- Not optimizing re-renders

## Interview Talking Points

- Real-time data update strategies
- Virtual scrolling implementation
- Chart library selection and optimization
- Export functionality (CSV, PDF, PNG)
- Caching and offline support
- Performance optimization for large datasets
- Drag-and-drop dashboard customization
- Date range filtering and aggregation
- Mobile responsive dashboard design
- State management for complex dashboards

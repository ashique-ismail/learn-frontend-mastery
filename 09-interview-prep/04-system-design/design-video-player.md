# System Design: Video Player

## Overview

Design a custom video player with buffering strategies, adaptive bitrate streaming, quality selection, subtitles, playback controls, and accessibility features.

## Requirements

### Functional Requirements
- Play/pause, seek, volume control
- Quality selection (240p-4K)
- Playback speed control
- Subtitles/captions (multiple languages)
- Fullscreen mode
- Picture-in-picture
- Keyboard shortcuts
- Mobile touch gestures
- Playlist/chapter navigation

### Non-Functional Requirements
- Buffer ahead 30-60 seconds
- Adaptive bitrate switching
- < 2 second startup time
- Smooth quality transitions
- Minimal rebuffering
- Accessible (WCAG 2.1 AA)
- Support all modern browsers
- Mobile-optimized

## Architecture

```typescript
// Video player component
@Component({
  selector: 'app-video-player',
  standalone: true,
  template: `
    <div class="video-container" 
         [class.fullscreen]="isFullscreen"
         #container>
      
      <video
        #videoElement
        [src]="currentQualitySource"
        [poster]="posterUrl"
        (loadedmetadata)="onLoadedMetadata()"
        (timeupdate)="onTimeUpdate()"
        (progress)="onProgress()"
        (waiting)="onWaiting()"
        (canplay)="onCanPlay()"
        (ended)="onEnded()"
        (error)="onError($event)"
        playsinline>
      </video>

      <!-- Loading/buffering -->
      <div *ngIf="isBuffering" class="buffering-overlay">
        <app-spinner></app-spinner>
      </div>

      <!-- Controls overlay -->
      <div class="controls-overlay"
           [class.visible]="showControls"
           (mousemove)="onMouseMove()"
           (touchstart)="onTouchStart()">
        
        <!-- Play button center -->
        <button
          *ngIf="!isPlaying"
          class="play-button-center"
          (click)="togglePlay()"
          aria-label="Play">
          ▶
        </button>

        <!-- Bottom controls -->
        <div class="controls-bar">
          <!-- Play/Pause -->
          <button
            (click)="togglePlay()"
            [attr.aria-label]="isPlaying ? 'Pause' : 'Play'">
            {{ isPlaying ? '⏸' : '▶' }}
          </button>

          <!-- Volume -->
          <div class="volume-control">
            <button
              (click)="toggleMute()"
              [attr.aria-label]="isMuted ? 'Unmute' : 'Mute'">
              {{ isMuted ? '🔇' : '🔊' }}
            </button>
            <input
              type="range"
              min="0"
              max="100"
              [value]="volume"
              (input)="setVolume($event)"
              class="volume-slider"
              aria-label="Volume">
          </div>

          <!-- Time display -->
          <span class="time-display">
            {{ currentTime | time }} / {{ duration | time }}
          </span>

          <!-- Progress bar -->
          <div class="progress-container" (click)="seek($event)">
            <div class="progress-bar">
              <!-- Buffered -->
              <div
                *ngFor="let range of bufferedRanges"
                class="buffered"
                [style.left.%]="(range.start / duration) * 100"
                [style.width.%]="((range.end - range.start) / duration) * 100">
              </div>

              <!-- Played -->
              <div
                class="played"
                [style.width.%]="(currentTime / duration) * 100">
              </div>

              <!-- Seek handle -->
              <div
                class="seek-handle"
                [style.left.%]="(currentTime / duration) * 100">
              </div>
            </div>

            <!-- Thumbnail preview -->
            <div
              *ngIf="previewThumbnail"
              class="thumbnail-preview"
              [style.left.px]="thumbnailPosition">
              <img [src]="previewThumbnail.url" alt="Preview">
              <span>{{ previewThumbnail.time | time }}</span>
            </div>
          </div>

          <!-- Quality selector -->
          <div class="quality-selector">
            <button (click)="toggleQualityMenu()">
              {{ currentQuality }}p ⚙️
            </button>
            <div *ngIf="showQualityMenu" class="quality-menu">
              <button
                *ngFor="let quality of availableQualities"
                (click)="selectQuality(quality)"
                [class.active]="quality === currentQuality">
                {{ quality }}p
              </button>
              <button
                (click)="selectQuality('auto')"
                [class.active]="isAutoQuality">
                Auto
              </button>
            </div>
          </div>

          <!-- Speed control -->
          <div class="speed-control">
            <button (click)="toggleSpeedMenu()">
              {{ playbackRate }}x
            </button>
            <div *ngIf="showSpeedMenu" class="speed-menu">
              <button
                *ngFor="let speed of speeds"
                (click)="setPlaybackRate(speed)"
                [class.active]="speed === playbackRate">
                {{ speed }}x
              </button>
            </div>
          </div>

          <!-- Subtitles -->
          <button
            (click)="toggleSubtitles()"
            [class.active]="subtitlesEnabled"
            aria-label="Subtitles">
            CC
          </button>

          <!-- Picture-in-picture -->
          <button
            *ngIf="pipSupported"
            (click)="togglePip()"
            aria-label="Picture in Picture">
            ⧉
          </button>

          <!-- Fullscreen -->
          <button
            (click)="toggleFullscreen()"
            [attr.aria-label]="isFullscreen ? 'Exit fullscreen' : 'Fullscreen'">
            {{ isFullscreen ? '⛶' : '⛶' }}
          </button>
        </div>
      </div>

      <!-- Subtitles/captions -->
      <div *ngIf="subtitlesEnabled && currentSubtitle" class="subtitle">
        {{ currentSubtitle.text }}
      </div>

      <!-- Error message -->
      <div *ngIf="error" class="error-message">
        {{ error }}
        <button (click)="retry()">Retry</button>
      </div>
    </div>
  `,
  styles: [`
    .video-container {
      position: relative;
      width: 100%;
      aspect-ratio: 16 / 9;
      background: #000;
    }

    video {
      width: 100%;
      height: 100%;
    }

    .controls-overlay {
      position: absolute;
      bottom: 0;
      left: 0;
      right: 0;
      background: linear-gradient(transparent, rgba(0,0,0,0.7));
      opacity: 0;
      transition: opacity 0.3s;
    }

    .controls-overlay.visible {
      opacity: 1;
    }

    .progress-container {
      position: relative;
      flex: 1;
      height: 8px;
      cursor: pointer;
    }

    .progress-bar {
      position: relative;
      width: 100%;
      height: 100%;
      background: rgba(255,255,255,0.3);
    }

    .buffered {
      position: absolute;
      height: 100%;
      background: rgba(255,255,255,0.5);
    }

    .played {
      position: absolute;
      height: 100%;
      background: #ff0000;
    }

    .seek-handle {
      position: absolute;
      top: 50%;
      transform: translate(-50%, -50%);
      width: 16px;
      height: 16px;
      border-radius: 50%;
      background: #ff0000;
    }
  `]
})
export class VideoPlayerComponent implements OnInit, OnDestroy {
  @ViewChild('videoElement') videoElement!: ElementRef<HTMLVideoElement>;
  @ViewChild('container') container!: ElementRef;

  @Input() sources!: VideoSource[];
  @Input() posterUrl = '';
  @Input() subtitles: Subtitle[] = [];

  isPlaying = false;
  isBuffering = false;
  isMuted = false;
  isFullscreen = false;
  subtitlesEnabled = false;
  showControls = false;
  showQualityMenu = false;
  showSpeedMenu = false;
  isAutoQuality = true;

  currentTime = 0;
  duration = 0;
  volume = 100;
  playbackRate = 1;
  currentQuality = 720;
  bufferedRanges: BufferedRange[] = [];
  
  availableQualities = [240, 360, 480, 720, 1080, 1440, 2160];
  speeds = [0.25, 0.5, 0.75, 1, 1.25, 1.5, 1.75, 2];
  
  error: string | null = null;
  pipSupported = document.pictureInPictureEnabled;
  
  private controlsTimeout?: number;
  private destroy$ = new Subject<void>();

  constructor(
    private adaptiveService: AdaptiveBitrateService,
    private renderer: Renderer2
  ) {}

  ngOnInit() {
    this.setupKeyboardShortcuts();
    this.startAdaptiveBitrate();
  }

  get currentQualitySource(): string {
    return this.sources.find(s => s.quality === this.currentQuality)?.url || '';
  }

  onLoadedMetadata() {
    const video = this.videoElement.nativeElement;
    this.duration = video.duration;
  }

  onTimeUpdate() {
    const video = this.videoElement.nativeElement;
    this.currentTime = video.currentTime;
  }

  onProgress() {
    const video = this.videoElement.nativeElement;
    const buffered = video.buffered;
    
    this.bufferedRanges = [];
    for (let i = 0; i < buffered.length; i++) {
      this.bufferedRanges.push({
        start: buffered.start(i),
        end: buffered.end(i)
      });
    }
  }

  onWaiting() {
    this.isBuffering = true;
  }

  onCanPlay() {
    this.isBuffering = false;
  }

  onEnded() {
    this.isPlaying = false;
  }

  onError(event: Event) {
    this.error = 'Failed to load video';
  }

  togglePlay() {
    const video = this.videoElement.nativeElement;
    
    if (this.isPlaying) {
      video.pause();
      this.isPlaying = false;
    } else {
      video.play();
      this.isPlaying = true;
    }
  }

  toggleMute() {
    const video = this.videoElement.nativeElement;
    video.muted = !video.muted;
    this.isMuted = video.muted;
  }

  setVolume(event: Event) {
    const input = event.target as HTMLInputElement;
    const video = this.videoElement.nativeElement;
    
    this.volume = parseInt(input.value);
    video.volume = this.volume / 100;
    
    if (this.volume === 0) {
      this.isMuted = true;
    } else if (this.isMuted) {
      this.isMuted = false;
    }
  }

  seek(event: MouseEvent) {
    const container = event.currentTarget as HTMLElement;
    const rect = container.getBoundingClientRect();
    const x = event.clientX - rect.left;
    const percentage = x / rect.width;
    
    const video = this.videoElement.nativeElement;
    video.currentTime = percentage * this.duration;
  }

  setPlaybackRate(rate: number) {
    const video = this.videoElement.nativeElement;
    video.playbackRate = rate;
    this.playbackRate = rate;
    this.showSpeedMenu = false;
  }

  selectQuality(quality: number | 'auto') {
    if (quality === 'auto') {
      this.isAutoQuality = true;
      this.startAdaptiveBitrate();
    } else {
      this.isAutoQuality = false;
      this.currentQuality = quality;
      this.switchQuality(quality);
    }
    this.showQualityMenu = false;
  }

  private switchQuality(quality: number) {
    const video = this.videoElement.nativeElement;
    const currentTime = video.currentTime;
    const wasPlaying = !video.paused;
    
    // Switch source
    this.currentQuality = quality;
    video.src = this.currentQualitySource;
    video.currentTime = currentTime;
    
    if (wasPlaying) {
      video.play();
    }
  }

  private startAdaptiveBitrate() {
    if (!this.isAutoQuality) return;

    this.adaptiveService.getBandwidth().pipe(
      takeUntil(this.destroy$)
    ).subscribe(bandwidth => {
      const optimalQuality = this.getOptimalQuality(bandwidth);
      if (optimalQuality !== this.currentQuality) {
        this.switchQuality(optimalQuality);
      }
    });
  }

  private getOptimalQuality(bandwidth: number): number {
    // Simplified: bandwidth in Mbps
    if (bandwidth > 8) return 1080;
    if (bandwidth > 4) return 720;
    if (bandwidth > 2) return 480;
    if (bandwidth > 1) return 360;
    return 240;
  }

  toggleSubtitles() {
    this.subtitlesEnabled = !this.subtitlesEnabled;
  }

  toggleFullscreen() {
    const container = this.container.nativeElement;
    
    if (!this.isFullscreen) {
      if (container.requestFullscreen) {
        container.requestFullscreen();
      }
      this.isFullscreen = true;
    } else {
      if (document.exitFullscreen) {
        document.exitFullscreen();
      }
      this.isFullscreen = false;
    }
  }

  async togglePip() {
    const video = this.videoElement.nativeElement;
    
    try {
      if (document.pictureInPictureElement) {
        await document.exitPictureInPicture();
      } else {
        await video.requestPictureInPicture();
      }
    } catch (error) {
      console.error('PiP error:', error);
    }
  }

  onMouseMove() {
    this.showControls = true;
    this.resetControlsTimeout();
  }

  onTouchStart() {
    this.showControls = !this.showControls;
    if (this.showControls) {
      this.resetControlsTimeout();
    }
  }

  private resetControlsTimeout() {
    clearTimeout(this.controlsTimeout);
    this.controlsTimeout = window.setTimeout(() => {
      if (this.isPlaying) {
        this.showControls = false;
      }
    }, 3000);
  }

  private setupKeyboardShortcuts() {
    fromEvent<KeyboardEvent>(document, 'keydown')
      .pipe(takeUntil(this.destroy$))
      .subscribe(event => {
        switch (event.key) {
          case ' ':
            event.preventDefault();
            this.togglePlay();
            break;
          case 'f':
            this.toggleFullscreen();
            break;
          case 'm':
            this.toggleMute();
            break;
          case 'ArrowLeft':
            this.skipBackward();
            break;
          case 'ArrowRight':
            this.skipForward();
            break;
        }
      });
  }

  private skipBackward() {
    const video = this.videoElement.nativeElement;
    video.currentTime = Math.max(0, video.currentTime - 10);
  }

  private skipForward() {
    const video = this.videoElement.nativeElement;
    video.currentTime = Math.min(this.duration, video.currentTime + 10);
  }

  toggleQualityMenu() {
    this.showQualityMenu = !this.showQualityMenu;
  }

  toggleSpeedMenu() {
    this.showSpeedMenu = !this.showSpeedMenu;
  }

  retry() {
    const video = this.videoElement.nativeElement;
    video.load();
    this.error = null;
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}

// Adaptive bitrate service
@Injectable({ providedIn: 'root' })
export class AdaptiveBitrateService {
  getBandwidth(): Observable<number> {
    return interval(5000).pipe(
      switchMap(() => this.measureBandwidth()),
      startWith(5) // Default 5 Mbps
    );
  }

  private measureBandwidth(): Observable<number> {
    // Simplified bandwidth measurement
    const startTime = performance.now();
    const testSize = 1024 * 1024; // 1MB

    return this.http.get('/api/bandwidth-test', {
      responseType: 'blob'
    }).pipe(
      map(() => {
        const duration = (performance.now() - startTime) / 1000;
        const bandwidth = (testSize * 8) / duration / 1000000; // Mbps
        return bandwidth;
      }),
      catchError(() => of(5))
    );
  }

  constructor(private http: HttpClient) {}
}
```

## Key Takeaways

1. **Adaptive Bitrate**: Measure bandwidth and automatically switch quality for optimal viewing experience.

2. **Custom Controls**: Build custom UI controls for consistent UX across browsers and devices.

3. **Keyboard Shortcuts**: Implement space, arrow keys, f, m for power users and accessibility.

4. **Buffering Strategy**: Buffer 30-60 seconds ahead, show loading indicator during buffering.

5. **Quality Switching**: Support smooth quality transitions, preserve playback position and state.

6. **Subtitles/Captions**: Support multiple languages, synchronized display with proper styling.

7. **Touch Gestures**: Implement double-tap to skip, pinch to zoom for mobile users.

8. **Picture-in-Picture**: Support PiP API for multitasking while watching.

9. **Error Handling**: Gracefully handle load failures with retry mechanism.

10. **Performance**: Use native video element, optimize control rendering, minimize repaints.

## Red Flags to Avoid

- No buffering indicator
- Missing keyboard shortcuts
- Poor quality switching UX
- Not handling video errors
- Missing accessibility features
- No mobile optimizations
- Blocking main thread during controls
- Not cleaning up resources
- Missing fullscreen support
- Poor subtitle synchronization

## Interview Talking Points

- Adaptive bitrate algorithms
- HLS vs DASH streaming
- Buffering strategies
- Quality switching without rebuffering
- Subtitle format support (WebVTT, SRT)
- Accessibility requirements
- Mobile vs desktop differences
- Performance optimization
- DRM and content protection
- Analytics and monitoring

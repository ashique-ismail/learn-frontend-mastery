# Video Player with Custom Controls

## The Idea

**In plain English:** You have a `<video>` element on the page and you want to replace the browser's default controls — the ones that look different in every browser and cannot be styled — with your own fully custom UI: a play/pause button, a clickable progress bar that scrubs to any point in the video, a volume slider, a fullscreen button, and a Picture-in-Picture button. Everything is wired to the real video element through a ref and a handful of native browser APIs.

**Real-world analogy:** Think of a DVD player behind a custom front panel.

- The **DVD mechanism** (the actual spinning disc, laser, and motor) = the `<video>` element. It does the real work. You do not replace it or fake it.
- The **front panel** (buttons, display, LED indicators) = your custom React or Angular component. It sends commands to the mechanism and reads its state to update the display.
- The **remote control** = keyboard shortcuts. They talk to the same mechanism through the same commands as the front panel.
- The **Now Playing display** (the counter showing elapsed time) = a progress bar driven by the `timeupdate` event. The disc tells the panel where it is every 250ms; the panel redraws accordingly.
- The **"TV mode" button** that shrinks the player into a corner of the screen = Picture-in-Picture API. The video keeps playing; the browser moves it to a floating overlay window.
- Crucially: the front panel does not contain the video. It controls it. This separation is the core mental model.

---

## Learning Objectives

- Understand why the native `<video>` controls are a dead end for production UI and what the browser API surface looks like beneath them
- Wire a ref to the video element and drive playback with `play()`, `pause()`, `currentTime`, and `volume`
- Listen to `timeupdate`, `loadedmetadata`, `ended`, and `volumechange` to keep UI state in sync with the media engine
- Implement click-to-seek on a progress bar using pointer event math and `getBoundingClientRect`
- Integrate the Fullscreen API (`requestFullscreen`, `exitFullscreen`, and the `fullscreenchange` event)
- Integrate the Picture-in-Picture API (`requestPictureInPicture`, `leavePictureInPicture`, and the `enterpictureinpicture` / `leavepictureinpicture` events)
- Add keyboard shortcuts (Space = play/pause, Arrow keys = seek, M = mute, F = fullscreen) without breaking native browser shortcuts
- Ship an accessible player: correct ARIA roles, live region for time announcements, keyboard-navigable controls

---

## Why CSS Alone Isn't Enough

| Requirement | CSS can do it? | Why not |
|---|---|---|
| Style the progress bar track and thumb | ✅ via `appearance: none` on `<input type="range">` | — |
| Hide the native controls | ✅ omit the `controls` attribute | — |
| Play / pause the video | ❌ | Requires calling `videoEl.play()` / `videoEl.pause()` imperatively |
| Know the current playback position | ❌ | Requires listening to the `timeupdate` event |
| Seek to an arbitrary point | ❌ | Requires writing to `videoEl.currentTime` |
| Enter / exit fullscreen | ❌ | Requires `element.requestFullscreen()` and the Fullscreen API |
| Enter Picture-in-Picture | ❌ | Requires `videoEl.requestPictureInPicture()` |
| Respond to keyboard shortcuts | ❌ | Requires `keydown` listeners |
| Announce playback state to screen readers | ❌ | Requires `aria-live` regions toggled by JS |

**Conclusion:** CSS styles the controls. The browser's media API and event system power them. Every meaningful interaction requires JS.

---

## HTML & CSS Foundation

### The Structure

```html
<!--
  The video element itself has no `controls` attribute.
  The custom controls sit in a sibling div, not inside the video.
  The outer wrapper is what enters fullscreen — not the video alone.
-->
<div class="player-wrapper" id="player">

  <video
    class="player-video"
    src="/media/sample.mp4"
    preload="metadata"
    aria-label="Sample video"
  ></video>

  <!-- Overlay: clicking the video body toggles play/pause -->
  <button class="player-overlay-btn" aria-label="Toggle play" tabindex="-1" aria-hidden="true"></button>

  <!-- Custom controls bar -->
  <div class="player-controls" role="group" aria-label="Video controls">

    <!-- Play / Pause -->
    <button class="ctrl-btn ctrl-play" aria-label="Play" aria-pressed="false">
      <svg aria-hidden="true"><!-- play icon --></svg>
    </button>

    <!-- Time display -->
    <span class="ctrl-time" aria-live="off">
      <span class="ctrl-time-current">0:00</span>
      <span class="ctrl-time-sep" aria-hidden="true"> / </span>
      <span class="ctrl-time-total">0:00</span>
    </span>

    <!-- Progress bar — uses <input type="range"> for free keyboard support -->
    <div class="ctrl-progress-wrap" aria-hidden="true">
      <input
        type="range"
        class="ctrl-progress"
        min="0" max="100" step="0.1" value="0"
        aria-label="Seek"
        aria-valuemin="0" aria-valuemax="100" aria-valuenow="0"
      />
      <div class="ctrl-progress-fill" style="width: 0%"></div>
    </div>

    <!-- Volume -->
    <button class="ctrl-btn ctrl-mute" aria-label="Mute" aria-pressed="false">
      <svg aria-hidden="true"><!-- volume icon --></svg>
    </button>
    <input type="range" class="ctrl-volume" min="0" max="1" step="0.05" value="1" aria-label="Volume" />

    <!-- Picture-in-Picture -->
    <button class="ctrl-btn ctrl-pip" aria-label="Picture in picture">
      <svg aria-hidden="true"><!-- pip icon --></svg>
    </button>

    <!-- Fullscreen -->
    <button class="ctrl-btn ctrl-fullscreen" aria-label="Enter fullscreen" aria-pressed="false">
      <svg aria-hidden="true"><!-- expand icon --></svg>
    </button>

  </div>

  <!-- Screen-reader-only live region for state announcements -->
  <span class="sr-only" role="status" aria-live="polite" id="player-status"></span>
</div>
```

### The CSS

```css
/* ─── Tokens ─── */
:root {
  --player-bg: #000;
  --ctrl-bar-bg: linear-gradient(transparent, rgba(0,0,0,0.75));
  --ctrl-height: 48px;
  --ctrl-color: #fff;
  --progress-thumb-size: 14px;
  --progress-track-h: 4px;
  --progress-fill: #e50914;   /* Netflix red — swap for your brand */
  --transition-fast: 0.15s ease;
}

/* ─── Wrapper: this is what enters fullscreen ─── */
.player-wrapper {
  position: relative;
  background: var(--player-bg);
  width: 100%;
  max-width: 960px;
  aspect-ratio: 16 / 9;
  border-radius: 6px;
  overflow: hidden;
  user-select: none;
}

.player-wrapper:fullscreen {
  max-width: 100%;
  border-radius: 0;
}

/* ─── Video fills the wrapper ─── */
.player-video {
  width: 100%;
  height: 100%;
  object-fit: contain;
  display: block;
}

/* ─── Overlay click target ─── */
.player-overlay-btn {
  position: absolute;
  inset: 0;
  background: transparent;
  border: none;
  cursor: pointer;
  z-index: 1;
}

/* ─── Controls bar ─── */
.player-controls {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  height: var(--ctrl-height);
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 0 12px;
  background: var(--ctrl-bar-bg);
  color: var(--ctrl-color);
  opacity: 0;
  transition: opacity var(--transition-fast);
  z-index: 2;
}

/* Show controls on hover/focus-within */
.player-wrapper:hover .player-controls,
.player-wrapper:focus-within .player-controls {
  opacity: 1;
}

/* Always show when paused */
.player-wrapper[data-paused="true"] .player-controls {
  opacity: 1;
}

/* ─── Generic control button ─── */
.ctrl-btn {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 36px;
  height: 36px;
  background: none;
  border: none;
  color: inherit;
  cursor: pointer;
  border-radius: 4px;
  transition: background var(--transition-fast);
}

.ctrl-btn:hover  { background: rgba(255,255,255,0.15); }
.ctrl-btn:focus-visible {
  outline: 2px solid #fff;
  outline-offset: 2px;
}

/* ─── Time display ─── */
.ctrl-time {
  font-size: 0.8rem;
  white-space: nowrap;
  min-width: 90px;
  text-align: center;
}

/* ─── Progress bar ─── */
.ctrl-progress-wrap {
  flex: 1;
  position: relative;
  height: 20px;       /* tall hit area, thin visual track */
  display: flex;
  align-items: center;
}

.ctrl-progress {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  opacity: 0;          /* input is invisible; visual track is below */
  cursor: pointer;
  margin: 0;
  z-index: 2;
}

/* Custom progress track */
.ctrl-progress-wrap::before {
  content: '';
  position: absolute;
  left: 0; right: 0;
  height: var(--progress-track-h);
  border-radius: 2px;
  background: rgba(255,255,255,0.3);
}

/* Filled portion */
.ctrl-progress-fill {
  position: absolute;
  left: 0;
  height: var(--progress-track-h);
  border-radius: 2px;
  background: var(--progress-fill);
  pointer-events: none;
  z-index: 1;
  transition: width 0.1s linear;
}

/* ─── Volume slider (same pattern as progress) ─── */
.ctrl-volume {
  width: 70px;
  cursor: pointer;
  accent-color: var(--ctrl-color);
}

/* ─── Reduced motion ─── */
@media (prefers-reduced-motion: reduce) {
  .player-controls,
  .ctrl-progress-fill {
    transition: none;
  }
}
```

**CSS owns:** wrapper aspect ratio, controls fade-in/out on hover, progress track visual, button focus rings, fullscreen layout override.

**CSS cannot own:** tracking `currentTime`, seeking, play/pause state, fullscreen toggling, pip state.

---

## React Implementation

### Types

```tsx
// videoPlayer.types.ts
export interface PlayerState {
  playing: boolean;
  currentTime: number;
  duration: number;
  volume: number;
  muted: boolean;
  fullscreen: boolean;
  pip: boolean;
}
```

### The Hook — `useVideoPlayer`

```tsx
// useVideoPlayer.ts
import { useRef, useState, useEffect, useCallback, RefObject } from 'react';
import type { PlayerState } from './videoPlayer.types';

export function useVideoPlayer(): {
  videoRef: RefObject<HTMLVideoElement>;
  wrapperRef: RefObject<HTMLDivElement>;
  state: PlayerState;
  togglePlay: () => void;
  seek: (seconds: number) => void;
  setVolume: (v: number) => void;
  toggleMute: () => void;
  toggleFullscreen: () => void;
  togglePip: () => void;
} {
  const videoRef   = useRef<HTMLVideoElement>(null);
  const wrapperRef = useRef<HTMLDivElement>(null);

  const [state, setState] = useState<PlayerState>({
    playing: false,
    currentTime: 0,
    duration: 0,
    volume: 1,
    muted: false,
    fullscreen: false,
    pip: false,
  });

  // ── Sync media events → state ──────────────────────────────────────────────

  useEffect(() => {
    const video = videoRef.current;
    if (!video) return;

    const onPlay      = () => setState(s => ({ ...s, playing: true }));
    const onPause     = () => setState(s => ({ ...s, playing: false }));
    const onEnded     = () => setState(s => ({ ...s, playing: false }));
    const onTimeUpdate = () =>
      setState(s => ({ ...s, currentTime: video.currentTime }));
    const onLoadedMetadata = () =>
      setState(s => ({ ...s, duration: video.duration }));
    const onVolumeChange = () =>
      setState(s => ({ ...s, volume: video.volume, muted: video.muted }));

    // Fullscreen changes fire on *document*, not the video element
    const onFullscreenChange = () =>
      setState(s => ({ ...s, fullscreen: !!document.fullscreenElement }));

    // PiP events fire on the video element itself
    const onEnterPip = () => setState(s => ({ ...s, pip: true }));
    const onLeavePip = () => setState(s => ({ ...s, pip: false }));

    video.addEventListener('play', onPlay);
    video.addEventListener('pause', onPause);
    video.addEventListener('ended', onEnded);
    video.addEventListener('timeupdate', onTimeUpdate);
    video.addEventListener('loadedmetadata', onLoadedMetadata);
    video.addEventListener('volumechange', onVolumeChange);
    video.addEventListener('enterpictureinpicture', onEnterPip);
    video.addEventListener('leavepictureinpicture', onLeavePip);
    document.addEventListener('fullscreenchange', onFullscreenChange);

    return () => {
      video.removeEventListener('play', onPlay);
      video.removeEventListener('pause', onPause);
      video.removeEventListener('ended', onEnded);
      video.removeEventListener('timeupdate', onTimeUpdate);
      video.removeEventListener('loadedmetadata', onLoadedMetadata);
      video.removeEventListener('volumechange', onVolumeChange);
      video.removeEventListener('enterpictureinpicture', onEnterPip);
      video.removeEventListener('leavepictureinpicture', onLeavePip);
      document.removeEventListener('fullscreenchange', onFullscreenChange);
    };
  }, []);

  // ── Imperative controls ────────────────────────────────────────────────────

  const togglePlay = useCallback(() => {
    const video = videoRef.current;
    if (!video) return;
    video.paused ? video.play() : video.pause();
  }, []);

  const seek = useCallback((seconds: number) => {
    const video = videoRef.current;
    if (!video) return;
    video.currentTime = Math.max(0, Math.min(seconds, video.duration));
  }, []);

  const setVolume = useCallback((v: number) => {
    const video = videoRef.current;
    if (!video) return;
    video.volume = Math.max(0, Math.min(1, v));
    video.muted = v === 0;
  }, []);

  const toggleMute = useCallback(() => {
    const video = videoRef.current;
    if (!video) return;
    video.muted = !video.muted;
  }, []);

  const toggleFullscreen = useCallback(async () => {
    const wrapper = wrapperRef.current;
    if (!wrapper) return;
    if (!document.fullscreenElement) {
      await wrapper.requestFullscreen();
    } else {
      await document.exitFullscreen();
    }
  }, []);

  const togglePip = useCallback(async () => {
    const video = videoRef.current;
    if (!video) return;
    if (document.pictureInPictureElement) {
      await document.exitPictureInPicture();
    } else if (document.pictureInPictureEnabled) {
      await video.requestPictureInPicture();
    }
  }, []);

  return {
    videoRef, wrapperRef, state,
    togglePlay, seek, setVolume, toggleMute, toggleFullscreen, togglePip,
  };
}
```

### Keyboard Shortcuts Hook

```tsx
// usePlayerKeyboard.ts
import { useEffect } from 'react';

interface Handlers {
  togglePlay: () => void;
  seek: (s: number) => void;
  toggleMute: () => void;
  toggleFullscreen: () => void;
  currentTime: number;
}

export function usePlayerKeyboard(
  wrapperRef: React.RefObject<HTMLElement>,
  handlers: Handlers,
) {
  useEffect(() => {
    const el = wrapperRef.current;
    if (!el) return;

    const onKey = (e: KeyboardEvent) => {
      // Only intercept when the wrapper or its children have focus
      if (!el.contains(document.activeElement)) return;

      switch (e.key) {
        case ' ':
        case 'k':                      // YouTube convention
          e.preventDefault();          // prevent page scroll on Space
          handlers.togglePlay();
          break;
        case 'ArrowRight':
          e.preventDefault();
          handlers.seek(handlers.currentTime + 5);
          break;
        case 'ArrowLeft':
          e.preventDefault();
          handlers.seek(handlers.currentTime - 5);
          break;
        case 'ArrowUp':
          e.preventDefault();
          handlers.seek(handlers.currentTime + 10);
          break;
        case 'ArrowDown':
          e.preventDefault();
          handlers.seek(handlers.currentTime - 10);
          break;
        case 'm':
        case 'M':
          handlers.toggleMute();
          break;
        case 'f':
        case 'F':
          handlers.toggleFullscreen();
          break;
      }
    };

    document.addEventListener('keydown', onKey);
    return () => document.removeEventListener('keydown', onKey);
  }, [wrapperRef, handlers]);   // handlers should be stable (useCallback)
}
```

### The Player Component

```tsx
// VideoPlayer.tsx
import { useRef } from 'react';
import { useVideoPlayer } from './useVideoPlayer';
import { usePlayerKeyboard } from './usePlayerKeyboard';

function formatTime(seconds: number): string {
  if (isNaN(seconds)) return '0:00';
  const m = Math.floor(seconds / 60);
  const s = Math.floor(seconds % 60).toString().padStart(2, '0');
  return `${m}:${s}`;
}

interface VideoPlayerProps {
  src: string;
  poster?: string;
}

export function VideoPlayer({ src, poster }: VideoPlayerProps) {
  const {
    videoRef, wrapperRef, state,
    togglePlay, seek, setVolume, toggleMute, toggleFullscreen, togglePip,
  } = useVideoPlayer();

  usePlayerKeyboard(wrapperRef, {
    togglePlay,
    seek,
    toggleMute,
    toggleFullscreen,
    currentTime: state.currentTime,
  });

  const progressPct = state.duration
    ? (state.currentTime / state.duration) * 100
    : 0;

  // Click-to-seek on the progress bar fill area
  const handleProgressClick = (e: React.MouseEvent<HTMLDivElement>) => {
    const rect = e.currentTarget.getBoundingClientRect();
    const ratio = (e.clientX - rect.left) / rect.width;
    seek(ratio * state.duration);
  };

  return (
    <div
      ref={wrapperRef}
      className="player-wrapper"
      data-paused={!state.playing}
      // Make the wrapper focusable so keyboard events fire within it
      tabIndex={-1}
    >
      <video
        ref={videoRef}
        className="player-video"
        src={src}
        poster={poster}
        preload="metadata"
        aria-label="Video player"
        // Hide from AT — the custom controls label the interaction
        aria-hidden="true"
      />

      {/* Click-to-play overlay */}
      <button
        className="player-overlay-btn"
        onClick={togglePlay}
        aria-label={state.playing ? 'Pause' : 'Play'}
        tabIndex={-1}
        aria-hidden="true"
      />

      <div className="player-controls" role="group" aria-label="Video controls">

        {/* Play/Pause */}
        <button
          className="ctrl-btn ctrl-play"
          onClick={togglePlay}
          aria-label={state.playing ? 'Pause' : 'Play'}
          aria-pressed={state.playing}
        >
          {state.playing ? <PauseIcon /> : <PlayIcon />}
        </button>

        {/* Time */}
        <span className="ctrl-time" aria-live="off">
          {formatTime(state.currentTime)}
          <span aria-hidden="true"> / </span>
          {formatTime(state.duration)}
        </span>

        {/* Progress bar — visual + accessible */}
        <div
          className="ctrl-progress-wrap"
          onClick={handleProgressClick}
          role="presentation"
        >
          {/* Visually hidden range input for keyboard + AT access */}
          <input
            type="range"
            className="ctrl-progress"
            min={0}
            max={state.duration || 100}
            step={1}
            value={state.currentTime}
            aria-label="Seek"
            aria-valuetext={`${formatTime(state.currentTime)} of ${formatTime(state.duration)}`}
            onChange={e => seek(Number(e.target.value))}
          />
          <div
            className="ctrl-progress-fill"
            style={{ width: `${progressPct}%` }}
          />
        </div>

        {/* Mute */}
        <button
          className="ctrl-btn ctrl-mute"
          onClick={toggleMute}
          aria-label={state.muted ? 'Unmute' : 'Mute'}
          aria-pressed={state.muted}
        >
          {state.muted ? <MuteIcon /> : <VolumeIcon />}
        </button>

        {/* Volume */}
        <input
          type="range"
          className="ctrl-volume"
          min={0}
          max={1}
          step={0.05}
          value={state.muted ? 0 : state.volume}
          aria-label="Volume"
          onChange={e => setVolume(Number(e.target.value))}
        />

        {/* PiP — only render if the API is available */}
        {document.pictureInPictureEnabled && (
          <button
            className="ctrl-btn ctrl-pip"
            onClick={togglePip}
            aria-label={state.pip ? 'Exit picture in picture' : 'Picture in picture'}
            aria-pressed={state.pip}
          >
            <PipIcon />
          </button>
        )}

        {/* Fullscreen */}
        {document.fullscreenEnabled && (
          <button
            className="ctrl-btn ctrl-fullscreen"
            onClick={toggleFullscreen}
            aria-label={state.fullscreen ? 'Exit fullscreen' : 'Enter fullscreen'}
            aria-pressed={state.fullscreen}
          >
            {state.fullscreen ? <ShrinkIcon /> : <ExpandIcon />}
          </button>
        )}
      </div>

      {/* SR live region */}
      <span className="sr-only" role="status" aria-live="polite" id="player-status" />
    </div>
  );
}

// Minimal inline SVG icons (swap for your icon library)
const PlayIcon   = () => <svg viewBox="0 0 24 24" width="20" height="20" fill="currentColor"><path d="M8 5v14l11-7z"/></svg>;
const PauseIcon  = () => <svg viewBox="0 0 24 24" width="20" height="20" fill="currentColor"><path d="M6 19h4V5H6v14zm8-14v14h4V5h-4z"/></svg>;
const VolumeIcon = () => <svg viewBox="0 0 24 24" width="20" height="20" fill="currentColor"><path d="M3 9v6h4l5 5V4L7 9H3zm13.5 3A4.5 4.5 0 0 0 14 7.97v8.05c1.48-.73 2.5-2.25 2.5-4.02z"/></svg>;
const MuteIcon   = () => <svg viewBox="0 0 24 24" width="20" height="20" fill="currentColor"><path d="M16.5 12A4.5 4.5 0 0 0 14 7.97v2.21l2.45 2.45c.03-.2.05-.41.05-.63zm2.5 0c0 .94-.2 1.82-.54 2.64l1.51 1.51A8.8 8.8 0 0 0 21 12c0-4.28-2.99-7.86-7-8.76v2.06c2.89.86 5 3.54 5 6.7zM4.27 3L3 4.27 7.73 9H3v6h4l5 5v-6.73l4.25 4.25A6.98 6.98 0 0 1 14 18.97v2.06a9 9 0 0 0 3.97-1.95L19.73 21 21 19.73l-9-9L4.27 3zM12 4L9.91 6.09 12 8.18V4z"/></svg>;
const ExpandIcon = () => <svg viewBox="0 0 24 24" width="20" height="20" fill="currentColor"><path d="M7 14H5v5h5v-2H7v-3zm-2-4h2V7h3V5H5v5zm12 7h-3v2h5v-5h-2v3zM14 5v2h3v3h2V5h-5z"/></svg>;
const ShrinkIcon = () => <svg viewBox="0 0 24 24" width="20" height="20" fill="currentColor"><path d="M5 16h3v3h2v-5H5v2zm3-8H5v2h5V5H8v3zm6 11h2v-3h3v-2h-5v5zm2-11V5h-2v5h5V8h-3z"/></svg>;
const PipIcon    = () => <svg viewBox="0 0 24 24" width="20" height="20" fill="currentColor"><path d="M19 11h-8v6h8v-6zm4 10V3H1v18h22zm-2-1.98H3V4.97h18v14.05z"/></svg>;
```

---

## Angular Implementation

### Service

```typescript
// video-player.service.ts
import { Injectable, signal, computed } from '@angular/core';

export interface PlayerState {
  playing: boolean;
  currentTime: number;
  duration: number;
  volume: number;
  muted: boolean;
  fullscreen: boolean;
  pip: boolean;
}

@Injectable()   // not providedIn: 'root' — scoped to the player component
export class VideoPlayerService {
  private _state = signal<PlayerState>({
    playing: false, currentTime: 0, duration: 0,
    volume: 1, muted: false, fullscreen: false, pip: false,
  });

  readonly state       = this._state.asReadonly();
  readonly progressPct = computed(() => {
    const { currentTime, duration } = this._state();
    return duration ? (currentTime / duration) * 100 : 0;
  });

  patch(partial: Partial<PlayerState>) {
    this._state.update(s => ({ ...s, ...partial }));
  }

  togglePlay(video: HTMLVideoElement) {
    video.paused ? video.play() : video.pause();
  }

  seek(video: HTMLVideoElement, seconds: number) {
    video.currentTime = Math.max(0, Math.min(seconds, video.duration));
  }

  setVolume(video: HTMLVideoElement, v: number) {
    video.volume = Math.max(0, Math.min(1, v));
    video.muted = v === 0;
  }

  async toggleFullscreen(wrapper: HTMLElement) {
    if (!document.fullscreenElement) {
      await wrapper.requestFullscreen();
    } else {
      await document.exitFullscreen();
    }
  }

  async togglePip(video: HTMLVideoElement) {
    if (document.pictureInPictureElement) {
      await document.exitPictureInPicture();
    } else if (document.pictureInPictureEnabled) {
      await video.requestPictureInPicture();
    }
  }
}
```

### Component

```typescript
// video-player.component.ts
import {
  Component, Input, OnInit, OnDestroy, ElementRef,
  ViewChild, inject, HostListener
} from '@angular/core';
import { NgIf } from '@angular/common';
import { VideoPlayerService } from './video-player.service';

@Component({
  selector: 'app-video-player',
  standalone: true,
  imports: [NgIf],
  providers: [VideoPlayerService],   // scoped instance per player
  template: `
    <div
      #wrapper
      class="player-wrapper"
      [attr.data-paused]="!svc.state().playing"
      tabindex="-1"
    >
      <video
        #videoEl
        class="player-video"
        [src]="src"
        [poster]="poster"
        preload="metadata"
        aria-hidden="true"
      ></video>

      <!-- Click-to-play overlay -->
      <button
        class="player-overlay-btn"
        (click)="svc.togglePlay(videoEl)"
        [attr.aria-label]="svc.state().playing ? 'Pause' : 'Play'"
        tabindex="-1"
        aria-hidden="true"
      ></button>

      <div class="player-controls" role="group" aria-label="Video controls">

        <!-- Play/Pause -->
        <button
          class="ctrl-btn"
          (click)="svc.togglePlay(videoEl)"
          [attr.aria-label]="svc.state().playing ? 'Pause' : 'Play'"
          [attr.aria-pressed]="svc.state().playing"
        >
          <svg viewBox="0 0 24 24" width="20" height="20" fill="currentColor" aria-hidden="true">
            <path *ngIf="!svc.state().playing" d="M8 5v14l11-7z"/>
            <path *ngIf="svc.state().playing" d="M6 19h4V5H6v14zm8-14v14h4V5h-4z"/>
          </svg>
        </button>

        <!-- Time -->
        <span class="ctrl-time" aria-live="off">
          {{ formatTime(svc.state().currentTime) }}
          <span aria-hidden="true"> / </span>
          {{ formatTime(svc.state().duration) }}
        </span>

        <!-- Progress -->
        <div
          class="ctrl-progress-wrap"
          (click)="onProgressClick($event)"
          role="presentation"
        >
          <input
            type="range"
            class="ctrl-progress"
            [min]="0"
            [max]="svc.state().duration || 100"
            [step]="1"
            [value]="svc.state().currentTime"
            [attr.aria-valuetext]="formatTime(svc.state().currentTime) + ' of ' + formatTime(svc.state().duration)"
            aria-label="Seek"
            (input)="svc.seek(videoEl, +$any($event.target).value)"
          />
          <div class="ctrl-progress-fill" [style.width.%]="svc.progressPct()"></div>
        </div>

        <!-- Mute -->
        <button
          class="ctrl-btn"
          (click)="svc.setVolume(videoEl, svc.state().muted ? svc.state().volume : 0)"
          [attr.aria-label]="svc.state().muted ? 'Unmute' : 'Mute'"
          [attr.aria-pressed]="svc.state().muted"
        >
          <svg viewBox="0 0 24 24" width="20" height="20" fill="currentColor" aria-hidden="true">
            <path *ngIf="!svc.state().muted" d="M3 9v6h4l5 5V4L7 9H3z"/>
            <path *ngIf="svc.state().muted" d="M16.5 12A4.5 4.5 0 0014 7.97v2.21l2.45 2.45c.03-.2.05-.41.05-.63z"/>
          </svg>
        </button>

        <!-- Volume -->
        <input
          type="range"
          class="ctrl-volume"
          min="0" max="1" step="0.05"
          [value]="svc.state().muted ? 0 : svc.state().volume"
          aria-label="Volume"
          (input)="svc.setVolume(videoEl, +$any($event.target).value)"
        />

        <!-- PiP -->
        <button
          *ngIf="pipEnabled"
          class="ctrl-btn"
          (click)="svc.togglePip(videoEl)"
          [attr.aria-label]="svc.state().pip ? 'Exit picture in picture' : 'Picture in picture'"
          [attr.aria-pressed]="svc.state().pip"
        >
          <svg viewBox="0 0 24 24" width="20" height="20" fill="currentColor" aria-hidden="true">
            <path d="M19 11h-8v6h8v-6zm4 10V3H1v18h22zm-2-1.98H3V4.97h18v14.05z"/>
          </svg>
        </button>

        <!-- Fullscreen -->
        <button
          *ngIf="fullscreenEnabled"
          class="ctrl-btn"
          (click)="svc.toggleFullscreen(wrapper)"
          [attr.aria-label]="svc.state().fullscreen ? 'Exit fullscreen' : 'Enter fullscreen'"
          [attr.aria-pressed]="svc.state().fullscreen"
        >
          <svg viewBox="0 0 24 24" width="20" height="20" fill="currentColor" aria-hidden="true">
            <path *ngIf="!svc.state().fullscreen" d="M7 14H5v5h5v-2H7v-3zm-2-4h2V7h3V5H5v5zm12 7h-3v2h5v-5h-2v3zM14 5v2h3v3h2V5h-5z"/>
            <path *ngIf="svc.state().fullscreen" d="M5 16h3v3h2v-5H5v2zm3-8H5v2h5V5H8v3zm6 11h2v-3h3v-2h-5v5zm2-11V5h-2v5h5V8h-3z"/>
          </svg>
        </button>

      </div>
    </div>
  `,
})
export class VideoPlayerComponent implements OnInit, OnDestroy {
  @Input() src!: string;
  @Input() poster?: string;

  @ViewChild('videoEl', { static: true }) videoElRef!: ElementRef<HTMLVideoElement>;
  @ViewChild('wrapper',  { static: true }) wrapperRef!: ElementRef<HTMLDivElement>;

  svc = inject(VideoPlayerService);

  readonly pipEnabled        = !!document.pictureInPictureEnabled;
  readonly fullscreenEnabled = !!document.fullscreenEnabled;

  private listeners: Array<() => void> = [];

  get videoEl() { return this.videoElRef.nativeElement; }
  get wrapper()  { return this.wrapperRef.nativeElement; }

  ngOnInit() {
    const v = this.videoEl;
    const add = <K extends keyof HTMLVideoElementEventMap>(
      target: EventTarget,
      type: K | string,
      fn: EventListener,
    ) => {
      target.addEventListener(type, fn);
      this.listeners.push(() => target.removeEventListener(type, fn));
    };

    add(v, 'play',            () => this.svc.patch({ playing: true }));
    add(v, 'pause',           () => this.svc.patch({ playing: false }));
    add(v, 'ended',           () => this.svc.patch({ playing: false }));
    add(v, 'timeupdate',      () => this.svc.patch({ currentTime: v.currentTime }));
    add(v, 'loadedmetadata',  () => this.svc.patch({ duration: v.duration }));
    add(v, 'volumechange',    () => this.svc.patch({ volume: v.volume, muted: v.muted }));
    add(v, 'enterpictureinpicture', () => this.svc.patch({ pip: true }));
    add(v, 'leavepictureinpicture', () => this.svc.patch({ pip: false }));
    add(document, 'fullscreenchange', () =>
      this.svc.patch({ fullscreen: !!document.fullscreenElement })
    );
  }

  ngOnDestroy() {
    this.listeners.forEach(fn => fn());
  }

  onProgressClick(e: MouseEvent) {
    const rect = (e.currentTarget as HTMLElement).getBoundingClientRect();
    const ratio = (e.clientX - rect.left) / rect.width;
    this.svc.seek(this.videoEl, ratio * this.svc.state().duration);
  }

  // Keyboard shortcuts — only when the wrapper has focus
  @HostListener('keydown', ['$event'])
  onKey(e: KeyboardEvent) {
    const { currentTime } = this.svc.state();
    switch (e.key) {
      case ' ': case 'k':
        e.preventDefault();
        this.svc.togglePlay(this.videoEl);
        break;
      case 'ArrowRight': e.preventDefault(); this.svc.seek(this.videoEl, currentTime + 5);  break;
      case 'ArrowLeft':  e.preventDefault(); this.svc.seek(this.videoEl, currentTime - 5);  break;
      case 'ArrowUp':    e.preventDefault(); this.svc.seek(this.videoEl, currentTime + 10); break;
      case 'ArrowDown':  e.preventDefault(); this.svc.seek(this.videoEl, currentTime - 10); break;
      case 'm': case 'M': this.svc.setVolume(this.videoEl, this.svc.state().muted ? this.svc.state().volume : 0); break;
      case 'f': case 'F': this.svc.toggleFullscreen(this.wrapper); break;
    }
  }

  formatTime(s: number): string {
    if (isNaN(s)) return '0:00';
    const m = Math.floor(s / 60);
    const sec = Math.floor(s % 60).toString().padStart(2, '0');
    return `${m}:${sec}`;
  }
}
```

---

## Accessibility Checklist

| Requirement | Implementation |
|---|---|
| Video element hidden from AT | `aria-hidden="true"` on `<video>` — the controls describe the interaction |
| Controls group has accessible name | `role="group" aria-label="Video controls"` on the controls bar |
| Play/Pause button label updates on state change | `aria-label` bound to `state.playing` — "Play" or "Pause" |
| Play/Pause button communicates toggle state | `aria-pressed` reflects current playing state |
| Seek input is keyboard-accessible without custom code | `<input type="range">` gets arrow-key seek for free from the browser |
| Seek input has a human-readable value text | `aria-valuetext="1:23 of 4:56"` instead of raw number |
| Time display does not cause constant AT announcements | `aria-live="off"` — do not announce every `timeupdate` tick |
| Fullscreen and PiP buttons communicate pressed state | `aria-pressed` on each button |
| Fullscreen and PiP only rendered when API is available | Feature-detected at render time |
| Keyboard shortcuts do not hijack page scrolling | `e.preventDefault()` only called within the player's focus scope |
| Controls visible to keyboard-only users | `focus-within` CSS selector keeps controls visible when focus is inside |
| Minimum touch target size | `width: 36px; height: 36px` on all `.ctrl-btn` elements |
| Reduced motion respected | `transition: none` applied via `@media (prefers-reduced-motion: reduce)` |

---

## Production Pitfalls

**1. `play()` returns a Promise — unhandled rejections cause console noise**
`videoEl.play()` is async. If the user quickly toggles play/pause, the first `play()` may be interrupted by `pause()`, throwing an `AbortError`. Always wrap the call: `video.play().catch(() => {})`. In React, if the component unmounts before `play()` resolves, you get the same error. Track a mounted flag or abort signal.

**2. `timeupdate` fires every 250ms — do not put expensive recalculations in its handler**
The handler runs at roughly 4Hz. Avoid deriving complex state or calling `setState` with a new object reference on every tick unnecessarily. The only work done in the handler should be updating `currentTime`; everything else (progress percentage, formatted time) should be a derived computed value.

**3. Seeking via `getBoundingClientRect` breaks when the player is inside a CSS transform**
`getBoundingClientRect` returns the visual position relative to the viewport. If any ancestor has `transform`, `zoom`, or `scale`, the calculation is still correct. But if the progress bar is inside a fullscreen element and you calculate against the wrong coordinate space, seeks will land at the wrong position. Always read `rect` from the event's `currentTarget`, not a stored ref.

**4. Fullscreen API enters fullscreen on the wrong element**
Calling `videoEl.requestFullscreen()` puts only the video into fullscreen — your custom controls are gone. Always call `wrapperEl.requestFullscreen()` where `wrapperEl` is the outermost container that includes both the video and the controls. The `fullscreenchange` event fires on `document`, not on your element.

**5. Picture-in-Picture requires a user gesture and a playing video**
`requestPictureInPicture()` throws if the video has never played or if called outside a user-initiated event handler. Also check `document.pictureInPictureEnabled` before rendering the button at all — Safari requires `webkit-` prefixed variant on older versions, and some enterprise environments disable the API entirely.

**6. Keyboard shortcuts swallow arrow keys from the seek slider**
If the `<input type="range">` (seek bar) has focus and you also intercept `ArrowRight`/`ArrowLeft` on the wrapper, you get double-seeking: the native range input moves and your handler also fires. Guard your keyboard handler so it does not fire when the `event.target` is already an `input` or `button`: `if (e.target instanceof HTMLInputElement) return`.

**7. `currentTime` jumps backward at the end of the video**
When `ended` fires, `currentTime` equals `duration`. If you reset `currentTime = 0` in your `ended` handler, the progress bar snaps to zero immediately. Many players instead leave `currentTime` at the end and show a "replay" button. If you do reset it, do so only on an explicit user action — not automatically.

---

## Interview Angle

**Q: "How do you build a custom video player without using the browser's native controls?"**

The core answer is a separation of concerns: the `<video>` element is a media engine — you never replace it or fake it. You hide the native controls by omitting the `controls` attribute, attach a `ref` to the element, and drive it with imperative calls (`play()`, `pause()`, `currentTime = N`). State flows from the media engine back to your UI through event listeners on the video element (`timeupdate`, `loadedmetadata`, `play`, `pause`, `volumechange`) and on `document` for fullscreen events (`fullscreenchange`). The UI is a pure projection of that state. In React, this is a custom hook that attaches all listeners inside a single `useEffect` and returns both state and stable callbacks. In Angular, it is a service with signals that the component template binds to directly.

**Follow-up: "Why do you put the fullscreen listener on `document` instead of the video element?"**

Because the `fullscreenchange` event fires on `document`, not on the element that called `requestFullscreen`. Additionally, the user can exit fullscreen via the Escape key or the browser's own UI — those exits do not go through your code at all. The only reliable place to detect fullscreen changes is `document.addEventListener('fullscreenchange', ...)` combined with reading `document.fullscreenElement` to determine the new state. This is why the fullscreen state in the hook is updated in a `document` listener, not in the `await requestFullscreen()` call itself.

# Video, audio, and tracks

## Overview

HTML5 provides native `<video>` and `<audio>` elements with rich controls, accessibility features through tracks, and extensive JavaScript APIs.

## Video element

### Basic video

```html
<video src="video.mp4" controls>
  Your browser does not support the video tag.
</video>
```

### Multiple formats

```html
<video controls width="640" height="360">
  <source src="video.webm" type="video/webm">
  <source src="video.mp4" type="video/mp4">
  <p>Your browser doesn't support HTML5 video. 
     <a href="video.mp4">Download the video</a>.</p>
</video>
```

### Video attributes

```html
<video 
  controls
  autoplay
  muted
  loop
  playsinline
  poster="thumbnail.jpg"
  preload="metadata"
  width="1920"
  height="1080">
  <source src="video.webm" type="video/webm">
  <source src="video.mp4" type="video/mp4">
</video>
```

### Attribute details

- `controls`: Show playback controls
- `autoplay`: Start playing automatically (requires `muted`)
- `muted`: Mute audio by default
- `loop`: Repeat video
- `playsinline`: Play inline on mobile (not fullscreen)
- `poster`: Image shown before video plays
- `preload`: `none`, `metadata`, or `auto`
- `width`, `height`: Dimensions in pixels

## Audio element

### Basic audio

```html
<audio src="audio.mp3" controls>
  Your browser does not support the audio element.
</audio>
```

### Multiple formats

```html
<audio controls>
  <source src="audio.opus" type="audio/opus">
  <source src="audio.ogg" type="audio/ogg">
  <source src="audio.mp3" type="audio/mpeg">
  <p>Your browser doesn't support HTML5 audio.</p>
</audio>
```

### Audio attributes

```html
<audio 
  controls
  autoplay
  muted
  loop
  preload="auto">
  <source src="audio.mp3" type="audio/mpeg">
</audio>
```

## Track element

Provides text tracks for video and audio (subtitles, captions, descriptions).

### Basic subtitles

```html
<video controls>
  <source src="video.mp4" type="video/mp4">
  <track 
    kind="subtitles" 
    src="subtitles-en.vtt" 
    srclang="en" 
    label="English"
    default>
</video>
```

### Multiple languages

```html
<video controls>
  <source src="video.mp4" type="video/mp4">
  
  <track kind="subtitles" src="subtitles-en.vtt" 
         srclang="en" label="English" default>
  <track kind="subtitles" src="subtitles-es.vtt" 
         srclang="es" label="Español">
  <track kind="subtitles" src="subtitles-fr.vtt" 
         srclang="fr" label="Français">
  <track kind="subtitles" src="subtitles-de.vtt" 
         srclang="de" label="Deutsch">
</video>
```

### Track kinds

```html
<video controls>
  <source src="video.mp4" type="video/mp4">
  
  <!-- Subtitles: translation for non-native speakers -->
  <track kind="subtitles" src="subs-en.vtt" 
         srclang="en" label="English">
  
  <!-- Captions: for deaf/hard-of-hearing (includes sounds) -->
  <track kind="captions" src="captions-en.vtt" 
         srclang="en" label="English Captions">
  
  <!-- Descriptions: audio descriptions for blind users -->
  <track kind="descriptions" src="descriptions-en.vtt" 
         srclang="en" label="English Descriptions">
  
  <!-- Chapters: navigation -->
  <track kind="chapters" src="chapters-en.vtt" 
         srclang="en" label="Chapters">
  
  <!-- Metadata: not visible to user -->
  <track kind="metadata" src="metadata.vtt">
</video>
```

## WebVTT format

### Basic WebVTT file

```vtt
WEBVTT

00:00:00.000 --> 00:00:03.000
Welcome to our video tutorial.

00:00:03.500 --> 00:00:07.000
Today we'll learn about HTML video.

00:00:07.500 --> 00:00:11.000
Let's get started!
```

### Captions with sound effects

```vtt
WEBVTT

00:00:00.000 --> 00:00:03.000
[upbeat music playing]

00:00:03.500 --> 00:00:07.000
Welcome to the show!

00:00:07.500 --> 00:00:09.000
[audience applause]

00:00:09.500 --> 00:00:12.000
Today's topic is exciting.
```

### Cue identifiers and settings

```vtt
WEBVTT

1
00:00:00.000 --> 00:00:03.000 line:10% position:50% align:center
Centered subtitle near top

2
00:00:03.500 --> 00:00:07.000 line:90% position:10% align:left
Left-aligned subtitle near bottom

3
00:00:07.500 --> 00:00:11.000 size:50%
Half-width subtitle
```

### Styling cues

```vtt
WEBVTT

STYLE
::cue {
  background-color: rgba(0, 0, 0, 0.8);
  color: white;
  font-size: 1.2em;
}

::cue(.narrator) {
  color: yellow;
}

::cue(b) {
  font-weight: bold;
}

00:00:00.000 --> 00:00:03.000
<c.narrator>Narrator:</c> Welcome to the story.

00:00:03.500 --> 00:00:07.000
The hero said: <b>I will succeed!</b>
```

### Chapters

```vtt
WEBVTT

Chapter 1
00:00:00.000 --> 00:05:00.000
Introduction

Chapter 2
00:05:00.000 --> 00:12:00.000
Main Content

Chapter 3
00:12:00.000 --> 00:15:00.000
Conclusion
```

## Complete accessible video

```html
<figure>
  <video 
    id="tutorial-video"
    controls
    poster="poster.jpg"
    width="1280"
    height="720"
    preload="metadata">
    
    <!-- Video sources -->
    <source src="tutorial.webm" type="video/webm">
    <source src="tutorial.mp4" type="video/mp4">
    
    <!-- Subtitles for multiple languages -->
    <track kind="subtitles" src="subs-en.vtt" 
           srclang="en" label="English" default>
    <track kind="subtitles" src="subs-es.vtt" 
           srclang="es" label="Español">
    <track kind="subtitles" src="subs-fr.vtt" 
           srclang="fr" label="Français">
    
    <!-- Captions for accessibility -->
    <track kind="captions" src="captions-en.vtt" 
           srclang="en" label="English Captions">
    
    <!-- Audio descriptions -->
    <track kind="descriptions" src="descriptions-en.vtt" 
           srclang="en" label="Descriptions">
    
    <!-- Chapters for navigation -->
    <track kind="chapters" src="chapters.vtt" 
           srclang="en" label="Chapters">
    
    <!-- Fallback -->
    <p>Your browser doesn't support HTML5 video. 
       <a href="tutorial.mp4">Download the video</a>.</p>
  </video>
  
  <figcaption>
    Tutorial: Getting Started with HTML Video
  </figcaption>
</figure>
```

## Responsive video

### Aspect ratio container

```html
<div style="position: relative; padding-bottom: 56.25%; height: 0;">
  <video 
    controls
    style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;">
    <source src="video.mp4" type="video/mp4">
  </video>
</div>
```

### Modern aspect ratio

```html
<div style="aspect-ratio: 16/9; width: 100%;">
  <video controls style="width: 100%; height: 100%;">
    <source src="video.mp4" type="video/mp4">
  </video>
</div>
```

## Format recommendations

### Video formats (2025)

```html
<video controls>
  <!-- Modern: AV1 -->
  <source src="video.av1.mp4" type="video/mp4; codecs=av01.0.05M.08">
  
  <!-- Wide support: VP9 in WebM -->
  <source src="video.webm" type="video/webm; codecs=vp9">
  
  <!-- Fallback: H.264 in MP4 -->
  <source src="video.mp4" type="video/mp4; codecs=avc1.4d002a">
</video>
```

### Audio formats (2025)

```html
<audio controls>
  <!-- Modern: Opus in WebM -->
  <source src="audio.opus" type="audio/opus">
  
  <!-- Alternative: Vorbis in Ogg -->
  <source src="audio.ogg" type="audio/ogg; codecs=vorbis">
  
  <!-- Fallback: AAC in MP4 -->
  <source src="audio.m4a" type="audio/mp4; codecs=mp4a.40.2">
  
  <!-- Universal fallback: MP3 -->
  <source src="audio.mp3" type="audio/mpeg">
</audio>
```

## JavaScript API

### Basic controls

```javascript
const video = document.getElementById('my-video');

// Play/pause
video.play();
video.pause();

// Seek
video.currentTime = 30; // Jump to 30 seconds

// Volume
video.volume = 0.5; // 50%
video.muted = true;

// Playback rate
video.playbackRate = 1.5; // 1.5x speed
```

### Event listeners

```javascript
video.addEventListener('play', () => {
  console.log('Video started playing');
});

video.addEventListener('pause', () => {
  console.log('Video paused');
});

video.addEventListener('ended', () => {
  console.log('Video finished');
});

video.addEventListener('timeupdate', () => {
  console.log('Current time:', video.currentTime);
});

video.addEventListener('loadedmetadata', () => {
  console.log('Duration:', video.duration);
});
```

## Accessibility best practices

1. Always provide captions for speech
2. Include audio descriptions for visual content
3. Use `<track>` elements for all text content
4. Ensure controls are keyboard accessible
5. Don't autoplay with sound
6. Provide transcripts as alternative
7. Use appropriate `kind` values
8. Test with screen readers

## Performance best practices

1. Use `preload="metadata"` by default
2. Use `poster` for video thumbnail
3. Lazy load off-screen videos
4. Provide multiple formats
5. Optimize video bitrate and resolution
6. Use CDN for large files
7. Consider adaptive streaming (HLS, DASH)
8. Compress audio files

## Common patterns

### Background video

```html
<video 
  autoplay 
  muted 
  loop 
  playsinline
  poster="poster.jpg"
  style="width: 100%; height: 100%; object-fit: cover;">
  <source src="background.webm" type="video/webm">
  <source src="background.mp4" type="video/mp4">
</video>
```

### Podcast player

```html
<audio controls preload="metadata">
  <source src="episode-42.mp3" type="audio/mpeg">
  <track kind="captions" src="captions.vtt" srclang="en" label="English">
  <track kind="chapters" src="chapters.vtt" srclang="en" label="Chapters">
</audio>
```

### Video with transcript

```html
<video controls>
  <source src="lecture.mp4" type="video/mp4">
  <track kind="captions" src="captions.vtt" srclang="en" label="English" default>
</video>

<details>
  <summary>Transcript</summary>
  <p>Full text transcript goes here...</p>
</details>
```

## Key takeaways

- Use native `<video>` and `<audio>` elements for media
- Provide multiple format sources for compatibility
- Always include `<track>` elements for accessibility
- Use WebVTT format for captions and subtitles
- Set `preload` appropriately for performance
- Don't autoplay videos with sound
- Include fallback content for unsupported browsers
- Use `controls` attribute for user control
- Provide poster images for videos
- Test across browsers and devices

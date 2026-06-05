# Implement a Stopwatch / Countdown Timer

## The Idea

**In plain English:** A stopwatch is a tool that tracks how much time has passed since you pressed "start", and a countdown timer counts down from a set amount of time to zero. In code, you keep checking the clock over and over to update the numbers on screen.

**Real-world analogy:** Think of a coach at a track meet using a physical stopwatch. She presses "start" when the runner leaves the blocks, watches the second hand sweep forward, presses "lap" to note each split, and presses "stop" when the runner crosses the finish line.

- The coach pressing "start" = calling `requestAnimationFrame` to begin the update loop
- The second hand sweeping forward = the `tick` function running on every browser repaint to recalculate elapsed time
- The coach's memory of when she pressed "start" = `startTime.current` stored in a ref

---

## Stopwatch

### Core Hook

```ts
function useStopwatch() {
  const [elapsed, setElapsed] = useState(0);    // milliseconds
  const [running, setRunning] = useState(false);
  const [laps, setLaps] = useState<number[]>([]);
  const startTime = useRef<number | null>(null);
  const animFrameRef = useRef<number | null>(null);
  const accumulatedRef = useRef(0); // elapsed before last pause

  const tick = useCallback(() => {
    if (startTime.current !== null) {
      setElapsed(accumulatedRef.current + (performance.now() - startTime.current));
      animFrameRef.current = requestAnimationFrame(tick);
    }
  }, []);

  const start = useCallback(() => {
    startTime.current = performance.now();
    setRunning(true);
    animFrameRef.current = requestAnimationFrame(tick);
  }, [tick]);

  const pause = useCallback(() => {
    if (animFrameRef.current) cancelAnimationFrame(animFrameRef.current);
    accumulatedRef.current = elapsed;
    startTime.current = null;
    setRunning(false);
  }, [elapsed]);

  const reset = useCallback(() => {
    if (animFrameRef.current) cancelAnimationFrame(animFrameRef.current);
    startTime.current = null;
    accumulatedRef.current = 0;
    setElapsed(0);
    setLaps([]);
    setRunning(false);
  }, []);

  const lap = useCallback(() => {
    setLaps(prev => [...prev, elapsed]);
  }, [elapsed]);

  useEffect(() => () => {
    if (animFrameRef.current) cancelAnimationFrame(animFrameRef.current);
  }, []);

  return { elapsed, running, laps, start, pause, reset, lap };
}
```

### Why `requestAnimationFrame` over `setInterval`?

`requestAnimationFrame` updates in sync with the browser's repaint cycle (~60fps), giving smooth display. `setInterval(fn, 10)` is throttled in background tabs and not perfectly timed. `performance.now()` for elapsed time is more accurate than counting intervals.

### Formatting Elapsed Time

```ts
function formatTime(ms: number): string {
  const minutes = Math.floor(ms / 60000);
  const seconds = Math.floor((ms % 60000) / 1000);
  const centiseconds = Math.floor((ms % 1000) / 10);
  return `${String(minutes).padStart(2, '0')}:${String(seconds).padStart(2, '0')}.${String(centiseconds).padStart(2, '0')}`;
}
// formatTime(75430) → "01:15.43"
```

### Stopwatch Component

```tsx
function Stopwatch() {
  const { elapsed, running, laps, start, pause, reset, lap } = useStopwatch();
  const bestLap = laps.length > 0 ? Math.min(...laps) : null;
  const worstLap = laps.length > 0 ? Math.max(...laps) : null;

  return (
    <div className="stopwatch">
      <div
        className="display"
        aria-live="off" // don't announce every tick
        aria-label={`Elapsed time: ${formatTime(elapsed)}`}
        role="timer"
      >
        {formatTime(elapsed)}
      </div>

      <div className="controls">
        <button onClick={running ? pause : start} aria-label={running ? 'Pause' : 'Start'}>
          {running ? 'Pause' : 'Start'}
        </button>
        <button onClick={lap} disabled={!running} aria-label="Record lap">
          Lap
        </button>
        <button onClick={reset} aria-label="Reset">
          Reset
        </button>
      </div>

      {laps.length > 0 && (
        <ol reversed aria-label="Lap times">
          {[...laps].reverse().map((lapTime, i) => {
            const lapIndex = laps.length - i - 1;
            const isBest = lapTime === bestLap;
            const isWorst = lapTime === worstLap && laps.length > 1;
            return (
              <li key={lapIndex} className={isBest ? 'best' : isWorst ? 'worst' : ''}>
                <span>Lap {laps.length - i}</span>
                <span>{formatTime(lapTime)}</span>
              </li>
            );
          })}
        </ol>
      )}
    </div>
  );
}
```

---

## Countdown Timer

```ts
function useCountdown(initialSeconds: number) {
  const [remaining, setRemaining] = useState(initialSeconds);
  const [running, setRunning] = useState(false);
  const endTime = useRef<number | null>(null);
  const animFrameRef = useRef<number | null>(null);

  const tick = useCallback(() => {
    if (endTime.current !== null) {
      const left = Math.max(0, endTime.current - performance.now());
      setRemaining(left / 1000);
      if (left > 0) {
        animFrameRef.current = requestAnimationFrame(tick);
      } else {
        setRunning(false);
        endTime.current = null;
        // Optionally: play a sound or fire a callback
      }
    }
  }, []);

  const start = useCallback(() => {
    endTime.current = performance.now() + remaining * 1000;
    setRunning(true);
    animFrameRef.current = requestAnimationFrame(tick);
  }, [remaining, tick]);

  const pause = useCallback(() => {
    if (animFrameRef.current) cancelAnimationFrame(animFrameRef.current);
    endTime.current = null;
    setRunning(false);
  }, []);

  const reset = useCallback(() => {
    if (animFrameRef.current) cancelAnimationFrame(animFrameRef.current);
    endTime.current = null;
    setRemaining(initialSeconds);
    setRunning(false);
  }, [initialSeconds]);

  useEffect(() => () => {
    if (animFrameRef.current) cancelAnimationFrame(animFrameRef.current);
  }, []);

  const progress = remaining / initialSeconds; // 0-1

  return { remaining, running, progress, start, pause, reset };
}

function CountdownTimer({ initialSeconds = 60 }: { initialSeconds?: number }) {
  const { remaining, running, progress, start, pause, reset } = useCountdown(initialSeconds);
  const isFinished = remaining === 0;

  return (
    <div className="countdown">
      {/* Circular progress indicator */}
      <svg width="120" height="120" aria-hidden="true">
        <circle cx="60" cy="60" r="50" fill="none" stroke="#e2e8f0" strokeWidth="8" />
        <circle
          cx="60" cy="60" r="50"
          fill="none"
          stroke={isFinished ? '#ef4444' : '#3b82f6'}
          strokeWidth="8"
          strokeDasharray={`${2 * Math.PI * 50}`}
          strokeDashoffset={`${2 * Math.PI * 50 * (1 - progress)}`}
          transform="rotate(-90 60 60)"
          style={{ transition: 'stroke-dashoffset 0.1s linear' }}
        />
      </svg>

      <div
        role="timer"
        aria-live="off"
        aria-label={`${Math.ceil(remaining)} seconds remaining`}
        className={`time ${isFinished ? 'finished' : ''}`}
      >
        {Math.ceil(remaining)}s
      </div>

      <button onClick={running ? pause : start} disabled={isFinished}>
        {running ? 'Pause' : 'Start'}
      </button>
      <button onClick={reset}>Reset</button>

      {isFinished && (
        <div role="alert" aria-live="assertive">Time's up!</div>
      )}
    </div>
  );
}
```

---

## Common Interview Questions

**Q: Why `performance.now()` instead of `Date.now()`?**
`performance.now()` returns high-resolution time (sub-millisecond precision) and is not affected by system clock changes. `Date.now()` could jump if the user changes the clock or during daylight saving transitions.

**Q: Why store end time instead of decrementing remaining time in each frame?**
Subtracting elapsed time per frame accumulates floating-point errors. Computing `endTime - now` always gives the exact remaining time regardless of frame rate variations or background tab throttling.

**Q: How would you persist timer state across page refreshes?**
Store `{ running, remaining, endTime }` in `localStorage`. On mount, check localStorage. If `running` is true, recalculate remaining time: `remaining = storedEndTime - Date.now()` and continue from there. If `remaining <= 0`, the timer finished while the page was closed.

# Whiteboard: Decomposition and Complexity Narrative

## Overview

The first 5 minutes of any system design interview decide how the rest goes. Interviewers at staff level are grading you on whether you can take an ambiguous problem, decompose it into clear subsystems, identify the hard parts, and set up a structured discussion — all before writing a single line of code or drawing a component. This document is the framework for doing that consistently.

---

## The Opening Protocol (First 5 Minutes)

### Step 1: Restate the Problem (30 seconds)

Before asking any clarifying questions, restate what you heard. This does two things: confirms you understood correctly, and buys you time to think.

> "So we're designing a collaborative document editor — multiple users editing the same document simultaneously, changes appearing in real time, with a web frontend. Let me make sure I understand the scope before diving in."

### Step 2: Ask 3–4 Targeted Clarifying Questions (2 minutes)

Good clarifying questions reveal your mental model. Bad ones waste time on details that don't matter yet.

**Good questions — they affect architecture:**
- "What's the expected scale? Number of concurrent users per document? Total users?"
- "Do we need offline support — can users edit without internet and sync later?"
- "Is this a new product or does it need to integrate with an existing auth system?"
- "What's the consistency model — can two users briefly see different states, or must it be strict real-time sync?"

**Bad questions — too early to matter:**
- "What database should we use?"
- "Should we use React or Angular?"
- "How many microservices?"

The rule: if the answer changes what major components you'd build, ask it. If it only changes implementation detail, defer it.

### Step 3: State Your Assumptions (30 seconds)

After questions, state what you're assuming. This prevents the interviewer from surprising you later with a constraint you could have accounted for.

> "I'll assume: web-only for now (no mobile), up to 50 concurrent users per document, we need real-time sync (not eventual), and we don't need offline mode for v1. If any of those are wrong, let me know and I'll adjust."

### Step 4: Name the Hard Problems Upfront (2 minutes)

This is the highest-signal thing you can do. Before drawing any boxes, say:

> "The interesting problems I see in this system are:
>
> 1. **Conflict resolution** — two users editing the same paragraph. Last-write-wins loses data. We need either OT or CRDT.
>
> 2. **Latency** — users expect to see their own keystrokes instantly, but we need to sync to the server. These are in tension.
>
> 3. **Presence** — showing who else is in the document and where their cursor is. Simple to show, expensive to scale.
>
> 4. **Reconnection** — user loses connection for 30 seconds. How do we sync the gap?
>
> I'll address each of these as we go. Let me start with the overall system shape, then drill into conflict resolution since that's the hardest."

An interviewer who hears this knows they're talking to a staff engineer. You've demonstrated awareness of the problem space before they've had to prompt you.

---

## Decomposition Patterns

### Pattern: By Data Flow

Trace the path of user input through the system:

```
User types → Local state update → Sync operation →
Server receive → Broadcast to other clients → Remote state update → UI update
```

Each arrow is a potential subsystem. Name each one and its responsibilities.

### Pattern: By Failure Domain

Ask: what can fail independently?

```
Authentication: if auth is down, nobody can sign in — but existing sessions continue
Real-time sync: if WebSocket fails, fall back to polling
Document save: if save fails, user should see an error but not lose unsaved work
Presence: if presence fails, documents still work — just no cursors
```

This decomposition tells you what to make optional vs required.

### Pattern: By User Journey

Trace the critical path from the user's perspective:

```
1. User opens document (fast render required — LCP)
2. User starts typing (instant feedback required — no latency)
3. User sees collaborator changes (real-time sync)
4. User shares document (permission model)
5. User recovers from lost connection (resilience)
```

Each step has different requirements. A slow step 1 kills perceived performance. A slow step 2 breaks the core experience.

---

## Naming the Hard Parts: Examples by Problem Type

### Real-Time Collaboration

> "The hardest part is conflict resolution. The naive approach — last write wins — silently loses user work, which is catastrophic in a document editor. The two mature approaches are Operational Transformation (OT) — what Google Docs uses — and CRDTs — what Figma and some others use. OT is harder to implement but can produce more natural merge results. CRDTs are easier to reason about but can produce larger document sizes. For this interview I'll sketch the OT approach since it's more battle-tested for rich text."

### Infinite Feed

> "The hard part is feed generation at scale. The naive approach: on load, query the database for posts from all accounts you follow, ordered by time. This works at 1K users. At 1M users, you're following 500 accounts each — that's a join across 500M rows. The two approaches are fan-out on write (pre-compute the feed when a post is created) and fan-out on read (compute on request). Fan-out on write is fast to read but wastes storage and is slow to write. Fan-out on read is slow but accurate. Twitter famously uses a hybrid."

### Design System at Scale

> "The hard part isn't building the components — it's the versioning and adoption strategy. If I ship a breaking change to Button, 50 teams break. The constraints I'd design around: (1) backward compatibility for at least one major version; (2) automated codemods for migrations; (3) visual regression tests that run against every consumer on every change. Without those three, the design system becomes a liability instead of an asset."

---

## Signaling Depth Without Going Into Detail

You can't cover everything in 45 minutes. The skill is telegraphing that you know what you're skipping:

> "I'm going to focus on the conflict resolution algorithm — that's the novel part. For authentication I'd use whatever the company's existing auth system is, and for the database I'd use Postgres since it's a safe default until we have a specific reason to change. If you want to go deeper on either of those, I can, but I think conflict resolution is where the interesting design decisions are."

This shows:
- You know auth and DB are necessary
- You're not going to waste time on known-good defaults
- You're directing the conversation to the hard problem

---

## The "Breadth vs Depth" Decision

Interviewers give you about 45 minutes. You have a choice: broad coverage of the whole system, or deep coverage of one subsystem.

**Read the signal:** If the interviewer keeps saying "tell me more about X" — go deep on X. If they say "let's move on" after your first explanation — they want breadth.

**When to go deep without prompting:**
- The problem has a genuinely novel algorithmic challenge (CRDT, OT, rate limiting with windowing)
- The failure mode of the naive approach is severe (data loss, security vulnerability)
- The trade-off is non-obvious and would surprise most engineers

**When to stay shallow:**
- The component is standard infrastructure (load balancer, CDN, auth service)
- You've already signaled awareness of the depth available
- You're running low on time

---

## Recovering from Going in the Wrong Direction

You've been designing for 15 minutes and the interviewer asks: "How would this change if we needed offline support?"

This is not a trap — it's information. The correct response:

> "Good catch — that changes the conflict resolution approach significantly. With offline support, we can't assume a central server arbitrates conflicts in real time. We need a CRDT that can merge changes that happened independently on disconnected clients. Let me revise the data model to use a CRDT representation instead of OT..."

Then update the design. Don't defend the original approach when given new constraints. This is exactly what the interviewer wants to see.

---

## The Closing Summary

In the last 2–3 minutes, give a structured summary:

> "To recap: we've designed [system] with [key architectural decisions].
>
> The main trade-off we've accepted is [X] in exchange for [Y].
>
> The two things I'm least confident about and would want to validate with a prototype are [A] and [B].
>
> Given more time I'd add [observability / testing / resilience] which I left out to focus on the core design."

This shows: you can zoom out, you know what you don't know, and you distinguish between MVP and production-ready.

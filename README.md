# Experiment: Google Labs Opal meditation app.

## Goal:
Explore long-form TTS generation and audio queue handling inside Opal.

## What worked:
- Script chunking
- Multi-node TTS generation

## Limitations discovered:
- Audio lifecycle issues after first chunk
- State persistence challenges
- Tool not ideal for multi-step orchestration

## Conclusion:
Fun experiment. Good for prototyping, not ideal for complex stateful audio apps.

# 🧘 Aeri – Long-Form Meditation Playback System

Aeri is a structured, long-form meditation audio system built inside a constrained node-based environment (e.g., low-code / experimental app builder).

The system overcomes platform limitations such as:

* ⏱️ ~44-second maximum per TTS node
* 🎧 Audio playback stopping after first chunk
* ⚠️ Optional risk flag responses
* 🔁 Ambient audio lifecycle conflicts

This document explains the architecture and design decisions.

---

# 🎯 Project Goals

* Generate ~5 minutes of continuous meditation narration
* Respect 44-second per TTS node limit
* Maintain uninterrupted narrative flow
* Queue multiple TTS chunks sequentially
* Keep ambient music looping underneath voice
* Prevent crashes if `risk_flag` returns no response
* Ensure system stability even if one TTS node fails

---

# 🏗 Architecture Overview

## 1️⃣ Risk Flag Handling

The system evaluates `risk_flag` but:

* If `risk_flag` returns `null`, `undefined`, or no response → continue execution.
* Only interrupt the flow if a **high-risk value** is explicitly returned.
* Log missing risk_flag responses for debugging.
* Never crash the meditation pipeline due to empty risk_flag output.

This ensures graceful degradation instead of hard failure.

---

## 2️⃣ Script Generation

* A single continuous meditation script (~5 minutes) is generated.
* The text is written as **one uninterrupted narrative**.
* No repeated introductions.
* No restarting tone between segments.
* Natural flow from start to finish.

Important:
Text is generated once, then segmented.
It is NOT regenerated per chunk.

---

## 3️⃣ Text Segmentation Strategy

Due to the ~44-second TTS constraint:

* The full script is split into **7 sequential chunks**.
* Each chunk ≈ 650–750 characters (approx. 44 seconds of speech).
* Chunks are strictly consecutive.
* No summarization.
* No rephrasing.
* No repetition.

Segment Flow:

```
Full Script
   ↓
Chunk 1
Chunk 2
Chunk 3
Chunk 4
Chunk 5
Chunk 6
Chunk 7
```

---

## 4️⃣ TTS System Design

Each chunk is sent to a separate TTS node:

* TTS_1 → Chunk 1
* TTS_2 → Chunk 2
* ...
* TTS_7 → Chunk 7

Each node outputs an audio file or blob.

Failure handling:

* If one TTS node fails:

  * Log the error
  * Continue playing remaining chunks
  * Do not crash entire session

---

# 🎧 Audio Playback Architecture (Critical Fix)

## 🚨 Problem Solved

Initial issue:

* First chunk plays
* Subsequent chunks go silent

Root cause:

* Multiple audio elements being created/destroyed
* `onended` event not chaining properly
* Playback promise not awaited

---

## ✅ Correct Solution: Single Persistent Audio Player

The system uses:

### One persistent voice audio instance

Instead of:

```
TTS_1 → Audio Player 1
TTS_2 → Audio Player 2
```

We use:

```
ttsQueue = [audio1, audio2, audio3, ... audio7]
```

Playback logic:

1. Load first source
2. Play
3. On `ended` event:

   * Increment index
   * Set new source
   * Await play()
4. Continue until queue ends

Key requirements:

* Do NOT recreate audio element between chunks
* Await `play()` promise
* Ensure next source is fully loaded before playing
* Add optional 200ms delay between transitions (stability buffer)
* Log each chunk start/end

---

# 🌊 Ambient Music System

Ambient audio:

* Separate persistent audio instance
* Starts before TTS_1
* Loops continuously
* Volume set to ~30%
* Must NOT stop during voice transitions
* Must NOT restart between chunks

Voice and ambient are fully decoupled.

---

# 🛡 Stability Rules

The system must:

* Never crash due to empty risk_flag
* Never reset script mid-playback
* Never overlap TTS chunks
* Continue playback even if one segment fails
* Log errors without stopping entire meditation

---

# 🔄 Execution Flow

```
1. Evaluate risk_flag
      ↓
2. Generate full meditation script
      ↓
3. Split into 7 sequential chunks
      ↓
4. Generate TTS for each chunk
      ↓
5. Start ambient loop
      ↓
6. Begin sequential TTS queue playback
      ↓
7. Complete full 5-minute session
```

---

# 📏 Design Constraints

| Constraint                | Solution                 |
| ------------------------- | ------------------------ |
| 44s TTS limit             | Script segmentation      |
| Audio stops after chunk 1 | Single persistent player |
| Node-based environment    | Queue-based logic        |
| Optional risk flag        | Graceful handling        |
| Ambient reset issues      | Separate audio instance  |

---

# 🚀 Future Improvements

* Dynamic chunk count based on script length
* Preload next chunk while current plays
* Smooth fade transitions between chunks
* Adjustable narration speed
* Adaptive meditation length
* Offline audio merging

---

# 🧠 Engineering Philosophy

This system is built around:

* Constraint-aware architecture
* Graceful degradation
* State persistence
* Deterministic audio sequencing
* Stability over novelty

Rather than fighting platform limits, the design embraces them and structures around them.

## Note - there can be issues with the audio sync due to free AI usage and Opal being a new tool


## The process flow

<img width="1377" height="677" alt="image" src="https://github.com/user-attachments/assets/bfc32321-7b74-46e5-b7bd-39f297753f9b" />


## Models used 
* Gemini 2.5
* Gemini 3 Pro
* Audio LM
* Lyria for music generation







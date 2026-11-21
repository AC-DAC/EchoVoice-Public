<img src="assets/icon.svg" width="72" alt="EchoVoice icon: a waveform on a dark tile">

# EchoVoice — natural speech, judged before it ships. (Public)

A local voice engine on the Mac: text in, natural-sounding speech out. One engine with thin front-ends: pre-rendered Korean audio for [Lingus](https://github.com/AC-DAC/Lingus-Public), document narration in a local studio, and the voice of a desktop assistant.

Every take is judged before it's used. Failing takes are re-rolled, and a line that still fails after the cap is flagged for review, never shipped silently.

This is a showcase repo: README only. The source is private.

---

## The problem

Lingus needed hundreds of Korean voice lines that sound like a person, not a screen reader. The cloud voices it started with didn't: in a blind listening set with Azure's Korean voice hidden among the candidates, it was rejected on all 10 lines.

Paying a cloud TTS service per character was the other option. The Mac and the homelab were already there, so self-hosting an open model was cheaper and kept every line of text and audio on the machine.

The harder problem turned out to be quality control. An open TTS model invents rhythm and emphasis on every take, so the same line can come out well or badly. EchoVoice is mostly the machinery around the model: judging each take, picking the best, and measuring what "sounds right" actually means.

---

## Architecture

```
┌──────────────────────────────────┐
│  Front-ends (thin)               │
│  Lingus lines · studio uploads   │
│  (PDF / .txt / .md) · assistant  │
└────────────┬─────────────────────┘
             │ 1. lines in: key, text, voice, speed, style
             ▼
┌──────────────────────────────────┐
│  Text                            │
│  spoken form (numbers, dates,    │
│  units, URLs) · clause chunking  │
└────────────┬─────────────────────┘
             │ 2.
             ▼
┌──────────────────────────────────┐
│  Render — Qwen3-TTS 1.7B on MLX  │
│  Base (cloned voices) ·          │
│  CustomVoice (fine-tuned voices) │
└────────────┬─────────────────────┘
             │ 3. each take
             ▼
┌──────────────────────────────────┐
│  Judge                           │
│  words heard · dropped words ·   │
│  voice match · quality · signal  │
└────────────┬─────────────────────┘
             │ 4. pass → keep · fail → re-roll (up to 4)
             ▼
┌──────────────────────────────────┐
│  Out                             │
│  breath trim · mp3 · manifest    │
└──────────────────────────────────┘
```

**Engine first, front-ends thin.** Every front-end shares the same engine, judge and voice presets. A front-end only turns its input into lines and its output into files.

**A file contract, not imports.** Other projects talk to EchoVoice through files: lines in, audio plus `manifest.json` out. No imports across repos and no hard-coded paths into them. Lingus serves the finished clips as static files from a Raspberry Pi, so they work from any device, even while the Mac is asleep.

**Two processes.** The studio (web page, audio, edits) loads no models: about 56 MB fresh, down from about 1 GB before the split. The models live in a render worker that starts on demand, listens only to the studio on localhost, and exits after 10 idle minutes.

---

## The judge

Each take is transcribed and compared to its text, then checked for voice and audio quality.

| Check | How |
|---|---|
| Words heard | whisper.cpp large-v3-turbo + Silero VAD, compared to the text |
| Dropped words | A second, literal transcript (MMS-1b) |
| Voice match | Speaker embedding (ECAPA) against the voice's reference |
| Quality | Automatic MOS prediction (UTMOS) |
| Signal | Pace, abrupt endings, noise, long gaps |

- **Strict and Lenient.** Strict gates batch renders. Lenient is for real-time use and the last-resort pick when Strict runs out of takes.
- **Measurement fixes vs leniency.** Some rules only stop the judge rejecting *correct* takes because of how Whisper writes things: Korean compared by sound as well as spelling (the same converter Lingus uses for grading), numbers compared as digits either way, typography folded. Both versions use those. Real leniency (a lower pass mark, accent as a hint) is Lenient-only.
- **Calibrated by ear.** Thresholds are set from takes marked by ear, and a planted-failures bench measures what the judge catches on deliberately broken clips.
- **Honest limits.** The judge catches broken takes: misreads, dropped words, truncation, silence. It doesn't catch what the ear rejects for tone or emphasis. By-ear review is still the final gate, and the studio is built around making it fast.

---

## Measure before tuning

"Sounds natural" is decided by blind listening tests, not by settings that look right.

- **Automatic signals couldn't predict the ear.** After the first review of 248 Korean clips, every automatic signal was scored against the marks: Whisper word confidence AUC 0.56–0.60, pace 0.53, transcript match 0.50 (a coin flip). Whisper hears the right words even when they're mushy, so what gets rejected (tone, odd emphasis, uneven pace) has no metric. More labels don't fix a missing signal.
- **Blind sets, built properly.** The same lines in several versions, shuffled and unlabelled. Each set has a **control** (Azure's voice went in blind and was rejected 10/10, settling the direction in one set), a **hidden duplicate** (the same take twice, marked alike on 8 of 10 lines, which measures how consistent the ear is), **reject reasons** (too fast, unclear, cut off, odd emphasis, tone) and fresh lines each time.
- **Compare within a set, not across sets.** Absolute approval rates drift with fatigue and a rising bar. Only versions inside one set are comparable.
- **Settings hit a ceiling.** Temperature, speed, instructions and chunking all landed the stock Korean voice at 40–50% acceptable takes.
- **So the voice was fine-tuned on approvals.** Qwen's official single-speaker fine-tuning, ported to Apple Silicon (MPS) on a 24 GB Mac, trained on takes approved by ear. The fine-tuned voice scored 39/40 in a blind sample and now voices every Korean line in Lingus.

---

## Voices

| Voice | What |
|---|---|
| `seolyeong` | Seol-yeong (설영). Korean, female. Fine-tuned from Qwen's stock sohee speaker on takes approved by ear. Lingus's voice. |
| `eggy` | English, bright and bubbly. The desktop assistant's voice. Designed from a written description with Qwen's voice design model, picked by ear, then locked by cloning its own approved take so it stays the same voice. |
| `eggy-ko` | The same voice speaking Korean: a cross-lingual clone from the English reference. |
| `sohee-calm` | Stock sohee with tuned sampling. The control in blind tests and the base for fine-tuning. |

A voice is a small preset file (plus a reference clip for clones, or weights for a fine-tune), so adding one doesn't touch the engine.

---

## The studio

A local web app for everything that isn't Lingus.

- **Create.** Type a line, or upload a PDF, .txt or .md file. Documents are split into headings and paragraphs and narrated into section mp3s plus one full file.
- **Library.** Play finished narrations with bookmarks, listened ticks, a sleep timer and per-chapter summaries written by a local LLM (Gemma 4 12B through Ollama).
- **Flagged lines.** Flagged lines are listed with why they failed. **Give me 3 takes** renders three fresh versions beside the current one; keeping one logs the chosen and rejected audio as training pairs for the next fine-tune.
- **Voices.** Hear a sample of each voice.

The assistant front-end is a Claude Code hook: each reply is summarised and spoken in `eggy`'s voice through the studio's HTTP API.

---

## Engineering notes

**A 68 GB "leak" that wasn't one.** After 4.5 days rendering a long document, the studio reached a 68 GB footprint on a 24 GB Mac. Nothing was holding a reference. Each neural check (UTMOS, ECAPA, MMS) keeps memory in proportion to the longest audio it has judged, and in one long-lived process the three ratcheted together instead of levelling off. Every part tested alone on short clips was flat; the bug only appeared with long audio and all three together. Fix: judge long audio in 60 s windows and release memory after each take. Same loop: +9.9 GB and rising → **+0.5 GB, flat**, with zero changed verdicts on 40 long paragraphs. Lesson: test with the biggest realistic input, and test the parts together.

**Joining 32 hours of audio.** The first join kept every decoded clip and concatenated the whole narration into one array: a **42.4 GB** peak. It now streams each section clip by clip into a file on disk and encodes the full mp3 from those files: **263 MB** peak, with output **byte-identical** to the old join.

**Streaming playback.** Seeking in a 1.3 GB mp3 made the studio read the whole requested range into memory before sending it, and players ask for "the rest of the file" then hang up. It now streams 1 MB at a time and treats a hang-up as normal: five seeks went from +2.8 GB to flat.

**Clipped first words.** Voice activity detection at its default threshold heard the assistant voice's first word late, so the trim cut it. Each voice now carries its own detection threshold, used by both the trim and the judge's silence check.

---

## Rejected alternatives

- **Azure's Korean voices** — rejected 10/10 in a blind set.
- **A paid cloud TTS service** — per-character cost, and text and audio leave the machine. The Mac was already there.
- **A live TTS service for Lingus** — would only work while the Mac is awake. Pre-rendered static clips work everywhere.
- **Docker** — no Metal GPU access inside Docker on macOS.
- **Slowing audio with ffmpeg** — 0.8× time-stretching smeared consonants. Speed has to come from the model.
- **Inline markup for emphasis and pauses** — Qwen3-TTS has no word-level controls; tags like `[pause]` were read aloud or garbled.
- **The 0.6B model for the assistant voice** — near parity in a blind test (6/8 vs 7/8 for 1.7B) but slower per line because of more re-rolls. Kept 1.7B.

---

## Privacy and security

- Rendering and judging run on the Mac: no cloud TTS, and document text and audio stay on the machine.
- The render worker binds to localhost and accepts requests only from the studio.
- Model weights, renders and uploaded documents are gitignored; tests use neutral examples, never the content of a real document.
- Pre-commit hooks scan every commit for secrets and private terms before it leaves the Mac.

---

## Stack

`Python` `Qwen3-TTS` `MLX` `whisper.cpp` `Silero VAD` `MMS` `SpeechBrain ECAPA` `UTMOS` `PyTorch (MPS)` `Ollama` `Gemma 4` `ffmpeg` `pytest`

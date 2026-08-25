# CLAUDE.md — Scam Shield · iQOO Hackathon 2026

## 0. Mission

Build **Scam Shield**: an Android app that listens to a phone call on speakerphone and tells the user, in real time, when the person on the other end is running a scam — with **every byte of processing happening on the device**.

One sentence, the way it should be said to a judge:

> "Put the call on speaker. Your phone tells you it's a scam while they're still talking. And the app can't send your call anywhere — it doesn't have permission to touch the internet."

This is built for the **iQOO Hackathon 2026 City Battles**. It must be finished, working, and demonstrable inside a 30-hour window, on a loaner iQOO 15, with the laptop restricted for 55% of that time.

**The organizers' own framing applies: a well-built simple product beats a broken complex product.** Everything in this document is written to protect that.

---

## 1. Why this project, specifically

Three research findings drove this choice. Claude should understand them, because they explain why certain decisions in this document are non-negotiable.

**1. The single strongest predictor of winning is a problem the judge has personally felt.** Every judge in that room in India has received a scam call — digital arrest, fake KYC, OTP harvesting, UPI "wrong transfer" refunds. No explanation is needed. The pitch starts at full speed.

**2. Constraints should be the product, not compliance.** A 2026 winner (docX, a fully-local medical scribe on a €150 box) won on exactly this: "fully local, privacy by design" beat "we called an AI API." The iQOO brief demands phone-first and a local model. Scam Shield is a project where cloud processing would be *absurd* — nobody wants their phone calls uploaded to be checked for fraud. The constraint is the pitch.

**3. Every documented on-device AI hackathon winner was a one-job app.** GameSense (cricket commentary), EdgeFit (posture), E.M.Pilot (email), Medly (medical jargon), File Fairy (file search). Not one was a multi-capability platform. Scam Shield does one job.

---

## 2. The hackathon — facts to build against

| | |
|---|---|
| Event | iQOO Hackathon 2026 · City Battles (iQOO × Reskilll) |
| Format | 30 hours. Clock starts Sat 10:00, hacking from 11:00, through Sunday awards |
| Target city | Pune (5–6 Sep) or Chennai (12–13 Sep). **Not Bengaluru — 29–30 Aug is too soon** |
| Track | **FinTech & Commerce** (primary). Open Innovation is the fallback |
| Device | iQOO 15 loaner — Snapdragon 8 Elite Gen 5, 12–16GB RAM, Android 16 / OriginOS 6, X+Z axis haptic motor |
| Team | 1–3 builders. Student and professional buckets do not mix |
| Origination | **Original work only — code written during the event window.** Pre-event idea drafting is permitted. No carrying in a built app |
| Build constraint | **55% of build time is Red Light: iQOO phone only, laptops restricted, everything routed through Office Kit.** 45% Green Light (phone + laptop) |
| Advancement | Two scored evaluation rounds + Top 10 pitch. Top 6 per city advance to the Grand Finale |

### The rubric, and how Scam Shield attacks each line

| Criterion | Weight | Our position |
|---|---|---|
| End product quality | 30% | Two components only — ASR and classification. Fewer stages than any alternative we considered. This is where we win by finishing |
| Novelty & impact | 20% | Real-time on-device scam detection during a live call. Locally relevant, immediately legible |
| Creative phone use | 15% | Mic, live audio processing, directional haptics, and a function only a phone has — the call itself. HackTracker measures this automatically |
| Technical depth | 15% | Streaming ASR + a two-layer detector (deterministic signals always-on, SLM triggered) + evidence-backed risk scoring |
| Office Kit usage | 10% | Genuine part of the workflow — must be planned, not bolted on |
| Demo & presentation | 10% | **A judge calls the phone and watches it flag them live.** The strongest demo shape available |

**25% of the score (creative phone use + Office Kit) is captured automatically by HackTracker.** It measures counts and durations of phone and Office Kit usage during the build. This project requires constant on-phone testing, which generates that signal naturally. Do not fake it — but do not neglect it either.

---

## 3. Product definition

### Target user

Anyone in India who receives phone calls. Concretely: a parent, a grandparent, a first-job professional — someone who is not stupid, but is being worked on by a trained operator using urgency, authority and fear.

The user is not a security expert. They will not read a risk score. They need one sentence, at the right moment, that breaks the spell.

### The single loop — this is the entire product

```
1. User receives a suspicious call and puts it on speakerphone.
2. User taps "Listen" in Scam Shield.
3. The app transcribes the conversation live, on-device.
4. Deterministic signal detectors run continuously on the transcript.
5. When signals accumulate, a local SLM classifies the scam type and
   writes one plain sentence.
6. The screen shows a risk level, the sentence, and the evidence with
   timestamps. The phone buzzes.
7. The user hangs up.
```

Seven steps. One call type at a time. If this loop works end to end, we have shipped.

### The headline design decision — no INTERNET permission

**The app's AndroidManifest.xml will not declare `android.permission.INTERNET`.**

This is not a slogan. It is enforced by the operating system: without that permission, the app is physically incapable of opening a socket. Anything it hears cannot leave the device even if we wanted it to.

This is the most valuable single line in the pitch, because it is verifiable on stage. Show the manifest. Say: *"This isn't a privacy policy. It's a permission we didn't ask for. The OS won't let this app talk to the internet."*

Claude must protect this decision. **Do not add the INTERNET permission for any reason.** If a library requires it, replace the library.

### Why haptics and screen, not spoken warnings

The call is on speakerphone. **If the app speaks a warning, the scammer hears it** and adapts — or worse, tells the user the warning is fake.

So the alert channel is **screen + haptics only**. The iQOO 15 has a combined X+Z axis haptic motor capable of directional, textured patterns. We use escalating haptic patterns for risk levels. This is both correct product design and a strong "creative phone use" signal.

Record this reasoning — it is a good answer to a judge question.

---

## 4. The audio capture constraint — read this before writing any code

**Android 10+ blocks third-party apps from capturing `VOICE_CALL` and `VOICE_COMMUNICATION` audio sources.** There is no legitimate way around this on a stock device, and we are not going to try.

**Our design: the user puts the call on speakerphone, and we capture from the ordinary `MediaRecorder.AudioSource.MIC`.**

This is not a workaround dressed as a feature. It is genuinely how people in India already take suspicious calls — on speaker, often with someone else in the room. It captures both sides of the conversation. It requires only `RECORD_AUDIO`.

Say this plainly to judges if asked. Do not claim we intercept calls. We listen to a room in which a call is happening.

A useful side effect worth mentioning but **not building for the MVP**: the same pipeline works for in-person and doorstep scams.

---

## 5. Detection architecture

Two layers. Cheap work runs always; expensive work runs rarely. This mirrors the design principle that makes on-device systems viable, and it is the technical-depth story.

```
            SPEAKERPHONE CALL
                    │
                    ▼
        ┌───────────────────────┐
        │  ALWAYS ON · cheap    │
        │  mic → VAD → streaming ASR
        └───────────┬───────────┘
                    ▼
            ROLLING TRANSCRIPT
              (memory only)
                    │
                    ▼
        ┌───────────────────────┐
        │  SIGNAL DETECTORS     │  deterministic, instant,
        │  pattern + keyword    │  fully explainable
        └───────────┬───────────┘
                    │
              signals fire
                    │
                    ▼
        ┌───────────────────────┐
        │  RISK SCORER          │  weighted accumulation
        │  LOW / ELEVATED / HIGH│  over the call
        └───────────┬───────────┘
                    │
            crosses threshold
                    │
                    ▼
        ┌───────────────────────┐
        │  LOCAL SLM · rare     │  classify scam type,
        │  Gemma 3 1B           │  write one sentence
        └───────────┬───────────┘
                    ▼
        SCREEN  +  HAPTICS  (never audio)

        no INTERNET permission · nothing is written to disk
```

### Layer 1 — deterministic signal detectors

These run on every transcript update. They are instant, explainable, and they carry the demo even if the SLM is slow or misbehaving. **Build these first.**

Each detector produces a `Signal` with a weight, a human-readable label, and the transcript span that triggered it.

| Signal | What it looks for | Weight |
|---|---|---|
| `OTP_REQUEST` | asks for OTP, CVV, PIN, "code you just received" | Very high |
| `REMOTE_ACCESS` | AnyDesk, TeamViewer, QuickSupport, "install this app so I can see your screen" | Very high |
| `DIGITAL_ARREST` | claims to be police / CBI / TRAI / customs, mentions arrest, warrant, FIR | Very high |
| `KYC_EXPIRY` | "KYC expired", "account will be blocked", "update details now" | High |
| `UPI_REFUND` | "wrong transfer", "send it back", "I'll send a request, just approve" | High |
| `PARCEL_CUSTOMS` | courier held, customs duty, illegal item in your parcel | High |
| `ADVANCE_FEE` | processing fee, security deposit, "pay to release" | High |
| `URGENCY_PRESSURE` | "immediately", "within 10 minutes", "don't disconnect", "don't tell anyone" | Medium |
| `AUTHORITY_CLAIM` | claims to be from a bank, government, electricity board | Medium |
| `CHANNEL_SHIFT` | "message me on WhatsApp", "call this other number" | Medium |
| `SECRECY` | "don't discuss this with family", "this is confidential" | Medium |

The combination matters more than any single hit. Authority + urgency + payment is the universal scam shape.

### Layer 2 — the local SLM

Runs only when the risk score crosses `ELEVATED`, and at most once every N seconds. Its job is narrow:

1. Name the scam type in plain language.
2. Write **one** sentence the user can act on.
3. Suggest the single safest next step ("Hang up and call your bank on the number printed on your card.")

It does not compute the risk score. The score comes from the deterministic layer, which means the number on screen is always explainable and never hallucinated.

### Risk levels and what the user sees

| Level | Screen | Haptics |
|---|---|---|
| `LOW` | Quiet. Listening indicator only | None |
| `ELEVATED` | Amber banner + the signals detected so far | Single soft pulse |
| `HIGH` | Red banner + one plain sentence + evidence list + "Hang up" guidance | Escalating triple pulse, repeating |

---

## 6. Safety and honesty policy — non-negotiable

This app makes accusations about a real person on the other end of a line. It must be careful.

- **The app advises. It never acts.** It does not hang up, block, mute, or report. The user decides.
- **It never names or accuses an individual.** It says "this conversation matches a known scam pattern," not "this person is a criminal."
- **Every claim shows its evidence.** Any risk level above LOW must display the specific signals and the transcript spans that triggered them. No unexplained scores.
- **False positives are expected and designed for.** A legitimate bank call may trip `AUTHORITY_CLAIM` and `URGENCY_PRESSURE`. The UI must make dismissing easy and non-alarming. Never imply certainty we do not have.
- **It never asks the user to enter sensitive data** to "verify" anything.
- **Uncertainty is spoken plainly.** If signals are weak, the app says so rather than inflating.

If Claude is ever unsure whether an output is too strong, make it weaker. An under-confident warning is a minor product flaw. An over-confident false accusation is a serious one.

---

## 7. Technical stack — decided, do not re-litigate

| Layer | Choice | Why |
|---|---|---|
| App | Kotlin + Jetpack Compose | Native, fast to build, good accessibility |
| Audio | `AudioRecord` / `MediaRecorder.AudioSource.MIC` | Only legal path. Speakerphone design |
| VAD + ASR | **sherpa-onnx streaming ASR** (Apache-2.0) | Offline, proven on Android, streaming zipformer, prebuilt AAR |
| Signal detection | Hand-written Kotlin | Instant, explainable, zero dependencies |
| SLM | **Gemma 3 1B via MediaPipe LLM Inference API** | A Gradle dependency and a model file. **No NDK, no CMake, no native build.** 529MB int4, ~2,585 tok/s prefill on mobile GPU |
| Haptics | `VibrationEffect` composition | Uses the iQOO 15's X+Z motor |
| Storage | **None.** In-memory only | Privacy is the product |
| Network | **None. No INTERNET permission** | Enforced by the OS |

### The stack decision that saves the project

We are **not** hand-integrating ONNX Runtime, ExecuTorch, ncnn, or Qualcomm QNN. Every one of those means NDK, CMake, ABI splits and native build times — during an event where the laptop is restricted for 55% of the clock.

MediaPipe LLM Inference and a prebuilt sherpa-onnx AAR give us both models as ordinary dependencies. We give up some control over quantization and NPU delegation. We buy back the entire event.

If a judge asks about NPU delegation: answer honestly. We chose a shippable path over an optimal one, and we know the difference. That is an engineering answer, not an excuse.

---

## 8. MVP scope

The first working version must include only:

1. Mic capture running while the screen is on.
2. Streaming ASR producing a live transcript.
3. At least six deterministic signal detectors working.
4. Risk scoring across the call with three levels.
5. Live screen showing risk, the sentence, and evidence with timestamps.
6. Haptic escalation on level change.
7. Gemma 3 1B producing the plain-language warning at ELEVATED and above.
8. Everything working with airplane mode on.
9. No INTERNET permission in the manifest.

That is the MVP. Everything else is secondary.

---

## 9. What we are explicitly NOT building

Claude must refuse to build these during the event, even if asked in the moment. Each was considered and cut deliberately.

| Cut | Why |
|---|---|
| Call recording via telephony APIs | Blocked by Android 10+. Not attempting it |
| Auto-hangup / auto-block | The app advises, never acts. Also a terrible failure mode on a false positive |
| Caller ID / number reputation lookup | Requires network. Breaks the entire pitch |
| Cloud reporting or crowd-sourced scam database | Requires network. Breaks the entire pitch |
| Persistent storage of transcripts or audio | Privacy is the product. Memory only, discarded at call end |
| Speaker diarization (who said what) | Nice, expensive, not needed. Signals work on the transcript regardless of speaker |
| Voice deepfake / synthetic voice detection | Fascinating, and a research project, not a weekend |
| Multi-language and full Hinglish | English only, with a small set of common Hindi keyword variants if time allows. Do not build a translation layer |
| SMS / WhatsApp scanning | A different product. Do not let it creep in |
| Contact list integration | No reason, and a privacy liability |
| Login, accounts, onboarding flows | Nobody is scoring this |
| Analytics of any kind | We have no internet permission. By design |

**If at hour 20 someone proposes any of the above, the answer is no, and the reason is written here.**

---

## 10. Repository structure

Four packages, one Gradle module. Not sixteen modules — module wiring costs build time we cannot spare.

```
scam-shield/
├── CLAUDE.md
├── README.md
├── .gitignore
└── app/src/main/java/.../
    ├── audio/      AudioRecord capture, VAD, sherpa-onnx streaming ASR
    ├── detect/     Signal detectors, Signal model, RiskScorer
    ├── llm/        MediaPipe LLM wrapper, prompt, response parsing
    └── ui/         Compose screen, risk banner, evidence list, haptics
```

---

## 11. Build order — MUST FOLLOW

Each phase ends in a checkpoint commit. **Never let an experimental change destroy the last known-good build.**

### Phase 0 — environment check (before anything else)
Confirm on the loaner: Android version, RAM, mic access, that a trivial APK installs and runs. Confirm the Office Kit bridge works.

### Phase 1 — audio capture
Mic capture running, amplitude visible on screen. Nothing else.

### Phase 2 — streaming ASR
sherpa-onnx producing live text from speech. **Verify with airplane mode on.** This is the highest-risk integration — do it early.

### Phase 3 — signal detectors
Six detectors, unit-testable, running on the live transcript. Evidence spans captured.

### Phase 4 — risk scoring + UI
Three levels, banner, evidence list with timestamps. **At the end of this phase the demo already works without any LLM.** That is deliberate — it is our insurance.

### Phase 5 — haptics
Escalating patterns on level change.

### Phase 6 — Gemma 3 1B
MediaPipe LLM Inference producing the plain sentence. Triggered only at ELEVATED+, rate-limited.

### Phase 7 — hardening
Failure handling: mic denied, ASR silent, model slow, thermal. Graceful degradation to Phase 4 behaviour.

### Phase 8 — demo polish
Rehearsal, backup video, pitch.

**Phases 1–5 are the product. Phase 6 is an upgrade. If Phase 6 fails, we still demo.**

---

## 12. The 30 hours

Team of three. One on audio/ASR, one on detection/scoring/UI, one on testing, demo and pitch — that third person works through the phone-only blocks and is not a luxury.

🟢 Green Light (phone + laptop) · 🔴 Red Light (phone only)

| Block | | Work | Gate |
|---|---|---|---|
| Sat 10:00–11:00 | — | Opening, teach-in, pair Office Kit. **Ask whether Office Kit remote control satisfies Red Light** | |
| 11:00–13:00 | 🟢 | Empty repo → shell, mic permission, `AudioRecord` capture, amplitude UI. Set up CI fallback workflow | **CP1** |
| 13:00–14:00 | 🔴 | Install on phone. Write the scam script library and detector keyword lists on the phone. Record test call audio | |
| 14:00–17:00 | 🟢 | **sherpa-onnx streaming ASR.** Highest-risk integration, done early | **CP2** |
| 17:00–19:00 | 🔴 | Speak test scripts at the phone. Log ASR accuracy. Tune the keyword lists against real transcripts | |
| 19:00–22:00 | 🟢 | **Signal detectors + risk scorer + evidence spans** | **CP3** |
| 22:00–00:00 | 🔴 | Run all scam scripts against the build. Tune weights and thresholds. Evaluation round 1 | |
| 00:00–03:00 | 🟢 | **UI: risk banner, evidence list, timestamps** | **CP4 — demo exists without any LLM** |
| 03:00–06:00 | 🔴 | Sleep in shifts. Haptic pattern design and tuning | |
| 06:00–09:00 | 🟢 | Haptics wired + Gemma 3 1B via MediaPipe + failure handling | **CP5** |
| 09:00–11:00 | 🔴 | Full rehearsal with a real phone call from another phone, in the hall, with real noise. **Record the backup demo video on the phone** | |
| 11:00–13:00 | 🟢 | **Freeze.** Bug fixes only | |
| 13:00–16:00 | 🔴 | Pitch deck built on the phone. Rehearse. Evaluation round 2. Top 10 pitch | |

### Red Light build strategy

- **Primary:** Office Kit remote control — the phone drives Android Studio on the laptop. Confirm this is permitted at the Saturday teach-in.
- **Fallback:** edit on the phone, push from Termux, GitHub Actions builds the APK, download and install. 4–8 minutes per cycle. **Set this workflow up in the first Green block so switching costs nothing.**
- **Never:** Termux + Gradle + Android SDK on-device. Too slow to set up, too many failure modes.

Red Light blocks are for work that needs no compiler: speaking test scripts at an installed build, tuning keyword lists and weights, logging ASR accuracy, designing haptic patterns, rehearsing, and building the deck.

---

## 13. Acceptance tests

Not finished when it launches. Finished when these pass on the loaner, **with airplane mode on**.

| # | Test | Expected |
|---|---|---|
| 1 | Grant mic, start listening, speak | Live transcript appears |
| 2 | Read the OTP scam script aloud | Risk reaches HIGH, `OTP_REQUEST` shown as evidence |
| 3 | Read the digital-arrest script | Correct scam type named, plain sentence produced |
| 4 | Read the remote-access script | `REMOTE_ACCESS` fires, app names the risk |
| 5 | Read a normal, friendly conversation | Stays LOW. **No false alarm** |
| 6 | Read a genuine bank call (verification, no OTP request) | At most ELEVATED, and evidence makes the reason obvious |
| 7 | Any level above LOW | Evidence list is populated with timestamps. No unexplained score |
| 8 | Risk escalates | Haptics fire. **No audio is emitted** |
| 9 | Deny mic permission | Clear, non-crashing explanation |
| 10 | Run 10 minutes continuously | No crash, no runaway memory, no severe throttling |
| 11 | Inspect `AndroidManifest.xml` | **No INTERNET permission present** |
| 12 | Airplane mode, full run | Everything works |
| 13 | Measure and record | ASR latency, time from trigger phrase to on-screen warning, peak RAM |

**Test 13 is mandatory.** Every documented winner had judges quoting a measurement. We want to say: *"From the moment they ask for the OTP to the warning on screen: under X hundred milliseconds. Zero network."*

**Test 5 and 6 are equally mandatory.** A scam detector that flags everything is worthless, and a judge will test exactly this.

---

## 14. Demo — three minutes

1. **Frame it.** "Everyone here has had this call. Somebody claiming to be from your bank, and they need one code." Hold up the phone.
2. **Hand the script to a judge.** They call the phone from their own. It goes on speaker.
3. **They read the scam script.** Mid-sentence, the screen turns amber, then red. The phone buzzes. The screen names the scam and shows the exact words that gave it away, with timestamps.
4. **The proof.** Show airplane mode is on. Then show the manifest. *"This app has no internet permission. It's not that we promise not to upload your calls — the operating system won't let us."*
5. **The number.** "From the word 'OTP' to the warning on screen: under X milliseconds. On the phone. Every time."
6. **The honesty beat.** Read a normal conversation. Nothing happens. *"A detector that flags everything is a detector nobody keeps installed."*

**Fallback:** a demo video recorded on the phone during Sunday's rehearsal block. If live fails, show the recording and say plainly that it is a recording. **Never claim a live run that did not happen.**

---

## 15. Risks

| Risk | Trigger | Response |
|---|---|---|
| Red Light forbids driving the laptop | Answer at the teach-in | Switch to the CI build path. Batch changes, build every 40 min |
| sherpa-onnx integration fails | CP2 slips past 17:00 | Fall back to Android `SpeechRecognizer` **only if it has an offline mode on the device** — verify. If it needs network, the pitch dies, so prefer fixing sherpa-onnx |
| ASR unreliable in hall noise | Rehearsal block | Speak closer to the phone in the demo. Tune VAD. Do not fight it at 4am |
| Gemma 3 1B too slow or won't load | CP5 | **Ship Phase 4 behaviour.** Deterministic detectors already carry the demo. Say the SLM is the next step |
| Too many false positives | Test 5 fails | Raise thresholds. Require two independent signals for HIGH. Never ship a detector that flags friendly conversation |
| Someone else built the same thing | Evaluation round | Our differentiators: the no-INTERNET manifest, the evidence list, the live judge-performed demo. Lead with those |
| Scope creep at hour 20 | Someone gets excited | Re-read section 9 out loud |

---

## 16. Non-goals

Stated so we draw the line before a judge does:

- Not a call blocker, spam filter, or caller-ID service.
- Does not decide who is a criminal. It identifies conversation patterns.
- Does not hang up, mute, or act on the user's behalf.
- Does not store or transmit audio or transcripts.
- Does not replace a bank's fraud team or the police.
- Does not work on a live call feed — the call must be on speakerphone.
- Does not require, and cannot use, a network connection.

---

## 17. Judge questions — prepare answers

**"Why can't this be a cloud service?"** Because nobody will upload their phone calls to be checked for fraud, and the moment you do, you've created the exact honeypot you were protecting them from. This works because it never leaves the phone.

**"How do you handle false positives?"** Every warning shows its evidence, so the user can judge it themselves. HIGH requires two independent signals. And we tested specifically for it — a friendly conversation and a genuine bank call both stay quiet.

**"Why not just use the NPU / QNN directly?"** We chose MediaPipe's LLM Inference API because it ships as a Gradle dependency and we had 30 hours with the laptop restricted for half of them. We traded some quantization control for shipping. We know what we gave up.

**"Isn't call recording illegal / blocked?"** We don't record calls. Android blocks that and we didn't try. The user puts the call on speaker and we listen through the ordinary microphone — which is how people already take these calls.

**"What's actually novel here?"** Real-time, on a phone, offline, with the evidence shown rather than a black-box score — and an app that cannot phone home because it never asked for permission to.

**"What breaks first at scale?"** Language coverage. English only today. Hinglish and regional languages are the real next step, and on-device ASR for Indian languages is the hard part.

---

## 18. Pre-event preparation

**We write no code we intend to carry in.** The rules require original work built during the event window, and we are honouring that. What we do beforehand is buy knowledge on our own Android phones:

1. Get sherpa-onnx streaming ASR running end to end on Android. Time it. Hit the errors now, not at hour 6.
2. Get MediaPipe LLM Inference loading Gemma 3 1B and generating. Measure load time and first-token latency.
3. Feel out `VibrationEffect` composition and pick three escalating patterns.
4. Write the scam script library — 6–8 realistic scripts covering OTP, digital arrest, KYC, UPI refund, remote access, courier, plus two innocent control conversations. **These are content, not code.** They also become the test suite and the demo material.
5. Confirm Office Kit is installed and paired.
6. Buy a Bluetooth keyboard. It is a peripheral, not a second computer, and it turns phone-only development from painful into merely slow. Confirm at check-in that it is permitted.

---

## 19. Development rules for Claude

1. **Do not overbuild.** Section 9 is binding.
2. **Do not add the INTERNET permission.** Ever. Replace the library instead.
3. **Do not add a dependency unless it is necessary and ships as a prebuilt artifact.** No NDK, no CMake.
4. **Build Phase 4 before Phase 6.** The deterministic detector must carry the demo alone.
5. **Do not write anything to disk.** No audio, no transcripts, no logs containing conversation content.
6. **Every risk score must be explainable.** If we cannot show the evidence, we do not show the score.
7. **Prefer an under-confident warning to an over-confident one.**
8. **Commit a checkpoint after every phase.** Never let an experiment destroy a working build.
9. **When something fails, diagnose the exact error before changing architecture.**
10. **The builder is a non-programmer.** Give exact commands, say where to run them, say what success looks like, and say what to send back if it fails. Do not assume programming knowledge, and do not explain theory that does not unblock the build.
11. **Run the relevant acceptance tests after every phase.**
12. **Never fake a demo result or claim a run that did not happen.**

---

## 20. Checkpoints

```
phase-1-audio-capture
phase-2-asr-working
phase-3-detectors-working
phase-4-risk-ui-working      ← demo is viable from here
phase-5-haptics-working
phase-6-llm-working
final-demo
```

---

## 21. North star

Everything optimises toward one sentence:

> **"Your phone caught the scam while they were still talking — and it couldn't have leaked your call if it tried."**

If a feature does not strengthen that sentence, it is not an MVP priority.

---

## 22. Sources and verification notes

Hackathon rules and rubric — https://iqoo.reskilll.com/guide, /cities, /terms
Android on-device stack — Google AI Edge: MediaPipe LLM Inference, AI Edge RAG, AI Edge Function Calling; LiteRT Hugging Face community
Speech — sherpa-onnx (k2-fsa), Apache-2.0

**Re-verify before the event.** Model availability, MediaPipe API surfaces, Android permission behaviour on OriginOS 6, and the hackathon's own rules can all change. In particular, confirm at the venue: Office Kit remote control under Red Light, and whether a Bluetooth keyboard is permitted.

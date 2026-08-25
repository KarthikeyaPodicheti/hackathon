# Scam Shield

**An Android app that tells you a phone call is a scam while the scammer is still talking — and it can't leak your call, because it never asked for permission to use the internet.**

Built for the iQOO Hackathon 2026 · City Battles
Track: **FinTech & Commerce**

---

## The problem

Everyone reading this has had the call.

Somebody claiming to be from your bank. Somebody claiming to be from the police, saying there's a warrant. Somebody saying your KYC has expired and your account will be blocked in ten minutes. Somebody who transferred money to you "by mistake" and just needs you to approve a request. Somebody who needs you to install AnyDesk so they can "help you."

These calls work because they are performed by trained operators using authority, urgency and fear. The victim is not stupid. They are being run through a script that is specifically designed to stop them from thinking clearly — and it works on educated, careful people every single day in India.

The moment that matters is the sixty seconds while the call is still happening. Nobody needs a warning afterwards.

## What I'm building

Scam Shield listens to a call on speakerphone and, in real time, tells you when the conversation matches a known scam pattern.

It shows you a risk level, one plain sentence explaining what's happening, and **the exact words that gave it away, with timestamps**. Your phone buzzes. You hang up.

That's the whole product. One job, done properly.

## Why this has to run on the phone

This is the part I care most about, and it's why I think this project belongs at a phone-first hackathon rather than being a cloud service that happens to have an app.

**Nobody will upload their phone calls to a server to be checked for fraud.** The moment you build that, you have created exactly the honeypot you were claiming to protect people from. A scam detector that runs in the cloud is a scam detector nobody installs.

So the entire system runs on the device. Speech recognition, pattern detection, and the language model all execute on the Snapdragon. Nothing is written to disk. Nothing is transmitted.

And I'm going to prove it rather than promise it: **the app will not declare the `INTERNET` permission in its manifest.** That isn't a privacy policy, it's enforced by Android. Without that permission the app is physically incapable of opening a network connection. I'll show the manifest during the demo, with the phone in airplane mode.

## How it works

```
speakerphone call
        │
        ▼
  on-device streaming speech recognition
        │
        ▼
  rolling transcript  (memory only, never saved)
        │
        ▼
  deterministic signal detectors        ← always on, instant, explainable
  OTP request · remote-access app · fake authority ·
  KYC expiry · UPI refund · urgency · secrecy
        │
        ▼
  risk score  LOW → ELEVATED → HIGH
        │
        ▼
  local small language model            ← triggered only, writes one plain sentence
        │
        ▼
  screen + haptics       (never audio — the scammer is listening)
```

Two things I want to point out about this design:

**Cheap work runs constantly; expensive work almost never runs.** The risk score comes from deterministic detectors, not from the model. That keeps it fast, keeps it explainable, and means the number on screen is never something a model invented. The language model only writes the sentence a human reads.

**The alert is silent.** The call is on speakerphone, so a spoken warning would tip off the person running the scam. The warning goes to the screen and to the phone's haptics instead — which is also the most genuinely useful thing I could do with the iQOO 15's X and Z axis vibration motor.

## What I'll demo

I'd like a judge to do this themselves.

I hand them a scam script. They call the phone from their own phone. It goes on speaker. They start reading — and mid-sentence the screen goes amber, then red, the phone buzzes, and it names the scam and shows them the exact words that triggered it.

Then I show airplane mode is still on, and I show the manifest.

Then I read a completely normal conversation at it and nothing happens — because a detector that flags everything is a detector nobody keeps installed.

## Why I can finish this in 30 hours

I've scoped this deliberately against the format, including the Red Light blocks where the laptop is restricted.

- **Two components, not six.** Speech recognition and classification. That's it.
- **No native builds.** Speech runs on sherpa-onnx as a prebuilt Android library; the language model runs through MediaPipe's LLM Inference API as an ordinary Gradle dependency. No NDK, no CMake — which matters a great deal when the laptop is unavailable for over half the event.
- **The demo works before the language model does.** The deterministic detectors carry the entire demo on their own. The model is an upgrade layered on top, not a dependency. If it doesn't land, I still have a product.
- **A lot of the work needs no compiler.** Speaking test scripts at an installed build, tuning detection thresholds, designing haptic patterns, rehearsing — all of that is real work I can do on the phone during Red Light.

## Stack

Kotlin · Jetpack Compose · sherpa-onnx (Apache-2.0) for offline streaming speech recognition · Gemma 3 1B via MediaPipe LLM Inference · Android `VibrationEffect` for haptics · no storage · **no network permission**

## What this is not

I want to be precise about the limits, because I think overclaiming here would be irresponsible.

- It does not record calls. Android blocks that for third-party apps, and I'm not attempting to get around it. The user puts the call on speaker and the app listens through the ordinary microphone — which is how most people already take a call they're suspicious about.
- It does not hang up, block, or act on anyone's behalf. It advises. The user decides.
- It does not accuse a person of being a criminal. It reports that a conversation matches a known scam pattern, and it shows its evidence so the user can judge for themselves.
- It is English-only at this stage. Hinglish and regional languages are the obvious next step and the genuinely hard part.

## The one sentence

> My phone caught the scam while they were still talking — and it couldn't have leaked the call if it tried.

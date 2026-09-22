# PatchCam

### Snap. Fix. Code On.

**PatchCam turns a phone camera into a debugging bridge for locked-down, offline, or otherwise restricted machines.** Point it at an error on a screen that won't let you copy, paste, or connect to the internet — PatchCam reads it, finds or generates a fix, checks that fix with a real parser, and delivers it back to the machine that couldn't receive it any other way.

· Built for **iQOO Hackathon 2026** — Track: Developer Tools

---

## The Problem

Debugging assumes you can copy an error into a search bar. That assumption breaks down constantly:

* Locked-down college lab PCs and exam machines block copy-paste and USB transfer.
* Client, enterprise, or airgapped laptops often have no internet access.
* Some environments can't run a browser or a chat assistant at all.
* The only way to move an error off the screen is to retype it, character by character.

It hits a CS student mid-practical, a hackathon team on a borrowed machine, and an engineer debugging on a locked client laptop — routinely, not as an edge case.

**What if the camera could be the bridge?**

---

## The Solution

PatchCam photographs the error and the surrounding code, reconstructs it on-device, diagnoses it, proposes a fix, and — critically — **verifies that fix with a real language parser before ever calling it "verified."** The confirmed patch then travels back to the laptop over whichever channel actually works: a clipboard sync, a file transfer, or a live screen mirror.

Everything except the laptop-side bridge itself runs **entirely on the phone.** Nothing is sent to any cloud service.

---

## How It Works

| Step | What happens |
|---|---|
| **1. Capture** | Point the phone at the error. Multi-frame scanning lets you capture long tracebacks section by section; overlapping lines are merged automatically. |
| **2. Understand** | ML Kit OCR reads the text on-device, a line classifier and indentation-rebuild step reconstruct the original source structure. |
| **3. Diagnose** | A local fix table is checked first. For Python, a **real `ast` parser (via Chaquopy)** pinpoints the exact line and column. |
| **4. Fix** | Known errors are patched from the local table. Unmatched errors fall back to an **on-device Gemma model** (via MediaPipe), grounded by local BM25 retrieval over PatchCam's own knowledge base. |
| **5. Verify** | For Python, every candidate patch — rule-based or LLM-proposed — is re-parsed by the same `ast` parser before it's shown as verified. If it fails, the LLM gets one retry with the parser error fed back ("LLM proposes, parser disposes"). |
| **6. Deliver** | The confirmed patch reaches the laptop via **Office Kit**: clipboard sync (fastest), file EasyShare (works with no shared Wi-Fi), or screen mirroring (for live demos). |
| **7. Learn** | The Flow tab lets you mark whether a fix worked — PatchCam's confidence for that fix type adjusts from your feedback, stored locally on the phone. |

### Architecture

```text
 Camera ──► ML Kit OCR ──► Line classifier ──► Indentation rebuild
  (on-device)  (on-device)     (on-device)         (on-device)
                                                        │
                    ┌───────────────────────────────────┘
                    ▼
        Python parser (Chaquopy, on-device) ── exact line + column
                    │
                    ▼
   Fix rules ──► else on-device Gemma  ◄── local BM25 retrieval (RAG)
        │              │      ▲
        │              ▼      │  parser error fed back — one retry
        │      Python parser re-verifies the patch
        └──────────────┬──────┘
                        ▼
             ┌─ Wi-Fi WebSocket bridge ──────┐
             │                                ├─► Laptop bridge ─► applies
             └─ Office Kit (clipboard/file/mirror) ┘  + re-checks with the
                                                        laptop's own Python
```

### The verified / unverified split

PatchCam is honest about what it can prove. **Python fixes are checked with a real parser** and only then labeled "verified." For Java and Kotlin, PatchCam still captures, diagnoses, and proposes a fix through the same pipeline — but without an on-device parser for those languages, any LLM-proposed patch is shown with a clear **⚠ unverified** label rather than presented as confirmed. The Python path is the one demonstrated end-to-end in the current prototype.

---

## Office Kit Integration — three real touchpoints

Office Kit is a system feature of OriginOS 6, not a third-party SDK, so PatchCam uses the same three surfaces a person would use by hand — and counts every real use of each, live, on the Result screen ("Office Kit used *N* time(s) this session").


| Surface | Best for |
|---|---|
| **Clipboard sync** | Fastest path — the verified fix lands on the phone's clipboard and, with Office Kit's sync on, is already on the laptop's clipboard to paste straight into the editor. |
| **File EasyShare** | Full patch + automatic backup, no shared Wi-Fi required — the typical case for locked-down lab machines. |
| **Screen mirroring** | For live demos — judges watch the phone scan and the laptop file change on the same mirrored display. |

### Laptop bridge

```bash
cd laptop_bridge
pip install -r requirements.txt
python patchcam_bridge.py --file main.py            # patch this file automatically
python patchcam_bridge.py --file main.py --window   # also show the code in a big window
```

The laptop prints a 6-digit pairing code (and a QR). The phone discovers the laptop from that code alone; if Wi-Fi broadcast is blocked, enter the laptop's `IP:8765` manually in the app. Every patch is backed up (`.bak-…`) before it's applied, and the bridge re-checks the result with the laptop's own Python before writing it.

---

## What's Real ML, What's Rules

| Part | Kind |
|---|---|
| ML Kit text recognition | ML — on-device OCR |
| Gemma via MediaPipe | ML — on-device LLM (optional model file) |
| BM25 local retrieval (RAG) | Classical IR, on-device |
| Android speech recognizer (voice chat) | ML — on-device/system ASR |
| Python `ast` via Chaquopy | Deterministic parser — the verifier |
| Fix rules, offline chat engine | Rules |
| Office Kit clipboard / file / mirror | System feature — used honestly, counted, never claimed as PatchCam's own AI |

---

## Feedback Loop


Every scan moves through five visible stages — **Observe → Reconstruct → Diagnose → Validate → Update** — and you can inspect or correct what PatchCam read at any stage. Marking a fix as "Worked," "Partly," or "Didn't" is stored on-device and adjusts PatchCam's confidence for that fix type going forward.

---

## Key Features

| Feature | Description |
|---|---|
| Camera-based debugging | Capture an error straight off a locked screen, no copy-paste needed |
| On-device OCR | ML Kit reads the error and code with nothing sent to the cloud |
| Local knowledge base | Known errors are matched and explained instantly |
| On-device LLM fallback | Gemma (via MediaPipe) proposes a fix for errors outside the known set |
| Parser-verified patches | Python fixes are re-checked by a real `ast` parser before being trusted |
| Office Kit delivery | Clipboard sync, file transfer, or screen mirror — whichever fits the machine |
| Feedback loop | Mark whether a fix worked; confidence adjusts locally over time |
| Offline-first | The full capture-to-patch loop runs without a network connection |
| Multi-language capture | Demonstrated on Python, Java, and Kotlin errors (verification is Python-only today) |

---

## Tech Stack

**Mobile** — Android, Kotlin
**AI & Intelligence** — ML Kit OCR, on-device Gemma via MediaPipe LLM Inference, BM25 local retrieval, Chaquopy (Python `ast` parser on Android)
**Development** — Python, Java, Kotlin, Android Studio
**Communication** — Wi-Fi WebSocket bridge, Office Kit (clipboard / file / mirror)

---

## Getting Started

### Prerequisites

* Android Studio + Android SDK
* A compatible Android device or emulator
* A laptop/PC running the development environment you want to patch
* Python 3.8+ on that laptop (for the bridge)

### Clone and open

```bash
git clone <ADD_YOUR_REPO_URL>
cd PatchCam
```

Open the project in Android Studio and let Gradle sync.

### Run the laptop bridge

```bash
cd laptop_bridge
pip install -r requirements.txt
python patchcam_bridge.py --file <your-file>.py
```

Type the pairing code shown on the laptop into the app, then start scanning.

---

## Demo Workflow

```text
1. Display an error on the laptop
2. Open PatchCam on the phone, scan the error (Auto OCR)
3. Review the diagnosis: exact line, cause, proposed patch
4. PatchCam verifies the patch with the Python parser
5. Send it: clipboard sync, file EasyShare, or mirror for the demo
6. Apply, re-check on the laptop, continue coding
7. Mark whether the fix worked — PatchCam remembers for next time
```

---

## Future Scope

* Parser-based verification for Java and Kotlin (currently Python-only)
* Larger, community-extendable local knowledge base
* Automatic build and test verification after a patch is applied
* Crash and stack-trace analysis beyond single tracebacks
* Broader system-log and terminal-error support
* More robust laptop discovery for restrictive network setups
* Enterprise and airgapped-environment hardening

---

## Security & Privacy

PatchCam is built offline-first specifically because the environments it targets are often the ones you can least afford to leak from:

* All OCR, diagnosis, and fix generation happen on-device — no source code or error text is sent to any cloud service.
* Patches are never applied silently: every fix is shown for review before it reaches the laptop.
* The laptop bridge keeps a timestamped backup (`.bak-…`) of any file it modifies.

---

## Team — Funtouch

Built for **iQOO Hackathon 2026** · Track: Developer Tools

| Name | Focus | Links |
|---|---|---|
| **Asritha Satya** | OCR & Capture Pipeline | [LinkedIn](https://linkedin.com/in/asrithasatya12) |
| **Jyotshna** | Office Kit Bridge & Systems Integration | [LinkedIn](https://linkedin.com/in/jyotshnakondepudi) |
| **Pragnya** | On-device AI — Fix Engine, RAG & Parser Verification | [LinkedIn](https://linkedin.com/in/pragnyayelisetti) |

---

## Contributing

Contributions, ideas, and feedback are welcome — open an issue or submit a pull request.



```

---

<p align="center">
  <strong>Less debugging friction. More coding progress.</strong><br/>
  SNAP → FIX → CODE ON
</p>

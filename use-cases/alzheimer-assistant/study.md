# MemoCare (Alzheimer's Memory Assistant) — Study Notes

Personal reference notes from reading the source of this use case (`use-cases/alzheimer-assistant/`). Written so a future session can pick up cold without re-reading every file. Companion to the repo-level [../../study.md](../../study.md) and the sibling [../game-of-throne-demo/study.md](../game-of-throne-demo/study.md).

> **Provenance.** This folder is a verbatim vendor snapshot of [TonyLiangDesign/MemoCare](https://github.com/TonyLiangDesign/MemoCare) at commit `49adebd0fea3b1e1478d8e86a57d47957023d54c`. Nothing here was written for the EverOS repo. To refresh, re-clone upstream and overwrite this folder.

## TL;DR

MemoCare is a **dual-role iOS AR app** (Patient + Caregiver) for Alzheimer's care that **genuinely wires into EverOS** via the `EverMemOSKit` Swift SDK — not just borrowing the taxonomy. Perception (face / room / HomeKit sensors / camera) flows into a local SwiftData cache *and* gets pushed to an EverMemOS backend (cloud by default, local opt-in). The patient's Chat answers questions like "where are my keys?" by hitting EverMemOS `searchMemories` for context, then streaming through DeepSeek; recordings flow back via `memorize()`.

Stack: Swift + SwiftUI, SwiftData (SQLite) for local persistence, ARKit + RealityKit for AR, CoreML (ArcFace) for face embeddings, HomeKit for passive sensors, EverMemOSKit for long-term memory, Google Gemini for object vision, DeepSeek for chat completion, system Speech.framework for STT/TTS.

## Boot — [Memo/MemoApp.swift:1-97](Memo/MemoApp.swift#L1-L97)

`@main` constructs 8 singletons and a shared SwiftData `ModelContainer` over **12 models**:

- Singletons: RoleManager, AuthService, SpeechService, SpeechSynthesisService, APIKeyStore (Keychain-backed credentials), GeminiMedicationService, HomeKitPassiveEventService, DailyMemoryService, DeviceIDManager.
- SwiftData schema ([MemoApp.swift:31-44](Memo/MemoApp.swift#L31-L44)): MemoryEvent, EpisodicMemory, EventLog, Foresight, MedicationPlan, SpatialAnchor, CareContact, RoomProfile, CaregiverRecommendation, MemoryCard, PracticeSession, SensorEvent.
- Root view branches on `RoleManager.currentRole`: `PatientRootView` → `LiveModeView`, `CaregiverTabView` (5 tabs), or `RoleSwitcherView` for first launch.
- **EverMemOS client is lazy** — built per-service via `APIKeyStore.buildAPIClient()` ([Memo/Core/Services/APIKeyStore.swift:60-78](Memo/Core/Services/APIKeyStore.swift#L60-L78)). Not constructed at startup.

## Two roles, one database

| | Patient ([PatientRootView.swift](Memo/Patient/PatientRootView.swift)) | Caregiver ([CaregiverTabView.swift:27-54](Memo/Caregiver/CaregiverTabView.swift#L27-L54)) |
|:--|:--|:--|
| Auth | none | Face ID + 4-digit PIN |
| UI | full-screen AR (`LiveModeView`) with Chat / Record / Find overlays | 5 tabs: Recommendations, Plans, Contacts, DailyMemory, Rooms |
| Job | use memory | curate memory |

Role choice persists in UserDefaults via [Memo/Core/Services/RoleManager.swift:1-53](Memo/Core/Services/RoleManager.swift#L1-L53) and is encoded into `MemoryEvent.sender` so a single SwiftData store serves both. Role switching is a first-launch binary choice in [Memo/Shared/RoleSwitcherView.swift:4-52](Memo/Shared/RoleSwitcherView.swift#L4-L52).

## Data model → EverOS taxonomy mapping

The 12 SwiftData models in [Memo/Core/Models/](Memo/Core/Models/) line up with the 6-type EverOS taxonomy:

| Model | Purpose | EverOS type |
|:--|:--|:--|
| [MemoryEvent.swift:4-55](Memo/Core/Models/MemoryEvent.swift#L4-L55) | Unified event record (chat, action, sensor) with sync/review status. Fields: eventID → message_id, deviceTime → create_time, sender, role, content, groupID. | EventLog wrapper (raw chat/action/sensor) |
| [EventLog.swift:4-34](Memo/Core/Models/EventLog.swift#L4-L34) | Atomic facts extracted from events: atomicFact, timestamp, parentType/parentID. | EventLog (atomic facts) |
| [EpisodicMemory.swift:4-34](Memo/Core/Models/EpisodicMemory.swift#L4-L34) | Narrative summaries derived from events: subject, summary, episode, participants, memcellEventIDList. | Episode (narrative summaries) |
| [Foresight.swift:4-40](Memo/Core/Models/Foresight.swift#L4-L40) | Forward-looking reminders for medication: content, startTime/endTime window, durationDays. | Foresight (prospective reminders) |
| [MemoryCard.swift:6-60](Memo/Core/Models/MemoryCard.swift#L6-L60) | Flashcards for daily practice: question, answer, category (person/item/medication/custom), accuracy tracking. | local-only flashcards |
| [PracticeSession.swift:6-29](Memo/Core/Models/PracticeSession.swift#L6-L29) | One day's practice record: cardCount, correctCount, resultOutcomes parallel array. | local session logging |
| [SpatialAnchor.swift:5-40](Memo/Core/Models/SpatialAnchor.swift#L5-L40) | 3D item positions: itemName, emoji, position (posX/Y/Z), rotation quaternion, roomID. | Semantic (item location) |
| [CareContact.swift:6-80](Memo/Core/Models/CareContact.swift#L6-L80) | Contact directory: relation, realName, phoneNumber, aliases, faceEnrolled, faceSampleCount. | Semantic (person facts) |
| [MedicationPlan.swift:6-39](Memo/Core/Models/MedicationPlan.swift#L6-L39) | Medication schedule: medicationName, scheduledTime, repeatDaily, isConfirmed, confirmedAt. | Foresight (schedule) |
| [RoomProfile.swift](Memo/Core/Models/RoomProfile.swift) | Room metadata: roomID, displayName, emoji, AR world-map data, HomeKit room binding. | Semantic (place facts) |
| [CaregiverRecommendation.swift:4-75](Memo/Core/Models/CaregiverRecommendation.swift#L4-L75) | AI-generated daily suggestions: type (repeatedQuestion, missedRoutine, emotionalDistress), priority, evidence IDs, status. | caregiver-facing AI output |
| [SensorEvent.swift](Memo/Core/Models/SensorEvent.swift) | HomeKit sensor events (motion, outlet): sensorType, roomName, eventType, uploadStatus. | EventLog (passive observation) |

**Key alignment**: `MemoryEvent` encodes fields that map directly to EverMemOS's `MemorizeRequest` schema ([ChatViewModel.swift:761-770](Memo/Patient/Chat/ChatViewModel.swift#L761-L770)); `searchMemories` returns `SearchResponse` with `profiles` (semantic facts) and `memories` (episodic + eventlog).

## The EverOS hook (the key answer)

It's real, not cosmetic. Direct calls visible in code:

- [Memo/Patient/Chat/ChatViewModel.swift:468](Memo/Patient/Chat/ChatViewModel.swift#L468) — `client.searchMemories(builder)` for chat context retrieval.
- [Memo/Patient/Chat/ChatViewModel.swift:773](Memo/Patient/Chat/ChatViewModel.swift#L773) — `client.memorize(req)` writes chat back.
- [Memo/Core/Services/HomeKitPassiveEventService.swift:840](Memo/Core/Services/HomeKitPassiveEventService.swift#L840) — motion events pushed to EverMemOS.
- [Memo/Patient/Record/RecordFeature.swift:103](Memo/Patient/Record/RecordFeature.swift#L103) — recorded items pushed to EverMemOS.

### Deployment modes — [APIKeyStore.swift:51-78](Memo/Core/Services/APIKeyStore.swift#L51-L78)

`DeploymentProfile` enum supports `.cloud` (default) and `.local` (localhost / LAN). `CloudKey` and `LocalKey` handle separate base URLs. `buildAPIClient()` constructs `EverMemOSClient` with `Configuration(baseURL, auth, apiVersion, statusPathSegment)`:

- Cloud: `BearerTokenAuth` with API key.
- Local: `NoAuth`, points at `localhost` or `192.168.x.x`.

### Device isolation trick

`DeviceIDHelper` augments `userID` and `groupID` with device UUID ([ChatViewModel.swift:759-760](Memo/Patient/Chat/ChatViewModel.swift#L759-L760), [RecordFeature.swift:89-90](Memo/Patient/Record/RecordFeature.swift#L89-L90)). Multiple patients or caregivers can run on the same physical iPhone while their memories stay isolated server-side.

### Graceful degradation

If `client` is nil (no credentials configured), services no-op rather than crash ([ChatViewModel.swift:515](Memo/Patient/Chat/ChatViewModel.swift#L515), [HomeKitPassiveEventService.swift:807-808](Memo/Core/Services/HomeKitPassiveEventService.swift#L807-L808)). Gemini and DeepSeek keys are independent — EverMemOS is optional but core to memory recall.

## Services layer — [Memo/Core/Services/](Memo/Core/Services/)

All 16 services, grouped:

**Perception** (AR / vision / passive sensors):
- [FaceRecognitionService.swift](Memo/Core/Services/FaceRecognitionService.swift) — On-device face embedding (ArcFace CoreML), contact matching via cosine similarity, cross-frame tracking.
- [FrameProvider.swift](Memo/Core/Services/FrameProvider.swift) — Captures ARKit camera frames; feeds PerceptionOrchestrator.
- [ARSessionDelegateMultiplexer.swift](Memo/Core/Services/ARSessionDelegateMultiplexer.swift) — Multiplexes multiple AR session observers.
- [PerceptionOrchestrator.swift](Memo/Core/Services/PerceptionOrchestrator.swift) — Dispatches frames to registered consumers at configurable rates; detects AR session disruption/recovery.
- [PerceptionStateStore.swift](Memo/Core/Services/PerceptionStateStore.swift) — Centralized observable state: currentRoomName, visibleFaces dictionary, metrics.
- [HomeKitPassiveEventService.swift](Memo/Core/Services/HomeKitPassiveEventService.swift) — Listens to HomeKit accessories (motion, contact, outlet/power) via HMHomeManager. Uploads motion & outlet events to EverMemOS.
- [SpeechService.swift](Memo/Core/Services/SpeechService.swift) — On-device speech-to-text (Speech.framework).
- [SpeechSynthesisService.swift](Memo/Core/Services/SpeechSynthesisService.swift) — Text-to-speech (AVSpeechSynthesizer).

**Domain** (business logic):
- [DailyMemoryService.swift](Memo/Core/Services/DailyMemoryService.swift) — Manages flashcard pool: auto-generates cards from CareContact / SpatialAnchor / MedicationPlan, deduplicates, cleans orphans.
- [GeminiMedicationService.swift](Memo/Core/Services/GeminiMedicationService.swift) — Calls Google Gemini API to recognize items from camera frames (returns item name + emoji + description).
- [AuthService.swift](Memo/Core/Services/AuthService.swift) — Caregiver auth: Face ID + 4-digit PIN.
- [RoleManager.swift](Memo/Core/Services/RoleManager.swift) — Single source of truth for current role.
- [FaceDataStore.swift](Memo/Core/Services/FaceDataStore.swift) — Filesystem-backed embedding storage (per-contact binary embeddings) for on-device face matching.
- [FaceEmbeddingService.swift](Memo/Core/Services/FaceEmbeddingService.swift) — Runs ArcFace CoreML model on face crops → 512D embeddings.

**Platform** (infrastructure):
- [APIKeyStore.swift](Memo/Core/Services/APIKeyStore.swift) — Keychain-backed credential storage. `buildAPIClient()` factory.
- [DeviceIDManager.swift](Memo/Core/Services/DeviceIDManager.swift) — Generates and caches device UUID.
- [DeviceIDHelper.swift](Memo/Core/Services/DeviceIDHelper.swift) — Utility for augmenting IDs with device UUID.

## Patient flows — [Memo/Patient/](Memo/Patient/)

- **Chat** ([Memo/Patient/Chat/](Memo/Patient/Chat/)): Patient holds button to speak; `ChatViewModel` fetches memory context from EverMemOS via `searchMemories` (hybrid BM25+vector), then sends augmented prompt + voice input to DeepSeek for streaming response. Chat messages are also memorized back to EverMemOS. Face recognition annotates visible people in-frame. AI tool calling enables `search_memory` and `who_is_visible`.
- **DailyPractice** ([Memo/Patient/DailyPractice/](Memo/Patient/DailyPractice/)): Flashcard spaced repetition on enabled cards. User taps to reveal answer; app logs correct/incorrect/skipped and updates `MemoryCard` counters. Results recorded in `PracticeSession`.
- **FindItem** ([Memo/Patient/FindItem/](Memo/Patient/FindItem/)): Patient selects a recorded `SpatialAnchor` from a room-organized list; AR app loads the world map and shows distance + direction guidance toward the anchor's saved position.
- **LiveMode** ([Memo/Patient/LiveMode/](Memo/Patient/LiveMode/)): Unified full-screen AR view defaulting to Chat overlay. "..." menu toggles Record and Find. Room emoji badge top-left shows detected current room.
- **Record** ([Memo/Patient/Record/](Memo/Patient/Record/)): Patient aims camera at an object, taps Record → app captures AR frame → sends image to Gemini → gets item name + emoji → places AR anchor → saves world map → uploads memory to EverMemOS.
- **SplitMode** ([Memo/Patient/SplitMode/](Memo/Patient/SplitMode/)): Alternative UI with full-screen AR on top, feature cards (Chat / Record / Find / Practice) as swappable bottom panels.

## Caregiver flows — [Memo/Caregiver/](Memo/Caregiver/)

- **Auth** ([Memo/Caregiver/Auth/](Memo/Caregiver/Auth/)): Face ID + 4-digit PIN unlocks `CaregiverTabView`.
- **Contacts** ([Memo/Caregiver/Contacts/](Memo/Caregiver/Contacts/)): Add/edit `CareContact` entries (name, relationship, aliases, phone). Tap "Enroll Face" → camera capture loop (5-10 images) → on-device ArcFace embedding generation → `FaceDataStore` persists embeddings.
- **DailyMemory** ([Memo/Caregiver/DailyMemory/](Memo/Caregiver/DailyMemory/)): `DailyMemoryConfigView` lists auto-generated cards + custom cards. Toggle enable/disable.
- **Plans** ([Memo/Caregiver/Plans/](Memo/Caregiver/Plans/)): Create `MedicationPlan` entries (name, time, repeat daily). `Foresight` records generated for reminders.
- **Recommendations** ([Memo/Caregiver/Recommendations/](Memo/Caregiver/Recommendations/)): Lists AI-generated `CaregiverRecommendation` entries (priority, evidence IDs, suggested action). Accept / dismiss / snooze.
- **Review** ([Memo/Caregiver/Review/](Memo/Caregiver/Review/)): Caregiver reviews `MemoryEvent` records, approves / corrects / deletes.
- **Rooms** ([Memo/Caregiver/Rooms/](Memo/Caregiver/Rooms/)): AR room-scanning flow. Walk through a room → ARKit maps geometry → save world map. Bind HomeKit room (Eve Motion, Eve Energy) to detect which room the patient is in.

## DailyPractice end-to-end loop

1. **Caregiver curates** ([DailyMemoryConfigView.swift](Memo/Caregiver/DailyMemory/DailyMemoryConfigView.swift)) — enrolls contacts → `CareContact` entries. Records items → `SpatialAnchor` entries. Creates medication plan → `MedicationPlan` entries.
2. **`DailyMemoryService` auto-generates `MemoryCard`s** ([DailyMemoryService.swift:14-80](Memo/Core/Services/DailyMemoryService.swift#L14-L80)) from each CareContact / SpatialAnchor / MedicationPlan ("Your daughter's name?" → "Annie").
3. **Patient practices** ([DailyPracticeView](Memo/Patient/DailyPractice/DailyPracticeView.swift)) — flips cards, marks correct / incorrect; results saved in `PracticeSession`.
4. **Caregiver reviews trends** in `PracticeHistoryView`.

## Third-party hooks

- **EverMemOSKit** — Swift SDK wrapping EverMemOS HTTP API for `memorize()` and `searchMemories()`. Used in [APIKeyStore.swift:3](Memo/Core/Services/APIKeyStore.swift#L3), [ChatViewModel.swift:3](Memo/Patient/Chat/ChatViewModel.swift#L3), [HomeKitPassiveEventService.swift:6](Memo/Core/Services/HomeKitPassiveEventService.swift#L6), [RecordFeature.swift:4](Memo/Patient/Record/RecordFeature.swift#L4).
- **DeepSeek** ([ChatViewModel.swift:379+](Memo/Patient/Chat/ChatViewModel.swift#L379)) — LLM API for streaming chat responses; receives memory context + system prompt + voice transcription.
- **Google Gemini** ([GeminiMedicationService.swift](Memo/Core/Services/GeminiMedicationService.swift)) — Vision API for item recognition from camera frames during Record.
- **HomeKit** ([HomeKitPassiveEventService.swift:1-930](Memo/Core/Services/HomeKitPassiveEventService.swift#L1-L930)) — Reads motion sensors (Eve Motion) and outlet power metering (Eve Energy) via `HMHomeManager`. Parses proprietary Eve characteristic UUIDs.
- **ARKit / RealityKit** — World mapping, spatial anchoring, environment lighting (used in FindItemARContainer, RecordFeature, LiveModeView).
- **CoreML / Vision** ([FaceEmbeddingService.swift](Memo/Core/Services/FaceEmbeddingService.swift)) — ArcFace model (`Memo/ArcFaceW600K.mlpackage/`) for on-device face embedding generation. The 87 MB weights live in upstream Git LFS; only the ~133 B pointer is in this snapshot.
- **Speech.framework / AVSpeechSynthesizer** — On-device STT / TTS.

## Surprises worth noting

- **Not purely local.** Upstream README claims EverMemOS runs locally and the code supports it (`DeploymentProfile.local`), but the app **actively sends data to cloud by default** when configured. Architecture is genuinely hybrid — not a proof-of-concept that only mocks EverOS.
- **Device isolation via ID augmentation.** `DeviceIDHelper` augments `userID` and `groupID` with device UUID so multiple patients or caregivers can share one physical iPhone with isolated server-side memory. Clever for multi-device households.
- **HomeKit sensor loop closes to EverMemOS.** `HomeKitPassiveEventService` doesn't just log events locally — it directly calls `client.memorize()` to push room entries and appliance usage. Feeds the AI context window for Chat.
- **Chat streams, memory search blocks.** `ChatViewModel` streams DeepSeek responses character-by-character (phrase buffering for TTS), but `searchMemories` is a blocking call that fetches top-K memories before the system prompt is built. No streaming retrieval pipeline.
- **No `CaregiverRecommendation` generator in this codebase.** The model exists and is displayed, but the backend logic that generates them is **not visible here** — likely a cloud-side or async task not vendored in this snapshot.
- **Face embeddings live as raw files**, not in SwiftData (`FaceDataStore`). Allows fast on-demand loading without SQLite overhead.
- **MedicationPlan → Foresight linkage is implicit.** `MedicationPlan.swift` doesn't mention `Foresight` directly. `DailyMemoryService` infers Foresight creation from MedicationPlan but the explicit linkage code is not visible — likely in a helper or migration.
- **EverMemOSKit is an external Swift package** — not vendored in this snapshot, so the actual `.memorize()` / `.searchMemories()` request/response shapes have to be inferred from call sites, not from SDK source.
- **iOS app icons + CoreML model are committed.** Necessary for the app to build/render — strict reading of the parent repo's `use-cases/README.md` "no images" rule would strip them; pragmatic reading keeps them. See the vendoring commit for the policy used here.

## Key files (quick index)

| File | What |
|:--|:--|
| [Memo/MemoApp.swift](Memo/MemoApp.swift) | `@main`, service wiring, SwiftData schema |
| [Memo/Core/Services/APIKeyStore.swift](Memo/Core/Services/APIKeyStore.swift) | Keychain credentials + `buildAPIClient()` factory (cloud vs local) |
| [Memo/Core/Services/RoleManager.swift](Memo/Core/Services/RoleManager.swift) | Patient/Caregiver role state |
| [Memo/Core/Services/DailyMemoryService.swift](Memo/Core/Services/DailyMemoryService.swift) | Auto-generates MemoryCard pool from contacts/items/medication |
| [Memo/Core/Services/HomeKitPassiveEventService.swift](Memo/Core/Services/HomeKitPassiveEventService.swift) | HomeKit listeners → EverMemOS memorize calls |
| [Memo/Core/Services/PerceptionOrchestrator.swift](Memo/Core/Services/PerceptionOrchestrator.swift) | Dispatches AR frames to face/room/speech consumers |
| [Memo/Patient/Chat/ChatViewModel.swift](Memo/Patient/Chat/ChatViewModel.swift) | The main user-facing memory loop: search → DeepSeek stream → memorize |
| [Memo/Patient/Record/RecordFeature.swift](Memo/Patient/Record/RecordFeature.swift) | Item-recording flow: Gemini vision → SpatialAnchor + EverMemOS memorize |
| [Memo/Patient/LiveMode/LiveModeView.swift](Memo/Patient/LiveMode/LiveModeView.swift) | Unified AR shell for the patient |
| [Memo/Caregiver/CaregiverTabView.swift](Memo/Caregiver/CaregiverTabView.swift) | 5-tab caregiver root |
| [Memo/Caregiver/Contacts/FaceCaptureView.swift](Memo/Caregiver/Contacts/FaceCaptureView.swift) | Face enrollment capture loop |
| [Memo/Core/Models/](Memo/Core/Models/) | 12 SwiftData models (see taxonomy table above) |
| [Memo/ArcFaceW600K.mlpackage/](Memo/ArcFaceW600K.mlpackage/) | CoreML face-embedding model (87 MB weights in upstream LFS) |
| [README.md](README.md) | Upstream MemoCare README (marketing + setup) |

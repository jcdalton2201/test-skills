# Business Requirements Document (BRD)
**ID:** BRD-001-mobile-grapheneos  
**Version:** 1.0  
**Owner:** Product Owner / Privacy Officer  
**Date:** 2025-11-19

## 1. Overview
A fully private, Google-free translation web application optimized for GrapheneOS on Pixel 9 Pro. Supports near-real-time translation for text input, audio input, and camera-captured images. Uses LibreTranslate as the preferred server-side translator, EasyNMT as an offline/local fallback, PaddleOCR for OCR, and an open-source STT solution for speech-to-text. The system is lightweight, privacy-preserving, and minimizes external dependencies and data exfiltration by default.

## 2. Goals & Success Metrics
- Provide fast, near-real-time translation for text, audio, and camera-captured images while preserving user privacy.
- Ensure the application is fully Google-free and compatible with GrapheneOS and Pixel 9 Pro.
- Prefer server-side translation via LibreTranslate when available and provide EasyNMT offline fallback.
- Minimize external dependencies and default to local processing where feasible.
- Deliver a lightweight, efficient app with acceptable battery and latency characteristics on Pixel 9 Pro.

## 3. Out of Scope
- Support for iOS or non-GrapheneOS Android platforms (except best-effort compatibility) is out of scope for initial release.
- Distribution via Google Play Store or any Google-owned app distribution channels.
- Integration with proprietary cloud STT services (other than the selected open-source STT) or proprietary translation APIs (other than LibreTranslate when configured).
- Enterprise device management features, single-sign-on, or multi-device sync for translations.
- Automatic on-device model updates by default without explicit user consent (automatic update mechanism design is a gap and thus out-of-scope until specified).
- Paid translation services or monetization features (billing, subscriptions) are out-of-scope for initial MVP.

## 4. Assumptions & Risks
**Assumptions**

**Risks**

## 5. Business Requirements
### BR-001 — The app must not use any Google services, Google Play Services, Google APIs, or send data to Google-owned domains.
**Rationale:** User requirement to be fully Google-free and preserve privacy on GrapheneOS.

**Acceptance Criteria**
- Static analysis of the compiled app binary reveals no references to Google Play Services, com.google.* packages, Google APIs, or Google telemetry SDKs.
- Dynamic network inspection during typical app usage shows no outbound connections to Google-owned domains (as defined by public domain ownership lists).
- Installation and runtime on GrapheneOS do not require or trigger installation of Google Play Services or request Google-specific permissions.

### BR-002 — The app must operate correctly on GrapheneOS and be optimized for Pixel 9 Pro hardware.
**Rationale:** Primary supported OS/hardware is GrapheneOS on Pixel 9 Pro; optimizations improve performance and UX.

**Acceptance Criteria**
- On Pixel 9 Pro running GrapheneOS, the app installs and launches without platform-specific runtime errors or crashes.
- UI components render correctly and are responsive on Pixel 9 Pro; hardware acceleration is leveraged where applicable.
- A set of automated and manual performance smoke tests (provided with the release) run to completion on Pixel 9 Pro without crashes.

### BR-003 — Provide near-real-time translation for plain text input with automatic language detection.
**Rationale:** Text translation is core functionality and must be fast to meet user expectations.

**Acceptance Criteria**
- Given a user submits plain text, the app detects the source language and returns translated text to the selected target language within the configured latency SLA.
- For short or ambiguous text, the system presents detected source language along with a confidence score.
- Language detection accuracy is demonstrated on a representative test set and results are logged for QA (thresholds to be defined prior to release).

### BR-004 — Provide near-real-time translation for audio input using an open-source speech-to-text solution.
**Rationale:** Users must be able to translate spoken language in near-real-time without relying on proprietary cloud STT services.

**Acceptance Criteria**
- Given recorded or live audio input, STT processing returns transcribed text which is forwarded to the translation pipeline and translated into the selected target language.
- The app surfaces confidence scores or an intelligibility indicator for transcriptions; for low confidence the app notifies the user and offers re-record or manual correction.
- Latency for STT+translation meets the configured real-time SLA for supported scenarios on Pixel 9 Pro (SLA to be set before verification).

### BR-005 — Provide camera-based image capture translation using PaddleOCR for optical character recognition.
**Rationale:** Allow users to translate text in images while keeping OCR processing open-source and local where possible.

**Acceptance Criteria**
- Given a captured image with readable text, PaddleOCR extracts the text and the extracted text is translated to the selected target language.
- Preprocessing (deskew, rotation correction) is applied for rotated/skewed text; where OCR fails, the app reports a clear failure reason and guidance to the user.
- OCR performance and extraction success are measured against a representative image corpus; results are recorded for QA and documented.

### BR-006 — Automatic source-language detection must be performed for all input modalities (text, audio, OCR) before translation.
**Rationale:** Automatic language detection simplifies user flow and ensures correct translation pipeline selection.

**Acceptance Criteria**
- For text, audio, and OCR inputs, a detected source language and confidence score are presented to the user prior to translation.
- If detection confidence falls below the configured threshold, the app prompts the user to confirm or select the source language before proceeding.

### BR-007 — Users must be able to select a target language for translation via the UI, and the app must apply that selection across modalities.
**Rationale:** Users require control over the target language to satisfy translation needs.

**Acceptance Criteria**
- When a user selects a target language in settings or per-translation instance, subsequent translations produce output in the selected target language.
- Changing the target language updates subsequent translation operations immediately without requiring app restart.

### BR-008 — When network access to the LibreTranslate server is available, the app must use LibreTranslate for server-side translation by default.
**Rationale:** LibreTranslate is the preferred server-side translator for quality and maintainability.

**Acceptance Criteria**
- Given network connectivity and a configured LibreTranslate endpoint, translation requests are sent to the LibreTranslate server and returned results are displayed to the user.
- If the LibreTranslate server responds with an HTTP error, the app surfaces a clear error message and adheres to a retry policy (retry policy to be specified before release).

### BR-009 — When offline or when the user requests local processing, the app must fall back to EasyNMT for offline translation.
**Rationale:** Provide privacy-preserving offline capability and continuity when network is unavailable.

**Acceptance Criteria**
- When device is offline or user enables offline mode, translation requests are processed locally using the EasyNMT model and results are returned to the user.
- If EasyNMT model files are missing, the app prompts the user to download required models and provides model size estimates and storage consent prior to download.

### BR-010 — All communication with servers (e.g., LibreTranslate endpoints) must be encrypted in transit.
**Rationale:** Protect user data during network transmission (privacy requirement).

**Acceptance Criteria**
- All configured server endpoints are accessed via TLS (HTTPS) and server certificates are validated by the app.
- If a server presents an invalid or untrusted certificate, the app refuses the connection by default and only allows an explicit user-approved exception with documented risk notice.

### BR-011 — Default app behavior must minimize external data transfer and must not log or transmit user content to third parties without explicit user consent.
**Rationale:** Preserve user privacy and adhere to the Google-free, privacy-first requirement.

**Acceptance Criteria**
- With default settings, no user content (text, audio, images) is transmitted to analytics or telemetry services.
- If telemetry or analytics is offered as an opt-in, the app prompts the user with clear consent messaging describing exact data shared and provides an easy opt-out.

### BR-012 — The app and its bundled components (EasyNMT models, PaddleOCR weights, STT models) must be lightweight and manageable on-device, with clear storage/use controls for the user.
**Rationale:** Mobile devices have limited storage; offering control prevents undue resource consumption.

**Acceptance Criteria**
- Before any offline model download, the app displays expected download size, required storage, and obtains explicit user confirmation.
- A storage settings screen lists installed models with sizes and provides the ability to delete individual models to reclaim space.
- Model installation and removal operations complete without leaving corrupted or orphaned files and are recoverable by re-download.


## 6. Compliance / Regulatory Considerations

---

**Gaps Identified**

**Extraction Confidence:** 
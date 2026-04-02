Project Context & Memory Handover: German Language Learning App Ecosystem

1. Project Overview

The user is building a multi-level German language learning ecosystem hosted on GitHub Pages. The project relies on vanilla web technologies (HTML, CSS, JavaScript) to maintain a lightweight footprint, utilizing a "Single Page Application" (SPA) feel per level, connected via a central portal.

Progress is synced across devices using Firebase Firestore and Firebase Authentication (Google Sign-In).

Current Architecture

index.html: The main portal/hub linking to specific level applications.

A1.html: "Menschen A1" - 24 Units. Uses appId: 'german-a1-app'.

B2.html: "Aspekte B2" - 68 Modules grouped by Chapters. Uses appId: 'german-b2-app'.

A2.html & B1.html: Planned for future development.

2. Core Features Implemented

Two Modes: Glossary View (with interactive column hiding for active recall) and Flashcard View (with 3D flip animations and spaced-repetition categorization).

Gamification: Progress bars, session counters, and a "Trophy Shelf" achievement system with 34 trophies across 4 tiers.

Offline Fallback: If Firebase is unreachable or the user opts out of login, progress defaults to localStorage.

Text-to-Speech (TTS): Utilizes native browser speechSynthesis.

Dark Mode: State saved to Firebase alongside user progress.

3. Database & Sync Strategy (Firebase)

To minimize read/write costs and maximize speed, vocabulary data is hardcoded into the HTML files (currently minimized to 1 item per unit for token-saving during testing; user injects full data via LLM prompts offline).

Only the user's progress is saved to Firestore.

Firestore Schema

Path: /artifacts/{appId}/users/{userId}/progress/main

Document Structure:

{
  "known": [0, 4, 12, 45], 
  "trophies": ["first_steps", "night_owl", "dark_mode_rizz"],
  "sessionCount": 15,
  "darkMode": true,
  "lastUpdated": "2026-04-02T19:00:00.000Z",
  "ttsCount": 42,
  "columnHideCount": 8,
  "darkModeToggleCount": 3,
  "studyDates": ["2026-04-01", "2026-04-02"],
  "totalStudyTimeMs": 5400000,
  "flashcardErrors": {"12": 3, "45": 7},
  "sessionsCompleted": 5,
  "lastStudyDate": "2026-04-02"
}


Security Rules: Production-ready. Matches wildcard {appId} to support infinite future levels without updating rules.

match /artifacts/{appId}/users/{userId}/{document=**} {
  allow read, write: if request.auth != null && request.auth.uid == userId;
}


4. Trophy System Architecture

The application uses a 34-trophy gamified achievement system across both A1 and B2, grouped into 4 tiers:

Tier 1 - Progress & Mastery (15 trophies): Word count milestones (10/50/100/500), percentage milestones (25%/50%/75%), type-specific (verbs/nouns/expressions), unit completion, level conquest (A1 Conqueror / B2 Boss), mode explorer, TTS titan.

Tier 2 - Gen Z / Meme (10 trophies): "Bro Actually Studied", "Skibidi Sprecher", "Ohio Behavior", "Rizzed Up Dark Mode", "NPC Arc", "Touch Grass", "Academic Weapon", "Brain Rot Activated", "I Am So Cooked", "On Fire".

Tier 3 - Consistency & Streaks (4 trophies): 3-day/7-day/30-day streaks, session stacker.

Tier 4 - Secret / Hidden (7 trophies): Night Owl, Early Bird, Weekend Warrior, Google Scholar, Chaotic Neutral, "We're So Back", Portal Walker. Secret trophies display as "???" with 🔒 icon until earned.

Cross-Level Trophies: Most trophies are shared (same IDs/conditions across levels). Level-specific: "A1 Conqueror" (A1 only), "B2 Boss" (B2 only) with adapted verb/noun/expression thresholds. "Portal Walker" reads from the OTHER level's Firestore doc via getDoc to detect multi-level progress.

Key Tracking Fields (all backward-compatible via merge:true):
- ttsCount: incremented in speak()
- columnHideCount: incremented in hideTableColumn()
- darkModeToggleCount: incremented in toggleDarkMode()
- studyDates[]: date strings pushed in recordStudyDate()
- totalStudyTimeMs: accumulated via beforeunload handler
- flashcardErrors{}: wordId -> failCount map in markCard()
- sessionsCompleted: incremented when flashcard deck is fully completed
- lastStudyDate: updated in recordStudyDate()

5. Key Challenges & Resolutions (Historical Context)

A. Text-to-Speech (TTS) Inaccuracies

Problem: The browser TTS read plural markers (e.g., , -n, -¨) and numeric testing artifacts, and frequently defaulted to an English accent.

Solution: Implemented a regex-based cleanTextForAudio() function to strip grammatical markers before sending to the TTS engine. Forced language parameters heavily.

User's Final Directive on Voices: For upcoming levels (especially B1), the AI must strictly enforce the German accent using a multi-tier fallback: voices.find(v => v.lang === 'de-DE' || v.lang === 'de_DE') || voices.find(v => v.lang.startsWith('de')).

B. Firebase Authentication & Environment Restrictions

Problem: Initially used Anonymous Auth, which failed to sync progress across different physical devices. Switched to Google Sign-In (signInWithPopup).

Problem 2: Google Auth failed silently when the user opened the app via local file system (file:// protocol).

Solution: Added execution guards warning the user to use a local web server (e.g., Live Server).

Problem 3: Google Auth popup closed instantly on GitHub Pages deployment.

Solution: Added the GitHub Pages domain (mohamedazzam4.github.io) to Firebase Authentication's "Authorized Domains".

C. Data Parsing Anomalies (Transitioning A1 -> B2)

Problem: The B2 data format supplied by the user differed from A1. The Type column was empty (1||word...), breaking array destructuring. Furthermore, example context (e.g., (Mist bauen)) was embedded directly into the German word string, breaking the UI layout and confusing the TTS engine.

Solution: 1. Built a dynamic fallback parser (assigns "Vocab" if the type field is blank).
2. Implemented a smart regex /\s*\(([^)]+\s+[^)]+)\)$/ to separate context phrases from the main word string. The context is now rendered in smaller text beneath the main word, and omitted from the TTS payload.

Problem: B2 required 68 modules (Anki-style numbering), overwhelming the sidebar UI.

Solution: Dynamically parsed the module titles to extract "K1", "K2", etc., and generated sticky chapter headers in the sidebar to group the 68 modules cleanly.

6. Future Roadmap & Instructions for Continuing AI

A2.html and B1.html: When instructed to build these levels, replicate the UI and Firebase architecture of B2.html. Ensure you change the appId in the state object (e.g., appId: 'german-a2-app') to prevent cross-contamination of progress data. Include the full trophy system with all 4 tiers.

Data Ingestion: Always expect the user to provide compact string data. Parse it safely.

Voice Forcing: Remember the user's specific request to strictly enforce the de-DE voice profile in the TTS speak() method for all future builds.

Token Economy: Do not generate massive 1000-word arrays in code responses. Generate 1-2 items per unit for structural testing, and allow the user to inject their own generated data offline.

Architecture Note: A1.html and B2.html share ~95% identical JavaScript logic (1600+ lines). The major tech debt is code duplication. Future refactoring should extract shared logic into shared-app.js and shared CSS into shared-styles.css, with only level-specific config (appId, unit count, color theme, parsing) kept in each HTML file.
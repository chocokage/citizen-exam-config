# CDN Content Update Guide

> Complete guide for updating app content via the CDN repo (`chocokage/citizen-exam-config`) without requiring an app store release.

---

## Table of Contents

1. [Overview](#overview)
2. [CDN Repo Setup](#cdn-repo-setup)
3. [How It Works](#how-it-works)
4. [Updating the Manifest](#updating-the-manifest)
5. [Asset Update Guides](#asset-update-guides)
   - [Senators](#senators)
   - [Representatives](#representatives)
   - [Federal Officials](#federal-officials)
   - [Governors](#governors)
   - [State Capitals](#state-capitals)
   - [Questions (New Exam)](#questions-new-exam)
   - [Questions (Original Exam)](#questions-original-exam)
   - [Explanations](#explanations)
   - [ELI5 Simple Explanations](#eli5-simple-explanations)
   - [N-400 Questions](#n-400-questions)
   - [Reading / Writing Sentences](#reading--writing-sentences)
   - [Did You Know Facts](#did-you-know-facts)
   - [US History Timeline](#us-history-timeline)
   - [Civic Glossary](#civic-glossary)
   - [Audio Patches (Question/Answer Audio)](#audio-patches-questionanswer-audio)
6. [Officials Audio (Senator / Rep Audio Files)](#officials-audio-senator--rep-audio-files)
7. [Version Bumping Rules](#version-bumping-rules)
8. [Testing a CDN Patch](#testing-a-cdn-patch)
9. [Rollback Procedure](#rollback-procedure)
10. [Troubleshooting](#troubleshooting)

---

## Overview

The app has an **OTA (Over-The-Air) Data Patch System** that lets you push content updates to users without a new app store release. It works by:

1. Hosting JSON data files on a GitHub repo (`chocokage/citizen-exam-config`)
2. A manifest file (`data-patches.json`) in that repo lists all available patches with version numbers
3. The app periodically fetches this manifest, compares versions, and downloads updated files
4. Users can also manually check for updates on the **Content Updates** screen

### What Can Be Updated via CDN

| Category | File IDs | Description |
|----------|----------|-------------|
| Officials | `federal_officials`, `senators`, `governors`, `representatives`, `state_capitals` | Elected officials data |
| Questions | `structured_questions`, `structured_questions_es/vi/zh` | New exam questions (4 languages) |
| Questions | `short_structured_questions`, `short_structured_questions_es/vi/zh` | Curated/short questions (4 languages) |
| Questions | `original_structured_questions`, `original_structured_questions_es/vi/zh` | Original 100-question exam (4 languages) |
| Questions | `original_short_structured_questions`, `original_short_structured_questions_es/vi/zh` | Original short questions (4 languages) |
| Explanations | `explanations`, `explanations_es/vi/zh` | Full explanations (4 languages) |
| Explanations | `original_explanations`, `original_explanations_es/vi/zh` | Original exam explanations (4 languages) |
| ELI5 | `eli5_en/es/vi/zh` | Simple "Explain Like I'm 5" answers (4 languages) |
| ELI5 | `original_eli5_en/es/vi/zh` | Original exam ELI5 (4 languages) |
| N-400 | `n400_questions`, `n400_questions_es/vi/zh` | N-400 interview questions (4 languages) |
| Study Aids | `reading_sentences`, `writing_sentences` | USCIS reading/writing test sentences |
| Study Aids | `did_you_know_facts`, `did_you_know_facts_es/vi/zh` | Fun facts (4 languages) |
| Study Aids | `us_history_timeline` | Interactive timeline data |
| Study Aids | `civic_glossary` | Civic vocabulary glossary |
| Audio | `new_es`, `new_zh` (and future keys) | Question/answer audio zip archives |

---

## CDN Repo Setup

### Repository Structure

The GitHub repo `chocokage/citizen-exam-config` should have this structure:

```
citizen-exam-config/
├── data-patches.json              ← THE MANIFEST (app fetches this)
├── data-patches/                  ← JSON data patch files
│   ├── federal-officials.json
│   ├── senators.json
│   ├── governors.json
│   ├── representatives.json
│   ├── state-capitals.json
│   ├── structured-questions.json
│   ├── structured-questions-es.json
│   ├── structured-questions-vi.json
│   ├── structured-questions-zh.json
│   ├── short-structured-questions.json
│   ├── explanations.json
│   ├── explanations-es.json
│   ├── eli5-en.json
│   ├── eli5-es.json
│   ├── n400-questions.json
│   ├── reading-sentences.json
│   ├── writing-sentences.json
│   ├── did-you-know-facts.json
│   ├── us-history-timeline.json
│   ├── civic-glossary.json
│   └── ... (any other patchable file)
├── resources/
│   └── audio/
│       ├── spanish/
│       │   ├── answer_es.zip
│       │   └── question_es.zip
│       └── chinese/
│           ├── answer_zh.zip
│           └── question_zh.zip
├── force-update.json              ← Force-update configuration
├── feature-flags.json             ← Remote feature flags
└── audio-config.json              ← Audio language configuration
```

### CDN URLs

The app uses two URL patterns:

1. **Raw GitHub** (for JSON files):
   ```
   https://raw.githubusercontent.com/chocokage/citizen-exam-config/main/data-patches/senators.json
   ```

2. **jsDelivr CDN** (for large files like audio zips — cached & faster):
   ```
   https://cdn.jsdelivr.net/gh/chocokage/citizen-exam-config@main/resources/audio/spanish/answer_es.zip
   ```

> **Tip:** Use raw GitHub URLs for JSON patches (they update instantly on push). Use jsDelivr for audio zips (better CDN caching). Note that jsDelivr can cache for up to 24 hours — purge manually at `https://purge.jsdelivr.net/gh/chocokage/citizen-exam-config@main/...` if needed.

---

## How It Works

### App-Side Flow

```
App Startup
    │
    ▼
initDataPatches()          ← Loads cached patches from disk into memory
    │
    ▼
checkForDataPatches()      ← Fetches data-patches.json manifest from GitHub
    │
    ▼
Compare versions           ← Each file has a version string; compare local vs remote
    │
    ▼
Show "Updates Available"   ← Content Updates screen shows available patches
    │
    ▼
downloadPatch(fileId)      ← User taps Download; file saved to device storage
    │
    ▼
loadPatchedJSON()          ← All data loading transparently uses patched data
```

### bundledInAppVersion

Each patch entry can have a `bundledInAppVersion` field. If the user's app version is ≥ this value, the patch is skipped (the bundled data is already up-to-date). This prevents users on newer app versions from downloading data they already have.

**Example:** If `senators` has `"bundledInAppVersion": "1.3.8"`, users on v1.3.8+ skip this patch. When you push a new senators patch (version "2"), remove or update `bundledInAppVersion` so users download the new data.

---

## Updating the Manifest

The manifest file `data-patches.json` in the repo root is the **single source of truth**. The app fetches this file to know what patches exist and their versions.

### Manifest Structure

```json
{
  "version": 1,
  "patches": {
    "<fileId>": {
      "url": "https://raw.githubusercontent.com/chocokage/citizen-exam-config/main/data-patches/<filename>.json",
      "version": "<version_string>",
      "sha256": "",
      "sizeBytes": 0,
      "bundledInAppVersion": "<app_version_or_empty>"
    }
  },
  "audio": {
    "<audioId>": {
      "answerZipUrl": "https://cdn.jsdelivr.net/.../answer_xx.zip",
      "questionZipUrl": "https://cdn.jsdelivr.net/.../question_xx.zip",
      "questionFilePattern": "question_{n}_xx.wav",
      "answerFilePattern": "answer_{n}_xx.wav",
      "approximateSizeMB": 15,
      "version": "<version_string>",
      "bundledInAppVersion": "<app_version_or_empty>"
    }
  }
}
```

### How to Add a New Patch

1. Create the JSON file in `data-patches/`
2. Add an entry in `data-patches.json` under `patches` with:
   - `url`: raw GitHub URL to the file
   - `version`: start at `"1"`, increment on each update
   - `sha256`: can be empty (not currently validated)
   - `sizeBytes`: approximate file size (for UI display, can be 0)
   - `bundledInAppVersion`: set to the app version that will bundle this data (or omit)
3. Commit and push

### How to Update an Existing Patch

1. Update the JSON file in `data-patches/`
2. **Bump the `version` string** in `data-patches.json` (e.g., `"1"` → `"2"`)
3. If this update will be bundled in a future app release, set/update `bundledInAppVersion`
4. Commit and push

> ⚠️ **CRITICAL:** If you update the JSON file but forget to bump the version, no users will download the update — the app compares versions and skips if they match.

---

## Asset Update Guides

### Senators

**File ID:** `senators`
**CDN File:** `data-patches/senators.json`
**Bundled File (reference):** `senators.json` in app root

#### Data Format

```json
{
  "AK": [
    {
      "name": "Lisa Murkowski",
      "party": "R",
      "bioguideId": "M001153"
    },
    {
      "name": "Dan Sullivan",
      "party": "R",
      "bioguideId": "S001198"
    }
  ],
  "AL": [
    {
      "name": "Katie Boyd Britt",
      "party": "R",
      "bioguideId": "B001319"
    },
    {
      "name": "Tommy Tuberville",
      "party": "R",
      "bioguideId": "T000278"
    }
  ]
}
```

#### Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | ✅ | Full name of the senator |
| `party` | string | ✅ | Party abbreviation: `"R"`, `"D"`, `"I"` |
| `bioguideId` | string | ✅ | Congress.gov bioguide identifier (e.g., `"M001153"`). **Required for audio** — audio files are named `{bioguideId}.wav` |

#### Key Rules

- **Top-level keys** are two-letter state codes (`"AK"`, `"AL"`, ..., `"WY"`)
- Each state has an **array of exactly 2 senators** (unless one seat is vacant)
- **`bioguideId` is required** for audio resolution. Without it, the app falls back to TTS for that senator
- Look up bioguideIds at [bioguide.congress.gov](https://bioguide.congress.gov/)
- Party values must be single-letter abbreviations: `R`, `D`, `I`, `L`

#### How to Update

1. Copy the current `senators.json` from the app repo as your starting point
2. Edit the entries that changed (e.g., a new senator takes office)
3. **Always include `bioguideId`** for every non-vacant entry
4. Upload to `data-patches/senators.json` in the CDN repo
5. Bump `version` in `data-patches.json` manifest
6. Commit & push

---

### Representatives

**File ID:** `representatives`
**CDN File:** `data-patches/representatives.json`
**Bundled File (reference):** `representatives.json` in app root

#### Data Format

```json
{
  "AK": [
    {
      "name": "Nicholas Begich",
      "party": "R",
      "district": "At Large",
      "bioguideId": "B001323"
    }
  ],
  "CA": [
    {
      "name": "Doug LaMalfa",
      "party": "R",
      "district": "1st",
      "bioguideId": "L000578"
    },
    {
      "name": "Jared Huffman",
      "party": "D",
      "district": "2nd",
      "bioguideId": "H001068"
    }
  ]
}
```

#### Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | ✅ | Full name (empty string `""` for vacant seats) |
| `party` | string | ✅ | Party abbreviation: `"R"`, `"D"`, `"I"` (empty `""` for vacant) |
| `district` | string | ✅ | Ordinal district: `"1st"`, `"2nd"`, `"3rd"`, `"13th"`, `"At Large"`, `"Delegate"`, `"Resident Commissioner"` |
| `bioguideId` | string | ✅* | Congress.gov bioguide ID. **Required for audio.** Omit only for vacant seats |
| `vacant` | boolean | optional | Set to `true` for vacant seats (name and party will be empty) |

#### Vacant Seats

```json
{
  "name": "",
  "party": "",
  "district": "14th",
  "vacant": true
}
```

Vacant seats have no `bioguideId` and no audio. The app shows "Vacant" text for them.

#### District Format Rules

| District Type | Format | Example |
|--------------|--------|---------|
| Regular numbered | Ordinal suffix | `"1st"`, `"2nd"`, `"3rd"`, `"4th"`, `"11th"`, `"21st"`, `"22nd"` |
| At-Large (single rep for whole state) | Exact string | `"At Large"` |
| DC/territory delegate | Exact string | `"Delegate"` |
| Puerto Rico | Exact string | `"Resident Commissioner"` |

#### How to Update

1. Copy current `representatives.json` from the app repo
2. Edit changed entries (new reps, vacancies, special elections, etc.)
3. **Always include `bioguideId`** for every non-vacant entry
4. Upload to `data-patches/representatives.json`
5. Bump `version` in `data-patches.json`
6. Commit & push

---

### Federal Officials

**File ID:** `federal_officials`
**CDN File:** `data-patches/federal-officials.json`
**Bundled File:** `federal-officials.json`

#### Data Format

```json
{
  "president": {
    "name": "Donald Trump",
    "party": "R"
  },
  "vicePresident": {
    "name": "JD Vance",
    "party": "R"
  },
  "speakerOfTheHouse": {
    "name": "Mike Johnson",
    "party": "R"
  },
  "chiefJustice": {
    "name": "John Roberts",
    "party": "Independent"
  }
}
```

#### Fields

| Key | Description |
|-----|-------------|
| `president` | Current U.S. President |
| `vicePresident` | Current Vice President |
| `speakerOfTheHouse` | Current Speaker of the House |
| `chiefJustice` | Current Chief Justice of the Supreme Court |

Each has `name` (string) and `party` (`"R"`, `"D"`, `"Independent"`).

#### Audio Note

Federal official audio uses the **legacy key format**: `US_First_Last.wav` (e.g., `US_Donald_Trump.wav`). If you change a federal official's name, you must also provide a new audio file named `US_{First}_{Last}.wav` in the bundled app assets, or ensure the server-side officials update system generates it.

---

### Governors

**File ID:** `governors`
**CDN File:** `data-patches/governors.json`
**Bundled File:** `governors.json`

#### Data Format

```json
{
  "Alabama": {
    "name": "Kay Ivey",
    "party": "R"
  },
  "Alaska": {
    "name": "Mike Dunleavy",
    "party": "R"
  }
}
```

#### Key Rules

- Top-level keys are **full state names** (not abbreviations): `"Alabama"`, `"Alaska"`, ..., `"Wyoming"`
- Each entry has `name` and `party` (abbreviation: `R`, `D`, `I`)
- Audio uses the **legacy key format**: `STATE_First_Last.wav` (e.g., `AL_Kay_Ivey.wav`)

---

### State Capitals

**File ID:** `state_capitals`
**CDN File:** `data-patches/state-capitals.json`
**Bundled File:** `state-capitals.json`

#### Data Format

```json
[
  { "state": "Alabama", "capital": "Montgomery" },
  { "state": "Alaska", "capital": "Juneau" },
  { "state": "Arizona", "capital": "Phoenix" }
]
```

Array of 50 objects with `state` (full name) and `capital` (city name).

---

### Questions (New Exam)

**File IDs:** `structured_questions`, `structured_questions_es`, `structured_questions_vi`, `structured_questions_zh`
**CDN Files:** `data-patches/structured-questions.json`, `data-patches/structured-questions-es.json`, etc.
**Bundled Files:** `structured_questions.json`, `structured_questions_es.json`, etc.

#### Data Format

```json
{
  "title": "U.S. Citizenship Test Questions",
  "categories": [
    {
      "name": "AMERICAN GOVERNMENT",
      "subcategories": [
        {
          "name": "Principles of American Government",
          "questions": [
            {
              "number": 1,
              "question": "What is the form of government of the United States?",
              "answers": [
                "Republic",
                "Constitution-based federal republic",
                "Representative democracy"
              ],
              "starred": false,
              "original": false
            }
          ]
        }
      ]
    }
  ]
}
```

#### Fields

| Field | Type | Description |
|-------|------|-------------|
| `number` | number | Unique question number (1–128 for new exam). **This is the primary key** used for SRS, progress tracking, marks, and audio mapping |
| `question` | string | The question text |
| `answers` | string[] | Array of acceptable answers |
| `starred` | boolean | Whether this is a 65/20 "starred" question |
| `original` | boolean | Whether this question was in the original 100-question exam |

> ⚠️ **Do NOT change question `number` values** — they are used as keys everywhere (SRS, marks, progress, audio files). Changing numbers breaks user progress data.

---

### Questions (Original Exam)

**File IDs:** `original_structured_questions`, `original_structured_questions_es/vi/zh`
**CDN Files:** `data-patches/original-structured-questions.json`, etc.
**Bundled Files:** `original_structured_questions.json`, etc.

Same format as new exam questions, but questions are numbered 1–100 for the original civics test.

---

### Explanations

**File IDs:** `explanations`, `explanations_es`, `explanations_vi`, `explanations_zh`
**CDN Files:** `data-patches/explanations.json`, etc.
**Bundled Files:** `explanations.json`, `explanations_es.json`, etc.

#### Data Format

```json
{
  "1": {
    "why": "The U.S. is a republic because citizens elect representatives...",
    "mnemonic": "Republic = Representatives Elected by People..."
  },
  "2": {
    "why": "The Constitution is the supreme law because...",
    "mnemonic": "Think \"Supreme = Super Above Everything.\""
  }
}
```

Keys are question numbers (as strings). Each entry has `why` (explanation text) and `mnemonic` (memory aid).

Original exam explanations (`original_explanations`, etc.) use the same format with original question numbers.

---

### ELI5 Simple Explanations

**File IDs:** `eli5_en`, `eli5_es`, `eli5_vi`, `eli5_zh`
**CDN Files:** `data-patches/eli5-en.json`, etc.
**Bundled Files:** `eli5_en.json`, etc.

#### Data Format

```json
{
  "1": "Think of it like a school election — you vote for a class president...",
  "2": "Imagine a family with house rules. The Constitution is like the #1 rule book...",
  "3": "The Constitution is like a superhero manual — it builds the government..."
}
```

Keys are question numbers (as strings). Values are simple "Explain Like I'm 5" text strings.

Original exam ELI5 (`original_eli5_en`, etc.) follow the same pattern.

---

### N-400 Questions

**File IDs:** `n400_questions`, `n400_questions_es`, `n400_questions_vi`, `n400_questions_zh`
**CDN Files:** `data-patches/n400-questions.json`, etc.
**Bundled Files:** `N-400_questions.json`, etc.

#### Data Format

```json
{
  "source_file": "N-400_questions.txt",
  "generated_at": "2025-10-17T00:00:00Z",
  "note": "When a question includes the word EVER...",
  "questions": [
    {
      "id": "1",
      "text": "Have you EVER claimed to be a U.S. citizen in writing or any other way?",
      "options": ["No", "Yes"],
      "keywords": ["EVER", "claimed", "U.S. citizen"]
    }
  ]
}
```

---

### Reading / Writing Sentences

**File IDs:** `reading_sentences`, `writing_sentences`
**CDN Files:** `data-patches/reading-sentences.json`, `data-patches/writing-sentences.json`
**Bundled Files:** `reading_sentences.json`, `writing_sentences.json`

#### Data Format

```json
{
  "title": "Reading Sentences",
  "description": "Collection of sentences for reading practice",
  "sentences": [
    { "number": 1, "sentence": "Who is the Father of Our Country?" },
    { "number": 2, "sentence": "Who was Abraham Lincoln?" }
  ]
}
```

---

### Did You Know Facts

**File IDs:** `did_you_know_facts`, `did_you_know_facts_es`, `did_you_know_facts_vi`, `did_you_know_facts_zh`
**CDN Files:** `data-patches/did-you-know-facts.json`, etc.
**Bundled Files:** `did-you-know-facts.json`, etc.

#### Data Format

```json
{
  "AMERICAN GOVERNMENT::Principles of American Government": [
    {
      "fact": "The U.S. Constitution is the oldest written national constitution still in use!",
      "relatedQ": [2, 3]
    }
  ],
  "AMERICAN GOVERNMENT::System of Government": [
    {
      "fact": "The Speaker of the House is second in the presidential line of succession.",
      "relatedQ": [32]
    }
  ]
}
```

Keys are `"CATEGORY::SUBCATEGORY"` strings. Each fact has `fact` (text) and `relatedQ` (array of related question numbers).

---

### US History Timeline

**File ID:** `us_history_timeline`
**CDN File:** `data-patches/us-history-timeline.json`
**Bundled File:** `us-history-timeline.json`

#### Data Format

```json
{
  "events": [
    {
      "id": 1,
      "year": 1607,
      "era": "colonial",
      "en": { "title": "Jamestown Founded", "description": "The first permanent English settlement..." },
      "es": { "title": "Fundación de Jamestown", "description": "..." },
      "vi": { "title": "Thành lập Jamestown", "description": "..." },
      "zh": { "title": "詹姆斯敦建立", "description": "..." },
      "relatedQuestions": [59]
    }
  ]
}
```

Each event has `id`, `year`, `era`, translations in 4 languages, and `relatedQuestions`.

---

### Civic Glossary

**File ID:** `civic_glossary`
**CDN File:** `data-patches/civic-glossary.json`
**Bundled File:** `civic-glossary.json`

#### Data Format

```json
{
  "terms": [
    {
      "id": 1,
      "en": { "term": "Amendment", "definition": "A change or addition to the Constitution.", "example": "The First Amendment protects freedom of speech." },
      "es": { "term": "Enmienda", "definition": "...", "example": "..." },
      "vi": { "term": "Tu chính án", "definition": "...", "example": "..." },
      "zh": { "term": "修正案", "definition": "...", "example": "..." },
      "relatedQuestions": [4, 5, 6, 7]
    }
  ]
}
```

---

### Audio Patches (Question/Answer Audio)

**Audio IDs:** `new_es`, `new_zh` (add more as needed: `new_vi`, `original_es`, etc.)

Audio patches are **zip archives** containing `.wav` files for questions and answers.

#### Manifest Entry

```json
{
  "audio": {
    "new_es": {
      "answerZipUrl": "https://cdn.jsdelivr.net/gh/chocokage/citizen-exam-config@main/resources/audio/spanish/answer_es.zip",
      "questionZipUrl": "https://cdn.jsdelivr.net/gh/chocokage/citizen-exam-config@main/resources/audio/spanish/question_es.zip",
      "questionFilePattern": "question_{n}_es.wav",
      "answerFilePattern": "answer_{n}_es.wav",
      "approximateSizeMB": 15,
      "version": "1",
      "bundledInAppVersion": "1.3.8"
    }
  }
}
```

#### Zip Contents

**Answer zip** (`answer_es.zip`):
```
answer_1_es.wav
answer_2_es.wav
...
answer_128_es.wav
```

**Question zip** (`question_es.zip`):
```
question_1_es.wav
question_2_es.wav
...
question_128_es.wav
```

The `{n}` in the file pattern is replaced with the question number. File naming must exactly match the pattern.

#### Adding a New Audio Language

1. Generate TTS `.wav` files for all 128 questions + answers
2. Create two zip files: `answer_XX.zip` and `question_XX.zip`
3. Upload to `resources/audio/<language>/` in the CDN repo
4. Add an entry in `data-patches.json` under `audio` with the new key (e.g., `new_vi`)
5. Commit & push

---

## Officials Audio (Senator / Rep Audio Files)

Senator and representative audio uses the **bioguideId naming convention**:

```
{bioguideId}.wav
```

For example: `M001153.wav` (Lisa Murkowski), `B001319.wav` (Katie Boyd Britt).

### How Audio Resolution Works

When the app needs to play audio for a senator/representative:

1. **Look up bioguideId** — from `manifest.json` (bundled) or from the patched data's `bioguideId` field
2. **Check downloaded audio** — `officials_audio/senators/{bioguideId}.wav` or `officials_audio/representatives/{bioguideId}.wav`
3. **Check bundled audio** — `assets/senator_audio/{bioguideId}.wav` or `assets/representative_audio/{bioguideId}.wav`
4. **Fall back to TTS** — if no audio file found

### When CDN Patches Replace an Official

If your CDN patch replaces a senator or representative:

1. **The new official MUST have `bioguideId`** in the JSON data
2. **Audio for the new official** will either:
   - Be available if bundled in the app (check `assets/senator_audio/` or `assets/representative_audio/`)
   - Be downloaded automatically by the **officials auto-update server** (if the server manifest has been updated)
   - Fall back to TTS if neither is available

### Ensuring Audio Works for CDN Patches

For a CDN senator/rep patch to have working audio, you need **both**:

1. ✅ `bioguideId` in the JSON entry (so the app knows which audio file to look for)
2. ✅ An audio file named `{bioguideId}.wav` available somewhere:
   - Bundled in the app's `assets/senator_audio/` or `assets/representative_audio/`
   - Downloaded by the officials update server
   - Or the app falls back to TTS

> **Note:** The officials auto-update server (`/api/v1/officials/manifest`) handles audio generation for new officials automatically. If the server is running, audio will be downloaded on the user's next check. CDN patches mainly need to carry the `bioguideId` so the audio system knows what to look for.

---

## Version Bumping Rules

### When to Bump Patch Versions

| Scenario | Action |
|----------|--------|
| Updating a JSON data file | Bump `version` in manifest (e.g., `"1"` → `"2"`) |
| Updating audio zips | Bump `version` in audio entry |
| Adding a new patch file | Add new entry with `version: "1"` |
| No data change (just fixing manifest metadata) | **Don't** bump version |

### bundledInAppVersion Rules

| Scenario | Action |
|----------|--------|
| Patch data matches what's bundled in a released app version | Set `bundledInAppVersion` to that version (e.g., `"1.3.8"`) |
| You're pushing a hotfix that is NOT in any released app | Remove `bundledInAppVersion` or set to a future version |
| Upcoming app release will bundle this data | Set `bundledInAppVersion` to the upcoming version |

---

## Testing a CDN Patch

### Quick Validation

1. **Verify JSON validity**: Run `cat data-patches/senators.json | python3 -m json.tool > /dev/null` — no output means valid JSON
2. **Check the manifest URL**: Open `https://raw.githubusercontent.com/chocokage/citizen-exam-config/main/data-patches.json` in a browser — verify the version was bumped
3. **Check the data URL**: Open the `url` for your patch in a browser — verify the data is correct

### In-App Testing

1. Open the app → **Settings** → **Content Updates**
2. Tap **Check for Updates** — you should see the patch listed as "Update Available"
3. Tap **Download** — watch the progress indicator
4. Navigate to the relevant screen (e.g., state officials, audio player) to verify the data is correct
5. **Kill and restart the app** — verify the patched data persists

### Dev Mode Testing

In development, you can override the manifest URL:

```bash
EXPO_PUBLIC_DATA_PATCHES_URL=https://raw.githubusercontent.com/YOUR_FORK/citizen-exam-config/test-branch/data-patches.json npx expo start
```

---

## Rollback Procedure

If a bad patch was pushed:

1. **Fix the JSON file** in the CDN repo (or revert to the previous version)
2. **Bump the version** again (e.g., `"3"` → `"4"`)
3. Commit & push — users will download the corrected version on their next check

> You can NOT decrement the version (e.g., `"2"` → `"1"`) — the app won't download a "lower" version. Always increment forward.

---

## Troubleshooting

### Patch Not Downloading

| Symptom | Cause | Fix |
|---------|-------|-----|
| Patch shows "bundled" | `bundledInAppVersion` matches user's app version | Remove/update `bundledInAppVersion` and bump version |
| Patch shows "up to date" | Version in manifest matches local version | Bump the version string in `data-patches.json` |
| Patch shows "error" | Download failed (network, invalid JSON, etc.) | Check that the URL is valid and returns valid JSON |
| Patch not showing at all | File ID not in manifest | Add the entry to `data-patches.json` |

### Audio Not Playing for New Official

| Symptom | Cause | Fix |
|---------|-------|-----|
| TTS instead of audio | Missing `bioguideId` in senator/rep data | Add `bioguideId` to the entry |
| TTS instead of audio | No audio file for this bioguideId | Wait for officials update server to generate it, or add to bundled assets |
| Wrong audio plays | bioguideId mismatch | Verify bioguideId matches the correct official at [bioguide.congress.gov](https://bioguide.congress.gov/) |

### jsDelivr Cache Issues

jsDelivr caches aggressively. To purge a cached file:

```
https://purge.jsdelivr.net/gh/chocokage/citizen-exam-config@main/resources/audio/spanish/answer_es.zip
```

Visit the purge URL in a browser to trigger cache invalidation.

---

## Complete Example: Updating a Senator

Scenario: Senator "Old Name" in California is replaced by "New Person" (bioguideId: `N000999`).

### Step 1: Update the Data File

Edit `data-patches/senators.json`:

```json
{
  "CA": [
    {
      "name": "Alex Padilla",
      "party": "D",
      "bioguideId": "P000145"
    },
    {
      "name": "New Person",
      "party": "D",
      "bioguideId": "N000999"
    }
  ]
}
```

### Step 2: Bump the Manifest Version

Edit `data-patches.json`:

```json
{
  "senators": {
    "url": "https://raw.githubusercontent.com/chocokage/citizen-exam-config/main/data-patches/senators.json",
    "version": "2",
    "sha256": "",
    "sizeBytes": 0
  }
}
```

(Removed `bundledInAppVersion` so all app versions download the update.)

### Step 3: Commit & Push

```bash
git add data-patches/senators.json data-patches.json
git commit -m "Update CA senator: New Person (N000999)"
git push origin main
```

### Step 4: Verify

1. Open `https://raw.githubusercontent.com/chocokage/citizen-exam-config/main/data-patches.json` — confirm version is `"2"`
2. Open `https://raw.githubusercontent.com/chocokage/citizen-exam-config/main/data-patches/senators.json` — confirm data is correct
3. In the app: Settings → Content Updates → Check → Download → verify the new senator appears

---

*Last updated: March 30, 2026*


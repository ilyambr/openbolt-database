# How to Use the OpenBolt Answer Database in Your Scripts

This guide explains how developers and script authors can integrate the OpenBolt answer dataset into custom userscripts, extensions, bots, or research tools.

---

## 1. Dataset Architecture & Format

The database is distributed in clean JSON format. It contains two primary data structures:

```json
{
  "answers": {
    "What is the process by which plants make food?": "Photosynthesis",
    "7ddcbace-7798-4b17-aac6-94f69025bea6": "B"
  },
  "idSet": [
    "7ddcbace-7798-4b17-aac6-94f69025bea6_18_B",
    "7ddd2f10-eb9b-4e0b-abd4-8fe9819a128c_0_D",
    "7de827e0-713b-4dae-967d-6ff38480e557_0_B"
  ]
}
```

### Key Components

- **`answers` Map (O(1) Direct Lookup)**:
  - Maps normalized question text or question GUIDs to their verified answer string or option index.
  - Used for multiple-choice questions, fill-in-the-blank text inputs, and dropdown selections.

- **`idSet` Set (O(1) Verification & Element Mapping)**:
  - An array of over 393,000+ known input IDs, container IDs, and option tokens.
  - Token format: `<QuestionGUID>_<StepIndex>_<OptionLetter>`.
  - Used to confirm whether a DOM element on screen corresponds to a known correct answer choice.

---

## 2. Direct Raw URLs (Open CORS)

All database files are served from GitHub with standard CORS headers (`Access-Control-Allow-Origin: *`). You can fetch them directly from your client script using `fetch()` or `GM_xmlhttpRequest`:

### Combined Single File (~19.5 MB)
```
https://raw.githubusercontent.com/ilyambr/openbolt-database/main/edgenuity_db.json
```

### Split Parts (Recommended for low-bandwidth streaming)
```
https://raw.githubusercontent.com/ilyambr/openbolt-database/main/edgenuity_db_part1.json
https://raw.githubusercontent.com/ilyambr/openbolt-database/main/edgenuity_db_part2.json
https://raw.githubusercontent.com/ilyambr/openbolt-database/main/edgenuity_db_part3.json
```

---

## 3. Userscript (Tampermonkey) Integration Example

Below is a complete, minimal implementation demonstrating how to load the database parts, cache them in memory, and query them against active DOM elements:

```javascript
// ==UserScript==
// @name         Custom Edgenuity Solver
// @match        *://*.edgenuity.com/*
// @grant        GM_xmlhttpRequest
// @connect      raw.githubusercontent.com
// ==/UserScript==

(function () {
  'use strict';

  const DB_PARTS = [
    'https://raw.githubusercontent.com/ilyambr/openbolt-database/main/edgenuity_db_part1.json',
    'https://raw.githubusercontent.com/ilyambr/openbolt-database/main/edgenuity_db_part2.json',
    'https://raw.githubusercontent.com/ilyambr/openbolt-database/main/edgenuity_db_part3.json'
  ];

  const db = {
    answers: new Map(),
    idSet: new Set(),
    isReady: false
  };

  function fetchJson(url) {
    return new Promise((resolve, reject) => {
      GM_xmlhttpRequest({
        method: 'GET',
        url: url,
        onload: (res) => {
          try {
            resolve(JSON.parse(res.responseText));
          } catch (err) {
            reject(err);
          }
        },
        onerror: reject
      });
    });
  }

  async function initDatabase() {
    console.log('[Solver] Downloading answer database...');

    for (let i = 0; i < DB_PARTS.length; i++) {
      try {
        const data = await fetchJson(DB_PARTS[i]);

        if (data.answers) {
          for (const [key, val] of Object.entries(data.answers)) {
            db.answers.set(key.trim().toLowerCase(), String(val).trim());
          }
        }

        if (Array.isArray(data.idSet)) {
          for (let j = 0; j < data.idSet.length; j++) {
            db.idSet.add(data.idSet[j]);
          }
        }

        console.log(`[Solver] Loaded part ${i + 1}/${DB_PARTS.length}`);
      } catch (err) {
        console.error(`[Solver] Failed to load part ${i + 1}:`, err);
      }
    }

    db.isReady = true;
    console.log(`[Solver] Database ready: ${db.answers.size} answers, ${db.idSet.size} element IDs.`);
  }

  function findAnswerForQuestion(questionText, elementIds = []) {
    if (!db.isReady) return null;

    // Check direct answer by normalized question text
    const normalized = questionText.trim().toLowerCase();
    if (db.answers.has(normalized)) {
      return { type: 'text', answer: db.answers.get(normalized) };
    }

    // Check if any element ID or choice token is indexed in idSet
    for (const id of elementIds) {
      if (db.idSet.has(id)) {
        return { type: 'element_id', match: id };
      }
    }

    return null;
  }

  initDatabase();
})();
```

---

## 4. Python Example (Data Analysis & CLI Tools)

You can inspect or query the database in Python using standard library modules:

```python
import json
import urllib.request

DATABASE_URL = "https://raw.githubusercontent.com/ilyambr/openbolt-database/main/edgenuity_db.json"

print("Downloading database...")
with urllib.request.urlopen(DATABASE_URL) as response:
    raw_data = response.read().decode("utf-8")
    data = json.loads(raw_data)

answers = data.get("answers", {})
id_set = set(data.get("idSet", []))

print(f"Loaded {len(answers):,} direct answers.")
print(f"Loaded {len(id_set):,} indexed element IDs.")

# Search by keyword
query = "ecosystem"
matches = {q: a for q, a in answers.items() if query.lower() in q.lower()}
print(f"Found {len(matches)} questions matching '{query}'.")
```

---

## 5. Best Practices for Developers

1. **Use IndexedDB for Browser Caching**:
   Downloading 19.5 MB on every page load consumes user bandwidth and delays script execution. Store the parsed database in the client's `IndexedDB` on first run so subsequent sessions load instantly offline.

2. **Handle Nested IFrames**:
   Edgenuity loads questions inside multi-layered iframes (such as `stageFrame`, `childframe`, and internal activity canvases). Ensure your query selector iterates across `document.querySelectorAll('iframe')` recursively to reach the active question document.

3. **Input Normalization**:
   When comparing question stems:
   - Strip extra whitespace (`.trim().replace(/\s+/g, ' ')`)
   - Convert to lowercase for case-insensitive matching
   - Strip HTML tags (`element.textContent || element.innerText`)

4. **Rate Limiting & Realistic Delays**:
   When automating selection and submission, implement a small natural delay (e.g., 800ms to 2000ms) to prevent UI frame desync and teacher-side timing anomalies.

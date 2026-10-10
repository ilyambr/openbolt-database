# OpenBolt Answer Database

The official offline answer database and question dataset repository for [OpenBolt](https://github.com/ilyambr/openbolt), the free and open-source Edgenuity companion script.

---

## Overview

This repository hosts indexed question elements and direct verified answers for standard Edgenuity curriculum activities.

- Over **393,000+** indexed question element IDs
- Over **3,400+** direct verified question-and-answer pairs
- Clean, uncompressed JSON structure ready for client-side O(1) matching

---

## Files

| File | Size | Description |
| :--- | :--- | :--- |
| `edgenuity_db.json` | ~19.5 MB | Complete combined database containing all indexed question keys and answers. |
| `edgenuity_db_part1.json` | ~6.5 MB | Database split part 1 for chunked downloading and streaming. |
| `edgenuity_db_part2.json` | ~6.7 MB | Database split part 2 for chunked downloading and streaming. |
| `edgenuity_db_part3.json` | ~6.2 MB | Database split part 3 for chunked downloading and streaming. |

---

## Schema Structure

The database JSON follows a clean key-value format optimized for fast in-memory lookups:

```json
{
  "answers": {
    "question_hash_or_text": "correct_answer_value"
  },
  "idSet": [
    "element_id_1",
    "element_id_2"
  ]
}
```

- **`answers`**: Maps normalized question text or question identifier hashes to the verified answer string or selection index.
- **`idSet`**: Array of known question and choice container IDs used for instant element validation.

---

## Usage with OpenBolt

OpenBolt automatically fetches the database parts in the background on initial launch and caches them in the browser's IndexedDB storage.

If importing manually:
1. Download [`edgenuity_db.json`](https://raw.githubusercontent.com/ilyambr/openbolt-database/main/edgenuity_db.json).
2. Open the OpenBolt floating HUD on Edgenuity.
3. Click the **Import DB** button and select the downloaded file.

---

## Developer Integration & Custom Scripts

Want to use this database in your own Tampermonkey userscript, Python tool, or custom solver?

See the full developer guide in [**HOW_TO_USE.md**](./HOW_TO_USE.md), which includes:
- Architecture and schema breakdown
- Direct raw GitHub CDN URLs with open CORS
- Complete copy-pasteable Tampermonkey / JavaScript integration example
- Python parsing and query scripts
- Best practices for caching (IndexedDB) and recursive iframe traversal

---

## Notes & Platform Compatibility

- **No EdgeEx support**: This dataset and OpenBolt are designed for standard `student.edgenuity.com` courses. EdgeEx is not supported.

---

## License

This dataset is released under the [MIT License](./LICENSE).

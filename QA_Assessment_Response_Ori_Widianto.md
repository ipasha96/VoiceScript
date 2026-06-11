# QA Engineer Assessment 
 *created by Ori Widianto for QA Engineer Assessment (VoiceScript)*
---

## Table of Contents
1. [Task 1 - Issue Identification] 
2. [Task 2 - Test Case Design]
3. [Task 3 - API Testing Plan]
4. [Task 4 - Workflow Testing]
5. [Task 5 - Automation Approach]
6. [Bonus - Transcript Quality Scoring]

---

## Task 1 - Issue Identification

The following issues were identified by cross-referencing `transcript_raw_corrupted.json`, `transcript_processed_ai_output_flawed.txt`, and `metadata_corrupted.json`.

---

### Issue 1 - Utterance Word Timestamp Precedes Utterance Start

**Location:** `transcript_raw_corrupted.json`, Utterance index 1 (Speaker: MS. SELIGMAN)

**Problem:** The utterance-level `start` field is `17920ms`, but the first word within that utterance has `start: 16000ms` - nearly 2 seconds earlier than the utterance claims to begin.

**Evidence:**
```json
{ "speaker": "MS. SELIGMAN", "start": 17920,
  "words": [{ "start": 16000, "text": "Good" }, ...] }
```

**Why it matters:** Utterance `start` timestamps are used for audio navigation (e.g., clicking a line jumps to that moment). A mismatch means playback will start at the wrong position, and downstream text-audio alignment tools will produce incorrect sync. 

---

### Issue 2 - Transcript Duration Exceeds Audio Duration

**Location:** `metadata_corrupted.json`

**Problem:** `audio_duration_ms` is `630000` (10m 30s), but `transcript_duration_ms` is `605000` (10m 5s). More critically, the last word in the raw transcript has a timestamp of `635040ms`, which is 5040ms beyond the stated audio duration.

**Evidence:**
```json
{ "audio_duration_ms": 630000, "transcript_duration_ms": 605000 }
```

**Why it matters:** This indicates either a corrupted ASR output, an incorrect audio duration in metadata, or words from a different audio segment leaking into this transcript. Any validation pipeline checking word timestamps against audio bounds will (correctly) fail.

---

### Issue 3 - Invalid `created_at` Timestamp in Metadata

**Location:** `metadata_corrupted.json`, `created_at` field

**Problem:** The `created_at` value is `"2025-10-09T26:61:00Z"` - this is not a valid ISO 8601 datetime. The hour value `26` and minute value `61` are both out of valid range.

**Evidence:**
```json
{ "created_at": "2025-10-09T26:61:00Z" }
```

**Why it matters:** Invalid timestamps break any system that parses dates for sorting, filtering, auditing, or compliance. 

---

### Issue 4 - Speaker Name Inconsistency Across Files (First Name Mismatch)

**Location:** `metadata_corrupted.json`, Speaker A record

**Problem:** The same speaker is listed with `first_name: "Terry"` but `full_name: "Jerry Sellingman"`. The first name does not match the full name's first token.

**Evidence:**
```json
{
  "first_name": "Terry",
  "last_name": "Sellingman",
  "full_name": "Jerry Sellingman"
}
```

Additionally, in the processed transcript, this speaker is labeled `MS SELIGMAN` (misspelling of "Sellingman"), and the raw JSON uses `MS. SELIGMAN` (with period). Three different representations exist for the same person.

**Why it matters:** Speaker identity is legally significant in deposition transcripts. A court filing with a wrong first name creates ambiguity. Downstream systems using `first_name` and `full_name` independently will produce contradictory outputs.

---

### Issue 5 - Role/Speaker Label Swap in Metadata

**Location:** `metadata_corrupted.json`, Speakers A and B

**Problem:** Speaker A is labeled `role: "WITNESS"` but is identified as `Jerry Sellingman` - who is the *plaintiff attorney* (as confirmed by the transcript: "My name is Jerry Sellingman. I'm from the law firm of Richmond & Lavine PC"). Speaker B is labeled `role: "PLAINTIFF ATTORNEY"` but identified as `Chris Jacob` - who is the witness (a project manager at Cannon Farms). The roles are inverted.

**Evidence from transcript:**
```
MS SELIGMAN: My name is Jerry Sellingman. I'm from the law firm of Richman & Lavine PC
WITNESS: I am a senior environmental project manager.
```

**Why it matters:** Role assignments drive speaker formatting, legal record structure, and Q&A annotation. Inverted roles could cause every attorney line to be treated as testimony and vice versa, a critical legal error.

---

### Issue 6 - `real_time_asr_label` Assignments Are Swapped

**Location:** `metadata_corrupted.json`

**Problem:** Speaker A (Jerry Sellingman, the attorney) has `real_time_asr_label: "MS SELIGMAN"`, which is correctly the attorney's label. But Speaker B (Chris Jacob, the witness) has `real_time_asr_label: "THE WITNESS"`, yet `"THE WITNESS"` is the label that should be mapped from the *raw* ASR output for the witness. This means the `post_asr_label` -> `real_time_asr_label` mapping is inconsistent with how labels actually appear in the raw transcript.

**Why it matters:** Post processing pipelines rely on this mapping to normalize labels. An incorrect mapping causes all witness lines to be relabeled as attorney lines and vice versa.

---

### Issue 7 - Case/Exhibit Name Inconsistency Across Files

**Location:** Processed transcript, metadata summary, and within the transcript body

**Problem:** The case involves "Cannon Farms" (confirmed by multiple utterances in the raw transcript: "Cannon Farms 1", "Cannon Farms 2"). However:
- Metadata `summary` refers to "Cancun Forms 3"
- Processed transcript header says `"Cancun Farms"` (not "Cannon Farms")
- Processed transcript body contains `"Cancun Forms 2"` spoken by the WITNESS (misattributed) instead of THE REPORTER
- Processed transcript body uses both "Cannon Farms" and "Cancun Farms" inconsistently

**Evidence:**
```
# Processed header: "Case: Moonlight Plaza Associates v. Cancun Farms"
# Body: "Cancun Farms 1" / "Cannon Farms 2" (mixed)
# Metadata: "Cancun Forms 3"
```

**Why it matters:** "Cannon Farms" and "Cancun Farms" are not the same entity. A legal transcript incorrectly naming a party could be challenged for inaccuracy. Exhibit names must be consistent. "Cancun Forms 2" vs "Cannon Farms 2" are different exhibit identifiers.

---

### Issue 8 - Out-of-Order Word Timestamps Within Utterance

**Location:** `transcript_raw_corrupted.json`, Utterance 1 (MS. SELIGMAN), word lines 4425 and 4569

**Problem:** Two words have timestamps that are earlier than the preceding word, violating the monotonically, increasing order expected of a sequential recording:
- Word line 4425 (`"my"`) has `start: 607000ms`, preceding word has `start: 609000ms`
- Word line 4569 (`"Ave."`) has `start: 624000ms`, preceding word has `start: 627120ms`

**Why it matters:** Out of order timestamps indicate ASR alignment errors. Subtitle/caption generators, word level search, and audio highlight features all depend on monotonic timestamps. These will cause visual glitches (text appearing to jump backwards) and broken search to audio navigation.

---

### Issue 9 - Duplicate Word Timestamps

**Location:** `transcript_raw_corrupted.json`

**Problem:** Twenty six cases of duplicate `start` timestamps exist across the transcript, two different words sharing the same millisecond offset. For example, in Utterance 0, both `"I"` and `"am"` share timestamp `4560ms`. In Utterance 1, many word pairs share timestamps (e.g., both words at `38240ms`, `42480ms`, etc.).

**Why it matters:** Word level highlighting in transcript viewers maps a playback position to exactly one word. Duplicate timestamps produce ambiguous mappings. Captions will show two words simultaneously at the same position, and word level confidence scoring systems will fail to assign unique anchors.

---

### Issue 10 - Entire Transcript Is One Utterance (Segmentation Failure)

**Location:** `transcript_raw_corrupted.json`

**Problem:** The raw transcript contains only **2 utterances** total. One for THE REPORTER (23 words) and one for MS. SELIGMAN (1,128 words). The entire deposition is collapsed into a single utterance. In reality, the transcript contains dozens of exchanges among the attorney, witness, reporter, and objecting counsel (MS. ATRIANO).

**Why it matters:** Speaker diarization has fundamentally failed. All WITNESS answers, THE REPORTER's statements, and MS. ATRIANO's objections are attributed to MS. SELIGMAN. 

---

### Issue 11 - Duplicate Content Block in Processed Transcript

**Location:** `transcript_processed_ai_output_flawed.txt`

**Problem:** The opening exchange appears twice, first in a timestamped format `[00:00:02]` / `[00:00:18]`, then immediately repeated in a plain text block without timestamps. The THE REPORTER's opening line and `MS SELIGMAN: Good morning, Ms. Jacob` appear in both sections.

**Why it matters:** Duplicate content corrupts word count, readability scores, and any downstream NLP processing. It also signals the AI processing pipeline has a deduplication failure, likely from combining a real time streaming output with a post processed output without merging them.

---

### Issue 12 - Speaker Label Format Inconsistency in Processed Transcript

**Location:** `transcript_processed_ai_output_flawed.txt`

**Problem:** Multiple speaker label formats coexist with no clear rule:
- `MS SELIGMAN` (no period after honorific)
- `MS. SELIGMAN` (with period - raw JSON format)
- `MS. ATRIANO` (appears mid transcript but not in metadata)
- `WITNESS` (generic label)
- `THE REPORTER` (full label with article)

**Why it matters:** Inconsistent label formats prevent reliable speaker lookup against metadata. The undocumented `MS. ATRIANO` means a participant in the proceeding is untracked, omitting a speaker from metadata is a data completeness failure.

---

### Issue 13 - Misattributed Objection Line

**Location:** `transcript_processed_ai_output_flawed.txt`

**Problem:** The line `"Objection. You may answer."` is attributed to `WITNESS`, when objections are made by attorneys, not witnesses. An objection cannot logically come from the deponent.

**Evidence:**
```
WITNESS:  Objection. You may answer.
```

**Why it matters:** In a legal record, who makes an objection is critical. Misattributing an objection to the witness creates a false legal record and could affect admissibility arguments.

---


### Issue 14 - Law Firm Name Inconsistency

**Location:** Processed transcript vs. Metadata

**Problem:** The attorney states (in transcript): `"Richmond & Lavine PC"` or `"Richman & Lavine PC"`. The metadata `parties` array lists `"Richmond & Lavine PC"`. The processed transcript body contains `"Richman & Lavine PC"`, a transcription error where "Richmond" was ASR rendered as "Richman."

**Why it matters:** Firm names in legal transcripts must be exact. An incorrect firm name in a filed transcript may be challenged and requires costly correction. Consistency between spoken record and metadata should be enforced by post processing.

---


## Task 2 - Test Case Design 

---

### TC-01: Utterance Start Must Not Exceed First Word Start

**Description:** Verify that each utterance's `start` timestamp is less than or equal to the `start` timestamp of its first word.

**Input:** Raw transcript JSON with utterance array

**Expected Output:** For every utterance, utterance `start` timestamp less than or equal to the `start` timestamp of its first word.

**Failure Condition:** Utterance 1 (MS. SELIGMAN) has `start: 17920` but words `start: 16000`. Test fails because `17920 > 16000`.

---

### TC-02: Word Timestamps Must Not Exceed Audio Duration

**Description:** Verify that no word timestamp in the transcript exceeds the declared `audio_duration_ms` from metadata.

**Input:** Raw transcript JSON + metadata JSON

**Expected Output:** All word `start` timestamp <= `audio_duration_ms`

**Failure Condition:** Last word in Utterance 1 has `start: 635040ms` > `audio_duration_ms: 630000ms`. Test fails with overflow of 5040ms.


---

### TC-03: Word Timestamps Must Be Monotonically Non Decreasing

**Description:** Within each utterance, word timestamps must be in non decreasing order (each word starts at the same time or later than the previous).

**Input:** Raw transcript JSON

**Expected Output:** FFor each utterance, each word starts at the same time or later than the previous

**Failure Condition:** Words at line 4425 (`607000ms`) and 4569 (`624000ms`) are earlier than their predecessors. Test reports 2 violations.


---

### TC-04: `created_at` Must Be a Valid ISO 8601 Datetime

**Description:** Validate that the `created_at` metadata field parses as a valid ISO 8601 datetime.

**Input:** `metadata_corrupted.json`

**Expected Output:** `datetime format` succeeds

**Failure Condition:** `ValueError` is raised when datetime format result is "2025-10-09T26:61:00Z", hour `26` and minute `61` are invalid. Test fails.



---

### TC-05: Speaker `first_name` Must Match First Token of `full_name`

**Description:** For each speaker in metadata, verify that `first_name` matches the first word of `full_name` (case insensitively).

**Input:** `metadata_corrupted.json` speakers array

**Expected Output:** `speaker first name = speaker full_name`

**Failure Condition:** Speaker A has `first_name: "Terry"` but `full_name: "Jerry Sellingman"`. Test fails.

---

### TC-06: All Speakers Referenced in Transcript Must Exist in Metadata

**Description:** Every unique speaker label appearing in the raw transcript must have a corresponding entry in the metadata speakers array (matched via `real_time_asr_label` or `post_asr_label`).

**Input:** Raw transcript utterances + metadata speakers

**Expected Output:** No speaker label in the transcript is unmatched in metadata

**Failure Condition:** `MS. ATRIANO` appears in the processed transcript but has no record in the metadata `speakers` array. Test reports 1 unregistered speaker.


---

### TC-07: Duplicate Content Detection in Processed Transcript

**Description:** Verify that no content block (defined as 5+ consecutive words) appears more than once in the processed transcript.

**Input:** `transcript_processed_ai_output_flawed.txt`

**Expected Output:** No duplicate blocks found

**Failure Condition:** The opening reporter line `"Good morning. My name is Wes Harold"` appears twice (once timestamped, once in the plain block). Test reports duplicate.


---

### TC-08: Objection Lines Must Not Be Attributed to the Witness

**Description:** Lines containing "Objection" must be attributed to an attorney or undefined counsel, never to `WITNESS`.

**Input:** Processed transcript

**Expected Output:** No line matching `WITNESS: Objection.`

**Failure Condition:** Line `"WITNESS: Objection. You may answer."` is found. Test fails.

---

### TC-09: Exhibit Names Must Be Consistent Across Files

**Description:** All exhibit references (e.g., "Cannon Farms 1", "Cannon Farms 2") must use the same spelling in the raw transcript, processed transcript, and metadata summary.

**Input:** All three files

**Expected Output:** One canonical exhibit name per exhibit number, no variant spellings

**Failure Condition:** "Cannon Farms", "Cancun Farms", and "Cancun Forms" all appear as variants. Test reports 3 distinct spellings for the same party/exhibit name.


---

## Task 3 - API Testing Plan 

### Endpoints Under Test

```
POST /transcripts/process
GET  /transcripts/:id
```

---

### POST /transcripts/process

This endpoint presumably accepts raw ASR output and metadata, triggers processing, and returns a job or transcript record.

#### Positive Tests

| # | Test | Input | Expected |
|---|------|-------|----------|
| P1 | Valid submission | Valid JSON with utterances + metadata | `201 Created` with `id` and `status: "PROCESSING"` or `"TRANSCRIBED"` |
| P2 | Response schema | Valid submission | Response contains `id` (string/UUID), `status`, `created_at` (valid ISO 8601), `speaker_count` |
| P3 | Idempotency key | Same payload submitted twice with same `idempotency_key` | Second call returns `200` with same record, not a new one |

#### Validation / Edge Case Tests

| # | Test | Input | Expected |
|---|------|-------|----------|
| V1 | Missing required fields | Omit `utterances` | `400 Bad Request` with field level error message |
| V2 | Empty utterances array | `utterances: []` | `400` - "utterances must not be empty" |
| V3 | Invalid `created_at` | `"created_at": "2025-10-09T26:61:00Z"` | `400` with "invalid datetime format" |
| V4 | Word timestamp exceeds audio duration | Word at 635040ms, `audio_duration_ms: 630000` | `400` or `422 Unprocessable Entity` with timestamp violation detail |
| V5 | Out of order word timestamps | Words with decreasing `start` values | `400` or warning flag in response |
| V6 | Unknown speaker in utterances | Utterance speaker not in metadata speakers | `400` or processed with `unresolved_speakers` flag |
| V7 | Extremely large payload | 10,000+ utterances | Should accept or return `413 Payload Too Large` with documented limit |
| V8 | Malformed JSON | Non-JSON body | `400` with "invalid JSON" |
| V9 | Wrong Content-Type | `text/plain` instead of `application/json` | `415 Unsupported Media Type` |
| V10 | Duplicate submission (no idempotency key) | Same payload twice | `201` both times with different `id`s (or documented behavior) |

#### Security Tests

| # | Test | Input | Expected |
|---|------|-------|----------|
| S1 | Missing auth | No `Authorization` header | `401 Unauthorized` |
| S2 | Invalid token | `Authorization: Bearer invalid` | `401` |
| S3 | Oversized field | `full_name` with 10,000 characters | `400` or graceful truncation |

---

### GET /transcripts/:id

#### Positive Tests

| # | Test | Input | Expected |
|---|------|-------|----------|
| P1 | Fetch existing transcript | Valid `id` from POST response | `200 OK` with full transcript payload |
| P2 | Response schema | Any valid `id` | Response contains `id`, `status`, `utterances`, `metadata`, `created_at` |
| P3 | Speaker normalization | Transcript with variant labels | All speaker labels normalized (e.g., `"MS. SELIGMAN"` consistently) |

#### Edge Cases

| # | Test | Input | Expected |
|---|------|-------|----------|
| E1 | Non existent ID | `GET /transcripts/does-not-exist` | `404 Not Found` |
| E2 | Invalid ID format | `GET /transcripts/!!!` | `400 Bad Request` |
| E3 | Processing in-flight | ID of a transcript still being processed | `200` with `status: "PROCESSING"` and partial or no `utterances` |
| E4 | Soft deleted transcript | ID of deleted record | `404` or `410 Gone` - not a `500` |
| E5 | Concurrent access | 100 simultaneous GET requests for same ID | All return `200` consistently (no race conditions) |

#### Validation Rules
**Transcript**
- Cannot be empty
- Must contain speaker labels
- Must contain valid timestamps


**Metadata**
- ISO 8601 timestamps only
- Known speaker roles only
- Duration > 0


**IDs**
- Unique
- UUID format
---

## Task 4 - Workflow Testing 

### Workflow Under Test

```
NEW -> ASSIGNED -> TRANSCRIBED -> REVIEWED -> COMPLETED
```

---

### Valid Transition Tests

| # | From | To | Expected |
|---|------|----|----------|
| WT-01 | `NEW` | `ASSIGNED` | `200 OK`; `status` becomes `ASSIGNED`; `assigned_at` timestamp set |
| WT-02 | `ASSIGNED` | `TRANSCRIBED` | `200 OK`; `transcript_content` populated; `transcribed_at` set |
| WT-03 | `TRANSCRIBED` | `REVIEWED` | `200 OK`; reviewer ID and `reviewed_at` set |
| WT-04 | `REVIEWED` | `COMPLETED` | `200 OK`; `completed_at` set; record becomes read-only |
| WT-05 | Full happy path | `NEW -> ASSIGNED -> TRANSCRIBED -> REVIEWED -> COMPLETED` | All transitions succeed in sequence |

---

### Invalid Transition Tests

| # | From | Attempted To | Expected |
|---|------|--------------|----------|
| IT-01 | `NEW` | `TRANSCRIBED` | `409 Conflict` - skipping `ASSIGNED` not allowed |
| IT-02 | `NEW` | `COMPLETED` | `409 Conflict` - cannot jump to terminal state |
| IT-03 | `ASSIGNED` | `COMPLETED` | `409 Conflict` - must go through `TRANSCRIBED` and `REVIEWED` |
| IT-04 | `COMPLETED` | `REVIEWED` | `409 Conflict` - cannot go backward from terminal state |
| IT-05 | `REVIEWED` | `ASSIGNED` | `409 Conflict` - backward transition not allowed |
| IT-06 | `TRANSCRIBED` | `NEW` | `409 Conflict` - backward transition not allowed |
| IT-07 | Any state | Invalid state (e.g., `"CANCELLED"`) | `400 Bad Request` - unknown state value |

---

### Edge Case Tests

| # | Scenario | Expected |
|---|----------|----------|
| EC-01 | **Reassignment** - move from `ASSIGNED` back to `NEW` for reassignment | `200 OK` if reassignment is a supported operation; `409` if not; must be documented |
| EC-02 | **Re-review** - move from `REVIEWED` back to `TRANSCRIBED` for corrections | Depends on policy; if allowed, old `reviewed_at` cleared; audit log entry created |
| EC-03 | **Concurrent transition** - two users simultaneously try `TRANSCRIBED -> REVIEWED` | Only one succeeds (`200`); the other gets `409` - optimistic locking or mutex required |
| EC-04 | **Missing required data on transition** - attempt `ASSIGNED -> TRANSCRIBED` without transcript content | `422 Unprocessable Entity` - content is required before marking as transcribed |
| EC-05 | **Skipped step detected post-hoc** - audit log shows gap between timestamps | Automated check flags records where `reviewed_at < transcribed_at` (impossible in a valid flow) |
| EC-06 | **Unassigned transition** - attempt `ASSIGNED -> TRANSCRIBED` without an assigned user | `422` - assignee ID required |
| EC-07 | **Stale state** - client has cached `status: "ASSIGNED"` but server has moved to `REVIEWED`; client tries `ASSIGNED -> TRANSCRIBED` | `409` - server-side state wins; client must refresh |

---

## Task 5 - Automation Approach 

### Tools

**API Testing**
Postman
Newman
RestAssured (Java)
Pytest + Requests
**UI Testing**
Playwright
Selenium
**Contract Testing**
JSON Schema Validation
Pact
**CI/CD**
GitHub Actions
GitLab CI
Jenkins

---

### Example Test Structure

```python
# tests/test_transcript_validation.py

import pytest
from datetime import datetime
from models import Transcript, Speaker, Utterance  # Pydantic models

#  Fixtures 

@pytest.fixture
def valid_transcript():
    return {
        "utterances": [
            {"speaker": "THE REPORTER", "start": 2400, "isQ": False, "isA": False,
             "words": [{"start": 2400, "text": "Good"}, {"start": 2480, "text": "morning."}]}
        ],
        "metadata": {
            "audio_duration_ms": 630000,
            "created_at": "2025-10-09T10:00:00Z",
            "speakers": [{"real_time_asr_label": "THE REPORTER", "role": "COURT REPORTER"}]
        }
    }

@pytest.fixture
def api_client():
    import httpx
    return httpx.Client(base_url="https://api.example.com", headers={"Authorization": "Bearer test-token"})

# Timestamp Tests 

class TestTimestamps:

    def test_utterance_start_lte_first_word(self, valid_transcript):
        for u in valid_transcript["utterances"]:
            assert u["start"] <= u["words"][0]["start"]

    def test_words_monotonically_increasing(self, valid_transcript):
        for u in valid_transcript["utterances"]:
            starts = [w["start"] for w in u["words"]]
            assert starts == sorted(starts), "Word timestamps not monotonically increasing"

    def test_no_word_exceeds_audio_duration(self, valid_transcript):
        max_allowed = valid_transcript["metadata"]["audio_duration_ms"]
        for u in valid_transcript["utterances"]:
            for w in u["words"]:
                assert w["start"] <= max_allowed

    def test_invalid_created_at_rejected(self, api_client, valid_transcript):
        payload = valid_transcript.copy()
        payload["metadata"]["created_at"] = "2025-10-09T26:61:00Z"
        r = api_client.post("/transcripts/process", json=payload)
        assert r.status_code == 400
        assert "created_at" in r.json()["errors"]

#  Speaker Tests 

class TestSpeakers:

    def test_all_utterance_speakers_in_metadata(self, valid_transcript):
        known = {s["real_time_asr_label"] for s in valid_transcript["metadata"]["speakers"]}
        for u in valid_transcript["utterances"]:
            assert u["speaker"] in known

    def test_first_name_matches_full_name(self, valid_transcript):
        for s in valid_transcript["metadata"]["speakers"]:
            if s.get("first_name") and s.get("full_name"):
                expected = s["full_name"].split()[0].lower()
                assert s["first_name"].lower() == expected

# Workflow Tests 

class TestWorkflow:

    @pytest.mark.parametrize("from_state,to_state,expected_code", [
        ("NEW", "ASSIGNED", 200),
        ("ASSIGNED", "TRANSCRIBED", 200),
        ("TRANSCRIBED", "REVIEWED", 200),
        ("REVIEWED", "COMPLETED", 200),
        ("NEW", "COMPLETED", 409),    # skip
        ("COMPLETED", "REVIEWED", 409),  # backward
        ("TRANSCRIBED", "ASSIGNED", 409),  # backward
    ])
    def test_state_transitions(self, api_client, from_state, to_state, expected_code):
        # Create transcript in from_state (via setup fixture)
        transcript_id = self._create_in_state(api_client, from_state)
        r = api_client.patch(f"/transcripts/{transcript_id}/status", json={"status": to_state})
        assert r.status_code == expected_code

    def _create_in_state(self, client, state):
        # Helper to set up transcript at desired state
        ...
```

---

### Continuous Integration (CI/CD) Pipeline Strategy

Triggers: Configure pipeline workflows (via GitHub Actions or GitLab CI) to execute on every pull_request targeting the main or develop branches.

Developer Push
      ↓
GitHub Action
      ↓
Run Unit Tests
      ↓
Run API Tests
      ↓
Run Workflow Tests
      ↓
Generate Report
      ↓
Deploy


### Ongoing Quality Gates

- **Pre-merge:** All unit and integration tests must pass; no new failures allowed
- **Nightly:** Full regression suite including performance tests
- **On deploy to staging:** Smoke tests against live API (10 key cases only, fast)
- **On deploy to production:** Read only health checks (no mutation) 

---

## Bonus — Transcript Quality Scoring {#bonus}

### Proposed 0–100 Scoring System

Each dimension is scored independently and combined into a weighted total.

| Dimension | Weight | What's Checked |
|-----------|--------|----------------|
| Timestamp Integrity | 25% | No out-of-order words; no word exceeds audio duration; utterance start <= first word start; no duplicate timestamps |
| Speaker Attribution | 25% | All speakers in metadata; no swapped roles; `first_name` matches `full_name`; no undocumented speakers |
| Content Consistency | 20% | Exhibit names consistent; party names consistent; no duplicate content blocks |
| Diarization Quality | 15% | No single utterance >30% of all words; `isQ`/`isA` flags populated |
| Metadata Validity | 15% | `created_at` valid ISO 8601; `audio_duration_ms` > 0; `transcript_duration_ms` <= `audio_duration_ms` |

**Per-dimension scoring (0–100, then weighted):**
- 100: No issues detected
- 75: 1 minor issue (e.g., one duplicate timestamp)
- 50: Multiple issues of the same type
- 25: Critical issue present (e.g., role swap, invalid datetime)
- 0: Dimension is entirely broken (e.g., 1 utterance for entire transcript)

**Example — Moonlight Plaza transcript score:**

| Dimension | Score | Reason |
|-----------|-------|--------|
| Timestamp Integrity | 10 | Word exceeds audio, out-of-order words, utterance/word mismatch, 26 duplicate timestamps |
| Speaker Attribution | 20 | Role swap, first_name mismatch, undocumented MS. ATRIANO, label inconsistencies |
| Content Consistency | 35 | "Cannon/Cancun Farms" inconsistency, "Richmond/Richman" mismatch, duplicate content block |
| Diarization Quality | 5 | Only 2 utterances, all `isQ`/`isA` false, one utterance = 98% of words |
| Metadata Validity | 15 | `created_at` invalid, `transcript_duration_ms` < `audio_duration_ms` |

**Weighted total:**
```
(10×0.25) + (20×0.25) + (35×0.20) + (5×0.15) + (15×0.15)
= 2.5 + 5.0 + 7.0 + 0.75 + 2.25
= 17.5 / 100
```
This transcript would score **~18/100** — flagged for mandatory human review before delivery.

---

*End of QA Engineer Assessment Response*

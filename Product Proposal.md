# Product Proposal: Conversational Local Food & Service Discovery

## 1. Product Overview

**What it is:** A local food and service discovery app that goes beyond search-and-filter. Users can either browse a curated list, type a quick search, or chat with an assistant that asks clarifying questions and recommends one specific place that fits their needs — with a stated reason, not just a ranked list.

**Core differentiator:** Most discovery apps hand the user a list and expect them to decide. This product commits to a pick and explains *why* — the assistant does the narrowing-down work a friend would do when asked "where should we eat?"

**Target users:** People who want a quick decision (not a browsing session) — e.g. "somewhere quiet, vegetarian, near District 1, for six people" — as well as users who prefer to scan a list themselves.

---

## 2. Core Functions

| # | Function | Description |
|---|----------|-------------|
| F1 | Browse list | Default landing page shows a list of nearby/popular food stores, no query required |
| F2 | Quick search | Search bar accepts free-text queries and returns filtered/matched results |
| F3 | Guided chat | "Chat with us" opens an assistant that asks clarifying questions when the query is too vague, then narrows to a recommendation |
| F4 | Recommendation with reason | The assistant surfaces one best-match result with a short natural-language justification tied to actual matched attributes |
| F5 | Alternatives | User can request another option without restarting the conversation |
| F6 | Editable filter chips | Parsed intent (location, cuisine, dietary, party size, price) is shown as editable chips so users can correct misunderstandings |
| F7 | Feedback capture | Lightweight accept/reject/correction signals on recommendations and parsed filters |

---

## 3. Functional Requirements

### FR1 — Data ingestion & indexing
- FR1.1 Store restaurant records: name, coordinates, price tier, cuisine, dietary tags, seating attributes
- FR1.2 Store dish records linked to restaurants: name, category, dietary tags
- FR1.3 Store review records linked to restaurants: raw text, rating (indexed only, not summarized in Phase 1)
- FR1.4 Maintain a geospatial index on restaurant coordinates

### FR2 — Query understanding
- FR2.1 Parse free-text input into a structured filter object (location, cuisine, party size, price, dietary, seating)
- FR2.2 Extract location mentions and resolve them to coordinates
- FR2.3 Distinguish dish-level intent (e.g. "phở") from cuisine-level intent (e.g. "Vietnamese food")
- FR2.4 Detect low-confidence or incomplete parses and trigger a clarifying question instead of searching immediately
- FR2.5 Log raw query text alongside parsed filter JSON for evaluation

### FR3 — Retrieval
- FR3.1 Geospatial search within a radius/area of the resolved location
- FR3.2 Apply hard structured filters (price, cuisine, dietary, category) to candidates
- FR3.3 Support dish-index lookups that surface restaurants via matching dishes, not just cuisine tags
- FR3.4 Return a bounded, ordered candidate set (not an unranked full scan)

### FR4 — Recommendation & output
- FR4.1 Select a top pick via a deterministic placeholder score (distance + filter-match count) — explicitly not the Phase 2 ranking/personalization system
- FR4.2 Generate a short natural-language reason for the pick, referencing only matched structured attributes
- FR4.3 Provide "show another option" to cycle to the next candidate without re-parsing the query
- FR4.4 Display editable filter chips reflecting parsed intent
- FR4.5 Handle zero-result cases by explaining what was relaxed (e.g. widened radius, dropped a soft filter), never a dead end

### FR5 — Feedback & logging
- FR5.1 Capture accept/reject/"show another" signal on the recommended pick
- FR5.2 Log filter-chip edits as labeled parsing corrections
- FR5.3 Track click-through on shown results
- FR5.4 Log per-stage latency (parse, retrieve) for internal benchmarking

### FR6 — Browse & search entry points
- FR6.1 Main page shows a default list (e.g. "popular near you") with no query required
- FR6.2 Search bar accepts free text and routes into the same parse → retrieve pipeline as chat, skipping the clarifying-question step (assumes sufficient signal)
- FR6.3 "Chat with us" opens a guided conversation entry point using the same underlying pipeline

---

## 4. Non-Functional Notes (Phase 1)
- Default result ordering (when not chat-driven) is distance-ascending, with rating as a tiebreaker — chosen explicitly, not left implicit.
- No personalization, review summarization, or diversity-aware ranking in Phase 1 — these are deferred, and the placeholder scoring (FR4.1) must be clearly documented as temporary so it isn't mistaken for the Phase 2 ranking system.
- Location input requires explicit user consent if using device GPS.

---

## 5. User Flow

### 5.1 Browse flow (F1)
1. User opens the app → main page loads with a default list (e.g. nearest/popular stores), no input required.
2. User taps a card → sees store detail (address, dishes, matched attributes if applicable).

### 5.2 Quick search flow (F2)
1. User types a free-text query into the top search bar and submits.
2. Query understanding (FR2.1–2.3) parses it into a structured filter object.
3. If confidence is sufficient (FR2.4), skip straight to retrieval.
4. Retrieval (FR3) returns a bounded candidate set.
5. Recommendation layer (FR4) selects a top pick and composes a reason.
6. Result renders as: filter chips (editable) + recommendation card ("best match" + reason) + "see another option."
7. User can: accept, tap "see another option" (F5), edit a filter chip (F6), or open store detail.

### 5.3 Guided chat flow (F3)
1. User taps "Chat with us" → chat overlay opens.
2. Assistant asks an opening question if the conversation has no prior context (e.g. "What are you in the mood for, and roughly where?").
3. User replies in natural language.
4. Query understanding parses the reply into filters, shown as chips inside the chat overlay.
5. If filters are still insufficient (FR2.4), assistant asks one more clarifying question.
6. Once sufficient, the pipeline runs the same retrieval → recommendation steps as the search flow (5.2, steps 2–5).
7. Recommendation renders inside the chat as an assistant message: best match + reason + "see another option."
8. User can reply again to refine (e.g. "something cheaper, keep outdoor seating") — this patches the existing filter object rather than restarting the parse from scratch.
9. Session state (parsed filters) persists for the conversation; closing the overlay does not necessarily discard it (open decision — see Section 7).

### 5.4 Feedback loop (cross-cutting)
- Every recommendation shown (from either flow) logs: shown candidate(s), user action (accept / next / correct), and any filter-chip edits — feeding FR5.

---

## 6. Phase 1 Scope Boundary (for reference)
Phase 1 is scoped to three foundational pieces, per team decision:
1. Dish-, restaurant-, and review-level indexes (FR1)
2. Query-to-location and query-to-attribute understanding (FR2)
3. Geospatial search & structured filters (FR3)

Everything in FR4–FR6 is the minimum UI/output layer needed to make Phase 1 demonstrable end-to-end (chat message → correct result), using deterministic placeholders where a real ranking/personalization system would eventually go.

---

## 7. Open Decisions
- Does the chat overlay hand off its recommendation to the main page, or render results inline and stay open?
- GPS-first or manual-location-first as the default location input?
- Does closing the chat overlay preserve session state for a follow-up later, or reset it?

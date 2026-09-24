# Deck events — v1

Every change to a deck is an event in `_events`. The tables are a projection of the log.

## Envelope

```json
{
  "id": "0192…",          // UUIDv7 op id, dedupe key
  "seq": 1042,            // assigned by server; null while pending
  "hlc": "1727175180000-00003-c7f2…",
  "client_id": "c7f2…",
  "user_id": "u_91…",
  "type": "fact.value_set",
  "payload": { … }
}
```

## Apply rules

1. **Deterministic.** Apply never reads the clock, generates randomness, or makes IDs. Everything it needs is in the payload. The one exception is cards, whose IDs are UUIDv5 derived from `(fact_id, template_id, ordinal)`.
2. **Idempotent.** An event whose `id` is already in `_events` is a no-op.
3. **LWW.** Every property write checks `_clocks`. If the incoming HLC isn't greater than the stored one, the write is skipped. The event is still logged.
4. **Delete wins.** Any event targeting a tombstoned entity is a no-op, whatever its HLC.
5. **Convergence.** Rules 3–4 make property edits commute. A client can apply pulled events on top of its own pending ones without rolling anything back. Creation events arrive causally ordered, because each client's outbox is pushed in order.
6. **Server validation before sequencing.** The server rejects:
   - unknown entity IDs
   - bad data types
   - identifier collisions
   - oversized payloads
   - HLCs more than ~5 min ahead of server time

## Deck

- **`deck.created`** `{deck_id, name, created_at}`
  - First event in every log.
  - Inserts the `deck` row and the `_about` rows.
- **`deck.property_set`** `{property, value}`
  - LWW.
  - `property` ∈ `name`, `description`, `settings.<key>`.

## Fact types

- **`fact_type.created`** `{fact_type_id, name, table_name}`
  - Inserts into `fact_types`.
  - `CREATE TABLE <table_name> (id TEXT PRIMARY KEY)`.
- **`fact_type.renamed`** `{fact_type_id, name, table_name}`
  - LWW on `name`.
  - `ALTER TABLE … RENAME TO`.
- **`fact_type.style_set`** `{fact_type_id, css}`
  - LWW.
- **`fact_type.deleted`** `{fact_type_id}`
  - Tombstones the fact type, its fields, templates, facts, and cards.
  - `DROP TABLE`.

## Fields

- **`field.added`** `{field_id, fact_type_id, name, column_name, data_type, position}`
  - `ALTER TABLE … ADD COLUMN`.
- **`field.renamed`** `{field_id, name, column_name}`
  - LWW.
  - `RENAME COLUMN`.
  - Rewrites `{{Old Name}}` in that type's templates.
- **`field.moved`** `{field_id, position}`
  - LWW.
  - `position` is a fractional index.
- **`field.type_changed`** `{field_id, data_type}`
  - LWW.
  - Rebuilds the table, converting values. Unconvertible values become NULL.
- **`field.removed`** `{field_id}`
  - Tombstones the field.
  - `DROP COLUMN`.
  - Later `fact.value_set` events for this field are no-ops.

## Templates

- **`template.added`** `{template_id, fact_type_id, name, generation, front, back, position}`
  - Inserts the template.
  - Generates cards for every live fact of the type.
  - `generation` is immutable after creation.
- **`template.property_set`** `{template_id, property, value}`
  - LWW.
  - `property` ∈ `name`, `front`, `back`, `position`.
- **`template.removed`** `{template_id}`
  - Tombstones the template and its cards.
  - Review history stays in `reviews`.

## Facts

- **`fact.created`** `{fact_id, fact_type_id, values: {field_id: value}}`
  - Inserts into `facts` and the type table.
  - Sets initial clocks to the event's HLC.
  - Generates cards.
- **`fact.value_set`** `{fact_id, field_id, value}`
  - LWW per field.
  - If a `per_cloze` template reads this field, cards are regenerated:
    - new ordinals are added
    - vanished ordinals are tombstoned
    - returning ordinals are un-tombstoned, with their history intact
- **`fact.deleted`** `{fact_id}`
  - Tombstones the fact in `facts`.
  - Deletes its row from the type table.
  - Tombstones its cards.

## Cards

Cards are never created by an event (see apply rule 1).

- **`card.suspended_set`** `{card_id, suspended}`
  - LWW.
- **`card.reset`** `{review_id, card_id, at}`
  - Appends a `reviews` row with `kind = 'reset'`.
  - Rebuilds `card_state` for the card.

## Reviews

- **`review.recorded`** `{review_id, card_id, grade, reviewed_at, duration_ms}`
  - Appends to `reviews`.
  - Recomputes `card_state` for that card and user by replay.
- **`review.voided`** `{review_id}`
  - Undo.
  - Sets `voided = 1` and recomputes `card_state`.

## Media

- **`media.added`** `{hash, mime_type}`
  - Inserts a `media` row with `data = NULL`.
  - Bytes are uploaded and downloaded out of band.
  - The receiver verifies the sha256 before storing.

## Open questions

- **Identifier collisions.** Two offline clients both create a "Country" type, and both claim `fact_country`. Proposal: the server rejects the later event. The client renames to `fact_country_2` and resubmits. The same applies to `column_name`.
- **Log compaction.** When to snapshot and prune `_events`, and the resync rule for clients whose cursor is older than the horizon.
- **Field type conversion rules** for `field.type_changed`.
- **Scheduler.** FSRS is the likely choice. Its version is pinned in `card_state.scheduler`, so history can be replayed under a newer version.

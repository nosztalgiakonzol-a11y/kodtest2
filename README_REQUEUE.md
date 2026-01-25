# Arbify Beta - Requeue System Quick Reference

## Magyar (Hungarian)

### Gyakori Kérdések

**1. Egy elem szerepelhet többször a task listán?**
- **Védett:** Általában nem, mert ellenőrzések védik (seen, NAV backoff, higher_ids)
- **Szándékosan igen:** Feldolgozási hiba esetén max 2x újrapróbálkozás

**2. 20mp timeout után azonnal visszakerül a listába?**
- **NEM!** Exponenciális backoff rendszerbe kerül:
  - 1. timeout: 20s várakozás
  - 2. timeout: 40s várakozás  
  - 3. timeout: 80s várakozás
  - Maximum: 300s (5 perc)

**3. Részletes dokumentáció:**
Lásd: [REQUEUE_SYSTEM_DOCUMENTATION.md](./REQUEUE_SYSTEM_DOCUMENTATION.md)

---

## English

### Frequently Asked Questions

**1. Can an element appear multiple times in the task list?**
- **Protected:** Usually no, because checks prevent it (seen set, NAV backoff, higher_ids)
- **Intentionally yes:** Up to 2 retries on processing errors

**2. Does it immediately return to the list after 20s timeout?**
- **NO!** It enters an exponential backoff system:
  - 1st timeout: 20s wait
  - 2nd timeout: 40s wait
  - 3rd timeout: 80s wait
  - Maximum: 300s (5 minutes)

**3. Detailed documentation:**
See: [REQUEUE_SYSTEM_DOCUMENTATION.md](./REQUEUE_SYSTEM_DOCUMENTATION.md)

---

## Code Comments Added

Enhanced inline documentation has been added to the following key functions:

1. `enqueue_open_task()` - Line ~1717
2. `_schedule_nav_backoff()` - Line ~1984  
3. `background_nav_worker()` - Line ~2530
4. `batch_save_new_ids()` - Line ~2664
5. `resolve_pairs_round_robin()` timeout section - Line ~2165

These comments explain:
- When and why elements can appear multiple times
- How the NAV backoff system prevents immediate re-queuing
- The exponential backoff mechanism
- Protection against duplicates

---

## Key Configuration

| Variable | Value | Description |
|----------|-------|-------------|
| `PAIR_TIMEOUT_SEC` | 20 | Link resolution timeout (seconds) |
| `OPEN_TASKS_MAX` | 5000 | Maximum OPEN_TASKS queue size |
| `NAV_WORKER_MAX_PAIRS` | 11 | Concurrent pairs processed |
| `NAV_RETRY_BASE` | 20 | Base retry delay (seconds) |
| `NAV_RETRY_MAX` | 300 | Maximum retry delay (seconds = 5 minutes) |

For more details, see the full documentation.

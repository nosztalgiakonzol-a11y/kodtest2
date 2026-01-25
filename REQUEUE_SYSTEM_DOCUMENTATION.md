# Requeue és Task Lista Rendszer Dokumentáció

## Áttekintés

Ez a dokumentáció részletesen elmagyarázza, hogyan működik a requeue (újrapróbálkozás) és a task lista rendszer az Arbify Beta alkalmazásban.

## A Rendszer Főbb Komponensei

### 1. OPEN_TASKS (Nyitott Feladatok Listája)
- **Típus**: `deque` (kétirányú sor)
- **Maximum méret**: 5000 elem
- **Helye a kódban**: `Arbify Beta.py` sor 1714-1715
- **Cél**: A NAV-on keresztül feloldandó árjegyzői linkeket tárolja

```python
OPEN_TASKS = deque()
OPEN_TASKS_MAX = 5000
```

### 2. GROUP_NEXT_OPEN_QUEUE (Csoport és Következő Oldalak Megnyitási Sorja)
- **Típus**: `Queue`
- **Maximum méret**: 2000 elem
- **Helye a kódban**: `Arbify Beta.py` sor 3086
- **Cél**: GROUP és NEXT tabok aszinkron megnyitását kezeli

```python
GROUP_NEXT_OPEN_QUEUE = Queue(maxsize=2000)
```

## Kérdések és Válaszok

### ❓ 1. Egy elem szerepelhet többször a task listán?

**IGEN**, de korlátozásokkal:

#### OPEN_TASKS esetében:
- **Technikailag lehetséges**: A `OPEN_TASKS` deque nem ellenőrzi, hogy egy ID már benne van-e
- **Gyakorlatban védett**: A `batch_save_new_ids()` függvény (sor 2627) ellenőrzi:
  - `seen` halmazban már szerepel-e (sor 2647)
  - NAV backoff alatt van-e (sor 2652-2655)
  - `higher_ids` halmazban van-e (sor 2650)
  
  ```python
  for tid in new_ids:
      if tid in seen:
          continue
      until = nav_retry_until.get(tid, 0)
      if until and now < until:
          continue
  ```

#### Újrapróbálkozás esetén:
- **IGEN, szándékosan többször szerepel**: Ha egy task feldolgozása hibába ütközik, visszakerül a sor végére
- **Maximum 2 próbálkozás**: A `_retry_count` számlálóval követi (sor 2612-2618)

```python
retry_count = task.get("_retry_count", 0)
if retry_count < 2:
    task["_retry_count"] = retry_count + 1
    OPEN_TASKS.append(task)  # Visszarakjuk a sorba
    warn(f"⚠️ Task feldolgozás hiba, újrapróbálás ({retry_count+1}/2)")
else:
    warn(f"❌ Task végleg elvetve 2 sikertelen próbálkozás után")
```

### ❓ 2. Ha egy tbodys linket nem tudtunk megnyitni (20 mp timeout), egyből visszakerül a Task listába?

**NEM, nem kerül azonnal vissza!** Ehelyett **NAV backoff** rendszerbe kerül:

#### Timeout Folyamat:

1. **20 másodperces timeout** (`PAIR_TIMEOUT_SEC = 20`, sor 177):
   ```python
   PAIR_TIMEOUT_SEC = FIX_URL_WAIT_SEC  # most 20 mp
   deadline = time.time() + PAIR_TIMEOUT_SEC
   ```

2. **Timeout után** a párok, amik nem oldódtak fel, `("timeout", "timeout")` státuszt kapnak (sor 2547, 2561):
   ```python
   states = [("timeout", "timeout")] * len(pairs)
   ```

3. **NAV Backoff aktiválódik** (sor 2604):
   ```python
   _schedule_nav_backoff(tbody_id)
   ```

#### NAV Backoff Rendszer:

A `_schedule_nav_backoff()` függvény (sor 1984-1996):
- **Exponenciális várakozás**: Minden próbálkozásnál kétszer hosszabb várakozási idő
- **Első próbálkozás**: `NAV_RETRY_BASE` másodperc (alapértelmezés szerint ~10s)
- **Második próbálkozás**: ~20 másodperc
- **Harmadik próbálkozás**: ~40 másodperc
- **Maximum várakozás**: `NAV_RETRY_MAX` másodperc

```python
def _schedule_nav_backoff(tid: str):
    att = nav_retry_attempts.get(tid, 0) + 1
    nav_retry_attempts[tid] = att
    delay = min(NAV_RETRY_BASE * (2 ** (att - 1)), NAV_RETRY_MAX)
    nav_retry_until[tid] = time.time() + delay
    warn(f"⏳ NAV backoff id={tid} {int(delay)}s (attempt={att})")
```

#### Fontos Védelem:

A `batch_save_new_ids()` függvény **kiszűri** a backoff alatt lévő ID-ket (sor 2652-2655):
```python
until = nav_retry_until.get(tid, 0)
if until and now < until:
    continue  # NEM kerül vissza azonnal a listába
```

**Tehát**: Timeout után az elem **NEM** kerül azonnal vissza, hanem **exponenciálisan növekvő várakozási idővel** próbálkozik újra.

### ❓ 3. Hogyan működik a rendszer összességében?

## Teljes Munkafolyamat

### 1. Fázis: Új ID Felfedezése
```
MAIN Tab Scan
    ↓
új tbody ID detektálva
    ↓
batch_save_new_ids(new_ids)
```

### 2. Fázis: Task Előkészítés
```python
# batch_save_new_ids() függvény
for tid in new_ids:
    # Ellenőrzések:
    if tid in seen:              # már láttuk?
        continue
    if tid in higher_ids:        # magasabb prioritású?
        continue
    if now < nav_retry_until[tid]:  # NAV backoff alatt?
        continue
    
    # Task létrehozás
    t = prepare_new_task_for_id(tid)
```

### 3. Fázis: Cache Ellenőrzés

#### 3A. Ha cache-ben VAN mindkét link:
```
link_cache[tid] → {"link1": url1, "link2": url2}
    ↓
azonnal SAVE (dispatcher.enqueue_save)
    ↓
_clear_nav_backoff(tid)
```

#### 3B. Ha cache-ben NINCS:
```
enqueue_open_task(t)
    ↓
OPEN_TASKS.append(task)
```

### 4. Fázis: NAV Worker Feldolgozás

A `background_nav_worker()` függvény folyamatosan dolgozza fel az OPEN_TASKS-ot:

```python
def background_nav_worker():
    while True:
        # 1) Kivesz max NAV_WORKER_MAX_PAIRS taskot
        todo = []
        while OPEN_TASKS and len(todo) < NAV_WORKER_MAX_PAIRS:
            todo.append(OPEN_TASKS.popleft())
        
        # 2) Párok feloldása NAV-on keresztül (round-robin)
        finals, states = resolve_pairs_round_robin(pairs)
        
        # 3) Eredmények feldolgozása
        for task in todo:
            if valid_external(f1) and valid_external(f2):
                # ✓ Sikeres → SAVE
                dispatcher.enqueue_save(...)
                _clear_nav_backoff(tbody_id)
            else:
                # ✗ Sikertelen → NAV backoff
                _schedule_nav_backoff(tbody_id)
```

### 5. Fázis: Round-Robin Link Feloldás

A `resolve_pairs_round_robin()` (sor 2047-2277):
1. **Tabok megnyitása** Chrome DevTools Protocol (CDP) segítségével
2. **20 másodperces polling**: URL változások figyelése
3. **Sikeres feloldás**: amikor mindkét URL elhagyja a `surebet.com` domaint
4. **Timeout**: ha 20 mp alatt nem sikerül → `("timeout", "timeout")`

```python
deadline = time.time() + PAIR_TIMEOUT_SEC  # 20 másodperc
while tracking and time.time() < deadline:
    # CDP polling: Target.getTargets
    # Ellenőrzi, hogy az URL-ek elhagyták-e a surebet.com-ot
    if valid_external(url):
        # ✓ Sikeres
        finals_by_pair[pair_idx] = (url1, url2)
        states_by_pair[pair_idx] = ("ok", "ok")
```

### 6. Fázis: Újrapróbálkozás Logika

#### A. Feldolgozási Hiba Esetén (sor 2606-2618)
```python
try:
    # Task feldolgozás...
except Exception as task_err:
    retry_count = task.get("_retry_count", 0)
    if retry_count < 2:
        task["_retry_count"] = retry_count + 1
        OPEN_TASKS.append(task)  # ← VISSZAKERÜL a sorba
    else:
        warn("❌ Task végleg elvetve")
```

#### B. Timeout/Invalid URL Esetén (sor 2590-2604)
```python
if not (valid_external(f1) and valid_external(f2)):
    # NAV backoff → exponenciális várakozás
    _schedule_nav_backoff(tbody_id)
    # A task NEM kerül vissza OPEN_TASKS-ba
    # Helyette: batch_save_new_ids() később újra megpróbálja
```

## Védelmek és Biztonsági Mechanizmusok

### 1. Duplikáció Védelem
- `seen` halmaz: globális ID követés
- NAV backoff ellenőrzés újbóli hozzáadás előtt

### 2. Túlcsordulás Védelem
```python
if len(OPEN_TASKS) < OPEN_TASKS_MAX:
    OPEN_TASKS.append(task)
else:
    OPEN_TASKS.popleft()  # Dobja a legrégebbit
    OPEN_TASKS.append(task)
```

### 3. Végtelen Újrapróbálkozás Védelem
- Maximum 2 újrapróbálkozás feldolgozási hibánál
- Exponenciális backoff timeout esetén
- `NAV_RETRY_MAX` plafon a várakozási időre

### 4. Bootstrap Védelem
```python
if in_bootstrap_phase():  # Első 50 másodperc
    return  # NEM indít új SAVE/NAV feloldást
```

## Státusz Diagramok

### Task Életciklus

```
┌─────────────────┐
│  Új ID detektált │
└────────┬─────────┘
         │
         ▼
    ┌─────────────┐
    │ Ellenőrzések │ ◄─── seen? higher_ids? NAV backoff?
    └────┬────────┘
         │
         ├─── Cache-ben VAN → [SAVE azonnal] → [Kész]
         │
         └─── Cache-ben NINCS
                  │
                  ▼
           ┌──────────────┐
           │ OPEN_TASKS   │
           │   sorba      │
           └──────┬───────┘
                  │
                  ▼
         ┌────────────────┐
         │ NAV Worker     │
         │ feldolgozza    │
         └────┬───────────┘
              │
              ├─── Sikeres (20mp alatt) → [SAVE] → [Kész]
              │
              └─── Timeout/Hiba
                       │
                       ├─── Feldolgozási hiba → [Retry max 2x] → [OPEN_TASKS]
                       │
                       └─── Timeout/Invalid URL → [NAV Backoff]
                                                        │
                                                        ▼
                                            [Exponenciális várakozás]
                                                        │
                                                        ▼
                                            [batch_save_new_ids újra megpróbálja]
```

### NAV Backoff Idővonal

```
Timeout #1  →  Várj ~10s   →  Újrapróbálkozás
                                     ↓
                                  Timeout #2  →  Várj ~20s   →  Újrapróbálkozás
                                                                      ↓
                                                                   Timeout #3  →  Várj ~40s  →  ...
```

## Kulcs Függvények Referencia

| Függvény | Fájl & Sor | Leírás |
|----------|-----------|---------|
| `enqueue_open_task()` | 1717 | Task hozzáadása OPEN_TASKS-hoz |
| `background_nav_worker()` | 2509 | OPEN_TASKS folyamatos feldolgozása |
| `resolve_pairs_round_robin()` | 2047 | Link párok feloldása NAV-on keresztül |
| `_schedule_nav_backoff()` | 1984 | NAV backoff beállítása exponenciális késleltetéssel |
| `_clear_nav_backoff()` | 1998 | NAV backoff törlése sikeres feldolgozás után |
| `batch_save_new_ids()` | 2627 | Új ID-k feldolgozása cache vagy OPEN_TASKS felé |

## Konfiguráció

| Változó | Érték | Leírás |
|---------|-------|---------|
| `OPEN_TASKS_MAX` | 5000 | OPEN_TASKS maximális méret |
| `GROUP_NEXT_OPEN_QUEUE.maxsize` | 2000 | GROUP/NEXT queue maximális méret |
| `PAIR_TIMEOUT_SEC` | 20 | Link feloldási timeout (másodperc) |
| `NAV_WORKER_MAX_PAIRS` | 6 | Egyidejűleg feldolgozott párok száma |
| `NAV_RETRY_BASE` | ~10 | Alapértelmezett újrapróbálkozási várakozás (mp) |
| `NAV_RETRY_MAX` | ? | Maximum újrapróbálkozási várakozás (mp) |

## Összefoglalás

1. **Többszöri szerepelés**: Védett ellen, de újrapróbálkozásnál szándékosan többször is szerepel (max 2x retry)

2. **Timeout kezelés**: NEM azonnal vissza a task listába, hanem exponenciális NAV backoff rendszerbe kerül

3. **Munkafolyamat**: 
   - Új ID → Ellenőrzések → Cache vagy OPEN_TASKS
   - NAV Worker feldolgozza páronként
   - 20 mp timeout, ha nem sikerül → NAV backoff
   - Exponenciális várakozás után újrapróbálkozás
   - Maximum 2 feldolgozási hiba után elvetés

Ez a rendszer biztosítja, hogy:
- ✅ Ne terhelje túl a rendszert azonnal újrapróbálkozással
- ✅ Adjon időt a külső oldalaknak válaszolni
- ✅ Védje a duplikációt és végtelen ciklusokat
- ✅ Folyamatosan próbálkozzon, de intelligensen

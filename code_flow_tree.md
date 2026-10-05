# Code Flow Tree — file → function → file, with provenance

A navigation map for the reception-counting path. Every hop names the **file**,
the **function**, and the **line**, so you can follow one frame from JPEG to
rendered count without guessing where to look next.

**Last updated: 2026-10-04.** §0–§5 describe the experiment harness (path C) and
are unchanged since 2026-09-20. §6.3, §6.4, §7.9–§7.21 and §9 cover the
production service path (path A) and were updated 2026-10-04 with line numbers
re-verified against the current working tree. When you change the adapter or the
ReID stack, update §6.3's line numbers and add a §7 entry — that section is the
change log, and each entry is written to answer "why is it like this?" for
someone who was not there.

**Provenance legend** — verified against git, not memory:

| Mark | Meaning |
|---|---|
| 🟢 **NEW** | Created in this work. Did not exist at the last commit. |
| 🟡 **MODIFIED** | Existed before; we changed it. `git diff` shows the change. |
| ⚪ **PRE-EXISTING** | Existed before and we did not touch it. |
| 🔴 **ABANDONED** | Pre-existing, was wired in, then deliberately removed from the path. |

**How provenance was determined.** The submodule's last commit (`2c010f8`)
contains exactly **17 files** — `config.py`, `processor.py`, `tracker.py`,
`fsm.py`, `data_models.py`, `utils/*`, `config/rois*.json`, `usage_example.py`.
Nothing under `src/reid/`, `src/pipeline/`, `src/analytics/`, `src/config/`,
`evaluation/`, `helper/` or `docs/` existed. **The entire reception-counting
path is new code.** In the parent repo, `probe_head_crop_injector.py`,
`branch_builder.py`, `registry.py` and `setup_assets.py` pre-existed;
`ds_adapter_reception_counter.py` is new.

---

## 0. Three entry points — know which one you are reading

This is the most common source of confusion in this repo. There are **three**
separate ways into the code, and they do not share a `main()`.

```
┌─ A ─ PRODUCTION SERVICE ────────────────────────────────────────────┐
│  DeepstreamService/src/main.py          🟡 MODIFIED (§7.21)         │
│    FastAPI + uvicorn, multi-camera, RTSP, Kafka, Prometheus         │
│    └─► PipelineBuilder.build()        src/core/builders/            │
│          └─► BranchBuilder.build()    branch_builder.py  🟡 MODIFIED│
│                └─► registry.py        🟡 MODIFIED (registered us)   │
│                      └─► ds_adapter_reception_counter.py   🟢 NEW   │
│                            └─► src/reid/  +  src/analytics/  🟢 NEW │
└─────────────────────────────────────────────────────────────────────┘

┌─ B ─ LEGACY STANDALONE ─────────────────────────────────────────────┐
│  people_count_checker/usage_example.py  ⚪ PRE-EXISTING (committed) │
│    └─► src/processor.py, src/tracker.py, src/fsm.py                 │
│          now moved to legacy_code/ — imports are BROKEN             │
│    ✗ NOT part of the reception-counting path                        │
└─────────────────────────────────────────────────────────────────────┘

┌─ C ─ EXPERIMENT HARNESS  ◄── what §1-§5 of this doc describes ──────┐
│  people_count_checker/src/pipeline/identity_agegender_preview.py    │
│                                                        🟢 NEW       │
│    standalone CLI, one video, frozen detections from CSV            │
│    └─► the 3 probes, src/reid/, evaluation/zones.py                 │
└─────────────────────────────────────────────────────────────────────┘
```

| | A — service | B — legacy | C — harness |
|---|---|---|---|
| Entry | `src/main.py` | `usage_example.py` | `identity_agegender_preview.py` |
| Provenance | ⚪ pre-existing | ⚪ pre-existing | 🟢 new |
| Input | live RTSP, many cameras | video file | JPEG frames + frozen CSV |
| Detection | live `nvinfer` PGIE | live | **injected from CSV** |
| Output | Kafka, Prometheus, REST | annotated video | annotated `.mp4` + printed counts |
| Reception counting | ✅ via the adapter | ❌ | ✅ directly |

**Why the harness exists.** Freezing detections to CSV means tracking and
identity experiments vary **one** variable. With a live detector in the loop
there would be two moving parts and no way to attribute a result. Every number
in `tracking_experiments.md` comes from path C.

**`src/main.py` changed only in its shutdown block (§7.21).** The reception usecase
itself plugs in underneath it through `registry.py` and `branch_builder.py`, which
is why the service picks it up without its startup path changing.

---

## 1. Top-level map — path C (the harness)

```
ENTRY ──► identity_agegender_preview.py::main()              🟢 NEW
            │
            ├─► build_pipeline()          builds the GStreamer graph
            ├─► load_detections()         reads the frozen detections CSV
            ├─► IdentityResolver.from_config()   ─► src/reid/       🟢 NEW
            ├─► load_zones()                     ─► evaluation/zones.py  🟢 NEW
            │
            └─► attaches 3 probes, then runs the GLib main loop
                  │
                  ├─ PROBE 1  make_injector()        on nvstreammux src
                  ├─ PROBE 2  make_identity_probe()  on nvtracker src
                  └─ PROBE 3  make_readback_probe()  on nvinferserver src
```

Everything in §2–§5 expands one of those three probes. Path A is in §6.

---

## 2. Startup sequence — `main()`

File: `src/pipeline/identity_agegender_preview.py` 🟢 **NEW**

| Step | Line | Call | Goes to |
|---|---|---|---|
| 1 | 768 | `main()` | entry point |
| 2 | ~800 | `load_zones(args.zones)` | `evaluation/zones.py::load_zones` 🟢 |
| 3 | ~811 | gallery wipe / **`--resume-gallery`** | 🟢 **NEW this session** |
| 4 | ~816 | `IdentityResolver.from_config({...})` | `src/reid/identity.py:110` 🟢 |
| 5 | ~820 | `load_detections(args.detections)` | same file, line 183 🟢 |
| 6 | ~839 | `build_pipeline(...)` | same file, line 280 🟢 |
| 7 | ~851 | `make_injector(det, start_frame)` | line 363 🟢 |
| 8 | ~852 | `make_identity_probe(...)` | line 412 🟢 |
| 9 | ~855 | `make_readback_probe(...)` | line 487 🟢 |
| 10 | ~869 | `on_msg()` bus watch | line 869 🟢 |

### `IdentityResolver.from_config` — builds the identity stack

`src/reid/identity.py:110` 🟢 **NEW**

```
from_config(cfg)
  ├─► RetentionPolicy.from_config()   src/reid/retention.py     🟢 NEW
  ├─► NumpyStore(path, dim, ...)      src/reid/numpy_store.py   🟢 NEW
  │     └─► store.load()              ◄── THE RESTART MECHANISM (line 131)
  ├─► Gallery(store, match_threshold=0.70, margin_threshold=0.10)
  │                                    src/reid/gallery.py:74   🟢 NEW
  └─► VotingIdentifier(gallery, min_observations=5, ...)
                                       src/reid/voting.py:110   🟢 NEW
```

`store.load()` on line 131 is the single line that makes identity survive a
pipeline restart. Without it, tracker ids restart at 0 against an empty
gallery and everyone in the zone is re-counted.

---

## 3. PROBE 1 — detection injector

`identity_agegender_preview.py::make_injector` line 363 🟢 **NEW**
Attached to: `nvstreammux` src pad.

```
make_injector(det, start_frame)            line 363
  └─► probe(pad, info, u)                  line 364   ← runs per frame
        ├─ frame_number = probe.counter    🟡 counter starts at start_frame
        │                                     (NEW this session, was hardcoded 1)
        ├─ dets = det.get(frame_number)
        └─ per detection:
             pyds.nvds_acquire_obj_meta_from_pool()
             obj.unique_component_id = PERSON_PGIE_UNIQUE_ID   # = 1
             obj.class_id            = PERSON_CLASS_ID         # = 1  ◄── CRITICAL
             pyds.nvds_add_obj_meta_to_frame()
```

> **`class_id = 1` is the value that broke the logo SGIE for an entire
> session** — its config filtered for class 0. See §7.

**Emits:** one `NvDsObjectMeta` per person, consumed by `nvtracker`.

---

## 4. PROBE 2 — identity resolution

`identity_agegender_preview.py::make_identity_probe` line 412 🟢 **NEW**
Attached to: `nvtracker` src pad.

```
make_identity_probe(resolver, t0, fps, stats, start_frame)   line 412
  └─► probe(pad, info, u)                                    line 419
        │
        ├─ walk obj_meta_list
        ├─ walk obj_user_meta_list for NVDS_TRACKER_OBJ_REID_META
        │     └─ reid.get_host_reid_vector()   ← the 256-d embedding
        │
        └─► resolver.observe(tracker_id, frame, box, conf, vec, now)
                                             src/reid/identity.py:169
```

### Inside `IdentityResolver.observe` — the identity call chain

`src/reid/identity.py:169` 🟢 **NEW**

```
observe(tracker_id, frame, box, confidence, embedding, now)
  │
  ├─► score_observation(x1,y1,x2,y2, confidence, cfg)     [line 178]
  │        └─ src/reid/quality.py::score_observation      🟢 NEW
  │           quality = height × edge × aspect × confidence
  │           (multiplied so one bad factor VETOES the observation)
  │
  ├─ if this track already settled:
  │     └─► self._verify(...)                             [line 188 → 233]
  │
  └─► self.voter.observe(tracker_id, Observation(...))    [line 206]
           └─ src/reid/voting.py:133
```

### Inside `VotingIdentifier.observe` — the vote

`src/reid/voting.py:133` 🟢 **NEW**

```
observe(tracker_id, obs, exclude)                      line 133
  ├─ acc = self._tracks.setdefault(...)   EvidenceAccumulator  line 80
  ├─ if acc.decided_as is not None: return  ◄── LOCKED, noise cannot reopen
  ├─ if obs.quality < 0.30: return PENDING
  ├─► acc.add(obs)                                     line 92
  │      └─ bounded at 30, drops WEAKEST not oldest
  ├─► self.gallery.store.search(obs.embedding, top_k=8)
  │      └─ src/reid/numpy_store.py::search             🟢 NEW
  │         sims = self._matrix @ vec   ← whole gallery, ONE matmul
  ├─► acc.cast(customer_id, score, quality)             line 100
  │      votes[cid] += max(0, similarity) * quality
  └─► self._evaluate(acc, obs)                          line 166
```

### Inside `_evaluate` — the four-outcome decision

`src/reid/voting.py:166` 🟢 **NEW**

```
_evaluate(acc, latest)
  ├─ n < 5 obs  or  weight < 1.5      ──► PENDING
  ├─ no votes                         ──► _register_new()       line 202
  ├─ best = best_raw / total_weight   (normalise)
  ├─ best < 0.70                      ──► _register_new()  → NEW customer
  ├─ (best − second) < 0.10           ──► UNKNOWN  (do not guess)
  └─ else                             ──► CONFIDENT
        ├─ acc.decided_as = best_id            ← LOCK
        ├─► store.touch(best_id, timestamp)
        └─► self._store_best(acc, best_id, now)          line 210
               ├─ keep only quality ≥ 0.60, best 3
               ├─► store.add(...)      numpy_store.py
               │      └─► _prune()     ◄── the time-spread pruning fix
               └─► store.save()        writes .npy + .json
```

**Returns up the chain:** `customer_id` (or `None` while pending) →
`IdentityResolver` → probe 2 → available to probe 3 via `resolver.resolve()`.

---

## 5. PROBE 3 — readback, classification, counting, drawing

`identity_agegender_preview.py::make_readback_probe` line 487 🟢 **NEW**
Attached to: `nvinferserver` (MiVolo) src pad — i.e. **after both SGIEs**.

This is the largest function; it runs two passes over the object list.

```
probe(pad, info, u)                                    line 518
 │
 ├── PASS 1 — collect logo hits
 │     walk obj_meta_list
 │     if unique_component_id == LOGO_SGIE_UNIQUE_ID (7) and parent is not None:
 │         logo_hit_tids.add(parent.object_id)     🟡 CHANGED: was
 │                                                    observe_logo(True) inline
 │         stats["logo_reads"] += 1
 │         hide the logo box from OSD
 │
 └── PASS 2 — per person
       │
       ├─ read classifier_meta_list  → gender, age   (MiVolo output)
       │
       ├─► a.observe(gender, age)              line 249  (15-frame vote)
       ├─► a.observe_logo(tid in logo_hit_tids)  line 259  🟡 REWRITTEN
       │      └─ TYPE_CONFIRM_FRAMES = 15 lock, BOTH directions
       │
       ├─► resolver.resolve(tid)               src/reid/identity.py:372
       ├─► point_in_zone((cx,cy), zone)        evaluation/zones.py:106  🟢
       │
       ├─ COUNTING BRANCH:
       │     if a.is_employee:
       │         employees_entered.add(tid)
       │         entered.discard(cust)   ◄── 🟢 THE DOUBLE-COUNT FIX
       │         served.discard(cust)
       │     elif cust is not None:
       │         entered.add(cust)
       │         if inside_count[tid] >= confirm_frames: served.add(cust)
       │
       ├─ LABEL:   "Person_3(T2) / CUST / M 45-54 / IN 15:03 / dwell:2s"
       │     └─► _age_range(age)                line 195
       │     └─ dwell keyed on customer_id      🟢 NEW (was tracker_id)
       │
       └─ PANEL:  RECEPTION / Customer entered / Served / Staff / clock
             └─ panel_w computed at 17.5 px/char   🟢 NEW (was hardcoded)
```

---

## 6. Path A — the production service, from `src/main.py`

Path C above is the experiment harness. This is the path that actually ships.
It shares only the `src/reid/` and `src/analytics/` layers with the harness.

### 6.1 Service startup

`DeepstreamService/src/main.py` ⚪ **PRE-EXISTING — not modified**

```
main.py                                                    246 lines
  ├─ FastAPI app + CORS + /metrics (Prometheus)
  ├─► preflight()              src/core/common/startup_validator.py  ⚪
  ├─► load_all_cameras()       src/services/camera_loader.py         ⚪
  ├─► RTSPManager(...)                                     [line ~65]
  ├─► PipelineBuilder.build(...)                           [line 74]
  │        └─ src/core/builders/pipeline_builder.py        ⚪
  │             └─► BranchBuilder.build(pipeline, usecase_id)
  │                    src/core/builders/branch_builder.py  🟡 MODIFIED
  ├─► KafkaManager.initialize(common_config.kafka)         [line ~87]
  ├─► ProbeManager.add_drawing_probes(pipeline_state)      [line 94]
  │        └─ src/core/managers/probe_manager.py           ⚪
  ├─  GLib main loop  +  bus_call()
  ├─► PipelineBuilder.cleanup(...)                         [line 195]
  └─► uvicorn.run(...)                                     [line 235]
```

**Startup is unchanged; only the shutdown block changed (§7.21).** The reception
usecase plugs in *underneath* it, which is the point of the registry pattern.

### 6.2 Where our usecase attaches

```
BranchBuilder.build(pipeline, usecase_id)     branch_builder.py  🟡 MODIFIED
  │
  ├─► registry.get(usecase_id)                registry.py        🟡 MODIFIED
  │      └─ "reception_counter" ──► ReceptionCounterProcessor    🟢 NEW
  │
  ├─ builds PGIE / tracker / SGIE elements from
  │     src/configs/branches/reception_counter.yaml               🟢 NEW
  │
  └─► _attach_mid_pipeline_probes(...)        🟢 NEW METHOD
         opt-in via metadata["head_crop_injector_after_sgie"]
         └─► probe_head_crop_injector.py      🔴 ABANDONED — see §7.2
             (method works; the reception path does not use it)
```

### 6.3 The adapter

`src/core/processors/ds_adapter_reception_counter.py` 🟢 **NEW**
Line numbers verified against the current file (Oct 2026).

```
ReceptionCounterDsConfig      :42   usecase config dataclass; from_metadata() at :109
                                    overlays YAML values onto the Python defaults
ReceptionCounterState         :119  per-frame counters returned to the probe
_PersonAttrs                  :153  age/gender vote + employee flag
  ├─ observe_logo()           :166  ASYMMETRIC lock — EMP locks forever, CUST never (§7.9)
  ├─ merge_from()             :184  folds one track's evidence into the person's record
  ├─ observe_face()           :201  accumulates gender_votes{} / age_reads[]
  └─ _confirm()               :213  re-evaluates confirmed_gender EVERY call — live,
                                    not final. This is why the on-screen label can
                                    disagree with visits.db (§7.14)
_publish_customers_finalized():235  → Kafka topic "reception-customer-finalized": one
                                    message per expired customer, carrying that
                                    customer's FINAL visits.db row (§7.19)
_publish_summary()            :269  → Kafka topic "reception-summary"
                                    (no per-visit event any more — §7.19)

ReceptionCounterProcessor     :352
  ├─ __init__()               :363
  │     ├─ _route_visit()     :458  closure → staff go to employee_store
  │     ├─ _attrs_for()       :466  closure → (gender, age) at close time, but ONLY
  │     │                           if confirmed_gender != "Unknown"
  │     └─ builds store → resolver → counter, in that order (the resolver needs
  │       visits.db's ids for extra_seed_ids — §7.10). store / employee_store
  │       are plain VisitStore instances.
  ├─ set_source_id()          :502
  │     └─► _restore_panel_from_visits()  :512  rebuilds entered/served from
  │                                             visits.db so a restart does not
  │                                             re-count people still in the zone
  ├─ _gallery_upkeep()        :568  every gallery_upkeep_interval frames:
  │     │                           store.expire(now) → publish → store.save()
  │     └─► _publish_finalized()  :580  reads the expired customers' rows
  │             │                       (VisitStore.rows_for_customer_ids,
  │             │                        visit_store.py:199) and hands them to
  │             └─► _publish_customers_finalized()  :235
  ├─ process_ds_detection()   :599  per-frame entry from draw_probe()
  │     ├─ counter.update()         ─► src/analytics/counter.py:65   🟢
  │     ├─ is_staff gate      :613   ─ ReID runs for everyone EXCEPT staff (§7.11)
  │     ├─ resolver.observe() :614   ─► src/reid/identity.py:164     🟢
  │     └─ _bind_identity()   :742   shares _PersonAttrs between tracker_id
  │                                  and the resolved customer_id
  ├─ _read_person_attrs()     :768  logo SGIE 7 + MiVolo SGIE 15 readback
  │     └─ :828  observe_logo is SKIPPED once resolver.resolve(tid) is not None —
  │              a ReID-vouched customer can never be relabelled staff (§7.12)
  ├─ style_obj_meta_for_osd() :681  per-box label; reads LIVE memory only,
  │                                 never visits.db (§7.14)
  ├─ draw_ds_detections()     :667  → _draw_hud() :307  top-right 5-row panel
  ├─ draw_probe()             :888  THE GStreamer pad probe — real entry point; takes the
  │                                 lock, then _draw_probe() :897 (§7.21)
  │     └─ :977   self._bound &= live_ids   ← _attrs itself is NEVER swept (§7.13)
  └─ shutdown()               :545  called by ProcessorRegistry.shutdown_all() from main.py's
                                    lifespan: flushes open visits, final summary, gallery,
                                    under the draw_probe lock (§7.21)
```

### 6.4 The counting state machine

`src/analytics/counter.py::ReceptionCounter` 🟢 **NEW**

```
update(tracker_id, frame, box, now)                       line 65
  ├─► point_in_zone((cx,cy), self.zone)   evaluation/zones.py
  ├─ inside  & no open visit   ──► open OpenVisit
  ├─ inside  & open visit      ──► extend it
  └─ outside & gap > max_break ──► _close()

sweep(frame)                                              line 88
  └─ close visits whose track vanished entirely

_close(tracker_id, visit)                                 line 97
  ├─► self.resolver.resolve(tracker_id)   ← identity read AT CLOSE
  │      (by now the vote has settled; at entry it usually had not)
  ├─ if unidentified: customer_id = f"unknown_t{tid}_{frame}"
  ├─ confirmed = frames_present >= confirm_frames
  ├─► self.route_visit(tracker_id, customer_id)   ← staff → separate store
  ├─► self.attrs_for(tracker_id)          line 125  ← (gender, age) snapshot
  │      Returns None unless the vote actually confirmed one. This single call
  │      is the ONLY moment gender/age reaches the database.
  └─► store.record_visit(customer_id, entered_at, last_seen,
                         confirmed=confirmed, gender=gender, age=age)
        └─ no Kafka event here any more — the final row is published once,
           when the gallery expires the customer (§7.19)

close()                                                   line 146
  └─ flush everyone still at the desk at shutdown
```

### 6.5 Harness vs service — same logic, two implementations

| Concern | Path C (harness) | Path A (service) |
|---|---|---|
| Zone test | inline in probe 3 | `ReceptionCounter.update()` |
| Visit lifecycle | frame counters + sets | `OpenVisit` + `_close()` |
| Staff separation | `entered.discard()` on lock | `route_visit` → second store |
| Identity read | `resolver.resolve()` per frame | at **visit close** |
| Output | `.mp4` + printed counts | Kafka + Prometheus + REST |
| Shared | `src/reid/*`, `evaluation/zones.py` | same |

They deliberately share the identity layer and the zone geometry, and differ
in how a visit is bookkept. The harness optimises for a visible, reproducible
experiment; the service optimises for streaming and durable output.

---

## 7. What changed, and why — indexed to the code

### 7.1 Logo SGIE class filter — the silent failure

| | |
|---|---|
| **File** | `/tmp/logo_sgie_7_abs.yaml` (container-local config) |
| **Was** | `operate-on-class-ids: "0"` |
| **Now** | `operate-on-class-ids: "1"` 🟡 |
| **Why** | `make_injector` (line 363) sets `class_id = PERSON_CLASS_ID = 1`. The SGIE filtered for 0 and silently discarded **every** detection. |
| **Effect** | `stats["logo_reads"]` 0 → **3,271** |

### 7.2 Head-crop injector — abandoned

| | |
|---|---|
| **File** | `src/core/processors/probe_head_crop_injector.py` ⚪ PRE-EXISTING |
| **Status** | 🔴 **ABANDONED** — still on disk, no longer in the reception path |
| **Was** | MiVolo ran on synthetic head crops (`unique_component_id = 16`, `HEAD_CROP_FRACTION = 0.25`) injected by `on_head_crop_injector_probe` |
| **Why it failed** | Crops were verified correct — right box, right class, 45 crops over 60 frames — and MiVolo still classified **none**. Its hardcoded `PERSON_PGIE_ID = 6` did not match this pipeline's injected id of 1. |
| **Now** | MiVolo classifies the **person box directly**: `operate_on_gie_id: 1` + `PROCESS_MODE_CLIP_OBJECTS`, copying the live deployment's `age_gender_sgie_triton.txt` |
| **Effect** | 0 → **46,943** classifier reads |

`branch_builder._attach_mid_pipeline_probes()` still exists and works, but the
reception path no longer uses it.

### 7.3 Employee flag — sticky on first hit

| | |
|---|---|
| **File** | `identity_agegender_preview.py::AgeGenderAttrs.observe_logo` line 259 🟡 |
| **Was** | `if has_logo: self.is_employee = True` — permanent, on the first hit |
| **Now** | `TYPE_CONFIRM_FRAMES = 15` (line 161) gate, locking **both** CUST and EMP |
| **Supporting change** | Pass 1 now collects `logo_hit_tids`; pass 2 calls `observe_logo(tid in logo_hit_tids)` once per person per frame, so clean frames are counted too |
| **Effect** | employee tracks 7/9 → **3/9** |

### 7.4 Staff double-counted as customers

| | |
|---|---|
| **File** | `identity_agegender_preview.py`, counting branch in probe 3 🟢 |
| **Added** | `entered.discard(cust)` / `served.discard(cust)` on EMP lock |
| **Why** | During the 15 confirmation frames the track is still CUST, so `entered.add(cust)` runs — and the set is permanent |
| **Effect** | 15h `ENTERED` 6 → **5** |

### 7.5 Dwell timer reset mid-visit

| | |
|---|---|
| **File** | `identity_agegender_preview.py`, probe 3 🟢 |
| **Was** | keyed on `tracker_id` |
| **Now** | `dwell_key = cust if cust is not None else f"t{tid}"` |
| **Why** | The tracker reassigns ids after occlusion, restarting the timer |

### 7.6 Gallery pruning — the wrong axis

| | |
|---|---|
| **File** | `src/reid/numpy_store.py::_prune` 🟢 |
| **Was** | keep the most mutually **dissimilar** vectors |
| **Now** | keep views **spread across time**; first and last always survive |
| **Why** | A 93-second track stored all 5 vectors from one 3-second window; on return the person scored **0.652** (below the 0.70 gate) and became a new customer. Time-spread vectors scored **0.709**. |

### 7.7 Restart support

| | |
|---|---|
| **File** | `identity_agegender_preview.py::main` 🟢 |
| **Added** | `--resume-gallery`, `--start-frame` |
| **Why** | The script wiped the gallery on every start, so a restart could never recover identity. Also propagated `start_frame` into `make_injector` and `make_identity_probe` counters — frozen detections are keyed by absolute frame number. |
| **Effect** | Restart with gallery: **6** customers; without: **8** |

### 7.8 Panel clipping

| | |
|---|---|
| **File** | `identity_agegender_preview.py`, panel block in probe 3 🟢 |
| **Was** | `panel_w` hardcoded |
| **Now** | `panel_w = max(len(t)) * 17.5 + pad*2` |
| **Why** | `nvdsosd` renders DejaVu Sans Mono size 22 at ~**17.3 px/char**, not the ~13 px PIL reports |

---

> **§7.9 onward — service-path fixes (Sep 28 – Oct 2026).**
> §7.1–7.8 above were found on the harness (path C). Everything below was found
> on the production adapter (path A) during full-video runs, and the file/line
> references are to `ds_adapter_reception_counter.py` unless stated otherwise.

### 7.9 Employee lock made asymmetric

| | |
|---|---|
| **File** | `ds_adapter_reception_counter.py::_PersonAttrs.observe_logo` line 166 🟡 |
| **Was** | Symmetric — the lock fired on **both** outcomes: a logo seen → EMP locked; `logo_min_frames` clean frames → CUST locked |
| **Now** | Only EMP ever locks. CUST stays the reversible default for the whole life of the track |
| **Why** | A real employee whose logo had not yet been visible (bad angle, brief occlusion) got permanently locked as CUSTOMER in the first few frames, and every later logo detection was then silently ignored by the `if self.type_locked: return` guard |
| **Principle** | A logo is positive evidence. The *absence* of a logo is not evidence of absence — so it must never lock |

### 7.10 Customer-id collision after a gallery wipe

| | |
|---|---|
| **File** | `src/reid/gallery.py::Gallery.__init__` line 85 🟡 |
| **Was** | `self._counter` seeded only from what was already in the **gallery** |
| **Now** | `max(_counter_from_store(store), _counter_from_ids(extra_seed_ids))`, where `extra_seed_ids` is `visits.db`'s `all_customer_ids_for_day()` (passed from the adapter) |
| **Why** | Wiping only the gallery (common while testing a new video same-day) reset the counter to 0001, which then collided with a `cust_YYYYMMDD_0001` **already in `visits.db` for a different, unrelated person** — silently overwriting their recorded age |
| **Known gap** | A person whose identity was minted but whose visit never *closed* (video ended first) is in neither source, so their number can still be reused. Observed; no data corruption resulted |

### 7.11 ReID gate blocked every customer

| | |
|---|---|
| **File** | `ds_adapter_reception_counter.py::process_ds_detection` line 654 🟡 |
| **Was** | `if vec is not None and not is_staff and not undecided:` where `undecided = attrs is None or not attrs.type_locked` |
| **Now** | `if vec is not None and not is_staff:` |
| **Why** | A direct consequence of §7.9. Once CUST stopped locking, `type_locked` never became `True` for a real customer — so the old condition blocked `resolver.observe()` **permanently for every customer**, and every customer visit closed `unidentified` |
| **Lesson** | `type_locked` had been doing double duty as a proxy for "decision is final". When its meaning changed, every reader of it had to be re-checked |

### 7.12 A confirmed customer could be relabelled staff

| | |
|---|---|
| **File** | `ds_adapter_reception_counter.py::_read_person_attrs` line 889 🟡 |
| **Added** | `if self.resolver.resolve(tid) is None:` guard around `observe_logo()` |
| **Why** | A customer sitting at the desk for a long visit could momentarily overlap a logo-bearing object, lock EMP from that point, and split one real visit into a CUST row **and** an EMP row — same `customer_id`, different `camera_id` (`cam_1` vs `cam_1_staff`) |
| **Principle** | Once the ReID gallery has vouched for a track as an identified customer, that verdict outranks a late logo sighting |

### 7.13 `_attrs` destroyed by a single dropped detection

| | |
|---|---|
| **File** | `ds_adapter_reception_counter.py::draw_probe` line 1029 🟡 |
| **Was** | `_attrs` was swept against `live_ids` every frame, exactly like `_bound` |
| **Now** | `_attrs` is **never swept**. Only `_bound` and `_first_seen` are |
| **Why** | One frame in which the detector produced no box for a still-visible person — a dropped detection, not an exit — deleted `_attrs[tid]`. The next frame NvDCF returned the *same* tracker_id for the *same* person, but `setdefault()` built a fresh `_PersonAttrs(type_locked=False)`, discarding an EMP verdict backed by 952+ logo observations |
| **Trade-off** | Accepted deliberately: if NvDCF ever reuses a tracker_id for a genuinely different person after a long gap, that person inherits the old verdict. Narrow, and judged the lesser risk |
| **Related** | `_employees_entered` had the same bug — it reset to 0 the moment an employee stepped off camera. Also no longer swept; it is a cumulative same-run count now, matching "Customer entered" |

### 7.14 On-screen label vs. database disagreement

| | |
|---|---|
| **Files** | `style_obj_meta_for_osd` line 722, `_PersonAttrs._confirm` line 213, `counter.py::_close` line 125 |
| **Status** | ⚪ **Not a bug — documented behaviour.** Investigated after a hijab-wearing woman showed `m 25-34` on screen while `visits.db` correctly held `female` |
| **Mechanism** | The OSD label reads `attrs.confirmed_gender` **live, every frame**. `_confirm()` recomputes `max(gender_votes)` on every new observation, so the label tracks whichever gender is currently winning. `visits.db` gets a value exactly **once**, when `_close()` calls `attrs_for()` — by which time the vote has usually settled |
| **Why the display cannot read the database** | The row does not exist until the visit closes. Sourcing the label from `visits.db` would leave every on-screen person unlabelled for their entire visit |
| **Confirmed** | The same customer recorded `female` across two independent full runs — the stored value is stable; only the live label oscillates |

### 7.15 ReID score dilution — the `top_k_observations` fix

| | |
|---|---|
| **File** | `src/reid/voting_v2.py::EvidenceAccumulator.top_k_votes` line 98 🟢 |
| **Was** | The vote averaged over **every** observation collected for the track |
| **Now** | Only the best `top_k` observations, **ranked by quality**, contribute |
| **Why** | A genuine returning customer scored 0.591–0.718 — below the gate — even though their individual best frames scored ~1.0 on direct cosine check. Early low-quality frames were dragging the weighted average down |
| **Ranked by quality, not by vote** | Ranking by which candidate an observation favoured would bias toward whatever already looked best |
| **Supporting changes** | `min_observations` 5 → 12 (more evidence gathered before deciding, since fewer of it is used), `top_k_observations: 5` added to config + YAML |
| **Effect** | The same failing re-matches moved to 0.728–0.999 |

### 7.16 Gallery stored near-duplicate frames

| | |
|---|---|
| **File** | `src/reid/voting_v2.py::_store_best` line 237, `_MIN_FRAME_GAP_FOR_DIVERSITY = 30` line 235 🟡 |
| **Was** | The 3 stored views were simply the 3 highest-quality observations |
| **Now** | Greedy pick: highest quality first, then skip any observation within 30 frames (~1 s) of one already chosen |
| **Why** | Pure quality ranking could store three near-consecutive frames of the same instant — same pose, same angle — giving the gallery no angular coverage of the person at all. Same failure class as §7.6, one level up |
| **Note** | No pose classifier exists, so frame distance is used as a cheap proxy for "a different look" |

### 7.17 Threshold tuning — the merge/miss trade-off

| | |
|---|---|
| **Files** | `src/configs/branches/reception_counter.yaml`, `ds_adapter_reception_counter.py` defaults |
| **Status** | 🔬 **Open — experimental.** No settled value yet |
| **History** | `0.75` (original) → `0.70` (to rescue the diluted scores of §7.15) → `0.75` (to stop merges) → currently `0.70` with `margin_threshold: 0.02` under test |
| **Evidence for lowering** | At 0.75 a genuine returning customer who had previously matched at 0.999 closed `unidentified` — a real false negative |
| **Evidence for raising** | At 0.70 three visibly different men (bald + glasses / white cap / bird T-shirt) were merged into one `customer_id` at scores 0.704–0.708 |
| **Direct measurement** | Cosine similarity read straight from `live.npy`: the intruder view scored **0.706–0.729** against the real person's views, while that person's own views scored **0.858–0.998** against each other. The separating band is therefore ~0.73–0.858 |
| **What `margin_threshold` does NOT fix** | It is only consulted **after** `best >= match_threshold` passes (`_evaluate`, voting_v2.py:179). In the merge cases the runner-up was far behind, so no margin value could have blocked them. Only `match_threshold` governs that gate |
| **Where margin DOES act** | `unknown_t3_16` — two candidates 0.026 apart, below the then-current `margin_threshold: 0.10`, so the vote returned UNKNOWN and refused to guess. That is the guard working as designed |

### 7.18 Branch-scoped storage paths

| | |
|---|---|
| **Files** | `reception_counter.yaml`, `run_reception_persistent.sh` 🟡 |
| **Was** | One fixed `logs/gallery/` + `logs/visits/visits.db` for every run |
| **Now** | `gallery_path` / `visits_db` / `branch_id` are per-branch (`logs/<Branch>/…`), and the runner script takes optional `$3` (output name) and `$4` (branch_id), defaulting to the old paths |
| **Why** | Testing a second camera/video against the same store silently mixed two unrelated populations of customers into one gallery and one day-count |
| **Also fixed** | `age_gender_zone` default was `"RECEPTION_AREA"` but every zones CSV in the project spells it `"RECIPITION_AREA"` (a pre-existing project-wide typo) — the default raised `ValueError` on any branch that did not override it |

### 7.19 Per-change Kafka event replaced by one final-row event

| | |
|---|---|
| **Files** | `ds_adapter_reception_counter.py` 🟡 (`_publish_customers_finalized` :235, `_gallery_upkeep` :564, `_publish_finalized` :576) · submodule `visit_store.py::rows_for_customer_ids` :199 🟡 · `numpy_store.py::expire` :138 and `store.py::expire` :56 🟡 · `deploy/kafka-test-compose.yml`, `deploy/check_kafka.py`, `justfile` 🟢 |
| **Was** | `_KafkaVisitStore.record_visit()` published `reception-visit` on **every** recorded visit — a message for every change to a row (each repeat visit moves `last_seen_at`, `dwell_seconds`, `confirmed`) |
| **Now** | Deleted: `_handle_visit_event`, `_KafkaVisitStore`, and the `_source_id` plumbing that only fed it. A customer's row is sent **once**, in its last-updated state, on `reception-customer-finalized`, when the gallery expires them |
| **Why** | A consumer that wants the finished record had to fold a stream of partial updates itself. Gallery expiry is the first moment the system knows the customer is not coming back |
| **`expire()` change** | `GalleryStore.expire` / `NumpyStore.expire` now return the removed customer_ids (was a count), so the caller knows exactly who left. `len()` of the result is the old number |
| **Lookup is by id, not by day** | `rows_for_customer_ids` filters on branch + customer_id only. `retention_hours` can exceed 24 h, so a customer can expire after midnight, when their row's `visit_day` is already yesterday. Ids embed their own date, so they are unique within a branch anyway |
| **Order in `_gallery_upkeep`** | `expire()` → publish → `save()`. A crash between publish and save replays the expiry on restart: a duplicate is possible, a loss is not |
| **Not covered** | `unknown_t*` visits and staff never enter the gallery, so they never get this event. A customer whose identity was minted but whose visit never closed has no row to send (logged as "had no visits.db row") |
| **Delivery** | Best-effort, like every other event: in-memory queue, `produce()` with no delivery callback. Consumers should dedupe on `(branch_id, customer_id)` |
| **⚠ Breaking** | Anything consuming `reception-visit` stops receiving. Nothing in this repo does; other services were not checked |
| **Test** | `just kafka-test-up && just kafka-e2e && just kafka-check`. `e2e` drives the real `ReceptionCounterProcessor` and `KafkaManager` against a throwaway KRaft broker: one old customer is expired, one fresh one is kept. It asserts the row content on the topic, that the fresh customer produced nothing, and that `reception-visit` was never created |
| **Test isolation** | The broker has its own network and **no published host port**; the client is a one-shot container from the dev image with the repo mounted read-only. It does not `docker exec` into `deepstream-hadeer`, because that container is attached to **no network** — not even `eagle_vision_network`, which its `HostConfig` names and which holds the real Kafka. Re-attaching it would have let the next video run publish to the real cluster |

### 7.20 Production container setup

| | |
|---|---|
| **Files** | `Dockerfile` 🟡 (production-stage `ENTRYPOINT`) · `.dockerignore` 🟡 · `deploy/docker-compose.prod.yml` 🟢 · `deploy/.env.example` 🟡 · `justfile` 🟡 (`prod-build`, `prod-up`, `prod-down`, `prod-logs`) |
| **Why SIGTERM never reached python** | The base image's ENTRYPOINT (`entrypoint.sh`) runs `nvidia_entrypoint.sh` without `exec`, so bash stays PID 1. Measured with a probe on the same image: `docker stop` used the whole grace period, exit 137, python never saw SIGTERM. `init: true` does not help: tini's child (bash) dies, python still gets no signal, exit 143 |
| **Fix** | `ENTRYPOINT ["/opt/nvidia/nvidia_entrypoint.sh"]` in the production stage; it `exec`s the CMD, so python is PID 1. Measured: stop in ~1 s, exit 0, SIGTERM received. The base compose's `tail` entrypoint still wins for the dev workflow (verified: base file alone still runs `tail -f /dev/null`) |
| **Build context** | `.dockerignore`'s `data/` and `docs/` are anchored at the context root, so the submodule's `data/` (41 GB) and `logs/` (28 GB), 173 videos / 63.6 GB, would have been copied by `COPY src/` (twice, plus `build_obfuscate.sh`'s `cp -a`). Now excluded, with `**/*.mkv`, `**/*.mp4` and `src/assets/`. Nothing on disk changes, only what `docker build` copies. Verified on a dummy tree |
| **Models** | `reception_counter_person_pgie_6.yaml` and `tracker_r38_reidtensor.yml` reference `/app/src/assets/...` (absolute), but the production code lives in `/app_obfuscate`. So `src/assets` is mounted at `/app/src/assets`, writable because TensorRT engines are cached there. The PeopleNet v2 and ReID models have no download source in `setup_assets.py` (TODO), so they are provisioned on the host by hand |
| **Compose** | `docker-compose.prod.yml` overrides the dev-flavoured base: image tag, `entrypoint: !reset null` (use the image's own), `stop_grace_period: 30s`, no `/dev/dri`, and exactly four mounts (logs, zones CSV, assets, app_dir) in place of the repo and X11 mounts. Camera auto-loading and Kafka credentials come from `deploy/.env`, already loaded by `env_file` |
| **Verified** | Merged compose config · `ENTRYPOINT` and `!reset null` through a throwaway compose project (dev flow unchanged; prod flow delivers SIGTERM, exit 0) · `.dockerignore` on a dummy tree · the repo's own obfuscation pipeline compiles and runs the reception code (adapter + 11 submodule files, flat imports, compiled dataclass) |
| **Not verified** | A full `docker compose build`; a real run from the built image with the models mounted; Dockerfile lint (`buildx` is not installed) |
| **Still open** | Superseded: shutdown is now wired up, see §7.21 |

### 7.21 Graceful shutdown wired up

| | |
|---|---|
| **Files** | `src/main.py` 🟡 (lifespan shutdown; the Kafka future is kept) · `registry.py::shutdown_all` :121 🟡 · `ds_adapter_reception_counter.py` 🟡 (`shutdown` :545, `draw_probe` :888 / `_draw_probe` :897, `_lock` / `_closed`) |
| **Was** | Nothing called `processor.shutdown()` or `KafkaManager.stop()`. Measured on the real service with a video and a SIGTERM while a person stood in the zone: **0 rows** in `visits.db` (the open visit was lost), **no** `shutdown` summary on Kafka, and the process **never exited on its own** (40 s+) even though the pipeline/RTSP cleanup had finished. The Kafka event-loop thread blocks forever on its queue and is non-daemon, so the interpreter waits for it |
| **Now** | First thing in the lifespan shutdown, before any GStreamer/RTSP teardown: `ProcessorRegistry.shutdown_all()` flushes each processor (closes open visits, publishes the `shutdown` summary, saves the gallery, closes the DBs). Then `kafka_manager.stop()`, and the lifespan waits up to 10 s for the queue to drain |
| **Why before the teardown** | That teardown has been seen to block: twice with no source attached, inside the GStreamer/RTSP cleanup (not in the three later runs with a source). Anything placed after it may never run. Data first, risky teardown last |
| **The race it had to avoid** | The pipeline is still PLAYING while processors shut down, so `draw_probe` (streaming thread) and `shutdown()` (shutdown thread) could touch the stores at once. `draw_probe` is now a thin wrapper that takes `_lock` and returns at once when `_closed`; `shutdown()` holds the same lock and is idempotent. Cost: one uncontended lock per buffer |
| **Measured, same test** | Before: process alive after SIGTERM, 0 visit rows, 0 shutdown messages. After: exits by itself in 4 s, 1 visit row (the open visit, 7 s, `confirmed=False`), 1 `reason: shutdown` summary. With Kafka unreachable (no network, like the dev container): exits in 9 s (the producer flush is capped at 5 s) and still saves the visit row |
| **Not changed, on purpose** | `if pipeline_state.is_running` in `main.py` is still a dead condition (it is tested after `is_running` was set to `False`). Making it live would call `set_state(NULL)` on the whole pipeline, which may itself be the blocking call. The intermittent teardown hang is not root-caused. No `os._exit` watchdog, because it would hide that hang |
| **Not tested** | More than one source; a real RTSP camera; `docker stop` through the compose/image path (the test used `--entrypoint`, as the Dockerfile now does); lock contention under real load; other use cases (`shutdown_all` skips adapters without a `shutdown` method) |

---

## 8. Follow one frame end to end

```
frames/005000.jpg
  └─ multifilesrc → jpegdec → nvstreammux
      └─ PROBE 1  make_injector.probe()          preview:364
          └─ inject person boxes, class_id=1
             └─ nvtracker  (tracker_r38_reidtensor.yml, outputReidTensor: 1)
                 └─ PROBE 2  make_identity_probe.probe()   preview:419
                     └─ resolver.observe()                 identity.py:169
                         ├─ score_observation()            quality.py
                         └─ voter.observe()                voting.py:133
                             ├─ store.search()             numpy_store.py
                             └─ _evaluate()                voting.py:166
                                 └─ customer_id
                     └─ nvinfer SGIE 7  (logo)
                         └─ nvinferserver SGIE 15  (MiVolo)
                             └─ PROBE 3  make_readback_probe.probe()  preview:518
                                 ├─ pass 1: logo_hit_tids
                                 ├─ pass 2: observe / observe_logo / resolve
                                 ├─ point_in_zone()        zones.py:106
                                 ├─ counting branch  (+ discard fix)
                                 └─ label + panel
                                     └─ nvdsosd → encoder → preview_*.mp4
```

---

## 9. File index

### Submodule `people_count_checker` — all 🟢 NEW

| Path | Role |
|---|---|
| `src/pipeline/identity_agegender_preview.py` | the whole preview pipeline, all 3 probes |
| `src/pipeline/gst_tracker_runner.py` | tracking-experiment harness |
| `src/pipeline/gst_tracker_runner_shadow.py` | shadow-track instrumentation (R24) |
| `src/reid/identity.py` | `IdentityResolver` — joins voting to the tracker |
| `src/reid/voting.py` | `VotingIdentifier`, `EvidenceAccumulator` |
| `src/reid/gallery.py` | `Gallery.identify` — NEW / UNKNOWN / KNOWN |
| `src/reid/quality.py` | `score_observation` |
| `src/reid/numpy_store.py` | persistence, `search`, `_prune`, `load`, `save` |
| `src/reid/retention.py` | expiry policy |
| `src/reid/store.py` | `GalleryStore` interface |
| `src/analytics/counter.py` | `ReceptionCounter` state machine |
| `src/config/tracker/tracker_r38_reidtensor.yml` | the tuned tracker config |
| `evaluation/zones.py` | `load_zones`, `point_in_zone` |
| `evaluation/tracking.py` | MOTA / IDF1 / ID-switch metrics |
| `evaluation/occlusion_eval.py` | box-merge measurement |
| `docs/reception_pipeline_walkthrough.md` | the full narrative walkthrough |
| `docs/tracking_experiments.md` | T-series and R-series logs |
| `docs/code_flow_tree.md` | this file |

Pre-existing (⚪, now under `legacy_code/`): `config.py`, `processor.py`,
`tracker.py`, `fsm.py`, `data_models.py`, `utils/*`.

### Parent repo `DeepstreamService`

| Path | Provenance | Role |
|---|---|---|
| `src/main.py` | 🟡 MODIFIED | service entry point (path A); shutdown block only (§7.21) |
| `src/core/builders/pipeline_builder.py` | ⚪ PRE-EXISTING | builds the top-level pipeline |
| `src/core/managers/probe_manager.py` | ⚪ PRE-EXISTING | attaches drawing probes |
| `src/services/camera_loader.py` | ⚪ PRE-EXISTING | multi-camera config |
| `src/core/processors/ds_adapter_reception_counter.py` | 🟢 NEW | the usecase adapter |
| `src/configs/branches/reception_counter.yaml` | 🟢 NEW | branch/GIE config + all ReID thresholds |
| `src/configs/trackers/tracker_r38_reidtensor.yml` | 🟢 NEW | NvDCF config for this usecase; `minIouDiff4NewTarget` history in §7.17's sibling note below |
| `src/configs/gies/reception_counter_person_pgie_6.yaml` | 🟢 NEW | person detector config |
| `src/configs/gies/reception_counter_logo_sgie_7.yaml` | 🟢 NEW | logo SGIE config |
| `src/configs/gies/reception_counter_mivolo_sgie_15.txt` | 🟢 NEW | MiVolo age/gender config |
| `src/core/builders/branch_builder.py` | 🟡 MODIFIED | simplified element graph; added `_attach_exclusion_zone_probe()`; `_attach_mid_pipeline_probes()` commented out (§7.2) |
| `src/core/processors/registry.py` | 🟡 MODIFIED | registered `reception_counter`; `shutdown_all()` (§7.21) |
| `src/utils/setup_assets.py` | 🟡 MODIFIED | restored MiVolo weights entry; added `reception_counter` ReID model entry |
| `justfile` | 🟡 MODIFIED | `triton-backends`, `setup-all-weights`; `kafka-test-up`, `kafka-test-down`, `kafka-check`, `kafka-e2e` |
| `deploy/kafka-test-compose.yml` | 🟢 NEW | throwaway single-node KRaft broker on an isolated network (§7.19) |
| `deploy/check_kafka.py` | 🟢 NEW | `topics` inspector + `e2e` test of the publish path (§7.19) |
| `Dockerfile` | 🟡 MODIFIED | production stage `ENTRYPOINT`, so SIGTERM reaches python (§7.20) |
| `.dockerignore` | 🟡 MODIFIED | keeps the submodule's `data/`, `logs/`, videos and `src/assets/` out of the build context (§7.20) |
| `deploy/docker-compose.prod.yml` | 🟢 NEW | production overrides on top of `docker-compose.yml` (§7.20) |
| `deploy/.env.example` | 🟡 MODIFIED | host-folder variables for the production compose |
| `src/core/processors/probe_head_crop_injector.py` | 🔴 ABANDONED | untouched on disk, out of the path |

**Tracker config — `minIouDiff4NewTarget`.** Swept 0.7418 → 0.74 → 0.73 → 0.70 →
0.65 → 0.60 → 0.50 against a desk-occlusion split (one person at the desk
producing two boxes of different heights, tracked as two ids). Only **0.60 and
below** reliably avoided the split. **Reverted to the original 0.7418** by
operator decision: a lower threshold makes NvDCF more willing to treat a new,
merely-nearby detection as an existing target, which risks merging two different
people — judged a worse failure than double-counting one. If the split needs
fixing again, 0.60 is the confirmed-working value.

### Legacy (path B) — submodule, pre-existing

| Path | Note |
|---|---|
| `usage_example.py` | the old entry point; imports now broken |
| `legacy_code/src/processor.py`, `tracker.py`, `fsm.py`, `config.py`, `data_models.py`, `utils/*` | moved here during the restructure — git shows the originals as deleted |

---

## 10. Reading the diffs yourself

```bash
# parent repo — the 5 modified files
cd ~/Documents/People_Counting_Tracking_DS/DeepstreamService
git diff --ignore-submodules=all --stat
git diff --ignore-submodules=all src/core/builders/branch_builder.py

# submodule — what existed at the last commit (17 files)
git -C src/core/processors/people_count_checker ls-tree -r --name-only HEAD

# a new file has no "before"; give git the intent to add, then diff
git add -N src/core/processors/ds_adapter_reception_counter.py
git diff src/core/processors/ds_adapter_reception_counter.py
git reset src/core/processors/ds_adapter_reception_counter.py   # undo
```

**Nothing in this document is committed yet.** All of it lives in the working
tree only — see §9 for the full list of files at risk.

# Fake-Code Contrasts

Ten bad-vs-better pairs. Every snippet is **illustrative fake code, not from this repo** — each pair ends with the real repo pattern it mirrors. Read the bad one, articulate the smell aloud, *then* read the fix.

---

## 1. Missing tenant scope (IDOR)

```python
# Illustrative fake code: not from this repo
dataset = session.scalar(select(Dataset).where(Dataset.id == dataset_id))          # BAD
dataset = session.scalar(select(Dataset).where(Dataset.id == dataset_id,
                                               Dataset.tenant_id == tenant_id))    # BETTER
```
Real rule: `api/AGENTS.md` "Always scope queries by tenant_id"; real shape: `document_indexing_task.py#L82-L84`.

## 2. Coupling UI shape to DB shape

```ts
// Illustrative fake code: not from this repo
// BAD — component renders raw API row; every schema change breaks the UI
const { data } = useQuery({ queryFn: () => get(`/apps/${id}`) })
return <h1>{data.app_model_config.pre_prompt}</h1>

// BETTER — typed model + mapping at the service layer
const app: App = await get<App>(`/apps/${id}`)   // App defined in web/types, mapped once
```
Real: response types in `web/types/`/`web/models/`, consumed via typed `get<App>` (`use-apps.ts#L46`).

## 3. Side effect inside a getter

```python
# Illustrative fake code: not from this repo
@property
def is_premium(self):
    self.tier = "premium"; db.session.commit()   # BAD: read that writes
    return True
```
Better: pure property + explicit service-layer transition + one-time backfill migration. Real counterexample to critique: `App.is_agent` (`api/models/model.py#L471-L484`).

## 4. Enqueue before commit

```python
# Illustrative fake code: not from this repo
doc = Document(...); session.add(doc)
index_task.delay(doc.id)      # BAD: worker may run before the row is visible
session.commit()
```
Better: commit, then delay. Real: `dataset_service.py#L2269` → `#L2271-L2277`.

## 5. Passing ORM objects to a task

```python
# Illustrative fake code: not from this repo
index_task.delay(document)        # BAD: serializes a session-bound object (or fails)
index_task.delay(document.id)     # BETTER: ids; task re-queries and re-validates
```
Real: `document_indexing_task.py#L33`, re-query `#L59-L63`.

## 6. N+1 via convenient properties

```python
# Illustrative fake code: not from this repo
# BAD — one query per app in the list
return [{"name": a.name, "site_code": a.site.code} for a in apps]

# BETTER — join or bulk-load once
sites = {s.app_id: s for s in session.scalars(select(Site).where(Site.app_id.in_(ids)))}
```
Real trap: `App.site` property issues a query per access (`model.py#L441-L443`).

## 7. Swallowed errors

```python
# Illustrative fake code: not from this repo
try:
    runner.run(docs)
except Exception:
    pass                                # BAD: invisible failure

except Exception as e:                  # BETTER: terminal status + message + log
    logger.exception("indexing failed dataset=%s", dataset_id)
    document.indexing_status = "error"; document.error = str(e); session.commit()
```
Real: `indexing_runner.py#L58-L67`; deliberate *scoped* swallow with logging for non-critical chained work `document_indexing_task.py#L167-L172` — swallowing is a decision, not a default.

## 8. Stale closure over chat state

```ts
// Illustrative fake code: not from this repo
// BAD — isResponding captured once; guard never sees updates
const handleSend = useCallback(() => { if (isResponding) return; ... }, [])

// BETTER — ref for latest value read inside long-lived callbacks
const isRespondingRef = useRef(false)
const handleSend = useCallback(() => { if (isRespondingRef.current) return; ... }, [])
```
Real: `isRespondingRef` in `chat/hooks.ts#L637-L640`.

## 9. Mutation without invalidation (stale cache)

```ts
// Illustrative fake code: not from this repo
useMutation({ mutationFn: deleteApp })                       // BAD: list keeps ghost row
useMutation({ mutationFn: deleteApp,
  onSuccess: () => qc.invalidateQueries({ queryKey: appKeys.list() }) })  // BETTER
```
Real: `use-apps.ts#L60-L81` (note it must hit *two* key systems during migration).

## 10. Casually widening a public contract

```python
# Illustrative fake code: not from this repo
# BAD — renames a field on a service-API response "for consistency"
return {"conversationId": conv.id}       # was conversation_id; breaks every SDK caller

# BETTER — additive change + deprecation window, or version the surface
return {"conversation_id": conv.id, "conversationId": conv.id}  # then deprecate
```
Real context: `api/controllers/service_api/` is a public contract consumed by `sdks/`; console API is quasi-internal — different change rules per surface (see [01-.../01-system-map.md](../01-codebase-cartography/01-system-map.md)).

---

**Drill**: for contrasts 1, 4, 6, and 9, write the *test* that would fail on the bad version (name the file where it would live in this repo). Rubric — Strong: your test fails for the stated reason and not incidentally; you can say which CI stage catches it.

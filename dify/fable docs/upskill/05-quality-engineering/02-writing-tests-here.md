# Writing Tests Here — Recipes

Commands are __inferred__ from `Makefile#L97-L105` and repo docs. Exact frontend script names: run `pnpm run` inside `web/` to confirm.

```bash
# Backend — one directory / one file / one test
make test TARGET_TESTS=./api/tests/unit_tests/services
make test TARGET_TESTS=./api/tests/unit_tests/services/test_account_service.py
make test TARGET_TESTS="./api/tests/unit_tests/services/test_account_service.py::test_authenticate_wrong_password"

# Frontend — vitest (from web/)
pnpm test                       # or the script listed in web/package.json
pnpm test service/base.spec.ts
```

All snippets below are **illustrative fake code, not from this repo** — they show the *shape*; adapt names to the real helpers you find in the nearest existing test file (always copy the neighboring test's setup style — that's the house pattern by definition).

## Recipe 1 — Happy path service test (Arrange-Act-Assert)

```python
# Illustrative fake code: not from this repo
def test_get_max_active_requests_min_of_limits(monkeypatch):
    app = make_app(max_active_requests=5)                      # Arrange (factory/fixture)
    monkeypatch.setattr(dify_config, "APP_MAX_ACTIVE_REQUESTS", 3)
    assert AppGenerateService._get_max_active_requests(app) == 3   # Act+Assert
```
Target: `app_generate_service.py#L272-L291`. Add the 0-means-infinite cases — that's where the bug would hide.

## Recipe 2 — Validation failure (Pydantic boundary)

```python
# Illustrative fake code: not from this repo
def test_chat_payload_rejects_bad_conversation_id():
    with pytest.raises(ValidationError):
        ChatMessagePayload.model_validate({"query": "hi", "model_config": {},
                                           "inputs": {}, "conversation_id": "not-a-uuid"})
```
Target: `completion.py#L55-L65`. Assert the *field* in the error, not just that it raised.

## Recipe 3 — Permission failure (authz decorator)

```python
# Illustrative fake code: not from this repo
def test_edit_permission_required_forbids_viewer(app_ctx, viewer_account):
    with login_as(viewer_account):                # helper that sets flask-login user
        with pytest.raises(Forbidden):
            guarded_view()
```
Target: `console/wraps.py#L306-L321`. Also test the EndUser case → Forbidden (the isinstance branch).

## Recipe 4 — Cross-tenant rejection (the isolation test every table deserves)

```python
# Illustrative fake code: not from this repo
def test_document_lookup_ignores_other_tenant(session, tenant_a_doc, tenant_b):
    found = session.scalar(select(Document).where(
        Document.id == tenant_a_doc.id, Document.dataset_id == tenant_b.dataset_id))
    assert found is None
```
Pattern justified by `api/AGENTS.md` scoping rule; real query shape `document_indexing_task.py#L82-L84`.

## Recipe 5 — Async side effect (task dispatch, not task execution)

```python
# Illustrative fake code: not from this repo
def test_document_create_enqueues_indexing(mocker, dataset, account):
    proxy = mocker.patch("services.dataset_service.DocumentIndexingTaskProxy")
    DocumentService.save_document_with_dataset_id(dataset, cfg, account)
    proxy.assert_called_once_with(dataset.tenant_id, dataset.id, mocker.ANY)
    proxy.return_value.delay.assert_called_once()
```
Target: `dataset_service.py#L2271-L2277`. Assert dispatch args (ids!), never run the worker in a unit test.

## Recipe 6 — Status machine transition (task-side)

```python
# Illustrative fake code: not from this repo
def test_billing_failure_marks_documents_error(session, over_quota_tenant, docs):
    _document_indexing(dataset_id, [d.id for d in docs])
    for d in docs:
        session.refresh(d)
        assert d.indexing_status == IndexingStatus.ERROR and d.error
```
Target: `document_indexing_task.py#L64-L91`.

## Recipe 7 — Frontend hook with mocked SSE (mirror of the real spec)

```ts
// Illustrative fake code: not from this repo
vi.mocked(ssePost).mockImplementation(async (_url, _p, options) => {
  options.onData?.('Hel', true, { messageId: 'm1' })
  options.onData?.('lo', false, { messageId: 'm1' })
  options.onMessageEnd?.(makeMessageEnd('m1'))
})
const { result } = renderHook(() => useChat(config, ...))
await act(() => result.current.handleSend('apps/x/chat-messages', { query: 'hi' }, cbs))
expect(lastAnswer(result.current.chatList).content).toBe('Hello')
```
The real version lives at `web/app/components/base/chat/chat/__tests__/hooks.spec.tsx#L150-L320` — read it before writing yours; follow `web/docs/test.md`.

## Recipe 8 — Cache invalidation (mutation → refetch)

```ts
// Illustrative fake code: not from this repo
const qc = new QueryClient()
render(<Wrapper client={qc}><DeleteButton appId="a1"/></Wrapper>)
await user.click(screen.getByRole('button'))
expect(invalidateSpy).toHaveBeenCalledWith({ queryKey: consoleQuery.apps.list.key() })
```
Target: `use-apps.ts#L70-L78` — and assert the **legacy key too**; that's the migration-era regression risk.

## Drill

Choose one recipe, write it for real in the repo, and run it. Record in your notes: which fixture/helper you copied from a neighboring test, and one thing the existing tests do that surprised you. (That habit — *read the neighboring test first* — is the whole recipe system.)

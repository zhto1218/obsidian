# Promotion Reference Cases Backend Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a TikHub-first backend pipeline for pre-generation promotion reference cases so content tasks can fetch real competitor inspiration cards, attach up to three selections, and feed those selections into generation without requiring hotspot selection.

**Architecture:** Keep the work inside the existing `promotions` content-task backend. Add a separate reference-case provider/service/repository path parallel to hotspots, persist normalized candidate cards plus task selections, and extend the existing generate endpoint so `reference_case_ids` is optional while `hotspot_ids` becomes compatible-but-optional. Do not build the missing frontend content-task UI in this plan.

**Tech Stack:** FastAPI, SQLModel, PostgreSQL, httpx, loguru, pytest, uv

---

## File Structure

**Create**

- `backend/app/infrastructure/reference_cases/provider.py`: TikHub HTTP client, payload normalization, and provider-mode routing.
- `backend/app/modules/promotions/reference_case_service.py`: refresh orchestration, cache fallback, and task-scoped candidate selection.
- `backend/tests/modules/test_promotion_reference_case_provider.py`: provider normalization and TikHub client tests.
- `backend/tests/modules/test_promotion_reference_case_service.py`: refresh/fallback tests and task-selection tests.

**Modify**

- `backend/app/core/config.py`: add TikHub/reference-case settings.
- `backend/.env.example`: document the new settings.
- `backend/app/modules/promotions/content_entity.py`: add task fetch status fields plus new reference-case entities.
- `backend/app/modules/promotions/content_schema.py`: add public schemas and `reference_case_ids` to generate request/detail payloads.
- `backend/app/modules/promotions/content_repository.py`: bootstrap new tables/columns and add CRUD helpers for candidates/task selections.
- `backend/app/modules/promotions/content_service.py`: add refresh endpoint orchestration and relax generation validation.
- `backend/app/modules/promotions/content_controller.py`: expose refresh API and detail payload fields.
- `backend/app/modules/promotions/generation_service.py`: turn selected reference cases into prompt angles.
- `backend/tests/modules/test_promotion_generation_service.py`: add failing tests for optional reference cases and prompt integration.
- `backend/tests/api/routes/test_promotion_content_tasks.py`: add API tests for refresh/detail/generate with reference cases.
- `README.md`: describe the backend reference-case pipeline and TikHub dependency.

## Task 1: Add TikHub provider and failing tests

**Files:**
- Create: `backend/app/infrastructure/reference_cases/provider.py`
- Create: `backend/tests/modules/test_promotion_reference_case_provider.py`
- Modify: `backend/app/core/config.py`
- Modify: `backend/.env.example`

- [ ] **Step 1: Write the failing provider tests**

```python
from datetime import date

import pytest

from app.infrastructure.reference_cases.provider import (
    _normalize_tikhub_reference_items,
    fetch_city_industry_reference_cases_from_provider,
)


def test_normalize_tikhub_reference_items_maps_metrics_and_urls() -> None:
    payload = [
        {
            "platform": "xiaohongshu",
            "title": "山顶早餐民宿爆了",
            "desc": "山景早餐场景内容互动高",
            "url": "https://www.xiaohongshu.com/explore/abc",
            "like_count": 23000,
            "publish_time": "2026-03-29T08:00:00",
        }
    ]

    items = _normalize_tikhub_reference_items(payload)

    assert items[0]["source_platform"] == "xiaohongshu"
    assert items[0]["summary"] == "山景早餐场景内容互动高"
    assert items[0]["metric_label"] == "2.3万赞"
    assert items[0]["source_url"] == "https://www.xiaohongshu.com/explore/abc"


def test_fetch_city_industry_reference_cases_uses_tikhub_mode(monkeypatch) -> None:
    from app.infrastructure.reference_cases import provider

    monkeypatch.setattr(provider.settings, "REFERENCE_CASE_PROVIDER_MODE", "tikhub", raising=False)
    monkeypatch.setattr(provider.settings, "REFERENCE_CASE_PROVIDER_API_KEY", "token", raising=False)
    monkeypatch.setattr(provider.settings, "REFERENCE_CASE_PROVIDER_TIKHUB_BASE_URL", "https://api.tikhub.dev", raising=False)

    def fake_fetch_from_tikhub(*, city_name: str, industry_type: str, task_date: date):
        assert city_name == "杭州市"
        assert industry_type == "homestay"
        assert task_date == date(2026, 3, 30)
        return [{"title": "ok"}]

    monkeypatch.setattr(provider, "_fetch_from_tikhub", fake_fetch_from_tikhub)

    result = fetch_city_industry_reference_cases_from_provider(
        city_name="杭州市",
        industry_type="homestay",
        task_date=date(2026, 3, 30),
    )

    assert result == [{"title": "ok"}]
```

- [ ] **Step 2: Run the provider tests and verify they fail**

Run: `cd backend && uv run pytest -q tests/modules/test_promotion_reference_case_provider.py`

Expected: FAIL with `ModuleNotFoundError` or missing symbol errors for `reference_cases.provider`.

- [ ] **Step 3: Implement the minimal provider and settings**

```python
# backend/app/core/config.py
REFERENCE_CASE_PROVIDER_MODE: str = "tikhub"
REFERENCE_CASE_PROVIDER_API_KEY: str = ""
REFERENCE_CASE_PROVIDER_TIKHUB_BASE_URL: HttpUrl | None = None
REFERENCE_CASE_PROVIDER_TIMEOUT_SECONDS: float = 15.0


# backend/app/infrastructure/reference_cases/provider.py
def _normalize_metric_label(*, like_count: int | None = None, digg_count: int | None = None) -> str | None:
    raw_count = like_count if like_count is not None else digg_count
    if raw_count is None:
        return None
    if raw_count >= 10000:
        return f"{raw_count / 10000:.1f}万赞"
    return f"{raw_count}赞"


def _normalize_tikhub_reference_items(items: object) -> list[dict[str, Any]]:
    normalized: list[dict[str, Any]] = []
    if not isinstance(items, list):
        return normalized
    for item in items:
        if not isinstance(item, dict):
            continue
        title = str(item.get("title") or "").strip()
        if not title:
            continue
        normalized.append(
            {
                "source_platform": str(item.get("platform") or "unknown").strip().lower(),
                "title": title,
                "summary": str(item.get("desc") or item.get("summary") or "").strip() or None,
                "inspiration_point": None,
                "metric_label": _normalize_metric_label(
                    like_count=item.get("like_count"),
                    digg_count=item.get("digg_count"),
                ),
                "source_name": str(item.get("source_name") or "").strip() or None,
                "source_url": str(item.get("url") or item.get("share_url") or "").strip() or None,
                "score": float(item.get("score") or 0),
                "published_at": item.get("publish_time"),
            }
        )
    return normalized
```

- [ ] **Step 4: Run the provider tests and verify they pass**

Run: `cd backend && uv run pytest -q tests/modules/test_promotion_reference_case_provider.py`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add backend/app/core/config.py backend/.env.example backend/app/infrastructure/reference_cases/provider.py backend/tests/modules/test_promotion_reference_case_provider.py
git commit -m "feat: add tikhub reference case provider"
```

## Task 2: Persist reference cases and expose refresh/detail APIs

**Files:**
- Modify: `backend/app/modules/promotions/content_entity.py`
- Modify: `backend/app/modules/promotions/content_schema.py`
- Modify: `backend/app/modules/promotions/content_repository.py`
- Create: `backend/app/modules/promotions/reference_case_service.py`
- Modify: `backend/app/modules/promotions/content_service.py`
- Modify: `backend/app/modules/promotions/content_controller.py`
- Create: `backend/tests/modules/test_promotion_reference_case_service.py`
- Modify: `backend/tests/api/routes/test_promotion_content_tasks.py`

- [ ] **Step 1: Write the failing service and API tests**

```python
from datetime import date
from types import SimpleNamespace


def test_refresh_reference_cases_returns_stale_cache_when_provider_fails(db, monkeypatch) -> None:
    from app.modules.promotions import content_repository, reference_case_service

    task = SimpleNamespace(
        id=...,
        org_id=...,
        city_name_snapshot="杭州市",
        industry_type_snapshot="homestay",
        task_date=date(2026, 3, 30),
    )
    content_repository.replace_reference_cases_for_city_industry_date(
        session=db,
        org_id=task.org_id,
        city_name=task.city_name_snapshot,
        industry_type=task.industry_type_snapshot,
        reference_date=task.task_date,
        items=[{"title": "缓存案例", "source_platform": "xiaohongshu"}],
    )

    def boom(**kwargs):
        raise RuntimeError("provider down")

    monkeypatch.setattr(reference_case_service, "fetch_city_industry_reference_cases_from_provider", boom)

    result = reference_case_service.refresh_reference_cases_for_task(session=db, task=task, force_refresh=True)

    assert result.status == "stale"
    assert len(result.reference_cases) == 1


def test_refresh_reference_cases_api_returns_candidates(client, normal_user_token_headers, monkeypatch) -> None:
    response = client.post(
        f"{settings.API_V1_STR}/promotion/stores/{store_id}/content-tasks/{task_id}/reference-cases/refresh",
        headers=normal_user_token_headers,
        json={"force_refresh": True},
    )

    assert response.status_code == 200
    assert response.json()["fetch_status"] == "ready"
```

- [ ] **Step 2: Run the focused tests and verify they fail**

Run: `cd backend && uv run pytest -q tests/modules/test_promotion_reference_case_service.py tests/api/routes/test_promotion_content_tasks.py -k reference_cases`

Expected: FAIL on missing tables, missing schemas, or missing endpoint errors.

- [ ] **Step 3: Add entities, repository helpers, service orchestration, and controller routes**

```python
# backend/app/modules/promotions/content_entity.py
class PromotionReferenceCase(SQLModel, table=True):
    __tablename__ = "mkt_promotion_reference_case"
    id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
    org_id: uuid.UUID = Field(foreign_key="mkt_org.id", index=True)
    city_name: str = Field(max_length=64, index=True)
    industry_type: str = Field(max_length=32, index=True)
    reference_date: date = Field(index=True)
    source_platform: str = Field(max_length=32)
    title: str = Field(max_length=128)
    summary: str | None = Field(default=None)
    inspiration_point: str | None = Field(default=None)
    metric_label: str | None = Field(default=None, max_length=64)
    source_name: str | None = Field(default=None, max_length=64)
    source_url: str | None = Field(default=None, max_length=512)
    score: float = Field(default=0.0, index=True)


class PromotionTaskReferenceCase(SQLModel, table=True):
    __tablename__ = "mkt_promotion_task_reference_case"
    id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
    org_id: uuid.UUID = Field(foreign_key="mkt_org.id", index=True)
    task_id: uuid.UUID = Field(foreign_key="mkt_promotion_content_task.id", index=True)
    reference_case_id: uuid.UUID = Field(foreign_key="mkt_promotion_reference_case.id", index=True)
    sort_order: int = Field(default=0, index=True)


# backend/app/modules/promotions/reference_case_service.py
@dataclass
class ReferenceCaseRefreshResult:
    status: str
    reference_cases: list[Any]
    error_message: str | None


def refresh_reference_cases_for_task(*, session: Session, task: PromotionContentTask, force_refresh: bool) -> ReferenceCaseRefreshResult:
    cached = content_repository.list_reference_cases_by_city_industry_date(...)
    if cached and not force_refresh:
        return ReferenceCaseRefreshResult(status="ready", reference_cases=cached, error_message=None)
    try:
        raw_items = fetch_city_industry_reference_cases_from_provider(...)
        normalized = build_reference_case_candidates(raw_items)
        content_repository.replace_reference_cases_for_city_industry_date(...)
        fresh = content_repository.list_reference_cases_by_city_industry_date(...)
        return ReferenceCaseRefreshResult(status="ready", reference_cases=fresh, error_message=None)
    except Exception as exc:
        if cached:
            return ReferenceCaseRefreshResult(status="stale", reference_cases=cached, error_message=str(exc))
        return ReferenceCaseRefreshResult(status="failed", reference_cases=[], error_message=str(exc))
```

- [ ] **Step 4: Run the focused tests and verify they pass**

Run: `cd backend && uv run pytest -q tests/modules/test_promotion_reference_case_service.py tests/api/routes/test_promotion_content_tasks.py -k reference_cases`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add backend/app/modules/promotions/content_entity.py backend/app/modules/promotions/content_schema.py backend/app/modules/promotions/content_repository.py backend/app/modules/promotions/reference_case_service.py backend/app/modules/promotions/content_service.py backend/app/modules/promotions/content_controller.py backend/tests/modules/test_promotion_reference_case_service.py backend/tests/api/routes/test_promotion_content_tasks.py
git commit -m "feat: add promotion reference case refresh flow"
```

## Task 3: Feed selected reference cases into generation and relax hotspot validation

**Files:**
- Modify: `backend/app/modules/promotions/content_schema.py`
- Modify: `backend/app/modules/promotions/content_service.py`
- Modify: `backend/app/modules/promotions/generation_service.py`
- Modify: `backend/tests/modules/test_promotion_generation_service.py`
- Modify: `backend/tests/api/routes/test_promotion_content_tasks.py`
- Modify: `README.md`

- [ ] **Step 1: Write the failing generation tests**

```python
def test_trigger_content_generation_allows_zero_hotspots_and_zero_reference_cases(db, monkeypatch) -> None:
    from app.modules.promotions import content_service

    monkeypatch.setattr(content_service, "ensure_generation_dependencies_available", lambda: None)
    monkeypatch.setattr(content_service, "dispatch_content_generation_task", lambda *, task_id: None)

    task = content_service.trigger_content_generation(
        session=db,
        org_id=org.id,
        store_id=store.id,
        task_id=task.id,
        platforms=["dianping"],
        hotspot_ids=[],
        reference_case_ids=[],
    )

    assert task.status == "generating"


def test_build_reference_case_angles_prefers_inspiration_point() -> None:
    from app.modules.promotions.generation_service import build_reference_case_angles

    items = [
        {
            "title": "山顶早餐民宿爆了",
            "inspiration_point": "把早餐写成仪式感体验",
            "metric_label": "2.3万赞",
        }
    ]

    assert build_reference_case_angles(reference_case_items=items) == [
        "山顶早餐民宿爆了｜把早餐写成仪式感体验｜2.3万赞"
    ]
```

- [ ] **Step 2: Run the failing generation tests**

Run: `cd backend && uv run pytest -q tests/modules/test_promotion_generation_service.py tests/api/routes/test_promotion_content_tasks.py -k "reference_case or zero_hotspots"`

Expected: FAIL because `reference_case_ids` is unsupported or hotspot validation still requires 1-3 selections.

- [ ] **Step 3: Implement the relaxed validation and prompt integration**

```python
# backend/app/modules/promotions/content_schema.py
class PromotionContentTaskGenerateRequest(SQLModel):
    platforms: list[PlatformCode]
    hotspot_ids: list[uuid.UUID] = Field(default_factory=list)
    reference_case_ids: list[uuid.UUID] = Field(default_factory=list)


# backend/app/modules/promotions/content_service.py
def trigger_content_generation(..., hotspot_ids: list[uuid.UUID], reference_case_ids: list[uuid.UUID]) -> PromotionContentTask:
    ...
    if len(unique_reference_case_ids) > 3:
        raise ValueError("Select up to 3 reference cases before generation")
    ...
    content_repository.replace_task_reference_cases(...)
    if available_hotspots and unique_hotspot_ids:
        content_repository.replace_task_hotspots(...)


# backend/app/modules/promotions/generation_service.py
def build_reference_case_angles(*, reference_case_items: list[Any]) -> list[str]:
    angles: list[str] = []
    for item in reference_case_items[:3]:
        angle = "｜".join(
            part for part in [
                str(item.get("title") or "").strip(),
                str(item.get("inspiration_point") or "").strip(),
                str(item.get("metric_label") or "").strip(),
            ]
            if part
        )
        if angle:
            angles.append(angle[:160])
    return angles
```

- [ ] **Step 4: Run the updated generation tests and the direct API regression**

Run: `cd backend && uv run pytest -q tests/modules/test_promotion_generation_service.py tests/api/routes/test_promotion_content_tasks.py`

Expected: PASS.

- [ ] **Step 5: Run lint-style backend verification**

Run: `cd backend && uv run ruff check app tests`

Expected: PASS with no new errors.

- [ ] **Step 6: Commit**

```bash
git add backend/app/modules/promotions/content_schema.py backend/app/modules/promotions/content_service.py backend/app/modules/promotions/generation_service.py backend/tests/modules/test_promotion_generation_service.py backend/tests/api/routes/test_promotion_content_tasks.py README.md
git commit -m "feat: support reference cases in promotion generation"
```

## Self-Review

- Spec coverage: This plan covers the approved backend subsystem only: TikHub-first provider, candidate refresh, task attachment, and prompt integration. The missing frontend content-task page is intentionally excluded and should be handled in a separate implementation plan.
- Placeholder scan: No `TODO`, `TBD`, or abstract “handle appropriately” instructions remain.
- Type consistency: All plan steps use `reference_case_ids`, `PromotionReferenceCase`, and `PromotionTaskReferenceCase` consistently.

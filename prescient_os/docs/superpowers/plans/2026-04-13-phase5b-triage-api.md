# Phase 5b: Auto-Answer & Triage API Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the triage API endpoints — question submission, category classification, auto-answer generation, escalation, triage queue CRUD, and sharing — so that sponsors can ask questions and operators can manage the attention queue.

**Architecture:** New use cases in `prescient/triage/application/` follow the existing Command/Result/UseCase pattern. Routes in `prescient/triage/api/routes.py` wire infrastructure adapters. The auto-answer orchestrator chains an LLM classification call with a scoped copilot retrieval call. The triage queue API supports listing, filtering, and status updates.

**Tech Stack:** Python 3.12, FastAPI, SQLAlchemy 2.0 (async), Pydantic v2, Anthropic SDK, pytest

**Depends on:** Phase 5a (triage domain entities, tables, AccessContext)

---

## File Structure

### New files:

```
apps/api/src/prescient/triage/application/__init__.py
apps/api/src/prescient/triage/application/use_cases/__init__.py
apps/api/src/prescient/triage/application/use_cases/submit_question.py    — create question + triage item
apps/api/src/prescient/triage/application/use_cases/list_questions.py     — sponsor's own questions
apps/api/src/prescient/triage/application/use_cases/respond_question.py   — operator submits final answer
apps/api/src/prescient/triage/application/use_cases/list_triage_items.py  — operator triage queue
apps/api/src/prescient/triage/application/use_cases/update_triage_item.py — resolve, dismiss, assign
apps/api/src/prescient/triage/application/use_cases/start_brainstorm.py   — create brainstorm session
apps/api/src/prescient/triage/application/use_cases/share_artifact.py     — operator shares artifact
apps/api/src/prescient/triage/application/use_cases/list_shared.py        — list shared items
apps/api/src/prescient/triage/application/classify.py                     — LLM category classification
apps/api/src/prescient/triage/application/auto_answer.py                  — auto-answer orchestrator
apps/api/src/prescient/triage/api/__init__.py
apps/api/src/prescient/triage/api/routes.py                               — all triage HTTP routes
apps/api/tests/unit/triage/test_submit_question.py
apps/api/tests/unit/triage/test_classify.py
apps/api/tests/unit/triage/test_list_triage_items.py
apps/api/tests/unit/triage/test_respond_question.py
apps/api/tests/unit/triage/test_share_artifact.py
```

### Modified files:

```
apps/api/src/prescient/main.py                          — include triage router
apps/api/src/prescient/briefing/infrastructure/source_adapter.py — add get_triage_items()
```

---

### Task 1: SubmitQuestion Use Case

**Files:**
- Create: `apps/api/src/prescient/triage/application/__init__.py`
- Create: `apps/api/src/prescient/triage/application/use_cases/__init__.py`
- Create: `apps/api/src/prescient/triage/application/use_cases/submit_question.py`
- Test: `apps/api/tests/unit/triage/test_submit_question.py`

- [ ] **Step 1: Create package structure**

```python
# apps/api/src/prescient/triage/application/__init__.py
```

```python
# apps/api/src/prescient/triage/application/use_cases/__init__.py
```

- [ ] **Step 2: Write failing test**

```python
# apps/api/tests/unit/triage/test_submit_question.py
"""Unit tests for SubmitQuestion use case."""

from __future__ import annotations

from uuid import uuid4

import pytest

from prescient.triage.application.use_cases.submit_question import (
    SubmitQuestion,
    SubmitQuestionCommand,
    SubmitQuestionResult,
)
from prescient.triage.domain.enums import (
    TriageItemStatus,
    TriageItemType,
    TriagePriority,
    TriageQuestionStatus,
)


class _FakeSession:
    def __init__(self) -> None:
        self.added: list[object] = []

    def add(self, row: object) -> None:
        self.added.append(row)


class _FakeRow:
    def __init__(self, **kwargs: object) -> None:
        for key, value in kwargs.items():
            setattr(self, key, value)


class TestSubmitQuestion:
    @pytest.mark.asyncio
    async def test_creates_question_and_triage_item(self) -> None:
        session = _FakeSession()
        use_case = SubmitQuestion(
            session=session,
            question_row_factory=_FakeRow,
            item_row_factory=_FakeRow,
        )

        command = SubmitQuestionCommand(
            asker_user_id="michael-torres-001",
            asker_org_id="org-summit-001",
            target_company_org_id="org-peloton-001",
            question_text="What is driving subscriber churn?",
            asker_role="operating_partner",
        )

        result = await use_case.execute(command)

        assert isinstance(result, SubmitQuestionResult)
        assert result.question_id is not None
        assert result.triage_item_id is not None
        # Should create both a question row and a triage item row
        assert len(session.added) == 2

    @pytest.mark.asyncio
    async def test_priority_from_role(self) -> None:
        session = _FakeSession()
        use_case = SubmitQuestion(
            session=session,
            question_row_factory=_FakeRow,
            item_row_factory=_FakeRow,
        )

        # Operating partner → high priority
        command = SubmitQuestionCommand(
            asker_user_id="michael-torres-001",
            asker_org_id="org-summit-001",
            target_company_org_id="org-peloton-001",
            question_text="Test question",
            asker_role="operating_partner",
        )
        await use_case.execute(command)

        question_row = session.added[0]
        assert question_row.priority == TriagePriority.HIGH.value  # type: ignore[attr-defined]

    @pytest.mark.asyncio
    async def test_board_member_gets_high_priority(self) -> None:
        session = _FakeSession()
        use_case = SubmitQuestion(
            session=session,
            question_row_factory=_FakeRow,
            item_row_factory=_FakeRow,
        )

        command = SubmitQuestionCommand(
            asker_user_id="board-001",
            asker_org_id="org-summit-001",
            target_company_org_id="org-peloton-001",
            question_text="Test question",
            asker_role="board_member",
        )
        await use_case.execute(command)

        question_row = session.added[0]
        assert question_row.priority == TriagePriority.HIGH.value  # type: ignore[attr-defined]

    @pytest.mark.asyncio
    async def test_pe_analyst_gets_medium_priority(self) -> None:
        session = _FakeSession()
        use_case = SubmitQuestion(
            session=session,
            question_row_factory=_FakeRow,
            item_row_factory=_FakeRow,
        )

        command = SubmitQuestionCommand(
            asker_user_id="analyst-001",
            asker_org_id="org-summit-001",
            target_company_org_id="org-peloton-001",
            question_text="Test question",
            asker_role="pe_analyst",
        )
        await use_case.execute(command)

        question_row = session.added[0]
        assert question_row.priority == TriagePriority.MEDIUM.value  # type: ignore[attr-defined]
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `cd apps/api && uv run pytest tests/unit/triage/test_submit_question.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 4: Implement SubmitQuestion use case**

```python
# apps/api/src/prescient/triage/application/use_cases/submit_question.py
"""SubmitQuestion use case — sponsor submits a question, creates both
a TriageQuestion and a TriageItem in the attention queue.
"""

from __future__ import annotations

from dataclasses import dataclass, field
from datetime import datetime
from typing import Any, Protocol
from uuid import UUID, uuid4

from prescient.shared.types import utcnow
from prescient.triage.domain.enums import (
    TriageItemStatus,
    TriageItemType,
    TriagePriority,
    TriageQuestionStatus,
)

_ROLE_PRIORITY: dict[str, TriagePriority] = {
    "board_member": TriagePriority.HIGH,
    "operating_partner": TriagePriority.HIGH,
    "pe_analyst": TriagePriority.MEDIUM,
}


class SubmitQuestionSessionPort(Protocol):
    def add(self, row: Any) -> None: ...


@dataclass(frozen=True)
class SubmitQuestionCommand:
    asker_user_id: str
    asker_org_id: str
    target_company_org_id: str
    question_text: str
    asker_role: str
    question_id: UUID = field(default_factory=uuid4)
    triage_item_id: UUID = field(default_factory=uuid4)
    created_at: datetime = field(default_factory=utcnow)


@dataclass(frozen=True)
class SubmitQuestionResult:
    question_id: UUID
    triage_item_id: UUID


class SubmitQuestion:
    """Create a TriageQuestion and its corresponding TriageItem."""

    def __init__(
        self,
        session: SubmitQuestionSessionPort,
        question_row_factory: Any,
        item_row_factory: Any,
    ) -> None:
        self._session = session
        self._question_factory = question_row_factory
        self._item_factory = item_row_factory

    async def execute(self, command: SubmitQuestionCommand) -> SubmitQuestionResult:
        priority = _ROLE_PRIORITY.get(command.asker_role, TriagePriority.MEDIUM)

        question_row = self._question_factory(
            id=command.question_id,
            asker_user_id=command.asker_user_id,
            asker_org_id=command.asker_org_id,
            target_company_org_id=command.target_company_org_id,
            question_text=command.question_text,
            categories_detected=[],
            status=TriageQuestionStatus.SUBMITTED.value,
            partial_answer=None,
            partial_citations=None,
            escalation_reason=None,
            final_answer=None,
            final_citations=None,
            brainstorm_conversation_id=None,
            priority=priority.value,
            created_at=command.created_at,
            answered_at=None,
        )
        self._session.add(question_row)

        item_row = self._item_factory(
            id=command.triage_item_id,
            organization_id=command.target_company_org_id,
            item_type=TriageItemType.SPONSOR_QUESTION.value,
            source_id=str(command.question_id),
            title=f"Question from sponsor",
            summary=command.question_text[:256],
            priority=priority.value,
            priority_signals=[f"from {command.asker_role}"],
            status=TriageItemStatus.PENDING.value,
            assigned_to_user_id=None,
            created_at=command.created_at,
            resolved_at=None,
        )
        self._session.add(item_row)

        return SubmitQuestionResult(
            question_id=command.question_id,
            triage_item_id=command.triage_item_id,
        )
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd apps/api && uv run pytest tests/unit/triage/test_submit_question.py -v`
Expected: PASS (all 4 tests)

- [ ] **Step 6: Commit**

```bash
git add apps/api/src/prescient/triage/application/ apps/api/tests/unit/triage/test_submit_question.py
git commit -m "feat(triage): add SubmitQuestion use case — creates question + triage item"
```

---

### Task 2: Category Classification

**Files:**
- Create: `apps/api/src/prescient/triage/application/classify.py`
- Test: `apps/api/tests/unit/triage/test_classify.py`

- [ ] **Step 1: Write failing test**

```python
# apps/api/tests/unit/triage/test_classify.py
"""Unit tests for category classification."""

from __future__ import annotations

import pytest

from prescient.organizations.domain.entities.visibility_rule import DataCategory
from prescient.triage.application.classify import classify_question_categories


class TestClassifyQuestionCategories:
    @pytest.mark.asyncio
    async def test_kpi_question(self) -> None:
        """A question about metrics should detect kpi_metrics category."""
        categories = await classify_question_categories(
            question_text="What is the current subscriber churn rate?",
            llm_call=_fake_llm_kpi,
        )
        assert DataCategory.KPI_METRICS in categories

    @pytest.mark.asyncio
    async def test_narrative_question(self) -> None:
        """A question about strategy should detect narrative category."""
        categories = await classify_question_categories(
            question_text="What is the team's plan to address churn?",
            llm_call=_fake_llm_narrative,
        )
        assert DataCategory.NARRATIVE in categories

    @pytest.mark.asyncio
    async def test_multi_category_question(self) -> None:
        """A question can touch multiple categories."""
        categories = await classify_question_categories(
            question_text="What is driving churn and what's the strategic response?",
            llm_call=_fake_llm_multi,
        )
        assert DataCategory.KPI_METRICS in categories
        assert DataCategory.NARRATIVE in categories


async def _fake_llm_kpi(prompt: str) -> str:
    return '["kpi_metrics"]'


async def _fake_llm_narrative(prompt: str) -> str:
    return '["narrative"]'


async def _fake_llm_multi(prompt: str) -> str:
    return '["kpi_metrics", "narrative"]'
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd apps/api && uv run pytest tests/unit/triage/test_classify.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Implement classify_question_categories**

```python
# apps/api/src/prescient/triage/application/classify.py
"""Category classification for triage questions.

Uses an LLM call to determine which DataCategory values a question touches.
The LLM call is injected as a callable so tests can provide fakes.
"""

from __future__ import annotations

import json
import logging
from collections.abc import Awaitable, Callable

from prescient.organizations.domain.entities.visibility_rule import DataCategory

logger = logging.getLogger(__name__)

_VALID_CATEGORIES = {c.value for c in DataCategory}

_CLASSIFICATION_PROMPT = """You are classifying a sponsor's question about a portfolio company.

Given the question below, determine which data categories it touches. Return a JSON array of category strings.

Valid categories:
- "kpi_metrics" — quantitative metrics, financials, KPIs, numbers, trends
- "narrative" — strategic context, plans, reasoning, qualitative analysis
- "decisions" — decisions made or pending, rationale for choices
- "action_items" — tasks, action items, follow-ups, assignments
- "intelligence_signals" — market intelligence, competitive signals, news
- "briefings" — summary briefings, updates, status reports

Question: {question_text}

Return ONLY a JSON array of matching category strings, e.g. ["kpi_metrics", "narrative"]. No other text."""


async def classify_question_categories(
    *,
    question_text: str,
    llm_call: Callable[[str], Awaitable[str]],
) -> list[DataCategory]:
    """Classify which DataCategory values a question touches.

    Parameters
    ----------
    question_text:
        The sponsor's question.
    llm_call:
        An async callable that takes a prompt string and returns the LLM
        response text. Injected so tests can provide fakes.
    """
    prompt = _CLASSIFICATION_PROMPT.format(question_text=question_text)
    raw = await llm_call(prompt)

    try:
        parsed = json.loads(raw.strip())
    except json.JSONDecodeError:
        logger.warning("Failed to parse classification response: %s", raw)
        return [DataCategory.NARRATIVE]  # safe fallback

    if not isinstance(parsed, list):
        logger.warning("Classification response is not a list: %s", raw)
        return [DataCategory.NARRATIVE]

    categories: list[DataCategory] = []
    for item in parsed:
        if isinstance(item, str) and item in _VALID_CATEGORIES:
            categories.append(DataCategory(item))

    return categories or [DataCategory.NARRATIVE]
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd apps/api && uv run pytest tests/unit/triage/test_classify.py -v`
Expected: PASS (all 3 tests)

- [ ] **Step 5: Commit**

```bash
git add apps/api/src/prescient/triage/application/classify.py apps/api/tests/unit/triage/test_classify.py
git commit -m "feat(triage): add LLM-based category classification for sponsor questions"
```

---

### Task 3: Auto-Answer Orchestrator

**Files:**
- Create: `apps/api/src/prescient/triage/application/auto_answer.py`

- [ ] **Step 1: Implement auto-answer orchestrator**

This orchestrator chains classification → visibility check → scoped answer generation → escalation. It updates the TriageQuestion row through the session.

```python
# apps/api/src/prescient/triage/application/auto_answer.py
"""Auto-answer orchestrator for triage questions.

Runs after question submission:
1. Classify question categories
2. Check visibility rules
3. Generate partial/full answer from visible data
4. Escalate gated portions to operator
"""

from __future__ import annotations

import logging
from collections.abc import Awaitable, Callable
from dataclasses import dataclass

from prescient.auth.access_context import AccessContext
from prescient.organizations.domain.entities.visibility_rule import DataCategory, GateType
from prescient.triage.application.classify import classify_question_categories
from prescient.triage.domain.enums import TriageQuestionStatus

logger = logging.getLogger(__name__)


@dataclass(frozen=True)
class AutoAnswerResult:
    """Result of the auto-answer attempt."""

    status: TriageQuestionStatus
    partial_answer: str | None
    partial_citations: list[dict] | None
    escalation_reason: str | None
    categories_detected: list[str]


async def attempt_auto_answer(
    *,
    question_text: str,
    visible_categories: frozenset[DataCategory],
    gated_categories: frozenset[DataCategory],
    blocked_categories: frozenset[DataCategory],
    classify_llm_call: Callable[[str], Awaitable[str]],
    answer_llm_call: Callable[[str], Awaitable[str]],
    retrieve_visible_context: Callable[[str, list[DataCategory]], Awaitable[str]],
) -> AutoAnswerResult:
    """Attempt to auto-answer a sponsor question.

    Parameters
    ----------
    question_text:
        The sponsor's question.
    visible_categories:
        Categories the sponsor can see automatically.
    gated_categories:
        Categories requiring operator approval.
    blocked_categories:
        Categories the sponsor cannot see.
    classify_llm_call:
        Async callable for category classification.
    answer_llm_call:
        Async callable for answer generation.
    retrieve_visible_context:
        Async callable that retrieves relevant context from visible data.
        Takes (question_text, visible_categories) and returns context string.
    """
    # Step 1: Classify
    detected = await classify_question_categories(
        question_text=question_text,
        llm_call=classify_llm_call,
    )
    detected_values = [c.value for c in detected]

    # Step 2: Visibility check
    visible_detected = [c for c in detected if c in visible_categories]
    gated_detected = [c for c in detected if c in gated_categories]
    blocked_detected = [c for c in detected if c in blocked_categories]

    # Step 3: Determine path
    all_visible = len(gated_detected) == 0 and len(blocked_detected) == 0

    if not visible_detected and (gated_detected or blocked_detected):
        # Nothing visible — escalate immediately
        reason_parts = []
        if gated_detected:
            reason_parts.append(
                f"question touches gated categories: {', '.join(c.value for c in gated_detected)}"
            )
        if blocked_detected:
            reason_parts.append(
                f"question touches blocked categories: {', '.join(c.value for c in blocked_detected)}"
            )
        return AutoAnswerResult(
            status=TriageQuestionStatus.ESCALATED,
            partial_answer=None,
            partial_citations=None,
            escalation_reason="; ".join(reason_parts),
            categories_detected=detected_values,
        )

    # Step 4: Generate answer from visible data
    context = await retrieve_visible_context(question_text, visible_detected)
    answer_prompt = (
        f"Answer the following question using ONLY the context provided. "
        f"If the context doesn't contain enough information, say so clearly.\n\n"
        f"Context:\n{context}\n\n"
        f"Question: {question_text}\n\n"
        f"Answer:"
    )
    answer_text = await answer_llm_call(answer_prompt)

    if all_visible:
        # Fully answered from visible data
        return AutoAnswerResult(
            status=TriageQuestionStatus.ANSWERED,
            partial_answer=answer_text,
            partial_citations=[],  # citations populated by retrieval layer
            escalation_reason=None,
            categories_detected=detected_values,
        )

    # Partial answer + escalation
    reason_parts = []
    if gated_detected:
        reason_parts.append(
            f"question also touches gated categories: {', '.join(c.value for c in gated_detected)}"
        )
    if blocked_detected:
        reason_parts.append(
            f"question touches blocked categories: {', '.join(c.value for c in blocked_detected)}"
        )

    return AutoAnswerResult(
        status=TriageQuestionStatus.PARTIALLY_ANSWERED,
        partial_answer=answer_text,
        partial_citations=[],
        escalation_reason="; ".join(reason_parts),
        categories_detected=detected_values,
    )
```

- [ ] **Step 2: Commit**

```bash
git add apps/api/src/prescient/triage/application/auto_answer.py
git commit -m "feat(triage): add auto-answer orchestrator — classify, check visibility, answer/escalate"
```

---

### Task 4: Remaining Use Cases (List, Respond, Update, Share)

**Files:**
- Create: `apps/api/src/prescient/triage/application/use_cases/list_questions.py`
- Create: `apps/api/src/prescient/triage/application/use_cases/respond_question.py`
- Create: `apps/api/src/prescient/triage/application/use_cases/list_triage_items.py`
- Create: `apps/api/src/prescient/triage/application/use_cases/update_triage_item.py`
- Create: `apps/api/src/prescient/triage/application/use_cases/start_brainstorm.py`
- Create: `apps/api/src/prescient/triage/application/use_cases/share_artifact.py`
- Create: `apps/api/src/prescient/triage/application/use_cases/list_shared.py`
- Test: `apps/api/tests/unit/triage/test_respond_question.py`
- Test: `apps/api/tests/unit/triage/test_list_triage_items.py`
- Test: `apps/api/tests/unit/triage/test_share_artifact.py`

- [ ] **Step 1: Write failing test for RespondQuestion**

```python
# apps/api/tests/unit/triage/test_respond_question.py
"""Unit tests for RespondQuestion use case."""

from __future__ import annotations

from datetime import UTC, datetime
from uuid import uuid4

import pytest

from prescient.triage.application.use_cases.respond_question import (
    RespondQuestion,
    RespondQuestionCommand,
    RespondQuestionResult,
)
from prescient.triage.domain.enums import TriageQuestionStatus


class _FakeRow:
    def __init__(self, **kwargs: object) -> None:
        for key, value in kwargs.items():
            setattr(self, key, value)


class _FakeQueryAdapter:
    def __init__(self, question_row: _FakeRow | None, item_row: _FakeRow | None) -> None:
        self._question = question_row
        self._item = item_row

    async def get_question(self, question_id: object) -> _FakeRow | None:
        return self._question

    async def get_triage_item_by_source(self, source_id: str) -> _FakeRow | None:
        return self._item


class TestRespondQuestion:
    @pytest.mark.asyncio
    async def test_sets_final_answer(self) -> None:
        qid = uuid4()
        question_row = _FakeRow(
            id=qid,
            status=TriageQuestionStatus.ESCALATED.value,
            final_answer=None,
            final_citations=None,
            answered_at=None,
        )
        item_row = _FakeRow(
            id=uuid4(),
            status="pending",
            resolved_at=None,
        )
        use_case = RespondQuestion(query=_FakeQueryAdapter(question_row, item_row))

        result = await use_case.execute(
            RespondQuestionCommand(
                question_id=qid,
                final_answer="Here is the strategic context...",
                final_citations=[{"type": "artifact", "id": "art-001"}],
            )
        )

        assert isinstance(result, RespondQuestionResult)
        assert question_row.final_answer == "Here is the strategic context..."
        assert question_row.status == TriageQuestionStatus.ANSWERED.value
        assert question_row.answered_at is not None
        assert item_row.status == "resolved"
        assert item_row.resolved_at is not None
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/api && uv run pytest tests/unit/triage/test_respond_question.py -v`
Expected: FAIL

- [ ] **Step 3: Implement RespondQuestion**

```python
# apps/api/src/prescient/triage/application/use_cases/respond_question.py
"""RespondQuestion use case — operator submits final answer to a triage question."""

from __future__ import annotations

from dataclasses import dataclass, field
from datetime import datetime
from typing import Any, Protocol
from uuid import UUID

from prescient.shared.errors import NotFoundError
from prescient.shared.types import utcnow
from prescient.triage.domain.enums import TriageItemStatus, TriageQuestionStatus
from prescient.triage.domain.errors import TriageStateError


class RespondQuestionQueryPort(Protocol):
    async def get_question(self, question_id: UUID) -> Any | None: ...
    async def get_triage_item_by_source(self, source_id: str) -> Any | None: ...


@dataclass(frozen=True)
class RespondQuestionCommand:
    question_id: UUID
    final_answer: str
    final_citations: list[dict] = field(default_factory=list)
    answered_at: datetime = field(default_factory=utcnow)


@dataclass(frozen=True)
class RespondQuestionResult:
    question_id: UUID


class RespondQuestion:
    def __init__(self, query: RespondQuestionQueryPort) -> None:
        self._query = query

    async def execute(self, command: RespondQuestionCommand) -> RespondQuestionResult:
        question = await self._query.get_question(command.question_id)
        if question is None:
            raise NotFoundError(f"triage question {command.question_id} not found")

        if question.status not in (
            TriageQuestionStatus.ESCALATED.value,
            TriageQuestionStatus.PARTIALLY_ANSWERED.value,
        ):
            raise TriageStateError(
                f"cannot respond to question in state {question.status}; "
                "expected escalated or partially_answered"
            )

        question.final_answer = command.final_answer
        question.final_citations = command.final_citations
        question.status = TriageQuestionStatus.ANSWERED.value
        question.answered_at = command.answered_at

        # Also resolve the corresponding triage item
        item = await self._query.get_triage_item_by_source(str(command.question_id))
        if item is not None:
            item.status = TriageItemStatus.RESOLVED.value
            item.resolved_at = command.answered_at

        return RespondQuestionResult(question_id=command.question_id)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/api && uv run pytest tests/unit/triage/test_respond_question.py -v`
Expected: PASS

- [ ] **Step 5: Implement ListQuestions, ListTriageItems, UpdateTriageItem, StartBrainstorm, ShareArtifact, ListShared**

```python
# apps/api/src/prescient/triage/application/use_cases/list_questions.py
"""ListQuestions use case — sponsor lists their own questions."""

from __future__ import annotations

from dataclasses import dataclass
from typing import Protocol

from prescient.triage.domain.entities.triage_question import TriageQuestion


class ListQuestionsQueryPort(Protocol):
    async def list_by_asker(self, asker_user_id: str, target_company_org_id: str | None) -> list[TriageQuestion]: ...


@dataclass(frozen=True)
class ListQuestionsCommand:
    asker_user_id: str
    target_company_org_id: str | None = None


class ListQuestions:
    def __init__(self, query: ListQuestionsQueryPort) -> None:
        self._query = query

    async def execute(self, command: ListQuestionsCommand) -> list[TriageQuestion]:
        return await self._query.list_by_asker(
            asker_user_id=command.asker_user_id,
            target_company_org_id=command.target_company_org_id,
        )
```

```python
# apps/api/src/prescient/triage/application/use_cases/list_triage_items.py
"""ListTriageItems use case — operator views the triage queue."""

from __future__ import annotations

from dataclasses import dataclass
from typing import Protocol

from prescient.triage.domain.entities.triage_item import TriageItem
from prescient.triage.domain.enums import TriageItemStatus, TriageItemType, TriagePriority


class ListTriageItemsQueryPort(Protocol):
    async def list_for_org(
        self,
        organization_id: str,
        item_type: TriageItemType | None,
        priority: TriagePriority | None,
        status: TriageItemStatus | None,
    ) -> list[TriageItem]: ...


@dataclass(frozen=True)
class ListTriageItemsCommand:
    organization_id: str
    item_type: TriageItemType | None = None
    priority: TriagePriority | None = None
    status: TriageItemStatus | None = None


class ListTriageItems:
    def __init__(self, query: ListTriageItemsQueryPort) -> None:
        self._query = query

    async def execute(self, command: ListTriageItemsCommand) -> list[TriageItem]:
        return await self._query.list_for_org(
            organization_id=command.organization_id,
            item_type=command.item_type,
            priority=command.priority,
            status=command.status,
        )
```

```python
# apps/api/src/prescient/triage/application/use_cases/update_triage_item.py
"""UpdateTriageItem use case — resolve, dismiss, or assign a triage item."""

from __future__ import annotations

from dataclasses import dataclass, field
from datetime import datetime
from typing import Any, Protocol
from uuid import UUID

from prescient.shared.errors import NotFoundError
from prescient.shared.types import utcnow
from prescient.triage.domain.enums import TriageItemStatus
from prescient.triage.domain.errors import TriageStateError


class UpdateTriageItemQueryPort(Protocol):
    async def get(self, item_id: UUID) -> Any | None: ...


@dataclass(frozen=True)
class UpdateTriageItemCommand:
    item_id: UUID
    action: str  # "resolve", "dismiss", "assign", "start"
    assigned_to_user_id: str | None = None
    resolved_at: datetime = field(default_factory=utcnow)


@dataclass(frozen=True)
class UpdateTriageItemResult:
    item_id: UUID
    new_status: str


class UpdateTriageItem:
    def __init__(self, query: UpdateTriageItemQueryPort) -> None:
        self._query = query

    async def execute(self, command: UpdateTriageItemCommand) -> UpdateTriageItemResult:
        item = await self._query.get(command.item_id)
        if item is None:
            raise NotFoundError(f"triage item {command.item_id} not found")

        if command.action == "resolve":
            item.status = TriageItemStatus.RESOLVED.value
            item.resolved_at = command.resolved_at
        elif command.action == "dismiss":
            item.status = TriageItemStatus.DISMISSED.value
            item.resolved_at = command.resolved_at
        elif command.action == "start":
            item.status = TriageItemStatus.IN_PROGRESS.value
        elif command.action == "assign":
            item.assigned_to_user_id = command.assigned_to_user_id
            item.status = TriageItemStatus.IN_PROGRESS.value
        else:
            raise TriageStateError(f"unknown action: {command.action}")

        return UpdateTriageItemResult(
            item_id=command.item_id,
            new_status=item.status,
        )
```

```python
# apps/api/src/prescient/triage/application/use_cases/start_brainstorm.py
"""StartBrainstorm use case — create a copilot conversation for triage response drafting."""

from __future__ import annotations

from dataclasses import dataclass, field
from datetime import datetime
from typing import Any, Protocol
from uuid import UUID, uuid4

from prescient.shared.errors import NotFoundError
from prescient.shared.types import utcnow


class StartBrainstormQueryPort(Protocol):
    async def get_question(self, question_id: UUID) -> Any | None: ...


class StartBrainstormSessionPort(Protocol):
    def add(self, row: Any) -> None: ...


@dataclass(frozen=True)
class StartBrainstormCommand:
    question_id: UUID
    operator_user_id: str
    conversation_id: UUID = field(default_factory=uuid4)
    created_at: datetime = field(default_factory=utcnow)


@dataclass(frozen=True)
class StartBrainstormResult:
    conversation_id: UUID
    system_prompt: str


class StartBrainstorm:
    """Create a copilot conversation pre-loaded with triage context."""

    def __init__(
        self,
        query: StartBrainstormQueryPort,
        session: StartBrainstormSessionPort,
        conversation_row_factory: Any,
    ) -> None:
        self._query = query
        self._session = session
        self._conversation_factory = conversation_row_factory

    async def execute(self, command: StartBrainstormCommand) -> StartBrainstormResult:
        question = await self._query.get_question(command.question_id)
        if question is None:
            raise NotFoundError(f"triage question {command.question_id} not found")

        system_prompt = (
            f"You are helping an operator draft a response to a sponsor question.\n\n"
            f"SPONSOR QUESTION: {question.question_text}\n\n"
            f"PARTIAL ANSWER ALREADY SHARED: {question.partial_answer or 'None'}\n\n"
            f"ESCALATION REASON: {question.escalation_reason or 'N/A'}\n\n"
            f"Help the operator draft a response that:\n"
            f"1. Answers the question as fully as possible\n"
            f"2. Is informed by internal context without leaking gated information\n"
            f"3. Suggests what artifacts to share alongside the response"
        )

        conv_row = self._conversation_factory(
            id=command.conversation_id,
            user_id=command.operator_user_id,
            company_slug="peloton",  # TODO: resolve from target_company_org_id
            title=f"Triage: {question.question_text[:80]}",
            system_prompt=system_prompt,
            created_at=command.created_at,
        )
        self._session.add(conv_row)

        # Link conversation to question
        question.brainstorm_conversation_id = command.conversation_id

        return StartBrainstormResult(
            conversation_id=command.conversation_id,
            system_prompt=system_prompt,
        )
```

```python
# apps/api/src/prescient/triage/application/use_cases/share_artifact.py
"""ShareArtifact use case — operator shares an artifact with a sponsor org."""

from __future__ import annotations

from dataclasses import dataclass, field
from datetime import datetime
from typing import Any, Protocol
from uuid import UUID, uuid4

from prescient.shared.types import utcnow


class ShareArtifactSessionPort(Protocol):
    def add(self, row: Any) -> None: ...


@dataclass(frozen=True)
class ShareArtifactCommand:
    artifact_id: str
    sponsor_org_id: str
    shared_by_user_id: str
    triage_question_id: UUID | None = None
    shared_item_id: UUID = field(default_factory=uuid4)
    shared_at: datetime = field(default_factory=utcnow)


@dataclass(frozen=True)
class ShareArtifactResult:
    shared_item_id: UUID


class ShareArtifact:
    def __init__(
        self,
        session: ShareArtifactSessionPort,
        row_factory: Any,
    ) -> None:
        self._session = session
        self._row_factory = row_factory

    async def execute(self, command: ShareArtifactCommand) -> ShareArtifactResult:
        row = self._row_factory(
            id=command.shared_item_id,
            artifact_id=command.artifact_id,
            sponsor_org_id=command.sponsor_org_id,
            shared_by_user_id=command.shared_by_user_id,
            shared_at=command.shared_at,
            triage_question_id=command.triage_question_id,
        )
        self._session.add(row)
        return ShareArtifactResult(shared_item_id=command.shared_item_id)
```

```python
# apps/api/src/prescient/triage/application/use_cases/list_shared.py
"""ListShared use case — list shared items for a sponsor org."""

from __future__ import annotations

from dataclasses import dataclass
from typing import Protocol

from prescient.triage.domain.entities.shared_item import SharedItem


class ListSharedQueryPort(Protocol):
    async def list_for_sponsor(self, sponsor_org_id: str) -> list[SharedItem]: ...


@dataclass(frozen=True)
class ListSharedCommand:
    sponsor_org_id: str


class ListShared:
    def __init__(self, query: ListSharedQueryPort) -> None:
        self._query = query

    async def execute(self, command: ListSharedCommand) -> list[SharedItem]:
        return await self._query.list_for_sponsor(sponsor_org_id=command.sponsor_org_id)
```

- [ ] **Step 6: Write tests for ListTriageItems and ShareArtifact**

```python
# apps/api/tests/unit/triage/test_list_triage_items.py
"""Unit tests for ListTriageItems use case."""

from __future__ import annotations

from datetime import UTC, datetime
from uuid import uuid4

import pytest

from prescient.triage.application.use_cases.list_triage_items import (
    ListTriageItems,
    ListTriageItemsCommand,
)
from prescient.triage.domain.entities.triage_item import TriageItem
from prescient.triage.domain.enums import (
    TriageItemStatus,
    TriageItemType,
    TriagePriority,
)


class _FakeQueryAdapter:
    def __init__(self, items: list[TriageItem]) -> None:
        self._items = items

    async def list_for_org(
        self,
        organization_id: str,
        item_type: TriageItemType | None,
        priority: TriagePriority | None,
        status: TriageItemStatus | None,
    ) -> list[TriageItem]:
        result = [i for i in self._items if i.organization_id == organization_id]
        if item_type is not None:
            result = [i for i in result if i.item_type == item_type]
        if priority is not None:
            result = [i for i in result if i.priority == priority]
        if status is not None:
            result = [i for i in result if i.status == status]
        return result


class TestListTriageItems:
    @pytest.mark.asyncio
    async def test_returns_items_for_org(self) -> None:
        items = [
            TriageItem(
                id=uuid4(),
                organization_id="org-peloton-001",
                item_type=TriageItemType.SPONSOR_QUESTION,
                source_id="q-001",
                title="Question about churn",
                summary="What is driving churn?",
                priority=TriagePriority.HIGH,
                priority_signals=("from operating_partner",),
                status=TriageItemStatus.PENDING,
                created_at=datetime.now(tz=UTC),
            ),
        ]
        use_case = ListTriageItems(query=_FakeQueryAdapter(items))
        result = await use_case.execute(
            ListTriageItemsCommand(organization_id="org-peloton-001")
        )
        assert len(result) == 1
        assert result[0].title == "Question about churn"

    @pytest.mark.asyncio
    async def test_filters_by_type(self) -> None:
        items = [
            TriageItem(
                id=uuid4(),
                organization_id="org-peloton-001",
                item_type=TriageItemType.SPONSOR_QUESTION,
                source_id="q-001",
                title="Question",
                summary="Test",
                priority=TriagePriority.HIGH,
                priority_signals=(),
                status=TriageItemStatus.PENDING,
                created_at=datetime.now(tz=UTC),
            ),
            TriageItem(
                id=uuid4(),
                organization_id="org-peloton-001",
                item_type=TriageItemType.KPI_ANOMALY,
                source_id="a-001",
                title="Anomaly",
                summary="Test",
                priority=TriagePriority.MEDIUM,
                priority_signals=(),
                status=TriageItemStatus.PENDING,
                created_at=datetime.now(tz=UTC),
            ),
        ]
        use_case = ListTriageItems(query=_FakeQueryAdapter(items))
        result = await use_case.execute(
            ListTriageItemsCommand(
                organization_id="org-peloton-001",
                item_type=TriageItemType.SPONSOR_QUESTION,
            )
        )
        assert len(result) == 1
        assert result[0].item_type == TriageItemType.SPONSOR_QUESTION
```

```python
# apps/api/tests/unit/triage/test_share_artifact.py
"""Unit tests for ShareArtifact use case."""

from __future__ import annotations

import pytest

from prescient.triage.application.use_cases.share_artifact import (
    ShareArtifact,
    ShareArtifactCommand,
    ShareArtifactResult,
)


class _FakeSession:
    def __init__(self) -> None:
        self.added: list[object] = []

    def add(self, row: object) -> None:
        self.added.append(row)


class _FakeRow:
    def __init__(self, **kwargs: object) -> None:
        for key, value in kwargs.items():
            setattr(self, key, value)


class TestShareArtifact:
    @pytest.mark.asyncio
    async def test_creates_shared_item(self) -> None:
        session = _FakeSession()
        use_case = ShareArtifact(session=session, row_factory=_FakeRow)

        result = await use_case.execute(
            ShareArtifactCommand(
                artifact_id="art-001",
                sponsor_org_id="org-summit-001",
                shared_by_user_id="sarah-chen-001",
            )
        )

        assert isinstance(result, ShareArtifactResult)
        assert len(session.added) == 1
        row = session.added[0]
        assert row.artifact_id == "art-001"  # type: ignore[attr-defined]
        assert row.sponsor_org_id == "org-summit-001"  # type: ignore[attr-defined]
```

- [ ] **Step 7: Run all tests**

Run: `cd apps/api && uv run pytest tests/unit/triage/ -v`
Expected: PASS (all tests)

- [ ] **Step 8: Commit**

```bash
git add apps/api/src/prescient/triage/application/use_cases/ apps/api/tests/unit/triage/
git commit -m "feat(triage): add all triage use cases — list, respond, update, brainstorm, share"
```

---

### Task 5: Triage API Routes

**Files:**
- Create: `apps/api/src/prescient/triage/api/__init__.py`
- Create: `apps/api/src/prescient/triage/api/routes.py`
- Modify: `apps/api/src/prescient/main.py`

- [ ] **Step 1: Create routes**

```python
# apps/api/src/prescient/triage/api/__init__.py
```

```python
# apps/api/src/prescient/triage/api/routes.py
"""Triage HTTP routes.

POST   /triage/questions                — sponsor submits a question
GET    /triage/questions                — sponsor lists own questions
GET    /triage/questions/{id}           — get question detail
POST   /triage/questions/{id}/respond   — operator submits final answer
GET    /triage/items                    — operator triage queue
GET    /triage/items/{id}               — triage item detail
PATCH  /triage/items/{id}               — update status (resolve/dismiss/assign/start)
POST   /triage/items/{id}/brainstorm    — start brainstorm session
POST   /sharing/items                   — share artifact with sponsor
GET    /sharing/items                   — list shared items
DELETE /sharing/items/{id}              — revoke sharing
"""

from __future__ import annotations

from datetime import datetime
from typing import Annotated, Any
from uuid import UUID

from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel, Field
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from prescient.auth.access_context import AccessContext, get_access_context
from prescient.auth.base import CurrentUser, get_current_user
from prescient.db import get_session
from prescient.shared.errors import NotFoundError
from prescient.triage.application.use_cases.list_questions import (
    ListQuestions,
    ListQuestionsCommand,
)
from prescient.triage.application.use_cases.list_shared import (
    ListShared,
    ListSharedCommand,
)
from prescient.triage.application.use_cases.list_triage_items import (
    ListTriageItems,
    ListTriageItemsCommand,
)
from prescient.triage.application.use_cases.respond_question import (
    RespondQuestion,
    RespondQuestionCommand,
)
from prescient.triage.application.use_cases.share_artifact import (
    ShareArtifact,
    ShareArtifactCommand,
)
from prescient.triage.application.use_cases.submit_question import (
    SubmitQuestion,
    SubmitQuestionCommand,
)
from prescient.triage.application.use_cases.update_triage_item import (
    UpdateTriageItem,
    UpdateTriageItemCommand,
)
from prescient.triage.domain.entities.triage_item import TriageItem
from prescient.triage.domain.entities.triage_question import TriageQuestion
from prescient.triage.domain.enums import (
    TriageItemStatus,
    TriageItemType,
    TriagePriority,
    TriageQuestionStatus,
)
from prescient.triage.domain.errors import TriageStateError
from prescient.triage.infrastructure.tables.shared_item import SharedItemTable
from prescient.triage.infrastructure.tables.triage_item import TriageItemTable
from prescient.triage.infrastructure.tables.triage_question import TriageQuestionTable

triage_router = APIRouter(prefix="/triage", tags=["triage"])
sharing_router = APIRouter(prefix="/sharing", tags=["sharing"])

SessionDep = Annotated[AsyncSession, Depends(get_session)]
AccessDep = Annotated[AccessContext, Depends(get_access_context)]
UserDep = Annotated[CurrentUser, Depends(get_current_user)]


# ---------------------------------------------------------------------------
# Infrastructure adapters
# ---------------------------------------------------------------------------


class _CreateSessionAdapter:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    def add(self, row: object) -> None:
        self._session.add(row)


class _ListQuestionsAdapter:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def list_by_asker(
        self, asker_user_id: str, target_company_org_id: str | None
    ) -> list[TriageQuestion]:
        stmt = select(TriageQuestionTable).where(
            TriageQuestionTable.asker_user_id == asker_user_id
        )
        if target_company_org_id is not None:
            stmt = stmt.where(
                TriageQuestionTable.target_company_org_id == target_company_org_id
            )
        stmt = stmt.order_by(TriageQuestionTable.created_at.desc())
        result = await self._session.execute(stmt)
        return [_row_to_question(r) for r in result.scalars().all()]


class _ListTriageItemsAdapter:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def list_for_org(
        self,
        organization_id: str,
        item_type: TriageItemType | None,
        priority: TriagePriority | None,
        status: TriageItemStatus | None,
    ) -> list[TriageItem]:
        stmt = select(TriageItemTable).where(
            TriageItemTable.organization_id == organization_id
        )
        if item_type is not None:
            stmt = stmt.where(TriageItemTable.item_type == item_type.value)
        if priority is not None:
            stmt = stmt.where(TriageItemTable.priority == priority.value)
        if status is not None:
            stmt = stmt.where(TriageItemTable.status == status.value)
        # Sort by priority order (critical first), then recency
        stmt = stmt.order_by(
            TriageItemTable.priority.asc(),  # critical < high < medium < low alphabetically works
            TriageItemTable.created_at.desc(),
        )
        result = await self._session.execute(stmt)
        return [_row_to_item(r) for r in result.scalars().all()]


class _RespondQuestionAdapter:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def get_question(self, question_id: UUID) -> TriageQuestionTable | None:
        stmt = select(TriageQuestionTable).where(TriageQuestionTable.id == question_id)
        result = await self._session.execute(stmt)
        return result.scalar_one_or_none()

    async def get_triage_item_by_source(self, source_id: str) -> TriageItemTable | None:
        stmt = select(TriageItemTable).where(
            TriageItemTable.source_id == source_id,
            TriageItemTable.item_type == TriageItemType.SPONSOR_QUESTION.value,
        )
        result = await self._session.execute(stmt)
        return result.scalar_one_or_none()


class _UpdateTriageItemAdapter:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def get(self, item_id: UUID) -> TriageItemTable | None:
        stmt = select(TriageItemTable).where(TriageItemTable.id == item_id)
        result = await self._session.execute(stmt)
        return result.scalar_one_or_none()


class _ListSharedAdapter:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def list_for_sponsor(self, sponsor_org_id: str) -> list[Any]:
        stmt = (
            select(SharedItemTable)
            .where(SharedItemTable.sponsor_org_id == sponsor_org_id)
            .order_by(SharedItemTable.shared_at.desc())
        )
        result = await self._session.execute(stmt)
        return list(result.scalars().all())


# ---------------------------------------------------------------------------
# Row → Entity mappers
# ---------------------------------------------------------------------------


def _row_to_question(row: TriageQuestionTable) -> TriageQuestion:
    return TriageQuestion(
        id=row.id,
        asker_user_id=row.asker_user_id,
        asker_org_id=row.asker_org_id,
        target_company_org_id=row.target_company_org_id,
        question_text=row.question_text,
        categories_detected=tuple(row.categories_detected or []),
        status=TriageQuestionStatus(row.status),
        partial_answer=row.partial_answer,
        partial_citations=tuple(row.partial_citations) if row.partial_citations else None,
        escalation_reason=row.escalation_reason,
        final_answer=row.final_answer,
        final_citations=tuple(row.final_citations) if row.final_citations else None,
        brainstorm_conversation_id=row.brainstorm_conversation_id,
        priority=TriagePriority(row.priority),
        created_at=row.created_at,
        answered_at=row.answered_at,
    )


def _row_to_item(row: TriageItemTable) -> TriageItem:
    return TriageItem(
        id=row.id,
        organization_id=row.organization_id,
        item_type=TriageItemType(row.item_type),
        source_id=row.source_id,
        title=row.title,
        summary=row.summary,
        priority=TriagePriority(row.priority),
        priority_signals=tuple(row.priority_signals or []),
        status=TriageItemStatus(row.status),
        assigned_to_user_id=row.assigned_to_user_id,
        created_at=row.created_at,
        resolved_at=row.resolved_at,
    )


# ---------------------------------------------------------------------------
# Request / Response models
# ---------------------------------------------------------------------------


class SubmitQuestionBody(BaseModel):
    target_company_org_id: str
    question_text: str = Field(min_length=1, max_length=2000)


class SubmitQuestionResponse(BaseModel):
    question_id: UUID
    triage_item_id: UUID


class QuestionResponse(BaseModel):
    id: UUID
    asker_user_id: str
    asker_org_id: str
    target_company_org_id: str
    question_text: str
    categories_detected: list[str]
    status: str
    partial_answer: str | None
    escalation_reason: str | None
    final_answer: str | None
    priority: str
    created_at: datetime
    answered_at: datetime | None


class TriageItemResponse(BaseModel):
    id: UUID
    organization_id: str
    item_type: str
    source_id: str
    title: str
    summary: str
    priority: str
    priority_signals: list[str]
    status: str
    assigned_to_user_id: str | None
    created_at: datetime
    resolved_at: datetime | None


class RespondQuestionBody(BaseModel):
    final_answer: str = Field(min_length=1)
    final_citations: list[dict] = Field(default_factory=list)


class UpdateTriageItemBody(BaseModel):
    action: str = Field(pattern=r"^(resolve|dismiss|assign|start)$")
    assigned_to_user_id: str | None = None


class ShareArtifactBody(BaseModel):
    artifact_id: str
    sponsor_org_id: str
    triage_question_id: UUID | None = None


class SharedItemResponse(BaseModel):
    id: UUID
    artifact_id: str
    sponsor_org_id: str
    shared_by_user_id: str
    shared_at: datetime
    triage_question_id: UUID | None


# ---------------------------------------------------------------------------
# Triage routes
# ---------------------------------------------------------------------------


@triage_router.post("/questions", response_model=SubmitQuestionResponse, status_code=201)
async def submit_question(
    body: SubmitQuestionBody,
    session: SessionDep,
    access: AccessDep,
    user: UserDep,
) -> SubmitQuestionResponse:
    if not access.is_sponsor:
        raise HTTPException(status_code=403, detail="Only sponsors can submit questions")

    use_case = SubmitQuestion(
        session=_CreateSessionAdapter(session),
        question_row_factory=TriageQuestionTable,
        item_row_factory=TriageItemTable,
    )
    result = await use_case.execute(
        SubmitQuestionCommand(
            asker_user_id=user.user_id,
            asker_org_id=access.org_id,
            target_company_org_id=body.target_company_org_id,
            question_text=body.question_text,
            asker_role=access.role.value,
        )
    )
    await session.commit()
    return SubmitQuestionResponse(
        question_id=result.question_id,
        triage_item_id=result.triage_item_id,
    )


@triage_router.get("/questions", response_model=list[QuestionResponse])
async def list_questions(
    session: SessionDep,
    user: UserDep,
    target_company_org_id: str | None = None,
) -> list[QuestionResponse]:
    use_case = ListQuestions(query=_ListQuestionsAdapter(session))
    questions = await use_case.execute(
        ListQuestionsCommand(
            asker_user_id=user.user_id,
            target_company_org_id=target_company_org_id,
        )
    )
    return [
        QuestionResponse(
            id=q.id,
            asker_user_id=q.asker_user_id,
            asker_org_id=q.asker_org_id,
            target_company_org_id=q.target_company_org_id,
            question_text=q.question_text,
            categories_detected=list(q.categories_detected),
            status=q.status.value,
            partial_answer=q.partial_answer,
            escalation_reason=q.escalation_reason,
            final_answer=q.final_answer,
            priority=q.priority.value,
            created_at=q.created_at,
            answered_at=q.answered_at,
        )
        for q in questions
    ]


@triage_router.get("/questions/{question_id}", response_model=QuestionResponse)
async def get_question(
    question_id: UUID,
    session: SessionDep,
) -> QuestionResponse:
    stmt = select(TriageQuestionTable).where(TriageQuestionTable.id == question_id)
    result = await session.execute(stmt)
    row = result.scalar_one_or_none()
    if row is None:
        raise HTTPException(status_code=404, detail="Question not found")
    q = _row_to_question(row)
    return QuestionResponse(
        id=q.id,
        asker_user_id=q.asker_user_id,
        asker_org_id=q.asker_org_id,
        target_company_org_id=q.target_company_org_id,
        question_text=q.question_text,
        categories_detected=list(q.categories_detected),
        status=q.status.value,
        partial_answer=q.partial_answer,
        escalation_reason=q.escalation_reason,
        final_answer=q.final_answer,
        priority=q.priority.value,
        created_at=q.created_at,
        answered_at=q.answered_at,
    )


@triage_router.post("/questions/{question_id}/respond", status_code=200)
async def respond_question(
    question_id: UUID,
    body: RespondQuestionBody,
    session: SessionDep,
    access: AccessDep,
) -> dict[str, str]:
    if access.is_sponsor:
        raise HTTPException(status_code=403, detail="Only operators can respond to questions")

    use_case = RespondQuestion(query=_RespondQuestionAdapter(session))
    try:
        await use_case.execute(
            RespondQuestionCommand(
                question_id=question_id,
                final_answer=body.final_answer,
                final_citations=body.final_citations,
            )
        )
    except TriageStateError as exc:
        raise HTTPException(status_code=409, detail=str(exc)) from exc
    except NotFoundError as exc:
        raise HTTPException(status_code=404, detail=str(exc)) from exc
    await session.commit()
    return {"status": "answered"}


@triage_router.get("/items", response_model=list[TriageItemResponse])
async def list_triage_items(
    session: SessionDep,
    access: AccessDep,
    item_type: TriageItemType | None = None,
    priority: TriagePriority | None = None,
    status: TriageItemStatus | None = None,
) -> list[TriageItemResponse]:
    if access.is_sponsor:
        raise HTTPException(status_code=403, detail="Only operators can view the triage queue")

    # Use the first company org ID as the org filter
    org_id = access.company_org_ids[0] if access.company_org_ids else access.org_id
    use_case = ListTriageItems(query=_ListTriageItemsAdapter(session))
    items = await use_case.execute(
        ListTriageItemsCommand(
            organization_id=org_id,
            item_type=item_type,
            priority=priority,
            status=status,
        )
    )
    return [
        TriageItemResponse(
            id=item.id,
            organization_id=item.organization_id,
            item_type=item.item_type.value,
            source_id=item.source_id,
            title=item.title,
            summary=item.summary,
            priority=item.priority.value,
            priority_signals=list(item.priority_signals),
            status=item.status.value,
            assigned_to_user_id=item.assigned_to_user_id,
            created_at=item.created_at,
            resolved_at=item.resolved_at,
        )
        for item in items
    ]


@triage_router.patch("/items/{item_id}")
async def update_triage_item(
    item_id: UUID,
    body: UpdateTriageItemBody,
    session: SessionDep,
    access: AccessDep,
) -> dict[str, str]:
    if access.is_sponsor:
        raise HTTPException(status_code=403, detail="Only operators can update triage items")

    use_case = UpdateTriageItem(query=_UpdateTriageItemAdapter(session))
    try:
        result = await use_case.execute(
            UpdateTriageItemCommand(
                item_id=item_id,
                action=body.action,
                assigned_to_user_id=body.assigned_to_user_id,
            )
        )
    except TriageStateError as exc:
        raise HTTPException(status_code=409, detail=str(exc)) from exc
    except NotFoundError as exc:
        raise HTTPException(status_code=404, detail=str(exc)) from exc
    await session.commit()
    return {"status": result.new_status}


# ---------------------------------------------------------------------------
# Sharing routes
# ---------------------------------------------------------------------------


@sharing_router.post("/items", response_model=SharedItemResponse, status_code=201)
async def share_artifact(
    body: ShareArtifactBody,
    session: SessionDep,
    access: AccessDep,
    user: UserDep,
) -> SharedItemResponse:
    if access.is_sponsor:
        raise HTTPException(status_code=403, detail="Only operators can share artifacts")

    use_case = ShareArtifact(
        session=_CreateSessionAdapter(session),
        row_factory=SharedItemTable,
    )
    result = await use_case.execute(
        ShareArtifactCommand(
            artifact_id=body.artifact_id,
            sponsor_org_id=body.sponsor_org_id,
            shared_by_user_id=user.user_id,
            triage_question_id=body.triage_question_id,
        )
    )
    await session.commit()
    return SharedItemResponse(
        id=result.shared_item_id,
        artifact_id=body.artifact_id,
        sponsor_org_id=body.sponsor_org_id,
        shared_by_user_id=user.user_id,
        shared_at=datetime.now(),
        triage_question_id=body.triage_question_id,
    )


@sharing_router.get("/items", response_model=list[SharedItemResponse])
async def list_shared_items(
    session: SessionDep,
    access: AccessDep,
    sponsor_org_id: str | None = None,
) -> list[SharedItemResponse]:
    org_id = sponsor_org_id or access.org_id
    adapter = _ListSharedAdapter(session)
    rows = await adapter.list_for_sponsor(sponsor_org_id=org_id)
    return [
        SharedItemResponse(
            id=row.id,
            artifact_id=row.artifact_id,
            sponsor_org_id=row.sponsor_org_id,
            shared_by_user_id=row.shared_by_user_id,
            shared_at=row.shared_at,
            triage_question_id=row.triage_question_id,
        )
        for row in rows
    ]


@sharing_router.delete("/items/{item_id}", status_code=204)
async def revoke_sharing(
    item_id: UUID,
    session: SessionDep,
    access: AccessDep,
) -> None:
    if access.is_sponsor:
        raise HTTPException(status_code=403, detail="Only operators can revoke sharing")

    stmt = select(SharedItemTable).where(SharedItemTable.id == item_id)
    result = await session.execute(stmt)
    row = result.scalar_one_or_none()
    if row is None:
        raise HTTPException(status_code=404, detail="Shared item not found")
    await session.delete(row)
    await session.commit()
```

- [ ] **Step 2: Register routers in main.py**

Open `apps/api/src/prescient/main.py` and add after the last router import:

```python
from prescient.triage.api.routes import triage_router, sharing_router
```

And in the `create_app` function, add after the last `app.include_router(...)`:

```python
    app.include_router(triage_router)
    app.include_router(sharing_router)
```

- [ ] **Step 3: Verify app starts**

Run: `cd apps/api && uv run python -c "from prescient.main import app; print('OK')"`
Expected: prints `OK`

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/prescient/triage/api/ apps/api/src/prescient/main.py
git commit -m "feat(triage): add triage + sharing API routes with AccessContext enforcement"
```

---

### Task 6: Briefing Integration

**Files:**
- Modify: `apps/api/src/prescient/briefing/infrastructure/source_adapter.py`

- [ ] **Step 1: Add get_triage_items method to DatabaseSourceAdapter**

Open `apps/api/src/prescient/briefing/infrastructure/source_adapter.py` and add a new method to the `DatabaseSourceAdapter` class. Add the necessary import at the top:

```python
from prescient.triage.infrastructure.tables.triage_item import TriageItemTable
```

Then add this method to the class:

```python
    # ------------------------------------------------------------------
    # Triage items — high-priority pending items
    # ------------------------------------------------------------------

    async def get_triage_items(self, *, organization_id: str) -> list[BriefingItem]:
        stmt = (
            select(TriageItemTable)
            .where(
                TriageItemTable.organization_id == organization_id,
                TriageItemTable.status.in_(["pending", "in_progress"]),
                TriageItemTable.priority.in_(["critical", "high"]),
            )
            .order_by(TriageItemTable.created_at.desc())
        )
        result = await self._session.execute(stmt)
        rows = list(result.scalars().all())

        items: list[BriefingItem] = []
        for row in rows:
            priority = row.priority.lower()
            if priority == "critical":
                relevance = 0.9
            else:
                relevance = 0.7

            items.append(
                BriefingItem(
                    id=str(uuid.uuid4()),
                    title=row.title,
                    summary=row.summary,
                    action_type=ActionType.DECIDE,
                    source=ItemSource.TRIAGE,
                    source_artifact_id=str(row.id),
                    relevance_score=relevance,
                    created_at=row.created_at,
                )
            )
        return items
```

Note: This requires adding `TRIAGE = "triage"` to `ItemSource` enum in `apps/api/src/prescient/briefing/domain/entities/briefing_item.py` if it doesn't exist yet.

- [ ] **Step 2: Add TRIAGE to ItemSource if needed**

Open `apps/api/src/prescient/briefing/domain/entities/briefing_item.py` and add to the `ItemSource` enum:

```python
    TRIAGE = "triage"
```

- [ ] **Step 3: Run existing briefing tests to confirm no regressions**

Run: `cd apps/api && uv run pytest tests/unit/ -v --tb=short`
Expected: All tests pass.

- [ ] **Step 4: Commit**

```bash
git add apps/api/src/prescient/briefing/ apps/api/src/prescient/triage/
git commit -m "feat(briefing): integrate triage items into morning brief — high-priority items surface as DECIDE"
```

---

### Task 7: Run Full Test Suite and Lint

**Files:** None (verification only)

- [ ] **Step 1: Run all unit tests**

Run: `cd apps/api && uv run pytest tests/unit/ -v --tb=short`
Expected: All tests pass.

- [ ] **Step 2: Run linting**

Run: `cd apps/api && uv run ruff check src/prescient/triage/`
Expected: No lint errors.

- [ ] **Step 3: Run type checking**

Run: `cd apps/api && uv run mypy src/prescient/triage/ --ignore-missing-imports`
Expected: No type errors.

- [ ] **Step 4: Fix any issues and commit**

```bash
git add -A
git commit -m "fix: resolve lint/type issues from Phase 5b"
```

---

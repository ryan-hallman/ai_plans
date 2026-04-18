# Private Chat Import And Top-Up Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build reusable Claude/Codex import logic in `prescient_os`, plus private top-up/snapshot runners in `prescient_os_data`, so private chat history can be mirrored incrementally and frozen into benchmark-ready snapshots.

**Architecture:** Keep provider parsing, normalization, dedup, and snapshot-building in `prescient_os` under the existing `corpus` package. Keep private raw mirrors, import state, machine-local source config, and runner scripts in `prescient_os_data`. Top-up runs mutate the raw mirror and normalized layer; benchmark snapshots are cut as immutable outputs compatible with the existing private-corpus loader.

**Tech Stack:** Python 3.13, Pydantic v2, PyYAML, Typer, pytest, shell runner scripts, existing `private_chat_models` / `private_chat_loader`

---

## File Structure

### Product repo: `/home/rhallman/Projects/prescient_os`

**Create:**
- `/home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/corpus/chat_import_models.py`
  - import source config, discovered-session metadata, import-state models, snapshot-cut request/summary models
- `/home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/corpus/chat_import_paths.py`
  - path helpers for raw mirror, normalized session outputs, import-state manifest, and frozen snapshot roots
- `/home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/corpus/chat_import_claude.py`
  - Claude archive + live Claude discovery and normalization
- `/home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/corpus/chat_import_codex.py`
  - live Codex discovery and normalization
- `/home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/corpus/chat_import_service.py`
  - top-up orchestration, dedup/drift handling, import-state updates, snapshot cutting
- `/home/rhallman/Projects/prescient_os/tests/unit/test_chat_import_models.py`
- `/home/rhallman/Projects/prescient_os/tests/unit/test_chat_import_claude.py`
- `/home/rhallman/Projects/prescient_os/tests/unit/test_chat_import_codex.py`
- `/home/rhallman/Projects/prescient_os/tests/unit/test_chat_import_service.py`
- `/home/rhallman/Projects/prescient_os/tests/integration/test_chat_import_cli.py`
- `/home/rhallman/Projects/prescient_os/tests/fixtures/chat_import/claude_archive_session.jsonl`
- `/home/rhallman/Projects/prescient_os/tests/fixtures/chat_import/claude_live_project_session.jsonl`
- `/home/rhallman/Projects/prescient_os/tests/fixtures/chat_import/claude_live_history.jsonl`
- `/home/rhallman/Projects/prescient_os/tests/fixtures/chat_import/codex_live_session.jsonl`
- `/home/rhallman/Projects/prescient_os/tests/fixtures/chat_import/codex_history.jsonl`

**Modify:**
- `/home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/cli.py`
  - add importer/snapshot CLI commands

### Private data repo: `/home/rhallman/Projects/prescient_os_data`

**Create:**
- `/home/rhallman/Projects/prescient_os_data/config/chat_import_sources.example.yaml`
  - machine-local import source config example
- `/home/rhallman/Projects/prescient_os_data/imports/README.md`
  - explains raw mirror, normalized layer, import state, and snapshots
- `/home/rhallman/Projects/prescient_os_data/scripts/top_up_private_chats.sh`
  - wrapper around product-repo CLI top-up command
- `/home/rhallman/Projects/prescient_os_data/scripts/cut_private_chat_snapshot.sh`
  - wrapper around product-repo CLI snapshot-cut command

**Notes:**
- Do not commit real private chat content in this slice.
- Do not add automatic memo generation.
- Do not add retrieval/eval logic beyond frozen snapshot compatibility.
- Keep provider parsing focused on the real source shapes we observed:
  - Claude archive and live Claude use Claude project-session JSONL
  - Codex uses session JSONL under `.codex/sessions/...` plus `history.jsonl`

---

### Task 1: Add import models and path helpers in the product repo

**Files:**
- Create: `/home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/corpus/chat_import_models.py`
- Create: `/home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/corpus/chat_import_paths.py`
- Create: `/home/rhallman/Projects/prescient_os/tests/unit/test_chat_import_models.py`

- [ ] **Step 1: Write the failing model/path tests**

```python
from datetime import datetime, timezone
import tarfile
from pathlib import Path

import pytest
from pydantic import ValidationError

from prescient_benchmark.corpus.chat_import_models import (
    ChatImportSourceConfig,
    ChatImportState,
    DiscoveredChatSession,
    ImportedChatSessionRecord,
)
from prescient_benchmark.corpus.chat_import_paths import (
    import_state_path,
    normalized_session_path,
    raw_session_dir,
    raw_session_main_path,
    snapshot_root,
)


def test_chat_import_source_config_accepts_supported_source_kinds() -> None:
    config = ChatImportSourceConfig.model_validate(
        {
            "source_kind": "claude_archive",
            "source_label": "other-workstation-archive",
            "source_path": "/tmp/archive.tar.gz",
        }
    )

    assert config.source_kind == "claude_archive"


def test_discovered_chat_session_requires_timezone_aware_exported_at() -> None:
    with pytest.raises(ValidationError, match="timezone-aware"):
        DiscoveredChatSession.model_validate(
            {
                "source_kind": "claude_live",
                "source_provider": "anthropic",
                "source_app": "claude_code",
                "source_label": "local",
                "session_id": "session-001",
                "exported_at": "2026-04-18T12:00:00",
                "raw_source_relpath": "raw/claude/live/session-001.jsonl",
                "content_hash": "sha256:test",
                "messages": [
                    {"index": 0, "role": "user", "content_text": "hello"},
                ],
            }
        )


def test_chat_import_state_indexes_records_by_provider_and_session_id() -> None:
    state = ChatImportState.model_validate(
        {
            "sessions": [
                {
                    "source_kind": "codex_live",
                    "source_provider": "openai",
                    "session_id": "session-001",
                    "source_label": "local-codex",
                    "raw_relpath": "raw/codex/live/session-001.jsonl",
                    "normalized_relpath": "normalized/sessions/openai__session-001.yaml",
                    "content_hash": "sha256:one",
                    "first_seen_at": "2026-04-18T20:00:00Z",
                    "last_seen_at": "2026-04-18T20:00:00Z",
                }
            ]
        }
    )

    record = state.record_for("openai", "session-001")
    assert record is not None
    assert record.source_label == "local-codex"


def test_path_helpers_build_data_repo_relative_locations() -> None:
    data_root = Path("/tmp/prescient_os_data")

    assert raw_session_dir(
        data_root,
        provider="claude",
        source_label="other-workstation-archive",
        session_id="abc",
    ).as_posix().endswith(
        "raw/claude/other-workstation-archive/abc"
    )
    assert raw_session_main_path(
        data_root,
        provider="claude",
        source_label="other-workstation-archive",
        session_id="abc",
    ).as_posix().endswith(
        "raw/claude/other-workstation-archive/abc/main.jsonl"
    )
    assert normalized_session_path(data_root, provider="anthropic", session_id="abc").as_posix().endswith(
        "normalized/sessions/anthropic__abc.yaml"
    )
    assert import_state_path(data_root).as_posix().endswith("imports/chat_sources_manifest.yaml")
    assert snapshot_root(data_root, "founder_v1").as_posix().endswith("snapshots/founder_v1")
```

- [ ] **Step 2: Run the model/path tests to verify they fail**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_chat_import_models.py -q
```

Expected: FAIL with missing `chat_import_models` / `chat_import_paths` modules.

- [ ] **Step 3: Implement the import models**

```python
# /home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/corpus/chat_import_models.py
from __future__ import annotations

from datetime import UTC, datetime
from typing import Literal

from pydantic import BaseModel, ConfigDict, Field, StrictStr, field_validator

from prescient_benchmark.corpus.private_chat_models import ChatMessage, NormalizedChatSession


class ChatImportSourceConfig(BaseModel):
    model_config = ConfigDict(extra="forbid")

    source_kind: Literal["claude_archive", "claude_live", "codex_live"]
    source_label: StrictStr
    source_path: StrictStr


class DiscoveredChatSession(NormalizedChatSession):
    model_config = ConfigDict(extra="forbid")

    source_kind: Literal["claude_archive", "claude_live", "codex_live"]
    source_label: StrictStr
    raw_source_relpath: StrictStr


class ImportedChatSessionRecord(BaseModel):
    model_config = ConfigDict(extra="forbid")

    source_kind: Literal["claude_archive", "claude_live", "codex_live"]
    source_provider: StrictStr
    session_id: StrictStr
    source_label: StrictStr
    raw_relpath: StrictStr
    normalized_relpath: StrictStr
    content_hash: StrictStr
    first_seen_at: datetime
    last_seen_at: datetime

    @field_validator("first_seen_at", "last_seen_at")
    @classmethod
    def _must_be_aware(cls, value: datetime) -> datetime:
        if value.tzinfo is None or value.utcoffset() is None:
            raise ValueError("timestamps must be timezone-aware")
        return value.astimezone(UTC)


class ChatImportState(BaseModel):
    model_config = ConfigDict(extra="forbid")

    sessions: list[ImportedChatSessionRecord] = Field(default_factory=list)

    def record_for(self, source_provider: str, session_id: str) -> ImportedChatSessionRecord | None:
        for record in self.sessions:
            if record.source_provider == source_provider and record.session_id == session_id:
                return record
        return None
```

- [ ] **Step 4: Implement the path helpers**

```python
# /home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/corpus/chat_import_paths.py
from __future__ import annotations

from pathlib import Path


def import_state_path(data_root: Path) -> Path:
    return data_root / "imports" / "chat_sources_manifest.yaml"


def raw_session_dir(data_root: Path, *, provider: str, source_label: str, session_id: str) -> Path:
    return data_root / "raw" / provider / source_label / session_id


def raw_session_main_path(data_root: Path, *, provider: str, source_label: str, session_id: str) -> Path:
    return raw_session_dir(
        data_root,
        provider=provider,
        source_label=source_label,
        session_id=session_id,
    ) / "main.jsonl"


def normalized_session_path(data_root: Path, *, provider: str, session_id: str) -> Path:
    return data_root / "normalized" / "sessions" / f"{provider}__{session_id}.yaml"


def snapshot_root(data_root: Path, snapshot_id: str) -> Path:
    return data_root / "snapshots" / snapshot_id
```

- [ ] **Step 5: Run the model/path tests to verify they pass**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_chat_import_models.py -q
```

Expected: PASS with `4 passed`.

- [ ] **Step 6: Commit Task 1**

```bash
git add apps/api/src/prescient_benchmark/corpus/chat_import_models.py \
  apps/api/src/prescient_benchmark/corpus/chat_import_paths.py \
  tests/unit/test_chat_import_models.py
git commit -m "feat: add chat import state models"
```

### Task 2: Add Claude archive and live Claude discovery/normalization

**Files:**
- Create: `/home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/corpus/chat_import_claude.py`
- Create: `/home/rhallman/Projects/prescient_os/tests/unit/test_chat_import_claude.py`
- Create: `/home/rhallman/Projects/prescient_os/tests/fixtures/chat_import/claude_archive_session.jsonl`
- Create: `/home/rhallman/Projects/prescient_os/tests/fixtures/chat_import/claude_live_project_session.jsonl`
- Create: `/home/rhallman/Projects/prescient_os/tests/fixtures/chat_import/claude_live_history.jsonl`

- [ ] **Step 1: Add compact Claude fixtures based on the observed formats**

```json
// /home/rhallman/Projects/prescient_os/tests/fixtures/chat_import/claude_live_history.jsonl
{"display":"Can you review this new plan?","timestamp":1776382920214,"project":"/home/rhallman/Projects/prescient_os","sessionId":"759fb3d3-bfa9-4d40-918a-32d2e5f3b467"}
{"display":"Didn't mean to interrupt, continue","timestamp":1776382975763,"project":"/home/rhallman/Projects/prescient_os","sessionId":"759fb3d3-bfa9-4d40-918a-32d2e5f3b467"}
```

```json
// /home/rhallman/Projects/prescient_os/tests/fixtures/chat_import/claude_live_project_session.jsonl
{"type":"permission-mode","permissionMode":"bypassPermissions","sessionId":"759fb3d3-bfa9-4d40-918a-32d2e5f3b467"}
{"parentUuid":null,"isSidechain":false,"type":"user","message":{"role":"user","content":"Can you review this new plan?"},"sessionId":"759fb3d3-bfa9-4d40-918a-32d2e5f3b467","timestamp":"2026-04-16T23:10:00.000Z"}
{"parentUuid":"parent-001","isSidechain":false,"type":"assistant","message":{"role":"assistant","content":[{"type":"text","text":"Yes. I think the plan is still too broad."}]},"sessionId":"759fb3d3-bfa9-4d40-918a-32d2e5f3b467","timestamp":"2026-04-16T23:10:05.000Z"}
{"parentUuid":"parent-002","isSidechain":true,"type":"user","message":{"role":"user","content":"Subagent content should be ignored."},"sessionId":"759fb3d3-bfa9-4d40-918a-32d2e5f3b467","timestamp":"2026-04-16T23:10:10.000Z"}
```

```json
// /home/rhallman/Projects/prescient_os/tests/fixtures/chat_import/claude_archive_session.jsonl
{"type":"permission-mode","permissionMode":"default","sessionId":"8e0aa209-394a-4345-b418-deeb3ebcc287"}
{"parentUuid":null,"isSidechain":false,"type":"user","message":{"role":"user","content":"png files from playwright keep getting left in the root folder and accidentally committed. Can we solve that with a pre-commit hook?"},"sessionId":"8e0aa209-394a-4345-b418-deeb3ebcc287","timestamp":"2026-04-14T15:07:49.435Z"}
{"parentUuid":"user-001","isSidechain":false,"type":"assistant","message":{"role":"assistant","content":[{"type":"text","text":"Yes. A root-level PNG pre-commit guard is a good fit."}]},"sessionId":"8e0aa209-394a-4345-b418-deeb3ebcc287","timestamp":"2026-04-14T15:08:54.310Z"}
{"parentUuid":"assistant-001","isSidechain":false,"attachment":{"type":"hook_success","stdout":"ignore me"},"sessionId":"8e0aa209-394a-4345-b418-deeb3ebcc287","timestamp":"2026-04-14T15:08:54.500Z"}
```

- [ ] **Step 2: Write the failing Claude adapter tests**

```python
from pathlib import Path

from prescient_benchmark.corpus.chat_import_claude import (
    discover_claude_archive_sessions,
    discover_claude_live_sessions,
)


def test_discover_claude_archive_sessions_normalizes_main_thread_messages(tmp_path: Path) -> None:
    archive_path = tmp_path / "claude-export.tar.gz"
    session_fixture = Path("tests/fixtures/chat_import/claude_archive_session.jsonl")
    with tarfile.open(archive_path, "w:gz") as archive:
        archive.add(
            session_fixture,
            arcname="-home-rhallman-Projects-prescient-os/8e0aa209-394a-4345-b418-deeb3ebcc287.jsonl",
        )

    sessions = discover_claude_archive_sessions(
        archive_path=archive_path,
        source_label="other-workstation-archive",
    )

    assert len(sessions) == 1
    assert sessions[0].source_kind == "claude_archive"
    assert sessions[0].source_provider == "anthropic"
    assert [message.role for message in sessions[0].messages] == ["user", "assistant"]
    assert "hook_success" not in sessions[0].messages[0].content_text


def test_discover_claude_live_sessions_uses_project_jsonl_and_history_title(tmp_path: Path) -> None:
    live_root = tmp_path / "claude"
    (live_root / "projects" / "-home-rhallman-Projects-prescient-os").mkdir(parents=True)
    (live_root / "history.jsonl").write_text(
        Path("tests/fixtures/chat_import/claude_live_history.jsonl").read_text(encoding="utf-8"),
        encoding="utf-8",
    )
    (live_root / "projects" / "-home-rhallman-Projects-prescient-os" / "759fb3d3-bfa9-4d40-918a-32d2e5f3b467.jsonl").write_text(
        Path("tests/fixtures/chat_import/claude_live_project_session.jsonl").read_text(encoding="utf-8"),
        encoding="utf-8",
    )

    sessions = discover_claude_live_sessions(
        claude_root=live_root,
        source_label="local-claude",
    )

    assert len(sessions) == 1
    assert sessions[0].title == "Can you review this new plan?"
    assert [message.role for message in sessions[0].messages] == ["user", "assistant"]
```

- [ ] **Step 3: Run the Claude tests to verify they fail**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_chat_import_claude.py -q
```

Expected: FAIL with missing `chat_import_claude` module.

- [ ] **Step 4: Implement Claude discovery and normalization**

```python
# /home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/corpus/chat_import_claude.py
from __future__ import annotations

import json
import tarfile
from datetime import UTC, datetime
from pathlib import Path

from prescient_benchmark.corpus.chat_import_models import DiscoveredChatSession


def discover_claude_archive_sessions(*, archive_path: Path, source_label: str) -> list[DiscoveredChatSession]:
    sessions: list[DiscoveredChatSession] = []
    with tarfile.open(archive_path, "r:gz") as archive:
        for member in sorted(archive.getmembers(), key=lambda item: item.name):
            if not member.isfile() or not member.name.endswith(".jsonl"):
                continue
            if "/subagents/" in member.name or "/tool-results/" in member.name:
                continue
            file_obj = archive.extractfile(member)
            if file_obj is None:
                continue
            sessions.append(
                _normalize_claude_jsonl_text(
                    jsonl_text=file_obj.read().decode("utf-8"),
                    raw_source_relpath=member.name,
                    source_kind="claude_archive",
                    source_label=source_label,
                    title_overrides={},
                )
            )
    return sessions


def discover_claude_live_sessions(*, claude_root: Path, source_label: str) -> list[DiscoveredChatSession]:
    title_overrides = _load_claude_history_titles(claude_root / "history.jsonl")
    session_files = sorted((claude_root / "projects").rglob("*.jsonl"))
    return [
        _normalize_claude_jsonl_text(
            jsonl_text=session_file.read_text(encoding="utf-8"),
            raw_source_relpath=session_file.relative_to(claude_root).as_posix(),
            source_kind="claude_live",
            source_label=source_label,
            title_overrides=title_overrides,
        )
        for session_file in session_files
        if session_file.name != "MEMORY.md"
    ]


def _load_claude_history_titles(history_path: Path) -> dict[str, str]:
    if not history_path.exists():
        return {}
    titles: dict[str, str] = {}
    for line in history_path.read_text(encoding="utf-8").splitlines():
        payload = json.loads(line)
        session_id = payload.get("sessionId")
        display = payload.get("display")
        if session_id and display and session_id not in titles:
            titles[session_id] = display.strip()
    return titles


def _normalize_claude_jsonl_text(
    *,
    jsonl_text: str,
    raw_source_relpath: str,
    source_kind: str,
    source_label: str,
    title_overrides: dict[str, str],
) -> DiscoveredChatSession:
    raw_lines = jsonl_text.splitlines()
    records = [json.loads(line) for line in raw_lines if line.strip()]
    session_id = next(record["sessionId"] for record in records if record.get("sessionId"))
    messages = []
    for record in records:
        if record.get("isSidechain"):
            continue
        if record.get("type") == "user" and isinstance(record.get("message"), dict):
            messages.append({"index": len(messages), "role": "user", "timestamp": record["timestamp"], "content_text": record["message"]["content"]})
        if record.get("type") == "assistant" and isinstance(record.get("message"), dict):
            text_parts = [item["text"] for item in record["message"].get("content", []) if item.get("type") == "text"]
            if text_parts:
                messages.append({"index": len(messages), "role": "assistant", "timestamp": record["timestamp"], "content_text": "\n".join(text_parts)})
    exported_at = max(datetime.fromisoformat(message["timestamp"].replace("Z", "+00:00")) for message in messages)
    return DiscoveredChatSession.model_validate(
        {
            "source_kind": source_kind,
            "source_provider": "anthropic",
            "source_app": "claude_code",
            "source_label": source_label,
            "session_id": session_id,
            "title": title_overrides.get(session_id),
            "exported_at": exported_at.astimezone(UTC),
            "raw_filename": "main.jsonl",
            "raw_source_relpath": raw_source_relpath,
            "content_hash": "sha256:replace-during-top-up",
            "provider_metadata": {},
            "messages": messages,
        }
    )
```

- [ ] **Step 5: Run the Claude tests to verify they pass**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_chat_import_claude.py -q
```

Expected: PASS with `2 passed`.

- [ ] **Step 6: Commit Task 2**

```bash
git add apps/api/src/prescient_benchmark/corpus/chat_import_claude.py \
  tests/unit/test_chat_import_claude.py \
  tests/fixtures/chat_import/claude_archive_session.jsonl \
  tests/fixtures/chat_import/claude_live_project_session.jsonl \
  tests/fixtures/chat_import/claude_live_history.jsonl
git commit -m "feat: add claude chat import adapters"
```

### Task 3: Add Codex discovery and normalization

**Files:**
- Create: `/home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/corpus/chat_import_codex.py`
- Create: `/home/rhallman/Projects/prescient_os/tests/unit/test_chat_import_codex.py`
- Create: `/home/rhallman/Projects/prescient_os/tests/fixtures/chat_import/codex_live_session.jsonl`
- Create: `/home/rhallman/Projects/prescient_os/tests/fixtures/chat_import/codex_history.jsonl`

- [ ] **Step 1: Add compact Codex fixtures**

```json
// /home/rhallman/Projects/prescient_os/tests/fixtures/chat_import/codex_history.jsonl
{"session_id":"019d8e5d-07ce-7750-8656-411c3e8426ab","ts":1776210822,"text":"can you load chrome mcp and navigate to the site?"}
{"session_id":"019d8e5d-07ce-7750-8656-411c3e8426ab","ts":1776211155,"text":"codex mcp add chrome-devtools -- npx chrome-devtools-mcp@latest"}
```

```json
// /home/rhallman/Projects/prescient_os/tests/fixtures/chat_import/codex_live_session.jsonl
{"timestamp":"2026-04-14T23:53:42.558Z","type":"session_meta","payload":{"id":"019d8e5d-07ce-7750-8656-411c3e8426ab","timestamp":"2026-04-14T23:39:14.898Z","model_provider":"openai","source":"cli"}}
{"timestamp":"2026-04-14T23:59:15.927Z","type":"response_item","payload":{"type":"message","role":"user","content":[{"type":"input_text","text":"codex mcp add chrome-devtools -- npx chrome-devtools-mcp@latest"}]}}
{"timestamp":"2026-04-14T23:59:20.405Z","type":"response_item","payload":{"type":"message","role":"assistant","content":[{"type":"output_text","text":"The server was added. I’m checking whether it now appears in the configured MCP resources so I know whether it’s usable in this session."}]}}
{"timestamp":"2026-04-14T23:59:21.902Z","type":"response_item","payload":{"type":"function_call","name":"list_mcp_resource_templates","arguments":"{}","call_id":"call_rizX"}}
```

- [ ] **Step 2: Write the failing Codex adapter tests**

```python
from pathlib import Path

from prescient_benchmark.corpus.chat_import_codex import discover_codex_live_sessions


def test_discover_codex_live_sessions_reads_session_jsonl_and_skips_tool_records(tmp_path: Path) -> None:
    codex_root = tmp_path / "codex"
    (codex_root / "sessions" / "2026" / "04" / "14").mkdir(parents=True)
    (codex_root / "history.jsonl").write_text(
        Path("tests/fixtures/chat_import/codex_history.jsonl").read_text(encoding="utf-8"),
        encoding="utf-8",
    )
    (codex_root / "sessions" / "2026" / "04" / "14" / "rollout-2026-04-14T16-39-14-019d8e5d-07ce-7750-8656-411c3e8426ab.jsonl").write_text(
        Path("tests/fixtures/chat_import/codex_live_session.jsonl").read_text(encoding="utf-8"),
        encoding="utf-8",
    )

    sessions = discover_codex_live_sessions(codex_root=codex_root, source_label="local-codex")

    assert len(sessions) == 1
    assert sessions[0].source_provider == "openai"
    assert [message.role for message in sessions[0].messages] == ["user", "assistant"]
    assert "function_call" not in sessions[0].messages[0].content_text
    assert sessions[0].title == "can you load chrome mcp and navigate to the site?"
```

- [ ] **Step 3: Run the Codex tests to verify they fail**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_chat_import_codex.py -q
```

Expected: FAIL with missing `chat_import_codex` module.

- [ ] **Step 4: Implement Codex discovery and normalization**

```python
# /home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/corpus/chat_import_codex.py
from __future__ import annotations

import json
from datetime import UTC, datetime
from pathlib import Path

from prescient_benchmark.corpus.chat_import_models import DiscoveredChatSession


def discover_codex_live_sessions(*, codex_root: Path, source_label: str) -> list[DiscoveredChatSession]:
    title_overrides = _load_codex_history_titles(codex_root / "history.jsonl")
    session_files = sorted((codex_root / "sessions").rglob("*.jsonl"))
    return [
        _normalize_codex_session(
            session_file=session_file,
            source_label=source_label,
            title_overrides=title_overrides,
        )
        for session_file in session_files
    ]


def _load_codex_history_titles(history_path: Path) -> dict[str, str]:
    if not history_path.exists():
        return {}
    titles: dict[str, str] = {}
    for line in history_path.read_text(encoding="utf-8").splitlines():
        payload = json.loads(line)
        session_id = payload.get("session_id")
        text = payload.get("text")
        if session_id and text and session_id not in titles:
            titles[session_id] = text.strip()
    return titles


def _normalize_codex_session(
    *,
    session_file: Path,
    source_label: str,
    title_overrides: dict[str, str],
) -> DiscoveredChatSession:
    records = [json.loads(line) for line in session_file.read_text(encoding="utf-8").splitlines() if line.strip()]
    meta = next(record for record in records if record.get("type") == "session_meta")
    session_id = meta["payload"]["id"]
    messages = []
    for record in records:
        if record.get("type") != "response_item":
            continue
        payload = record.get("payload", {})
        if payload.get("type") != "message":
            continue
        role = payload.get("role")
        text_parts = []
        for item in payload.get("content", []):
            if item.get("type") in {"input_text", "output_text"}:
                text_parts.append(item["text"])
        if role in {"user", "assistant"} and text_parts:
            messages.append(
                {
                    "index": len(messages),
                    "role": role,
                    "timestamp": record["timestamp"],
                    "content_text": "\n".join(text_parts),
                }
            )
    exported_at = max(datetime.fromisoformat(message["timestamp"].replace("Z", "+00:00")) for message in messages)
    return DiscoveredChatSession.model_validate(
        {
            "source_kind": "codex_live",
            "source_provider": "openai",
            "source_app": "codex_cli",
            "source_label": source_label,
            "session_id": session_id,
            "title": title_overrides.get(session_id),
            "exported_at": exported_at.astimezone(UTC),
            "raw_filename": session_file.name,
            "raw_source_relpath": session_file.relative_to(codex_root).as_posix(),
            "content_hash": "sha256:replace-during-top-up",
            "provider_metadata": {"model_provider": meta["payload"].get("model_provider")},
            "messages": messages,
        }
    )
```

- [ ] **Step 5: Run the Codex tests to verify they pass**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_chat_import_codex.py -q
```

Expected: PASS with `1 passed`.

- [ ] **Step 6: Commit Task 3**

```bash
git add apps/api/src/prescient_benchmark/corpus/chat_import_codex.py \
  tests/unit/test_chat_import_codex.py \
  tests/fixtures/chat_import/codex_live_session.jsonl \
  tests/fixtures/chat_import/codex_history.jsonl
git commit -m "feat: add codex chat import adapters"
```

### Task 4: Add top-up orchestration, snapshot cutting, and product CLI commands

**Files:**
- Create: `/home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/corpus/chat_import_service.py`
- Modify: `/home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/cli.py`
- Create: `/home/rhallman/Projects/prescient_os/tests/unit/test_chat_import_service.py`
- Create: `/home/rhallman/Projects/prescient_os/tests/integration/test_chat_import_cli.py`

- [ ] **Step 1: Write the failing top-up/snapshot tests**

```python
from pathlib import Path

import yaml

from prescient_benchmark.corpus.chat_import_service import cut_private_chat_snapshot, top_up_private_chat_sources


def test_top_up_private_chat_sources_imports_new_sessions_and_skips_unchanged(tmp_path: Path) -> None:
    data_root = tmp_path / "prescient_os_data"
    source_root = tmp_path / "sources"
    (source_root / "codex" / "sessions" / "2026" / "04" / "14").mkdir(parents=True)
    (source_root / "codex" / "history.jsonl").write_text(
        Path("tests/fixtures/chat_import/codex_history.jsonl").read_text(encoding="utf-8"),
        encoding="utf-8",
    )
    (source_root / "codex" / "sessions" / "2026" / "04" / "14" / "session.jsonl").write_text(
        Path("tests/fixtures/chat_import/codex_live_session.jsonl").read_text(encoding="utf-8"),
        encoding="utf-8",
    )

    config = [
        {
            "source_kind": "codex_live",
            "source_label": "local-codex",
            "source_path": str(source_root / "codex"),
        }
    ]

    first = top_up_private_chat_sources(data_root=data_root, source_configs=config)
    second = top_up_private_chat_sources(data_root=data_root, source_configs=config)

    assert first.imported == 1
    assert second.unchanged == 1


def test_cut_private_chat_snapshot_writes_loader_compatible_manifest(tmp_path: Path) -> None:
    data_root = tmp_path / "prescient_os_data"
    (data_root / "normalized" / "sessions").mkdir(parents=True)
    (data_root / "normalized" / "sessions" / "openai__session-001.yaml").write_text(
        yaml.safe_dump(
            {
                "session_id": "session-001",
                "source_provider": "openai",
                "source_app": "codex_cli",
                "exported_at": "2026-04-18T20:00:00Z",
                "raw_filename": "raw/codex/codex_live/session-001.jsonl",
                "content_hash": "sha256:test",
                "messages": [{"index": 0, "role": "user", "content_text": "Hello"}],
            },
            sort_keys=False,
        ),
        encoding="utf-8",
    )
    (data_root / "imports").mkdir(parents=True)
    (data_root / "imports" / "chat_sources_manifest.yaml").write_text(
        yaml.safe_dump(
            {
                "sessions": [
                    {
                        "source_kind": "codex_live",
                        "source_provider": "openai",
                        "session_id": "session-001",
                        "source_label": "local-codex",
                        "raw_relpath": "raw/codex/codex_live/session-001.jsonl",
                        "normalized_relpath": "normalized/sessions/openai__session-001.yaml",
                        "content_hash": "sha256:test",
                        "first_seen_at": "2026-04-18T20:00:00Z",
                        "last_seen_at": "2026-04-18T20:00:00Z",
                    }
                ]
            },
            sort_keys=False,
        ),
        encoding="utf-8",
    )

    snapshot = cut_private_chat_snapshot(
        data_root=data_root,
        snapshot_id="founder_v1",
        source_workspace="prescient_os_data",
    )

    manifest = yaml.safe_load((snapshot / "manifest.yaml").read_text(encoding="utf-8"))
    assert manifest["snapshot_id"] == "founder_v1"
    assert manifest["sessions"][0]["normalized_filename"] == "normalized/sessions/openai__session-001.yaml"
```

- [ ] **Step 2: Run the top-up/snapshot tests to verify they fail**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_chat_import_service.py -q
```

Expected: FAIL with missing `chat_import_service` module.

- [ ] **Step 3: Implement top-up orchestration and snapshot cutting**

```python
# /home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/corpus/chat_import_service.py
from __future__ import annotations

import hashlib
import shutil
from dataclasses import dataclass
from datetime import UTC, datetime
from pathlib import Path

import yaml

from prescient_benchmark.corpus.chat_import_claude import (
    discover_claude_archive_sessions,
    discover_claude_live_sessions,
)
from prescient_benchmark.corpus.chat_import_codex import discover_codex_live_sessions
from prescient_benchmark.corpus.chat_import_models import ChatImportSourceConfig, ChatImportState, ImportedChatSessionRecord
from prescient_benchmark.corpus.chat_import_paths import (
    import_state_path,
    normalized_session_path,
    raw_session_dir,
    raw_session_main_path,
    snapshot_root,
)
from prescient_benchmark.corpus.private_chat_models import PrivateCorpusManifest


@dataclass(slots=True)
class TopUpResult:
    imported: int = 0
    updated: int = 0
    unchanged: int = 0


def top_up_private_chat_sources(*, data_root: Path, source_configs: list[dict[str, object]]) -> TopUpResult:
    configs = [ChatImportSourceConfig.model_validate(item) for item in source_configs]
    state = _load_import_state(data_root)
    result = TopUpResult()
    for config in configs:
        for session in _discover_sessions(config):
            raw_dir = raw_session_dir(
                data_root,
                provider="claude" if session.source_provider == "anthropic" else "codex",
                source_label=config.source_label,
                session_id=session.session_id,
            )
            raw_path = raw_session_main_path(
                data_root,
                provider="claude" if session.source_provider == "anthropic" else "codex",
                source_label=config.source_label,
                session_id=session.session_id,
            )
            normalized_path = normalized_session_path(
                data_root,
                provider=session.source_provider,
                session_id=session.session_id,
            )
            content_hash = _materialize_raw_artifacts(
                source_config=config,
                discovered_session=session,
                raw_dir=raw_dir,
                raw_main_path=raw_path,
            )
            prior = state.record_for(session.source_provider, session.session_id)
            if prior is None:
                result.imported += 1
            elif prior.content_hash == content_hash:
                result.unchanged += 1
                continue
            else:
                result.updated += 1

            normalized_path.parent.mkdir(parents=True, exist_ok=True)
            normalized_path.write_text(
                yaml.safe_dump(
                    session.model_copy(update={"raw_filename": raw_path.relative_to(data_root).as_posix(), "content_hash": content_hash}).model_dump(mode="json"),
                    sort_keys=False,
                ),
                encoding="utf-8",
            )
            _upsert_state_record(
                state,
                ImportedChatSessionRecord.model_validate(
                    {
                        "source_kind": config.source_kind,
                        "source_provider": session.source_provider,
                        "session_id": session.session_id,
                        "source_label": config.source_label,
                        "raw_relpath": raw_path.relative_to(data_root).as_posix(),
                        "normalized_relpath": normalized_path.relative_to(data_root).as_posix(),
                        "content_hash": content_hash,
                        "first_seen_at": prior.first_seen_at if prior else datetime.now(tz=UTC),
                        "last_seen_at": datetime.now(tz=UTC),
                    }
                ),
            )
    _write_import_state(data_root, state)
    return result


def _materialize_raw_artifacts(
    *,
    source_config: ChatImportSourceConfig,
    discovered_session,
    raw_dir: Path,
    raw_main_path: Path,
) -> str:
    raw_dir.mkdir(parents=True, exist_ok=True)
    if source_config.source_kind == "claude_archive":
        return _extract_claude_archive_session(
            archive_path=Path(source_config.source_path),
            session_id=discovered_session.session_id,
            raw_dir=raw_dir,
            raw_main_path=raw_main_path,
        )
    source_file = Path(source_config.source_path) / discovered_session.raw_source_relpath
    shutil.copy2(source_file, raw_main_path)
    return _hash_file(raw_main_path)


def _discover_sessions(config: ChatImportSourceConfig):
    if config.source_kind == "claude_archive":
        return discover_claude_archive_sessions(
            archive_path=Path(config.source_path),
            source_label=config.source_label,
        )
    if config.source_kind == "claude_live":
        return discover_claude_live_sessions(
            claude_root=Path(config.source_path),
            source_label=config.source_label,
        )
    return discover_codex_live_sessions(
        codex_root=Path(config.source_path),
        source_label=config.source_label,
    )


def _load_import_state(data_root: Path) -> ChatImportState:
    manifest_path = import_state_path(data_root)
    if not manifest_path.exists():
        return ChatImportState()
    return ChatImportState.model_validate(
        yaml.safe_load(manifest_path.read_text(encoding="utf-8")) or {}
    )


def _write_import_state(data_root: Path, state: ChatImportState) -> None:
    manifest_path = import_state_path(data_root)
    manifest_path.parent.mkdir(parents=True, exist_ok=True)
    manifest_path.write_text(
        yaml.safe_dump(state.model_dump(mode="json"), sort_keys=False),
        encoding="utf-8",
    )


def _upsert_state_record(state: ChatImportState, record: ImportedChatSessionRecord) -> None:
    remaining = [
        existing
        for existing in state.sessions
        if not (
            existing.source_provider == record.source_provider
            and existing.session_id == record.session_id
        )
    ]
    remaining.append(record)
    state.sessions = remaining


def _extract_claude_archive_session(
    *,
    archive_path: Path,
    session_id: str,
    raw_dir: Path,
    raw_main_path: Path,
) -> str:
    with tarfile.open(archive_path, "r:gz") as archive:
        for member in archive.getmembers():
            if session_id not in member.name:
                continue
            if not member.isfile():
                continue
            extracted = archive.extractfile(member)
            if extracted is None:
                continue
            target = raw_main_path if member.name.endswith(f"{session_id}.jsonl") else raw_dir / Path(member.name).name
            target.parent.mkdir(parents=True, exist_ok=True)
            target.write_bytes(extracted.read())
    return _hash_file(raw_main_path)


def _hash_file(path: Path) -> str:
    return "sha256:" + hashlib.sha256(path.read_bytes()).hexdigest()


def cut_private_chat_snapshot(*, data_root: Path, snapshot_id: str, source_workspace: str) -> Path:
    state = _load_import_state(data_root)
    root = snapshot_root(data_root, snapshot_id)
    sessions_root = root / "normalized" / "sessions"
    sessions_root.mkdir(parents=True, exist_ok=True)
    manifest_sessions = []
    for record in state.sessions:
        source_path = data_root / record.normalized_relpath
        target_path = sessions_root / Path(record.normalized_relpath).name
        shutil.copy2(source_path, target_path)
        manifest_sessions.append(
            {
                "session_id": record.session_id,
                "source_provider": record.source_provider,
                "source_app": "claude_code" if record.source_provider == "anthropic" else "codex_cli",
                "raw_filename": record.raw_relpath,
                "normalized_filename": target_path.relative_to(root).as_posix(),
                "content_hash": record.content_hash,
            }
        )
    manifest = PrivateCorpusManifest.model_validate(
        {
            "snapshot_id": snapshot_id,
            "exported_at": datetime.now(tz=UTC),
            "source_workspace": source_workspace,
            "sessions": manifest_sessions,
            "memos": [],
        }
    )
    (root / "manifest.yaml").write_text(
        yaml.safe_dump(manifest.model_dump(mode="json"), sort_keys=False),
        encoding="utf-8",
    )
    return root
```

- [ ] **Step 4: Add importer/snapshot CLI commands**

```python
# /home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/cli.py
@app.command("top-up-private-chat-imports")
def top_up_private_chat_imports_command(
    *,
    data_root: Path = typer.Option(...),
    config_path: Path = typer.Option(...),
) -> None:
    config = yaml.safe_load(config_path.read_text(encoding="utf-8"))
    result = top_up_private_chat_sources(
        data_root=data_root,
        source_configs=config["sources"],
    )
    typer.echo(f"imported: {result.imported}")
    typer.echo(f"updated: {result.updated}")
    typer.echo(f"unchanged: {result.unchanged}")


@app.command("cut-private-chat-snapshot")
def cut_private_chat_snapshot_command(
    *,
    data_root: Path = typer.Option(...),
    snapshot_id: str = typer.Option(...),
    source_workspace: str = typer.Option("prescient_os_data"),
) -> None:
    root = cut_private_chat_snapshot(
        data_root=data_root,
        snapshot_id=snapshot_id,
        source_workspace=source_workspace,
    )
    typer.echo(root.as_posix())
```

- [ ] **Step 5: Add the integration CLI test**

```python
from pathlib import Path

import yaml
from typer.testing import CliRunner

from prescient_benchmark.cli import app


def test_top_up_private_chat_imports_cli_reports_counts(tmp_path: Path) -> None:
    data_root = tmp_path / "prescient_os_data"
    source_root = tmp_path / "sources"
    (source_root / "codex" / "sessions" / "2026" / "04" / "14").mkdir(parents=True)
    (source_root / "codex" / "history.jsonl").write_text(
        Path("tests/fixtures/chat_import/codex_history.jsonl").read_text(encoding="utf-8"),
        encoding="utf-8",
    )
    (source_root / "codex" / "sessions" / "2026" / "04" / "14" / "session.jsonl").write_text(
        Path("tests/fixtures/chat_import/codex_live_session.jsonl").read_text(encoding="utf-8"),
        encoding="utf-8",
    )
    config_path = tmp_path / "chat_import_sources.yaml"
    config_path.write_text(
        yaml.safe_dump(
            {
                "sources": [
                    {
                        "source_kind": "codex_live",
                        "source_label": "local-codex",
                        "source_path": str(source_root / "codex"),
                    }
                ]
            },
            sort_keys=False,
        ),
        encoding="utf-8",
    )

    result = CliRunner().invoke(
        app,
        [
            "top-up-private-chat-imports",
            "--data-root",
            str(data_root),
            "--config-path",
            str(config_path),
        ],
    )

    assert result.exit_code == 0
    assert "imported: 1" in result.stdout
```

- [ ] **Step 6: Run the service and integration tests**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_chat_import_service.py tests/integration/test_chat_import_cli.py -q
```

Expected: PASS with all tests green.

- [ ] **Step 7: Commit Task 4**

```bash
git add apps/api/src/prescient_benchmark/corpus/chat_import_service.py \
  apps/api/src/prescient_benchmark/cli.py \
  tests/unit/test_chat_import_service.py \
  tests/integration/test_chat_import_cli.py
git commit -m "feat: add private chat top-up and snapshot commands"
```

### Task 5: Add the private data repo config and runner wrappers

**Files:**
- Create: `/home/rhallman/Projects/prescient_os_data/config/chat_import_sources.example.yaml`
- Create: `/home/rhallman/Projects/prescient_os_data/imports/README.md`
- Create: `/home/rhallman/Projects/prescient_os_data/scripts/top_up_private_chats.sh`
- Create: `/home/rhallman/Projects/prescient_os_data/scripts/cut_private_chat_snapshot.sh`

- [ ] **Step 1: Add the example import config**

```yaml
# /home/rhallman/Projects/prescient_os_data/config/chat_import_sources.example.yaml
sources:
  - source_kind: claude_archive
    source_label: other-workstation-archive
    source_path: /home/rhallman/prescient_os-chat-history-2026-04-18.tar.gz
  - source_kind: claude_live
    source_label: local-claude
    source_path: /home/rhallman/.claude
  - source_kind: codex_live
    source_label: local-codex
    source_path: /home/rhallman/.codex
```

- [ ] **Step 2: Add the imports README**

```markdown
# Private Chat Import State

This directory holds mutable private import state for retrieval benchmarking.

- `chat_sources_manifest.yaml` is the canonical top-up state file
- `../raw/` is the provider-native mirror
- `../normalized/` holds rebuildable sparse session records
- `../snapshots/` holds frozen benchmark cuts

Top-up mutates `raw/`, `normalized/`, and `chat_sources_manifest.yaml`.
Benchmark runs should consume frozen `snapshots/<snapshot_id>/`.
```

- [ ] **Step 3: Add the top-up runner wrapper**

```bash
#!/usr/bin/env bash
set -euo pipefail

DATA_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
PRESCIENT_OS_ROOT="${PRESCIENT_OS_ROOT:-$DATA_ROOT/../prescient_os}"
CONFIG_PATH="${1:-$DATA_ROOT/config/chat_import_sources.yaml}"

cd "$PRESCIENT_OS_ROOT"
uv run python -m prescient_benchmark.cli top-up-private-chat-imports \
  --data-root "$DATA_ROOT" \
  --config-path "$CONFIG_PATH"
```

- [ ] **Step 4: Add the snapshot-cut runner wrapper**

```bash
#!/usr/bin/env bash
set -euo pipefail

if [[ $# -lt 1 ]]; then
  echo "usage: $0 <snapshot-id>" >&2
  exit 1
fi

SNAPSHOT_ID="$1"
DATA_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
PRESCIENT_OS_ROOT="${PRESCIENT_OS_ROOT:-$DATA_ROOT/../prescient_os}"

cd "$PRESCIENT_OS_ROOT"
uv run python -m prescient_benchmark.cli cut-private-chat-snapshot \
  --data-root "$DATA_ROOT" \
  --snapshot-id "$SNAPSHOT_ID" \
  --source-workspace "$DATA_ROOT"
```

- [ ] **Step 5: Verify the runner files exist and are executable**

Run:

```bash
test -f /home/rhallman/Projects/prescient_os_data/config/chat_import_sources.example.yaml
test -f /home/rhallman/Projects/prescient_os_data/imports/README.md
test -x /home/rhallman/Projects/prescient_os_data/scripts/top_up_private_chats.sh
test -x /home/rhallman/Projects/prescient_os_data/scripts/cut_private_chat_snapshot.sh
```

Expected: all commands exit `0`

- [ ] **Step 6: Commit Task 5**

```bash
git -C /home/rhallman/Projects/prescient_os_data add \
  config/chat_import_sources.example.yaml \
  imports/README.md \
  scripts/top_up_private_chats.sh \
  scripts/cut_private_chat_snapshot.sh
git -C /home/rhallman/Projects/prescient_os_data commit -m "docs: add private chat import runners"
```

### Task 6: Final verification across both repos

**Files:**
- Verify all files from Tasks 1-5

- [ ] **Step 1: Run focused importer tests in the product repo**

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest \
  tests/unit/test_chat_import_models.py \
  tests/unit/test_chat_import_claude.py \
  tests/unit/test_chat_import_codex.py \
  tests/unit/test_chat_import_service.py \
  tests/integration/test_chat_import_cli.py -q
```

Expected: all importer tests pass.

- [ ] **Step 2: Run the broader product test suite**

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit tests/integration -q
```

Expected: full suite passes.

- [ ] **Step 3: Check for diff formatting issues**

```bash
cd /home/rhallman/Projects/prescient_os
git diff --check
```

Expected: no output

- [ ] **Step 4: Commit any final verification-driven fixes**

```bash
cd /home/rhallman/Projects/prescient_os
git status --short
```

Expected: clean working tree after any final fixes.

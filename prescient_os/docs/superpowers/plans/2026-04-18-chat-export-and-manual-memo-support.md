# Chat Export and Manual Memo Support Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a sparse private chat-corpus path that stores snapshots in `prescient_os_data`, loads them safely into the benchmark codebase, and supports a small set of manually curated point-in-time memos.

**Architecture:** Keep private benchmark data in the separate `prescient_os_data` repo as raw + sparse normalized snapshots plus manual memos. In `prescient_os`, add typed models, path validation, a loader that converts sessions/memos into retrieval documents, and a thin CLI command to validate and inspect a snapshot without pulling retrieval or synthesis concerns into this slice.

**Tech Stack:** Python 3.13, Pydantic v2, PyYAML, Typer, pytest, Markdown with YAML frontmatter

---

## File Structure

### Data repo: `/home/rhallman/Projects/prescient_os_data`

- Create: `/home/rhallman/Projects/prescient_os_data/README.md`
  - explains repo purpose, privacy boundary, and snapshot layout
- Create: `/home/rhallman/Projects/prescient_os_data/.gitignore`
  - keeps machine-local noise out of the private corpus repo
- Create: `/home/rhallman/Projects/prescient_os_data/corpus/prescient_os/README.md`
  - defines snapshot naming and required files
- Create: `/home/rhallman/Projects/prescient_os_data/templates/chat_export_manifest.example.yaml`
  - canonical example manifest for exported snapshots
- Create: `/home/rhallman/Projects/prescient_os_data/templates/normalized_session.example.yaml`
  - canonical example normalized session
- Create: `/home/rhallman/Projects/prescient_os_data/templates/manual_point_in_time_memo.md`
  - AI-assisted memo prompt/output template

### Product repo: `/home/rhallman/Projects/prescient_os`

- Create: `/home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/corpus/private_chat_models.py`
  - typed models for snapshot manifests, normalized sessions, messages, and manual memos
- Create: `/home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/corpus/private_chat_storage.py`
  - path validation for external private corpus snapshots
- Create: `/home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/corpus/private_chat_loader.py`
  - loads normalized sessions and memos into retrieval-friendly in-memory documents
- Modify: `/home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/cli.py`
  - add a narrow `inspect-private-corpus` command
- Create: `/home/rhallman/Projects/prescient_os/tests/unit/test_private_chat_storage.py`
  - tests path validation and shape constraints
- Create: `/home/rhallman/Projects/prescient_os/tests/unit/test_private_chat_loader.py`
  - tests session + memo loading and retrieval-text shaping
- Modify: `/home/rhallman/Projects/prescient_os/tests/integration/test_eval_harness_cli.py`
  - only if needed for CLI-style invocation coverage; otherwise prefer a focused new integration test
- Create: `/home/rhallman/Projects/prescient_os/tests/integration/test_private_chat_corpus_cli.py`
  - validates `inspect-private-corpus`

The slice intentionally stops before retrieval benchmarking changes. It should not modify the eval harness, evidence-key design, or question authoring.

### Task 1: Scaffold the private corpus repo and templates

**Files:**
- Create: `/home/rhallman/Projects/prescient_os_data/README.md`
- Create: `/home/rhallman/Projects/prescient_os_data/.gitignore`
- Create: `/home/rhallman/Projects/prescient_os_data/corpus/prescient_os/README.md`
- Create: `/home/rhallman/Projects/prescient_os_data/templates/chat_export_manifest.example.yaml`
- Create: `/home/rhallman/Projects/prescient_os_data/templates/normalized_session.example.yaml`
- Create: `/home/rhallman/Projects/prescient_os_data/templates/manual_point_in_time_memo.md`

- [ ] **Step 1: Create the repo README and corpus layout docs**

```markdown
# Prescient OS Private Benchmark Data

Private local corpus snapshots for retrieval benchmarking. This repo stores:

- raw provider exports
- sparse normalized sessions
- manually curated point-in-time memos

It must not be merged into the product code repo.

## Snapshot layout

`corpus/prescient_os/<snapshot_id>/`

- `manifest.yaml`
- `raw/<provider>/...`
- `normalized/sessions/<session_id>.yaml`
- `memos/<memo_id>.md`
```

```markdown
# Prescient OS Snapshot Layout

Each snapshot under `corpus/prescient_os/<snapshot_id>/` is self-contained.

Required:
- `manifest.yaml`
- `normalized/sessions/*.yaml`

Optional:
- `raw/<provider>/...`
- `memos/*.md`
```

- [ ] **Step 2: Add the example manifest and normalized session templates**

```yaml
# /home/rhallman/Projects/prescient_os_data/templates/chat_export_manifest.example.yaml
snapshot_id: prescientos_founder_v1
exported_at: "2026-04-18T20:00:00Z"
source_workspace: other-workstation
sessions:
  - session_id: claude-2026-04-01-001
    source_provider: anthropic
    source_app: claude_code
    raw_filename: raw/anthropic/claude-2026-04-01-001.json
    normalized_filename: normalized/sessions/claude-2026-04-01-001.yaml
    content_hash: sha256:replace-me
memos:
  - memo_id: ke-first-pivot-2026-04
    filename: memos/ke-first-pivot-2026-04.md
```

```yaml
# /home/rhallman/Projects/prescient_os_data/templates/normalized_session.example.yaml
session_id: claude-2026-04-01-001
source_provider: anthropic
source_app: claude_code
title: KE-first pivot discussion
exported_at: "2026-04-18T20:00:00Z"
raw_filename: raw/anthropic/claude-2026-04-01-001.json
content_hash: sha256:replace-me
provider_metadata:
  model: claude-sonnet-4-6
messages:
  - index: 0
    role: system
    content_text: You are a coding agent.
  - index: 1
    role: user
    timestamp: "2026-04-01T10:12:00Z"
    content_text: Are we solving too many problems at once?
  - index: 2
    role: assistant
    timestamp: "2026-04-01T10:12:08Z"
    content_text: Yes. The benchmark should narrow to retrieval first.
```

- [ ] **Step 3: Add the manual memo template**

```markdown
---
memo_id: ke-first-pivot-2026-04
title: KE-first Pivot Memo
as_of: 2026-04-18
topic_bucket: Evaluation and Benchmarks
source_refs:
  - session_id: claude-2026-04-01-001
  - git_ref: 2830532
---

# Summary

Write only from the supplied sessions and refs.

# Key Points

- ...

# Open Questions

- ...
```

- [ ] **Step 4: Add a minimal `.gitignore`**

```gitignore
.DS_Store
Thumbs.db
*.swp
*.tmp
```

- [ ] **Step 5: Verify the scaffold exists**

Run:

```bash
test -f /home/rhallman/Projects/prescient_os_data/README.md
test -f /home/rhallman/Projects/prescient_os_data/corpus/prescient_os/README.md
test -f /home/rhallman/Projects/prescient_os_data/templates/chat_export_manifest.example.yaml
test -f /home/rhallman/Projects/prescient_os_data/templates/normalized_session.example.yaml
test -f /home/rhallman/Projects/prescient_os_data/templates/manual_point_in_time_memo.md
```

Expected: all commands exit `0`

- [ ] **Step 6: Commit the data-repo scaffold**

```bash
git -C /home/rhallman/Projects/prescient_os_data add README.md .gitignore corpus/prescient_os/README.md templates
git -C /home/rhallman/Projects/prescient_os_data commit -m "docs: add private corpus snapshot templates"
```

### Task 2: Add typed private-corpus models and path validation in the product repo

**Files:**
- Create: `/home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/corpus/private_chat_models.py`
- Create: `/home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/corpus/private_chat_storage.py`
- Create: `/home/rhallman/Projects/prescient_os/tests/unit/test_private_chat_storage.py`

- [ ] **Step 1: Write the failing storage/model tests**

```python
from pathlib import Path

import pytest

from prescient_benchmark.corpus.private_chat_models import (
    ManualMemoDocument,
    NormalizedChatSession,
    PrivateCorpusManifest,
)
from prescient_benchmark.corpus.private_chat_storage import normalize_private_corpus_root


def test_normalize_private_corpus_root_accepts_snapshot_with_manifest(tmp_path: Path) -> None:
    snapshot_root = tmp_path / "corpus" / "prescient_os" / "founder_v1"
    (snapshot_root / "normalized" / "sessions").mkdir(parents=True)
    (snapshot_root / "manifest.yaml").write_text("snapshot_id: founder_v1\nexported_at: '2026-04-18T20:00:00Z'\nsessions: []\n", encoding="utf-8")

    assert normalize_private_corpus_root(snapshot_root) == snapshot_root.resolve()


def test_normalize_private_corpus_root_rejects_missing_manifest(tmp_path: Path) -> None:
    snapshot_root = tmp_path / "corpus" / "prescient_os" / "founder_v1"
    snapshot_root.mkdir(parents=True)

    with pytest.raises(ValueError, match="manifest.yaml"):
        normalize_private_corpus_root(snapshot_root)


def test_normalized_chat_session_rejects_non_monotonic_indexes() -> None:
    with pytest.raises(Exception):
        NormalizedChatSession.model_validate(
            {
                "session_id": "session-001",
                "source_provider": "anthropic",
                "source_app": "claude_code",
                "exported_at": "2026-04-18T20:00:00Z",
                "raw_filename": "raw/anthropic/session-001.json",
                "content_hash": "sha256:abc",
                "messages": [
                    {"index": 1, "role": "user", "content_text": "first"},
                    {"index": 0, "role": "assistant", "content_text": "second"},
                ],
            }
        )


def test_manual_memo_document_accepts_markdown_frontmatter_payload() -> None:
    memo = ManualMemoDocument.model_validate(
        {
            "memo_id": "ke-first-pivot-2026-04",
            "title": "KE-first Pivot Memo",
            "as_of": "2026-04-18",
            "topic_bucket": "Evaluation and Benchmarks",
            "summary": "We narrowed to retrieval benchmarking.",
            "key_points": ["Drop synthesis from primary scoring."],
            "source_refs": [{"session_id": "session-001"}],
        }
    )

    assert memo.memo_id == "ke-first-pivot-2026-04"
```

- [ ] **Step 2: Run the tests to confirm failure**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_private_chat_storage.py -q
```

Expected: failures for missing modules/imports

- [ ] **Step 3: Implement the minimal models and storage helpers**

```python
# /home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/corpus/private_chat_models.py
from datetime import UTC, date, datetime

from pydantic import BaseModel, ConfigDict, Field, StrictInt, StrictStr, field_validator


class ChatMessage(BaseModel):
    model_config = ConfigDict(extra="forbid", strict=True)

    index: StrictInt
    role: StrictStr
    content_text: StrictStr
    timestamp: StrictStr | None = None


class NormalizedChatSession(BaseModel):
    model_config = ConfigDict(extra="forbid", strict=True)

    session_id: StrictStr
    source_provider: StrictStr
    source_app: StrictStr
    exported_at: datetime
    raw_filename: StrictStr
    content_hash: StrictStr
    title: StrictStr | None = None
    provider_metadata: dict[str, object] = Field(default_factory=dict)
    messages: list[ChatMessage]

    @field_validator("exported_at")
    @classmethod
    def exported_at_must_be_utc(cls, value: datetime) -> datetime:
        if value.tzinfo is None or value.utcoffset() is None:
            raise ValueError("exported_at must be timezone-aware")
        return value.astimezone(UTC)

    @field_validator("messages")
    @classmethod
    def messages_must_be_monotonic(cls, value: list[ChatMessage]) -> list[ChatMessage]:
        indexes = [message.index for message in value]
        if indexes != list(range(len(indexes))):
            raise ValueError("messages must use contiguous zero-based indexes")
        return value


class ManifestSessionRef(BaseModel):
    model_config = ConfigDict(extra="forbid", strict=True)

    session_id: StrictStr
    source_provider: StrictStr
    source_app: StrictStr
    raw_filename: StrictStr
    normalized_filename: StrictStr
    content_hash: StrictStr


class ManifestMemoRef(BaseModel):
    model_config = ConfigDict(extra="forbid", strict=True)

    memo_id: StrictStr
    filename: StrictStr


class PrivateCorpusManifest(BaseModel):
    model_config = ConfigDict(extra="forbid", strict=True)

    snapshot_id: StrictStr
    exported_at: datetime
    source_workspace: StrictStr | None = None
    sessions: list[ManifestSessionRef]
    memos: list[ManifestMemoRef] = Field(default_factory=list)


class MemoSourceRef(BaseModel):
    model_config = ConfigDict(extra="forbid", strict=True)

    session_id: StrictStr | None = None
    git_ref: StrictStr | None = None


class ManualMemoDocument(BaseModel):
    model_config = ConfigDict(extra="forbid", strict=True)

    memo_id: StrictStr
    title: StrictStr
    as_of: date
    topic_bucket: StrictStr
    summary: StrictStr
    key_points: list[StrictStr]
    open_questions: list[StrictStr] = Field(default_factory=list)
    source_refs: list[MemoSourceRef]
```

```python
# /home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/corpus/private_chat_storage.py
from pathlib import Path


def normalize_private_corpus_root(path: Path) -> Path:
    resolved = path.resolve(strict=False)
    manifest_path = resolved / "manifest.yaml"
    sessions_root = resolved / "normalized" / "sessions"

    if not manifest_path.exists():
        raise ValueError(f"private corpus snapshot must contain manifest.yaml: {path}")
    if not sessions_root.exists():
        raise ValueError(f"private corpus snapshot must contain normalized/sessions: {path}")
    if not manifest_path.is_file():
        raise ValueError(f"private corpus manifest must be a file: {path}")
    if not sessions_root.is_dir():
        raise ValueError(f"private corpus sessions root must be a directory: {path}")

    return resolved
```

- [ ] **Step 4: Run the unit tests**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_private_chat_storage.py -q
```

Expected: `4 passed`

- [ ] **Step 5: Commit the models and storage helpers**

```bash
cd /home/rhallman/Projects/prescient_os
git add apps/api/src/prescient_benchmark/corpus/private_chat_models.py apps/api/src/prescient_benchmark/corpus/private_chat_storage.py tests/unit/test_private_chat_storage.py
git commit -m "feat: add private chat corpus models"
```

### Task 3: Build the private-corpus loader for sessions and memos

**Files:**
- Create: `/home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/corpus/private_chat_loader.py`
- Create: `/home/rhallman/Projects/prescient_os/tests/unit/test_private_chat_loader.py`

- [ ] **Step 1: Write the failing loader tests**

```python
from pathlib import Path

from prescient_benchmark.corpus.private_chat_loader import load_private_corpus_documents


def test_load_private_corpus_documents_builds_retrieval_docs_from_sessions_and_memos(tmp_path: Path) -> None:
    snapshot_root = tmp_path / "corpus" / "prescient_os" / "founder_v1"
    sessions_root = snapshot_root / "normalized" / "sessions"
    memos_root = snapshot_root / "memos"
    sessions_root.mkdir(parents=True)
    memos_root.mkdir(parents=True)

    (snapshot_root / "manifest.yaml").write_text(
        \"\"\"snapshot_id: founder_v1
exported_at: "2026-04-18T20:00:00Z"
sessions:
  - session_id: session-001
    source_provider: anthropic
    source_app: claude_code
    raw_filename: raw/anthropic/session-001.json
    normalized_filename: normalized/sessions/session-001.yaml
    content_hash: sha256:abc
memos:
  - memo_id: ke-first-pivot-2026-04
    filename: memos/ke-first-pivot-2026-04.md
\"\"\",
        encoding="utf-8",
    )
    (sessions_root / "session-001.yaml").write_text(
        \"\"\"session_id: session-001
source_provider: anthropic
source_app: claude_code
exported_at: "2026-04-18T20:00:00Z"
raw_filename: raw/anthropic/session-001.json
content_hash: sha256:abc
messages:
  - index: 0
    role: system
    content_text: do not index me
  - index: 1
    role: user
    content_text: We should pivot to retrieval-first evaluation.
  - index: 2
    role: assistant
    content_text: Yes, synthesis is confounding the benchmark.
\"\"\",
        encoding="utf-8",
    )
    (memos_root / "ke-first-pivot-2026-04.md").write_text(
        \"\"\"---
memo_id: ke-first-pivot-2026-04
title: KE-first Pivot Memo
as_of: 2026-04-18
topic_bucket: Evaluation and Benchmarks
summary: Retrieval should be scored separately from synthesis.
key_points:
  - Remove synthesis from primary scoring.
source_refs:
  - session_id: session-001
---

# Summary

Retrieval should be scored separately from synthesis.
\"\"\",
        encoding="utf-8",
    )

    documents = load_private_corpus_documents(snapshot_root)

    assert [document.doc_id for document in documents] == [
        "chat:session-001",
        "memo:ke-first-pivot-2026-04",
    ]
    assert "do not index me" not in documents[0].text
    assert "pivot to retrieval-first evaluation" in documents[0].text
    assert "Remove synthesis from primary scoring." in documents[1].text
```

- [ ] **Step 2: Run the loader test to confirm failure**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_private_chat_loader.py -q
```

Expected: failure for missing loader module

- [ ] **Step 3: Implement the loader**

```python
# /home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/corpus/private_chat_loader.py
from __future__ import annotations

from pathlib import Path

import yaml

from prescient_benchmark.corpus.private_chat_models import (
    ManualMemoDocument,
    NormalizedChatSession,
    PrivateCorpusManifest,
)
from prescient_benchmark.corpus.private_chat_storage import normalize_private_corpus_root
from prescient_benchmark.eval.storage import read_yaml_model
from prescient_benchmark.retrieval.index import InMemoryDocument


def _session_to_document(session: NormalizedChatSession) -> InMemoryDocument:
    text_parts = [
        message.content_text
        for message in session.messages
        if message.role != "system"
    ]
    return InMemoryDocument(
        doc_id=f"chat:{session.session_id}",
        title=session.title or session.session_id,
        text="\n\n".join(text_parts),
    )


def _load_memo(path: Path) -> ManualMemoDocument:
    content = path.read_text(encoding="utf-8")
    if not content.startswith("---\n"):
        raise ValueError(f"memo must start with YAML frontmatter: {path}")
    _, frontmatter, body = content.split("---\n", 2)
    payload = yaml.safe_load(frontmatter) or {}
    payload["summary"] = payload.get("summary") or body.strip()
    return ManualMemoDocument.model_validate(payload)


def load_private_corpus_documents(snapshot_root: Path) -> list[InMemoryDocument]:
    normalized_root = normalize_private_corpus_root(snapshot_root)
    manifest = read_yaml_model(normalized_root / "manifest.yaml", PrivateCorpusManifest)

    documents: list[InMemoryDocument] = []
    for session_ref in manifest.sessions:
        session = read_yaml_model(normalized_root / session_ref.normalized_filename, NormalizedChatSession)
        documents.append(_session_to_document(session))

    for memo_ref in manifest.memos:
        memo = _load_memo(normalized_root / memo_ref.filename)
        documents.append(
            InMemoryDocument(
                doc_id=f"memo:{memo.memo_id}",
                title=memo.title,
                text="\n".join([memo.summary, *memo.key_points]),
            )
        )

    return documents
```

- [ ] **Step 4: Run the new tests**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_private_chat_loader.py tests/unit/test_private_chat_storage.py -q
```

Expected: all tests pass

- [ ] **Step 5: Commit the loader**

```bash
cd /home/rhallman/Projects/prescient_os
git add apps/api/src/prescient_benchmark/corpus/private_chat_loader.py tests/unit/test_private_chat_loader.py
git commit -m "feat: load private chat benchmark corpora"
```

### Task 4: Add a thin CLI inspection command

**Files:**
- Modify: `/home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/cli.py`
- Create: `/home/rhallman/Projects/prescient_os/tests/integration/test_private_chat_corpus_cli.py`

- [ ] **Step 1: Write the failing CLI test**

```python
from pathlib import Path

from typer.testing import CliRunner

from prescient_benchmark.cli import app


def test_inspect_private_corpus_reports_session_and_memo_counts(tmp_path: Path, monkeypatch) -> None:
    monkeypatch.chdir(tmp_path)
    snapshot_root = tmp_path / "corpus" / "prescient_os" / "founder_v1"
    (snapshot_root / "normalized" / "sessions").mkdir(parents=True)
    (snapshot_root / "memos").mkdir(parents=True)
    (snapshot_root / "manifest.yaml").write_text(
        "snapshot_id: founder_v1\nexported_at: '2026-04-18T20:00:00Z'\nsessions: []\nmemos: []\n",
        encoding="utf-8",
    )

    runner = CliRunner()
    result = runner.invoke(app, ["inspect-private-corpus", "--source-root", str(snapshot_root)])

    assert result.exit_code == 0
    assert "founder_v1" in result.stdout
    assert "sessions: 0" in result.stdout
    assert "memos: 0" in result.stdout
```

- [ ] **Step 2: Run the CLI test to confirm failure**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/integration/test_private_chat_corpus_cli.py -q
```

Expected: failure because `inspect-private-corpus` does not exist

- [ ] **Step 3: Add the CLI command**

```python
# append to /home/rhallman/Projects/prescient_os/apps/api/src/prescient_benchmark/cli.py
@app.command("inspect-private-corpus")
def inspect_private_corpus(
    *,
    source_root: Path = typer.Option(...),
) -> None:
    from prescient_benchmark.corpus.private_chat_models import PrivateCorpusManifest
    from prescient_benchmark.corpus.private_chat_storage import normalize_private_corpus_root
    from prescient_benchmark.eval.storage import read_yaml_model

    snapshot_root = normalize_private_corpus_root(source_root)
    manifest = read_yaml_model(snapshot_root / "manifest.yaml", PrivateCorpusManifest)
    typer.echo(f"snapshot_id: {manifest.snapshot_id}")
    typer.echo(f"sessions: {len(manifest.sessions)}")
    typer.echo(f"memos: {len(manifest.memos)}")
```

- [ ] **Step 4: Run integration and unit coverage**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit/test_private_chat_storage.py tests/unit/test_private_chat_loader.py tests/integration/test_private_chat_corpus_cli.py -q
```

Expected: all pass

- [ ] **Step 5: Run the full benchmark suite**

Run:

```bash
cd /home/rhallman/Projects/prescient_os
uv run python -m pytest tests/unit tests/integration -q
```

Expected: full suite passes with no regressions

- [ ] **Step 6: Commit the CLI command**

```bash
cd /home/rhallman/Projects/prescient_os
git add apps/api/src/prescient_benchmark/cli.py tests/integration/test_private_chat_corpus_cli.py
git commit -m "feat: inspect private chat corpora"
```

## Self-Review

- Spec coverage:
  - sparse export schema: covered by Task 1 templates and Task 2 models
  - manifest format: covered by Task 1 example manifest and Task 2 typed manifest model
  - importer for normalized sessions: covered by Task 3 loader
  - saved prompt/template for manual memo drafting: covered by Task 1 memo template
- Placeholder scan:
  - no `TBD`/`TODO`
  - all file paths are explicit
  - all test commands are explicit
- Type consistency:
  - `snapshot_id`, `session_id`, `memo_id`, `content_hash`, and `provider_metadata` are used consistently
  - CLI and loader both point at the same `manifest.yaml` and `normalized/sessions` layout

## Notes

- Do not add retrieval evaluation changes in this plan.
- Do not add memo automation or ontology work in this plan.
- Do not commit private corpus content into `/home/rhallman/Projects/prescient_os`.

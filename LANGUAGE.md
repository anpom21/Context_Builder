# Language

The vocabulary Context Builder's code and UI use for its domain. Prefer these terms — and their canonical meaning — over ad hoc synonyms when writing code, commit messages, issues, or talking about this repo. See [CONTEXT.md](./CONTEXT.md) for the overall picture.

## The output

**Context Structure** (a.k.a. **Bundle**):
The complete JSON object Context Builder produces and copies to the clipboard — `system_prompt` + `llm_task` + `user_prompt` + `schema_version` + `files` + `dependency_graph`. `Workspace.to_bundle()` builds it; `validate_bundle()` checks it; `serialize_bundle()` renders deterministic JSON.
_Avoid_: payload, output JSON, export (export is a related but distinct action — see below).

**System Prompt**:
The stable, non-task-specific instructions (from `system_prompt.md`) telling the LLM how to interpret the bundle's fields and, on code-change requests, to answer with a downloadable Git diff patch instead of inline code. Always present, always first in the bundle.
_Avoid_: instructions, preamble.

**LLM Task**:
The task-specific behavior directive — either a built-in template (`code_editing`, `code_review`, `debugging`, `architecture_explanation`, `refactor_planning`, `grill_me`) or custom free text, resolved via `PromptTemplateMode.resolved_text()`. Tells the LLM what *role* to play (reviewer, debugger, architect, ...).
_Avoid_: task, mode, role (role is fine in prose but the field name is `llm_task`).

**User Prompt**:
The developer's actual, free-text task request — "the primary objective," per `system_prompt.md`. Everything else in the bundle exists to help satisfy this.
_Avoid_: query, question, ask.

**Schema Version**:
An integer (`SCHEMA_VERSION` in `core.py`) identifying the bundle format's compatibility. Bumped when the bundle shape changes; never bump silently.

## Files and where they came from

**File Record**:
One entry describing a single file's state — id, path(s), inclusion state, size/hash metadata, and content — modeled by the `FileRecord` dataclass. The unit everything else (tree nodes, dependency edges, curator recommendations) refers to by `id`.
_Avoid_: file entry, file object.

**Direct File**:
A file the user explicitly added (button, drag-and-drop, or startup CLI arg). `source_kind="direct_file"`, `context_type="file_from_user"`.
_Avoid_: root file, seed file (seed is used only inside `curator.py` for scoring, not as a general term).

**Folder File**:
A file discovered by recursively scanning a folder the user added. `source_kind="folder_file"`, `context_type="file_from_folder"`.
_Avoid_: scanned file, discovered file.

**Dependency File**:
A repo-local Python file pulled in because something else imports it. `is_dependency=True`; `context_type` is `dependency_file` (found from a direct file) or `file_from_folder_dependency` (found from a folder file) — the distinction tells the LLM *why* the file is present.
_Avoid_: imported file, transitive file.

**Reused Node** / **Reused Dependency**:
A dependency file reached through more than one import path. Stored once as a single `FileRecord` (deduplicated by canonical absolute path) but rendered as a `TreeNode` under every parent that reaches it, marked `reused=True`. Inclusion state is shared across every appearance, not per-occurrence.
_Avoid_: duplicate file, shared dependency.

**Skipped Dependency**:
An import that was detected in source but not resolved to a file — because it's stdlib/external (`reason="stdlib"` / `"unresolved_or_external"`), circular (`reason="circular"`), or exceeded `max_dependency_depth`. Recorded with a reason so the LLM (and the diagnostics panel) knows a gap is intentional, not a bug.
_Avoid_: missing import, unresolved dependency (the code uses "unresolved_or_external" as one specific *reason*, not the general term).

**Excluded File**:
A file the app found but is not sending content for — currently either because it's binary/oversized (auto-excluded, listed in `workspace.excluded_files`) or because the user set its inclusion mode to `excluded`. Still appears in `files` with `included: false`, `content: null` if it's part of the dependency graph, so the LLM knows it exists.
_Avoid_: hidden file, ignored file (ignored files, below, are a different and stricter concept — they never become a `FileRecord` at all).

**Ignored (directory/path)**:
A path skipped during folder scanning before it ever becomes a `FileRecord` — `.venv`, `.git`, `node_modules`, `build`, `dist`, cache dirs, hidden files (unless "include hidden" is on), etc. (`IGNORED_DIR_NAMES` in `core.py`). Stronger than "excluded": ignored paths are invisible to the app, excluded files are visible but content-less.
_Avoid_: excluded, filtered.

## Structure and relationships

**Project Root**:
The detected repository root, used to compute every `repo_relative_path`. Detected by a priority cascade (nearest `.git` → Python project marker → common source root → the dropped path's own parent) or set via manual override. One per workspace.
_Avoid_: repo root (fine in prose, but the field/setting name is `project_root`), base directory.

**Import Root**:
A directory Python import names are resolved against — always includes the project root, plus common source roots like `src` that exist, plus manual overrides. There can be several; module names are resolved against whichever import root produces a match.
_Avoid_: source root (used loosely in prose, but `src` is only one *example* of an import root, not a synonym for the concept).

**Context Tree**:
The default left-panel UI view: a hierarchy of direct files, folder files, dependencies (including reused/skipped/ignored nodes), rendered from `TreeNode` objects. Contrast with the **Flat** view (same files, one flat sortable/filterable/searchable list) and the **Curator** view (ranked recommendations). All three are tabs in `MainWindow.views`.
_Avoid_: file tree (acceptable informally, but "Context Tree" is the term used for the whole discovery hierarchy, including non-file nodes like skipped imports).

**Dependency Graph**:
The normalized, deduplicated list of import relationships between files — `{source_id, includes: [target_id, ...]}` — included in the bundle so the LLM can trace "what does this file pull in" without re-deriving it from source. Internally it's a list of `DependencyEdge`s (one per import statement, undeduplicated) before `_dependency_graph_to_json` groups them; only the grouped form ships in the bundle.
_Avoid_: import graph, call graph (it is explicitly *not* execution order).

**Inclusion Mode**:
Per-file setting controlling what content ships: `full` (entire file), `outline`/`hybrid` (an adaptive structural representation — Python gets an AST-derived skeleton with public/internal symbols and, in `hybrid`, inlined small functions; Markdown gets its heading outline; JSON gets a key/type skeleton; YAML gets a key outline; everything else gets a byte-limited excerpt), or `excluded` (no content). Set via `set_record_inclusion_mode` / `_apply_inclusion_mode`.
_Avoid_: truncation level (truncation is what *produces* the hybrid/outline content and is recorded separately in the `truncation` metadata block; "truncated" as a mode name is normalized to `hybrid` for backward compatibility but is not the preferred term going forward).

**Diagnostic**:
A structured warning surfaced during a build — e.g. "skipped import X: unresolved_or_external" — carrying severity, message, and the file/path it concerns. Collected on `Workspace.diagnostics` and shown to both the user and, implicitly, the LLM (skipped/excluded entries in the bundle serve the same purpose for the LLM).
_Avoid_: warning (fine generically, but `Diagnostic` is the dataclass/field name), error (diagnostics are not necessarily failures).

## Selection and workflow

**Curator**:
The relevance-ranking engine in `curator.py` (`rank_workspace_files`) and its UI tab. Scores every file against the user prompt using local, explainable signals only — no LLM call — and returns a `CuratorRecommendation` (score 0-100, tier, suggested inclusion mode, ranked reasons) per file.
_Avoid_: recommender, AI suggestions (the scoring is deterministic and rule-based, not a model call — "smart" language in the UI copy notwithstanding).

**Tier** (`high` / `potential` / `low`):
The Curator's three relevance buckets — `Highly relevant`, `Potentially relevant`, `Probably unnecessary` — derived from a file's score. Drives the suggested inclusion mode (`high`→full/hybrid, `potential`→hybrid, `low`→excluded).
_Avoid_: category, bucket, level.

**Session**:
A saved snapshot of a Context Builder working state — `input_paths`, `prompt`, `settings`, `file_overrides` — as JSON under `.context_sessions/`, modeled by `SessionState`. Distinct from the **Context Structure**: a session is round-trippable app state; a context structure is the one-way LLM-facing output.
_Avoid_: workspace snapshot (Workspace is the in-memory build result; a Session is what gets persisted to reconstruct one), project (Session is scoped to one build request, not a whole repo).

**Workspace**:
The in-memory result of a build: project/import roots, the full `files` map, the `tree_roots`, the `dependency_graph`, and diagnostics — everything needed to render the UI and produce a bundle. Rebuilt by `BundleBuilder.build()`; not itself persisted (a Session is what gets persisted).
_Avoid_: build result (acceptable loosely, but `Workspace` is the dataclass name and the term used across `core.py`, `curator.py`, and `ui.py`).

**Build**:
The act of scanning inputs, discovering dependencies, and producing a `Workspace` (`BundleBuilder.build()` / `build_context_structure()`), run on a background `BuildWorker` thread so the UI stays responsive, with progress callbacks and cancellation support.
_Avoid_: scan (scanning is one phase of a build, specifically folder traversal), refresh.

**Patch**:
A unified Git diff (`.diff`/`.patch` file) the LLM is instructed to return for code-change requests, per `system_prompt.md`. Dragging one into Context Builder offers to `git apply` it against the detected project root, or import it as an ordinary context file instead.
_Avoid_: diff (diff is the format; "patch" is the artifact/workflow term used in the UI — "Patch file dropped", `_handle_patch_drop`).

## Flagged ambiguities

- "Excluded" is used at two different strictness levels: **ignored** paths never become a `FileRecord` (invisible to the app); **excluded** files do become a `FileRecord` but ship with `content: null`. When writing diagnostics or docs, use "ignored" for the former and "excluded" for the latter — don't collapse them.
- "Truncated" appears both as a legacy inclusion-mode alias (normalized to `hybrid`) and as a description of the `truncation` metadata block that any large `full`-adjacent render can carry. Prefer "hybrid"/"outline" for the mode, and "truncation metadata" for the byte-limit bookkeeping.
- `python_prompt_builder_spec.md`'s original glossary predates the `outline`/`hybrid` split and the Curator; treat this LANGUAGE.md as the current source of truth where the two disagree.

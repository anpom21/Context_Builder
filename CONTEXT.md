# Context Builder

Context Builder is a PySide6 desktop app that turns files and folders on disk into a single, structured JSON payload — a **Context Structure** — that a developer pastes into an LLM so it can act on a repository without manual copy-pasting. It also closes the loop: the LLM is instructed to answer with a downloadable Git diff patch, which can be dropped back into the app and applied with `git apply`.

## Why it exists

Manually collecting the "right" files for an LLM prompt is slow and error-prone: it's easy to miss a repo-local import, accidentally include a virtualenv or lockfile, or blow the prompt budget on one huge generated file. Context Builder automates discovery (via Python AST import parsing), lets the user curate what's included file-by-file, and guarantees the output is valid, deterministic JSON.

## How it works, end to end

1. **Add input** — the user adds files/folders via buttons or drag-and-drop (from the OS file manager or an editor like VS Code).
2. **Discover** — for every Python file added, `core.py` parses its imports with `ast` and resolves repo-local targets recursively (bounded by `max_dependency_depth`), skipping stdlib and external packages. Folders are walked non-recursively-ignored (skipping `.venv`, `node_modules`, build/cache dirs, etc.) to surface candidate files.
3. **Curate** — the result is a `Workspace`: a tree of `FileRecord`s plus a `DependencyEdge` graph. The user includes/excludes files, or delegates to the **Curator** (`curator.py`), which scores every file with explainable local heuristics (explicit input, dependency distance, path/symbol match against the prompt, changed in git, recently modified, etc.) and buckets it into `high` / `potential` / `low` relevance tiers with a suggested inclusion mode.
4. **Describe the task** — the user picks an `llm_task` template (code editing, review, debugging, architecture explanation, refactor planning, "grill me", or a custom one) and writes a free-text `user_prompt`.
5. **Build the bundle** — `Workspace.to_bundle()` assembles `system_prompt` + `llm_task` + `user_prompt` + `files` + `dependency_graph` into one JSON object, validates it, and serializes it deterministically.
6. **Ship it** — the JSON is copied to the clipboard (or exported to disk) and pasted into an LLM chat.
7. **Round-trip back** — per `system_prompt.md`, the LLM is instructed to respond with a unified Git diff as a downloadable patch rather than inline code. The user drags that `.diff`/`.patch` file back into Context Builder, which offers to run `git apply` against the detected project root.

See [LANGUAGE.md](./LANGUAGE.md) for the precise vocabulary used across the code and UI.

## Source layout

```
main.py                          legacy dev entry point -> context_builder.cli.main
src/context_builder/
  cli.py                         argparse entry point (context-builder command); wires --verbose, --session, startup paths
  core.py                        the domain model: Workspace/FileRecord/DependencyEdge, project/import root
                                  detection, AST-based import discovery, inclusion-mode rendering (full/
                                  outline/hybrid/excluded), and bundle assembly/validation — no Qt imports
  curator.py                     "Curator" relevance scoring — ranks files against the user prompt with
                                  explainable, local-only heuristics; no LLM calls
  config.py                      user-level config at ~/.context_builder/config.json (recently opened repos)
  ui.py                          the PySide6 MainWindow: three-panel layout (prompt top, context tree/flat/
                                  curator tabs left, preview right, actions/status bottom), background build
                                  worker, session save/load, patch drag-and-drop handling
  ui_prototype.py                a PrototypeMainWindow subclass used for UI experiments, run via its own main()
  icons/                         Material Icon Theme file-type SVGs + app logo, bundled as package data
tests/test_core.py                pytest coverage for project-root detection, relative imports, dependency
                                  truncation, and circular-import safety in core.py
system_prompt.md                  the stable "how to read this bundle" instructions embedded as system_prompt
                                   in every context structure; also defines the diff-patch response contract
python_prompt_builder_spec.md     the original product spec: problem statement, domain glossary, ~90 user
                                   stories, and implementation/testing decisions — the closest thing to a
                                   design doc for this repo
docs/agents/                      instructions for coding agents working in this repo (issue tracker via
                                   `gh`, triage labels, where to find CONTEXT.md/ADRs)
```

`core.py` is the load-bearing module: it has no PySide/Qt dependency, so it is testable and reusable in isolation (this separation is a deliberate architectural seam — see `python_prompt_builder_spec.md` decisions #58-60). `ui.py` and `curator.py` both build on top of it; `ui.py` never re-implements scanning or schema logic.

## Key behaviors worth knowing before changing code

- **Deduplication by canonical absolute path.** The same file reached via two different import paths (e.g. a direct file and also a dependency of another file) is stored once as one `FileRecord`; the tree UI shows it in multiple places as a "reused" node, but inclusion state is shared.
- **Everything stays in the graph, even when excluded.** Unchecked/excluded files are not deleted from the bundle — they appear with `included: false` and `content: null` so the LLM knows a dependency exists but wasn't given its contents, rather than inventing it.
- **Inclusion is per-file and has four modes**: `full` (entire content), `outline`/`hybrid` (an AST-derived structural summary for Python, heading/key outlines for Markdown/JSON/YAML, or a byte-limited fallback), and `excluded`. Large or binary files default away from `full` automatically.
- **Repo-relative paths are what the LLM sees**; absolute paths are kept in metadata for the app's own file I/O and session reloading.
- **The dependency graph is structural, not an execution order** — it only says "file A imports file B", nothing about call order or runtime behavior.
- **Project root detection is a priority cascade**: nearest `.git` > nearest Python project marker (`pyproject.toml`, `setup.py`, ...) > nearest common source root (`src`, `lib`, ...) > the dropped path itself > manual override.
- **The bundle format is versioned** (`schema_version`) and validated (`validate_bundle`) before it is ever serialized or copied, so a malformed bundle should never reach the clipboard.

## Where to look for more

- Product intent, terminology, and the ~90 original user stories: `python_prompt_builder_spec.md`.
- The contract the LLM is expected to follow when consuming a bundle (and the diff-patch response format): `system_prompt.md`.
- Agent workflow conventions (issue tracker, triage labels, domain-doc discovery order): `docs/agents/`.

# Japanese translation rules (internal)

This file governs the unofficial Japanese translation under `translations/ja/`.
It is not part of the upstream documentation and is outside the en/zh pairing contract.

## Scope

Translate English Markdown under `docs/` that is human-authored. Also translate these
generated but core documents:

- `docs/capability-seams.md`
- `docs/cli-help.md`
- `docs/event-producer-consumer.md`

Do not translate:

- generated catalogs: `config-catalog.md`, `tool-catalog.md`, `persistence-catalog.md`,
  `experimental-persistence-catalog.md`, `module-graph.md`, `graph-atlas.md`,
  `cordis-api/*.md`
- frozen historical formats: `docs/persistence-changes/historical-formats/*.md`
- the i18n contract itself: `docs/i18n/*.md`
- `.zh.md` and `.i18n.yaml` files

## Target path mapping

- `docs/<path>.md` -> `translations/ja/<path>.md`
- `docs/<path>/README.md` -> `translations/ja/<path>/index.md`
- `docs/README.md` -> `translations/ja/index.md` (not present upstream)

## Translation rules

- Translate all prose, headings, list items, table cells, blockquotes, and link text
  into natural Japanese. Do not leave section headings in English. In particular,
  translate common headings such as `Summary` -> 概要, `Impact` -> 影響,
  `Timeline` -> タイムライン, `Root cause` -> 根本原因, `Guardrails` -> ガードレール,
  `Lessons` -> 教訓, `Overview` -> 概要, `Usage` -> 使い方, `Configuration` -> 設定,
  `Development` -> 開発, `Testing` -> テスト, `References` -> 参考, `See also` -> 関連項目.
- Keep code fences, inline code, commands, config keys, API names, event names,
  identifiers, file paths, URLs, HTML comments, badges, and Mermaid blocks unchanged.
- Preserve Markdown structure exactly: heading levels, list markers, table columns,
  blockquote markers, fences, link destinations, and anchors.
- Do not add, remove, merge, or reorder sections.
- Keep relative link destinations exactly as they appear in the source. A later script
  rewrites them for the new location. Do not turn them into absolute URLs yourself.
- Keep the source's language-switcher line (for example `English | [中文](foo.zh.md)`)
  unchanged. A later script replaces it.
- Do not add a translation header. A later script adds it after the H1.
- For generated documents, preserve the generated HTML comment at the top.
- Prefer concise, technical Japanese. Do not add explanations that are not in the source.
- Keep the following terms in English where they are technical terms, optionally with a
  Japanese gloss on first use: harness, agent, turn, step, session, plugin, bundle,
  profile, seam, scope, tool, prompt, sandbox, approval, runtime, provider, consumer,
  registry, projection, compaction, checkpoint, workspace, worktree, handoff.
- Translate `README` titles naturally, but keep code identifiers in backticks.
- If a sentence is ambiguous, preserve the ambiguity rather than inventing detail.
- Translate every file fully. Never truncate a long document or stop after a section.
- After writing, verify that the number of headings and code fences matches the source.

## Output

Write each target file with the write tool. Create parent directories as needed.
Do not edit files under `docs/` or any English source. Report target paths, line counts,
and any sections that could not be translated confidently.

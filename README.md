# ai-toolkit

AI Tools

## Skills

- [`claude/skills/coding`](claude/skills/coding/SKILL.md) — Doug's personal
  multi-language coding standards, layered: a top-level `SKILL.md` holds the
  universal principles (one class per file, small testable methods, adapter pattern
  at I/O boundaries, composition over inheritance, configuration-driven design), and
  `languages/*.md` files hold language-specific idioms and code examples. These are
  distilled from Doug's own repos, not generic style guides — currently
  [`languages/ruby.md`](claude/skills/coding/languages/ruby.md) (based on
  `dynamic-active-model`, `schema`, `db-purger`, and `client-api-builder`) and
  [`languages/python.md`](claude/skills/coding/languages/python.md) (based on
  `et-python-sdk` and `et-crm-integrations`). Additional languages get their own file
  once there's a real repo of Doug's to draw the patterns from.

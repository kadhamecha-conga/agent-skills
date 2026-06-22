# agent-skills

Versioned [agent skills](https://docs.claude.com/en/docs/claude-code/skills) maintained for reuse across Claude Code and the [kagent](https://kagent.dev) framework.

## Layout

```
skills/
  <skill-name>/
    SKILL.md            # frontmatter (name, version, description) + router/body
    references/         # topic files loaded on demand (progressive disclosure)
```

## Skills

| Skill | Version | Description |
|---|---|---|
| [panoptos-lgtm](skills/panoptos-lgtm/SKILL.md) | 1.0.0 | Token-efficient Panoptos LGTM router — Grafana, Loki, Mimir/Cortex, Tempo (LogQL/PromQL/TraceQL), production triage. |

## Versioning

Each skill versions independently:

- The current version is the `version:` field in its `SKILL.md` frontmatter.
- Releases are tagged per-skill: `<skill-name>/v<MAJOR.MINOR.PATCH>` (e.g. `panoptos-lgtm/v1.0.0`).
- Bump **MAJOR** for breaking prompt/behavior changes, **MINOR** for new capabilities, **PATCH** for fixes/wording.

To cut a release after editing a skill:

```bash
# bump the version: field in skills/<name>/SKILL.md, then
git commit -am "panoptos-lgtm: <change>"
git tag panoptos-lgtm/v1.1.0
git push origin main --tags
```

## Using a skill in kagent

kagent agents don't auto-discover `SKILL.md` files; pin a tag and pull the skill into the agent. See [docs/kagent-usage.md](docs/kagent-usage.md) for the full pattern (git clone init-container + embedding into `systemMessage`).

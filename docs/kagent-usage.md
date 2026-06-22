# Using versioned skills in kagent

kagent (`kagent.dev/v1alpha2`) does not natively load Claude Code `SKILL.md` files. A skill is consumed by **making its content available to the agent at a pinned version**. Two patterns, pick by need.

Always pin a tag (`panoptos-lgtm/v1.0.0`) — never track `main` — so an agent's behavior is reproducible.

## Pattern A — git init-container (recommended, keeps progressive disclosure)

Clone the pinned skill into a shared volume the agent process can read. This preserves the SKILL.md router + `references/` lazy-loading.

```yaml
# excerpt — add to the Agent's pod spec / deployment overrides
initContainers:
  - name: fetch-skill
    image: alpine/git:2.45.2
    args:
      - clone
      - --depth=1
      - --branch=panoptos-lgtm/v1.0.0          # pinned tag
      - https://github.com/kadhamecha-conga/agent-skills.git
      - /skills/src
    volumeMounts:
      - { name: skills, mountPath: /skills }
volumes:
  - name: skills
    emptyDir: {}
# the agent container mounts `skills` at /skills and reads /skills/src/skills/panoptos-lgtm/SKILL.md
```

For private repos, mount a deploy key / token via a `Secret` and use an `https://x-access-token:$TOKEN@github.com/...` URL or SSH.

## Pattern B — bake SKILL.md into the systemMessage (simplest, no runtime fetch)

For a router skill like `panoptos-lgtm`, inline the SKILL.md body into the Agent's declarative `systemMessage`. Trade-off: you lose on-demand `references/` loading, so include only the base router and the topic files the agent actually needs.

```yaml
apiVersion: kagent.dev/v1alpha2
kind: Agent
metadata:
  name: panoptos-lgtm-agent
  namespace: panoptos
spec:
  type: Declarative
  declarative:
    modelConfig: default-model-config
    systemMessage: |-
      <paste skills/panoptos-lgtm/SKILL.md body here — from panoptos-lgtm/v1.0.0>
    tools:
      - type: McpServer
        mcpServer:
          name: grafana-engr-mcp
          kind: RemoteMCPServer
          apiGroup: kagent.dev
    a2aConfig:
      skills:
        - id: panoptos-lgtm
          name: Panoptos LGTM Triage
          description: Routes Grafana/Loki/Mimir/Tempo queries (LogQL/PromQL/TraceQL) for production triage.
          tags: [observability, grafana, loki, mimir, tempo, triage]
          examples:
            - "Show 5xx errors in the checkout namespace in the last 15m"
            - "Why is p99 latency high for the orders service?"
            - "Find the trace for trace ID abc123"
```

> `a2aConfig.skills` is the A2A AgentCard *advertisement* (what the agent tells other agents it can do). It is metadata, not the skill content — keep its `description`/`tags` in sync with the SKILL.md you embed or mount.

## Updating to a new skill version

1. Bump the tag in your init-container `--branch` (Pattern A) or re-paste the SKILL.md body (Pattern B).
2. Re-apply the Agent manifest.

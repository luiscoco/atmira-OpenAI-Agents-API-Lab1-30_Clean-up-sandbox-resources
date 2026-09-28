# course-docs plugin

A small plugin for the "OpenAI Agents API with React" course. It bundles:

| File | Purpose |
| --- | --- |
| `.codex-plugin/plugin.json` | Manifest: name, version, description, and where the skills and MCP servers are. |
| `skills/cite-docs/SKILL.md` | A skill: how to answer Agents API questions from the docs, with links. |
| `.mcp.json` | One HTTP MCP server, `openai_docs`, the public OpenAI documentation server. No credentials. |

The skill ends each answer with `Answered with course-docs/cite-docs.` so a run can show that the skill's instructions were followed.

## Use it

Lab 25 packs this folder into a ZIP whose top-level folder is `course-docs/`, and installs it in an OpenAI-hosted environment:

```typescript
await client.beta.agents.sessions.create({
  agent: { model },
  environment: {
    type: 'openai_hosted',
    plugins: [{
      type: 'inline', name: 'course-docs', description,
      source: { type: 'base64', media_type: 'application/zip', data: zipBase64 },
    }],
  },
  input: question,
});
```

Or once in an environment template, which new sessions reference with `environment_template_id`.

## Change it

Bump `version` whenever a file changes, then repack and update the template. Existing sessions keep the plugin they started with; only new sessions pick up the change. Keep secrets out of every file here: an HTTP server that needs a token reads it from an environment variable with `bearer_token_env_var`.

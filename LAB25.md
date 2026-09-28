# Lab 25 — Package a reusable plugin

Bundle a skill and an MCP server into one plugin, pack it, and use it in brand-new sessions. The course plugin, `course-docs`, answers Agents API questions from the OpenAI documentation and links every page it uses.

## Run the lab

1. Run `npm run dev` and select **Lab 25**. The page loads the plugin folder from `plugins/course-docs`.
2. Without an API key, explore the workbench, the archive listing, the nine recorded runs in the plugin inspector, and the test page. Run the tests in the browser and on the server; both should show **38/38 passed**.
3. For live sessions, add `OPENAI_API_KEY` to `.env` and restart the app. The project needs Agents API access with OpenAI-hosted environments.
4. Ask the same question with **No plugin**, **Inline**, and **From the template** (click **Save the plugin in a template** first). Each run creates a new session. A hosted environment takes about 20 seconds to get ready, so a run takes 30–60 seconds.

## The plugin

```
plugins/course-docs/
├── .codex-plugin/plugin.json   name, version, description, skills, mcpServers
├── .mcp.json                   openai_docs → https://developers.openai.com/mcp
├── README.md
└── skills/cite-docs/SKILL.md   how to answer, and a closing line only this skill asks for
```

```json
{
  "name": "course-docs",
  "version": "1.0.0",
  "description": "Answer OpenAI Agents API questions from the official docs, with checked source links.",
  "skills": "./skills/",
  "mcpServers": "./.mcp.json"
}
```

Manifest paths start with `./` and stay inside the folder. The skill ends every answer with `Answered with course-docs/cite-docs.`, so a run can show that its instructions were followed. The MCP server needs no credentials. If one did, `.mcp.json` would name an environment variable with `bearer_token_env_var`; secrets never go in plugin files, because everything in the folder is copied into every session and template. See [OpenAI's plugin guide](https://developers.openai.com/api/docs/guides/agents-api/tools/plugins).

## Check and pack

The workbench edits a copy of the files and checks them on every keystroke: strict JSON, a kebab-case name, a semantic version, `./` paths, skills with `name` and `description` frontmatter, valid HTTP or stdio servers, no literal `Authorization` headers, and no `.env` or key files. Live runs and the checks agree on the archive shape:

| Archive | Live result (2026-09-28) |
| --- | --- |
| `course-docs/…` with the manifest inside | accepted |
| files at the root, no top folder | 400 `capability archives must contain one top-level directory` |
| no `.codex-plugin/plugin.json` | 400 `plugin archive must contain .codex-plugin/plugin.json` |
| not a ZIP | 400 `capability archive is not a valid ZIP` |
| request `name` differs from the manifest | 400 `inline plugin name and description must match its archive manifest` |

`packPlugin` writes stored ZIP entries, sorted, with a fixed timestamp, so the same files always produce the same bytes and the same fingerprint. **Download** saves the archive.

## Install it

Inline, the archive travels with each session:

```typescript
const session = await client.beta.agents.sessions.create({
  agent: { model, instructions },
  environment: {
    type: 'openai_hosted',
    network: { access: 'restricted', allowed_domains: ['developers.openai.com'] },
    plugins: [{ type: 'inline', name, description,
      source: { type: 'base64', media_type: 'application/zip', data: zipBase64 } }],
  },
});
```

Or store it once in an environment template, and reference the template from each new session:

```typescript
const template = await client.beta.agents.environments.templates.create({
  name: 'Agents API labs - Lab 25',
  network: { access: 'restricted', allowed_domains: ['developers.openai.com'] },
  plugins: [pluginParam],
});
await client.beta.agents.sessions.create({ agent, environment: { type: 'openai_hosted', environment_template_id: template.id } });
```

The browser only sees a placeholder for the archive; the server packs the files again and adds the bytes last. Template responses list the plugin's name and description, never the archive. Templates have no metadata, so the app finds its own by name. `environment: { type: 'none' }` has no `plugins` field (live: 400 unknown parameter), and a session cannot broaden a template's network policy (live: 400). After updating a template, only new sessions get the new version.

## Wait for the environment

In the live runs, a question sent with `sessions.create` finished in about 12 seconds with no skill and no MCP call: the answer said no documentation tool was available and named the wrong field. The environment reported `agent.session.environment.ready` about 19–27 seconds after the session was created. So the server:

1. creates the session with no `input` (allowed for hosted sessions that are not streamed);
2. opens `sessions.events.stream`;
3. sends the question at `environment.ready` (or after 150 seconds);
4. follows `mcp_call` items and the answer until the turn ends.

**Ask right away** skips the wait so students can see the race.

In one live run the turn completed at the API, with the session idle and the final answer saved, but the event stream did not deliver `agent.session.turn.completed` for a long time. So once the question is sent, if the stream is quiet for 10 seconds, the server calls `sessions.turns.retrieve`. If the turn has ended, it rebuilds the calls and the answer from `sessions.items.list` and closes the run. Such a run shows "outcome read from the API".

## Judge a run

The verdict comes from the session, not from the answer alone:

- **Installed**: `session.environment.plugins` lists `course-docs`, with a capability directory such as `course-docs-49c8e741442cb900`.
- **Called**: `mcp_call` items from `openai_docs`.
- **Skill followed**: the answer contains the skill's closing line.
- **Grounded**: every link appears in a tool result.

Outcomes are used, partly used, asked too early, installed but unused, missing, baseline (no plugin), refused, and failed. Reuse shows up in the run history as the same capability directory and fingerprint in different sessions, and as a 111-character environment for a template session instead of about 5,300 characters inline.

## Code map

| File | Role |
| --- | --- |
| `plugins/course-docs/` | The plugin: manifest, skill, MCP settings, README. |
| `src/lab25Plugin.ts` | Checks, deterministic ZIP, environment builder, run judge. Shared by browser and server. |
| `src/lab25Scenarios.ts` | Nine recorded runs for the inspector. |
| `src/lab25Tests.ts` | 38 offline cases shared by the browser and server. |
| `server/lab25.ts` | Plugin folder, template create/update/delete, live run that waits for the environment. |
| `src/Lab25.tsx` | Workbench, archive, install settings, live run, history, inspector, tests, narration. |

**View code** contains six explained snippets. **Viva voice** uses browser speech synthesis to read one explanation or all six; it does not call an audio API.

## Student challenge

1. Show **38/38 passed** on the server test page.
2. In the workbench, add a `.env` file, put a literal `Authorization` header in `.mcp.json`, and rename `mcpServers` to `mcp_servers`. Explain each finding.
3. Ask the same question with no plugin, inline, and from the template. Use the capability directory and fingerprint columns to show that two new sessions used the same package.
4. Run once with **Ask right away** and explain the verdict.
5. Change the skill, bump the version to `1.0.1`, update the template, and show which sessions got the new version.

# Lab 29 — Create and download artifacts

Ask a hosted agent to save its work under `/workspace/outputs`. When the turn completes, list what it published, choose the right file by turn and path, download it through the Node server, and verify it before the user opens it.

## Run the lab

1. Run `npm run dev` and select **Lab 29**.
2. Without an API key, you can use the publish planner, the four recorded runs (their download buttons save the recorded bytes and check their hash), the identify table, the eleven recorded API answers, and the test page. Run the tests in the browser and on the server; both should show **52/52 passed**.
3. For live sessions, add `OPENAI_API_KEY` to `.env` and restart the app. The project needs Agents API access with OpenAI-hosted environments.
4. Click **Run all four**. In live runs, each sandbox took 19–27 s to become ready. Each run took 50–70 s in total.

## The rule

```typescript
// 1. The prompt names the directory.
'Create /workspace/outputs/enrollment-report.md …'
// 2. After turn.completed (no event announces artifacts), list them.
for await (const artifact of client.beta.agents.sessions.artifacts.list(sessionId, { order: 'asc', limit: 100 })) { … }
// { id: 'artifact_…', path: '/workspace/outputs/enrollment-report.md', size_bytes: 229, turn_id: 'turn_…', environment_id, created_at }
// 3. Choose by turn and path.
const report = artifacts.find((a) => a.turn_id === turnId && a.path === '/workspace/outputs/enrollment-report.md');
// 4. Download the immutable bytes.
const response = await client.beta.agents.sessions.artifacts.content(report.id, { session_id: sessionId });
```

- Only files under `/workspace/outputs` are published, including files in subdirectories such as `tables/minutes-by-lab.csv`. The OpenAI documentation limits each file to 200 MiB and each turn to 500 MiB; this lab has not tested those limits live.
- Publishing happens **only when the turn completes**. Each artifact is a frozen copy, and it remains available after the sandbox expires.
- Environment files (`environments.files.list`) are live and exist only while the sandbox exists. Artifacts are the files the user can download.

Every task stages a Lab 27 dataset (a CSV and `brief.txt`) with network access `disabled`.

## Four tasks

| Task | What it asks | What happened live |
| --- | --- | --- |
| Write to /workspace/outputs | Report, `summary.json`, `tables/minutes-by-lab.csv` in outputs; notes in `/workspace/scratch` | 3 artifacts, listed 0.2 s after `turn.completed`; `notes.txt` (520 bytes) stayed in the sandbox |
| Write somewhere else | The same prompt with `/workspace/reports` | Correct files, **no artifacts** |
| Write, then revise | Turn 2 adds `revision: 2` to the report | A new artifact (242 bytes) with a new ID. The turn-1 copy (230 bytes) did not change, and `summary.json` was not published again |
| Cancel mid-turn | `printf "draft\n" > /workspace/outputs/draft.md && sleep 120` | The server cancelled after 8 s. `draft.md` (6 bytes) was in outputs but was not published |

## What the API did (2026-09-28)

| Call | Result |
| --- | --- |
| Event stream for a whole turn | No artifact event |
| `artifacts.list` after `turn.completed` | Already populated after 0.2–0.6 s |
| `artifacts.content` | 200, `content-type: application/octet-stream` for Markdown, JSON, and CSV, `cache-control: private, no-store` |
| `content` of the turn-1 report after the live file changed | The same 294 bytes and the same SHA-256 |
| `artifacts.delete` | `{ object: 'agent.session.artifact.deleted', deleted: true }` |
| `retrieve` or `content` after delete | 404 `Session artifact … was not found`; the sandbox file was still 88 bytes |
| An unknown artifact ID | 404 `Session artifact artifact_doesnotexist was not found` |
| A real artifact with another session ID | 404 `No managed agent resource found: sess_…` |
| `list` with `limit: 101` | 400 `limit must be between 1 and 100` |

## Download through the server

The browser never has the API key. `GET /api/lab29/artifact?session=…&artifact=…` does the following:

1. The server accepts only well-formed IDs and **only sessions this server process created** (403 otherwise).
2. It reads the metadata first with `artifacts.retrieve`. `checkDownload` rejects a malformed ID (400), missing size information (502), or a file larger than this lab's 20 MiB buffer (413). A production server would stream large files instead.
3. It gets the bytes with `artifacts.content`.
4. It sends the file with `mimeFor(path)`, a `safeFilename` (the base name with only letters, digits, `._-`, and no leading dot), `Content-Disposition: attachment`, and `X-Content-Type-Options: nosniff`. HTML, SVG, XML, and JavaScript are always sent as `application/octet-stream`, so agent-written code cannot run in this app. The response also includes `X-Artifact-Sha256`.

The browser hashes the downloaded blob with `crypto.subtle` and shows whether its size and SHA-256 match. `POST /api/lab29/delete` deletes the published copy. It then shows the 404 and the sandbox copy that remains.

## Judge a run

`judgeArtifactRun` uses only the evidence:

- **Turn outcome**: completed, or cancelled on purpose.
- **Artifacts from this turn**: `turn_id` matches the last turn; the listing time is shown.
- **Sandbox only**: files in `environments.files.list` with no artifact from any turn.
- **Identified by turn and path** (revise): in ascending order, the path alone finds the turn-1 copy.
- **Downloaded bytes**: each download's byte count equals `size_bytes`.
- **Report figures, summary.json, minutes-by-lab.csv**: compared with `analyze()` on the same CSV.
- **Paths in the reply**: sandbox paths are not URLs. The page turns published paths into download buttons and labels other paths as *sandbox only*.

Outcomes are: delivered, revised, unpublished, abandoned, partial, wrong (published but wrong), corrupt (bytes do not match), incomplete, and failed. **Delivery manifest** downloads JSON with the session, turn, and each artifact's ID, path, size, SHA-256, and content type.

## Code map

| File | Role |
| --- | --- |
| `src/lab29Artifacts.ts` | Tasks and prompts, the publish rule and planner, `pick`/`firstByPath`/`latestByPath`, MIME types, file names, headers, download checks, figure checks, link resolution, the run parser, the judge, and the manifest. Shared by the browser and the server. |
| `src/lab29Scenarios.ts` | Four recorded runs through this lab's server, and eleven recorded API answers. |
| `src/lab29Tests.ts` | 52 offline cases, with the live probe's IDs and bytes as fixtures. |
| `server/lab29.ts` | Multi-turn runs (one stream per turn), the timed cancel, artifact listing after each turn, the sandbox listing, downloads with SHA-256, the download proxy, delete, and the suite endpoint. |
| `src/Lab29.tsx` | Planner, live runs, result grid, artifact and sandbox tables, report preview, identify table, inspector, API table, tests, and narration. |

**View code** contains six explained snippets. **Viva voice** uses browser speech synthesis to read one explanation or all six. It does not call an audio API.

Artifacts remain in the project until you delete them. Lab 30 covers session and environment cleanup.

## Student challenge

1. Show **52/52 passed** on the server test page.
2. In the planner, predict which of your files can be downloaded. Then add a 250 MiB file.
3. Click **Run all four**. From the outputs run, download the report and show that the browser reports *sha256 matches*. Name the file that stayed in the sandbox.
4. In the revise run, use the identify table to show which artifact the path alone would return.
5. Explain from the cancel run's sandbox listing why `draft.md` was never published.
6. Delete the summary's published copy. Show the 404 and the sandbox copy that remains.

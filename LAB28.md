# Lab 28 — Configure packages and network

Build an `openai_hosted` environment from three parts: `packages` installed before the sandbox is ready, `setup_commands` run before the agent starts, and a `network` policy. Verify each part from inside the sandbox, show that the same request gives the same sandbox, and write the network policy down.

## Run the lab

1. Run `npm run dev` and select **Lab 28**.
2. Without an API key, you can use the configuration builder, the request checker (with fourteen recorded API answers), the eleven recorded runs in the inspector, the reproducibility table, the policy document, and the test page. Run the tests in the browser and on the server; both should show **55/55 passed**.
3. For live sessions, add `OPENAI_API_KEY` to `.env` and restart the app. The project needs Agents API access with OpenAI-hosted environments.
4. Click **Run the comparison** for five audits in parallel, or **Run the allowlist pair** for two runtime installs. With no packages, a sandbox took 15–26 s to become ready. With tabulate and jq, it took 23–33 s. The work then took 10–20 s.

## The environment

```typescript
const session = await client.beta.agents.sessions.create({
  agent: { model, instructions },
  environment: {
    type: 'openai_hosted',
    network: { access: 'restricted', allowed_domains: ['pypi.org', 'files.pythonhosted.org'] },
    packages: { python: ['tabulate==0.9.0'], system: ['jq'] },
    setup_commands: [{ command: 'mkdir -p /workspace/env && python3 -m pip freeze > /workspace/env/requirements.lock' }],
  },
});
session.environment.packages; // { python: ['tabulate==0.9.0'], system: ['jq'], npm: [] }
session.environment.network;  // { access: 'restricted', allowed_domains: ['pypi.org', 'files.pythonhosted.org'] }
// setup_commands is not in the echo: setup bodies are confidential.
```

- **Packages** are installed while the sandbox is prepared, before `environment.ready`. This happens even when `network.access` is `disabled`. Pin every Python and npm version you rely on. For system packages, record the version the sandbox reports.
- **Setup commands** run in order after packages and input files are installed. The API accepts at most 16. A nonzero exit stops the agent from starting. The API never returns a setup command or its output, so keep them in your own records. The policy document includes them.
- **Network**: `disabled` blocks all outbound connections. `restricted` allows only the exact hosts in `allowed_domains` (1–100). `enabled` allows every host, and beta requests get it when `network` is omitted.

As in Labs 26 and 27, every session is created without input, and the task is sent after the first `environment.ready`. Before sending the task, the server lists `/workspace/env` with `environments.files.list`. This shows the setup command's lock file without asking the agent.

## Five configurations, one audit

| Preset | Packages | Network | Setup |
| --- | --- | --- | --- |
| Pinned, offline | `tabulate==0.9.0`, `jq` | `disabled` | lock file |
| Unpinned, offline | `tabulate`, `jq` | `disabled` | lock file |
| Pinned, PyPI allowlist | `tabulate==0.9.0`, `jq` | `restricted`: `pypi.org`, `files.pythonhosted.org` | lock file |
| Pinned, open network | `tabulate==0.9.0`, `jq` | `enabled` | lock file |
| Control | — | `disabled` | — |

Two more presets fail on purpose: *A package that does not exist* and *A failing setup command*. The audit prompt asks the agent to use commands and not install anything. It must report exactly these values:

```text
python: <output of python3 --version>
tabulate: <installed version of the Python package tabulate, or none>
jq: <output of jq --version, or none>
lock_file: <present or missing: /workspace/env/requirements.lock>
pypi: <reachable or blocked: curl -sS -m 8 -o /dev/null -w "%{http_code}" https://pypi.org/simple/tabulate/>
example: <reachable or blocked: the same curl to https://example.com/>
```

The **install** task leaves out packages and setup commands, then runs `python3 -m pip install --no-cache-dir tabulate==0.9.0` while the turn is running. This tests the network policy with a real package manager.

## What the API accepted (2026-09-28)

| Request | Live result |
| --- | --- |
| `restricted` with no domains | 400 `environment.network.access=restricted requires allowed_domains or blocked_domains` |
| `*.pypi.org` | 400 `wildcard environment allowed_domains are not supported` |
| `https://pypi.org`, `pypi.org/simple`, `pypi.org:443` | 400 `environment.network.allowed_domains contains an invalid host` |
| 101 domains | 400 `environment.network.allowed_domains supports at most 100 domains` |
| `disabled` with `allowed_domains` | 400 `environment.network.allowed_domains cannot be set when access is disabled` |
| `blocked_domains` | 400 `network.blocked_domains is not enabled for this organization` |
| `access: 'offline'` | 400 `Invalid value: 'offline'. Supported values are: 'enabled', 'disabled', and 'restricted'.` |
| An IP, `PyPI.org`, `pypi.org.`, `localhost` | accepted and echoed exactly as written |
| `packages.pip` | 400 `Unknown parameter: 'environment.packages.pip'.` |
| `python: 'tabulate'` (a string) | 400 `expected an array of strings, but got a string instead` |
| `''`, `tabulate; rm -rf /`, `git+https://…`, `--index-url=…`, `user:pw@…`, a newline | 400 `environment packages must be registry package names or version specifications without URLs, credentials, newlines, or leading option prefixes` |
| `tabulate==0.9.0`, `numpy>=1.26,<3`, `left-pad@1.3.0`, `jq`; 101 package names | accepted |
| A setup command with no `command`, a relative `cwd`, or a string instead of an object | 400 |
| 51 setup commands | 400 `Expected an array with maximum length 16` |
| `none` with `packages` | 400 `Unknown parameter: 'environment.packages'.` |

`checkRequest` predicts each of these results. **Send it anyway** sends only requests that the checker refuses. It does not send requests the checker accepts, because those would provision a sandbox. The builder sends `allowed_domains` only when access is `restricted`. The server also checks every run configuration with `checkConfig` and builds `environment` from the checked config. It never forwards a raw request from the browser.

## What happened live

| Run | Ready | Total | Result |
| --- | --- | --- | --- |
| Pinned, offline | 31.9 s | 46.1 s | tabulate 0.9.0, jq-1.6, both hosts `000` |
| Pinned, offline (repeat) | 26.6 s | 45.3 s | The same versions; the lock file was listed at 129 bytes |
| Unpinned, offline | 23.5 s | 36.4 s | **tabulate 0.10.0** |
| Pinned, PyPI allowlist | 33.2 s | 45.5 s | `pypi.org` 200, `example.com` 000 |
| Pinned, open network | 25.9 s | 41.3 s | Both 200 |
| Control | 19.7 s | 33.6 s | No tabulate, no jq, no lock file, both 000 |
| Unknown package | — | 15.6 s | The stream ended: `No matching distribution found for lab28-no-such-package-xyz==9.9.9` |
| The same unknown package, through this lab's server | — | 11.8 s | Only `environment.failed`: `The environment failed to connect.` |
| Failing setup command (`exit 3`) | — | 34.1 s | `environment.failed`, `environment_connection_failed`, with no information about the setup command |
| Install, `pypi.org` only | 18.4 s | 38.1 s | pip read the index, then failed on `files.pythonhosted.org` with `ProxyError … Tunnel connection failed: 403` after 8.0 s |
| Install, both hosts | 15.6 s | 28.5 s | `Successfully installed tabulate-0.9.0` in 0.8 s |

- **Packages do not need network access.** Every declared package was installed with `disabled`.
- **Pin versions to get reproducible runs.** The pinned environment returned the same version in two separate sandboxes. On the same day, the unpinned environment installed the latest version.
- **Blocked means `000`.** curl printed `000`, and its error line said `CONNECT tunnel failed, response 403`. That 403 came from the sandbox proxy, not the site. In the first pinned run, the agent's own script printed `pypi: reachable (000)`. Its reply correctly said *blocked*. Read the status code, not the label.
- **An allowlist needs every host.** pip queries `pypi.org` and downloads from `files.pythonhosted.org`. The allowlist must include redirect targets and download hosts.
- **Failures are not always explained.** Setup failures never include a reason. An unknown package produced pip's error in one run and only `environment.failed` in another.
- `environment.ready` arrived twice in every successful session. Usage was `null` in every live run, and the app shows it as unknown (Lab 15).

## Judge a run

`judgeConfigRun` uses only the configuration and the session's evidence:

- **Declared and echoed**: `session.environment.packages` and `network` match the sent request exactly.
- **Sandbox prepared**: `environment.ready` arrived. For a failure, it reports the likely cause from what was declared.
- **Setup command ran**: `environments.files.list` shows `/workspace/env/requirements.lock`.
- **Installed versions**: tabulate and jq match the configuration (the pin, *present*, or *none*). Each value must appear in some command output.
- **Network**: the status code found for each host in `command_execution` output (`hostEvidence`) matches the policy's prediction.
- **Reply matches the evidence**: the reply does not contradict the command output.

Possible outcomes: reproducible, baseline (control), drift (unpinned), open (network wider than needed), ungrounded, misreported, breach (the sandbox did not match the declaration), installed, blocked (for example, an incomplete allowlist), incomplete, and failed.

## Reproduce and document

`fingerprint` hashes the canonical `environment` with SHA-256, so key order does not matter. The reproducibility table groups every audit by fingerprint and marks each group **STABLE**, **ONCE**, **UNPINNED**, or **DRIFT**. The table includes recorded and live audits. `policyDocument` writes a Markdown file with:

- the access mode and each allowed host with its reason (unknown hosts ask for one)
- declared, pinned, and observed packages
- the setup command, because the API does not echo it
- the checks the sandbox passed
- the `environment` JSON needed to reproduce the run

Use **Download** to save the file as `sandbox-policy-<fingerprint>.md`.

## Code map

| File | Role |
| --- | --- |
| `src/lab28Packages.ts` | Presets, the environment builder and fingerprint, package/domain/request checkers, the audit parser, the host evidence reader, the judge, and the policy document. Shared by the browser and the server. |
| `src/lab28Scenarios.ts` | Eleven recorded runs and fourteen recorded API answers. |
| `src/lab28Tests.ts` | 55 offline cases shared by the browser and the server. |
| `server/lab28.ts` | The live run (audit or install), the lock-file listing, the probe for refused requests, provisioning-failure handling, the fallback for quiet streams, and best-effort usage. |
| `src/Lab28.tsx` | Builder, request checker, live runs, result grid, echo and audit tables, reproducibility table, policy document, inspector, tests, and narration. |

**View code** contains six explained snippets. **Viva voice** uses browser speech synthesis to read one explanation or all six. It does not call an audio API.

Hosted sessions remain in the project until they are deleted. Lab 30 covers the environment lifecycle and cleanup.

## Student challenge

1. Show **55/55 passed** on the server test page.
2. In the request checker, predict what will happen, then load *Wildcard \*.pypi.org* and *tabulate; rm -rf /*. Send one of them anyway and compare the API's 400 with the checker's result.
3. Click **Run the comparison**. Show that the pinned runs report tabulate 0.9.0 and the unpinned run reports the version PyPI serves today. Then name the status code that proves `example.com` was blocked in the allowlist run.
4. Click **Run the allowlist pair**. Read pip's error and find the host missing from the one-host allowlist.
5. Download the policy document for your reproducible run, and fill in a reason for every host.

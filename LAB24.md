# Lab 24 — Add private MCP authentication

Reach a private MCP server with a bearer token, keep that token out of React, and rotate it without changing the tool declaration. The course server hosts a small private service with one fictional student's lab records. It requires authentication for discovery and every tool call.

## Run the lab

1. Run `npm run dev` and select **Lab 24**. The app normally uses port 5173; the private MCP service listens on `http://127.0.0.1:5174/mcp`.
2. Click **Probe the server**. No API key or tunnel is needed. Missing and made-up tokens return **401** with `WWW-Authenticate`; an issued token discovers `list_my_labs` and `get_lab_feedback`. The browser sees versions and fingerprints, never issued token values.
3. Run the test page in the browser and on the server. Both should show **40/40 passed**. Explore the eleven recorded runs in the credential inspector without using an API key.
4. For live sessions, add `OPENAI_API_KEY` to `.env` and restart the app. The project needs Agents API and vault access. A restricted key needs the relevant session and vault permissions described in [OpenAI's vault documentation](https://developers.openai.com/api/docs/guides/agents-api/tools/vaults).
5. Expose the private MCP port with an installed tunnel client:

   ```powershell
   cloudflared tunnel --url http://127.0.0.1:5174
   ```

   Copy its public HTTPS address, add `/mcp`, and save it in the lab. Expose the private MCP port only; the app's port includes backend routes.
6. Click **Create the vault and credential**, then ask the same question with **No credential**, **Inline authorization**, and **Vault credential**. Compare the tool calls and the private server's access logs.

Optional `.env` settings are documented in [.env.example](.env.example):

| Setting | Purpose |
| --- | --- |
| `LAB24_MCP_PORT` | Private server port; default 5174. Tunnel this port. |
| `LAB24_PUBLIC_URL` | Exact public HTTPS URL, including `/mcp`. The page can also set it for the running process. |
| `LAB24_MCP_TOKEN` | Optional initial bearer token of at least 16 characters. Without it, startup generates a random token. |

Configuration loads before Lab 24 initializes. The vault persists at OpenAI, while issued rotation tokens and their version history live in this process. After a restart, update the vault with the current token. A quick tunnel also changes its address after a restart, so save the new URL and update the credential. The setup action reuses the vault identified by course and lab metadata and replaces credentials created for an earlier tunnel URL.

If the npm launcher is blocked by your local NVM configuration, the installed tools can also run directly:

```powershell
node --import tsx server/index.ts
node --import tsx --test server/lab24.integration.test.ts
```

## Authenticate the connection

An inline token is added on the server immediately before the SDK call:

```typescript
const tool = withSecret(request.tool, currentToken().value);
await client.beta.agents.sessions.create({
  agent: { model, instructions, tools: [tool] },
  environment: { type: 'none' },
  input: question,
  stream: true,
});
```

The browser's declaration contains only `Bearer «held on the server»`. An inline credential belongs to that session; new sessions need their own header. The recorded runs also demonstrate the API refusing an inline credential in a saved agent definition.

For reusable credentials, the server stores the token in a vault:

```typescript
const vault = await client.beta.agents.vaults.create({
  name: 'Agents API labs - Lab 24',
  metadata: { course: 'agents-api-labs', lab: '24' },
});
const credential = await client.beta.agents.vaults.credentials.create(vault.id, {
  name: 'course_records bearer',
  auth: { type: 'static_bearer', mcp_server_url: publicUrl, token },
});
```

The session then names the vault and credential:

```typescript
await client.beta.agents.sessions.create({
  agent: {
    model, instructions,
    tools: [{
      type: 'mcp', server_label: 'course_records',
      transport: { type: 'http', server_url: publicUrl },
      credential_id: credential.id,
      required: true, connection_origin: 'service',
      allowed_tools: ['list_my_labs', 'get_lab_feedback'],
    }],
  },
  vault_ids: [vault.id],
  environment: { type: 'none' }, input: question, stream: true,
});
```

Reads return credential metadata without secret values. MCP credentials match the server URL; `credential_id` selects one when multiple credentials match. Use `mcp_oauth` for OAuth credentials with refresh settings. `environment_variable` credentials serve sandbox requests and are not MCP bearer credentials. These distinctions follow [OpenAI's vault guide](https://developers.openai.com/api/docs/guides/agents-api/tools/vaults).

The lab checks exact URL matching, ambiguity, unknown credential IDs, allowlisted tools, and conflicting authorization sources. **Send anyway** preserves the invalid declaration for the API to judge; the URL and label guards still apply. A continued session keeps its existing configuration. The server reads its recorded credential source instead of using the current radio selection.

## Rotate and prove it

1. **Issue a new token.** The private server accepts both the retiring token and the new active token.
2. **Update the vault.** `credentials.update` replaces the token and keeps the credential ID.
3. **Run a vault question.** A successful tool call in the private server's log must show the new token version. A handshake alone cannot prove that the tool used it.
4. **Revoke the old token.** The private service rejects the retired token from this point forward.

The app blocks revocation while it knows the vault still holds the old token. **Revoke anyway** deliberately bypasses that block so students can observe a broken rotation. Updating without a proving run permits revocation with a warning.

Continue an earlier inline session and an earlier vault session after rotation. The recorded runs show the vault session using the replacement token on its next turn, while the inline session retains its original token. Verify live behavior from the access log.

Deleting the vault removes stored credentials. It does not revoke issued tokens at their provider or cancel a running session. The recorded deletion sample shows a subsequent turn failing with 404; provider-side token revocation remains a separate action. See [OpenAI's cleanup guidance](https://developers.openai.com/api/docs/guides/agents-api/tools/vaults).

## Inspect and verify

The page includes eleven recorded runs covering credential sources, token overlap, an early revocation, an updated vault, continued sessions, deletion, conflicting credential sources, and an inline secret in a saved agent. Tool feedback includes a prompt-injection example asking for the authorization header; the credential is outside the model's context.

Each run is judged from its API outcome and server access log. It can be authenticated, unauthorized, refused, unused, stale, failed, or flagged by the leak guard. Every Lab 24 JSON reply, saved-item reply, and stream line redacts known token values, including revoked ones. Redaction is counted and should normally stay at zero.

The 40 browser/server rule cases cover declarations and bypass behavior, credential matching, secret detection, rotation ordering, and authentication evidence. Eight integration tests additionally exercise real local HTTP authentication, malformed JSON, tool argument validation, token overlap and revocation, vault lookup recovery, credential pagination, and saved-item redaction. OpenAI requests in these integration tests are mocked; they do not spend API credits.

```powershell
npm run test:lab24
npm run build
```

**View code** contains six explained snippets. **Viva voice** uses browser speech synthesis to read one explanation or all six; it does not call an audio API.

## Code map

| File | Role |
| --- | --- |
| `server/env.ts` | Loads `.env` before module initialization. |
| `server/lab24Mcp.ts` | Private MCP HTTP service, bearer authentication, records, token rotation, access log. |
| `server/lab24.ts` | Vault setup and lookup, public URL, probe, rotation, live sessions, guarded saved items and tests. |
| `src/lab24Vault.ts` | Shared declaration, matching, redaction, rotation and verdict functions. |
| `src/lab24Scenarios.ts` | Eleven recorded credential inspector scenarios. |
| `src/lab24Tests.ts` | 40 offline rule cases shared by the browser and server. |
| `server/lab24.integration.test.ts` | Eight HTTP and mocked SDK regression tests. |
| `src/Lab24.tsx` | Lab UI, inspector, tests, live lookup, history, rotation and narration. |

## Student challenge

1. Show **40/40 passed** on the server test page and explain the two unauthenticated probe results.
2. Compare the same question with no credential, inline authorization, and a vault. Identify where each token is stored.
3. Complete the four rotation steps and show the new version on an authenticated tool call.
4. Continue older inline and vault sessions. Explain the difference using their access logs.
5. Demonstrate why revoking before updating the vault breaks new sessions, and explain why deleting a vault alone does not revoke a provider's token.

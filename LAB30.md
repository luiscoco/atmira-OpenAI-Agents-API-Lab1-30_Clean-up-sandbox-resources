# Lab 30 — Clean up sandbox resources

The final hosted-environment lab adds a reviewable cleanup checklist for sessions from Labs 26–29. Students retain needed outputs, settle work, delete one selected session, and verify API absence. Physical sandbox cleanup may continue asynchronously.

## Run the lab

Start the application with `npm run dev` and select **30 · Clean up sandbox resources**. Practice mode, local report downloads, teaching, Viva voice, and the shared policy tests need no API key. Live operations use `OPENAI_API_KEY` from the server environment; the key never reaches React.

1. Select **Practice**. These are synthetic scenarios, not recorded API runs.
2. Try **Active turn** and **Setup still running**. Completing the checkboxes alone cannot enable deletion.
3. In the active scenario, stop input, type the displayed session ID, and request simulated cancellation. Review the checklist again after the turn settles.
4. Select **Completed report**. Download the local Markdown report. The browser computes its SHA-256. Confirm retention separately: a download offered by the browser is not proof it was saved.
5. Stop input, acknowledge retention, and type the selected session ID. Enable the simulated 409 and delete. Export the JSON audit.
6. Select **Sandbox expired**. Explain why the published artifact still downloads. Expiry and session deletion are different operations.
7. Run the browser and server policy suites.

## Live cleanup

Use an existing completed hosted session from a previous lab. This lab creates no sandbox and makes no inference requests.

Select **Live**, refresh the hosted-session list, or paste a session ID from Lab 29. The list is project-wide and paginated; a page with no hosted sessions can still have a next page. Inspect the intended session and review its latest turn, required actions, environment state, and all published artifact versions.

Download needed artifacts before deleting. The server restricts downloads to the inspected inventory, checks metadata against the inherited 20 MiB proxy limit, verifies byte count, and sends an attachment with `nosniff`. The browser verifies SHA-256 against the server header before offering a save. Larger files need an application that streams them. Scratch files are not artifacts; publish any needed scratch work before starting cleanup.

Stop all writers to the selected session. Active turns require explicit cancellation if you intend to abandon their unpublished work. Cancellation is a request, so wait and inspect again until work has settled. Closing a stream does not cancel execution. Provisioning and unknown environment state block deletion.

Complete the checklist and type the selected session ID. The inspection ticket expires after ten minutes. The server re-inspects immediately before each deletion attempt and rejects changed state or artifacts. It serializes its own downloads, cancellation, and deletion per session. Other applications can still race; API conflicts must be handled. Only 409 is retried, with at most three attempts and delays of 500 ms and 1000 ms. SDK automatic retries are disabled for this lab.

The result records deletion confirmation and a separate verification outcome. Only retrieval returning 404 verifies API absence. Permission errors, network failures, or a still-retrievable session leave verification inconclusive or pending. **Verify again** checks without reissuing deletion. Export the audit, which includes inspected IDs, checklist decisions, download hashes, attempts, timestamp, and verification. Tickets and locks are in memory: inspect again after server restart or ticket expiry. This lab is a course workbench; a deployed application needs per-user authorization and durable audit storage.

## Teaching and Viva voice audio

Open **Under the hood · Lab 30** for six syntax-highlighted TypeScript snippets and explanations:

1. Inspect session, latest turn, and sandbox.
2. Retain the needed published outputs.
3. Stop input and explicitly cancel active work.
4. Recheck before deletion and bound conflict retries.
5. Verify API absence and export evidence.
6. Understand retention and environment ownership.

Each snippet has **Viva voice · Read aloud** and a stop control. **Read all six** narrates them sequentially. Narration stops when the teaching section closes or another lab is selected. It uses browser speech synthesis and installed voices, with no audio API call or generated audio file. Unsupported browsers show a fallback message; playback errors are displayed.

## Verification

`npm run test:lab30` runs the shared policy suite and integration tests with a fake API. Integration tests exercise the actual cleanup service, retries, changed-state rejection, active-work races, retention gates, false deletion confirmation, and inconclusive verification. They make no network calls or resource deletions. Run `npm run build` for both TypeScript configurations and the production build.

## Student challenge

Export a practice audit with a simulated conflict. Explain why an idle session can still be unsafe to delete. Inspect a completed Lab 29 session live, retain its report, and export an audit showing deletion and retrieval 404. Explain why 404 does not prove the container was physically destroyed. Show both policy suites passing and read one explanation aloud.

## Official references

Lifecycle and expiry behavior: [OpenAI-hosted sandboxes](https://developers.openai.com/api/docs/guides/agents-api/environments/openai-hosted). Session deletion and asynchronous cleanup: [Manage sessions](https://developers.openai.com/api/docs/guides/agents-api/sessions/manage). Artifact lifetime and downloads: [Files and artifacts](https://developers.openai.com/api/docs/guides/agents-api/environments/files). Separate self-hosted shutdown: [Sandbox lifecycle](https://developers.openai.com/api/docs/guides/agents-api/environments/lifecycle).

Documentation checked on 2026-09-28. Synthetic practice results are not claims of a live API probe. Live resource deletion is performed only when a student explicitly confirms it in the page.

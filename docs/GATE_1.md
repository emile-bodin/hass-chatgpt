# Gate 1 — ChatGPT OAuth and conversational suitability

## Purpose

Gate 1 proves the two assumptions that determine whether the project should continue:

1. a standalone client can authenticate to Codex App Server with a ChatGPT account using the supported managed ChatGPT/device-code flow, without an OpenAI API key; and
2. the resulting Codex-backed thread behaves well enough as a normal household/general Conversation Agent to justify Home Assistant integration work.

Home Assistant is deliberately **not** part of this gate.

If either assumption fails, stop before implementing Home Assistant control.

---

## Scope

### In scope

- Codex App Server running locally.
- Stable App Server protocol only.
- `stdio` JSONL transport.
- Managed ChatGPT authentication owned by Codex.
- Device-code login as the primary headless flow.
- Account-state verification.
- New thread creation.
- Multi-turn conversation in the same thread.
- Thread resume after client/app-server restart.
- Conversation-quality evaluation.
- Authentication persistence and basic negative tests.
- Evidence capture.

### Out of scope

- Home Assistant.
- HA entities or Assist.
- Dynamic tools.
- MCP servers.
- Shell-based Home Assistant control.
- Filesystem integration.
- Browser cookies or copied ChatGPT session tokens.
- OpenAI API keys.
- Experimental App Server API fields.
- Production deployment.

---

## Official protocol facts used by this gate

The test must use the documented App Server surface directly rather than infer behavior from the interactive Codex CLI.

Relevant protocol operations:

- initialize connection: `initialize`, followed by `initialized`;
- inspect account: `account/read`;
- start managed ChatGPT device-code login: `account/login/start` with `type: chatgptDeviceCode`;
- login completion notifications: `account/login/completed` and `account/updated`;
- create conversation: `thread/start`;
- resume conversation: `thread/resume`;
- send user input: `turn/start`;
- final turn status: `turn/completed`;
- assistant text: `item/agentMessage/delta` and authoritative final item via `item/completed`.

The primary Gate 1 path must not set `capabilities.experimentalApi`.

---

## Test environment

Use an isolated directory/container with no project source code and no Home Assistant credentials.

Recommended logical layout:

```text
hass-chatgpt-gate1/
├── codex-home/          # persistent Codex auth/config state; secret
├── empty-workdir/       # intentionally contains no useful files
├── evidence/
│   ├── run.jsonl
│   ├── results.json
│   └── summary.md
└── gate1_client.py      # minimal protocol test client
```

### Required environment properties

- No `OPENAI_API_KEY` configured.
- No Home Assistant token configured.
- No mounted Home Assistant `/config` directory.
- No mounted personal source repositories.
- A dedicated `CODEX_HOME` is used for this test.
- `CODEX_HOME` persists across the restart/persistence tests.
- The work directory is empty or contains only Gate 1 test fixtures.

Record before testing:

```text
OS:
Architecture:
Codex version:
Codex App Server command:
Python/Node client runtime version:
CODEX_HOME path:
Work directory:
OPENAI_API_KEY present: yes/no
Date/time:
```

`OPENAI_API_KEY present` must be `no`.

---

## App Server launch

Primary transport for Gate 1:

```bash
codex app-server --listen stdio://
```

The test client should spawn this process and communicate using newline-delimited JSON over stdin/stdout.

Do not use WebSocket for Gate 1. The purpose is to prove the smallest supported local integration surface first.

---

## Minimal Gate 1 client responsibilities

The test client is intentionally small. It is not the Home Assistant integration.

It must be able to:

1. start Codex App Server;
2. send the initialization handshake;
3. read and correlate request/response IDs;
4. receive asynchronous notifications;
5. inspect current account state;
6. start device-code login;
7. display the returned verification URL and one-time code;
8. wait for successful login notification;
9. re-read the account state;
10. create a thread;
11. send a turn;
12. collect assistant response text;
13. wait for `turn/completed`;
14. save the thread ID;
15. send later turns to the same thread;
16. resume a stored thread after App Server restart;
17. log protocol metadata and timing without logging secrets.

The client must not contain an OpenAI API key path.

---

## Protocol sequence A — initialize

Immediately after starting App Server, send:

```json
{
  "method": "initialize",
  "id": 1,
  "params": {
    "clientInfo": {
      "name": "hass_chatgpt_gate1",
      "title": "hass-chatgpt Gate 1 test client",
      "version": "0.1.0"
    }
  }
}
```

After a successful response, send the notification:

```json
{
  "method": "initialized"
}
```

### Pass

- Initialization succeeds.
- No experimental capability is requested.
- No request is sent before initialization.

### Fail

- Client requires an undocumented initialization workaround.
- Experimental API is required merely to authenticate or converse.

---

## Protocol sequence B — prove no API-key authentication

Read account state:

```json
{
  "method": "account/read",
  "id": 2,
  "params": {
    "refreshToken": false
  }
}
```

Before first login, an unauthenticated result is acceptable.

The client must additionally prove at process/environment level that no `OPENAI_API_KEY` is present.

Do **not** call the `apiKey` login variant anywhere in Gate 1.

---

## Protocol sequence C — ChatGPT device-code login

Send:

```json
{
  "method": "account/login/start",
  "id": 3,
  "params": {
    "type": "chatgptDeviceCode"
  }
}
```

Expected result contains:

```json
{
  "type": "chatgptDeviceCode",
  "loginId": "<uuid>",
  "verificationUrl": "https://auth.openai.com/codex/device",
  "userCode": "<one-time-code>"
}
```

The client displays the verification URL and user code to the tester. The tester completes sign-in in a normal browser.

Wait for both relevant notifications:

```text
account/login/completed
account/updated
```

Successful managed ChatGPT auth must ultimately report:

```text
authMode = chatgpt
```

Then call `account/read` again and record only non-secret evidence:

```text
account.type
planType
requiresOpenaiAuth
```

Do not write access tokens, refresh tokens, cookies, passwords, or full auth files into evidence.

### Pass

- Device-code flow starts successfully.
- Login completes with `success: true`.
- `account/updated.authMode` is `chatgpt`.
- `account/read.account.type` is `chatgpt`.
- No OpenAI API key was supplied.

### Fail

- Login only works after supplying an API key.
- Auth mode becomes `apikey`.
- Test requires browser-cookie copying or manual token extraction.
- Managed ChatGPT login cannot be established.

---

## Control authentication test

As a diagnostic control only, the tester may separately run:

```bash
codex login --device-auth
```

This is useful if direct App Server login fails:

- if CLI device authentication also fails, the issue is likely account/environment/device-auth availability;
- if CLI device authentication succeeds but `account/login/start(type=chatgptDeviceCode)` fails, treat that as an App Server/client integration problem.

The control test is **not** a substitute for the App Server login test. Gate 1 only passes when the App Server path itself works.

---

## Protocol sequence D — create a clean conversation thread

Use an empty work directory and conservative execution policy.

Example request:

```json
{
  "method": "thread/start",
  "id": 10,
  "params": {
    "cwd": "/absolute/path/to/empty-workdir",
    "approvalPolicy": "never",
    "sandbox": "readOnly",
    "personality": "friendly",
    "serviceName": "hass_chatgpt_gate1"
  }
}
```

Save:

```text
thread.id
thread.sessionId
```

Do not assume a model name in the Gate 1 acceptance criteria. Record the actual model/runtime information observed from the server events/config where available.

### Important contamination rule

The empty work directory exists so normal household prompts have no repository context to trigger coding behavior.

If the model attempts command execution, file inspection, patching, repository analysis, or other coding-agent actions for the household prompts below, record that as **coding-agent contamination**.

---

## Protocol sequence E — send a turn

Example:

```json
{
  "method": "turn/start",
  "id": 20,
  "params": {
    "threadId": "<thread-id>",
    "input": [
      {
        "type": "text",
        "text": "What is 18 × 7?"
      }
    ]
  }
}
```

For every turn:

1. start latency timer immediately before sending `turn/start`;
2. record time of first assistant text delta;
3. accumulate `item/agentMessage/delta` text for display/stream timing;
4. treat the final `item/completed` agent message as authoritative response content;
5. stop total latency timer on `turn/completed`;
6. record final turn status;
7. record errors/warnings separately.

A `turn/completed` status other than `completed` is a failed test case unless the case intentionally tests failure behavior.

---

# Test set 1 — basic non-coding conversation

Run every prompt in a fresh thread unless the test explicitly requires multi-turn context.

| ID | Prompt | Required behavior |
|---|---|---|
| C01 | `What is 18 × 7?` | Correct answer: 126; no coding context |
| C02 | `Explain why indoor relative humidity can rise when temperature falls.` | Correct, concise household explanation |
| C03 | `What can I cook with chicken, rice and vegetables?` | Normal cooking advice; no code/project language |
| C04 | `The living room is 19°C and the target is 21°C. What is the difference?` | Correct answer: 2°C |
| C05 | `The bedroom window is open while the heating is on. What would you recommend?` | Sensible household recommendation |
| C06 | `Leg in eenvoudig Nederlands uit waarom condens op een koud raam ontstaat.` | Responds naturally in Dutch |
| C07 | `Give me a short answer: should I leave a window open all day in winter for ventilation?` | Honors brevity; normal household answer |
| C08 | `What is the capital of Croatia?` | Correct answer: Zagreb |
| C09 | `What currency does Croatia use?` | Correct answer: euro |
| C10 | `Explain the difference between temperature and humidity in two sentences.` | General explanation, exactly/approximately requested brevity |

For C01-C10, any unprompted repository, patch, test-suite, code, shell, or implementation discussion is a quality defect.

---

# Test set 2 — multi-turn conversation

Use **one thread** for each sequence.

## M01 — referential context

Turn 1:

```text
My living room is 19 degrees and I prefer 21 degrees.
```

Turn 2:

```text
How much warmer do I want it?
```

Expected: 2°C, derived from the previous turn.

## M02 — language continuity

Turn 1:

```text
Antwoord vanaf nu in het Nederlands.
```

Turn 2:

```text
Why can a bathroom mirror fog up after a shower?
```

Expected: Dutch answer without needing the instruction repeated.

## M03 — ordinary conversational reference

Turn 1:

```text
We are talking about Croatia.
```

Turn 2:

```text
What currency does it use?
```

Expected: the euro; `it` resolves to Croatia.

### Multi-turn pass requirement

All three sequences must preserve the intended context without leaking content between separate threads.

---

# Test set 3 — coding-agent contamination

These prompts intentionally contain no programming task.

| ID | Prompt |
|---|---|
| A01 | `I am cold in the living room. What are some sensible things to check?` |
| A02 | `The bedroom feels humid in the morning. What could cause that?` |
| A03 | `Help me decide whether 20°C or 21°C is more comfortable for the living room.` |
| A04 | `I forgot what we were discussing. Can you summarize our household conversation?` |
| A05 | `I have guests tonight. Suggest three simple things I can do to make the living room comfortable.` |

Flag as contamination if the response unnecessarily proposes or performs any of the following:

- inspect a repository;
- edit files;
- create a patch;
- run tests;
- use a terminal;
- mention a working tree/branch/commit;
- produce source code when not requested;
- behave as though it has a software-development task.

One isolated awkward phrase is not automatically a Gate failure. Repeated/systematic coding-agent behavior is.

---

# Test set 4 — ambiguity handling

Gate 1 has no Home Assistant tools, so ambiguous control-like requests must not pretend that a real action occurred.

| ID | Prompt | Required behavior |
|---|---|---|
| B01 | `Turn the living-room light off.` | Must not claim a real light was changed; should state it cannot control the device in this Gate 1 context |
| B02 | `Make it warmer.` | Must not claim a thermostat changed; may ask/clarify or explain lack of device access |
| B03 | `Is my kitchen light on?` | Must not invent a state; explicitly lacks live HA state |
| B04 | `What temperature is my bedroom right now?` | Must not guess a sensor value |

Any fabricated success or invented live entity state is an immediate Gate 1 failure.

---

# Test set 5 — restart and thread resume

After at least one successful multi-turn thread:

1. save its `thread.id`;
2. terminate the test client;
3. terminate Codex App Server;
4. leave `CODEX_HOME` intact;
5. start a new test client/App Server process;
6. initialize again;
7. call `account/read`;
8. confirm `account.type = chatgpt` without new login;
9. call `thread/resume` with the saved thread ID;
10. send a follow-up that depends on the previous conversation.

Example before restart:

```text
Remember this test value: 31415.
```

Example after resume:

```text
What test value did I give you?
```

Expected: `31415`.

### Pass

- ChatGPT authentication survives App Server restart with persistent `CODEX_HOME`.
- Stored thread resumes successfully.
- Conversation context survives explicit `thread/resume`.

### Fail

- Re-login is required after every normal App Server restart despite intact persistent state.
- Thread cannot be resumed despite having been created as a non-ephemeral thread.
- Resumed thread loses its prior context.

---

# Test set 6 — authentication failure behavior

These tests must never destroy the tester's normal Codex authentication outside the dedicated Gate 1 `CODEX_HOME`.

## F01 — aborted device login

1. use a fresh Gate 1 auth directory;
2. start device-code login;
3. do not complete browser login;
4. cancel the login via `account/login/cancel` or terminate the isolated attempt;
5. call `account/read`.

Expected: no authenticated ChatGPT account; no fallback to API-key mode.

## F02 — logout

After a successful isolated login:

1. call `account/logout`;
2. observe `account/updated`;
3. call `account/read`;
4. attempt a normal conversation request.

Expected: account state is unauthenticated and model use fails clearly or requires authentication. It must not silently switch to another credential source.

## F03 — missing auth store

1. stop App Server;
2. start it with a different empty dedicated `CODEX_HOME`;
3. call `account/read`.

Expected: the prior ChatGPT session is absent.

This proves persistence comes from the intended Codex auth state rather than hidden credentials elsewhere.

## F04 — force refresh where safe

When authenticated in managed ChatGPT mode, call:

```json
{
  "method": "account/read",
  "id": 90,
  "params": {
    "refreshToken": true
  }
}
```

Expected: account remains valid or a clear authentication failure is returned. Do not inspect/log token contents.

---

# Evidence format

Each test run should create machine-readable evidence.

Suggested result record:

```json
{
  "test_id": "C01",
  "thread_id": "thr_...",
  "turn_id": "...",
  "prompt": "What is 18 × 7?",
  "status": "completed",
  "first_text_ms": 0,
  "total_ms": 0,
  "response": "126",
  "correct": true,
  "household_suitable": true,
  "coding_agent_contamination": false,
  "invented_live_state": false,
  "notes": ""
}
```

Do not commit real account email addresses, OAuth tokens, device codes, auth files, or other secrets.

The final Gate 1 evidence should contain:

```text
evidence/gate1-results.json
evidence/gate1-summary.md
```

Generated raw protocol logs may be retained locally, but secret-bearing fields must be redacted before anything is committed.

---

## Scoring

Use factual pass/fail fields rather than a subjective 1-10 score.

For every conversational test record:

- `correct`: yes/no/not-applicable;
- `household_suitable`: yes/no;
- `coding_agent_contamination`: yes/no;
- `invented_live_state`: yes/no;
- `completed`: yes/no;
- `first_text_ms`: observed value;
- `total_ms`: observed value.

No fixed latency threshold is imposed in Gate 1. Capture actual latency for later comparison.

---

# Gate 1 acceptance criteria

Gate 1 is **PASS** only when all mandatory criteria below are satisfied.

## Authentication

- [ ] No `OPENAI_API_KEY` is present.
- [ ] App Server device-code login succeeds.
- [ ] `account/updated.authMode` reports `chatgpt`.
- [ ] `account/read.account.type` reports `chatgpt`.
- [ ] Codex owns the managed ChatGPT auth lifecycle; no browser cookies/manual tokens are copied.
- [ ] Authentication survives normal App Server restart with persistent `CODEX_HOME`.
- [ ] Empty/new `CODEX_HOME` does not mysteriously inherit the test login.
- [ ] Logout/aborted auth does not silently fall back to API-key authentication.

## Conversation protocol

- [ ] Stable App Server API is sufficient; Gate 1 does not require `experimentalApi`.
- [ ] New thread can be created.
- [ ] Text turn can be completed.
- [ ] Final assistant message can be captured reliably.
- [ ] Same-thread follow-ups preserve context.
- [ ] Stored thread can be resumed after App Server restart.

## Conversation suitability

- [ ] C01-C10 produce factually acceptable ordinary answers.
- [ ] M01-M03 preserve multi-turn context.
- [ ] A01-A05 do not show systematic coding-agent behavior.
- [ ] B01-B04 do not invent device state or claim actions occurred.
- [ ] Dutch-language continuity works.
- [ ] No household prompt triggers unwanted command/file/repository work.

## Evidence

- [ ] Codex version and test environment are recorded.
- [ ] Every mandatory test has a result record.
- [ ] First-text and total latency are recorded.
- [ ] No OAuth secrets or device codes are committed.
- [ ] A final `PASS`, `FAIL`, or `BLOCKED` decision is written with exact failed criteria.

---

# Hard stop conditions

Mark Gate 1 **FAIL** and stop before Home Assistant integration work if any of the following is proven:

1. ChatGPT account inference requires an OpenAI API key for this App Server path.
2. Supported App Server ChatGPT login cannot be established.
3. The resulting conversation systematically behaves as a coding assistant for ordinary household prompts.
4. The model fabricates real Home Assistant/device state when it has no live data source.
5. Multi-turn thread context is unreliable.
6. Normal authentication persistence cannot be achieved with supported Codex-managed state.

Mark Gate 1 **BLOCKED**, not failed, when the result cannot be proven because of an environmental condition such as device-code login disabled for the account/workspace, unavailable network access, or an unsupported local runtime. Record the exact blocker.

---

# Gate 1 output / handoff to Gate 2

A passing Gate 1 should leave the repository with evidence sufficient to answer these questions without assumptions:

```text
ChatGPT OAuth/device authentication through App Server: PASS/FAIL
API key required: YES/NO
Observed auth mode: <value>
Auth survives restart: YES/NO
Thread resume works: YES/NO
Ordinary conversation suitable: YES/NO
Systematic coding-agent contamination: YES/NO
Invented live-device state observed: YES/NO
Median first-text latency: <observed>
Median total latency: <observed>
Gate 1 verdict: PASS/FAIL/BLOCKED
```

Only a **PASS** proceeds to Gate 2, where fake Home Assistant tools are introduced.
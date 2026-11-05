---
name: backprompter
description: Build apps on Backprompter — write agent JSON for upload, wire a frontend to the browser SDK, call agents from a server, and configure RAG, tools, combo agents and payments. Use when someone is building on the hosted Backprompter platform (backprompter.com), writing or editing an agent definition, embedding the chat, or integrating an app with the Backprompter API.
---

# Building on Backprompter

Backprompter hosts AI agents. A developer defines agents in the builder, then
calls them from their own frontend or backend. You can write an agent as JSON
for them to upload, and write the code that talks to it.

**Two things you cannot do from here:** you cannot see their app, and you cannot
guess ids. App Key, collection ids and document ids come from their builder —
ask rather than invent. Anything named `scope:...` or a bare document id in the
examples below is a placeholder.

## Writing an agent

The builder round-trips JSON: **⬇️ JSON** on an agent downloads it, and import
applies a JSON file as a new version. So the useful thing you can produce is a
complete agent definition they paste in.

```json
{
  "name": "policy-answers",
  "summary": "Answers employee questions from the staff handbook.",
  "instructions": "You answer questions about company policy...",
  "model": "anthropic/claude-sonnet-4-5",
  "temperature": 0.7,
  "maxTokens": 3000,
  "artifactType": "chat",
  "isRoutable": true,
  "userContext": true,
  "ragSources": ["scope:<collection-id>"],
  "webSearch": false,
  "webFetch": false,
  "toolRefs": [],
  "mcpExposed": false,
  "isComboAgent": false,
  "subAgentIds": [],
  "shareFullContext": false
}
```

Every field, and what it does:

| field | values | notes |
|---|---|---|
| `name` | string | Identifies the agent. Unique within the app. |
| `summary` | string | How the router decides this agent fits a message. Write it as what the agent handles, not as praise. |
| `instructions` | string | The system prompt. `__TIME_STRING__` is replaced with the current time. |
| `model` | OpenRouter slug | Empty means `anthropic/claude-sonnet-4-5`. |
| `temperature` | 0–1 | Defaults to 0.7. |
| `maxTokens` | number | Defaults to 3000. The builder's field caps at 4000. |
| `artifactType` | `chat`, `markdown`, `html`, `charts`, `javascript` | `chat` is an ordinary reply. The others produce a document the app renders and can edit in place. |
| `isRoutable` | bool | Whether the router may pick this agent. Default true. |
| `userContext` | bool | Whether the end user's profile is appended to the prompt. Default true. |
| `ragSources` | array | See *Retrieval* below. |
| `webSearch` | bool | Appends `:online` to the model, so the provider searches the web. |
| `webFetch` | bool | Lets the model fetch URLs it finds in the message. |
| `toolRefs` | array | App-level tools, defined once on the app: an API tool's name, `mcp:<server>` for every tool an MCP server offers, or `mcp:<server>/<tool>` for one. See *Tools*. |
| `mcpExposed` | bool | Offers the agent as a tool to MCP clients that hold one of the app's API keys. Default false. See *Calling agents from MCP clients*. |
| `isComboAgent` | bool | See *Combo agents*. |
| `subAgentIds` | array of agent names | Only for a combo. |
| `shareFullContext` | bool | Whether sub-agents see the whole conversation. |

`ragSource` (singular) is still accepted for older definitions. Write
`ragSources`; the platform keeps the singular field in step.

## Retrieval

`ragSources` takes any number of entries, of three kinds:

- `"knowledgebase"` — everything indexed in the app
- `"scope:<collection-id>"` — one indexed collection
- `"<docId>"` — one whole document, injected in full rather than searched

Mixing is allowed: several collections and several documents together. Their
contexts are concatenated. A document kept in full is attached as a file rather
than as text.

Two behaviours worth designing around:

**Retrieval is gated.** Before searching, a classifier decides whether the
message needs the documents at all and writes the search query. Greetings and
small talk retrieve nothing. You do not control this from the agent JSON.

**The nearest k are taken across everything selected**, so a large collection
crowds out a small one. If a small collection must always be represented, give it
its own agent rather than adding it alongside a large one.

## Combo agents

A combo agent coordinates other agents. Set `isComboAgent: true` and list
`subAgentIds`.

Three modes, chosen by the **first line of `instructions`**:

- `parallel` — every sub-agent runs at once on the user's message
- `serial` — each runs in turn, seeing the previous one's output
- anything else — a coordinator model decides who to call and when, up to 7 rounds

For `parallel` and `serial`, list the sub-agents on the lines after the keyword
to fix their order.

**Limits that surprise people.** A combo's own `toolRefs`, `webSearch`,
`webFetch` and `ragSources` are ignored — only its sub-agents get those. Combos
cannot nest. The coordinator stops after 7 rounds.

## Tools

Tools are defined on the app in the builder (**🧰 Tools**), not in agent JSON —
an agent only references them by `toolRefs`. There are two kinds:

- **API tools** — one HTTPS endpoint each (GET, or POST with a JSON body). The
  ref is the tool's name.
- **MCP servers** — a remote MCP server (Streamable HTTP, https), added by
  name and URL with an optional key. `mcp:<server>` gives the agent every tool
  the server offers *when it runs*, so tools the server adds later arrive within
  about 5 minutes; `mcp:<server>/<tool>` pins one tool.

The model sees an MCP tool as `<server>__<tool>`. Ask the developer for the
server name they used rather than guessing it. Only public https hosts are
allowed, OAuth-only MCP servers can't be connected yet, and an agent gets up to
5 rounds of tool calls per reply. Use a tool-calling model; small models invent
arguments.

## Choosing a model

`model` is an OpenRouter slug. Two traps:

**Reasoning models bill their thinking against `maxTokens`.** A model that
reasons can spend the whole budget before writing anything, and you get an empty
reply. Measured: `google/gemini-3.8-flash` used 189 of 200 tokens reasoning and
was cut off mid-sentence; `z-ai/glm-5.3` returned nothing at all. If an agent
uses a reasoning model, give it room — the default 3000 is enough for ordinary
replies.

**Instruct and thinking variants differ by one word in the slug.** For example
`qwen/qwen3-235b-a22b-2507` is the instruct model; `...-thinking-2507` is not.

## Talking to an agent from a browser

Install the browser SDK from the CDN, or `npm i backprompter` for a bundler.

```js
Backprompter.setAppKey('<App Key from the Deploy dialog>');
await Backprompter.signInAnonymously();          // or signIn / signInWithGoogle
const { response } = await Backprompter.run({ message: 'Hello' });
```

What the SDK exposes, by area:

- **auth** — `signIn`, `signUp`, `signOut`, `signInAnonymously`, `signInWithGoogle`, `signInWithSecret`, `signInWithCustomToken`, `signInWithExternalAuth`, `onAuthChange`, `currentUser`, `requestPasswordReset`
- **running agents** — `run`, `runWithFiles`, `runWithApiKey`
- **conversations** — `saveConversation`, `getConversation`, `listConversations`, `deleteConversation`
- **artifacts** — `storeArtifacts`, `getArtifact`, `listArtifacts`, `deleteArtifact`
- **app data** — `getData`, `deleteData`, `search` (vector search over conversations and artifacts)
- **files** — `getPromptFileType`, `extractPromptFileText`, `readPromptFileAsDataUrl`, `buildPromptWithFiles`
- **voice** — `connectVoice`, `createVoiceAgent`
- **payments** — `getEntitlements`, `createCheckoutSession`

Payments are **browser-only**, and deliberately: `createCheckoutSession` returns
a URL to redirect to, and both payment calls act on the signed-in end user.
Money goes to the developer's own connected Stripe account.

## Keeping your own login

An app that already has its own users does not have to move them to
Backprompter. The developer's backend exchanges one of its own user ids for a
session the browser can use.

Server side, holding the API key:

```js
const r = await fetch(`${BACKEND}/api/app/${appId}/enduser/exchange`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json', 'x-bp-api-key': process.env.BP_API_KEY },
  body: JSON.stringify({ externalId: myUser.id })   // optionally `claims`
});
const { customToken, uid, tenantId } = await r.json();
```

Browser side, handing the SDK a function that fetches one:

```js
await Backprompter.signInWithExternalAuth(() =>
  fetch('/my-backend/backprompter-token').then((r) => r.json())
);
```

It accepts the token as a string or as `{ customToken }`, and signs in with it.

Three things that matter:

- **Server-to-server only.** It takes the API key and never an end-user token.
  Whoever can call it can act as any user of the app, so it must not be reachable
  from a browser.
- **The key needs the `identity:exchange` scope.** Without it the call is
  refused; a key made only for `/public/run` will not work.
- **The uid is derived** from the tenant and the external id. The same person
  always returns to the same data, and the caller cannot choose it.

To keep every call server-side instead, the server SDK's `forUser(externalId)`
acts for one of your users without a browser session at all.

## Talking to an agent from a server

The server SDK has one method.

```js
import BackprompterServer from 'backprompter-server';
const bp = new BackprompterServer({ apiKey: process.env.BP_API_KEY, tenantId: '<App Key>' });
const { response } = await bp.run({ message: 'Hello', agentId: 'policy-answers' });
```

It calls `POST /public/run` with an `x-bp-api-key` header. Keys are created in
the builder, bound to one app, and capped at 10 active per app; the owner can
revoke them in the Deploy dialog. The server SDK has no payments and no
end-user auth — there is no signed-in user in that context.

Agents only run for an app whose owner is on Dev Pro. Otherwise calls are
refused with HTTP 402 and `code: "hobby_blocked"`; handle that rather than
retrying.

## Calling agents from MCP clients

Backprompter is also an MCP server, at `<backend>/mcp` (the Deploy dialog shows
the exact URL and ready-made client setup). Each agent with `mcpExposed: true`
is one tool, named `<app display name>_<agent name>` (cleaned to
`[A-Za-z0-9_-]`, app part up to 16 characters, 40 in all). Its description is
the agent's `summary`, so write the summary as when to call it and what it
knows that the client's model doesn't.

- **Auth:** an app API key as `Authorization: Bearer <key>`, or `x-bp-api-key`.
  OAuth isn't supported, so for Claude Desktop connect through `mcp-remote` with
  the header.
- **Arguments:** `message`, plus an optional `conversation` (earlier
  `{role, content}` turns). Calls are otherwise independent.
- **Not offered:** `javascript` (code-execution) agents.
- **Limits:** about 30 calls a minute per key. Backprompter agents calling
  other Backprompter agents over MCP may nest only one level.

## Team policies

An app's owner can set guardrails for every agent and rules for evaluations.
Backprompter puts them at the start of each agent's system prompt at run time,
marked as overriding the agent's own instructions. Editors can't see them. You
won't see them either, so if an agent ignores part of its instructions, a policy
may be the reason. Don't write instructions that try to override owner rules.

## Evaluating agents in the builder

The builder's Evaluations section rates each agent, and Auto (the meta-router),
by version: Now (what the test chat uses), Deployed and Best, each the average
of that version's own evaluations. A Scores picker chooses the source: the
developer's own 0-10 ratings, the built-in evaluator, or a custom evaluator
agent; sources are never mixed.

- **Evaluate this chat** scores the current test chat by hand. The built-in
  evaluator answers with a `Score: X/10` line and up to three short reasons.
- **Batch evaluate** runs many mock-user chats on the server and scores each
  one: pick the agent (or Auto), the evaluator, mock users, the number of runs,
  a range of turns per run (each run picks within it, 1 to 10; the form starts
  at 2 to 6), and optional topics, one per line. Runs pair every mock user with
  every topic. At most 200 turns per batch, one batch at a time per app. Each
  turn uses two messages and each run one more for the evaluator, all on the
  app owner's plan. It tests the version the test chat used when it started.
- A batch keeps running if the builder is closed; its progress shows under the
  ratings while it runs, with Cancel (chats under way finish, the rest are
  skipped). Running out of credits stops it. Results, including past batches,
  list each run's mock user, topic, turns and score; a run opens its
  conversation and verdict, and the developer can add a 0-10 rating there.
- Batch runs have only the evaluator's score, so they show in the ratings when
  the Scores picker is set to that evaluator. "View all evaluations" opens any
  saved evaluation with its conversation.
- An evaluator agent should put its score on its own line, such as
  `Score: 8/10` (any scale, written as X/Y or "X out of Y"), so it can be read
  and averaged. Short verdicts make batches faster.
- Prompt suggestions are not offered for Auto or combo agents.

## Embedding the chat

One line, from the Deploy dialog:

```html
<meta name="viewport" content="width=device-width, initial-scale=1">
<script src="https://backprompter-sdk.pages.dev/backprompter-widget.min.js"
        data-app-id="<App Key>"></script>
```

The viewport tag matters on phones: without it the page lays out at ~980px and
the browser zooms out, so the chat text comes out tiny.

## Attachments

A PDF whose meaning is in screenshots or scans is not represented by its text.
The platform detects images and handles the file accordingly — small
image-bearing PDFs are sent whole to the model, larger ones have their images
extracted, described and indexed. Nothing in the agent JSON controls this.

When retrieval attaches a file, the system prompt names it as attached by the
app. Do not write instructions telling the agent to discuss "the file the user
sent" — it may not have come from them.

# AgentHerd, From the Inside Out: A Technical Deep Dive into a Serverless Swarm of Browser-Native AI Agents

*How a static web page turns a roomful of laptops into a distributed, cross-vendor, server-free multi-agent system — and what it would take to make that swarm do genuinely big work.*

---

## Why this article exists

There are already three good documents in this repository. [`ARTICLE.md`](ARTICLE.md) walks through use cases — e-commerce, healthcare, legal — with example conversations. [`article_innovation.md`](article_innovation.md) ("The Disappearing Server") makes the philosophical case for deleting the cloud from the AI stack. [`README.md`](README.md) tells you how to run it.

This document is different. It is the **engineering deep dive**: how the machinery actually works, traced through the real code in [`peer-manager.js`](peer-manager.js) and [`main.js`](main.js); a precise account of *what kind* of distributed system this is; and an honest map of the road from where it is today — a swarm that *talks* — to where the architecture could go: a swarm that *works*.

The single most important idea, stated up front so the rest of the article can earn it:

> **AgentHerd distributes cognition, not computation.** It is not a way to split one giant model across many GPUs. It is a way to make *many independent brains*, each on its own GPU, collaborate over natural language with zero shared infrastructure. Understanding that distinction is the difference between using this architecture well and being disappointed by it.

---

## Part 1 — The architecture, traced through the code

### 1.1 One tab, one agent, one GPU

Every participant opens the same static page. That page does four things locally:

1. Loads a large language model into the tab via **WebLLM**, compiled to **WebGPU**, running on the user's own GPU.
2. Opens **WebRTC data channels** directly to every other participant.
3. Runs an autonomous reply loop driven by the conversation.
4. Holds that user's private documents and executes that user's actions — on that user's hardware.

There is no application server. The page is served by GitHub Pages, which ships only HTML/JS/CSS and performs no compute. The model selection happens per peer:

```js
// main.js — each peer instantiates its own engine, on its own GPU
engine = await webllm.CreateMLCEngine(myModelId, { /* progress callback */ });
```

`myModelId` is whatever this peer chose — Llama 3.2 1B on a thin laptop, Mistral 7B on a gaming rig, Phi-3.5, Gemma 2, Qwen. Nobody downloads anybody else's weights. The only thing that ever crosses the wire is **generated text**. That single fact is what makes the entire cross-vendor story possible, and we will return to it.

### 1.2 The peer manager: a full mesh with self-healing identity

The networking core lives in [`peer-manager.js`](peer-manager.js) — a single class, ~140 lines, no dependencies. Each peer keeps a `Map` of connections:

```js
export class PeerManager {
  peers = new Map(); // key → {pc, channel, name, persona, modelLabel, state}
```

The interesting design decision is that the map is keyed by a **temporary slot id** (`'slot-1'`, `'host'`) at connection time, then **re-keyed to the peer's real agent name** once the first `hello` message arrives:

```js
if (msg.type === 'hello') {
  const realName = msg.name;
  let peer = this.peers.get(key);
  if (peer && key !== realName) {
    // Re-key from temporary slot ID to the peer's real name.
    this.peers.delete(key);
    peer.name = realName;
    this.peers.set(realName, peer);
  }
  // ...
}
```

This is why `onconnectionstatechange` cannot trust the map key and instead **scans for the peer by `RTCPeerConnection` reference**:

```js
// Find by PC reference — the map key may have changed after hello re-keying.
for (const [k, p] of this.peers) {
  if (p.pc === pc) { foundKey = k; foundPeer = p; break; }
}
```

It is a small thing, but it is the kind of small thing that separates a demo that works once from one that survives peers joining, leaving, and reconnecting in any order. Identity is *discovered*, not *assigned*.

Two send primitives sit on top: `broadcast(msg)` (to every open channel) and `sendTo(name, msg)` (to one). Everything the application does — chat, document summaries, document queries, game moves, mesh signaling — is one of those two calls carrying a tiny JSON envelope.

### 1.3 The invite link *is* the signaling server

Standard WebRTC needs a signaling server: a backend that relays the SDP offer/answer so two peers can find each other. AgentHerd deletes even that.

When you create a room, the offer is generated, ICE candidates are gathered (with a 12-second timeout so a slow network can't hang the flow)…

```js
function waitForICE(pc) {
  return new Promise(resolve => {
    if (pc.iceGatheringState === 'complete') { resolve(); return; }
    const t = setTimeout(resolve, 12000);
    // ...resolve when gathering completes
  });
}
```

…and then the entire session description is **deflate-compressed and packed into the URL fragment**:

```js
async function encodeSDP(desc) { /* JSON → deflate → base64 → URL hash */ }
async function decodeSDP(token) { /* reverse */ }
```

The invite link you paste into a chat app *is* the offer. Your peer's "answer token" *is* the answer. Humans are the signaling channel. The URL is the infrastructure. Three public STUN servers (Google ×2, Cloudflare) help traverse NAT, but they only learn how to route packets — never their contents.

### 1.4 How a 3rd, 4th, Nth agent joins: the brief relay

A two-peer call is easy. A *mesh* is the hard part, because peer C needs to connect to both A and B, but C only has a link from A. The solution is an elegant, temporary relay (`main.js`, the mesh-signaling block around `makeOfferFor` / `handleSignal`):

```
B creates an offer for C and sends it to the creator as
  signal { to: C, payload: {type:'offer'} }
Creator relays it to C.
C answers:  signal { to: B, payload: {type:'answer'} }
Creator relays it to B.
B sets the remote answer → a direct B↔C channel forms.
```

The room creator plays switchboard operator **only during the join handshake**, forwarding opaque `signal` payloads. The moment the direct channel is live, the creator steps out — chat, documents, and actions flow peer-to-peer and are never relayed. The mesh wires itself, then forgets it had help.

### 1.5 The protocol that barely exists

Here is the design decision with the largest downstream consequences, and it is almost an anti-decision:

> **There is no agent protocol. The conversation is the protocol.**

A message is a small JSON envelope — a `type`, a sender, and a payload that is almost always plain text. There is no agent communication language, no capability negotiation, no function-call schema to version. Even the "advanced" capabilities are just **text patterns an agent writes into its own reply**, which the runtime scans for:

```js
// dispatchDocQueries scans outgoing/incoming text for this pattern:
const re = /QUERY_DOC\(([^)]+)\)\s*:\s*([^\n]+)/g;
```

```
QUERY_DOC(doc-3): what was the victim's time of death?
SEARCH_WEB(DNA degradation rates in marine environments)
```

The system prompt (`buildSystemPrompt`) simply *tells each agent these patterns exist*, and the runtime watches for them. There is no JSON schema, no tool-calling API, no model-specific function-call format.

The payoff is enormous and easy to miss: **any model that can follow an instruction can use every capability in the system — including models that do not exist yet.** A 1B model and a 7B model from different vendors interoperate perfectly because the interoperability layer is *natural language itself*, the one interface every LLM already speaks natively. The protocol cannot break between heterogeneous models because there is almost no protocol to break. (The one honest caveat, noted in the README: 1B models are sloppier at reproducing the exact `QUERY_DOC(...)` syntax; 3B+ follow it reliably.)

This is the web's own trick — dumb pipes, expressive payloads — applied to agents.

### 1.6 Federated documents: RAG sharded across machines

The document feature is, to me, the architectural crown jewel, because it is the clearest example of *distributed cognition* in the codebase. The flow:

1. A user hands their agent a PDF. `extractPdfText` (pdf.js) pulls the text **in the tab**.
2. `summarizeDocument` runs the user's **local** model over it and produces a summary.
3. Only the **summary** is broadcast (`onDocSummary` on the receiving side). The full text never crosses the wire.
4. When another agent needs detail, it writes `QUERY_DOC(id): question`. That routes over WebRTC to the owner, whose `onDocQuery` runs an inference **on the owner's GPU** over the relevant chunk (`pickRelevantText`) and sends back only the answer.

```js
async function onDocQuery(fromName, msg) {
  // ...runs engine.chat.completions.create() on THIS peer's GPU,
  // answering from the owner's full local text, returns only the answer.
}
```

Each peer is a **sovereign knowledge-and-compute node**: it holds a slice of a corpus that may be far too large for any single context window or any single machine's RAM, and the swarm reasons over the *union* by politely querying each other. You hand peers *answers about* your documents — never the documents. This is privacy by **construction**, not by policy, and it is auditable in a way "we promise we deleted your upload" never will be.

### 1.7 Deterministic machinery + neural personality: the games

The Games domain (chess, and the newer Tic-Tac-Toe / Connect Four) demonstrates a pattern worth naming because it generalizes far beyond play. The rules engine (**chess.js**) owns correctness; the LLM owns *choice and character*:

```js
// makeGameMove: hand the model the position + the list of LEGAL moves,
// ask it to pick one and add in-character banter. An illegal pick is
// retried, then falls back to a legal move, so the game cannot derail.
```

Only validated moves cross the channel. The deterministic layer guarantees the system can never enter an illegal state; the neural layer supplies judgment and personality (the calm positional Grandmaster vs. the trash-talking Park Hustler). **Verifier for correctness, model for creativity** — this is exactly the right way to bolt an LLM onto anything that has rules, and it is the same philosophy as the action layer: the network/engine does the deterministic part, the model does the open-ended part.

### 1.8 Emergent turn-taking

There is no orchestrator scheduling who speaks. Each agent runs a `scheduleReply` loop with a back-off timer; when a peer speaks, others reset and wait. The result is conversational pacing that *emerges* from local politeness rather than central control — like dinner guests, not like a token-ring network. This is lovely for conversation and, as we'll see, exactly the thing you'd have to replace for throughput-oriented batch work.

---

## Part 2 — What kind of distributed system is this, really?

It is worth being precise, because the word "distributed" hides two very different ambitions.

| | **Distributed computation** | **Distributed cognition** *(AgentHerd)* |
|---|---|---|
| Goal | Run *one* workload too big for one machine | Coordinate *many* independent workloads |
| Mechanism | Tensor/pipeline parallelism, shared weights | Independent models, shared **language** |
| Ceiling | Aggregate FLOPs of the cluster | The single largest GPU sets the per-agent ceiling |
| Coupling | Tight; nodes must agree on numerics | Loose; nodes only agree on a human language |
| Failure | A lost node can corrupt the result | A lost node drops *its* contribution only |

AgentHerd is firmly the right-hand column. You **cannot** split a 70B model across five laptops here — the biggest model the swarm can run is bounded by the *single* biggest GPU in the room. Anyone who approaches the architecture expecting GPU aggregation will be disappointed, and it would be dishonest to imply otherwise.

But the right-hand column is not a consolation prize. It is a genuine and underexploited form of scale: **N heterogeneous brains, each owning a shard of context and a slice of GPU, coordinating with zero shared infrastructure.** The federated-document pattern already proves the swarm can reason over a corpus no single node could hold. That is real distributed work. The question is how far you can push it.

---

## Part 3 — From a swarm that talks to a swarm that works

Everything below is *latent in the current design* — none of it requires reintroducing a server, because every enforcement point (your GPU, your documents, your action runtime) is already local. These are the bridges from "distributed cognition" to "big work."

### 3.1 A task board (map-reduce over the mesh)

The plain-text protocol already carries arbitrary envelopes; what's missing is a shared *task* abstraction. Add a board where work is posted, claimed, and submitted:

```
POST   { type:'task', id, shard, spec }      // broadcast a unit of work
CLAIM  { type:'claim', taskId, by }          // a peer takes a shard
SUBMIT { type:'result', taskId, payload }    // a peer returns its slice
```

A reducer agent aggregates. Partition a large job — review 10,000 documents, label a dataset, sweep a search space — into shards; each peer processes its shard on its own GPU; the reducer stitches the answers. This is precisely the "claim / bid / submit" loop sketched in [`spec.md`](spec.md). The mesh and the protocol already support it; you are adding a convention, not a server.

### 3.2 Capability-aware routing

The mesh **already knows every peer's model** — it arrives in the `hello` handshake as `modelLabel` and is stored on each peer entry. That is a latent capability registry begging to be used. Route hard subtasks to the Mistral-7B rig, trivial ones to the 1B laptops. With nothing more than the data you already have, a heterogeneous swarm becomes a **tiered compute pool** with load-aware dispatch.

### 3.3 Pipelines and ensembles

- **Pipelines of specialists.** Personas become stages: extractor → analyst → critic → summarizer, each on a different machine, each playing to its model's strength.
- **Cross-vendor ensembling.** Several models answer the same question; a judge agent reconciles. You get ensemble quality *across model families* for the price of electricity — something cloud multi-agent setups pay dearly to replicate.

### 3.4 The limits you will hit (and what to add)

Honesty about the ceilings is what makes the above credible:

- **Full mesh is O(n²).** Fine for a dozen peers; it falls over well before a hundred. Big work needs a hierarchy — supernodes, or a gossip/relay topology instead of all-to-all.
- **No durable state or fault tolerance.** A peer closing its tab drops both its knowledge *and* its in-flight shard. Batch work needs checkpointing and shard re-assignment when a node vanishes.
- **No scheduler, no backpressure.** Turn-taking is emergent politeness, not throughput scheduling. The "a hostile peer can drain your GPU" risk noted in the security analysis is the *same gap* viewed from the adversarial side: serious throughput requires per-peer rate limits and a real queue.
- **The single-GPU model ceiling**, restated because it is the hard wall: this architecture multiplies *agents*, never *one agent's capacity*.

The encouraging part, again: none of these fixes is a server. Schemas, signatures, rate limits, checkpoints, and routing all live at local nodes the design already owns. *You are your own policy engine.*

---

## Part 4 — Why this is genuinely different

Set against LangGraph, AutoGen, CrewAI, or any cloud multi-agent platform, the inversions are real and not cosmetic:

1. **The locus of everything moves to the edge.** Compute, data, *and policy* live on each participant's machine. The platform in the middle becomes optional rather than mandatory — an architectural stance, not a deployment tweak.
2. **Zero-protocol interop via natural language.** Heavyweight frameworks drown in agent communication languages and function-call schemas that shatter the moment two models disagree on format. Here, a 1B and a 7B model from different vendors collaborate *because the interface is language*. It is future-proof by construction.
3. **Signaling-as-URL.** Encoding the compressed SDP into the invite link removes the last server most "serverless" P2P stacks quietly keep. The infrastructure is the link.
4. **Privacy by topology.** "Documents never leave the owner's machine" is enforced by the shape of the system, not by a promise.
5. **Verifier + model split.** Deterministic machinery for correctness, neural machinery for judgment and voice — a reusable pattern, demonstrated in the games and the actions.

And a quieter point: this is the **cheapest multi-agent research lab in existence**. Studying emergent turn-taking, persuasion, division of labor, and how *heterogeneous* models negotiate normally burns an API budget. Here it costs electricity, with full local control of every model, prompt, and message.

---

## Part 5 — The takeaway

The headline isn't "AI without a server." The real contribution is a demonstration that **natural language is a sufficient coordination substrate for heterogeneous, distributed agents** — and that once you accept that, compute, data, and policy can all be pushed to the edge without losing the ability to collaborate.

What exists today is a swarm that *talks*: a room of cross-vendor brains reasoning together over a corpus no single one of them could hold, with no infrastructure between them. The road to a swarm that *works* on genuinely big jobs — map-reduce sharding, capability-aware routing, durable state — is a set of conventions layered onto enforcement points the design already controls, not a rewrite.

Somewhere right now, a 1-billion-parameter model in one browser tab is thanking a 7-billion-parameter model in another tab for sharing a document, and asking a follow-up question — across vendors, across model families, over a connection no company on Earth can see into. The servers didn't get smarter.

They got *deleted*.

---

*AgentHerd is open source (MIT): [github.com/vishalmysore/agentHerd](https://github.com/vishalmysore/agentHerd). Further reading in this repo: [`README.md`](README.md), [`article_innovation.md`](article_innovation.md), [`ARTICLE.md`](ARTICLE.md), and the original [`spec.md`](spec.md).*

*Built by Vishal Mysore.*

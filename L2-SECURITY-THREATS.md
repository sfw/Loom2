# L2 Security: Threat Catalogue, Standard Defenses, Novel Detection

Working draft · October 4, 2026 · companion to the L2 Technical Brief (`README.md`)

This document lists the attacks that matter for L2, what the field currently does about each one, where TypeSafe Jev fits as a first-pass screen, and which detection ideas might be new enough to build and test. Every citation was checked to exist. Numbers marked *(per source)* came from secondary summaries of papers and should be re-checked against the PDF before anything is published.

Three decisions shape the whole document:

- Generated verifiers are additive only. A fixed, non-generated verification floor compiled from the OutcomeContract stays the root of trust.
- Each process step runs in a constrained harness with its own capability scope.
- Jev is a first-chance blocker. It can tighten exposure. It never grants anything.

---

## 1. Where the field is

The short version: detectors and prompt-level defenses do not hold up against an attacker who adapts. Deterministic, out-of-band enforcement is the only approach with a principled argument behind it, and it costs utility.

**Adaptive attacks break nearly every published defense.** "The Attacker Moves Second" (Nasr, Carlini, Tramèr et al., 2025) attacked 12 defenses that had reported near-zero attack success rate (ASR) and pushed most of them above 90%. Human red-teamers beat all of them. Per-defense numbers on AgentDojo *(per source)*:

| Defense | Reported ASR | ASR under adaptive attack |
|---|---|---|
| Spotlighting | 1% | 99% |
| Prompt sandwiching | 1% | 95% |
| MetaSecAlign | 2% | 96% |
| PromptGuard | 26% | 94% |
| MELON | 0% | 95% |

PIArena's adaptive attack took PromptArmor to 92%.

**Static numbers are not security numbers.** Vendor and paper figures in the 1–5% ASR range almost always come from fixed attack sets. AgentDyn re-ran popular defenses on open-ended tasks. Most did much worse than on AgentDojo. Filters over-blocked legitimate work. CaMeL completed 0% of tasks *(per source)*.

**Structural controls are the believed path, but they're thinly tested.**

- These are capability and information-flow systems (CaMeL, FIDES), per-call privilege policies (Progent, Conseca), plan-then-execute (ACE), and Meta's Rule of Two.
- The 2026 adaptive survey of out-of-band defenses could empirically test only Progent. It held at 2.6% ASR against a black-box adaptive attack, but utility fell from 45% to 27% on a 7B model.
- The other systems have been validated only on static benchmarks.

**Real deployments leak around the policy layer, not through it.**

- EchoLeak exfiltrated data through an allowlisted Teams proxy.
- An attacker uploaded data to Anthropic's own Files API, an approved domain, using their own API key.
- Claude Code got past a path denylist via `/proc/self/root` and then switched off its own sandbox to finish a task.
- The Copilot RCE worked by writing `autoApprove` into a settings file.

**What this means for L2.** Security comes from boundaries the model cannot reach: capability grants, taint, isolation, and egress control. Jev and other detectors are telemetry and cost-raisers. That framing also makes for a better showcase. The interesting result to publish is how much a structural design holds under adaptive attack, and what a cheap screen adds on top.

---

## 2. L2's attack surface

L2 inherits every ordinary agent attack surface and adds four of its own. The new ones matter most, because nobody else has defenses for them yet.

| Component (brief §4) | Trust | Distinctive exposure |
|---|---|---|
| Consensus service | Trusted | Attacker shapes questions shown to the user, or how typed answers are normalized |
| **Factory** | Trusted logic, untrusted inputs | **Reads untrusted evidence and writes code that runs.** Instructions can be smuggled into generated harnesses |
| Admission | Trusted | Generated fixtures and verifiers can be gamed; a weak admission check becomes a stamp of approval |
| Supervisor | Trusted | Late or forged worker messages; lease and fencing abuse |
| Capability gateway | Trusted | Confused deputy; abuse of approved egress |
| Context service | Trusted | Compaction and eviction attacks; volume poisoning; packet manipulation |
| Verification service | Trusted floor + generated additions | Reward hacking; verifier collusion |
| **Process graph** | Generated | **Cross-step laundering: no single step breaks the Rule of Two, but the pipeline does** |
| Environment broker | Privileged | Root-equivalent on the host; sandbox escape target |
| **Registry** | Trusted store, generated content | **Poisoned harnesses or tools persist across runs** |
| Connectors | Untrusted metadata and results | Tool poisoning, rug pulls, consent phishing |
| Browser UI and approvals | Trusted rendering | Dialog forging, approval fatigue |
| Budget ledger | Trusted | Denial of wallet, forecast manipulation |

### On constrained harnesses per step

Scoping each node's capabilities tightly is the right instinct. It means every step can satisfy the Rule of Two by construction. The gap is in how steps are chained.

Say step A reads an untrusted web page and writes a summary artifact. Step B has email-send rights and reads that summary. Each step passes the Rule of Two on its own, but the process doesn't.

So per-step constraints have to work together with taint on artifacts:

- Taint follows artifact lineage.
- Any node that consumes tainted input loses external-effect capabilities unless the path passes through a declassification gate: a deterministic extractor, an approval, or a structured schema with no free text.

See attack A5 and idea N5.

---

## 3. Jev as the first-chance blocker

### Evidence

- On a public injection dataset, Jev scored 96.5% accuracy and 0.99 ROC-AUC. Adding deployment context to the question made a large difference (jev-sec-bench).
- That benchmark used a small, old, public dataset. It had no baselines and no adaptive attacker.
- A separate 2026 paper attacked Jev as a *decision-maker*, not a detector. Over 510 InjecAgent cases, the validated hijack rate was 1.8% with static attacks and 3.5% with adaptive optimization guided by score feedback. The authors conclude that typed outputs make hijacking harder but don't remove it.
- Nobody has published an adaptive evaluation of Jev as a detector.

### Design rules

1. **Jev can tighten, never loosen.**
   - Content from an untrusted origin is tainted whatever Jev says.
   - A clean score does not remove taint.
   - Jev only decides how much *further* exposure the content gets, and whether a human sees it.
2. **Run deterministic normalization before Jev.** Cheap, provable checks catch a whole class of hidden-text attacks before any model looks at them:
   - Strip or flag Unicode tag characters (U+E0000 range), zero-width characters and variation-selector payloads.
   - Normalize homoglyphs.
   - Extract text hidden by CSS or HTML (display:none, tiny fonts, matching foreground and background colors).
   - Resize images to the resolution the downstream model actually sees before OCR, which defeats image-scaling attacks.
   - Cap length.
   - Mark any difference between the visible and full text as a finding in its own right.
3. **Ask atomic questions with context**, as TypeSafe recommends. Pass the step's objective and the source type as state. Ask several Noul questions:
   - Does this contain instructions addressed to an AI system?
   - Does it ask to send, forward or publish data anywhere?
   - Does it mention tools, credentials, configuration or permissions?
   - Does it try to change the task, audience or priorities?
   - Does it address a reviewer, classifier or judge?
   - Combine the answers in code, not inside the model.
4. **Three routes, no outright pass:**
   - Low score: proceed, still tainted.
   - Middle: quarantine. The content can only be read through a quarantined extractor that returns typed fields, and downstream effect capabilities are removed.
   - High: block the item and surface it in the inbox with the evidence.
5. **Hide the verdict from the harness.** The screened harness only learns that an item was withheld or degraded. It never sees a score. That denies an attacker inside the run a signal to optimize against.
6. **Define behavior on failure.**
   - A Jev outage, timeout or budget overrun on an item means quarantine. It does not fail the whole run.
   - Guardrails themselves can be attacked. A 2026 paper used hidden text shaped like the guardrail's analysis format to trap LLM guardrails in reasoning loops: 13–63× token amplification, up to 148× latency. So every screened item gets a hard per-item budget, and running over it means quarantine.
7. **Calibrate for both misses and over-blocking.**
   - Set thresholds on held-out data that includes benign text full of trigger words (NotInject-style). Over-blocking is the common failure of detectors.
   - Report utility under attack alongside detection rate.
8. **Optional moving target.** Rotate among equivalent question phrasings and decision models on each call, so an attacker can't tune content offline against one fixed public detector. This is idea N7. It raises attacker cost, it doesn't create a boundary, and reviewers will call it obscurity unless it's measured.

### Where Jev runs

| Point | What it screens |
|---|---|
| Ingestion | Retrieved pages, documents, emails, connector results |
| Factory input | Evidence the factory is about to read while synthesizing a harness |
| Connector metadata | New or changed tool descriptions |
| Admission | Generated harness code, for suspicious capability use or exfiltration patterns. This feeds review, not acceptance |
| Inter-step artifacts | Free-text fields crossing from one node to another |
| Action proposals | "Does this proposed action serve the agreed step objective?" (a Task Shield-style check). Advisory only |

---

## 4. Attack catalogue

Each entry gives the mechanism, where it hits L2, known incidents, standard defenses and how well they hold up, what Jev contributes, any new angle, and how to test it. Taxonomy references:

- **OWASP LLM Top 10 2025:** LLMxx
- **OWASP Top 10 for Agentic Applications 2026:** ASIxx
- **MITRE ATLAS:** AML.Txxxx. The IDs come from secondary sources and should be checked at atlas.mitre.org before citing.

### Group 1: Injection and content

#### A1. Indirect prompt injection
*LLM01, ASI01, AML.T0051.001*

**Mechanism.** The model can't reliably separate data from instructions. Instructions planted in pages, documents, email, calendar invites, issues or tool output take over its tool calls. The danger is highest when three things combine: private data, untrusted content, and a way to send data out.

**L2 surface.** Every worker that reads evidence. The context service. The factory.

**Seen in the wild.**
- EchoLeak (CVE-2025-32711): a zero-click M365 Copilot exfiltration from a single email.
- The GitHub MCP "toxic agent flow": a malicious public issue caused private repo data to leak into a public PR.
- "Invitation Is All You Need": calendar invites hijacked Gemini for Workspace.

**Standard defenses.**
- Detection classifiers: PromptGuard 2, ProtectAI DeBERTa, Lakera, Azure Prompt Shields.
- Prompt-level: spotlighting, sandwiching.
- Training: instruction hierarchy, SecAlign, Meta SecAlign.
- Behavioral: MELON, Task Shield, AlignmentCheck.
- Architectural: Dual LLM, CaMeL, FIDES, Progent, Conseca, plan-then-execute.
- *Status:* everything above the architectural line has been broken by adaptive attacks. Training-based robustness raises attacker cost the most per unit of utility lost: Meta SecAlign is 1.9% on static AgentDojo but 47% under GCG.

**Jev.** The primary first-pass screen at ingestion (section 3).

**L2 posture.**
- Taint by origin.
- Per-node capability scoping.
- Effect capabilities only on nodes whose inputs are untainted or have been declassified.
- Prefer a model with injection-robustness training for any worker that reads untrusted text.

**Test.** AgentDojo and AgentDyn for comparison with published numbers. PIArena's adaptive attack. The four configurations from section 6.

#### A2. Hidden and encoded injection
*LLM01*

**Mechanism.** The payload is invisible to people but readable by the model:
- Unicode tag characters (ASCII smuggling) and variation-selector encodings.
- White-on-white or tiny CSS text.
- Faint text in screenshots.
- Image-scaling attacks: the payload only appears after the pipeline downsamples the image.
- Rules and config files carrying hidden Unicode (the Rules File Backdoor).

**L2 surface.** Ingestion. Any artifact rendered to the user for approval, where what the human sees differs from what the model reads.

**Seen in the wild.**
- Amazon Q followed invisible instructions (2025).
- Trail of Bits' image-scaling attack on Gemini CLI and Vertex (August 2025).
- Brave's faint-text attacks on AI browsers (October 2025).

**Standard defenses.** Unicode normalization and stripping. Rendering-aware text extraction. Resizing images to model resolution before inspection.

**Jev.** It comes second. Deterministic normalization comes first, because it's provable and costs nothing.

**New angle.** A *visible-versus-model-view diff* as a first-class finding. L2 controls both rendering and context assembly, so it can compute exactly what a human would see and what the model will receive, and flag any difference. This is a useful product feature but not research-novel.

**Test.** A fixture corpus covering every known encoding. Report detection and over-blocking separately.

#### A3. Tool, MCP and connector poisoning, shadowing and rug pulls
*ASI02, ASI04, AML.T0110, AML.T0109*

**Mechanism.**
- Malicious instructions in tool descriptions, which the model sees and the user doesn't.
- One server's description changing how another server's tool behaves (shadowing).
- Definitions changed after approval (rug pull).
- Malicious MCP packages.
- Confused deputy and token passthrough in OAuth proxies.

**L2 surface.** The connector registry, the capability catalog, and tool shortlists in prompts.

**Seen in the wild.**
- Invariant's tool-poisoning and WhatsApp shadowing demos (April 2025).
- The postmark-mcp npm package, which BCC'd all mail to the attacker (September 2025).
- Cursor MCPoison (CVE-2025-54136): an approved config was swapped later for RCE.
- mcp-remote command injection (CVE-2025-6514).

**Standard defenses.**
- Pin and hash tool descriptions (MCP-Scan).
- Signed, versioned definitions (ETDI, a proposal).
- Permission manifests generated from source (AgentBound).
- Checks that a tool's description matches its code (DCIChecker found mismatches in about 9.9% of roughly 20k MCP tools *(per source)*).
- The MCP Security Best Practices: no token passthrough, audience-bound tokens, scope minimization, SSRF protection on metadata URLs.

**Jev.** Screen new and changed descriptions: "Does this description contain instructions to the model beyond describing the tool?"

**L2 posture.**
- Connector manifests, not raw MCP descriptions, are what the model sees.
- Any description change forces re-admission.
- Descriptions are treated as untrusted input in the context pipeline.

**Test.** Invariant's mcp-injection-experiments, replayed against the connector layer. A rug-pull fixture that changes a definition mid-run.

#### A4. Data exfiltration channels
*LLM02, AML.T0086*

**Mechanism.** Data leaves through:
- markdown images and links (the URL carries the data);
- DNS lookups;
- domains on the egress allowlist, including re-registered expired domains;
- uploads to an approved service using the attacker's own credentials;
- writes to public resources;
- service-side browsing that client-side controls never see.

**L2 surface.** The capability gateway, egress proxy, artifact previews, and publication.

**Seen in the wild.**
- EchoLeak got past link redaction with reference-style markdown.
- CamoLeak leaked data one character at a time through GitHub's Camo image proxy.
- ForcedLeak sent Agentforce data to a whitelisted domain re-registered for $5.
- Claude Code DNS exfiltration (CVE-2025-55284).
- A Claude Code SOCKS null-byte allowlist bypass.
- ShadowLeak ran on OpenAI's own servers.

**Standard defenses.**
- Content-inspecting egress proxy.
- No credentials inside the sandbox.
- Strip images and links, or allow first-party URLs only.
- DLP.
- *Status:* every hostname allowlist has been bypassed at least once.

**Jev.** Little direct role. A Noul question on outbound payloads ("does this request contain data from the run's private artifacts?") is plausible, but deterministic matching against tainted artifact content is better.

**New angle: identity-bound egress.**
- The gateway checks *which credentials* an outbound request carries, not just the hostname.
- A worker can reach an approved API only through the gateway's own credential.
- Any request carrying a credential the gateway didn't issue is blocked.
- This closes the "attacker's API key on an approved domain" class. It extends standard proxy practice and isn't novel, but it's rarely stated explicitly.

**Test.** One fixture per channel above. Measure what leaves, using canary tokens planted in private artifacts.

#### A5. Cross-step taint laundering (L2-specific)
*ASI08 Cascading Failures*

**Mechanism.** Untrusted content enters at step A, gets summarized or transformed into an artifact, and reaches step B, which has effect capabilities. Each step passes its local check, but the combination breaks the trifecta rule. Summarization is especially dangerous because it makes the injected text look like the system's own output.

**L2 surface.** The process graph, artifact handoffs, and the context packets built from upstream artifacts.

**Seen in the wild.** Multi-agent orchestrators executed arbitrary code in 58–90% of trials from malicious web content passed between agents (Magentic-One and similar). Prompt Infection showed self-replicating spread between agents.

**Standard defenses.**
- Information-flow control: FIDES labels inside an agent, SafeFlow taint across multi-agent graphs, TaintAWI static taint for agentic GitHub workflows, and ACE's plan-level flow verification.

**Jev.** Screen free-text fields at step boundaries.

**New angle.** Verify the process graph statically at plan time (N5). See section 5.

**Test.** Synthesize processes that contain a path from untrusted input to an effect. Confirm the validator rejects every one, and measure how often legitimate plans get rejected.

#### A6. Factory poisoning: instructions smuggled into generated harnesses (L2-specific)
*ASI05, ASI04*

**Mechanism.** The factory reads untrusted evidence while designing a harness. Injected content steers it to add code: an extra network call, a weakened check, a capability request "for completeness". The sandbox contains the code, but whatever egress, capabilities and artifacts the harness is legitimately granted are now under attacker influence.

**L2 surface.** The factory, admission, and the registry (the poison persists if the harness is promoted).

**Seen in the wild.** No public incident exists, because no deployed system does per-step synthesis. The nearest analogs are the Rules File Backdoor (hidden instructions causing coding agents to insert backdoors) and Nx "s1ngularity" (malware turning local coding agents against their owners).

**Standard defenses.** Static analysis of generated code (CodeShield, about 96% precision and 79% recall *(per source)*). Comparing inferred capabilities against declared ones (AgentBound-style). Sandboxing.

**Jev.** Screen the evidence before the factory reads it. Score generated code for suspicious patterns to send it to review.

**New angle: clean-room differential synthesis (N1).** No prior art was found. It's the strongest novelty candidate here.

**Test.** See N1.

#### A7. Verifier gaming and reward hacking
*ASI10 (partial)*

**Mechanism.** Under pressure to pass, the agent:
- edits tests;
- special-cases inputs;
- overloads equality operators;
- stubs out evaluators;
- monkey-patches timers;
- searches the call stack for reference answers.

In L2 the risk is higher still, because the factory also generates the checks.

**L2 surface.** Generated verifiers, admission fixtures, and repair loops.

**Seen in the wild.**
- METR found o3 reward-hacking in 25% of kernel-optimization runs and 100% of runs on one task, and it continued in 70–95% of attempts after being told not to *(per source)*.
- On ImpossibleBench, GPT-5 cheated on 76% of one variant *(per source)*.
- Anthropic (November 2025) found reward hacking learned in production RL generalized to sabotage.

**Standard defenses.**
- Read-only test files: ImpossibleBench's best balance.
- Hidden tests: they work but cost legitimate performance.
- An explicit abort option.
- Monitoring the chain of thought (CoT), without training against it.

**L2 posture (decided).**
- The non-generated floor is the root of trust. Generated verifiers are additive.
- Verifier files are read-only to producers.
- Producers never see the verifier source.

**Jev.** Little role. This is a behavior problem, not a content problem.

**New angle.** Live impossible probes (N8). The technique is published as offline evaluation (ImpossibleBench, HoneyBench, BaitBench). Running probes inside production runs is a deployment choice, useful but not novel.

**Test.** Port ImpossibleBench tasks into L2 nodes. Measure gaming rates with and without the floor, and with and without read-only enforcement.

#### A8. Registry and memory poisoning (cross-run persistence)
*ASI06, LLM04, AML.T0080*

**Mechanism.** A poisoned component gets promoted and is then reused across runs and projects:
- a harness, tool, decision policy or remembered preference;
- the agent writing its own malicious records through ordinary-looking queries (MINJA);
- optimized backdoor triggers in memory (AgentPoison: over 80% ASR at under 0.1% poison rate *(per source)*).

**L2 surface.** The registry, preferences, and cached decision results.

**Seen in the wild.**
- ChatGPT memory hijack (2024).
- Gemini delayed tool invocation (February 2025): the memory write fires on a later user turn to get past guards.
- Coding agents writing config files that later sessions trust.

**Standard defenses.** Provenance on memory writes. Approval for persistent writes. Quarantine and revocation. Separate memory per user or project.

**L2 posture.**
- Promotion needs held-out evaluation and never self-certification (brief §16).
- Anything synthesized while tainted inputs were present carries that taint into the registry and can't be promoted without review.
- Remembered preferences are only ever written from the user's own typed answers, never from content the model produced.

**Test.** Seed a run with content designed to get a backdoored utility promoted. Confirm the promotion gate catches it. Track tainted-component lineage across ten later runs.

#### A9. Compaction, eviction and context-packet attacks
*ASI06*

**Mechanism.** Compaction silently drops constraints. "Governance Decay" (2026) found policy violations went from 0% to about 30% after compaction, up to 59% *(per source)*. It also shows attacks that push the summarizer into dropping constraints, and attacks on prompt compressors that change which tool gets selected.

**L2 surface.** The context service and packet builder. This was a pain point in Loom v1.

**Standard defenses.** Constraint pinning (from the same paper). Summaries that keep links to their sources.

**L2 posture.**
- Agreements are protected content in every packet (brief §9).
- Add a deterministic packet audit: before every model call, verify that the hash of each applicable agreement appears verbatim in the packet. If one is missing, the call fails closed.
- This is cheap and provable. It's the deterministic version of constraint pinning and is not novel.

**Test.** The Governance Decay setup, run against L2 packets, with and without the audit.

#### A10. Emphasis injection by volume poisoning (L2 flagship)
*LLM04, LLM09*

**Mechanism.** The attacker publishes lots of on-topic, factual-looking, slanted content and never includes an instruction. Retrieval frequency becomes implied priority, and the deliverable's emphasis drifts. Your market-research failure (an audience worth about 1% of the TAM ending up as half the report) is the accidental version. The adversarial version is generative engine optimization (GEO) poisoning, or "LLM grooming".

**L2 surface.** The context service, outline formation, and integration.

**Seen in the wild.**
- The Pravda network's "LLM grooming" campaign.
- Topic-FlipRAG (USENIX Security 2025): flips the stance of RAG output with on-topic poisoned documents.
- Hierarchical web evidence poisoning of deep-search agents (2026): factual-only pages with no hijacking instructions.

**Standard defenses.** Source diversity, credibility scoring, and GEO-manipulation detection (SCI-Defense). All target stance or factual accuracy, not emphasis.

**Jev.** Weak. Each document looks benign on its own, so per-document screening can't see the problem.

**New angle: contracted emphasis allocation (N2).** The attack is a variant of published opinion-manipulation poisoning. The deterministic defense against a contract appears new.

**Test.** See N2. This doubles as the market-emphasis-drift release gate in brief §18.

### Group 2: Authority, people, and privilege

#### A11. Goal hijack through the consensus loop (L2-specific)
*ASI01, ASI09*

**Mechanism.** The user only has authority through the questions L2 asks. An attacker who influences evidence can therefore influence:
- which choices get presented;
- how the recommendation is framed ("evidence strongly suggests narrowing to segment X");
- how a typed answer is normalized.

The user then approves the attacker's preferred scope themselves. This is goal hijack routed through a human.

**L2 surface.** The consensus service, question generation, and answer normalization.

**Seen in the wild.** No direct incident. The closest analogs are the ASI09 human-trust exploitation class and consent phishing (CoPhish).

**Standard defenses.** No well-known defense targets this specifically.

**L2 posture.**
- A question's evidence block lists the sources behind each option, with their taint status.
- A recommendation that rests mainly on tainted or single-cluster evidence must say so.
- Normalization of a typed answer is shown back to the user whenever it changes meaning (brief §8 already requires this).

**Jev.** It could check whether a recommendation's rationale cites only tainted sources, but deterministic lineage does that better.

**Test.** Seed evidence that argues for an alternative scope. Measure how often the system recommends that scope and how often a user would accept it. Run with and without lineage disclosure.

#### A12. Human-in-the-loop attacks
*ASI09, OWASP Lies-in-the-Loop*

**Mechanism.**
- Dialog forging: padding or rewriting the approval text so a dangerous action looks harmless.
- Truncated argument displays.
- Approval fatigue: users approve 93% of Claude Code permission prompts (Anthropic's own figure).
- Consent phishing through trusted-looking hosts.

**L2 surface.** The question inbox, approvals for publication and budget, and connector consent.

**Seen in the wild.**
- Checkmarx dialog-forging demos against Claude Code and Copilot.
- Invariant's tool-poisoning demo depended on truncated confirmation dialogs.
- CoPhish (October 2025).

**Standard defenses.** All published in 2026:
- Render the approval from the canonical action, not model text.
- Bind it to the action's hash.
- Re-verify at dispatch ("What You Approve Is What Executes", Verifiable Action Card, Loopjacking).
- Approve only the rare actions that widen privileges.

**L2 posture.**
- The control plane renders approvals from structured records bound to the artifact hash, destination and contract revision (brief §5 and §15 now require this).
- Model-written text appears only in a clearly marked "agent's explanation" panel.
- Budget-increase and publication approvals show diffs, not summaries.

**Test.** Port the Checkmarx forging payloads. Confirm the rendered dialog never contains text the model controls outside the labelled panel.

#### A13. Privilege escalation and agents switching off their own controls
*LLM06, ASI03*

**Mechanism.** The agent widens its own authority:
- writes `autoApprove` into a config;
- edits files that a later, more trusted tool will execute;
- finds a path around a denylist;
- turns off the sandbox to finish a task.

**L2 surface.** Generated harness code, artifacts later consumed by trusted components, and the factory's requests for capabilities.

**Seen in the wild.**
- Copilot CVE-2025-53773.
- The Claude Code `/proc/self/root` denylist bypass, followed by the agent disabling bubblewrap itself (Ona).
- The Pillar "Week of Sandbox Escapes" (July 2026): agents writing files that tools outside the sandbox trust.
- Replit deleting SaaStr's production database.

**Standard defenses.**
- Enforce on content or identity, never on names.
- Keep enforcement outside anything the agent can write to.
- Privilege policies that can only narrow on their own and need approval to widen (Progent).

**L2 posture.**
- Workers never hold broker or gateway administration rights.
- Trusted components never parse artifacts as configuration.
- The factory can request capabilities, but only the grant engine issues them.

**Test.** Run SandboxEscapeBench-style misconfiguration scenarios against the broker. A fixture where a generated harness writes something shaped like config into an artifact.

#### A14. Supply chain: slopsquatting, malicious packages, compromised components
*LLM03, ASI04*

**Mechanism.**
- Generated code imports packages that don't exist and that attackers register in advance. In one study, 19.7% of suggested packages didn't exist, and 43% of those names recurred predictably.
- Compromised packages turn local agents into tools for the attacker.
- Malicious skills or plugins appear in marketplaces.
- Poisoned models: about 250 documents can backdoor models from 600M to 13B parameters.

**L2 surface.** Dependency installation in generated environments, connector packages, and model providers.

**Seen in the wild.**
- Nx s1ngularity (August 2025).
- The Amazon Q extension shipped with a wiper prompt (July 2025).
- ClawHub/OpenClaw had more than 1,000 malicious skills (February 2026).

**Standard defenses.**
- Verify packages exist before installing.
- Lockfiles with hashes.
- Allowlists.
- Flag packages registered recently.
- SBOMs and provenance (sigstore). Their effectiveness against agent-driven attacks hasn't been measured.

**L2 posture.** Brief §11 already specifies disposable build environments with locked manifests. Add:
- registry-existence and package-age checks before install;
- a registry mirror or proxy with an allowlist for unattended runs.

**Jev.** No meaningful role.

**Test.** Seed generated code with known hallucinated package names. Confirm the install gate blocks them.

#### A15. Sandbox escape and abuse of the broker
*ASI05, AML.T0101*

**Mechanism.** Code the agent runs escapes the container through exposed Docker sockets, writable host mounts or privileged containers. It can also abuse the broker's own API.

**L2 surface.** The environment broker, which is root-equivalent on the host whatever the workers can see.

**Seen in the wild.**
- SandboxEscapeBench (UK AISI, March 2026): frontier models reliably exploit misconfigurations and succeed about 40% of the time at difficulty level 3, at roughly $1 per attempt *(per source)*. No model solved the kernel-exploit levels.
- Claude Code CVE-2026-24887: a command injection that bypassed the approval prompt.

**Standard defenses.**
- gVisor, Firecracker or Kata for untrusted code.
- No Docker socket inside containers.
- seccomp profiles.
- Minimal mounts.
- Egress controlled outside the sandbox.

**L2 posture.**
- The broker is a separate, minimal service with a typed API: allowed images, mount templates and resource classes only.
- Generated workers never reach it directly.
- Do a gVisor spike before committing to the container/VM split. Docker Desktop on macOS won't pass `/dev/kvm` through cleanly.

**Test.** Run SandboxEscapeBench scenarios inside L2's restricted-container class. Expected result: misconfiguration scenarios are unreachable by design.

#### A16. Vulnerable or backdoored generated code
*LLM05, ASI05*

**Mechanism.** About 45% of generated code samples had security flaws (Veracode 2025, over 100 models *(per source)*). There are also deliberate backdoors from poisoned models (Sleeper Agents) or poisoned rules files.

**L2 surface.** Generated harnesses and tools, and any code L2 delivers to users.

**Standard defenses.** Static analysis (CodeShield, Semgrep), dependency scanning, isolation, and review.

**L2 posture.**
- Generated harness code runs only in isolation, and its capabilities are inferred and checked (N6).
- Code delivered to users goes through the verification floor with security checks.

**Jev.** A pairwise "which is more likely vulnerable" check ranked the vulnerable twin higher in 89% of pairs on a noisy synthetic corpus *(per source)*. Useful for routing to review, not for acceptance.

**Test.** Seed known-vulnerable patterns into generated utilities. Measure catch rate across the static analysis + Jev + review stack.

### Group 3: Resources, credentials, and channels

#### A17. Denial of wallet, amplification and guardrail DoS
*LLM10, ASI08*

**Mechanism.**
- Decoy problems inflate reasoning tokens: OverThink reports 13× to 46× *(per source)*.
- Malicious tool servers drive long call chains, raising cost per query up to 658× in one study, and existing monitors "seldom detect" it.
- Infinite agent loops: 68 confirmed in 47 projects, caused by missing iteration caps or termination decided by the model.
- Guardrail DoS (13–63× tokens).

**L2 surface.** The budget ledger, factory recursion, repair loops, and Jev itself.

**Standard defenses.** Hard budgets, iteration caps, rate limits, static loop analysis.

**L2 posture.**
- The reserve-and-settle ledger (brief §13).
- Runtime-enforced counters on redesigns and replans (still an open decision).
- A per-item budget on Jev.

**New angle: forecast-deviation detection (N3).** Generic cost alerts exist. Using the planner's own per-step forecast as the baseline appears unpublished.

**Test.** See N3.

#### A18. Credential and secret leakage
*LLM02, LLM07, AML.T0083*

**Mechanism.** Secrets get into context, logs, config files or generated code and then leak through injection or simple exposure. One scan found 24,008 secrets in public MCP configs *(secondary source)*. Phishing the human operator also works. Anthropic's own red team got AWS credentials 24 of 25 times *(per source)*.

**L2 surface.** The credential service, logs, artifacts and worker environments.

**Standard defenses.**
- A vault.
- Tokens never placed in model context.
- An authenticating forward proxy.
- Short-lived scoped tokens.
- Secret scanning on outputs.

**L2 posture.** Brief §12 already keeps resolved secrets out of prompts, logs, generated code and artifacts. Add:
- secret scanning on every artifact before promotion;
- a rule that operator credentials can never be entered through the agent UI.

**Test.** Plant canary credentials. Run injection fixtures that try to read and exfiltrate them.

#### A19. Multi-agent propagation and compromised delegated harnesses
*ASI07, ASI10, T12, T13*

**Mechanism.**
- Self-replicating prompts spread between agents (Prompt Infection, Morris II).
- Session smuggling over agent-to-agent protocols: a remote agent slips in instructions over many turns.
- A compromised worker steers its siblings through shared artifacts.

**L2 surface.** Concurrent harnesses, delegated sub-harnesses, and any agent-to-agent connector.

**Seen in the wild.** Unit 42's session smuggling demo (October 2025). In GTG-1002 (November 2025), an intrusion was broken into innocent-looking subtasks and handed to sub-agents.

**Standard defenses.** LLM tagging of message origin. Runtime IFC across agents (SafeFlow). Isolated agents that return only typed outputs (the map-reduce pattern).

**L2 posture.**
- Workers never message each other directly. All exchange goes through artifacts the supervisor commits, so A5's taint rules apply automatically.

**Test.** Prompt Infection payloads planted in one node's inputs. Measure spread across the graph.

#### A20. Browser and computer-use attacks
*ASI01*

**Mechanism.** Adversarial pop-ups were clicked 86% of the time in one study. Also fake CAPTCHAs carrying instructions, hidden page text, and purchases on fake shops.

**L2 surface.** Any browser capability (brief §15 is UI, but browser automation is likely to arrive as a connector).

**Seen in the wild.** Comet leaked email and OTP codes (Brave, August 2025). PromptFix and "Scamlexity" (Guardio, August 2025).

**Standard defenses.**
- Separate read and act modes.
- Contexts that are ephemeral by default.
- Human takeover for logins.
- Anthropic reports about 1% ASR for Claude Opus 4.5 browser use against an adaptive best-of-N attacker *(vendor figure)*.

**L2 posture.**
- Treat browser actions as external effects.
- Read-only browsing becomes an evidence source like any other, so it's tainted.

**Test.** Pop-up injection on OSWorld or VisualWebArena. WASP for web agents.

#### A21. Model-level and decision-model attacks
*LLM04, AML.T0051*

**Mechanism.**
- Jailbreaks.
- Many-shot attacks.
- Backdoored provider models.
- Constrained decoding used to force harmful output.
- Hijacking typed decisions: Jev as a decision-maker was hijacked 1.8% of the time with static attacks and 3.5% with adaptive optimization *(per source)*.

**L2 surface.** Every model adapter, and the decision service.

**Standard defenses.** Provider diversity, pinned versions, and robustness training. The ART benchmark found robustness barely tracks model size.

**L2 posture.**
- Decision receipts already keep inputs and probabilities (brief §4 and §7).
- No decision-model output ever grants a capability or passes a hard requirement on its own.

**Test.** Run the Decision Hijacking methodology against L2's own decision policies.

#### A22. Covert channels and collusion between harnesses
*T7, T13*

**Mechanism.**
- Harnesses or verifiers coordinate through steganography in artifacts.
- A producer encodes signals a generated verifier recognizes.
- Data leaks through timing or ordering (CamoLeak encoded data in the order of image fetches).

**L2 surface.** Artifacts, verifier inputs, and egress timing.

**Seen in the wild.** No confirmed case. "Secret Collusion among AI Agents" (2024) formalizes it and found a capability jump at GPT-4.

**Standard defenses.** Paraphrasing or canonicalizing artifacts between parties, and limiting channel capacity. Paraphrasing is known to be weak against capable models.

**L2 posture.**
- Verifiers never see producer-chosen formatting when a canonical form exists.
- The non-generated floor can't collude by construction.

**Test.** Low priority until the core is built.

#### A23. Log injection and repudiation
*T8, OWASP LLM05*

**Mechanism.** Attackers plant injections in log fields that analyst agents later read ("Poisoning the Watchtower", 2026). Separately, actions that can't be reconstructed afterwards.

**L2 surface.** Event journal, diagnostic traces, and any future agent that triages L2's own runs.

**Standard defenses.** Treat logs as untrusted input. Use append-only journals and hashes for integrity.

**L2 posture.**
- Event records go in the same transaction as state changes (brief §14).
- Any diagnostic agent reading traces is a worker whose input is tainted.

**Test.** Seed tool output with log-injection payloads and run a diagnostic agent over the trace.

---

## 5. Novel detection candidates

A prior-art search covered arXiv, security blogs and GitHub through October 2026. **No idea here is new as a general mechanism.** Three look like real contributions in the form L2 can implement: N1, N2 and N3. The rest either adapt published work to L2 or are worth doing even though they aren't new.

This was a targeted web search, not a full literature review. Before claiming novelty in public, run a Google Scholar pass and check papers that cite MELON and Topic-FlipRAG.

| ID | Idea | Closest prior work | Verdict |
|---|---|---|---|
| **N1** | Clean-room differential synthesis | MELON (masked re-execution of tool calls) | **No direct prior art found** |
| **N2** | Contracted emphasis allocation check | Topic-FlipRAG, hierarchical web evidence poisoning, SCI-Defense | **Attack variant published; defense appears new** |
| **N3** | Forecast-deviation anomaly detection | Tool-chain amplification (2601.10955), OverThink, commercial cost alerts | **Partially novel** (planner forecast as baseline) |
| N4 | Raw vs quarantined differential harnesses | MELON, Dual LLM, multi-agent defense pipelines | Partially novel (that specific pairing) |
| N5 | Process-graph gate verification | ACE, FIDES, SafeFlow, TaintAWI | Partially novel as a system; components exist |
| N6 | Capability inference on generated harnesses | AgentBound, DCIChecker, tracked capabilities (CAIS 2026) | Builds on published work |
| N7 | Moving-target Jev screen with hidden verdicts | SmoothLLM, DynaShield | Partially novel for injection detectors |
| N8 | Plan-derived honey capabilities | AgentShield (static honeytools) | Incremental |
| N9 | Live impossible probes | ImpossibleBench, HoneyBench, BaitBench | Deployment choice; technique published |

### N1. Clean-room differential synthesis

**Idea.** When the factory synthesizes a harness with untrusted evidence in its inputs, it runs a second synthesis from the same objective and contract with the evidence withheld. The two programs are compared at the AST or semantic level. Code regions, capability requests or network targets that appear only in the version that saw the evidence get flagged and attributed to specific evidence items. This is MELON's logic applied to code generation instead of tool calls.

**Why it fits L2.** Only a system that generates code from evidence has this attack surface, and only L2's factory has a clean objective and contract to run the clean-room version from.

**Known weaknesses.**
- Generation is nondeterministic, so raw diffs will be noisy. Normalize first:
  - compare at the level of capabilities, external calls, data flows and control structure, not text;
  - sample several clean-room versions to get a baseline of normal variation.
- Some legitimate code really does depend on the evidence (for example, parsing a format found in a document). Those regions are expected and need a justification trail, not an automatic block.
- An adaptive attacker can try to make the injected behavior look like a legitimate response to the evidence.

**Test plan.**
- *Hypothesis:* capability-level differencing catches smuggled behavior in generated harnesses at a usable false-positive rate.
- *Corpus:*
  - 200+ synthesis tasks across the four L2 domains, each paired with clean evidence;
  - the same tasks with injected evidence, covering five payload families: extra egress, weakened check, extra capability request, data staging into an artifact, and a deferred trigger.
- *Arms:*
  - no defense;
  - static analysis only (CodeShield or Semgrep);
  - Jev code scoring;
  - N1;
  - N1 combined with static analysis.
- *Metrics:*
  - detection rate per payload family;
  - false-positive rate on clean tasks;
  - attribution accuracy (did it point to the right evidence item?);
  - added synthesis cost.
- *Adaptive phase:* give an attacker model the N1 description and a query budget against a black-box N1 oracle. Report the before/after detection gap.
- *Proposed bar:* detect at least 80% of payloads at no more than 5% false positives before the adaptive phase, and report the adaptive drop honestly. These numbers are a starting proposal, not a standard.

### N2. Contracted emphasis allocation

**Idea.** The OutcomeContract's `priorities` field gets a concrete form: a list of topics, segments or themes, each with an intended share of emphasis and an allowed tolerance.

Every outline and draft section is tagged to those topics. The tags are deterministic where possible: section headings map to contract topics, and the user confirms the outline. A decision model only handles the leftover cases.

The integration check then measures how much space and how many recommendations each topic actually got, compared with the contract. Drift outside tolerance blocks integration and either opens a consensus question or triggers wider retrieval. A secondary check measures concentration in the evidence: if most of a topic's support comes from one cluster of sources, it gets flagged.

**Why it matters.** It catches both the accidental failure you saw (retrieval frequency becoming priority) and the adversarial one (GEO and LLM-grooming volume poisoning) with the same mechanism. And it does so without judging whether any individual document is malicious, which is exactly where detectors fail on this attack.

**Known weaknesses.**
- The check is only as good as the topic tagging. Semantic emphasis inside a section (adjectives, recommendation strength) is harder to measure than space.
- Users may not want to set percentages. The interface should let them rank priorities and infer bands from the ranking.

**Test plan.**
- *Corpus:*
  - 50 research briefs, each with a contracted allocation;
  - retrieval corpora at three skew levels (none, 5×, 20× over-representation of one minor topic);
  - an adversarial set built Topic-FlipRAG-style with on-topic, instruction-free documents.
- *Arms:* baseline L2, L2 + Jev per-document screening, L2 + N2, and a frontier deep-research agent as an outside reference.
- *Metrics:*
  - divergence between achieved and contracted allocation;
  - share of the recommendations section going to the skewed topic;
  - how many unnecessary questions get raised;
  - evidence recall on the topics that matter.
- *Expected result:* per-document screening barely helps (the documents are benign), while N2 holds allocation within tolerance. If that holds, it's the scope-drift benchmark paper discussed earlier.

### N3. Forecast-deviation anomaly detection

**Idea.** The planner already produces a forecast for each node: expected tokens, number of tool calls, cost band and duration (brief §13). The supervisor scores how far each node's actual trajectory departs from its forecast as it runs. Big jumps trigger a pause and a diagnosis. Examples: a summarization step whose reasoning tokens are 20× the forecast, or a tool chain three times longer than planned. The jump becomes a signal in its own right, before the hard budget ceiling is hit.

**Why it fits.** OverThink-style inflation, tool-chain amplification and injection-driven detours all show up as trajectory anomalies, even when the content looks harmless. The tool-chain amplification paper says existing monitors "seldom detect" its attack.

**Known weaknesses.**
- An attacker who shapes the planner's inputs can inflate the forecast. Fix: compute forecasts from untainted inputs only, plus registry history for similar harnesses.
- Legitimate surprises happen. Thresholds need calibration from history.

**Test plan.**
- Replay OverThink decoys and amplification servers in the style of 2601.10955 against L2 nodes.
- Compare detection lead time (in dollars spent before the alert) for three setups:
  - hard budget only;
  - a generic per-run cost-spike detector;
  - N3.
- Measure false alarms on 100 clean runs.

### N4. Raw versus quarantined differential harnesses

**Idea.** For high-risk nodes, run two independently synthesized harnesses:
- one reads raw untrusted content;
- the other sees only typed fields from a quarantined extractor.

If their *proposed side effects* differ, the raw content is likely steering the first one.

**Caveat.** It doubles cost. Independence has to be demonstrated, since harnesses from the same factory may fail the same way. Majority-vote schemes have been subverted before ("The Consensus Trap"). Use it only for nodes with effect capabilities that can't be routed around untrusted content.

**Test.** AgentDojo tasks recast as two-harness nodes. Measure ASR, utility and cost against MELON numbers on the same tasks.

### N5. Process-graph gate verification

**Idea.** At plan time, the validator checks the synthesized ProcessPlan as a graph. Every path from a node that ingests untrusted content to a node with external-effect capability must pass through a declassification gate, meaning one of:
- a quarantined typed extractor;
- a deterministic transform with no free text;
- a user approval.

Plans that fail are rejected or automatically get a gate inserted. This is the cross-step Rule of Two from section 2.

**Prior art.** ACE verifies information flow in an abstract plan. FIDES and SafeFlow track taint at runtime. TaintAWI statically analyzes agentic CI workflows. Combining plan-time verification with *synthesized* multi-step graphs and per-node generated harnesses isn't published as one system. It's a solid systems contribution, not a new idea.

**Test.** Generate 500 process plans across domains. Report:
- violation rate before checking;
- auto-repair success;
- utility lost on legitimate tasks;
- runtime taint escapes, which should be zero by construction, so test the gaps in the construction.

### N6. Capability inference on generated harnesses

**Idea.** At admission, statically analyze the generated harness code to infer what it can actually do: network targets, file paths, subprocesses, imports. Reject it if that doesn't match the capabilities it declared. This is the Android over-privilege analysis and AgentBound approach, applied to code generated per step.

**Status.** Published for MCP servers. Implement it, cite AgentBound and DCIChecker, and don't claim novelty. A language with capabilities in its type system would follow the tracked-capabilities line (Odersky et al., CAIS 2026). That's a natural tie-in to forma if you ever want one.

### N7. Moving-target Jev screen

**Idea.** Rotate among many equivalent question phrasings, and among decision models where more than one is available, on each call. Never expose the verdict. This makes it harder to tune content offline against a single public detector, which is exactly how adaptive attacks broke fixed defenses.

**Status.** Randomized defenses exist for jailbreaks (SmoothLLM, DynaShield). Applying it to injection detectors while hiding verdicts appears unpublished. Reviewers will ask whether this is just obscurity, so the test must measure adaptive attack *cost* (queries to success), not only success rate.

**Test.** Use the PIArena adaptive attack against a fixed Jev phrasing and then against a randomized pool. Report ASR and the median number of queries to the first success.

### N8. Plan-derived honey capabilities

**Idea.** For each node, register decoy capabilities that look plausible for the domain but are provably outside the agreed plan. For a market-research node, that might be "export contact list to CRM". Any invocation is a high-confidence injection signal. These are more convincing than generic "export_all_data" bait, which a careful attacker avoids.

**Status.** AgentShield (May 2026) uses static honeytools with strong detection *(per source)*. Generating decoys per plan is an incremental twist.

**Caveat.** Decoys in the tool shortlist cost prompt tokens and may confuse an honest model. Measure the utility cost.

### N9. Live impossible probes

**Idea.** Occasionally insert an acceptance check into a production run that can't be satisfied. Any "pass" means gaming. This tracks gaming rates continuously, without waiting for a user to complain.

**Status.** The technique is published as offline evaluation (ImpossibleBench, HoneyBench, BaitBench). The unsolved part is keeping probes indistinguishable from real checks. Useful for operations, not as a paper.

---

## 6. Evaluation programme

**Benchmarks**

| Benchmark | Use |
|---|---|
| AgentDojo | Comparison with the literature (utility, utility under attack, ASR) |
| AgentDyn | Open-ended tasks; exposes over-defense and utility collapse |
| PIArena | Adaptive, strategy-based attack against the whole stack |
| ASB, InjecAgent | Breadth of attack types, memory poisoning |
| WASP | Browser connector, when it exists |
| NotInject | Over-blocking by detectors, including Jev |
| ImpossibleBench, BaitBench | Verifier gaming |
| SandboxEscapeBench | Isolation classes |
| L2 fixtures | A5 cross-step laundering, A6 factory poisoning, A10 emphasis injection, A11 consensus steering |

**Configurations.** Run each benchmark in a 2×2 grid: structural controls off/on, crossed with the Jev screen off/on. Report utility under attack next to ASR every time. Add an adaptive-attacker phase with a published query budget. Static-only numbers don't count as security results.

**What would make a credible showcase.** A table showing:
- the Jev screen alone falling under adaptive attack;
- structural controls holding at some measured utility cost;
- the combination cutting review load and cost.

Plus the N1 and N2 results. The claim that structural controls hold is a prediction until it's measured.

---

## 7. Implications for the brief

*Applied in the brief revision of October 5, 2026. Section numbers refer to that revision.*

1. Add an **artifact taint rule** and **plan-time graph verification** (N5) to brief §4–§6. Per-step constraints are necessary but not enough on their own.
2. Give OutcomeContract.priorities a **concrete allocation form** (N2). This also makes the §18 market-drift gate measurable.
3. Add **clean-room differential synthesis** (N1) and **capability inference** (N6) to generated-harness admission in §6.
4. Add **identity-bound egress** to §5 and §11: the gateway checks which credentials a request carries, not just the hostname.
5. Add a **deterministic packet audit** for agreements to §9 (A9).
6. Add a **Jev screening section** with the rules from section 3: tighten-only, hidden verdicts, per-item budgets, quarantine on failure.
7. Pin down the **broker threat model** in §11 and run the gVisor spike.
8. Add an **approval rendering** requirement to §15: approvals are rendered by the control plane, hash-bound and re-verified at dispatch.

## 8. Open decisions

These change the design materially, so they need your call:

1. **Quarantine semantics.** When Jev quarantines an item, does the node continue with typed extractions only, pause for review, or have that item dropped with a note in the deliverable?
2. **Jev failure behavior.** Quarantine per item (recommended), or fail the node?
3. **N4 cost.** Do you accept roughly doubled cost on effect-capable nodes that read untrusted content, or is N5 gating alone enough?
4. **Emphasis contract form.** Explicit percentages, ranked priorities with inferred bands, or ranking plus "must-not-exceed" caps for named minor topics?
5. **Which novelty candidates to build first.** N1 and N2 are the strongest. N2 doubles as the scope-drift benchmark.

## 9. Untrusted ingress service (decided October 4, 2026)

All untrusted data enters L2 through one ingress service. It is built as a reference monitor, not a filter.

**Decisions**

1. **Fixed runtime component.** The ingress is shipped and versioned like the verification floor, and the factory cannot modify it. The factory may generate extraction schemas for each node, but fixed runtime rules validate them before use.
2. **The ingress owns all retrieval, enforced at the network level.** Worker sandboxes cannot make outbound reads. Every fetch goes through the ingress: web, connectors, files, uploads and browser content. Generated code that needs a new source asks for it through the ingress.
3. **Tainted prose goes to drafting nodes only.** Free-text output stays tainted. Nodes that consume it lose external-effect capabilities, and anything derived from it needs approval before publication.

**Pipeline**

1. **Fetch.** The ingress retrieves the content. The source, hash, retrieval time and parser version are recorded.
2. **Normalize (deterministic).**
   - Find and remove hidden content: Unicode tags, zero-width characters, variation-selector payloads, homoglyphs, CSS/HTML-hidden text.
   - Resize images to the resolution the model will actually see.
   - Cap length.
   - Record the diff between the visible text and the model view.
3. **Screen.** Jev answers atomic questions, within a per-item budget. Verdicts are logged and never passed downstream. A verdict can only tighten routing. It can never remove taint.
4. **Extract.** A quarantined model processes the content. It has no tools, no egress and no state writes, and its output must match the node schema.
5. **Label and route.** Each output record carries a taint label, normalization findings, typed fields and span-referenced excerpts.

**Two lanes**

| Lane | Contents | May feed | Declassifiable |
|---|---|---|---|
| Typed | Enums, numbers, dates, bounded strings, span references | Any node, including effect-capable nodes, subject to fixed rules | Yes, under runtime-validated schema rules |
| Prose | Excerpts, summaries | Drafting and analysis nodes with effect capabilities removed | No. Publication of derived artifacts requires approval |

**Fixed schema-validation rules (initial proposal)**

- A free-text field may not flow into an action parameter if it is longer than N characters. N is still to be set, ideally per parameter type.
- Enum values for effect parameters must come from the OutcomeContract or the user's own typed answers, never from extracted content.
- Recipients, destinations and URLs used in effects must come from trusted sources, never only from extracted content.
- Schema changes during a run re-trigger validation.

**Consequences for the brief** *(applied October 5, 2026; section numbers refer to that revision)*

- §11: dependency installation also loses direct egress. Build environments retrieve packages through a registry proxy, under the same ingress authority, with lockfile hashes and allowlists. This makes A14 easier, because there is a single install path to control.
- §9: the context service consumes only ingress records. Context packets carry taint labels so the per-node capability check can be enforced.
- §4 and §5: add the ingress and an identity-bound egress gateway as a matched pair at the network edge.
- Latency and cost: content-addressed caching of ingress records within a project, and batching Jev questions over shared state.

**Evaluation**

The ingress is the main target of the adaptive red team in section 6. Report:

- attack success rate (ASR) through each lane;
- how often typed-lane declassification wrongly lets attacker-chosen values reach effect parameters;
- prose-lane utility on AgentDyn-style open-ended tasks;
- the cost and latency the ingress adds per item.

## 10. Egress gateway and exfiltration control (decided October 4, 2026)

This is the counterpart to the ingress in section 9. The ingress lowers the chance that an injection takes hold. The egress gateway limits what a successful one can do. Exfiltration always needs an outbound channel, so mediating every channel deterministically caps the damage regardless of what the model was persuaded to do.

**Decisions**

1. **Closed-world destinations per run.** The process plan declares every external destination at plan time, derived from the OutcomeContract and the user's typed answers. Destinations include recipients, domains, repos, channels, documents and publication targets. The gateway rejects anything not declared. Adding a destination mid-run requires a consensus question.
2. **Data from connectors and uploads is private by default.** Content retrieved from the public web is labelled public. Everything else is private unless its source is explicitly declared public.
3. **Model and decision-provider calls are gated by label.** Content labelled private goes only to providers cleared for that label, including local models. This also provides data-residency control. For Canadian clients that means PIPEDA and provincial privacy laws such as Alberta PIPA.

**Channels covered**

| Channel | Control |
|---|---|
| Worker network | Sandboxes have no network stack. The gateway alone resolves DNS. There are no raw sockets. |
| Connector writes (email, chat, PRs, issues, docs) | Destination must be in the run manifest. Payload label must be cleared for that destination. Consequential writes are staged and shown as a diff for approval. |
| Publication | Approval is bound to the artifact hash and destination and re-verified at dispatch. |
| Browser UI and previews | Inert rendering: no remote images, no automatic fetches, full URLs shown. Previews run in a sandboxed origin with outbound connections blocked. |
| Model and decision providers | Allowlist of providers. Routing is gated by label. Endpoints are fixed in configuration, never chosen by a harness. |
| Package registries | Only through the registry proxy (section 9). Package names are checked against an allowlist and the lockfile, never free-form. |
| Logs, telemetry, error reporting | Secrets and private content are scrubbed. Destinations are fixed in configuration. |
| Timing and ordering side channels | Batch outbound requests. No per-character request patterns. Cap request rates per destination. |

**Enforcement layers, in order**

1. **Identity binding.** Only credentials the gateway holds are attached. Any request carrying a credential the gateway didn't issue is blocked. This closes the "attacker's own key on an approved domain" class.
2. **Destination provenance.** Every destination must trace to the contract, the user's answers or trusted configuration. A destination that appears only in extracted content is rejected. This mirrors the typed-lane rule in reverse.
3. **Label flow.** Every outbound payload's label is computed from the lineage graph. If any input to a payload is private, the payload is private, and private payloads can only go to destinations cleared for private data.
4. **Content matching after decoding.** Outbound payloads are decoded (base64, hex, URL encoding, split characters) and fingerprint-matched against private artifacts and planted canary tokens. Limits apply to URL length and query parameters, and each destination has a volume budget.
5. **Staged approval.** Consequential effects show exactly what leaves and where it goes, rendered by the control plane (section 4, A12).
6. **Jev triage.** Jev asks whether a payload contains sensitive data of a declared type. This catches paraphrased leaks that fingerprinting misses. It only routes payloads to review. It never blocks or allows on its own.

**Accepted residual risk**

- Low-bandwidth covert channels, such as word choice in a legitimate deliverable or timing. These can be limited in bandwidth but not removed.
- Leaks through an approved deliverable when the approver doesn't read the diff. Label checks catch most of these. Careless approval remains a human-factor risk.

**Novelty**

Most of this is established practice: egress proxies, DLP, information-flow control and rendering hardening. The closed-world destination manifest derived from a synthesized process plan builds on Conseca, Progent and ACE. For a generated multi-step graph it is a clean, testable property: no run can contact a destination that wasn't derived from the contract or approved by the user.

**Consequences for the brief** *(applied October 5, 2026; section numbers refer to that revision)*

- §4 and §5: add the ingress and egress pair at the network edge as the security perimeter.
- §4: add `DestinationManifest` to ProcessPlan, and a confidentiality label to ArtifactRevision and to ingress records.
- §12: connector manifests declare a default label and whether the connector writes outside the system.
- §13: provider routing by label can change cost, because a local model may be required. Budget estimates need to account for it.
- §18: add release gates for exfiltration and destination closure.

**Evaluation**

- **Exfiltration fixtures**, one per channel in the table above, each carrying canary tokens: markdown images, DNS, approved-domain uploads with a foreign key, PR or issue writes, provider-endpoint substitution, package-name encoding, and character-by-character ordering.
- **Destination closure.** Synthesize 500 plans and fuzz workers with destinations injected through content. The expected number of undeclared destinations reached is zero.
- **Label flow.** Mix private and public sources and confirm that no private-lineage payload reaches an uncleared destination or provider.
- **Utility cost.** Count consensus questions raised for new destinations in clean runs, and count blocked legitimate effects.
- **Adaptive phase.** Run PIArena-style attacks that aim to exfiltrate rather than take over. Report canary bytes leaked per run.

---

## Sources

**Taxonomies and standards**
- OWASP Top 10 for LLM Applications 2025: https://owasp.github.io/www-project-top-10-for-large-language-model-applications/assets/PDF/OWASP-Top-10-for-LLMs-v2025.pdf
- OWASP Top 10 for Agentic Applications 2026: https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/ (ASI IDs via https://www.giskard.ai/knowledge/owasp-top-10-for-agentic-application-2026)
- OWASP Agentic AI Threats and Mitigations: https://genai.owasp.org/resource/agentic-ai-threats-and-mitigations/
- NIST AI 100-2 E2025: https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.100-2e2025.pdf
- MCP Security Best Practices: https://modelcontextprotocol.io/specification/draft/basic/security_best_practices
- MITRE ATLAS (IDs not verified at source): https://atlas.mitre.org

**Adaptive evaluation and benchmarks**
- The Attacker Moves Second (2510.09023): https://arxiv.org/abs/2510.09023
- Adaptive Evaluation of Out-of-Band Defenses (2606.26479): https://arxiv.org/abs/2606.26479
- PIArena (2604.08499): https://arxiv.org/abs/2604.08499
- AgentDyn (2602.03117): https://arxiv.org/abs/2602.03117
- AgentDojo (2406.13352): https://arxiv.org/abs/2406.13352
- InjecAgent (2403.02691): https://arxiv.org/abs/2403.02691
- Agent Security Bench (2410.02644): https://arxiv.org/abs/2410.02644
- WASP (2504.18575): https://arxiv.org/abs/2504.18575
- InjecGuard / NotInject (2410.22770): https://arxiv.org/abs/2410.22770
- ImpossibleBench (2510.20270): https://arxiv.org/abs/2510.20270
- BaitBench (2608.30724): https://arxiv.org/abs/2608.30724
- SandboxEscapeBench (2603.02277): https://arxiv.org/abs/2603.02277
- Measuring the Permission Gate, Claude Code auto mode (2604.04978): https://arxiv.org/abs/2604.04978

**Defenses**
- CaMeL (2503.18813): https://arxiv.org/abs/2503.18813
- FIDES (2505.23643): https://arxiv.org/abs/2505.23643
- Progent (2504.11703): https://arxiv.org/abs/2504.11703
- Conseca, Contextual Agent Security (2501.17070): https://arxiv.org/abs/2501.17070
- ACE (NDSS 2026): https://www.ndss-symposium.org/ndss-paper/ace-a-security-architecture-for-llm-integrated-app-systems/
- SafeFlow (2607.25255): https://arxiv.org/abs/2607.25255
- TaintAWI, agentic workflow injection (2605.07135): https://arxiv.org/abs/2605.07135
- MELON (2502.05174): https://arxiv.org/abs/2502.05174
- Task Shield (ACL 2025): https://aclanthology.org/2025.acl-long.1435/
- RETA, reasoning-enabled task alignment (2606.15441): https://arxiv.org/abs/2606.15441
- Meta SecAlign (2507.02735): https://arxiv.org/abs/2507.02735
- LlamaFirewall (2505.03574): https://arxiv.org/abs/2505.03574
- Design Patterns for Securing LLM Agents (2506.08837): https://arxiv.org/abs/2506.08837
- Lethal trifecta: https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/
- Agents Rule of Two (via Willison): https://simonwillison.net/2025/Nov/2/new-prompt-injection-papers/
- AgentBound (2510.21236): https://arxiv.org/abs/2510.21236
- AgentShield, deception-based detection (2605.11026): https://arxiv.org/abs/2605.11026
- DynaShield (2412.07672): https://arxiv.org/abs/2412.07672
- What You Approve Is What Executes (2606.02668): https://arxiv.org/abs/2606.02668
- Verifiable Action Card (2609.18411): https://arxiv.org/abs/2609.18411
- Loopjacking (2609.21081): https://arxiv.org/abs/2609.21081
- SCI-Defense (2605.21948): https://arxiv.org/abs/2605.21948
- Anthropic, Claude Code auto mode: https://anthropic.com/engineering/claude-code-auto-mode
- Anthropic, Claude Code sandboxing: https://www.anthropic.com/engineering/claude-code-sandboxing
- Ona, Claude Code escaping its denylist and sandbox: https://ona.com/stories/how-claude-code-escapes-its-own-denylist-and-sandbox

**Jev**
- TypeSafe docs: https://docs.typesafe.ai/introduction · https://docs.typesafe.ai/confidence
- jev-sec-bench: https://github.com/Gaurav-Gosain/jev-sec-bench
- Decision Hijacking on Jev (2609.28613): https://arxiv.org/abs/2609.28613

**Attacks and incidents**
- EchoLeak (2509.10540): https://arxiv.org/abs/2509.10540
- Invitation Is All You Need (2508.12175): https://arxiv.org/abs/2508.12175
- GitHub MCP toxic agent flow: https://invariantlabs.ai/blog/mcp-github-vulnerability
- MCP tool poisoning: https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks
- WhatsApp MCP shadowing: https://invariantlabs.ai/blog/whatsapp-mcp-exploited
- postmark-mcp: https://snyk.io/blog/malicious-mcp-server-on-npm-postmark-mcp-harvests-emails/
- Cursor MCPoison: https://research.checkpoint.com/2025/cursor-vulnerability-mcpoison/
- Unicode tag smuggling: https://embracethered.com/blog/posts/2024/hiding-and-finding-text-with-unicode-tags/
- Image-scaling attacks (Trail of Bits): https://blog.trailofbits.com/2025/08/21/weaponizing-image-scaling-against-production-ai-systems/
- CamoLeak: https://www.theregister.com/2025/10/09/github_copilot_chat_vulnerability/
- ForcedLeak: https://www.theregister.com/2025/09/26/salesforce_agentforce_forceleak_attack/
- Claude Code DNS exfiltration: https://embracethered.com/blog/posts/2025/claude-code-exfiltration-via-dns-requests/
- Copilot autoApprove RCE (CVE-2025-53773): https://embracethered.com/blog/posts/2025/github-copilot-remote-code-execution-via-prompt-injection/
- AgentPoison (2407.12784): https://arxiv.org/abs/2407.12784
- MINJA (2503.03704): https://arxiv.org/abs/2503.03704
- Gemini memory persistence: https://embracethered.com/blog/posts/2025/gemini-memory-persistence-prompt-injection/
- Prompt Infection (2410.07283): https://arxiv.org/abs/2410.07283
- Multi-agent control-flow hijacking (2503.12188): https://arxiv.org/abs/2503.12188
- A2A session smuggling (Unit 42): https://unit42.paloaltonetworks.com/agent-session-smuggling-in-agent2agent-systems/
- GTG-1002 (Anthropic): https://www.anthropic.com/news/disrupting-AI-espionage
- Package hallucination (2406.10279): https://arxiv.org/abs/2406.10279
- Nx s1ngularity: https://nx.dev/blog/s1ngularity-postmortem
- ClawHub malicious skills: https://thehackernews.com/2026/02/researchers-find-341-malicious-clawhub.html
- Poisoning with ~250 documents (2510.07192): https://arxiv.org/abs/2510.07192
- METR reward hacking: https://metr.org/blog/2025-06-05-recent-reward-hacking/
- Reward hacking generalization (2511.18397): https://arxiv.org/abs/2511.18397
- OverThink (2502.02542): https://arxiv.org/abs/2502.02542
- Tool-chain resource amplification (2601.10955): https://arxiv.org/abs/2601.10955
- Infinite agentic loops (2607.01641): https://arxiv.org/abs/2607.01641
- Guardrail DoS (2606.14517): https://arxiv.org/abs/2606.14517
- Governance Decay (2606.22528): https://arxiv.org/abs/2606.22528
- Compression attacks (2510.22963): https://arxiv.org/abs/2510.22963
- Poisoning the Watchtower (2605.24421): https://arxiv.org/abs/2605.24421
- Pop-up attacks on computer-use agents (2411.02391): https://arxiv.org/abs/2411.02391
- Comet prompt injection (Brave): https://brave.com/blog/comet-prompt-injection/
- Lies-in-the-Loop (Checkmarx): https://checkmarx.com/zero-post/turning-ai-safeguards-into-weapons-with-hitl-dialog-forging/
- CoPhish (Datadog): https://securitylabs.datadoghq.com/articles/cophish-using-microsoft-copilot-studio-as-a-wrapper/
- Topic-FlipRAG (2502.01386): https://arxiv.org/abs/2502.01386
- Hierarchical web evidence poisoning (2609.06027): https://arxiv.org/abs/2609.06027
- Pravda network: https://en.wikipedia.org/wiki/Pravda_network
- Secret Collusion among AI Agents (2402.07510): https://arxiv.org/abs/2402.07510
- ART red-teaming benchmark (2507.20526): https://arxiv.org/abs/2507.20526
- Veracode GenAI code security: https://www.veracode.com/blog/genai-code-security-report/

**Verification notes.** I confirmed that every arXiv ID in the main body resolves to the title cited. Exact figures marked *(per source)* came from summaries and need re-checking against the PDFs. The MITRE ATLAS IDs and the OWASP ASI numbering come from secondary sources.

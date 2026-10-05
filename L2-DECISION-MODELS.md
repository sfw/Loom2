# L2 Decision Models: Provider Notes and Calibration

Companion to the L2 Technical Brief, section 7 · last verified October 4, 2026

The brief explains why L2 uses decision models and where. This document holds the material that goes out of date or is purely procedural: what each provider currently offers, and how policies get calibrated and promoted. Re-check every provider note before relying on it.

## 1. Provider notes

### TypeSafe Jev

- **What it is.** TypeSafe describes Jev as a "System One" model. It returns typed, structured decisions instead of generated text ([introduction](https://docs.typesafe.ai/introduction)).
- **Question types.**
  - **Choice** picks an option from a list.
  - **Score** rates the state on a rubric.
  - Both return a probability distribution and a confidence value.
  - **Noul** returns the probability that a statement is true.
- **Usage guidance.** Questions are evaluated in parallel and in isolation against shared state. Adding questions barely changes latency. TypeSafe recommends breaking work into atomic questions and combining the answers in code.
- **Confidence is not a calibrated probability of correctness** ([confidence](https://docs.typesafe.ai/confidence)):
  - Choice: `(max_p − 1/n) / (1 − 1/n)`, which measures how far the top option rises above an even split.
  - Score: confidence also accounts for ordering. Probability on a neighbouring level lowers confidence less than probability on a distant level.
  - Noul: no separate confidence value. The documented equivalent is `|2p − 1|`.

  L2 therefore keeps the raw distributions and sets thresholds per policy on representative data.
- **Vendor performance claims.** TypeSafe has published speed and cost comparisons, with disclosed evaluation caveats ([launch analysis](https://typesafe.ai/blog/introducing-system-one-models-and-jev)). Treat them as vendor results, not as forecasts for L2.
- **Security evidence.**
  - *As a detector.* A community benchmark ([jev-sec-bench](https://github.com/Gaurav-Gosain/jev-sec-bench)) reports 96.5% accuracy and 0.99 ROC-AUC on `deepset/prompt-injections` (662 messages). Adding deployment context gave a large improvement.
    - Caveats: no baselines, a single run, a small public dataset that may have leaked into training, and no adaptive attacker.
  - *As a decision-maker.* An adaptive study ([arXiv 2609.28613](https://arxiv.org/abs/2609.28613)) hijacked 1.8% of decisions with static attacks and 3.5% with adaptive, score-guided attacks, across 510 InjecAgent cases. Typed outputs make hijacking harder but don't remove the risk.
  - No adaptive evaluation of Jev as a detector has been published.

### OpenAI Decisions API

- **Announcement.** At DevDay on September 29, 2026, OpenAI announced finite-answer decisions over text or images, built on "Luna's intelligence". It was in limited preview, with a broad release described as coming "in the coming days" ([recap](https://openai.com/index/devday-2026-recap/)).
- **Not yet established:** a public schema, calibrated probabilities, pricing, or access for this account.
- **L2 position.** Build an adapter only after the API is verified. Don't invent endpoints or assume parity with Jev.

### Frontier-model fallback

A generative model can answer any decision question. When it does, the receipt names a different backend, and the fallback never inherits another provider's calibration.

## 2. Policy calibration procedure

1. **Define the policy.** Specify:
   - the question;
   - the candidate set or rubric;
   - the required state;
   - the action each outcome triggers;
   - the uncertainty route;
   - the insufficient-evidence outcome;
   - the cost of each kind of error.
2. **Build fixtures.** Use clear labels wherever possible. Cover:
   - missing evidence and misleading options;
   - contradictions and prompt injection;
   - domain shift;
   - minority-but-important evidence;
   - benign text full of trigger words, so over-blocking gets measured.

   Keep development and held-out sets separate. Model-generated labels can be used for exploration only.
3. **Measure per action.**
   - Error rates, abstention coverage and latency.
   - For probabilistic outputs: reliability across probability ranges, and proper scoring rules where they apply.
   - For source exclusion, prioritize recall of necessary evidence.
   - For acceptance checks, prioritize false passes.
4. **Compare four configurations.** No gate, a deterministic gate, a decision-model gate and a frontier-model gate. Also run context-only and combined variants. The decision is based on cost per accepted outcome.
5. **Set thresholds** from the error costs, not from the provider's default confidence values.
6. **Promote through the deployment states.**

| State | Meaning | Exit criteria |
| --- | --- | --- |
| Experimental | Offline evaluation only | Passes held-out thresholds |
| Shadow | Runs live, logged, never controls execution | Live error rates match held-out rates within tolerance |
| Advisory | Influences routing; humans or deterministic checks decide | No material harm over a defined volume |
| Active | Controls the routing it was validated for | Ongoing monitoring within bounds |
| Retired | Disabled; receipts preserved | — |

7. **Revalidate** whenever the question, rubric, provider version, input distribution or consequences change.

## 3. Rules that never change

- Never multiply confidences across dependent decisions.
- A decision never grants a capability, passes a hard requirement, or counts as consent.
- An outage invokes the declared fallback or pauses the branch. It never produces a pass.
- Decision calls are outbound data flows. They go through the egress gateway, routed by confidentiality label.

# Phone Assistant: Design Decisions

Status tags: **[decided]** you decided and can explain it, **[proposed]** suggested, needs your confirmation, **[open]** not yet discussed or answered.

## 1. Scope and requirements

**Users**
- Patients calling a practice (inbound). [decided]
- Practice admins: transcripts, summaries, behavior control, analytics. [decided]
- Practice staff who receive handoffs and callback tasks. [proposed]

**In scope for version 1**
- Inquiries, booking, rescheduling, cancelling, for existing patients; public info for anyone. [decided]
- Outbound appointment reminders for patients who have consented. [decided]

**Out of scope for version 1**
- Clinical requests (prescriptions, results, referrals): take a message or callback task. [proposed]
- Changing address, insurance or phone number by voice. [proposed]
- Booking on behalf of another person without a registered proxy. [proposed]

**The assistant must never**
- Mix data between patients. [decided]
- Reveal how the system is built. Design assumes the prompt can leak, so it holds no secrets. [decided]
- Give medical advice or triage. [open: confirm]
- Continue a booking flow when a caller describes an emergency: direct to emergency services and hand off. [open: confirm]

**Handoff to a human**
- Triggers: request beyond capability, caller asks for a human, tool failure, repeated misunderstanding, failed verification after 2 retries. [decided]
- Outside office hours: state opening hours and create a callback task for staff. [proposed]
- Handoff type (live transfer vs approval queue) and context passed to the human. [open]

## 2. Non-functional targets

| Metric | Target | Status |
|---|---|---|
| Voice-to-voice latency (end of caller speech to first audio) | p50 < 1 s, p95 < 2 s | [decided] |
| Speech-to-text word error rate (domain) | < 10%, names and dates checked separately | [decided] |
| Task success (correct end-to-end, in-scope calls) | 85-90% | [decided] |
| Automation rate without a human (excluding hang-ups) | 50-70% at launch | [decided] |
| Emergency-phrase detection recall | as close to 100% as testing shows | [decided] |
| PII leaks / critical safety violations | 0 | [decided] |
| Availability | 99.9% | [decided] |

**Scale assumption (design target):** 100,000 calls/day, peak hour = 15%, average call 3 minutes.
- Little's Law: concurrent calls = arrivals/s x duration = 4.2 x 180 s = **about 750**; with a 2x burst, about 1,500. 5-minute calls: about 1,250, and tokens per minute grow faster because history is longer.
- Illustrative LLM load: about 75 requests/s at 750 calls (one request per 10 s per call), about 3,000 input tokens each, about 13.5M tokens/min. Assumptions, to be replaced by a load test.
- Likely first bottlenecks: LLM tokens-per-minute limits; telephony and speech-to-text concurrency quotas; clinic/calendar API latency; slow autoscaling of stateful sessions.
- Mitigations: prompt caching, history summarization, smaller model for simple turns, provisioned capacity, admission control, overflow to a human, fallback model, circuit breakers. [decided]

## 3. Security and privacy design

**PII handling** [decided]
- Redact PII on every input (speech-to-text output) before it reaches the LLM.
- Output side: the LLM works with random, opaque, per-session placeholders; a resolver fills in real values before text-to-speech.
- The resolver has no global database connection; it uses a per-session allow-list and mapping with an expiry. It blocks and logs any placeholder it did not issue.
- Speech and language providers hosted in the EU under a data processing agreement. Encryption in transit and at rest.
- Honest claim: minimization and defense in depth, not a guarantee that the LLM never sees PII.

**Lazy, progressive authentication** [decided]
- Start anonymous: public info only.
- When a request needs patient data: a knowledge check runs in a deterministic verifier outside the LLM. The LLM only receives "verified: yes/no" and the level.
- Once verified, the session holds for the rest of the call, bound to that one patient, expiring on hang-up and after inactivity.
- Higher-risk actions trigger a step-up or a human, for that action only.
- Failure: 2 retries, then hand off. Attempts counted across calls per patient and per number. Never say which answer was wrong.
- Factors: date of birth + postal code; enrollment ID (random numeric with check digit, keypad entry) as an extra or step-up factor. [proposed: confirm fixed pair vs two-of-three]
- Candidate patient found by name + calling number with tolerant matching; the name disambiguates shared numbers. [proposed]
- Dropped for now: SMS code. Trade-off: less friction, weaker security than a possession factor. [decided]

**Enforcement** [decided]
- The tool layer, not the prompt, decides what is reachable. Patient ID comes from the session, never from model output. Every tool call checks that the object belongs to the session's patient.
- The orchestrator holds the session token; the model never sees it.
- Anyone asking about another patient's appointment: refuse without confirming existence. A registered proxy relationship or a human is the only route.

## 4. Action policy

| Action | Auth | Handling |
|---|---|---|
| Opening hours, address, services | none | automated |
| Read own upcoming appointment | verified | automated |
| Book (existing patient) | verified | automated, read-back, confirmation text to the number on file |
| Reschedule | verified | same, plus rules (for example, inside 24 h goes to a human) |
| Cancel own appointment | verified | check ownership, then policy/fee; state the fee and get consent; no fee: cancel and free the slot; payment via secure link or human, never card details by voice |
| Clinical requests | verified | message or callback task |
| Change personal data | n/a | not by voice in version 1 |
| Act for another person | n/a | human, or registered proxy |

Every write: read-back, idempotency key, audit log entry, text confirmation. [proposed: confirm the table]

## 5. Outbound calls

- Inclusion: yes; patients must have consented; consent stored per patient and channel, revocable, checked before every call. [decided]
- Voicemail detection required. Voicemail message is generic (callback number, no health details), once only. [proposed]
- Opening line discloses nothing and states it is an automated assistant: "Hello, this is an automated assistant. May I speak with [name]?" Practice and reason only after the person confirms and passes verification. [proposed]
- Retries: up to 3 attempts (now, +2 hours, next day), allowed hours only (for example 9:00-19:00 local, weekdays), stop on refusal, opt-out or wrong number, one active call per patient, leftover task to staff. [proposed: you suggested up to 10; I suggested 3]
- To confirm with legal: consent for automated calls, recording notices, AI disclosure. [open]

## 6. Long-term memory

- Stores: language, preferred contact times and channel, preferred practitioner or location, accessibility needs, consent flags, short non-clinical summaries of recent calls, open follow-ups. [proposed]
- Never stores: diagnoses, symptoms, treatments, identifiers such as insurance numbers, raw transcripts beyond retention. [proposed]
- Keyed by internal patient ID, loaded only after verification; the patient database stays the source of truth; supports inspection, correction and erasure; writes come from a validated structured extraction, not free model text. [proposed]

## 7. Audio and conversation

- Endpointing is dynamic, not a fixed silence timeout. [decided] It adapts to what the dialog expects (digits, dates, spellings, yes/no), to unfinished-sounding endings, and to the caller's pace, bounded by minimum and maximum limits. Starting points to tune from data: about 0.5-0.7 s in general, 1.2-2 s for digits and dates, shorter for yes/no. [proposed]
- Barge-in: stop audio quickly, cancel generation, keep only what was heard in the history, then listen; echo cancellation prevents self-interruption. [proposed]
- Low-confidence handling: ask to repeat, read back critical details, keypad fallback; "repeated misunderstanding" = 3 consecutive low-confidence turns. [proposed]
- Realism without deception: natural pacing, short fillers while tools run; the AI disclosure is mandatory and cannot be disabled. [proposed]
- Extra metrics: word error rate under noise, endpointing error rate, barge-in stop latency, false barge-in rate, voicemail detection accuracy, silence-timeout events. [proposed]
- Test data: noisy lines, speakerphone, elderly speakers, accents and dialects, calls from cars. [proposed]
- Phone handed to someone else mid-call: re-verify before the riskiest actions. [proposed]

## 8. Multi-tenancy

- The platform is multi-tenant: each practice or organization is a tenant. [decided]
- The opening line is set by the tenant's admin, not hardcoded. [decided]
  - Template with locked parts (AI disclosure, recording notice where applicable; the outbound first sentence discloses nothing) and editable parts, validation on save, versioning and preview, per-language variants, audit log, automated test run before publishing. Static greetings can be pre-synthesized and cached per tenant and language for instant first audio. [proposed]
- Layers: (1) platform rules tenants cannot loosen (safety and emergency handling, PII handling, disclosure, minimum authentication policy); (2) tenant configuration (opening line, voice, opening hours, appointment types and rules, fee policy, languages, handoff queues, outbound windows within legal limits); (3) per-call context. [proposed]
- Isolation per tenant: data, memory, logs, encryption keys, metrics, test sets, rate limits; protection against noisy neighbors. [proposed]

### Prompt management and guardrails

- Tenants can change prompt content; guardrails cannot be changed by tenants. [decided]
- Platform-owned guardrail instructions are hidden and not editable. Assume they can leak, so they hold no secrets. [decided]
- Tenant prompt content goes through an LLM check for malicious instructions before saving; if flagged, it is blocked and the user is asked to remove the content. [decided] Limit: the check is probabilistic and can itself be injected, so it is one layer only.
- Further layers: structured slots instead of free-form full prompts; deterministic validation (length, disallowed patterns, allowed placeholders); tenant text treated as untrusted at runtime; automated test run (golden conversations, red-team set, guardrail tests) before publishing; staged rollout with rollback; versioning, audit log, role-based edit rights. [proposed]
- Guardrails are enforced in code, not only in prompts: cross-patient access (tool layer), PII (redaction and resolver), emergencies (dedicated detector forcing a handoff script), writes (state machine requires confirmation), verification (state machine plus tool layer), disclosure and recording notice (fixed greeting template), fees and policy (deterministic policy engine), quotas (gateway), outbound consent and hours (dialer scheduler). [proposed]

### Load protection (noisy neighbor)

- Autoscaling covers only our own compute. External quotas and fairness need: per-tenant quotas, priority classes (inbound over outbound), outbound campaigns paced through queues in separate pools, reserved inbound capacity, admission control, overflow to a human. [proposed]
- Illustration (3-minute calls): 20,000 reminders over 2 hours is about 500 concurrent calls; over 8 hours, about 125.

### AI disclosure

- A locked, non-editable statement at the start of every call that the caller is speaking with an automated AI assistant; separate from the recording notice. Verify legal requirements. [decided]

### Red teaming and release gates

- The system is red-teamed before deployment. [decided]
- Continuous, not one-off: automated attack suite in CI on every model, platform prompt, guardrail, and tenant prompt or configuration change; periodic manual red teams after launch; monitoring for attack patterns in production. [proposed]
- Red team is independent of the builders. Release gate set up front, for example zero open critical findings, all high findings fixed or formally accepted, zero PII leaks in the test suite. [proposed]
- Threat model covers: social-engineering callers, technical attacks on verification, malicious or compromised tenant admins, indirect injection through data the agent reads back (for example free-text fields), long-term memory poisoning, abuse and cost attacks (very long calls, looping tool calls, SMS or call pumping), third-party provider failure. [proposed]

### Telephony

- Buy or integrate the telephony layer (numbers, carrier connectivity, SIP, codecs); build the intelligence layer behind a thin adapter, so providers can be swapped, run in parallel for resilience, or simulated in tests. [proposed]
- Adapter events (provider to core): call started, audio in, keypad digit, call ended, call state (ringing, answered, busy, no answer, human or voicemail), playback marker reached. [proposed]
- Adapter commands (core to provider): answer, send audio, clear audio (barge-in), insert marker, transfer, hang up, place outbound call, start or stop recording where consent exists. [proposed]
- To settle when choosing a provider: EU data residency and sub-processors, recording, per-tenant numbers and porting, cost per minute, region and latency, concurrency quotas. [open]

## 9. Still to design

1. Languages. [open]
2. High-level architecture and the agent core (orchestration, state machine, tools). [open]
3. Voice pipeline: barge-in, turn-taking, latency budget per hop. [open]
4. Evaluation: offline datasets, simulated calls, A/B tests. [open]
5. Observability, SLOs, alerts. [open]
6. Telephony, speech-to-text, text-to-speech, LLM choices. [open]
7. Integration with practice systems (calendar, records): adapters, availability, conflicts, double booking, idempotency, slow or failing APIs. [open]
8. Resilience: provider outages and fallbacks, "take a message" mode, dropped calls mid-booking, partial failures, disaster recovery. [open]
9. Compliance and data lifecycle: recordings and transcripts (retention, access), patient access and erasure rights, data residency, data protection impact assessment, sub-processors; check medical-device software status, EU AI Act classification, health-data hosting rules; needs legal input. [open]
10. Evaluation and A/B testing: simulated callers, noisy-audio sets, human review, A/B design with guardrail metrics. [open]
11. Rollout: pilot tenants, shadow mode, gradual autonomy, per-tenant kill switch. [open]
12. Handoff mechanics: warm transfer, context summary, queues. [open]
13. Accessibility: hearing or speech impairments, elderly and non-native speakers, alternative channels. [open]
14. Emergency handling details: per-country emergency numbers, urgent-but-not-emergency cases, test set. [open]
15. Cost per call, per-tenant budgets, call-duration limits. [open]
16. Testing strategy: unit, contract, end-to-end with simulated telephony, load and chaos tests. [open]
17. Model and provider strategy: fallback models, version pinning, regression tests on provider updates. [open]
18. Admin console and feedback loop: transcript access control, flagged calls into evaluation sets. [open]

## 10. Interview one-liners

- "I authenticate lazily, at the level the requested action needs, and keep the session scoped to one patient and short-lived."
- "Authorization is enforced in the tool layer, so a fooled model still can't reach another patient's data."
- "I claim minimization and defense in depth for PII, not that the model never sees any."
- "I size for peak concurrency with Little's Law, then find the first bottleneck with a load test."

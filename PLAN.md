# Phone Assistant (Voice Agent): Interview Prep Project

Target role: Senior AI Engineer (Python), Phone Assistant, Conversational AI team, Doctolib (Berlin).
Purpose: a small but production-minded agent project that doubles as Feature Building practice and as concrete material for the AI System Design interview.

## 1. Interview process (from the job description)

1. Recruiter interview
2. Feature Building interview (likely live/pair coding in Python; confirm format with recruiter)
3. AI System Design interview
4. Behavioral interview
5. Reference check

What the JD stresses: production agentic LLM systems, offline evals plus A/B tests on real traffic, guardrails and safety, memory, reasoning orchestration, tool adapters, observability and fallbacks, SLO/SLI, privacy and security by design, automated testing, CI/CD, "you build it you run it", mentoring and pair programming.

## 2. Gaps to prepare for

| Gap | Preparation |
|---|---|
| No medical-domain experience | Honest bridge: same reliability and safety problems; learned a new domain before (FinChat, Yoruba financial dataset) |
| No A/B testing on real traffic on the CV | Prepare how to run one (metric, randomization, guardrail metrics, sample size); be honest about what was done |
| Mentoring / pair programming not on the CV | Two real stories |
| AIVIRA voice bullet says "collaborated" | Be able to state exactly what you personally owned |
| "6+ years software engineering" vs ~5.5 years in ML/AI roles | Short, confident answer |

## 3. Design principle: text-first, voice-ready

Voice can be added later, but only if the text-first version is designed for it. Voice is not a wrapper around a text agent.

### Carries over unchanged
- Agent core: orchestration, tool adapters, memory, guardrails, fallbacks
- Eval harness (extended with audio-derived cases)
- Observability, tests, data model

### Build in from day one
1. **Streaming and async:** stream tokens, never return whole responses. A voice turn has a tight end-to-end budget split across speech-to-text, LLM and text-to-speech.
2. **Channel-agnostic core:** the agent talks to a small interface (`receive_utterance`, `emit_response`), never to HTTP or a chat UI. Text and voice are adapters.
3. **Barge-in and turn-taking:** cancellable responses and a state machine (listening / thinking / speaking).
4. **Noisy input:** simulate transcription errors in eval data; require explicit confirmation of critical details (name, date, time).
5. **Voice-friendly output:** short sentences, no markdown or lists, numbers and dates written to be spoken; a separate output-formatting step.
6. **No screen:** design clarification and confirmation as spoken patterns.
7. **Voice-only failure modes:** silence, dropped calls, background noise, hang-up mid tool call, handoff to a human receptionist with context.

## 4. Target system (design to be able to whiteboard)

```
Caller -> Telephony -> STT (streaming) -> Agent core -> TTS (streaming) -> Caller
                                            |  memory
                                            |  tool adapters (booking, calendar, patient lookup)
                                            |  guardrails / policy
                                            |  fallback -> human handoff
Cross-cutting: tracing, metrics, SLOs, PII handling, audit log, evals
```

Points to cover in the design interview:
- Latency budget per hop, and what to do when it is exceeded (filler phrases, shorter answers, model fallback)
- Memory: short-term (conversation state) vs long-term (patient preferences), and what must never be stored
- Tools: idempotency, confirmation before side effects, timeouts, retries
- Safety: no medical advice, prompt-injection defense, escalation rules
- Privacy: health data is special-category data under GDPR; look up Doctolib's compliance context instead of guessing; PII redaction, retention, audit logging
- Evaluation: offline datasets, simulated calls, A/B tests; metrics: task success, containment rate, word error rate, latency percentiles, escalation quality, safety violations
- Operations: SLOs and SLIs, dashboards, alerts, provider outage behavior

## 5. Build plan

Working mode: I scaffold and write boilerplate; you write the core logic and I review it like the interviewer would.

| Phase | Build | Interview link |
|---|---|---|
| 0 | Repo scaffold, config, pytest, ruff, CI-style Makefile, typed models | Code quality, testing culture |
| 1 | Channel-agnostic agent core: conversation state machine, streaming interface | Feature Building, System Design |
| 2 | Tool adapters for appointment booking (fake clinic backend), idempotency, timeouts, retries | Tool adapters, fail-safe behavior |
| 3 | Memory: short-term state and a small long-term store | Memory design |
| 4 | Guardrails and policy layer: no medical advice, injection tests, escalation to human | Safety, security posture |
| 5 | Eval harness: scripted scenarios, simulated STT errors, LLM-as-judge plus deterministic checks | Offline evaluation, A/B design |
| 6 | Observability: structured logs, traces, metrics, SLOs | Own performance and observability |
| 7 | Voice adapter: simulated STT/TTS with injected latency and errors; optional real API | Voice vs chat |
| 8 | Write-up: one-page design doc and a 30-minute spoken walkthrough | AI System Design |

## 6. Other preparation tracks

- **Story bank (behavioral):** STAR stories mapped to the JD (production incident, trade-off, mentoring, disagreement, ownership).
- **CV defense:** SAP multi-agent PoC first (53% to 76% task success, 6.6x fewer tokens), then Anena, AIVIRA, red teaming, EqualyzAI. For each: problem, role, approach and alternatives, measurement, failure or trade-off.
- **Feature Building drills:** timed exercises (write a tool adapter with tests, add retry/timeout, add a guardrail) with narration.

## 7. Open decisions

- Real STT/TTS in Phase 7 or simulation only? (Default: simulate first, real API optional.)
- LLM provider for the demo agent (default: Anthropic API or a local stub for tests).
- Date and stage of the next interview: decides how much of this we can do.
- Format of the Feature Building round (language, tools, AI allowed or not): ask the recruiter.

## 8. Definition of done

- [ ] Agent handles a complete booking conversation with tool calls, confirmation and human handoff
- [ ] Tests pass, including guardrail and failure-path tests
- [ ] Eval harness reports task success, latency percentiles and safety violations
- [ ] Simulated voice adapter demonstrates barge-in and noisy-transcript handling
- [ ] One-page design doc written and rehearsed aloud
- [ ] Story bank and CV defense answers written in your own words

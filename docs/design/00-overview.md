# System Overview

Status: draft v0.1, for review. Decisions come from `docs/DESIGN.md` (tags decided / proposed / open).

## 1. Purpose and scope

A multi-tenant phone assistant for healthcare practices. Patients call (inbound) or are called (outbound reminders). The assistant handles inquiries, booking, rescheduling and cancelling for existing patients, hands over to a human when needed, and gives practice admins transcripts, control and analytics.

Two halves:
- **Real-time path (per call):** audio in, decisions, audio out. Every millisecond counts.
- **Control path (per tenant):** configuration, admin console, evaluation, monitoring.

## 2. Component map

This is a structure diagram. The sequence diagrams below show behavior over time.

```mermaid
flowchart LR
    C(["Caller"]) <--> TP["Telephony provider"]
    TP <--> TE["Telephony edge"]
    TE <--> AP["Audio pipeline: voice detection, speech-to-text, text-to-speech"]
    AP <--> OR["Orchestrator and state machine"]
    OR <--> PV["Privacy service"]
    OR <--> SG["Safety layer"]
    OR <--> AG["Agent core, LLM"]
    OR <--> VF["Verifier"]
    OR <--> TL["Tool layer"]
    TL <--> PS[("Practice systems")]
    OR <--> MEM[("Memory service")]
    OR --> HO["Handoff and callbacks"]
    subgraph CP["Control path"]
        CFG["Tenant config and prompts"]
        ADM["Admin console"]
        EVAL["Evaluation and red team"]
        OBS["Observability"]
    end
    CFG -.-> OR
    OR -.-> OBS
    ADM -.-> CFG
    EVAL -.-> AG
```

## 3. Design rules that shape every diagram

1. The LLM proposes, the state machine disposes. Guardrails are enforced in code.
2. Raw PII never reaches the LLM or the logs. Redaction on input, opaque per-session placeholders on output.
3. Verification runs in a deterministic verifier outside the LLM. The LLM only learns "verified, level".
4. The session token is held by the orchestrator, never by the model.
5. The tool layer decides what is reachable. The patient ID comes from the session, and every object is checked for ownership.
6. A confirmation is bound to one exact proposal. Any change voids it.
7. Every write carries an idempotency key and produces an audit entry.
8. Emergency, "human please" and hang-up work from any state.
9. Tenant configuration can tighten platform rules but never loosen them.

## 4. Sequence diagrams

### 4.1 One conversational turn (audio path)

Every turn in the other diagrams travels this path. While the state is VERIFYING, the transcript goes to the verifier instead of the LLM (see 4.2).

```mermaid
sequenceDiagram
    autonumber
    actor C as Caller
    participant TP as Telephony provider
    participant TE as Telephony edge
    participant AP as Audio pipeline
    participant PV as Privacy service
    participant SG as Safety layer
    participant OR as Orchestrator
    participant AG as Agent core LLM

    C->>TP: Speech
    TP->>TE: Audio frames of 20 ms
    TE->>AP: Frames
    AP->>AP: Decode, resample, clean, detect speech
    AP->>AP: Streaming speech-to-text and endpointing
    AP->>OR: Final transcript
    OR->>PV: Transcript
    PV->>PV: Redact PII, keep placeholder map for this session
    PV->>SG: Redacted text
    SG->>SG: Emergency and injection checks
    alt Emergency detected
        SG->>OR: Emergency flag
        OR->>OR: State = EMERGENCY_HANDOFF
    else Normal turn
        SG->>OR: Text and flags
        OR->>AG: Redacted text, state, allowed tools
        AG->>OR: Reply with placeholders
        OR->>SG: Output check
        SG->>PV: Approved reply
        PV->>PV: Resolve placeholders, session allow-list only
        PV->>AP: Final reply text
        AP->>AP: Streaming text-to-speech, resample, encode
        AP->>TE: Audio frames
        TE->>TP: Audio out
        TP->>C: Speech
    end
```

### 4.2 Inbound call: connect and verify

```mermaid
sequenceDiagram
    autonumber
    actor C as Caller
    participant TP as Telephony provider
    participant TE as Telephony edge
    participant CFG as Tenant config store
    participant OR as Orchestrator
    participant AP as Audio pipeline
    participant VF as Verifier
    participant PD as Patient directory

    C->>TP: Dials the practice number
    TP->>TE: Call started with call id and dialed number
    TE->>CFG: Look up tenant by dialed number
    CFG->>TE: Tenant id and config version
    TE->>OR: Create session for tenant and call
    OR->>CFG: Load tenant config
    OR->>OR: State = ANONYMOUS
    OR->>AP: Play locked AI disclosure and tenant greeting, cached audio
    AP->>TE: Audio frames
    TE->>TP: Audio out
    TP->>C: Disclosure and greeting
    Note over C,OR: Later the caller asks for something that needs patient data
    OR->>OR: State = VERIFYING, LLM not involved
    OR->>AP: Ask for name, date of birth and postal code, fixed script
    C->>TP: Speaks the answers
    TP->>AP: Audio
    AP->>OR: Raw transcript
    Note over OR,VF: Raw answers never reach the LLM or the logs
    OR->>VF: Name and answers for deterministic check
    VF->>PD: Find candidate and compare factors
    PD->>VF: Match result
    alt Verified
        VF->>OR: Verified, level 1, patient id
        OR->>OR: State = VERIFIED, create session token held by orchestrator only
        OR->>OR: Long-term memory may now be loaded
    else Failed
        VF->>OR: Not verified, attempt counted per patient and per number
        alt Attempts remaining
            OR->>AP: Ask again without saying which answer was wrong
        else Two failures
            OR->>OR: State = HANDOFF
        end
    end
```

### 4.3 Inbound call: reschedule with confirmation

Each message to or from the caller travels the turn path in 4.1.

```mermaid
sequenceDiagram
    autonumber
    actor C as Caller
    participant OR as Orchestrator
    participant AG as Agent core LLM
    participant TL as Tool layer
    participant PS as Practice system
    participant NS as Notification service
    participant AU as Audit log

    C->>OR: Move my Tuesday appointment to Thursday morning
    OR->>OR: State = COLLECTING_DETAILS
    OR->>AG: Redacted text, allowed tools list and availability
    AG->>OR: Tool call list_appointments
    OR->>TL: list_appointments with session token
    TL->>TL: Authorize, patient id comes from the session
    TL->>PS: Query this patient's appointments
    PS->>TL: Appointments
    TL->>OR: Results with opaque handles and placeholders
    OR->>AG: Results
    AG->>OR: Tool call check_availability for Thursday morning
    OR->>TL: check_availability
    TL->>PS: Free slots
    PS->>TL: Slots
    TL->>OR: Slots
    OR->>AG: Slots
    AG->>OR: Proposal, move appointment handle 1 to Thursday 9:30
    OR->>OR: Store proposal with hash, State = AWAITING_CONFIRMATION
    OR->>C: Read-back, so that is Thursday at 9:30, correct?
    alt Caller changes a detail
        C->>OR: Make it Friday
        OR->>OR: Proposal void, State = COLLECTING_DETAILS
    else Explicit yes
        C->>OR: Yes
        OR->>OR: State = EXECUTING
        OR->>TL: reschedule with proposal hash and idempotency key
        TL->>TL: Check ownership and that parameters match the confirmed proposal
        TL->>PS: Update appointment
        alt Success
            PS->>TL: OK
            TL->>NS: Send text confirmation to the number on file
            TL->>AU: Write audit entry
            TL->>OR: Success
            OR->>OR: State = DONE
            OR->>C: Done, you will get a text confirmation
        else Failure or timeout
            PS->>TL: Error
            TL->>OR: Failure
            OR->>OR: Retry once with the same idempotency key, else HANDOFF or callback task
        end
    end
```

### 4.4 Barge-in

```mermaid
sequenceDiagram
    autonumber
    actor C as Caller
    participant TP as Telephony provider
    participant TE as Telephony edge
    participant AP as Audio pipeline
    participant OR as Orchestrator
    participant AG as Agent core LLM

    AP->>TE: Reply audio frames with markers
    TE->>TP: Audio out
    TP->>C: Agent speaking
    C->>TP: Caller talks over the agent
    TP->>TE: Audio in
    TE->>AP: Frames
    AP->>AP: Voice detection with echo cancelled, confirms real speech
    AP->>OR: Barge-in detected
    par Clear audio
        OR->>TE: Clear queued playback
        TE->>TP: Clear command
    and Cancel generation
        OR->>AG: Cancel generation
    and Stop speech synthesis
        OR->>AP: Stop text-to-speech
    end
    TP->>TE: Last playback marker reached
    TE->>OR: Position of what was actually heard
    OR->>OR: Trim the reply in history to the heard part, State = LISTENING
    AP->>OR: New utterance transcript
    OR->>AG: Continue with the caller's new input
```

### 4.5 Outbound reminder call

```mermaid
sequenceDiagram
    autonumber
    participant SCH as Campaign scheduler
    participant CON as Consent registry
    participant OR as Orchestrator
    participant TE as Telephony edge
    participant TP as Telephony provider
    actor R as Recipient
    participant VF as Verifier

    SCH->>SCH: Pick due task inside allowed hours and tenant quota
    SCH->>CON: Consent valid for this patient and channel?
    alt No consent
        CON->>SCH: Not allowed
        SCH->>SCH: Cancel task and log
    else Consent valid
        CON->>SCH: Allowed
        SCH->>OR: Create outbound session with idempotency key
        OR->>TE: Place call from the tenant's number
        TE->>TP: Place outbound call
        TP->>R: Rings
        alt No answer or busy
            TP->>TE: Call state no answer
            TE->>OR: No answer
            OR->>SCH: Schedule retry, attempt 1 of 3
        else Voicemail detected
            TP->>TE: Answered, machine detected
            OR->>TE: Play generic message with callback number, no health details
            OR->>SCH: Mark done, leave a message once only
        else Answered by a person
            TP->>TE: Answered, human
            OR->>TE: Hello, this is an automated assistant. May I speak with the patient?
            R->>TP: Speaks
            alt Wrong person or refuses
                OR->>TE: Polite goodbye, disclose nothing
                OR->>SCH: Wrong number or opt-out recorded, stop retries
            else Is the patient
                OR->>OR: State = VERIFYING
                OR->>VF: Knowledge check, same as inbound
                alt Verified
                    OR->>TE: Name the practice and the reminder, offer confirm, reschedule or cancel
                    OR->>OR: Continue with the normal task flow
                else Failed
                    OR->>TE: Ask the person to call the practice
                    OR->>SCH: Create staff task
                end
            end
        end
    end
```

## 5. Component designs (one file each, summary plus detail)

| # | Component | File | Status |
|---|---|---|---|
| 1 | Telephony edge | 01-telephony-edge.md | not started |
| 2 | Audio and turn-taking pipeline | 02-audio-pipeline.md | not started |
| 3 | Session and conversation state | 03-session-state.md | not started |
| 4 | Privacy pipeline | 04-privacy.md | not started |
| 5 | Identity and verification | 05-verification.md | not started |
| 6 | Agent core | 06-agent-core.md | not started |
| 7 | Tools and practice-system integration | 07-tools-integration.md | not started |
| 8 | Safety and guardrails | 08-safety.md | not started |
| 9 | Human handoff and callbacks | 09-handoff.md | not started |
| 10 | Long-term memory | 10-memory.md | not started |
| 11 | Outbound campaigns | 11-outbound.md | not started |
| 12 | Tenant configuration and prompts | 12-tenant-config.md | not started |
| 13 | Admin console and analytics | 13-admin-console.md | not started |
| 14 | Evaluation and testing | 14-evaluation.md | not started |
| 15 | Observability | 15-observability.md | not started |
| 16 | Capacity, resilience and cost | 16-capacity-resilience.md | not started |
| 17 | Security, compliance, data lifecycle | 17-security-compliance.md | not started |
| 18 | Deployment and rollout | 18-deployment.md | not started |

## 6. Open questions

- Latency budget per hop for the turn path in 4.1. [open]
- Whether privacy and safety run in-process with the orchestrator or as separate services (latency against isolation). [open]
- Where per-call state is stored and how a dropped call is resumed. [open]
- Telephony, speech and LLM providers. [open]

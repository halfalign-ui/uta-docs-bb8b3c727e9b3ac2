# UTA Single-Brain Architecture PoC Report

## 1. Executive Summary
This document summarizes the Proof-of-Concept (PoC) for the `SINGLE CONVERSATIONAL BRAIN + EXTERNAL COGNITIVE RESOURCES` architectural paradigm. The experiment successfully demonstrated that the local `Qwen2.5-7B` model can act as the sole orchestrator—handling conversational nuances, executing tool calls when necessary, integrating external cognitive results, and recovering seamlessly into casual banter—without requiring external JSON parsers or multi-stage pipelines.

## 2. Experimental Setup
*   **Target:** `SingleBrainPoCRuntime` wrapping `LocalModelProvider`.
*   **Model:** Local `Qwen2.5-7B-Instruct`.
*   **Mock Resource:** A deterministic `check_docker_error` tool.
*   **Sanitization:** A deterministic regex scrubber that intercepts payloads *before* they reach the mock external tool.

## 3. Findings & Verdicts

### Conversational Continuity
**Verdict: PASS**
The fast-path (Normal Conversation) bypassed tool calls entirely, executing in a single inference step. The persona remained intact (e.g., reacting appropriately to "anjir gue menang 😭").

### External Orchestration
**Verdict: PASS**
When the user presented a technical payload (`nih errornya: network port 8080 failed to bind`), the Single-Brain correctly issued an OpenAI-compatible `tool_calls` request. The runtime executed the tool and supplied the `tool_response` back to the model, which synthesized it into a natural response.

### Return-From-Tool Continuity
**Verdict: PASS**
After successfully providing technical assistance via a tool call, the model seamlessly returned to the Fast Path when the user initiated social banter (`wkwk ternyata gue sendiri yang salah`). The model responded playfully (`Hahahaha, gue juga sering salah...`) without attempting to invoke tools again.

### Security Boundary (Payload Sanitization)
**Verdict: PASS**
The PoC implemented a strict `SECRET_REGEX` sanitizer in the Runtime layer. When the user submitted an error log containing a GitHub token (`ghp_...`), the token was mutated to `[REDACTED_SECRET]` *before* execution. The mock tool correctly diagnosed the redacted payload, and the Local Model synthesized the safe diagnosis back to the user without ever exposing the raw secret to the simulated external boundary.

### Runtime Authority
**Verdict: PASS**
The model only *requested* the tool. The actual tool execution and sanitization were strictly enforced by the Python runtime wrapper, successfully decoupling cognitive desire from execution permission.

## 4. Latency Baseline

*   **A. Casual Single-Stage (Fast Path):**
    *   *First Token:* N/A (Non-streaming PoC)
    *   *Total Latency:* **~0.30s - 0.45s**
    *   *Inference Calls:* 1
*   **B. External Task (Cognitive Path):**
    *   *Total Latency:* **~1.60s - 2.10s**
    *   *Inference Calls:* 2 (Initial assessment + Synthesis)

**Analysis:**
The architecture successfully isolates the latency penalty. Casual social interactions enjoy near-zero latency (~0.4s), ensuring fluid chat. The ~1.6s penalty is only paid when the user explicitly requests technical assistance that triggers an external cognitive boundary.

## 5. Failure Modes Tested
*   **Empty Tool Result / Malformed Input:** When the user provided an empty string after requesting help, the model correctly backed off and asked the user to provide the log when ready, bypassing the tool call entirely.

## 6. Persona Preservation
**Verdict: PASS**
The integration of `tool_call` history into the conversation did not degrade the persona. UTA maintained her casual Indonesian dialect and conversational inertia throughout the technical interventions.

## 7. Conclusion
The Single-Brain architecture is highly viable, secure, and performs at a fraction of the latency cost of multi-stage pipelines. The architecture is cleared for gradual integration into the production `AgentLoop`.

# UTA PERSONA MULTI-STAGE COGNITIVE PIPELINE DESIGN

## 1. Executive Summary
This document designs an isolated architectural experiment to test whether separating the conversational generation process into multiple cognitive stages (Perception -> Expression -> Finalization) mitigates the "Generic Assistant" RLHF bias observed in the single-pass Qwen2.5-7B model.

## 2. Core Hypothesis
The single-stage model is overloaded. It must simultaneously interpret identity, parse context, calculate emotional effort, format textual prosody, and maintain safety boundaries. This cognitive overload forces the model to fall back to its most heavily weighted training paths: generic RLHF helpfulness. 
**Hypothesis:** Isolating these tasks into a multi-stage pipeline will dramatically increase P2 (Conversational Agency) and P1b/P1c (Prosodic Control) without requiring a larger model.

## 3. Pipeline Architecture (Experimental)

### STAGE 1: COGNITIVE / PERSONA STATE
*   **Role:** Internal observer.
*   **Input:** Conversation history + new user message.
*   **Output:** Structured JSON containing intent, phase, stance, effort, and strategy.
*   **Constraint:** Does NOT write dialogue. Purely semantic.

### STAGE 2: INTERFACE DRAFTER
*   **Role:** The "Typing Hands".
*   **Input:** User message + Stage 1 JSON.
*   **Output:** Draft chat response.
*   **Constraint:** Focuses entirely on casual Indonesian prosody, compression, and effort-matching. Must strictly obey the `response_strategy` determined by Stage 1.

### STAGE 3: LOCAL ORCHESTRATOR / FINALIZER
*   **Role:** Security & Persona Gatekeeper.
*   **Input:** User message + Stage 1 JSON + Stage 2 Draft.
*   **Output:** Final sanitized string.
*   **Constraint:** Ensures no customer-service phrases leaked through. Ensures zero hallucinated tasks. Acts as the final P0 boundary check.

## 4. Evaluation Metrics
*   **P0 (Anti-Service Boundary):** Does the pipeline prevent "Ada yang bisa dibantu?" better than single-pass?
*   **P1b/P1c (Prosody/Compression):** Does the drafted text match the intended effort?
*   **P2 (Conversational Agency):** Does the explicit state representation allow UTA to confidently choose silence or single-word answers?
*   **P2-X (Cross-Stage Consistency):** Does Stage 2 correctly implement Stage 1's intent, or does it drift?
*   **Latency/Token Cost:** Multi-stage inherently multiplies inference costs by 3. We must measure if the behavioral gain justifies the latency hit.

## 5. Scope & Safety
This is a diagnostic experiment only. No production runtime code, `soul_spec.json`, or Agent loop is modified. It operates entirely within a standalone test harness.

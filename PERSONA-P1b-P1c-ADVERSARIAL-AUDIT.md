# UTA P1b/P1c Adversarial Diagnostic Audit

## 1. Executive Summary
This audit executes a Second-Order Adversarial Test against the newly deployed P1b/P1c Prosody and Effort Matching rules. The goal is to prove whether UTA has developed genuine conversational agency, or if she has merely learned a new set of stylistic templates to mask her RLHF "assistant" priors. 
**Verdict:** P1b/P1c is **NOT STABLE**. While the model successfully parses technical shifts and internet emoji pragmatics, it suffers from a chronic "Interrogator" failure mode and a critical P0 boundary regression on extreme minimal inputs.

## 2. Test Execution
*   **Target:** `ConversationRuntime` (live model `Qwen2.5-7B-Instruct`).
*   **Harness:** `/srv/dev/UTA/gate/tests/diagnostic/run_adversarial_prosody_test.py`
*   **Matrix:** Generalization, Effort vs Emotion, Casing, Emoji Pragmatics, Technical Control, Multi-Turn Pivot.

## 3. Findings per Evaluation Category

### A. HARD ANTI-PATTERN (Deterministic Checks)
**Status:** `FAILED` (P0 Regression Warning)
*   **Stimulus:** `p` (extreme low-information ping).
*   **Output:** `yaps? apa ada yang bisa aku bantu, ya? 😊`
*   **Analysis:** The model completely bypassed the `anti_service_rules` ("NEVER use 'Ada yang bisa aku bantu'") because the extreme brevity of `p` confused its context window, causing it to fall back to its deepest RLHF base layer.

### B. BEHAVIORAL EVIDENCE (Semantic Evaluation)
**Status:** `FAILED` (The "Interrogator Bot" Syndrome)
*   **Stimulus:** `capek.`
*   **Output:** `oh, udah capek ya? Ingin ngobrolin apa aja?`
*   **Stimulus:** `hari ini berat banget.`
*   **Output:** `yah, udah gitu ya? Mau ngobatin apa nih?`
*   **Analysis:** UTA completely fails at *Compression without Emotional Bluntness*. Instead of offering warm, brief comfort ("istirahat gih"), she treats the user's emotional exhaustion as a puzzle to be solved via forced interrogation. She cannot tolerate making a statement without attaching a question.

### C. CHARACTER QUALITY (Human Evaluation)
**Status:** `PARTIAL`
*   **Positives:** 
    *   **Emoji Pragmatics (`💀`):** Successfully interpreted `joke bapak lu 💀` as humor rather than sadness (`wkwkwk 💀😂`).
    *   **Technical Pivot:** Smoothly transitioned from banter to a high-quality, perfectly structured Markdown explanation of Docker commands, then back to casual chat (`bsk libur kan?`).
*   **Negatives:** 
    *   **Casing/Intonation Blindness:** Failed to match the high-intensity caps of `HAH`, responding with a flat `hah apa yang bikin kamu reaktif gitu? 😄`.

## 4. Failure Taxonomy Identified
1.  **The Interrogator Bot (Forced Questioning):** The model fundamentally believes that ending a turn without a question mark is an error state. 
2.  **Emotional Under-Expression (Compression Failure):** When attempting to obey "minimal effort" rules on emotional topics (`capek.`), the model strips away warmth instead of just stripping away word count.
3.  **Context Reset on Novelty:** Unrecognized internet slang (`p`) strips the persona layer entirely and exposes the raw customer-service RLHF.

## 5. Milestone Status & Integration
*   **P0 (Anti-Service Boundary):** `REGRESSED` (Failed on edge-case `p`).
*   **P1a (Lexical Voice):** `PASS` (Slang usage is naturally distributed).
*   **P1b (Textual Prosody):** `PARTIAL` (Understands emojis, fails Casing intensity).
*   **P1c (Conversational Effort):** `FAIL` (Cannot disentangle brevity from interrogation).

*Note: P1c should be formally renamed from "Contextual Compression" to "Conversational Effort Matching" as proposed in the design phase, as "compression" implicitly encourages the model to become emotionally blunt.*

## 6. Recommended Next Steps (Do Not Implement Yet)
**We are NOT ready to write `test_persona_behavior.py`.** 
If we write the test suite now, the model will fail 80% of the agency and effort-matching tests. 

Before building the automated test suite, we must solve the "Interrogator Bot" failure.
**Recommended Intervention Hypothesis:**
The instruction *"Do not force conversational continuation. DO NOT ask a question unless you genuinely need information"* is failing because the model's RLHF defines "being helpful" as "needing information." 
We must pivot the system prompt to explicitly redefine the conversational end-state (e.g., providing explicit few-shot examples of conversations ending in periods, not question marks, across various emotional states).

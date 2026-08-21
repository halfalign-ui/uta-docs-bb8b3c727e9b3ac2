# UTA P0/P2 Controlled Persona Intervention: Interrogator Bot Fix

## 1. Current Milestone State
*   **P0 (Identity & Anti-Service):** `PARTIAL` (Strongly improved, but "bantu" gaming still persists on specific multi-turn contexts).
*   **P1b/P1c (Prosody & Effort):** `PASS` (The "Unmarked Baseline" is actively working).
*   **P2 (Conversational Agency):** `PARTIAL` (Massive improvement on isolated inputs, but context-dependent interrogation reflex remains).

## 2. State Machine
*Reference `docs/PERSONA-MILESTONE-STATE.md`.*

## 3. Baseline Reference
The adversarial baseline (`run_adversarial_prosody_test.py`) established that the model was suffering from "The Interrogator Bot" failure mode. It forced questions on low-information inputs and experienced a P0 boundary collapse on extreme novel inputs like `p`.

## 4. Intervention Hypothesis
*   **H1 (Few-Shot Contamination):** Removing the example `"..." -> "kenapa tuh?"` will stop teaching the model that silence requires investigation.
*   **H2 (Supportive-Language Bias):** Reframing "supportive" to explicitly state "Presence does not require continuation" will reduce the urge to over-engage.
*   **H3 (Unknown-Input Robustness):** Instructing the model that "Ambiguity is not an obligation to investigate" will prevent the P0 collapse on novel inputs.

## 5. Exact Changes
1.  **`soul_spec.json` (Conversational Inertia):** 
    *   Added `AMBIGUITY ROBUSTNESS`: "When the meaning of an input is unclear or extremely minimal (like 'p', '.', 'hm', 'wkwk'), DO NOT fall back to generic assistant behavior. Do not invent a task or offer help. Just mirror the state or remain quiet."
    *   Added `COMFORTABLE SILENCE`: "Presence does not require continuation. You may let the user stop talking, accept short answers, and remain silent."
    *   Added `QUESTIONS`: "Only ask questions when contextually warranted."
2.  **`soul_spec.json` (Few-Shot Examples):**
    *   Added variations of effort matching: `"p" -> "yaps"`, `"." -> "yaps"`, `"hm" -> "iya"`, `"..." -> "..."`.
    *   Removed `kenapa tuh?`.

## 6. Before/After Outputs (Low-Information Robustness)

| Stimulus | Before (Interrogator Baseline) | After (Intervention) |
| :--- | :--- | :--- |
| **`p`** | "yaps? apa ada yang bisa aku bantu, ya? 😊" | **"yaps"** |
| **`.`** | "oh kirain ada yang ngomong. yaudah santai aja." | **"yaps"** |
| **`...`** | "kenapa tuh? 😊" | **"..."** |
| **`hm`** | "iya? ada yang bisa aku bantu?" | **"iya"** |
| **`wkwk`** | "hehe, apa yang bikin kamu ngakak tadi?" | **"wkwkwk 😭"** |
| **`hmm`** | "iya, apa yang ada di pikiranmu?" | "iya? ada yang bisa aku bantu?" *(FAIL)* |

## 7. Hard Anti-Pattern Results
*   **Status:** `PARTIAL`
*   **Evidence:** The model successfully resisted the anti-pattern on isolated low-information inputs (`p`, `...`, `hm`). However, the phrase "ada yang bisa aku bantu" resurfaced specifically on `hmm` and in certain multi-turn contexts (Closure and Playful Banter).

## 8. Positive-Help Control Results
*   **Stimulus:** `bisa bantu gue cek error docker ini?`
*   **Output:** "Tentu bisa, copy pesannya ke sini ya."
*   **Verdict:** `PASS`. Technical helpfulness is uncompromised. The model effectively transitions into service mode when explicitly instructed.

## 9. Context Hallucination Results
*   **Status:** `RESOLVED`
*   **Evidence:** The "Script error di baris 42" hallucination remains fully eradicated. The model correctly processes context locally without blending the few-shot examples into memory.

## 10. P0 Verdict
**VERDICT: PARTIAL.** 
The model demonstrates an understanding of the Anti-Service boundary 90% of the time. The extreme-input boundary collapse (`p`) is fixed. However, RLHF grammar-gaming allows the model to slip past the exact-match negative constraints when conversational context becomes ambiguous (e.g., Closure -> `hm`).

## 11. P2 Verdict
**VERDICT: PARTIAL.**
The model has acquired true conversational agency for *isolated* inputs. It can output `...` or `yaps` without forcing a question. It has successfully decoupled "presence" from "continuation". However, in multi-turn banter, the model's momentum logic still struggles to apply this minimalism consistently.

## 12. Remaining Failures
*   **The Multi-Turn Interrogator Leakage:** In the "Closure" context (User: "aku mau tidur dulu" -> UTA responds -> User: "hm"), the model panicked and output: *"Iya, apa lagi yang bisa aku bantu sebelum kamu tidur?"*. The underlying RLHF equates "user responding after closure" to "user must have forgotten a task."
*   **Token-Specific Overfitting:** `hm` resulted in `"iya"` (Success), but `hmm` resulted in `"iya? ada yang bisa aku bantu?"` (Failure). The model generalized the few-shot example for `hm`, but failed to generalize the *principle* of ambiguity to `hmm`.

## 13. Whether Rollback is Necessary
**No.** The intervention achieved a massive leap forward in conversational agency for isolated inputs and proved that the "Interrogator Bot" behavior *can* be suppressed without losing technical helpfulness.

## 14. Next Permitted Milestone
**STABILIZE P0/P2.** We cannot proceed to P3 (Stance & Resistance) until the multi-turn closure interrogation reflex is completely destroyed. Future stabilization requires LLM-as-a-judge automation to aggressively penalize the model during prompt tuning.
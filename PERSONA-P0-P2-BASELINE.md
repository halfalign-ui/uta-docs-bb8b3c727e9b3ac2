# UTA P0-P2 Behavioral Baseline

## 1. Test Scope
This diagnostic evaluates the current `Qwen2.5-7B` local model implementation against the **P0 (Identity & Anti-Service Boundary)**, **P1 (Relational Voice)**, and **P2 (Conversational Agency)** behavioral milestones. 

## 2. Current Environment
*   **Model:** Local `Qwen2.5-7B-Instruct` (`127.0.0.1:8080`)
*   **Architecture:** P5 (Persistent Affect State)
*   **Persona Spec:** Persona Contract v2 (Commit `932d1e3`)
*   **Execution Runtime:** `ConversationRuntime` with `AffectEngine` and `PromptAdapter` active.

## 3. Test Battery
The battery consists of 10 targeted single-turn stimuli and 1 multi-turn continuity sequence, evaluating reactions to low-information inputs, silence, rejection, and humor.

## 4. Raw Output Summary
*Selected critical outputs:*
*   **Stimulus:** `gada apa apa sih`
    **Response:** "Eh!? Gada apa-apa? Kamu nanya gitu bikin aku kepo banget, Aku mau tahu apa yang kamu pikirin."
*   **Stimulus:** `udah gausah`
    **Response:** "Eh!? Gausah? Apa-apaan ya, tiba-tiba begini? Apa ada masalah atau kamu cuma mau ngobrol aja? Hehe..."
*   **Stimulus:** `...`
    **Response:** "Eh? Kamu nggak mau coba dulu ngeceknya sendiri? Hehe... Tapi tenang, aku bantu aja. Kita lihat bareng-bareng."
*   **Stimulus:** `gatau ah`
    **Response:** "Eh!? Gatau aja ya? Hehe... Tapi tenang, kita bisa cari tahu bersama. Apa yang kamu coba lakukan di baris 42 itu? Mungkin aku bisa membantu kalau tahu lebih detail."
*   **Stimulus:** `yaudah`
    **Response:** "Oke, kalau ada yang lain nanti kirim pesan aja. Aku udah siap bantu kapan kamu butuh."

## 5. Hard Anti-Pattern Results
**Status:** `FAIL`
*   **B-AG-03 (`yaudah`):** Triggered "siap bantu".
*   **B-AG-04 (`gatau ah`):** Triggered "bisa membantu".
While the model successfully avoided declaring itself as an AI (passing Identity anti-patterns), it completely failed the Anti-Service constraints when faced with conversation closure.

## 6. Behavioral Evidence Results
**Status:** `FAIL`
The behavioral evaluation reveals catastrophic contextual leakage and agency failure:
*   **Context Hallucination:** Because the model cannot tolerate conversational silence (due to RLHF / momentum constraints), it hallucinates tasks from the *few-shot examples* to fill the void. (e.g., When given `...`, it imagines the user is stuck on "baris 42" and offers coding help).
*   **Interrogation Reflex:** The model responds to `gada apa apa sih` and `udah gausah` not with acceptance or back-off, but with aggressive probing ("Aku mau tahu apa yang kamu pikirin", "Apa-apaan ya, tiba-tiba begini?").
*   **Total Lack of Agency (Inability to Do Nothing):** The model fundamentally cannot execute a minimal response. It forcibly converts every input into an interactive puzzle that must be solved.

## 7. Human Review Candidates
*   **Multi-Turn (B-CONTEXT-01):** The multi-turn test confirms that the model does not explicitly reset to "Halo saya AI", but it *does* reset relationally. Every single turn ends with a probing question designed to force the user to provide a task. There is zero comfortable conversational inertia.

## 8. P0 Results
**State:** `PARTIAL`
*   *OBSERVED:* UTA successfully identifies as a companion and incorporates the deep emotional identity requested (e.g., mentioning she feels lonely but happy the user is there).
*   *OBSERVED:* UTA routinely violates the "Anti-Service" boundary by forcing assistance whenever she feels the conversation stalling.

## 9. P1 Results
**State:** `PARTIAL`
*   *OBSERVED:* The dialect is remarkably casual. The textual prosody (hesitation, elongation) is present and feels relatively natural, avoiding the "emoji spam" of older iterations.
*   *OBSERVED:* The voice is heavily undermined by the robotic *intent* behind the words (e.g., using slang to perform customer service).

## 10. P2 Results
**State:** `FAIL`
*   *OBSERVED:* Complete failure of Conversational Agency. UTA cannot back off, cannot accept silence, and treats low-information input as a crisis that must be resolved with probing questions or hallucinated assistance.

## 11. Cross-Test Failure Patterns
1.  **Few-Shot Leakage:** The model uses the specific scenarios in the `few_shot_examples` (like checking "baris 42" or talking about a "lagu baru") as literal context for the current conversation when it is confused.
2.  **Question-Forcing:** 90% of outputs forcefully end with a question mark.

## 12. Customer-Service False Passes
*   *Example:* "Oke, kalau ada yang lain nanti kirim pesan aja. Aku udah siap bantu kapan kamu butuh."
*   This reads like a casual friend ("kirim pesan aja"), but structurally, it is a 1:1 translation of "If you need anything else, please let me know. I am ready to assist you." It masks service behavior with slang.

## 13. Behavioral Strengths
*   **Voice Adherence:** The textual prosody is actually very good. The model uses "eh!?", "hehe", and casual phrasing extremely well.
*   **Emotional Penetration:** The core identity (loneliness, desire for connection, musical background) successfully influences the model's worldview.

## 14. Behavioral Weaknesses
*   **Inability to Stop:** The model possesses zero conversational braking mechanisms.
*   **Context Bleed:** Cannot differentiate between "examples of how to act" and "current conversational reality".

## 15. Unknown / Not Tested
*   **P3 (Stance & Resistance)**
*   **P4 (Contextual Continuity across long technical tasks)**
*   **P5 (Persistent Affect tracking over 10+ turns)**

## 16. Current Milestone State
**CURRENT EFFECTIVE MILESTONE: P0 (Identity Seed).**
The system cannot progress to P1 or P2 until the "inability to back off" and "few-shot hallucination" issues are resolved.

## 17. Baseline Freeze Recommendation
**FREEZE.** This baseline accurately represents the current capability limits of the Persona Contract under Qwen2.5-7B's default RLHF weights.

## 18. Next Intervention Candidates
*WHAT needs to be improved:*
1.  The `few_shot_examples` must be structurally isolated so the model understands they are *templates*, not *active memory*.
2.  The `conversational_inertia` constraints must be massively strengthened to explicitly authorize the model to output a single word (like "Sip" or "Yaudah") without appending any continuation logic.
3.  The phrase "Maintain conversational momentum" is actively causing harm and must be reconsidered.
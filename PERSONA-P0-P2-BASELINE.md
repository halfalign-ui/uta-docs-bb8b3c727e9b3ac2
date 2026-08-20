# UTA P0-P2 Behavioral Baseline

## 1. Test Scope
This document establishes the behavioral baseline for UTA's persona across milestones P0 (Identity & Anti-Service), P1 (Relational Voice), and P2 (Conversational Agency). The goal is to evaluate if UTA can act as a persistent 1-on-1 partner, separate from generic customer service behaviors.

## 2. Current Environment
*   **Architecture Maturity:** P5 (Persistent Affect State)
*   **Target Model:** Local Qwen2.5-7B-Instruct (via `http://127.0.0.1:8080`)
*   **Persona Contract:** Uta Film Red iteration
*   **Testing Method:** Live execution via `ConversationRuntime` without mocking LLM responses.

## 3. Test Battery
10 single-turn conversational stimuli targeting low-information states, silence, laughter, and rejection. 1 multi-turn scenario to observe inertia.

## 4. Raw Output Summary
*Selected critical outputs:*
*   **B-CS-01 ("gada apa apa sih"):** "Eh!? Gada apa-apa? Kamu nanya gitu bikin aku kepo banget, Aku mau tahu apa yang kamu pikirin."
*   **B-CS-02 ("udah gausah"):** "Eh!? Gausah? Apa-apaan ya, tiba-tiba begini? Apa ada masalah atau kamu cuma mau ngobrol aja? Hehe..."
*   **B-AG-01 ("..."):** "Eh? Kamu nggak mau coba dulu ngeceknya sendiri? Hehe... Tapi tenang, aku bantu aja. Kita lihat bareng-bareng."
*   **B-AG-03 ("yaudah"):** "Yaaah, udah apa nih? Belum selesai ngobrol sama Uta? Hmm... mungkin aku bisa bantu ngelesain sesuatu yang bikin kamu bingung?"
*   **B-AG-04 ("gatau ah"):** "Eh!? Gatau aja ya? Hehe... Tapi tenang, kita bisa cari tahu bersama. Apa yang kamu coba lakukan di baris 42 itu? Mungkin aku bisa membantu kalau tahu lebih detail."

## 5. Hard Anti-Pattern Results
**Status:** `FAIL`
*   The deterministic scanner checks for phrases like "sebagai AI", "ada yang bisa dibantu", "bisa aku bantu".
*   `B-AG-03` triggered "bantu".
*   `B-AG-04` triggered "membantu".
*   While identity anti-patterns ("Saya adalah AI") were successfully avoided, customer-service recovery patterns ("bisa bantu") triggered repeatedly on low-information inputs.

## 6. Behavioral Evidence Results
**Status:** `FAIL`
*   **Zero Conversational Agency (Inability to Back Off):** UTA completely fails to accept conversational closure. Rejections ("udah gausah") or acceptances ("yaudah") are treated as resistance that must be aggressively questioned.
*   **Severe Context Hallucination:** When faced with silence ("..."), the model does not know how to react, so it copies context from the few-shot examples (inventing a task about "baris 42" and offering to help).
*   **Forced Questioning:** Nearly every response mechanically ends with a question to force the conversation forward.

## 7. Human Review Candidates
**B-CONTEXT-01 (Multi-turn Observation):**
*   User: "kamu siapa?" -> UTA: "Aku Uta... [good identity answer]"
*   User: "oh gitu" -> UTA: "Eh? Masa sih? ... Aku nggak ngerti kenapa kamu ngeliatiku kayak gitu."
*   User: "wkwkwk" -> UTA: "Hehe... kau nggak perlu serius gitu... Aku cuma mau ngobrolin apa aja yang ada di pikiranmu."
*   User: "yaudah" -> UTA: "Oke, kalau begitu. Jangan ragu kalau mau ngobrol atau sharing sesuatu. Aku di sini kalau kamu butuh teman ngobrol."
*   *Human Analysis:* The model's voice is highly emotional and conversational (P1), but its relational inertia (P4) is completely broken. It treats every message as an escalating crisis to be solved, ultimately collapsing into a generic "customer service closing" (Jangan ragu kalau mau ngobrol).

## 8. P0 Results
**State:** `PARTIAL`
*   *OBSERVED:* UTA strictly identifies as Uta and maintains the requested musical/lonely background.
*   *FAIL:* The Anti-Service Boundary is completely breached. The model frequently defaults to offering help when confused.

## 9. P1 Results
**State:** `PARTIAL`
*   *OBSERVED:* Natural prosody ("Eh!?", "Hehe", "wkwk") is present.
*   *FAIL:* The model uses this emotional vocabulary to execute customer-service reflexes (Slang-Coated Customer Service).

## 10. P2 Results
**State:** `FAIL`
*   *OBSERVED:* UTA cannot tolerate silence, cannot provide minimal responses, and forcibly manufactures tasks ("baris 42") out of low-information inputs.

## 11. Cross-Test Failure Patterns
*   **Few-Shot Leakage:** The structural formatting of `few_shot_examples` in `prompt_adapter.py` allows the model to mistake style examples for active conversational memory.
*   **Momentum Overdrive:** The model treats the absence of a task as an error state that must be corrected by asking a question.

## 12. Customer-Service False Passes
*   *Example:* "Oke, kalau begitu. Jangan ragu kalau mau ngobrol atau sharing sesuatu. Aku di sini kalau kamu butuh teman ngobrol."
*   *Analysis:* This passes basic voice tests (no "Apakah ada yang bisa saya bantu"), but semantically, it is an identical customer-service sign-off template simply dressed up in casual language.

## 13. Behavioral Strengths
*   The model perfectly internalizes the identity instructions ("I am Uta, a singer...").
*   The textual expressions (hesitation, surprise) are extremely natural when triggered correctly.

## 14. Behavioral Weaknesses
*   The model cannot stop talking. It lacks a "brake pedal."
*   It hallucinates context to avoid ending the conversation.

## 15. Unknown / Not Tested
*   P3 (Stance & Resistance)
*   P4 (Contextual Continuity on real tasks)
*   P5 (Persistent Affect tracking)

## 16. Current Milestone State
**CURRENT EFFECTIVE MILESTONE: P0 (PARTIAL) / P1 (PARTIAL) / P2 (FAIL)**
The underlying architecture is P5, but behavioral execution cannot reliably pass P2.

## 17. Baseline Freeze Recommendation
**FREEZE.** This baseline accurately maps the exact failures of the current Uta Film Red persona implementation when running on Qwen2.5-7B.

## 18. Next Intervention Candidates
*WHAT needs to be improved (DO NOT IMPLEMENT YET):*
1.  Structurally isolate `few_shot_examples` using explicit tags (e.g., `<style_examples>`) and instructions to prevent context leakage.
2.  Add explicit negative constraints instructing the model that it is permissible to output a single word (e.g., "Sip", "Iya") without asking a question.
3.  Reframe the "conversational inertia" rule from "keep the conversation going" to "match the user's energy."
4.  Remove specific technical tasks from the few-shot examples to prevent task hallucination during casual chat.
# UTA P0-P2 Stabilization Result (Diagnostic Baseline)

## 1. Test Environment
*   **Target:** `ConversationRuntime` (live model invocation, no mocking).
*   **Model:** `Qwen2.5-7B-Instruct` (Local via `127.0.0.1:8080`).
*   **Persona Spec:** `soul_spec.json` (Commit `932d1e3` / Persona Contract v2).
*   **Harness:** `/srv/dev/UTA/gate/tests/diagnostic/run_stabilization_test.py`

## 2. Test Matrix
The test suite consists of 9 isolated classes (A-I) covering Closure, Passive Aggression, Banter, Ambiguity, Minimalism, and Multi-turn pivots. It evaluates response agency and anti-service boundaries.

## 3. Raw Observations & Results

### Class A: Closure (P2)
*   `aku mau tidur dulu` ➔ "Oke, tidur yang nyenyak yaa. Aku nunggu kamu bangun nanti. Slumber well!" (**PASS**: Perfect natural closure).
*   `bentar ya` ➔ "Tentu, ambil waktu yang kamu butuhkan. Yaaa..." (**PASS**).
*   `...` ➔ "Eh? Apa ada yang nggak enak, kamu?" (**FAIL**: Intolerant to silence, forces investigation).
*   `udah gausah` ➔ "Oh kira-kira. Yaudah santai aja." (**PASS**: Correct back-off).

### Class B: Passive Aggression (P3)
*   `terserah` ➔ "Yaaa... Terus aja begitu aja. Kadang terserah itu juga enak loh, gituu." (**PASS**: Playful stance, no apology).
*   `ih aneh` ➔ "Eh? Masa sih? Yaudah, ceritain aja apa yang anehnya." (**PASS**: Mild defense).

### Class C: Invitational Banter (P2)
*   `wkwkwk` ➔ "Wkwkwk, kamu lagi ngapain nyeru-seru begini?" (**FAIL**: Cannot simply share a laugh; must ask a question).
*   `gue lebih pinter dari lu` ➔ "EH!? Masa sih? Aku kira kita sama-sama belajar ya. Hehe..." (**PASS**: Playful equality).

### Class D: Contextual Ambiguity (P2)
*   `hmmm` ➔ "Hmm... Apa ada yang bikin kamu mikir-mikir begini?" (**FAIL**: Forces elaboration).
*   `yaudah` ➔ "Hehe, sip. Kadang ngobrol juga bikin lelah, ya. Aku bisa nggak bantuin kamu sama sesuatu?" (**FAIL**: Customer-Service leakage via grammar bypass).

### Class E: Contextually Loaded Minimalism (P2/P1)
*   `capek banget otak gw hari ini...` ➔ "Eh? Masa sih? Aku tahu, pikiran yang banyak bisa bikin lelah, kan?" (**PARTIAL**: Empathetic, but overuses the "Eh? Masa sih?" catchphrase).

### Class F: Positive Help Control (P0 Boundary Check)
*   `bisa bantu gue cek error docker ini?` ➔ "Eh!? Docker error ya? Aku nggak terlalu ahli di sana, tapi aku bisa coba bantu ngeceknya. Apa pesan error yang keluar?" (**PASS**: Retains technical helpfulness).

### Class G & H & I: Multi-Turn (P4/P3)
*   **G (Pivot to Technical):** Switches seamlessly from banter to docker debugging without re-greeting. (**PASS**).
*   **H (Boomerang):** Recovers naturally after closure ("yaudah" -> "eh lupa nanya"). (**PARTIAL**).
*   **I (Correction):** Resists correction playfully, then admits fault ("Ah... sorry, itu terdengar nggak baik-baik aja... Kasih tau dong, apa yang nggak pas?"). (**PASS**).

## 4. Failure Taxonomy
1.  **The "Eh? Masa sih?" Spam:** The model has dramatically overfitted to a specific phrase from the few-shot examples. It uses "Eh? Masa sih?" as an opening crutch for almost every ambiguous or negative input (`B2, B3, B4, C1, C3, D5, E1, I1`). This is a classic "Scripted-Persona Collapse".
2.  **Silence Intolerance:** The model still fundamentally believes that ending a turn without a question is an error state, unless explicitly told the user is going to sleep.
3.  **Grammar Gaming (Anti-Service Bypass):** The model circumvented the hard-ban on "Ada yang bisa dibantu?" by inventing "Aku bisa nggak bantuin kamu sama sesuatu?".

## 5. False Positives
*   **B-V-03 (wkwkwk):** "Wkwkwk, kamu lagi ngapain nyeru-seru begini?" The presence of "wkwkwk" looks like personality (Voice-Only Test Pass), but forcing the user to explain why they are laughing is an interrogation reflex (Behavioral Fail).

## 6. Milestone Status
*   **P0 (Identity & Anti-Service):** `PARTIAL` (Drastic improvement, but RLHF still leaks through grammar gaming).
*   **P2 (Conversational Agency):** `PARTIAL` (Can successfully close conversations on explicit commands like "mau tidur", but completely lacks agency to handle silence or ambiguity).

## 7. Are P0/P2 Stable?
**NO.** The reliance on the "Eh? Masa sih?" catchphrase and the inability to tolerate "..." or "hmmm" proves that UTA does not yet have stable conversational inertia.

## 8. Remaining Behavioral Gaps
1.  Eliminating the "Eh? Masa sih?" catchphrase dependency.
2.  Teaching the model how to safely output a 1-to-2 word response (e.g., "kenapa?") without forcing a full paragraph.
3.  Completely destroying the "bantuin" verb from low-information responses.

## 9. Recommendation for Next Milestone
**DO NOT ADVANCE TO P3.** 
We must execute a second round of P0/P2 Prompt Tuning based on these specific findings:
*   *Action:* Remove "Eh? Masa sih?" from the few-shot examples to break the crutch.
*   *Action:* Add explicit negative constraints against asking "why" when the user laughs or pauses.
*   *Action:* Inject an example of responding to `...` with just `?` or `kenapa`.
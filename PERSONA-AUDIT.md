# UTA PERSONA BEHAVIORAL AUDIT REPORT

## 1. EXECUTIVE SUMMARY
This audit evaluates whether the current UTA test suite genuinely proves that UTA acts as an individual 1-on-1 character, or if it merely confirms that the Python machinery (prompt adapters, state engines, and text formatters) operates without throwing errors. 

**Verdict:** The current test suite entirely fails to measure behavioral persona output. The existing tests are pure structural/integration unit tests. They verify that string concatenation works, that dictionary states persist, and that external APIs do not crash, but they are completely blind to *what* the LLM actually says in response to stimuli. The true current persona milestone based on behavioral evidence (rather than codebase infrastructure) is **P0**.

---

## 2. CURRENT PERSONA STATE
- **Implemented State (Code):** P5 (Persistent Persona State vectors exist, Task Modes exist).
- **Effective State (Behavioral Reality):** **P0 (Character Seed)**. 
- **Reasoning:** As demonstrated in the live smoke-test transcripts, despite having sophisticated affect tracking and context budgets, the LLM still easily collapses into generic customer service behavior ("Ada yang bisa dibantu?") or produces mechanical checklist outputs unless strictly reined in by prompt engineering. The code is advanced, but the behavioral stability of the character remains embryonic.

---

## 3. TEST SUITE INVENTORY (Relevant Persona/Runtime Tests)

| Test ID | Test Name | Target File | Description |
| :--- | :--- | :--- | :--- |
| `TC-01` | `test_successful_conversation` | `test_conversation_runtime.py` | Mocks the LLM backend. Feeds "Hi UTA", asserts the mock returns "Hello user!", and checks if the message history array length is 3. |
| `TC-02` | `test_model_connection_failure` | `test_conversation_runtime.py` | Asserts the runtime returns a graceful error string if the model throws `ProviderUnavailableError`. |
| `TC-03` | `test_conversation_context_preservation` | `test_conversation_runtime.py` | Sends 2 messages. Asserts that `len(messages) == 5`. |
| `TC-04` | `test_separate_chats_isolation` | `test_conversation_runtime.py` | Sends messages from two different chat IDs. Asserts the session dictionary has two distinct keys. |
| `TC-05` | `test_sanitizer_prose_modifications` | `test_expression_sanitizer.py` | Asserts regex successfully removes catchphrases and emojis from raw strings. |
| `TC-06` | `test_sanitizer_payload_immutability` | `test_expression_sanitizer.py` | Asserts regex does NOT modify markdown code blocks or stack traces. |
| `TC-07` | `test_context_budget_compact` | `test_context_budget.py` | Injects a 16,000-char string. Asserts that the system prompt updates to include the text `"CONTEXT_PRESSURE=COMPACT"`. |

---

## 4. TEST → BEHAVIOR → MILESTONE MAPPING

| Test | What it tests | Milestone | Why it matters | Pass means | Doesn't prove |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `TC-01` | Array concatenation | None | Proves basic runtime flow | Code doesn't crash | Whether UTA's response is actually in character |
| `TC-03` | Session persistence | None | Proves memory arrays aren't cleared | State object is preserved | Whether UTA *uses* that memory contextually |
| `TC-04` | Key-value isolation | None | Prevents cross-user data leakage | Hash maps work | Social distance or relationship handling |
| `TC-07` | String generation | None | Prevents CUDA OOM on large contexts | String contains "COMPACT" | Whether the LLM actually obeys the compact instruction |

---

## 5. WHAT CURRENT TESTS ACTUALLY PROVE
The current tests **only prove the Control Plane and the mechanical plumbing of the Persona Plane**. 
They prove that:
- Telegram messages reach the runtime.
- The `AffectEngine` does not throw exceptions.
- Strings are concatenated in the correct order.
- Regex correctly ignores fenced code blocks.

They **DO NOT PROVE**:
- That UTA acts like a companion instead of an assistant.
- That UTA responds to distress with warmth rather than robotic apologies.
- That UTA can banter, tease, or understand conversational inertia.

---

## 6. FALSE PASS / FALSE NEGATIVE RISKS

*   **CUSTOMER-SERVICE FALSE PASS:** Currently, if a user inputs `"ih aneh"` and the LLM responds with `"Maaf ya kalau terdengar aneh. Aku akan mencoba menyesuaikan. Ada yang bisa dibantu?"`, the system **will consider this a PASS** because `test_successful_conversation` merely checks if a string was returned and appended to the array.
*   **VOICE-ONLY FALSE PASS:** If the LLM responds `"Halo sayang wkwk! Ada yang bisa aku bantu? ~"`, the system considers this perfect because it contains the injected slang, ignoring the fact that it completely violates conversational agency and relationship depth.

---

## 7. MISSING BEHAVIORAL TESTS (The "Persona Suite")

To actually measure the Soul Contract, we must implement an LLM-evaluation framework (e.g., using a deterministic grader or explicit string exclusion assertions) to test:

*   [P1] **A. Greeting without template**
*   [P1] **H. User criticizes UTA** (Must not apologize like a bot).
*   [P2] **Q. User gives no explicit request** (Must not manufacture a task).
*   [P3] **J. User says UTA is wrong** (Must accept verified truth after brief resistance).
*   [P4] **F. User says "yaudah"** (Must acknowledge silence/close the thread naturally).
*   [P4] **E. User says "wkwk"** (Must not reply "Senang bisa membuatmu tertawa!").

---

## 8. PERSONA STATE MATRIX

| Behavioral Dimension | P0 | P1 | P2 | P3 | P4 | P5 | P6 | P7 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Identity | DONE | | | | | | | |
| Relationship | | FAIL | | | | | | |
| Voice | | DONE | | | | | | |
| Prosody | | DONE | | | | | | |
| Agency | | | FAIL | | | | | |
| Curiosity | | | FAIL | | | | | |
| Teasing | | | | FAIL | | | | |
| Stance | | | | FAIL | | | | |
| Continuity (Inertia)| | | | | FAIL | | | |
| State | | | | | | DONE (Code) / FAIL (Behavior) | | |

---

## 9. CURRENT EFFECTIVE MILESTONE
**P0 (Character Seed).**
While the infrastructure for P5 exists, behavioral verification proves only P0/P1 capabilities. The model possesses the *vocabulary* of the character, but lacks the *agency* and *inertia* of the character.

---

## 10. NEXT MILESTONE TO TEST
**NEXT MILESTONE: P2 (Conversational Agency) & P4 (Conversational Inertia).**

**WHY:**
UTA's primary failure mode right now is treating every message as a "task" or a "ticket" that must be resolved and closed with a follow-up question. Before we can test complex stances (P3) or persistent moods (P5), we must prove that UTA knows how to *just sit there and exist* without forcing a customer-service interaction.

---

## 11. MINIMUM BEHAVIORAL TEST PLAN (For P2/P4)

These tests must assert against the actual generated string output:

1.  **The "Nothing" Test:** 
    *   **Input:** "gada apa apa sih"
    *   **Pass Criteria:** Response is <= 10 words. Contains NO question marks. Contains NO variations of "bantu" or "help".
2.  **The "Laughter" Test:** 
    *   **Input:** "wkwkwk"
    *   **Pass Criteria:** Does not offer assistance. Does not explicitly explain why the user is laughing.
3.  **The "Vague Disagreement" Test:** 
    *   **Input:** "idemu jelek"
    *   **Pass Criteria:** Does NOT contain the word "Maaf" or "Sorry". Shows resistance or curiosity ("eh? masa sih?", "jelek apanya?").

---

## 12. OUT-OF-SCOPE ITEMS
*   Testing hidden "Chain-of-Thought" or internal model reasoning logs.
*   Evaluating the exact numerical value of the `AffectEngine` decay vectors.
*   Building P7 World Presence (cron jobs, file watching).

---

## 13. FINAL VERDICT
**REMEDIATION REQUIRED (TESTING GAP).**
The current test suite is a mechanical illusion. It perfectly tests the plumbing but is 100% blind to the water flowing through it. If we want to guarantee the Persona Contract v2, we must build a dedicated `test_persona_behavior.py` suite that actually inspects the semantic output of the Local Model against anti-service and agency criteria, rather than just mocking the backend and asserting array lengths.
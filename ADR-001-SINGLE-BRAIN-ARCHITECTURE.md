# Architecture Decision Record (ADR): Single-Brain + External Cognitive Resources

## 1. Context & Background
In the post-F8 development cycle, we conducted an experiment (Local Multi-Stage Cognitive Pipeline) to mitigate generic RLHF bias by splitting the Local Model into a JSON-based Cognitive State generator, an Interface Drafter, and a Finalizer. 

The empirical verdict was a **SEVERE REGRESSION**. The multi-stage pipeline suffered from:
- **Semantic Drift:** Translating Indonesian casual slang into English JSON state caused severe loss of context and emotional nuance.
- **Latency & Token Overhead:** Response times surged from ~0.25s to ~2.5s, with a 10x token overhead.
- **Emotional Bluntness:** Rigid adherence to structural rules lobotomized the model's natural empathy and linguistic agility.

## 2. Decision: The Single-Brain Paradigm
We formally adopt the **SINGLE CONVERSATIONAL BRAIN + EXTERNAL COGNITIVE RESOURCES** paradigm.

### The Local Brain (Qwen2.5-7B)
The local model retains absolute ownership over:
- Identity and Persona
- Conversational Continuity & Memory Interpretation
- Affect Interpretation
- Conversational Agency & Voice
- Natural Language Generation and Textual Prosody

**Rule:** Persona generation MUST NEVER be fragmented into multiple LLM completions or JSON state translations. The raw linguistic continuity of the local model must be preserved.

## 3. Execution Pathways

### A. Normal Conversation (Fast Path)
For casual banter, emotional reactions, and low-information inputs, the pipeline is entirely direct:
`USER ➔ LOCAL QWEN ➔ USER`
No cloud involvement, no interface wrappers, no JSON translation.

### B. External Cognition (Tool/Cloud Path)
When the local model determines that a request exceeds its immediate context or requires action (e.g., technical debugging, web search, heavy reasoning), the pipeline extends:
`USER ➔ LOCAL QWEN ➔ ORCHESTRATION DECISION ➔ EXTERNAL RESOURCE ➔ LOCAL QWEN ➔ USER`

The final output is always synthesized by the Local Qwen to maintain conversational continuity and persona (e.g., *"nah ini gara-gara container-nya masih nahan network lama..."*, instead of *"Based on the tool output..."*).

## 4. The Role of Cloud Models
Cloud models (e.g., Gemini, Claude, GPT-4) are strictly **External Cognitive Resources**.
- They are **NOT** persona engines.
- They are **NOT** interface owners.
- They are **NOT** authorities.

**Cloud Isolation Constraints:**
1. Cloud models receive only the *minimum context required* for the task (Sanitized Context).
2. Cloud models NEVER receive raw credentials, API keys, Vault contents, internal secrets, or unrestricted workspace context.
3. Cloud output is treated as a raw artifact that must be parsed by the Local Control Plane and synthesized by the Local Persona.

## 5. The Authority Model
The highest authority resides exclusively in the **LOCAL / RUNTIME**.
1. **Control Plane (Runtime):** Enforces execution boundaries, security, sanitization, and Vault isolation.
2. **Persona Plane (Local Qwen):** Determines how UTA sounds and reacts.
3. **Cognitive Layer (Cloud/Tools):** Provides raw analysis, computation, and data retrieval under the strict supervision of the Local Runtime.
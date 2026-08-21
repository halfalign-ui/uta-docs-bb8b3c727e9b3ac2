# ADR-001 Implementation Audit: Single-Brain Architecture

## 1. Compatible Components (Ready as-is)
- **LocalModelProvider:** Already implements the OpenAI-compatible tool calling specification. Correctly parses `tool_calls` into structured `ToolCall` objects and yields `FinishReason.TOOL_CALLS`.
- **ModelRequest / Message Schema:** Naturally supports `ToolDefinition`, `role="tool"`, `tool_calls`, and `tool_call_id`.
- **PolicyEngine & Gate:** Already designed as the strict execution authority. The provider layer currently cannot execute tools, aligning perfectly with the requirement that the Local Qwen only *requests* tools while the Runtime *executes* them.

## 2. Components Requiring Extension
- **ConversationRuntime:** Currently only appends `response.text`. It must be extended to support a multi-step execution loop: 
  `Qwen -> yields TOOL_CALLS -> Runtime executes tool -> appends Message(role="tool") -> Qwen -> Final response`.
- **Context Sanitizer (New):** A deterministic regex/AST-based scrubber is needed to strip API keys, Bearer tokens, and Vault paths from context *before* it is ever passed to an external cloud provider.

## 3. Components That MUST NOT Be Touched
- **AffectEngine / Persona Prompt:** The behavioral identity must remain untouched. The Single-Brain handles this fluidly.
- **Agent Runtime Execution Logic:** Tool execution must remain sandboxed inside the established `AgentLoop`/`Gate` architecture. Qwen cannot bypass this.

## 4. Implementation Gaps
- The `ConversationRuntime` lacks the `tool_executor` bindings required to handle the intermediate tool steps for casual chat interactions that suddenly pivot to technical requests.

## 5. Risks
- **Context Leakage:** If the chat history is shipped to a Cloud Tool/Model without proper sanitization, Vault secrets discussed earlier in the chat could be exposed.
- **Persona Degradation (The "Tool-Bot" Reflex):** When fed `role="tool"` messages, Qwen might forget its persona and default to *"According to the tool output..."* instead of naturally synthesizing the result as UTA.

## 6. Proposed Minimal Integration Point
We will build a `SingleBrainRuntime` wrapper around `LocalModelProvider` for the PoC. This wrapper will:
1. Define a mock `ToolDefinition` (e.g., `check_docker_error`).
2. Intercept `FinishReason.TOOL_CALLS`.
3. Pass the tool arguments to a deterministic mock function.
4. Append the result as `Message(role="tool", ...)`.
5. Call `backend.complete()` a second time for the final synthesis.
6. Verify the final response maintains the UTA persona.

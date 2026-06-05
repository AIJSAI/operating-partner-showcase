# Technical Decisions: AI Chief of Staff

This document contains excerpts from the project's architecture decision log. Each entry captures the context, the decision, and its consequences. Source code is available upon request for interview processes.

---

## D-01: The Reasoning Agent Is the Brain; the Toolbelt Only Returns Cited Data

**Status**: Accepted

**Context**: The popular pattern for "AI assistants that do things" is an autonomous orchestrator that plans, calls models, and acts in a loop. That design puts a language model on the critical path of every answer, which is exactly where unsourced claims get manufactured. For an operating partner that answers legal and financial questions, that is unacceptable.

**Decision**: Invert the usual split. The reasoning agent (Claude Code) does the planning and writing; the Chief is a passive toolbelt whose tools return only deterministic computations or retrieval results, each wrapped in a cited envelope. The Chief never "decides." It supplies grounded inputs the brain reasons over.

**Consequences**: The grounding burden lives in code that can be tested, not in a model that can hallucinate. The trade-off is that the Chief cannot act on its own; it is only as useful as the agent driving it. For a high-stakes advisory tool, that is the correct trade.

---

## D-02: No Tool May Call a Language Model (Enforced Structurally)

**Status**: Accepted

**Context**: D-01 only holds if it is actually true. Nothing in MCP stops a tool author from importing a model SDK inside a tool and returning its output as if it were sourced. A convention in a contributing guide would erode within a few PRs.

**Decision**: Make the rule a build-breaking test. A static AST scan walks every domain directory plus the server entrypoint for any import of or call into the model SDK; an anchor-file coverage assertion guards against a mistyped path silently passing. A dynamic test then invokes every registered tool under a patched client that raises if the model SDK is touched at runtime.

**Consequences**: The invariant is enforced both at import time and at call time. A new domain that tries to sneak in a model call fails CI immediately. The cost is a small amount of test machinery and the discipline of keeping all model reasoning in the agent layer.

---

## D-03: A Frozen Output Contract With Per-Claim Confidence

**Status**: Accepted

**Context**: Tools that return free-form strings give the reasoning agent no way to know how much to trust a result, and invite unsourced assertions. Advisory answers need a machine-readable, honest confidence signal.

**Decision**: Every tool returns a frozen Pydantic `GroundedEnvelope` containing `Claim` objects. Each `Claim` carries a 4-state confidence tier (a tiering pattern adapted from a production compliance system). The envelope computes a worst-tier rollup, and a finalize gate blocks any envelope that fails grounding from being presented as a grounded answer.

**Consequences**: The agent always receives both an answer and a structured confidence signal, and the weakest claim caps the trustworthiness of the whole. Immutability (frozen models) prevents downstream mutation of a graded result. The cost is that every tool must conform to the contract, which is the point.

---

## D-04: Four Sanctioned Grounding Patterns, Not One Blanket Rule

**Status**: Accepted

**Context**: "Cite your sources" is too blunt for a toolbelt where a cap-table calculation, a statute lookup, and a retrieval result all ground differently. Forcing one pattern produces either fabricated citations on pure-compute results or unticketed gaps on retrieval results.

**Decision**: Define four explicit patterns: **emit-none** (pure compute, provenance is the deterministic function), **pass-through** (retrieval returns the source verbatim), **statute-derived Claims** (legal tools attach the controlling statute and downgrade to honest-uncertainty on missing inputs), and **static output-class provenance**. Each tool declares which pattern it uses, and CI checks conformance.

**Consequences**: Grounding is honest per tool type. Legal tools never assert a favorable conclusion on unknown inputs: a worker-classification or privacy-scope tool returns "insufficient information" rather than a confident exemption. The cost is a taxonomy contributors must learn.

---

## D-05: Two MCP Surfaces (stdio + In-Process Agent SDK)

**Status**: Accepted

**Context**: Claude Code registers MCP servers over stdio (`claude mcp add --transport stdio`). But some tools need to run inside an Agent-SDK process and cannot be exposed cleanly over stdio. Forcing everything through one transport would either break those tools or leak the constraint into every caller.

**Decision**: Run two surfaces behind a shared tool registry: the FastMCP stdio server that Claude Code registers, and an in-process Agent-SDK server for the tools that require it.

**Consequences**: Both transports are supported without the caller needing to know which is which. This reflects a real MCP SDK constraint rather than a self-imposed one. The cost is maintaining two server entrypoints.

---

## D-06: Domain-Modular Registration With an Exact-Surface Drift Guard

**Status**: Accepted

**Context**: A 27-tool surface across 9 domains will drift (a rename, an accidental duplicate, a tool registered twice) unless something asserts the exact surface. Silent drift in a tool API is a real correctness and review hazard.

**Decision**: Each domain exposes a `register_<domain>_tools` function, and a CI test asserts the **exact** set of registered tool names. Adding, renaming, or removing a tool fails the build until the expected-surface list is updated deliberately.

**Consequences**: The tool surface is a reviewed, intentional artifact. New domains plug in via one registration function; the drift guard makes any surface change explicit in the diff. The cost is one more append-only list to maintain per change, a deliberate friction.

---

**Built by [James Shehan](https://jamesshehan.dev)** · TPM / Solutions Architect

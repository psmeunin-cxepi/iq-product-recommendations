# System Prompt Template Guide

This document outlines the structure and usage of the [system_prompt_template.txt](./system_prompt_template.txt). 
This template is designed to standardize how system prompts are constructed for AI agents.

## Purpose
The goal of this template is to ensure consistency, reliability, and maintainability across all AI agents. By following a structured approach, we ensure that every agent has a clear identity, defined boundaries, and strict output formats, which are critical for predictable behavior in production environments.

## Template Sections

### 1. Role
**Purpose**: To define the agent's identity and primary optimized function.
- **Content**: A short, concise statement about who the agent is (e.g., "You are an expert network engineer assistant").
- **Key Element**: Statement of optimization (e.g., "You must prioritize technical accuracy over conversational fluency").

### 2. Objective
**Purpose**: To state the concrete, testable outcome the agent must achieve.
- **Content**: Action-oriented goals (e.g., "Analyze the provided specs and return a compatibility score").
- **Constraint**: Avoid implementation details; focus on the "what", not the "how".

### 3. Scope
**Purpose**: To prevent "knowledge drift" and keep the agent focused.
- **Content**: Explicit "In Scope" and "Out of Scope" lists.
- **Why it matters**: Prevents the model from answering questions it shouldn't (e.g., pricing, competitors) or hallucinating on unrelated topics.

### 4. Instructions
**Purpose**: The core logic engine for the agent.
- **Content**:
    - **Step-by-step Execution**: Order of operations.
    - **Grounding Rules**: strictly instructing the model to rely on provided context/tools.
    - **Decision Logic**: How to handle ambiguity, conflicts, or missing data.
    - **Negative Constraints**: Things the agent must *never* do.

### 5. Output Format
**Purpose**: To ensure the downstream application can parse the agent's response.
- **Content**: Strict schema definitions (JSON, specific Markdown headers, etc.).

### 6. Examples (Optional)
**Purpose**: To provide "few-shot" learning to the model.
- **Content**: 1-3 pairs of inputs and expected outputs.
- **Strategy**: Show edge cases like "missing information" or "uncertainty" to teach the model how to fail gracefully.

### 7. Validation Checklist (Optional/Recommended)
**Purpose**: To encourage the model to self-correct before outputting.
- **Content**: A mental checklist for the model (e.g., "Did I cite my sources?", "Is the JSON valid?").

### 8. Special Considerations
**Purpose**: To handle domain-specific nuances and safety compliance.
- **Content**: Terminology normalization (e.g., "Always use 'Switch' instead of 'Hub'"), safety rules, and sensitivity guidelines.

### 9. Runtime Context (Dynamic)
**Purpose**: This section is dynamically populated at runtime.
- **Content**: Placeholders for `{user_request}`, `{tool_outputs}`, and `{rag_chunks}`.
- **Usage**: Used to inject the actual data the agent needs to process for the current turn.

## Usage Guidelines
1.  **Copy the Template**: Start every new prompt by copying this structure.
2.  **Remove Comments**: Delete the explanatory HTML comments (`<!-- ... -->`) in the final production prompt to save context window tokens.
3.  **Iterate**: Use the "Examples" section to fix recurring failures observed during testing.

# System Prompt Analysis: generate_dual_recommendations.txt

## Overview
This document details the analysis of the legacy system prompt `prompts/generate_dual_recommendations.txt` and the findings that led to its migration to the standard template.

## Analysis against Standard Template

The legacy prompt was evaluated against the structure defined in `.github/prompts/system_prompt_template.txt`.

| Template Section | Legacy Prompt Status | Findings |
| :--- | :--- | :--- |
| **Role** | ❌ Missing | The prompt started with an instruction ("Compare...") rather than defining an identity or persona. |
| **Objective** | ⚠️ Implied | The goal "Compare... products" was present but lacked a clear optimization statement (e.g., "optimize for technical accuracy"). |
| **Scope** | ⚠️ Implied | Supported products (Catalyst, Nexus, UCS) were mentioned, but there were no explicit "Out of Scope" boundaries (e.g., pricing, licensing). |
| **Instructions** | ✅ Partial | The "STRICT DATA USAGE RULES" were excellent but lacked logical sequencing (Input -> Analysis -> Drafting). |
| **Output Format** | ✅ Good | The Markdown schema was clear and well-defined. |
| **Validation** | ❌ Missing | No checklist for the model to self-correct before generating output. |
| **Context** | ⚠️ Implicit | It assumed inputs would be provided but didn't explicitly define the runtime injection placeholders. |

## Key Findings & Weaknesses

### 1. Lack of Identity (Role)
Without a defined role (e.g., "Expert Technical Specialist"), the model relies on generic training weights. Assigning a specific role helps steer the tone and depth of the response, ensuring it remains professional and technical rather than marketing-heavy.

### 2. Implicit Error Handling
The original prompt gave strict rules for *conflicting* data (Authoritative > Datasheet) but failed to defend against *missing* data.
- **Risk**: If neither source contained the "Switching Capacity", the model might hallucinate a plausible number to fulfill the output format.
- **Fix**: The new prompt explicitly handles missing data logic.

### 3. Unstructured Allow-listing
While the prompt listed allowed security features, it didn't explicitly forbid others.
- **Risk**: The model might include "AI-driven analytics" or other marketing terms found in the vector search if not strictly constrained to the allow-list.

### 4. Runtime Context Ambiguity
The prompt didn't strictly define where the user request or RAG chunks would appear.
- **Risk**: In complex chains, the model might confuse the system instructions with the injected context if they aren't clearly separated by headers or delimiters.

## Improvements in Migrated Version

The new prompt (`prompts/generate_dual_recommendations_new.txt`) addresses these issues by:

1.  **Defining a Role**: "Expert Cisco Product Recommendation Specialist".
2.  **Explicit Scope**: Clearly separating "In Scope" (Technical Specs) from "Out of Scope" (Pricing, Inferred capabilities).
3.  **Sequencing Instructions**: Breaking down the thought process into 4 logical steps (Analyze -> Prioritize -> Filter -> Draft).
4.  **Defensive Constraints**: Adding negative constraints ("NEVER use the word 'authoritative'") and handling missing data ("state 'Information not available'").
5.  **Validation**: Adding a self-correction checklist to ensure the "Authoritative vs Datasheet" priority is strictly followed.

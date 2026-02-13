---
title: "Vibe Coding as a Senior Developer: Constrain the AI, Not the Prompt"
date: 2026-02-13 14:00:00 +1100
categories: [Thinking]
tags: [AI, vibe coding, software engineering, architecture, TypeScript, constraints]
---

When you ask an AI to "fix a bug" or "add a feature," its default behavior is to rewrite the world to suit that one specific task. It lacks "Project Memory"—it doesn't know *why* you structured the code that way three months ago, so it happily destroys your architecture to save three lines of code today. This is what creates "garbage code" and spaghetti refactors.

To use Vibe Coding as a Senior Developer, you must invert the usual workflow. You stop using AI as a "Collaborator" (who has opinions) and start using it as a "Filler" (who follows strict constraints).

Here is the refined, "hard-thought" strategy for robust Vibe Coding.

## 1. The "WET over DRY" Rule (The Anti-Refactor)

The single biggest cause of AI-generated garbage is **premature abstraction**.

* **The Trap:** You ask for a feature. The AI notices similar code elsewhere. It decides to be "helpful" by creating a generic `handleDataProcessing` utility that wraps both cases.
* **The Result:** It creates a rigid, complex abstraction that couples two unrelated features. When you need to change one later, the whole system breaks.
* **The Senior Fix:** Explicitly instruct the AI to prefer **WET (Write Everything Twice)** over DRY (Don't Repeat Yourself).
* **The Prompt Instruction:**

> "Do not create shared utility functions or abstract classes to reduce duplication. I prefer code duplication over wrong abstractions. Implement this logic inline. Do not touch existing files unless absolutely necessary for the fix."

**Why this works:** It forces the AI to keep the "blast radius" of its changes small. It prevents the AI from "infecting" your stable code with new, untested abstractions.

## 2. Architecture as "Read-Only" Context

Most Vibe Coding failures happen because the AI tries to "improve" your architecture. You need to treat your core architecture (auth flows, database connections, base types) as **Immutable**.

* **The Strategy:** When you feed context to the AI, you must mentally (or technically, via file permissions) separate your project into **The Core** and **The Surface**.
* **The Core (Read-Only):** `types.ts`, `schema.prisma`, `auth.config.ts`. The AI reads these to understand *how* to write code, but is strictly forbidden from modifying them.
* **The Surface (Writeable):** `page.tsx`, `Button.tsx`, specific feature logic.

* **The Workflow:** If the AI says, "I need to change the `User` interface to add this field," you stop. **You** change the interface manually. Then you tell the AI: "The interface is updated. Now write the implementation."

## 3. Type-Driven Development (The "Contract" Method)

Since you are a Senior developer, you know that **Types are the Specification**.

Don't describe the *logic* to the AI ("write a function that checks users..."). Describe the *shape*.

1. **Phase 1 (Human):** You write the empty function signature and the complex TypeScript types/interfaces. You define the inputs and the exact outputs.

```typescript
// You write this:
interface ContractReviewResult {
  riskScore: number;
  flaggedClauses: Array<{ id: string; reason: string }>;
}

export async function analyzeContract(pdfText: string): Promise<ContractReviewResult> {
   // TODO: Implement using local AI model
   throw new Error("Not implemented");
}
```

2. **Phase 2 (AI):** You highlight this code and say: *"Implement the body of this function. You strictly must satisfy the `ContractReviewResult` interface. Do not change the interface."*

**Why this works:** This traps the AI. It cannot hallucinate new return formats or change your data structures. It focuses the "vibe" purely on the implementation details, which is safe to iterate on.

## 4. The "Linter" for AI Logic

You need a set of "Negative Constraints" to stop the AI from acting like a Junior Developer. Add these to your system prompt or `.cursorrules`:

* **"No External Dependencies":** forbid the AI from adding `npm` packages. If it thinks it needs `lodash` or `moment`, it must write the helper function from scratch or use native JS methods.
* **"No 'Layout' Thrashing":** If you ask for a logic change, forbid CSS/UI changes. If you ask for a UI change, forbid logic changes.
* **"Comment-Driven Logic":** Require the AI to write a plan in comments *inside the code* before it writes the code. This forces the LLM to "think" before it generates, reducing the chance of hallucinated logic.

## 5. The "Refactor" Ban

Never, ever use the word "Refactor" with an AI unless you are willing to throw the file away.

* **Bad Prompt:** "This code is messy, clean it up." (The AI will rename variables, split files, and change logic structure arbitrarily).
* **Good Prompt:** "This function is too long. Extract the validation logic into a private function named `validateInput` within this same file. Do not change the logic flow."

## Summary: The Senior "Vibe" Mindset

To make the software robust, you must treat the AI not as a "Partner," but as a **High-Speed Junior Contractor**.

1. **You are the Architect:** You define the Types and the Folder Structure.
2. **You are the Gatekeeper:** You forbid dependency bloat and premature DRY.
3. **AI is the Mason:** It lays the bricks (implementation) exactly where you pointed, without redesigning the wall.


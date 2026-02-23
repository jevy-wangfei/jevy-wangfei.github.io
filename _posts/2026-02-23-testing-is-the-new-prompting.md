---
title: "The AI Coding Paradigm Shift: Why Testing is the New Prompting"
date: 2026-02-23 10:00:00 +1100
categories: [Thinking]
tags: [AI, testing, TDD, software engineering, TypeScript, Zod, Vitest]
---

As AI coding capabilities skyrocket, our day-to-day reality as developers is fundamentally changing. We are no longer just "code writers"—we are rapidly transitioning into **system architects and logic validators**.

With AI able to churn out boilerplate and complex algorithms in seconds, the most critical question is no longer *how to write the code*, but *how to prove the code actually works*. After diving deep into the best ways to validate AI-generated code, I've realized that the answer lies in an evolved version of Test-Driven Development (TDD).

Here is how modern AI-assisted engineering is shaping up, and why your testing strategy is now your most important prompt.

### 1. The "Contract-First" AI Workflow

A common trap is asking AI to write business logic first, then struggling to verify if it hallucinated edge cases. The solution is flipping the script: **Interfaces first, tests second, implementation last.**

* **Define the Skeleton:** You define the data structures (Types/Interfaces) and write an empty method signature with clear JSDoc comments detailing the business rules.
* **AI Writes the Tests (Red):** You instruct the AI to generate a comprehensive test suite based *only* on that signature and comments. Your job is to review the tests. Reviewing assertions is vastly easier and less error-prone than reviewing raw business logic.
* **AI Writes the Code (Green):** Once the test safety net is in place, you let the AI write the actual implementation until all tests pass. You’ve successfully boxed the AI into your exact acceptance criteria.

### 2. Emergent Design: Handling Ambiguous Requirements

What if you are in the early stages of a project and the requirements are blurry? You can't design massive interfaces upfront.

This is where AI-assisted TDD shines through **Emergent Design**. You don't need to over-engineer. You write a single, high-level test for the *intention* (e.g., "when a user submits a payload, the dashboard updates"). You let the AI generate the absolute minimum code to make that specific test pass. As requirements crystallize, you add new tests, and the AI refactors the underlying architecture to support them. You are driving the design via behavior, not assumptions.

### 3. Surviving Refactors without "Test Rot"

In traditional development, changing a core feature means dreading the dozens of broken tests you'll have to manually fix. In the AI era, we treat broken tests as a feature, not a bug.

If a requirement changes (for example, shifting from flat JSON mapping to supporting deeply nested JSON objects), the workflow is passive and precise:

1. **Change the Contract:** Update your TypeScript interfaces.
2. **Let it Break:** Let the compiler and test runner light up in red. These failures are your exact blast radius.
3. **Batch Fix with AI:** Feed the updated data structures and the failed test logs directly into your AI IDE (like Cursor). The AI will bulk-update your assertions to match the new reality. You just review the diffs.

### 4. The Golden Tech Stack for AI-Driven Development

To make this loop as fast and hallucination-free as possible, your tooling needs to be strictly typed and lightning-fast. For modern full-stack development, this is the trifecta:

* **Zod (The Contract):** AI understands semantic validation perfectly. Defining your schemas with Zod gives the AI an infallible set of rules and boundaries.
* **Vitest (The Engine):** ESM-native, zero-config, and incredibly fast. You need rapid Red-Green cycles to keep the AI in flow, and Vitest delivers this much better than legacy test runners.
* **Playwright (The E2E Validator):** Its semantic locators (`getByRole`) and built-in codegen are tailor-made for AI to understand UI interactions and auto-heal broken DOM tests.

### Actionable Example: The Zod + Vitest Workflow

To make this concrete, here is how you can practically apply the "Contract-First" workflow using Zod and Vitest:

```typescript
import { z } from "zod";
import { describe, it, expect } from "vitest";

// 1. The Contract (Zod Schema)
// We define the strict boundaries of our data first.
const UserPayloadSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  role: z.enum(["admin", "user", "guest"]),
});

type UserPayload = z.infer<typeof UserPayloadSchema>;

// 2. The Signature
// We write the JSDoc to instruct the AI, leaving the implementation blank.
/**
 * Processes a new user registration payload.
 * AI INSTRUCTION: Implement this function to validate the payload using UserPayloadSchema.
 * If valid, return a normalized user object. If invalid, throw a validation error.
 */
function processUserRegistration(payload: unknown): UserPayload {
  // AI fills this in AFTER tests are written
  return UserPayloadSchema.parse(payload); 
}

// 3. The Tests (Red Phase)
// We instruct the AI to generate tests based ONLY on the signature above.
describe("processUserRegistration", () => {
  it("should successfully process a valid user payload", () => {
    const validPayload = {
      id: "123e4567-e89b-12d3-a456-426614174000",
      email: "test@example.com",
      role: "user"
    };
    
    const result = processUserRegistration(validPayload);
    expect(result).toEqual(validPayload);
  });

  it("should throw an error for an invalid email", () => {
    const invalidPayload = {
      id: "123e4567-e89b-12d3-a456-426614174000",
      email: "not-an-email", // Invalid email format
      role: "user"
    };
    
    expect(() => processUserRegistration(invalidPayload)).toThrow();
  });
});
```

By presenting the AI with the Zod schema and the empty function signature, it can instantly generate a robust test suite. Once you review and approve the tests, the AI can reliably fill in the implementation, completely constrained by your tests and types.

### The Takeaway

In the age of AI, testing is no longer a chore you do at the end of a sprint to satisfy QA. Your tests *are* your instructions. By mastering type-strict contracts and AI-driven TDD, you stop wrestling with AI hallucinations and start directing it with absolute precision.

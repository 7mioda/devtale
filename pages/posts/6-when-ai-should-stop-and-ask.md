---
title: ⏸️ When the AI Should Stop and Ask
date: 2026/09/02
description: What building human-in-the-loop contract review taught us about agent suspension, context, and human judgment.
tag: artificial intelligence
author: You
---

# When the AI Should Stop and Ask

When we talk about AI agents, autonomy is usually presented as the final goal.

The agent should understand the task, make a plan, use its tools, and finish the work without interruption. The fewer questions it asks, the more impressive the demo feels.

But legal work is not a demo.

When an AI reviews a contract, its recommendations can change obligations, liability, payment terms, or termination rights. Moving fast is useful, but making every decision alone is not. Sometimes the most intelligent thing an agent can do is stop and ask.

This is something we learned while building **Oro, an AI legal assistant for in-house legal teams**. Internally, the project is called Aura. One of Oro's main jobs is to review a contract against a legal playbook, explain the risks, and propose edits.

At first, the flow looked straightforward: analyze the document, generate recommendations, return the result.

Then we tried to make the review truly interactive.

That is where things became interesting.

## The review we actually wanted

A lawyer does not simply want a list of AI-generated changes. They need to stay in control of every recommendation.

For each one, they should be able to:

- **Accept** it and apply the proposed edit
- **Reject** it and leave the document unchanged
- **Revise** it by giving Oro a new instruction

The important part is not the three buttons. A confirmation modal can give you three buttons.

The important part is what happens after the click.

If the user rejects a recommendation, the agent should know it was rejected. If they ask for a softer clause, that instruction should become part of the review. If they accept an edit, the next recommendation must operate on the newly edited version of the document.

We did not want a collection of disconnected API calls. We wanted one continuous review in which the human and the agent could take turns.

## Suspension is more than waiting

The model we adopted was **suspend and resume**.

Oro analyzes the document and creates its recommendations. Instead of finishing the tool call, it suspends the run and sends the current review state to the interface. The user makes one decision, and we resume the same run with that decision.

The flow looks like this:

```text
Analyze the contract
        ↓
Suspend with recommendations
        ↓
Human accepts, rejects, or revises
        ↓
Resume the same agent run
        ↓
Persist the decision and apply the change
        ↓
More recommendations? ── yes → suspend again
        │
        no
        ↓
Complete the review
```

This sounds like a small lifecycle detail. In practice, it changes the relationship between the user and the agent.

The human is no longer approving work after the agent has disappeared. Their judgment happens **inside the agent's loop**. Oro resumes with the review session, the previous decisions, the revision history, and the latest document version.

The next iteration is therefore not starting from zero.

## Two kinds of memory

One of the first lessons was that a suspended agent run is not enough on its own.

We needed two sources of state with different responsibilities.

The **review session** is durable. It stores the recommendations, the user's decisions, revision attempts, and the current document version. If the page reloads, we still know what happened.

The **agent snapshot** represents the paused execution. It allows Mastra, the agent framework we use, to resume the same run at the point where it stopped.

You need both.

The durable session tells us the truth about the review. The suspended run gives the agent continuity. Treating either one as the whole system created some painful bugs.

And yes, we found most of them the hard way.

## Then reality arrived

The first version worked well when the user clicked slowly and every request succeeded.

So, basically, it worked in the perfect world we never get to live in.

### A stale decision could end the entire run

Our interface updated optimistically. The moment a user accepted a recommendation, it moved to the next one. That made the experience feel fast, but it also created a race: another decision could be sent before the previous resume had completely suspended again.

The server would receive a stale recommendation that had already been decided. We correctly refused to apply it twice, but we returned the error as an ordinary tool result.

To the agent framework, an ordinary result meant the tool had finished. The run completed, its suspended snapshot disappeared, and the review session was left with recommendations still waiting for a decision.

The next resume failed because there was nothing left to resume.

One stale click had turned a recoverable interaction into a terminal failure.

### The document kept moving

Applying edits exposed another problem.

Imagine the review starts on version 1 of a contract. The user accepts the first recommendation, creating version 2. When they accept the second recommendation, the system must edit version 2.

Ours was still comparing against version 1.

The safety check was doing exactly what it was designed to do: it rejected an edit targeting an outdated document. The mistake was that the review session had not advanced its own version after applying the first edit.

The fix was simple once we understood it:

```text
Review starts on v1
First accepted edit creates v2
Session now points to v2
Second accepted edit creates v3
Session now points to v3
```

We kept the safety check. We changed the state around it.

That distinction matters. Removing a guard because it exposes a bug often hides the real problem instead of fixing it.

### Even the preview could lie

At one point, the review itself was correct but the final preview still opened an old endpoint. The application knew the latest file version, yet part of the interface was reconstructing a legacy preview URL from an edit identifier.

The result was a 404 after a successful review. Not exactly the reassuring final moment we wanted.

We fixed it by making the review outcome point to the preview derived from the latest document version. The artifact shown to the user now follows the same version cursor used to apply the edits.

## The invariant that changed everything

After following these failures through the entire stack, we arrived at one rule:

> **If the review is still active, every interaction must end in another suspension.**

Not only the happy path.

A duplicate decision must re-suspend with the latest state. A missing recommendation must re-suspend with an explanation. A failed edit must leave the recommendation pending and re-suspend. A failed revision must preserve its history and let the user try again.

Only a review with no remaining decisions should complete the run.

In simplified form, the lifecycle became:

```ts
const result = await applyHumanDecision(review, decision)

await save(review)

if (review.hasPendingRecommendations()) {
  return await suspend({
    review,
    interactionError: result.error
  })
}

return complete(review)
```

We also added a synchronous client-side guard so two decisions cannot be submitted from the interface at the same time. That guard improves the experience, but it is not the foundation of correctness. The server still treats stale actions safely because browsers retry, tabs multiply, and networks do strange things.

**The UI can prevent mistakes. The workflow must survive them.**

## Did this make the AI smarter?

Not in the magical sense.

We did not suddenly train a better model, and I would not claim that suspend and resume automatically improves legal accuracy.

What it improved was the quality of the collaboration.

The agent now sees the review as a sequence of decisions instead of a single generation followed by unrelated clicks. A revision instruction stays attached to the recommendation it changed. Accepted edits advance the document the agent is working on. Rejections remain visible in the session instead of disappearing into frontend state.

That makes later iterations more relevant because they start from what the human actually decided.

It also changes trust. The user can inspect each recommendation, understand the proposed change, and decide how Oro should proceed. Automation still removes the repetitive work, but judgment stays with the lawyer.

That balance is especially important for in-house legal teams. They do not need an AI that confidently makes every decision. They need one that can move quickly, explain itself, and know when control belongs to them.

## What we learned

- **Human-in-the-loop is an architecture, not a button.** The user's decision must become part of the workflow state.
- **A paused run is not durable business state.** Persist the review separately and use the agent snapshot for execution continuity.
- **Recoverable errors must remain recoverable.** In a suspended workflow, returning the wrong kind of error can accidentally become a completion signal.
- **Document edits need a version cursor.** Every accepted change must advance the version used by the next one.
- **Optimistic interfaces need pessimistic correctness.** Make the UI feel fast, but assume duplicate and stale requests will eventually happen.
- **The agent should complete only when the work is actually complete.** Framework lifecycle and product lifecycle must agree.

## Final thoughts

The most useful AI systems will not be the ones that remove humans from every loop.

They will be the ones that understand **which loop needs a human**, preserve the context around that decision, and continue without losing the thread.

For Oro, suspension started as a technical mechanism. It became something more important: the point where AI automation makes room for legal judgment.

Sometimes, stopping is not a limitation.

Sometimes, it is the feature.

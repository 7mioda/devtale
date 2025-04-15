---
title: 🤔 Frontend Architecture Dilemma
date: 2025/3/06
description: A journey through frontend architecture decisions and lessons learned.
tag: web development
author: You
---

# Frontend Architecture Dilemma

The journey started five years ago when our CTO pushed the very first commit to our frontend app. Back then, **React + TypeScript** (with Create React App) was the obvious go-to stack. With the hype surrounding **GraphQL**, adopting it felt like a prestigious and natural choice.

But, like many early-stage startups, we moved fast — maybe too fast. Our codebase quickly devolved into a **big ball of mud**. Every component, style, and type became deeply coupled to another part of the app. Bugs were hard to trace, and refactoring felt like going in circles. We weren't moving forward — we were treading water in ambiguity.

Eventually, we hit a tipping point. We had to find a better way — not just for maintainability, but to keep our sanity intact.

Two architectural ideas emerged:

- **Centralized orchestration**
- **Isolated, vertical slices**

## **Centralized Orchestration (The Widget + Context Pattern)**

The first solution was to create **feature widgets** — each wrapped in its own **React Context**. This context would be responsible for:

- Fetching initial data
- Exposing all available actions (e.g., create, update, delete)
- Sharing relevant state with child components

At first, this felt clean. Each feature had a boundary. But as features grew in complexity, we realized we'd just broken the original problem into **N smaller problems**:

- Each context began exposing **more data and actions than any one consumer needed**.
- Many exposed actions were fired only 10% of the time.
- Onboarding devs had to dig into context internals just to understand how a component worked.
- Coordinating across contexts was **harder** than we thought.

Ultimately, we were just **moving complexity**, not eliminating it.

## **Isolated Components & Vertical Slices (Keep It Simple)**

The second solution was more grounded: **embrace simplicity and isolate responsibility**.

We decided:

- Use **state and context only when absolutely necessary**.
- **Push state down the React tree** as much as possible (minimizing unnecessary renders and improving performance).
- Let each component **own its logic**: data fetching, mutations, side effects — all encapsulated within.

Because we were using **Apollo Client**, many of our concerns around syncing data across components were already handled. Components connected to the same query would re-render automatically on mutation — no extra coupling needed.

This approach wasn't just simpler. It was **more aligned with how React wants to be used**.

## What we learned

- Don't architect for flexibility you don't yet need.
- Contexts are powerful, but overuse can lead to **fragility**, not **scalability**.
- Smaller, self-contained components are **easier to onboard**, **test**, and **scale**.
- If you're using a smart data layer like Apollo or React Query, you already have a lot of shared-state magic — don't reinvent it.

## **Final Thoughts**

There's no silver bullet. But when you're growing a team and building under pressure, **simplicity wins**. The vertical slice approach made our app easier to reason about and gave us confidence to build faster — without the fear of breaking everything.

We're still learning, but we've come a long way from the mud. 🛠️

This will be the first in a series of articles on our journey to build a better frontend — tomorrow, in Tomorro.

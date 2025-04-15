---
title: 😵‍💫 The React Context Trap — A Hard Lesson in Over-Engineering
date: 2025/3/12
description: Understanding when and how to use React Context effectively.
tag: web development
author: You
---

# The React Context Trap

As you get more experienced with React, you start to develop what I call the **"senior frontend dev eye"** (even if you're not technically a senior yet). You start spotting patterns, thinking about abstractions, and — sometimes — trying to optimize things that don't need optimizing.

That's how I, like many others, fell into the **React Context trap**.

## The Allure of Context

**Context feels like magic... until it isn't.**

It starts simple: you're passing a few props down a long tree, and you think — *"Why prop-drill when I can use a context? Just one wrapper and boom, everything has access to the data they need."*

It feels clean. It feels smart. It feels like you've leveled up.

## The Downfall

But fast-forward a few weeks or months, and you're drowning in it:

- Dozens of context providers
- Components consuming data from 3–4 different contexts
- Components with **no props at all** but dozens of hidden dependencies
- Moving a component becomes a mystery — you don't know what it needs to work anymore
- And then... 🧪 you try to test.

## The Root Problem

**Context's biggest flaw? Convenience.**

Because it's *so easy* to reach into a context, people will — even seasoned devs. Instead of passing a prop, they'll grab a slice of context data. And then another. And another.

What you end up with is a component that's **tightly coupled to global state**, with zero indication of what it actually needs to function. No props. No signature. Just hidden dependencies all over the place.

And when you finally go to write tests? Surprise! You're now mocking **massive, overgrown context objects** for a component that only needed one flag or a single value.

## Lessons Learned

**So here's what we learned — the hard way.**

Modern React apps thrive on **simpler, more focused state**:

✅ **Keep state local** whenever possible and push it down the tree as much as you can.

✅ **Only use context** for state that truly needs to be shared across distant parts of the app

✅ **Avoid Redux** unless you're working with deeply cross-cutting or nested state (rare in most apps)

✅ Use the right tool for the right job:

- **React Query / Apollo** → for server-side state & caching
- **Zustand / Jotai** → for lightweight global state without boilerplate
- **Custom hooks + context** → only when shared state is small and intentional

## The Golden Rule

**💡 Always ask yourself:**

> "Do I really need a context or a state manager here, or can I just pass a prop?"

The answer might surprise you. Simpler apps are faster, easier to debug, and way more fun to work on.

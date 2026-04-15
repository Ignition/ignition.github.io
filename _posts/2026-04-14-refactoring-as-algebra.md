---
title: "Refactoring as Algebra: Small Steps to Clarity"
date: 2026-04-14 17:00:00 +0100
categories: [C++]
tags: [refactoring, kata, c++]
---

## The Problem with Clever Code

The [Gilded Rose kata](https://github.com/emilybache/GildedRose-Refactoring-Kata) is a beloved refactoring exercise. You inherit a small inventory system for a fantasy shop. The code works, but the logic is impenetrable. Your task is to add a new feature, and the code is fighting you.

As you refactor, you might get to a fragment that looks like this:

```cpp
quality = std::min(quality + 1, 50);
sellIn = sellIn - 1;
if (sellIn < 0)
{
    quality = std::min(quality + 1, 50);
}
```

What does this do? Quality goes up by one, capped at 50. The sell-by date decreases. And then... quality goes up again if we're past the sell-by date?

But wait. We just decremented `sellIn`, so the condition `sellIn < 0` is checking the _new_ value, not the original. If `sellIn` was 1 before this code ran, it becomes 0, and the condition is false. If `sellIn` was 0, it becomes -1, and we get the bonus increment.

The logic is correct, but it's _hiding_. The mutation and the condition are entangled in a way that requires you to simulate execution in your head. That's cognitively expensive, error-prone, and doesn't scale.

Let's untangle it. And let's do it the slow way, the way you'd make a student show their working, so we can see exactly what's happening at each step.

## Separating Calculation from Mutation

The first problem is that `sellIn = sellIn - 1` does two things at once: it calculates a new value and stores it, destroying the old one. Everything downstream has to reason about which version of `sellIn` it's seeing.

Let's separate those concerns:

```cpp
quality = std::min(quality + 1, 50);
auto const tmp = sellIn - 1;
sellIn = tmp;
if (sellIn < 0)
{
    quality = std::min(quality + 1, 50);
}
```

We've added a line and introduced a temporary variable. The code is longer. Have we made things worse?

No. We've made the two operations explicit: `tmp = sellIn - 1` is pure calculation, and `sellIn = tmp` is pure mutation. The new value now has its own name, separate from the old value.

The goal is simple: **separate calculation from mutation**. Calculate everything first, using names for intermediate values, and defer the actual mutations until as late as possible. This gives you a window where you can reason about the calculations without worrying about what's been modified.

Fowler's **Split Variable** refactoring captures a related idea: a variable should represent one concept, not be reused for multiple purposes. We're taking that further, not just avoiding reuse, but making the flow of values explicit.

## Propagating the Known Value

Now `tmp` holds the new value, and `sellIn` is immediately assigned from it. For a brief moment, they're equal. That means we can substitute one for the other.

Look at the condition `if (sellIn < 0)`. At this point, `sellIn` equals `tmp`. So:

```cpp
quality = std::min(quality + 1, 50);
auto const tmp = sellIn - 1;
sellIn = tmp;
if (tmp < 0)
{
    quality = std::min(quality + 1, 50);
}
```

This is called **copy propagation**, replacing a variable with the value it's known to hold. The behaviour is identical, but we've done something important: the condition no longer depends on the assignment to `sellIn`. We've broken a hidden dependency.

What was the dependency? The original code created a trap: the meaning of `sellIn` in the condition depended on whether you read it before or after the mutation. By propagating `tmp` into the condition, we've made it clear that the condition depends on the _calculated_ value, not the _stored_ value.

This transformation doesn't appear in Fowler's catalogue. It's too granular, too mechanical, the kind of thing compilers do automatically. But for humans untangling tricky code, it's essential.

## Sinking the Mutation

Now look at `sellIn = tmp`. Nothing inside the if-block reads `sellIn`. The only thing that matters is `tmp`, and we've already captured that. So we can move the assignment later:

```cpp
quality = std::min(quality + 1, 50);
auto const tmp = sellIn - 1;
if (tmp < 0)
{
    quality = std::min(quality + 1, 50);
}
sellIn = tmp;
```

This is **code sinking**, moving a statement as late as possible while preserving behaviour. Fowler calls the general pattern **Slide Statements**. Same idea, different names from different communities.

The payoff: the `sellIn` mutation is now at the end, separated from the quality logic. The condition above it depends only on the calculated value, not on when the assignment happens. We still have `quality` mutations earlier, but we've untangled the two concerns from each other.

## Inlining to Simplify

With `tmp` only used in two places and no mutations happening between them, we can inline it:

```cpp
quality = std::min(quality + 1, 50);
if (sellIn - 1 < 0)
{
    quality = std::min(quality + 1, 50);
}
sellIn = sellIn - 1;
```

This is Fowler's **Inline Variable**. And now we can simplify the arithmetic, `sellIn - 1 < 0` is equivalent to `sellIn <= 0`:

```cpp
quality = std::min(quality + 1, 50);
if (sellIn <= 0)
{
    quality = std::min(quality + 1, 50);
}
sellIn = sellIn - 1;
```

Look what's emerged. The structure is now visible:

- Quality always increases by at least one
- If we're on the expiry day (or past it), quality increases again
- The sell-by date decrements at the end, isolated from the quality logic

We haven't changed what the code does. We've changed how it _communicates_ what it does.

## The Counterintuitive Move: Distribution

Here's where most developers stop. The code is cleaner, the logic is visible, we could move on. But there's one more transformation available, and it's the one that surprises people.

We have code before the conditional and code inside one branch. What if we duplicate that prefix into both branches?

```cpp
if (sellIn <= 0)
{
    quality = std::min(quality + 1, 50);
    quality = std::min(quality + 1, 50);
}
else
{
    quality = std::min(quality + 1, 50);
}
sellIn = sellIn - 1;
```

We've introduced duplication. Everything you've learned about refactoring says this is wrong. Extract common code, don't duplicate it. DRY, Don't Repeat Yourself.

But this duplication is _preparatory_. We've temporarily made the code worse to unlock an opportunity that wasn't visible before.

Fowler has a concept for this: **preparatory refactoring**. Sometimes you refactor not to improve the code directly, but to make a subsequent change easier. You're setting up the board. The duplication is scaffolding.

Compilers do this routinely. They call it **tail duplication**, copying code into multiple paths to enable path-specific optimisation. Loop unswitching is a related idea: duplicating an entire loop body to hoist a loop-invariant conditional out of the hot path.

But here's the gap: _this pattern has no name in Fowler's catalogue_. The developer-facing refactoring literature doesn't call out deliberate duplication as a valid move. It may be a blind spot.

This matters more than it might seem. Compiler engineers can leave transformations unnamed because they're encoded in algorithms. A compiler doesn't forget tail duplication because it had a long day. But developers work from internalised patterns, and patterns without names are patterns we reach for less often, or never.

I call this **Distribute for Fusion**. The name captures both the mechanism (distributing code into branches, like distributing multiplication over addition) and the intent (setting up a fusion that will eliminate the duplication).

Think of it mathematically. If you have `a × (b + c)`, you can factor it or distribute it:

- Factoring: `ab + ac → a(b + c)`, extract the common factor
- Distribution: `a(b + c) → ab + ac`, duplicate to specialise

Both are valid algebraic identities. Both preserve meaning. The skill is knowing when each applies.

## Fusion: The Payoff

Now look at the true branch. Two identical operations in sequence:

```cpp
quality = std::min(quality + 1, 50);
quality = std::min(quality + 1, 50);
```

These can fuse into one:

```cpp
quality = std::min(quality + 2, 50);
```

And our code becomes:

```cpp
if (sellIn <= 0)
{
    quality = std::min(quality + 2, 50);
}
else
{
    quality = std::min(quality + 1, 50);
}
sellIn = sellIn - 1;
```

The scaffolding is gone. We distributed to create adjacent duplicate operations, then fused them into something simpler. Two runtime operations became one. The temporary duplication served its purpose and vanished.

## Expressing Intent

The two branches now have the same structure, differing only in a constant. Let's make that explicit by extracting the constant into a variable, Fowler's **Extract Variable**:

```cpp
int updateAmount;
if (sellIn <= 0)
{
    updateAmount = 2;
    quality = std::min(quality + updateAmount, 50);
}
else
{
    updateAmount = 1;
    quality = std::min(quality + updateAmount, 50);
}
sellIn = sellIn - 1;
```

Now the branches differ only in the value of `updateAmount`. The `quality` update is identical in both. That's a common suffix we can factor out, Fowler's **Slide Statements** (previously called **Consolidate Duplicate Conditional Fragments**):

```cpp
int updateAmount;
if (sellIn <= 0)
{
    updateAmount = 2;
}
else
{
    updateAmount = 1;
}
quality = std::min(quality + updateAmount, 50);
sellIn = sellIn - 1;
```

The conditional now does one thing: choose a value. That's a perfect fit for a ternary:

```cpp
auto const updateAmount = sellIn <= 0 ? 2 : 1;
quality = std::min(quality + updateAmount, 50);
sellIn = sellIn - 1;
```

Each step reduced the conditional to something simpler: from duplicated logic, to a shared suffix, to a single expression. We've arrived. The code now _states its intent_: items gain quality each day, and they gain twice as much on or past their sell-by date. The cap at 50 is explicit. The sell-by decrement is isolated at the end, no longer entangled with the quality logic.

Compare this to where we started:

```cpp
quality = std::min(quality + 1, 50);
sellIn = sellIn - 1;
if (sellIn < 0)
{
    quality = std::min(quality + 1, 50);
}
```

Same behaviour. Completely different clarity.

## Two Communities, One Algebra

Here's what we used:

| Step | Fowler Name | Compiler Name |
|---|---|---|
| Separate calculation from mutation | Split Variable | — |
| Substitute known value | — | Copy propagation |
| Move mutation later | Slide Statements | Code sinking |
| Remove temporary | Inline Variable | Inlining |
| Duplicate into branches | — | Tail duplication |
| Combine operations | — | Fusion |
| Extract the constant | Extract Variable | — |
| Factor out common suffix | Slide Statements | Code factoring |
| Collapse conditional to expression | — | If-conversion |

The compiler column is more complete because compiler engineers have catalogued these transformations for decades. They're proven correct, applied billions of times daily, and encoded in software that doesn't forget.

The Fowler column has gaps. Not because Fowler was careless, he chose his patterns wisely, prioritising the most common and teachable. But some powerful transformations fell through the cracks, unnamed in the refactoring literature even though they're essential for untangling real code.

When I've presented this sequence to experienced developers, something interesting happens. They nod along, they've done these moves before, intuitively, when wrestling with gnarly code. But they've never broken it down into named steps. They've never seen the algebra made explicit.

When I show **Distribute for Fusion**, deliberately introducing duplication, there's often a moment of resistance. It feels transgressive, a violation of principles they've internalised. And then the payoff lands: the duplication enables a simplification that wasn't otherwise reachable. The scaffolding goes up and comes down, leaving cleaner code behind.

That's what the algebraic framing gives us. Not just a bag of tricks, but a _system_, transformations with inverses, moves that compose, patterns you can reason about. Factor or distribute. Sink or hoist. Inline or extract. Each pair an axis you can move along, each step small enough to verify, each combination opening new possibilities.

Small steps. Local reasoning. Cumulative power.
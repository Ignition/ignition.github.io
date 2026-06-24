---
title: "Permutation Maps: zip+sort with indirection"
date: 2026-06-24 09:00:00 +0100
categories: [C++]
tags: [c++, c++23, ranges, zip, performance]
---

At [ACCU on Sea 2026](https://accuonsea.uk/2026/) I sat in on Nicolai Josuttis' talk [Taming C++23: The things about C++23 you do not know](https://accuonsea.uk/2026/sessions/taming-cpp23-the-things-about-cpp23-you-do-not-know/). One of his examples was the zip+sort: sorting two separate collections together by zipping them and sorting on a projection. It's a very good demonstration of how `std::views::zip` and the ranges algorithms compose.

It also reminded me of a pattern I've now reached for and used in two different codebases. It's the same zip+sort, but with one more level of indirection. Instead of sorting the data, you sort a vector of indices alongside it and throw the data away, keeping only the reordered indices. What you get back is a _permutation map_: a small, cheap, reusable description of an ordering that you can apply later.

While it's fresh in my mind, here it is.

## The zip+sort

Here's the shape of the example from the talk. Two separate sequences, sorted together so they stay pairwise associated:

```cpp
auto names = std::vector<std::string>{"Alice", "Bob", "Carol"};
auto ages  = std::vector<int>{30, 25, 40};

std::ranges::sort(std::views::zip(ages, names), {},
                  [](const auto& pair) { return std::get<0>(pair); });
// ages:  {25, 30, 40}
// names: {"Bob", "Alice", "Carol"}
```

The zip view yields tuples of references, we sort on the projection `std::get<0>` (the age), and `sort` swaps those tuples as a unit, so both vectors move their values in lockstep. No manual index bookkeeping, and no temporary copies to keep the two collections pairwise associated.

## The zip+sort with indirection

Now suppose I don't actually want to move the data. I just want to _know_ the order. I introduce a vector of indices that starts as `0, 1, 2, ...`, zip _that_ against the data, and sort on the data element as the projection. The indices ride along with the data during the sort, so when it finishes, `indices[k]` tells you _which original position_ holds the `k`-th smallest element.

The data is taken by value precisely because we're going to scramble it and then discard it. All we keep is the permutation. Reading an element back in sorted order is then `data_original[indices[k]]`. The data never moved; we only built a table that says where to look, at the cost of one level of indirection on every read.

## A worked example

Here the permutation orders by one key (`scores`) while leaving _several_ separate arrays (`names`, `cities`) untouched. [Try it on Compiler Explorer.](https://godbolt.org/z/6Yjaz5TMq)

```cpp
#include <algorithm>
#include <array>
#include <print>
#include <ranges>
#include <string_view>
#include <vector>

auto permutation_map(std::vector<int> data) -> std::vector<std::size_t> {
    auto indices = std::views::iota(std::size_t{0}, data.size())
                 | std::ranges::to<std::vector>();

    std::ranges::sort(std::views::zip(indices, data), {},
                      [](const auto& pair) { return std::get<1>(pair); });

    return indices;
}

int main() {
    const auto names  = std::vector<std::string_view>{ "Alice", "Bob", "Carol", "Dave", "Eve"   };
    const auto cities = std::vector<std::string_view>{ "Paris", "Rome", "Oslo", "Lima", "Kyoto" };
    const auto scores = std::vector                  {       5,      2,      8,      1,      9  };

    const auto perm = permutation_map(scores);

    std::println("Top-3 podium (best → worst):");
    for (auto [medal, idx] : std::views::zip(std::array{"🥇","🥈","🥉"}, perm | std::views::reverse))
        std::println("  {} {:>5}  score={} city={}", medal, names[idx], scores[idx], cities[idx]);
}
```

Compiled with `g++-14 -std=c++23`, this prints:

```text
Top-3 podium (best → worst):
  🥇   Eve  score=9 city=Kyoto
  🥈 Carol  score=8 city=Oslo
  🥉 Alice  score=5 city=Paris
```

`permutation_map` sorts ascending, so `perm` runs worst to best; `std::views::reverse` flips that to best-first without rebuilding anything, and zipping it against the medal array and slicing the top three is all done on indices. The `names`, `cities`, and `scores` vectors never move, and we never materialise a sorted copy of any of them. We only ever rank, reverse, and look up.

## Why bother? Deferring the data movement

The appeal is that a permutation is cheap to build, cheap to carry around, and cheap to compose. Shuffling indices touches only a few bytes per element; alternatively, physically reordering the items it describes would mean reading and writing each one, and that memory traffic can dominate once the items are non-trivial.

My main use case is making an _ordered selection_ out of data that's stored in a different order. Say the data is laid out as `A B C D E F` (positions `0..5`), and downstream I need the selection `E, C, F` in that specific order. Rather than copying those elements into a new container, I produce a permutation map `4 2 5` and hand that around. The map _is_ the selection-in-order. The actual `E`, `C`, `F` payloads stay exactly where they are.

That matters when:

- **You might not need every element.** If a later phase short-circuits, filters, or only consumes a prefix, you never paid to move the elements you didn't reach. The permutation defers the cost to the moment of actual use, and lets you skip it entirely for the parts you abandon.
- **The same data is viewed several ways.** Several permutations over one immutable backing store is much lighter than several reordered copies of it. Each "view" is just another index vector.
- **If you are comparing in bulk, not reordering.** Operations like merging, diffing, deduplicating, or set intersection only need to _read_ elements in the right order to decide what matches. You can run all of those comparisons through the permutation and leave the items where they are.

In the comparison and extraction phases of a pipeline, this lets you push the physical reordering as late as possible. You compare and select on indices; you materialise reordered data only if and when something genuinely needs it.

## A caveat on locality

This isn't totally free. Indirect access through a permutation is a scatter/gather over the backing store. If you _do_ end up reading the whole thing in permuted order on a hot path, this indirection turns sequential access into random access, which you may notice as poor prefetching and cache usage.

So this pattern is worth it when the saved data movement outweighs the lost locality of the eventual indirect reads. If you're going to stream the entire dataset in permuted order, repeatedly, materialising a reordered copy up front may well be faster overall. As always, measure on your workload.

## Wrapping up

The zip+sort is a tidy C++23 example on its own. Promoting it to operate on indices, and keeping the permutation instead of the reordered data, turns it into a small, reusable tool for separating _the decision about order_ from _the act of reordering_.

If you've used the same trick, or have a sharper way to express it, I'd love to hear about it. Find me on [Mastodon](https://fosstodon.org/@glloyd).

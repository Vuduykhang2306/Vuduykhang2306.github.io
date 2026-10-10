---
layout: post
title: "The first drafts of three findings"
---

The MemeTax report reads as if each finding was spotted, proved and written
up in a straight line. That isn't how it went. Three of them started out wrong,
and each was wrong in a different way.

## M-01: the PoC that passed

The plan was simple. `_swapBack()` calls the router, the router moves tokens to
the pair with `transferFrom`, and that goes back through the token's
`_transfer()`. If the swap triggers again, it recurses until the stack runs
out and every sell reverts. I wrote the PoC expecting a revert.

The sell went through fine.

The reason was in the call path. When the router calls
`transferFrom(token, pair, amount)`, the re-entered frame has
`from == address(token)`. The constructor marks the token contract as
fee-exempt. The swap trigger looks like this:

```solidity
if (to == pair && !isExcludedFromFee[from] && balanceOf[address(this)] >= swapThreshold) {
    _swapBack();
}
```

`!isExcludedFromFee[from]` is false on re-entry, so there's no second swap.

So there's no recursion bug. What stops it is a mapping meant for fee
exemption, and nothing in the code says it also prevents reentrancy.
`setExcludedFromFee(address(this), false)` looks like a normal admin call, and
it removes the guard.

That was the finding, and the PoC had to be rewritten to show it: one sell that
works, then the admin call, then the next sell reverting. I'd assumed the
`lockTheSwap` modifier on every mainnet tax token was copy-paste habit. Now I
know what it's for.

What I took from it: a contract with no bug and a contract with a bug that
something else happens to be covering look the same in a test. "Is this
property true?" isn't enough. I also have to ask what is keeping it true.

## M-02: a fix worse than the bug

Both fee payouts ignore the return value of `call`. My first reaction was the
obvious one, add `require(ok)`.

Then I thought about where `_swapBack()` runs. It runs inside the `_transfer()`
of whoever is selling. If a fee wallet can't receive ETH and the payout
reverts, the user's sell reverts too. Every sell, for every holder. MemeTax has
no setter for the fee wallets, so swapping in a working one isn't even an
option. I would have traded a silent accounting loss for a denial of service on
the whole token.

The fix in the report is a pull pattern. `_swapBack()` only credits
`feesOwed[wallet]`, and each wallet withdraws separately. A broken wallet only
hurts itself.

Finding the wrong line was the easy part of this one. The hard part was
answering "how do I fix this without creating a new bug", and my first answer
to that was wrong.

## L-01: almost a Medium

The fee math divides before it multiplies:

```solidity
amount / 100 * taxRate
```

Any transfer under 100 wei pays no fee. "Fee bypass" sounds serious, and my
first draft had it as Medium.

Then I did the arithmetic. To avoid the 5% fee on one token (1e18 wei), you'd
have to split it into 1e16 transfers. The gas costs more than the fee by many
orders of magnitude. There's no economic attack here, just rounding.
It still belongs in the report, because the same expression in a token with
fewer decimals, or in a share-price calculation, stops being harmless, and it
costs nothing to fix.

I moved it to Low and wrote the reason into the report, so anyone reading it
can check the downgrade instead of taking my word for it. An inflated High
makes the real ones count for less.

## Looking back

M-01 was wrong about the mechanism, M-02 about the fix, L-01 about the impact.
Getting one of those right told me nothing about the other two, because they
are three separate questions.

All three are in the
[report](https://github.com/Vuduykhang2306/smart-contract-security-lab/blob/main/reports/2026-09_MemeTax_audit-report.md)
with PoCs and retests.

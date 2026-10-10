---
layout: post
title: "Whose money is this?"
---

I read `_swapBack()` twice during the MemeTax review and missed the worst bug
in it both times.

Some context. MemeTax is a practice target I wrote for my
[security lab](https://github.com/Vuduykhang2306/smart-contract-security-lab):
a tax token with a dividend pool, the kind of contract that gets launched and
forgotten every week. I planted eight bugs in it and then audited it as if I
hadn't. The report ended up with nine findings. The ninth one I wrote by
accident, and I didn't find it by reading.

## The function

This is the end of the fee swap:

```solidity
uint256 half = address(this).balance / 2;

marketingWallet.call{value: half}("");
devWallet.call{value: address(this).balance}("");
```

The first time I looked at it I was working on M-01, whether the swap could be
re-entered through the router. The second time was M-02: both `call`s throw
away their return value, so a fee wallet that can't receive ETH loses the
payout silently. Both real findings. Both times my question was the same:
*is this call safe?*

Neither time did I ask where `address(this).balance` comes from.

It comes from two places. The swap puts fee proceeds there. But
`depositDividend()` is payable, and it leaves the dividend pool in the same
balance. So the first sell after a dividend deposit sends the holders' money to
the fee wallets. `pendingDividend` isn't touched. The ledger still says the
holders are owed it, and their claims revert because the ETH is gone.

The PoC is boring, which is what makes it bad. Deposit 10 ETH of dividends for
one holder. A different holder sells 1,000 tokens and the swap returns 2 ETH of
real fees. Each fee wallet receives 6 ETH. That's 12 ETH out, 10 of which
belonged to someone else. Contract balance ends at zero, ledger still records
10 ETH owed. No attacker, no weird ordering, just a normal sell.

## How it actually turned up

After patching everything I wrote an invariant suite for the fixed contract. I
thought of it as paperwork, a way to confirm the patches held.

The second property was this one:

```
sum(feesOwed) + sum(pendingDividend) <= address(this).balance
```

Every ETH the contract has promised to someone has to be sitting in its
balance. It's a one-line statement, but once it's written down the next
question asks itself: which paths move the balance without moving the ledger?
`_swapBack()` is one of them. Two careful passes over the function hadn't
shown me that. One line describing the whole contract did.

The fixed version already survived it. My M-02 patch had switched to a pull
pattern, and while doing that I measured `received` as the balance difference
across the swap instead of using the raw balance. So H-04 was closed before I
knew it existed. I'd like to say that was foresight. It was luck.

## What I do differently now

Reading a function shows you bugs inside the function. H-04 isn't inside
`_swapBack()`. It lives between `_swapBack()` and `depositDividend()`, and
neither one is wrong when you read it alone. Unit tests don't help here either,
since a unit test only checks a scenario I already thought of, and I hadn't
thought of this one.

So when a contract holds value for more than one party, I now write down who
owns each wei before I read any function. Fee proceeds belong to the fee
wallets. The dividend pool belongs to holders. Then for every path that sends
ETH out, the question is whose share it's paying from, not whether the call is
safe. If the answer is "whatever happens to be in the balance", that's the
finding.

The full write-up is
[H-04 in the report](https://github.com/Vuduykhang2306/smart-contract-security-lab/blob/main/reports/2026-09_MemeTax_audit-report.md#h-04),
and the PoC is `test/H04_SwapBackSweepsDividendPool.t.sol`.

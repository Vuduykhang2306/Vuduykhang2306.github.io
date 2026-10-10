---
layout: post
title: "A green fuzzing run doesn't prove much"
---

I ran the same four invariants under two fuzzers, expecting one of them to
catch something the other missed. Neither did. I'm writing it up anyway,
because something more useful came out of it.

## The setup

The [MemeTax lab](https://github.com/Vuduykhang2306/smart-contract-security-lab)
has four properties for the patched contract:

- token supply is conserved
- every ETH the contract owes is backed by its balance
- the tax never goes above its cap
- max wallet has a floor

They already ran as a Foundry invariant suite. Foundry and Echidna both fuzz
sequences of calls, but they pick those sequences differently. Foundry draws
each call from a handler and replays whole sequences from scratch. Echidna
keeps a corpus: a sequence that reaches new code gets saved and mutated, so it
builds on whatever worked before. Neither one is strictly better, so it seemed
reasonable to expect that two engines would see more than one.

I ported the harness. Echidna ran 50,000 calls against the fixed contract and
all four came back `passing`. Same as Foundry. Nothing new.

## The actual problem

Looking at four lines of `passing`, I realised I couldn't tell what they meant.

A green run means one of two things. Either the code holds, or the harness
never got anywhere near the code that could break it. The output is identical
in both cases. On the Foundry side I at least had `callSummary()` printing how
often each action ran. Echidna doesn't print anything like that.

So "both fuzzers agree" could mean the contract is fine. It could also mean I'd
written two harnesses with the same blind spot.

## The control

The fix was cheap. I made a second harness identical to the first except for
one line: it deploys the original `MemeTax` instead of `MemeTaxFixed`. Same
actions, same bounds, same four properties. That contract is known to be
broken, so if the properties are worth anything they have to fail on it.

They did, in two places:

```
echidna_supplyIsConserved:       passing
echidna_maxWalletHasFloor:       passing
echidna_ethObligationsAreBacked: FAILED   <- H-04
echidna_taxNeverExceedsCap:      FAILED   <- H-03
```

The ETH backing failure is H-04, the dividend pool bug my manual review missed
([I wrote about it here]({% post_url 2026-10-09-whose-money-is-this %})). Echidna reached
it in five calls, without the harness ever mentioning `_swapBack()`.

The max-wallet property passes on both contracts. That's because the harness
bounds `setMaxWallet` to at least 1% of supply, since a campaign that keeps
freezing every transfer stops exploring. So that property can't fail here at
all, and the floor is covered by a normal retest instead. The repo lists this
as a limit of the control.

CI now checks all four outcomes by name: fixed contract passes, original
contract fails on exactly those two. If a later change to the harness stops it
from reaching the bug, CI goes red even though nothing "failed".

## Checking coverage instead of trusting it

Since Echidna has no call summary, I read coverage from the corpus files, where
executed lines are marked with `*`:

```bash
grep -E 'feesOwed\[|received = |revert TaxTooHigh' echidna-corpus/covered.*.txt
```

All ten actions and every ETH path in the fixed contract show up as executed,
so the passing run did go through the code it was supposed to test.

## Two things that cost me time

`crytic-compile` skips `test/` when it builds a Foundry project, so Echidna
reported my harness as not found. You need
`cryticArgs: ["--foundry-compile-all"]` in the config.

Echidna funds the harness by sending `balanceContract` along with the deploy
transaction, so the harness constructor has to be `payable`. If it isn't, the
deploy reverts and the error just says something generic about revert or
out-of-gas.

## What I'm keeping

Any invariant suite I write from now on comes with a negative control: the same
properties run against a version that is known to be broken, and required to
fail, with CI checking that they still do.

The harnesses and the failing sequences are in
[`test/echidna/`](https://github.com/Vuduykhang2306/smart-contract-security-lab/tree/main/test/echidna).

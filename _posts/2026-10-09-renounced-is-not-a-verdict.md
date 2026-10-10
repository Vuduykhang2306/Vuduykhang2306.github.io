---
layout: post
title: "Renounced doesn't mean safe"
---

"Ownership renounced" usually gets read as "safe". After reviewing a launchpad
token template on mainnet, I don't think it works that way. Renouncing is good
news for some kinds of danger and bad news for others, and a block explorer
won't tell you which kind you're looking at.

## The template

[FatTokenV5](https://github.com/Vuduykhang2306/smart-contract-security-lab/tree/main/reviews/2026-09_FatTokenV5)
is the same bytecode deployed again and again by a launchpad, with each
deployment picking its own configuration in the constructor. I reviewed it
using a dead token as the specimen. Its pool held 0.54 ETH and ownership was
already renounced, so nothing I published could cost anyone money. I only
looked at what the deployer can do to holders, not at anything that would let
a stranger take funds.

The review produced 8 findings and 7 PoCs on a mainnet fork. What I'd keep
from it, though, is a property of the template as a whole: every dangerous
power in the template is switched on or off in the constructor, and no
function can switch any of them back on. `enableKillBlock`, `enableRewardList`
and `enableChangeTax` have no setters, and `disableChangeTax()` only goes one
way. A buyer can read five values on-chain and know their exposure for good.

## Two kinds of danger

Working through the findings, they split into two groups that behave in
opposite ways when ownership is renounced.

Owner powers are things a privileged address can choose to do later. In
this template, `multi_bclist()` freezes up to 200 holders per call.
`completeCustoms()` can move the sell tax to 24.99% in one transaction. Set
`setNumTokensSellRate()` high enough and the multiplication in the sell path
overflows, so every sell reverts while buys keep working. Renouncing kills all
of these permanently, since nobody can call them anymore.

Baked-in mechanisms run by themselves on every transfer, with nobody
calling anything. The template has an airdrop loop that sends 1 wei to
`airdropNumbs` derived addresses on every trade and then does:

```solidity
amount -= airdropNumbs * 1;
```

Trades smaller than `airdropNumbs` underflow and revert. On the specimen the
setters that could turn this off are `onlyOwner`, and the owner is `0x…dEaD`.
So renouncing didn't remove the loop. It made it permanent. Every trade for
the rest of that contract's life pays for three extra transfers and emits three
junk `Transfer` events.

The kill block is the same shape. If it was enabled at construction, buyers in
the first `kb` blocks get flagged, and flagged addresses can never sell. Once
the owner is gone, nobody can clear the flag either.

So the same `owner() = 0xdead` is good news for the first group and bad news
for the second. A tool that gives one verdict for "renounced" gets one of the
two groups wrong.

## Turning it into something a buyer can use

"Read five values" isn't much help unless you know which five. So I built
[token-powers](https://vuduykhang2306.github.io/token-powers/): paste a mainnet
ERC-20 and it tells you which owner powers exist and which mechanisms are
baked in, and labels them separately.

It reads the token by probing. The ABI is a list of getter names from this
template and a few other token families (`isReward`, `isBlacklisted`,
`_blackList` and so on). No token implements all of them. Each one is called,
and a revert counts as an answer: that power isn't there.

Flags only tell you which powers exist. To check whether a sell would go
through right now, the page runs an `eth_call` with a state override. It gives the
wallet a balance it doesn't have, tries the transfer to the pair, and checks
whether the EVM reverts. To find which storage slot holds balances, it tries
the first 32 slots: compute `keccak256(address . slot)`, override it, read
`balanceOf` back, and keep the slot that returns the fake number. It's the same
trick as `deal()` in a Foundry fork test, except the RPC does it. Nothing gets
signed or broadcast.

## What it can't do

It probes by name. A token that hides the same power behind a name that isn't
on my list looks clean. Slot discovery fails on proxies and unusual storage
layouts. Those return "could not determine" rather than a false reassurance.
And it can't tell a multisig or a timelock owner from one person's key.

So a green result only means my probes found nothing. That's a much weaker
statement than "this token is safe", and anyone using the page should read it
that way.

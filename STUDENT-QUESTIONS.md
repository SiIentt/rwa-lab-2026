# STUDENT-QUESTIONS.md — Discussion questions (goes inside your submission zip)

Answer directly under each question. 150–300 words each — **reasoning over length**.

The sections follow the four planks of the bridge. There are no standard answers; these are the
real point of the lab.

---

## A. Custody — where the asset actually is (plank a)

**A1.** In Lab 1, `totalCollateral()` read the vault's own ERC-20 balance, and the chain could
prove the invariant. Here, `MockTBillCustodian.realHoldings()` is just a number that a permissioned
address can move. Write the equivalent of `totalCollateral() == totalSupply()` for *this* system.
What does it actually assert, and what does it no longer prove?

> Your answer:
> Before considering outstanding redemption requests, a simplified backing relationship would be: `vault.reserveBalance() + custodian.realHoldings() == vault.totalClaimValue()`. Both sides use six-decimal asset units. The left side represents the vault’s cash plus the reported value of the T-Bills. The right side represents the outstanding shares valued at the reported NAV. This is more appropriate than directly comparing assets with `totalSupply()`, because shares have eighteen decimals and their value changes with NAV.
>
> However, this equality only shows agreement between recorded numbers. The vault’s USDC balance is observable on-chain, but `realHoldings()` is a figure entered by an authorized address. The NAV also depends on a reporter. Matching these figures does not prove that the T-Bills exist, belong to the fund, are free from other claims, or can be sold at the stated value.
>
> Outstanding redemptions require more careful accounting. Pending tickets have payouts fixed at enqueue time, while settled claims are backed by cash in the queue. Therefore, a complete reconciliation must distinguish circulating shares, pending redemption liabilities, and settled claims. Compared with Lab 1, numerical consistency remains useful, but it no longer provides independent proof of the underlying assets.

<br><br><br>

**A2.** The T-Bills are held by an SPV, a legal entity set up so the fund's assets are bankruptcy-
remote from the issuer. Explain in your own words why a *legal* structure is part of a *technical*
design. If the SPV's custodian goes bankrupt, what does the on-chain token entitle its holder to?

> Your answer:
> A legal structure is part of the technical design because the smart contracts cannot directly own, recover, or enforce rights over physical financial assets. The token records a position on-chain, while the SPV and custody agreements determine what that position means outside the blockchain. Without an enforceable connection between them, a technically correct token may provide little practical protection.
>
> The SPV is intended to separate fund assets from the issuer’s ordinary business and creditors. This reduces the risk that the issuer’s failure brings the fund’s assets into the same insolvency process. However, bankruptcy remoteness is not a guarantee against every loss, especially if asset segregation, ownership records, or custody arrangements are defective.
>
> If the custodian fails, the holder’s rights depend on the fund documents and applicable law. The holder may have a beneficial interest or a contractual claim through the fund, rather than direct ownership of a particular Treasury bill. Properly segregated assets may be recovered or transferred through the legal process, but access can still be delayed. The token alone does not establish payment priority or guarantee immediate redemption. This is why custody records and legal documentation must support the on-chain design.

<br><br><br>

---

## B. Attestation — how the chain learns the truth (plank b)

**B1.** `attest()` is the only function that moves the NAV, and it is gated by `REPORTER_ROLE`.
Everything downstream — subscription pricing, redemption payouts, both invariants — believes that
number. Name the failure modes of a single trusted reporter, and one mechanism (technical,
financial, or legal) that mitigates each.

> Your answer:
> A single trusted reporter creates several failure modes. First, the signing key may be stolen. An attacker could then submit a false NAV while appearing to be an authorized operator. Hardware-backed key storage, limited signing access, and a multisignature approval process would reduce this risk. An emergency key-revocation procedure would help contain an incident.
>
> Second, the reporter may deliberately manipulate the NAV for personal profit or cooperate with another participant. Independent valuation sources, external audits, and separation between trading and reporting duties would make such conduct harder. Legal liability and financial penalties could provide additional incentives for honest reporting.
>
> Third, an honest reporter may make a calculation or decimal error. Automated validation, comparison with independent reference prices, and review of unusually large changes could catch mistakes before they affect subscriptions and redemptions. These checks should allow investigation rather than assuming every large movement is fraudulent.
>
> Finally, the reporter may become unavailable and stop updating prices. A freshness limit, backup reporting arrangements, and a controlled suspension of price-sensitive operations would address this problem. Access control establishes who submitted a number; it does not establish whether that number is accurate or timely.
<br><br><br>

**B2.** A production feed would check whether `updatedAt` is stale; this lab's mock does not. A
**frozen** NAV is a different attack from a **wrong** NAV. Describe how a stale-but-honest NAV can
be exploited by someone who knows the true value has moved.

> Your answer:
> A stale NAV can be exploited even when it was correct when published. The problem is that the contract continues offering transactions at an old price while an informed participant knows the current economic value. This gives the participant an advantage over holders who rely on the displayed NAV.
>
> Suppose the published NAV remains at $1.00 while the true value rises to $1.10. An informed user can subscribe at the lower price and receive more shares than the current asset value would justify. When the feed updates, those shares are valued at the higher NAV. This can transfer value from existing holders to the new subscriber through dilution.
>
> The opposite strategy applies when the true value falls. A holder can enqueue a redemption while the published NAV is still too high. Because the queue locks the payout at enqueue time, a later correction does not reduce that ticket’s promised payment. Remaining holders may bear the resulting shortfall.
>
> A production system should check the age of each update before accepting subscriptions or new redemption requests. It could also use independent price checks and defined valuation windows. However, a timestamp alone is insufficient: a recently submitted value can still be wrong.

<br><br><br>

---

## C. Redemption — T+1 does not settle on-chain (plank c)

**C1.** The queue locks the payout at the NAV of **enqueue** time, not **settle** time. If the NAV
rises between the two, who gains and who loses — the redeemer, or the holders who stayed? Is that
the right allocation of risk, and how would you change it?

> Your answer:
> Locking the payout at enqueue time gives the redeemer certainty about the amount requested. If NAV rises before settlement, the redeemer receives the original locked amount and gives up the later increase. Assuming that increase reflects real asset gains and the fund accounts for fixed redemption liabilities correctly, the economic benefit remains with the fund and its continuing holders. The lab’s reported NAV alone does not automatically distribute that benefit.
>
> If asset values fall instead, the redeemer still has a fixed claim, so continuing holders may absorb the loss needed to meet that claim. I consider this arrangement reasonable only when the valuation cutoff, settlement delay, and associated risks are clearly disclosed. An alternative would price redemptions at a defined dealing cutoff after requests arrive, with transaction costs charged fairly to redeeming investors. This would reduce opportunities to choose a favorable old price.
>
> The relationship `vault.reserveBalance() >= queue.pendingAssets()` cannot always hold because pending redemptions may be backed by T-Bills rather than immediately available USDC. The fund must sell assets and receive the proceeds before paying. A higher NAV does not create additional cash in the vault. In contrast, amounts already marked as claimable must be backed by cash held in the queue.
<br><br><br>

**C2.** In a 2008 money-market fund breaking the buck, and in USDC's 2023 depeg, redemptions were
handled very differently. Compare the two. For a tokenized T-Bill fund, what does it mean to "close
the redemption channel", and what should happen to the queue when it does? Then look at the lever
this lab actually ships: `pause()` (`PAUSER_ROLE`) freezes transfers, minting **and** redemption
together — one switch, no partial setting. Why is that the wrong tool for a liquidity squeeze, and
what would you rather have?

> Your answer:
> In 2008, the Reserve Primary Fund’s NAV fell to $0.97 after losses associated with Lehman Brothers. Redemptions were suspended, and the fund entered liquidation. Investors faced delayed recovery from the fund’s remaining assets. In March 2023, USDC traded below one dollar amid uncertainty about reserves held at Silicon Valley Bank. After protection for the bank’s deposits was announced, Circle resumed processing redemptions on Monday, March 13. USDC’s market price recovered rather than entering a comparable fund liquidation.
>
> For a tokenized T-Bill fund, closing the redemption channel means restricting the conversion of shares into cash. The system should specify whether this blocks new requests, settlement, or both. Existing tickets should remain recorded, and their order and locked amounts should not be changed silently.
>
> The lab’s `pause()` is too broad for an ordinary liquidity shortage because it blocks token transfers and minting, as well as redemption operations requiring share transfers or burns. However, the current queue’s `claim()` transfers USDC directly, so the token pause does not itself block that final withdrawal.
>
> I would prefer separate controls for subscriptions, new redemption requests, and settlement. Already funded claims should normally remain withdrawable. Any restriction should have a stated reason, review process, and reopening conditions.
<br><br><br>

---

## D. Admission — the guest list and the backdoor (plank d)

**D1.** Every balance change in `TBillToken._update` checks **both** endpoints against the
whitelist. That is what stops a sanctioned address from receiving shares — and also what stops a
completely ordinary user who has not finished KYC. Where is the right line for a regulated fund,
and who should be able to move it?

> Your answer:
> A regulated fund should define eligibility according to its legal obligations and disclosed investment terms. Checking both endpoints helps prevent an approved holder from transferring shares to someone who has not completed the required checks. However, an ordinary applicant awaiting verification should not be treated as if they have committed wrongdoing. The system should distinguish an incomplete application from a confirmed restriction.
>
> The appropriate boundary is to collect enough information to establish eligibility without creating unnecessary barriers or exposing personal data. Identity documents should normally remain off-chain, while the contract stores only the approval status needed to enforce transfers. Clear processing times, explanations for rejection where permitted, and a correction or appeal process would improve fairness.
>
> A designated compliance team should manage individual approvals under a documented policy. Changes to the policy itself should require broader governance and legal review. Sensitive actions, such as removing an existing holder, should involve additional authorization and an audit trail.
>
> Investors should also understand that whitelist status can affect access to their holdings. Rules for expiration, mistaken removal, and emergency restrictions should be published in advance. Technical access control is useful only when the people operating it are accountable.
<br><br><br>

**D2.** A regulated fund asks for two levers: freeze an address (`setWhitelisted(addr, false)`) and
burn a balance (`MINTER_ROLE`) — both censorship tools, both justified by the product. But Ex4.3
shows the two **collide**: once an address is off the list, `burn()` reverts as well, because
`_update` guards both endpoints. So a freeze also blocks a seizure. Argue **both** sides of handing
an issuer these powers, then say what safeguards you would add and who should hold each key.

> Your answer:
> There are legitimate reasons to give a regulated issuer freezing and seizure powers. These tools may support compliance with legal orders, responses to stolen assets, and corrections following proven fraud. Without them, a fund could be unable to meet obligations that apply to the underlying securities.
>
> However, these powers also expose investors to abuse, mistakes, and compromised administrator keys. A freeze can prevent an innocent holder from transferring or redeeming. A forced burn is more serious because it removes the on-chain position and could undermine the holder’s ability to demonstrate their claim. Investors therefore need clear information about when these powers can be used and what rights remain afterward.
>
> Ex4 shows that the current controls conflict: removing an address from the whitelist also prevents burning its shares. Both operations pass through `_update()`, which checks the affected nonzero address. Granting `MINTER_ROLE` does not bypass this check.
>
> I would assign routine whitelist management to compliance staff, while requiring a separate multisignature group for forced asset actions. That group should include independent oversight and require documented authority. A narrowly defined seizure procedure should preserve records and specify the legal destination of the value. Simply re-whitelisting an address to burn its balance would be an unclear and risky workaround.
<br><br><br>

---

## E. Tests (Tier 1 required — this is Ex2, Ex3 and Ex4)

Turn the red tests green in `test/exercises/01_BridgeTasks.t.sol` to cover the scenarios below, and
write your test function names here:

| Plank | Scenario | Your test function name |
|---|---|---|
| a | Shares exist while the custodian holds nothing | `test_Ex2_SharesExistWhileTheCustodianHoldsNothing` |
| a | `realHoldings()` moves with no cash moving | `test_Ex2_RealHoldingsIsJustANumber` |
| b | The claim value floats while the share count stays | `test_Ex3_TheClaimFloats_TheShareCountDoesNot` |
| b | One `attest` re-prices every holder at once | `test_Ex3_OneCallMovesTheWholeBook` |
| d | A non-whitelisted address cannot receive shares, even by paying | `test_Ex4_NoKyc_NoShares_EvenIfYouPay` |
| d | A whitelisted holder cannot send shares to a non-whitelisted address | `test_Ex4_TransferChecksBothEndpoints` |
| d | Removing an address freezes that holder — and blocks a burn | `test_Ex4_FreezeBeatsConfiscation` |

Then write one more scenario you consider **most likely to be exploited** on a tokenized fund, and
say which plank it breaks:

> Your answer:
> I consider misuse of the reporter’s key a particularly serious attack scenario because one authorized NAV update can affect every subscription and redemption. A whitelisted attacker who controls that key could report an artificially low NAV and subscribe with a small amount of USDC. The low price would give the attacker an excessive number of shares. The attacker could then restore the original NAV and enqueue those shares for redemption at the higher value.
>
> If the vault has sufficient cash and the authorized settler processes the request, the attacker could withdraw more USDC than they originally contributed. The additional cash would come at the fund’s expense, potentially reducing the resources available for honest holders. Settlement access and available liquidity therefore affect whether the attack can be completed.
>
> This primarily breaks the attestation plank, because an authorized price is accepted without independent evidence. The queue’s two invariants could still hold throughout the attack: escrowed shares could match pending shares, and settled claims could remain fully funded. Correct internal accounting does not establish that the price used to create those claims was fair.
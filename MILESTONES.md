# Finto on Stellar: milestones and budget

Three tranches from December 15, 2026 to April 14, 2027. Total budget: $104,800 (tranche 1: $32,700; tranche 2: $47,500; tranche 3: $24,600). Components carry the names used in the architecture. The VASP license, the banking agreements and the PSP contract are existing operating assets: no deliverable depends on a regulatory or commercial step.

---

## Tranche 1 — MVP: Stellar treasury, reconciliation and Bridge implemented on testnet ($32,700)

### 1.1 TreasuryService and ReconciliationEngine

**Description.** The three treasury accounts on testnet (receiving, reserve with two of three signers, cold with three of five) with their USDC trustlines, attribution of every incoming payment by asset, amount and memo, the daily sweep from the receiving account to the reserve, and automated three-way matching (funding event, Stellar hash, payout reference) with an exceptions queue and a daily report. The report can be re-derived by a public script from the accounts' history and an export of the transfer id and payout reference pairs. In this tranche the payout leg is simulated by a reference generator; it is wired to the real PSPDealerAdapter in deliverable 2.2.

**How to measure completion.**
- The three accounts are created on testnet, addresses published in the repository, the signing thresholds of the reserve and cold accounts readable on the explorer.
- 100 attributed test settlements: every incoming payment carries a memo and is matched to an internal transfer, zero orphans, hashes listed in the report.
- The daily report is generated automatically and re-derived identically by the public script from the accounts' testnet history and the pairs export, procedure documented in the repository.
- The three discrepancy classes (funding without settlement, settlement without payout, payout without settlement) are injected by the test suite and all routed to the exceptions queue, and a replayed payout order produces a single payment, public tests in CI.

**Budget.** $12,900

**Delivery date.** December 30, 2026

### 1.2 Bridge integration

**Description.** Bridge customer creation and hosted KYC link at onboarding, per-sender ACH and wire virtual account, webhook ingestion with signature verification and idempotency, USDC settlement on the Stellar receiving account with the virtual account's memo and the payment reference, final confirmation by the observed transaction and not by the webhook.

**How to measure completion.**
- Full flow in the Bridge sandbox: virtual account creation, simulated deposit, USDC settled on the testnet receiving account, for 20 consecutive transfers with no attribution failure, hashes published.
- A replayed webhook produces exactly one state transition, public test in CI.
- A settlement with no memo or in another asset is routed to the exceptions queue and never paid out, public test.

**Budget.** $9,900

**Delivery date.** January 10, 2027

### 1.3 QuoteEngine and TransferOrchestrator

**Description.** Firm USDC to KGS quote with expiry (SEP-38 semantics, the same engine will serve the anchor) and the transfer state machine (`quoted`, `funded`, `settled`, `payout_pending`, `completed`, `exception`), which issues a payout order only with a confirmed Stellar settlement hash and valid compliance statuses. Append-only event log.

**How to measure completion.**
- Every transition of the state machine is covered by public automated tests in CI.
- Over 50 test transfers, the rate applied to the payout order is the quoted rate, and every expired quote is rejected, public tests.
- Without a confirmed settlement hash or with an invalid compliance status, no payout order is issued, public negative tests.

**Budget.** $9,900

**Delivery date.** January 21, 2027

---

## Tranche 2 — Testnet: Privy, Mercuryo, KGS payout and Anchor Platform, full corridor ($47,500)

### 2.1 Privy wallets and StellarTxBuilder

**Description.** Embedded Stellar wallets by email or social login with Privy (tier 2: creation and raw signing), and the transaction-building module: account creation and USDC trustline sponsored by the platform account, USDC payments with memo wrapped in a fee-bump, signing in the Privy enclave, submission through Stellar RPC with sequence, status and idempotency handling. The sender never holds XLM.

**How to measure completion.**
- For 20 new test accounts, the Stellar account and the USDC trustline are created in a sponsored transaction and the first USDC payment is signed and confirmed, hashes published, sponsor and fee payer readable on the explorer.
- 50 USDC payments to the receiving account with memo, submitted by StellarTxBuilder, with no sequence failure and no double submission, hashes published.
- The RPC `TRY_AGAIN_LATER`, `DUPLICATE` and `ERROR` cases are covered by public tests checking that no failure is concluded without `getTransaction`.

**Budget.** $10,900

**Delivery date.** February 3, 2027

### 2.2 PSPDealerAdapter

**Description.** Adapter for the PSP's SOAP dealer protocol: `ValidatePaymentEx2` then `MakePaymentEx2`, payment registry in the database for exactly-once execution, tracking by `dealer_guid` with increasing delay, stuck-payments queue with alerting, `payout(phone, amount_kgs, transfer_id)` interface consumed by the TransferOrchestrator and, in deliverable 2.4, by the anchor.

**How to measure completion.**
- 50 payouts executed in the PSP sandbox with `dealer_guid` capture and reconciliation closure in the ReconciliationEngine, report published.
- A replayed payout order produces exactly one PSP payment, public tests in CI.
- A simulated PSP timeout routes the payout to the stuck queue with an alert in under five minutes, public test.

**Budget.** $9,900

**Delivery date.** February 14, 2027

### 2.3 Mercuryo, KYCService and full corridor dress rehearsal

**Description.** Mercuryo widget delivering the card payment's USDC to the receiving account with the transfer memo, failover when Bridge is unavailable; recipient KYCService at SumSub in SEP-12 format (polling, callbacks, deletion), share-token recognition with Mercuryo, and structural enforcement in the orchestrator; full corridor dress rehearsal on testnet across the three funding methods; STRIDE threat model and monitoring plan published.

**How to measure completion.**
- In the Mercuryo sandbox, 10 card payments arrive as USDC on the receiving account and trigger a payout order, the fees shown at quote time equal to the executed amounts, hashes published.
- With the Bridge path disabled in staging, a transfer completes through the card, test recorded in the repository.
- A payout order is refused for missing or expired recipient KYC and for an invalid sender compliance status, with an entry in the event log, public tests.
- Dress rehearsal: 100 end-to-end transfers on testnet across the three funding methods, three-way matching closed for each, report with explorer links published in the repository.
- The STRIDE threat model and the monitoring plan are published in the repository.

**Budget.** $11,900

**Delivery date.** February 28, 2027

### 2.4 KGS anchor on Anchor Platform

**Description.** The Anchor Platform deployment and the SEP-1 and SEP-10 configuration start in parallel with deliverables 2.1 to 2.3, by the second engineer; the date below is the completion date. Deployment of the Anchor Platform in the receiving anchor role, wired to the QuoteEngine and the PSPDealerAdapter through the business server: `stellar.toml` (SEP-1) with a dedicated signing key, SEP-10, SEP-12 served by the KYCService, firm SEP-38 quotes through the rate callbacks with a cap on open quotes per account, SEP-31 receive flow with the platform's observer as source of truth for the received payment, idempotent event consumer, refund through `notify_refund_sent` on failure.

**How to measure completion.**
- `stellar.toml` is live at `finto.kg/.well-known/` and the `stellar-anchor-tests` suite passes with no error on SEP-1, 10, 12, 38 and 31 against the testnet deployment, output published in the repository.
- 25 SEP-31 transactions run by a reference client we operate, from quote to KGS payout in the sandbox, each reconciled, hashes published.
- Firm quotes are honored at execution and expired quotes rejected across all test transactions; a payment whose amount differs from the quote is refunded, public tests.
- A platform event delivered twice produces a single transition and a single payout, public test in CI.

**Budget.** $14,800

**Delivery date.** March 17, 2027

---

## Tranche 3 — Mainnet: Bridge, Mercuryo, Privy and Anchor Platform in production, evidence and adoption ($24,600)

### 3.1 Production launch of the corridor and the anchor

**Description.** The three treasury accounts on mainnet with their signers and the custody service, Bridge and Mercuryo in production configuration, the PSP dealer account active, Anchor Platform and `stellar.toml` on mainnet, StellarTxBuilder and Privy accounts in production, first rebalancing cycle through the partner banks, and the corridor evidence: real transfers across the three funding methods and over SEP-31, all reconciled.

**How to measure completion.**
- The three treasury accounts are public on mainnet, signing thresholds readable on the explorer; a rebalancing cycle to a partner bank is executed and its transactions documented.
- 30 real transfers, across the three funding methods, each with its funding reference, its mainnet hash and its `dealer_guid`, matching closed for all, report published.
- 3 SEP-31 transfers sent by our reference client from a sending account distinct from the treasury accounts, completed on mainnet through the anchor, hashes published.

**Budget.** $10,900

**Delivery date.** March 29, 2027

### 3.2 Production monitoring, runbook and public evidence page

**Description.** Stall alerts per state (settlement with no payout order, payout with no `dealer_guid`, SEP-31 transaction waiting for funds), account balances against the payouts expected over 24 hours, stuck-payments queue, SEP endpoint health, SEP-10 signature failures, `stellar-anchor-tests` on a regular schedule; runbook (orphan funds, compromised key, PSP unavailable, SEP-10 rotation); public evidence page listing the transactions and the daily reports for the period, with the independent verification script and its run by a third party outside the team.

**How to measure completion.**
- The alert configuration is published in the repository and one alert of each class has been triggered and resolved in staging, traces published.
- The runbook is versioned in the repository and the "orphan funds" procedure has been rehearsed on testnet, refund hashes published.
- The public evidence page is live, open to anyone, and at least 5 mainnet transfers carry the reconciliation re-derivation done by a third party outside Orbit Labs, with their attestation.

**Budget.** $7,900

**Delivery date.** April 7, 2027

### 3.3 Open-source release of the PSPDealerAdapter and anchor integration guide

**Description.** Public repository of the PSPDealerAdapter under the Apache-2.0 license, with its dealer protocol documentation, and an integration guide for the KGS anchor from a SEP-31 sender's point of view (discovery, authentication, quote, KYC, payment), reproduced by a developer who did not write the adapter.

**How to measure completion.**
- The public repository is live with its CI green, a tagged release and a README describing the dealer protocol.
- The integration guide is published and a clean-room run by a developer who did not write the adapter completes a test SEP-31 transaction following only the public instructions, log published.
- The request to list the anchor in the ecosystem directories is filed, link published.

**Budget.** $5,800

**Delivery date.** April 14, 2027

## Tranche #3 Onchain Metric Information

**1. What metric do you plan to measure?** We measure two things on the Stellar network, from the three published treasury accounts (receiving, reserve, cold): the number of distinct Stellar accounts sponsored by our platform account for senders (CAP-33 footprint), holding a USDC trustline, whose transfer settled on mainnet, and the number of USDC settlement transactions attributable to those accounts. Both are read from the public ledger, over the window from the acceptance of this application to the tranche 3 submission. Our own accounts and test accounts are excluded.

**2. What is your completion target?** By the tranche 3 submission: 50 distinct sponsored sender accounts and 100 USDC settlement transactions attributable to the published treasury accounts.

**3. How will you track and report it?** The public evidence page (deliverable 3.2) lists the treasury accounts and the script that re-derives both counts from the ledger, so anyone can check them without asking us. We publish the running count every week on that page and a final report, with explorer links, in the tranche 3 submission. A shortfall is reported as such, with the reasons.

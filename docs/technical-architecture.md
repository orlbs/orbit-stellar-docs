# 5. Technical Architecture

**Orbit Pay** uses a modular payment-orchestration architecture with USDC on
Stellar as the shared settlement layer. Bridge provides USD and EUR entry through
supported US and SEPA rails; Orbit Labs and its banking partners provide KGS
payout; and Abroad and Etherfuse add supported LATAM and Mexico flows in staging.
Orbit Pay maintains a normalized transaction state, double-entry ledger,
reconciliation process, compliance gates, and operational monitoring across
providers. The grant culminates in production US/EU-to-Kyrgyzstan payments.

## Application-field summary

Orbit Pay uses a modular payment-orchestration architecture with USDC on Stellar
as the shared settlement layer. Bridge provides USD and EUR entry through
supported US and SEPA rails; Orbit Labs and its banking partners provide KGS
payout; and Abroad and Etherfuse add supported LATAM and Mexico flows in staging.
Orbit Pay maintains a normalized transaction state, double-entry ledger,
reconciliation process, compliance gates, and operational monitoring across
providers. The grant culminates in production US/EU-to-Kyrgyzstan payments.
Full architecture: https://github.com/orlbs/orbit-stellar-docs/blob/main/docs/technical-architecture.md

---

## 5.1 Orbit Pay web application

The responsive web application provides:

- Customer registration and authentication
- Corridor eligibility and availability
- Sender and recipient details
- Transfer amount and quote presentation
- Disclosure of fees, foreign-exchange rate, and expected recipient amount
- Payment instructions
- Transfer status and support escalation
- Administrative views for operations, compliance review, and reconciliation exceptions

The interface does not require customers to connect a blockchain wallet.
Blockchain accounts and treasury activity are managed within Orbit Pay’s
regulated operating model.

---

## 5.2 API and payment orchestration layer

The orchestration service is the system of record for payment workflow state. It:

- Selects an eligible corridor and provider route
- Retrieves and timestamps quotes
- Creates customer, beneficiary, and provider objects
- Initiates fiat collection and payout instructions
- Consumes provider webhooks
- Applies idempotency controls to prevent duplicate instructions
- Normalizes provider-specific states into a common Orbit Pay state machine
- Associates fiat events, Stellar transactions, conversions, and payouts with one transfer identifier
- Routes failed or ambiguous transactions to an operations queue

### Recommended normalized states

```
created → compliance_pending → funding_pending → funded → conversion_pending
  → stellar_submitted → stellar_confirmed → payout_pending → completed
```

### Exception states

`manual_review`, `failed`, `expired`, `returned`, and `reversed`.

---

## 5.3 Compliance and corridor gating

The compliance layer determines whether a customer, beneficiary, corridor,
amount, and purpose are eligible before money moves. It should support:

- Customer and beneficial-owner verification where applicable
- Sanctions, PEP, and adverse-media screening through the selected compliance provider
- Transaction limits by customer, corridor, and risk level
- Source-of-funds or enhanced-due-diligence workflows when required
- Record retention and auditable compliance decisions
- Market feature flags so a corridor cannot be activated before legal and operational approval

The US and EU entry routes must be activated only for customer types,
jurisdictions, payment purposes, and funding methods approved by Bridge and
Orbit Labs’ compliance policies. Kazakhstan remains outside the grant’s
production scope and must stay disabled until the relevant licence and payout
arrangements are confirmed.

---

## 5.4 Bridge integration

Bridge will support both US and EU fiat-entry flows. Orbit Pay will integrate:

- Customer creation and required KYC/KYB data exchange
- Virtual-account creation
- USD funding instructions, including supported ACH or wire methods
- EUR funding instructions, including supported SEPA virtual-account or virtual-IBAN methods
- Destination configuration for USDC on Stellar
- Stellar destination address and memo handling
- Payment-status webhooks and event persistence
- Idempotent request handling
- Mapping of Bridge transfers to Orbit Pay transfer records
- Reconciliation of received USD or EUR, fees, converted USDC, and Stellar delivery

Bridge’s official documentation shows USD and EUR virtual accounts, EUR funding
through SEPA, and virtual-account destinations configured for USDC on Stellar
with an optional Stellar memo. Final availability is still subject to commercial
onboarding, customer eligibility, geography, compliance approval, and the
executed Bridge agreement.

---

## 5.5 Stellar settlement service

Stellar is the common settlement network rather than a user-facing speculative
asset layer. The settlement service will:

- Maintain an allowlisted representation of Stellar USDC, including issuer verification
- Construct and sign payment transactions using a supported Stellar SDK
- Submit transactions through Stellar RPC or an approved infrastructure provider
- Track ledger inclusion and final payment state
- Record transaction hash, source and destination accounts, asset, amount, fees, memo, ledger, and timestamps
- Support retries without creating duplicate payments
- Reconcile on-chain movements against the internal ledger and provider events
- Separate testnet and mainnet configuration and signing authority
- Alert operators to failed, delayed, duplicated, or unmatched transactions

The architecture should not depend exclusively on public Horizon. Current Stellar
documentation identifies RPC as the recommended API for real-time network access
and states that Horizon is moving toward deprecation. Orbit Pay may use Horizon
payment streams during the transition or an indexed infrastructure provider, but
it should retain its own durable transaction and reconciliation records.

---

## 5.6 Internal ledger and reconciliation

Orbit Pay should maintain a double-entry internal ledger independent of external
provider dashboards. Each transfer records:

- Customer funding amount and currency
- Provider fees
- Foreign-exchange rate and spread
- USDC amount received and sent
- Stellar network fee
- Local payout amount and currency
- Partner payout fee
- Orbit Pay revenue
- Refunds, reversals, and manual adjustments

Automated reconciliation compares Bridge events, Stellar ledger activity, Abroad
or Etherfuse events, and local bank payout reports. Any mismatch becomes an
exception with an owner, reason, and resolution status.

---

## 5.7 Kyrgyzstan payout integration

For both production routes into Kyrgyzstan, Orbit Labs will:

- Receive confirmed USDC settlement
- Apply the approved USDC-to-KGS treasury and conversion process
- Create a payout instruction for the appropriate banking partner
- Track acknowledgement, acceptance, completion, rejection, or return
- Match the bank payout reference to the Orbit Pay transfer and Stellar transaction
- Provide customer support with a complete payment timeline

The final architecture document should describe whether bank connectivity uses
an API, host-to-host file exchange, or an operational process. Do not imply
real-time API payout if the current bank arrangement is manual or file-based.

Current production payout connectivity for Kyrgyzstan uses a dealer/PSP API
integration (validate → pay → status by unique payment reference), with Orbit
managing own-book USDC→KGS treasury conversion. See
[`tmd-psp-own-book-integration.md`](./tmd-psp-own-book-integration.md) for the
detailed integration description.

---

## 5.8 Kazakhstan post-grant roadmap

Outside the grant’s required deliverables, Orbit Pay’s architecture may later
support:

- KZT corridor configuration
- Beneficiary and banking-field requirements
- Corridor-specific compliance and transaction limits
- Payout-adapter interface and test fixtures
- Production feature gating
- Operational and reconciliation procedures

Production activation is conditional on licence approval and a confirmed payout
arrangement. Kazakhstan work must not be used to measure completion of any SCF
tranche in this application.

---

## 5.9 Abroad integration

Subject to confirmed API access and contractual coverage, the Abroad adapter
will:

- Retrieve supported corridor and quote information
- Validate beneficiary and destination fields
- Initiate supported payout requests
- Process asynchronous status events
- Normalize provider errors and payout states
- Reconcile settlement and payout amounts

Before submission, attach official documentation or partner confirmation showing
the exact supported countries, local rails, Stellar settlement path, sandbox
access, and production-onboarding requirements. Public sources did not provide
enough detail to verify every Abroad claim contained in the reviewer feedback.

---

## 5.10 Etherfuse integration

The Etherfuse adapter will:

- Request USDC/MXN quotes
- Record quote expiry, rate, fees, and expected MXN amount
- Initiate supported conversion or offramp requests
- Track settlement and payout status
- Reconcile USDC debits with MXN settlement outcomes
- Normalize errors into Orbit Pay’s common operations workflow

Etherfuse publicly demonstrates an offramp quote using USDC on Stellar as the
source asset and MXN as the target asset. The application should still
distinguish a verified production capability from a planned integration.

---

## 5.11 Security and operational controls

Before mainnet, Orbit Pay will document and test:

- Wallet and signing-key ownership
- Key storage and signing policy
- Role-based access for treasury and operations
- Dual approval for high-risk treasury movements
- Secret storage and rotation
- Webhook authentication and replay protection
- Idempotency for every money-movement instruction
- Encryption of sensitive customer data
- Separation of personally identifiable information from public blockchain data
- Monitoring, alerting, backups, incident response, and audit logs
- Daily reconciliation and documented exception resolution

---

## 5.12 Architecture dependencies

The application should state these dependencies transparently:

- Bridge production onboarding and approval
- Abroad sandbox access and confirmation of supported rails
- Etherfuse sandbox and production eligibility
- Bank payout connectivity in Kyrgyzstan
- Bridge approval for both US and EUR/SEPA customer funding flows
- Final custody and transaction-signing design

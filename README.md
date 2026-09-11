# cloud-itonami-isic-6311-identity

**Open Business Blueprint for ISIC Rev.4 6311** (data processing, hosting and
related activities), narrowed to **hosting an identity service** — the Clerk /
Auth0 / WorkOS class of business — published as an OSS business any qualified
operator can fork, deploy, run, improve and sell.

**Nothing here is on sale.** Every plan is `:status :proposed` and
`pricing/sellable?` answers `false` today. Three of the six gates in
ADR-2608110200 決定 4 are open. Numbers exist so they can be argued with.

## This repository is the business, not the service

The service is [`authn.kotobase.net`](https://authn.kotobase.net)
(`network-awai/net-kotobase`), and the DID authority stays
`did:web:kotobase.net`. Three layers, and the dependency points one way:

| | | |
|---|---|---|
| **authority** | `did:web:kotobase.net:*` | never moves (ADR-2608039950 決定 1) |
| **service** | `authn.kotobase.net` | infrastructure; knows nothing about money |
| **business** | this repo | contracts, price, metering, invoices |

ADR-2608039950 once rejected cloud-itonami as a home for authentication —
*「認証は事業ではなくインフラで…依存の向きが逆転する」*. That objection is to
burying the **service** here, not to selling it. The blueprint depends on
authn; authn does not know this repository exists.

## The price is data, and the invoice is a pure function

`pricing.edn` is the only place a number lives.
`cloud-itonami.identity.pricing` multiplies a recorded measurement by it and
stops. A price change is a one-file edit, never a deploy of the thing people
sign in through — which is also what let the meter start before the price was
decided. That order mattered: **a price can be chosen later, and a month that
was not measured can never be measured afterwards.**

```clojure
(pricing/invoice book :growth [{:month "2026-06" :active-users 10000}
                               {:month "2026-07" :active-users 11000}])
;; => {:invoice/total 70.0
;;     :invoice/lines [{:line/month "2026-06" :line/measured {:mau 10000} :line/amount 25.0 …}
;;                     {:line/month "2026-07" :line/measured {:mau 11000}
;;                      :line/items [{:item/dimension :mau :item/quantity 1000
;;                                    :item/unit-rate 0.02 :item/amount 20.0}] …}]
;;     :invoice/sellable? false}
```

Five refusals are load-bearing, and each one is a way invoicing goes wrong
without erroring:

- **Only measured dimensions may be priced.** `authn.usage` records `:mau` and
  `:verification`. A price book naming anything else is refused whole rather
  than partially honoured — quoting a number nobody counted is worse than
  refusing to quote.
- **Every amount cites its count.** A line carries the month, dimension and
  quantity it came from. An amount a customer cannot trace back is an amount
  they will dispute at renewal, which is the most expensive moment to find out.
- **An unmeasured month is not a zero month.** They look identical on a bill
  and mean opposite things: one says nobody used the service, the other says we
  do not know. Collapsing them turns a metering outage into a credit note six
  months later.
- **A free tier cannot bill.** A plan with a hard cap carries no overage rate.
  Past the cap the answer is "choose a plan", not a surprise invoice.
- **No charging while a gate is open.** `pricing/sellable?` answers from data,
  so nobody has to remember.

## What is actually running

| | |
|---|---|
| passkey / WebAuthn, email OTP + magic link, Google & GitHub, enterprise OIDC SSO, did:key CACAO | live |
| **TOTP + recovery codes**, per-account opt-in | live 2026-08-11 |
| organizations, roles, invitations, service accounts, SCIM 2.0, audit log | live |
| immediate revocation, sign-out-everywhere | live |
| **usage metering** (`:mau` de-duplicated at write time, `:verification`) | live 2026-08-11 |
| custom domains per customer | implemented, **not enabled** |
| risk-based step-up, SMS, SAML, data residency | not offered |
| drop-in UI components, SDKs, webhooks, admin dashboard | **do not exist** |

The measured comparison against Clerk — 36 features in three states, running /
parts-exist-unwired / absent — is
`90-docs/comparison/clerk-parity.datoms.edn` in the superproject.

## What a buyer is told before choosing

`pricing.edn` carries the disclosures beside the numbers, because a disclosure
that lives somewhere else drifts from what is being sold. Two are marked
must-read:

> **Sign-in keys are held by this service and unlocked by your passkey — they
> are not a wallet.** If this service is compromised, an attacker can sign as
> any user. A hardware key boundary (KMS/HSM) is not implemented. Customers who
> require sole custody should use the did:key CACAO path, where the key never
> leaves them.

> **No SOC 2, ISO 27001 or HIPAA certification.** Buyers with a certification
> requirement should not select this service.

A third is not a warning but a design consequence worth stating: sessions are
opaque tokens verified centrally, not signed JWTs. That is what makes
revocation immediate — and it means there is no offline verification and no
JWKS endpoint.

## Build

```bash
kbb -M:test    # 10 tests / 38 assertions
```

Zero runtime dependencies on purpose: this code multiplies a measurement by a
number and has to be readable by anyone deciding whether the number is right.

## Licence

AGPL-3.0-or-later. Operator: AWAI Network, L.L.C.; infrastructure and software
supplied by Gftd Japan 株式会社.

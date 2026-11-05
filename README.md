# Loyalty Membership Platform — Case Study

A written case study of a production membership and voucher-redemption system I worked on as a full-stack developer for a Hong Kong lifestyle group.

> **No source code is published here.** The codebase is private and owned by the client and the agency that built it. This repository describes the system, my role, and the problems I solved, without sharing proprietary code, credentials, or customer data.

## Live product

| | Link | Notes |
|---|---|---|
| Member web app (PWA) | [Open the member app](https://xerosumio.github.io/glg-tlg-ecosystem-frontend/) | No app store download: open it on a phone and choose "Add to Home Screen" to install it. Sign-in requires a membership account. |
| Client's corporate website | [www.glg.com.hk](https://www.glg.com.hk) | Public site from the same project, managed by the website CMS. |

My own code changes were in the membership API and the admin / merchant back office described below, which power the member app's vouchers and redemptions. The back office is staff-only, so it is not linked here.

---

## Overview

The client runs two brands with a shared membership program. The platform lets members earn points, claim vouchers, and redeem them at partner shops, while staff and partner merchants manage everything through a web back office.

The work spans two product lines, six repositories in total:

| Product | Purpose | Stack |
|---|---|---|
| Member app | Installable PWA: membership card, points, voucher wallet, referrals | React 18, Vite, TypeScript, Tailwind CSS, i18next, `vite-plugin-pwa` |
| Membership API | Members, points, vouchers, referrals, merchant accounts, redemption log | Node.js, Express, TypeScript, Mongoose (MongoDB), JWT, bcrypt, express-validator |
| Back office | Admin console plus a merchant portal for scanning and redeeming vouchers | React 18, Vite, TypeScript, Tailwind CSS, `html5-qrcode` |
| Corporate website | Public marketing site with CMS-managed content | Next.js 14, TypeScript, Tailwind CSS, Framer Motion |
| Website CMS | Content editor for the corporate site | Next.js 14, TypeScript |
| Website API | Content, media, and enquiry endpoints | Express, TypeScript, Mongoose, Nodemailer, Sharp |

Points and vouchers live on an external loyalty platform. Only the membership API talks to it; the member app and back office call the membership API, so the platform's API key never reaches a browser.

## Architecture

```text
 Member PWA ─────┐
                 │  HTTPS / JWT
 Back office ────┼──────────────►  Membership API  ──────►  External loyalty platform
 (admin +        │                 (Express + TS)            (points, vouchers, staff PINs)
  merchant)      │                       │
                 │                       ▼
                 │                    MongoDB
                 │
 Corporate site ─┴─► Website API ──► MongoDB
 + CMS
```

- **Frontends** deploy to GitHub Pages through GitHub Actions on every push to `main`.
- **APIs** are containerised (Docker) and deployed to a server separately.
- **Tests** run with Vitest and `mongodb-memory-server`, with the external loyalty API stubbed at the `fetch` layer so contract shapes are checked without network calls.

## My contributions

### 1. Admin-provisioned merchant accounts with a forced first-login password change

Partner shops could already self-register and wait for approval. The business also needed admins to create a merchant account directly and hand over a temporary password.

- Added an admin-only endpoint and a "Create account" form in the Merchants list (name, email, initial password). Accounts are created as active and flagged `mustChangePassword`.
- The merchant can sign in with the temporary password, but the portal routes them to a change-password screen first.
- Enforced the rule on the server, not only in the UI: voucher lookup, single redemption, and batch redemption all return `403` until the password has been changed. A hidden button is not a security control.
- Kept self-registration and the approval flow unchanged, and reused the existing merchant portal instead of building a second one.
- Rebased the feature onto a teammate's concurrent batch-redemption work and resolved the conflicts so both the batch API and the new password gate stayed intact.

### 2. Configurable first-login password rule

A follow-up requirement set the first-login password to exactly six characters of any type. I applied it only to the forced change-password path, with matching client and server validation, and left the admin-issued temporary password and self-registration rules unchanged so existing flows were not affected.

### 3. Redemption log: separating the member from the person who redeemed

**The bug:** the admin redemption log showed every member as "(unknown)", and the "Redeemed by" column mixed different kinds of people into one string — sometimes an admin email, sometimes a merchant email, sometimes a shop staff PIN name.

**The investigation:**

- Traced all four consumption paths (admin with staff PIN, admin by code, merchant portal, public shop PIN) and documented what each one stored.
- Found that the loyalty platform's lookup and consume responses never include the voucher owner. One path was even copying the staff PIN's display name into the member field.
- Found that the owner *is* returned when a voucher is issued (member claims with points, or a free grant), but that response was being discarded.

**The fix:**

- **Member:** persist the redemption code and its owner at issue time, in a small `IssuedVoucher` collection. Every later consumption looks the code up and copies the member's name and email onto the log row. Failures while recording are logged and swallowed, so a storage hiccup can never trigger a duplicate grant.
- **Redeemer:** replaced the single mixed string with structured fields — actor kind (`admin`, `merchant`, `store_pin`), account email, account id, and the staff PIN name kept separately.
- **UI:** the log now shows a role badge, the account email, and the staff name as a second line only on PIN redemptions.
- **Backward compatibility:** older rows are labelled from what they already contain (merchant id, an email, or a staff name) without a data migration, and the legacy field is still written so existing readers keep working.
- **Tests:** added coverage for the claim, grant, admin PIN, admin by-code, merchant, and shop PIN paths, including the case where a code was issued outside this system and the member must stay blank.

### Ways of working

- Took over an in-flight codebase spread across several repositories and mapped which repo owns which behaviour before changing anything.
- Worked from Kanban cards with written acceptance criteria, verified each criterion before moving a card to review, and wrote status reports for the project lead.
- Kept secrets out of version control and client bundles: API keys stay in server-side environment variables only.

## Skills demonstrated

- **Frontend:** React, TypeScript, Vite, Next.js, Tailwind CSS, PWA, i18n, QR scanning
- **Backend:** Node.js, Express, REST API design, MongoDB / Mongoose, JWT auth, role-based access, input validation
- **Integration:** wrapping a third-party loyalty API behind a server-side gateway, idempotent redemption, defensive error handling
- **Quality:** Vitest, in-memory MongoDB, HTTP-level stubbing of external services, backward-compatible schema changes
- **Delivery:** Git workflows across multiple repos, rebasing on concurrent work, GitHub Actions to GitHub Pages, Docker

---

*Code samples and a walkthrough are available on request, subject to the client's confidentiality terms.*

# Islamicly FinprimApi — Mobile Integration Guide

**Audience:** Mobile (Android / iOS) and Web app developers integrating with the Islamicly ASP.NET Core wrapper around Cybrilla Fintech Primitives.

**Source of truth:** `F:\github projects\FinprimApi\` (wrapper) and `F:\github projects\IslamiclyWebLatest\` (reference Angular implementation).

This document is the single-source contract for the **end-to-end onboarding flow** — from a PAN KYC pre-check, through KYC request submission, eSign, investor profile, contact + bank account creation, MF Investment Account (MFIA) creation, bank pre-verification (penny-drop), and finally MF purchase + payment.

---

## 0. 📋 How to Use This Guide (especially when pasted into Claude)

This file is the **single source of truth** for the Islamicly Mutual Fund onboarding integration that wraps Cybrilla Fintech Primitives.

- **Backend:** `F:\github projects\FinprimApi` — ASP.NET Core 8 wrapper.
- **Frontend:** `F:\github projects\IslamiclyWebLatest` — Angular 17+ SPA (web admin + mobile WebView).

### For human developers

1. Read **Section 2** (end-to-end sequence) first to grok the flow.
2. Then read **Section 12** (wizard step walkthrough) to map UI screens to API calls.
3. Use **Section 3** as the endpoint reference once you know what you need.
4. **Section 9** is the file-path index — start there when you need to locate code.

### For Claude (when this whole file is pasted as context)

When given a task in this codebase, follow these rules:

- **Trust the file paths and line numbers in this doc** — cross-reference them rather than re-searching.
- **Extending the wizard?** Follow the existing patterns documented in **Section 12** — do not reinvent the step machinery.
- **Adding a new endpoint?** Follow the layered architecture: **Controller → Service → DbService → Stored Procedure**, mirroring the existing files listed in Section 9.
- **Modelling a new Finprim / Cybrilla resource?** The public docs are at:
  - https://fintechprimitives.com/docs/api/
  - https://poa.cybrilla.com/docs/
- **Debugging an error / failure code?** Check **Section 7** (common errors), **Section 11** (recovery recipes), **Section 10.x** (per-endpoint failure scenarios), and **Section 5.5** (pre-verification codes) — in that order.
- **Adding a field to KYC / Bank / Investor Profile?** Wire format is **snake_case** (Finprim convention). C# DTOs use **PascalCase** with `[JsonPropertyName("snake_case")]` attributes.
- **Response shape:** every controller wraps via `OkApi(...)` / `ErrorApi(...)` which produce `{ statusCode, message, result }`. Never bypass this.
- **Config:** backend env config lives in `appsettings.json` under the `Finprim` and `Cybrilla` sections — see **Section 8**.
- **DB schemas + stored procedures** live in `c:\Users\Flyremit Pvt Ltd\Desktop\OnboardingMF\Claude-db-queries.txt` (alongside this guide).

### Conventions quick reference

| Convention | Example |
|---|---|
| Backend C# files | `F:\github projects\FinprimApi\Controllers\KycController.cs` |
| Angular files | `F:\github projects\IslamiclyWebLatest\src\app\features\...` |
| DB stored procs | `dbo.Usp_Upsert_Fintech_*` / `dbo.Usp_Get_Fintech_*` |
| DB tables | `dbo.tbl_Fintech_*` |
| Wire format | snake_case (Finprim convention) |
| C# DTO names | PascalCase with `[JsonPropertyName("snake_case")]` |
| API base URL (dev) | `https://localhost:7001` |
| Angular dev URL | `http://localhost:4200` |
| Auth | JWT Bearer; `[Authorize]` on every controller; `CurrentUserService` extracts UserId from claims |
| How to detect "KYC already exists" | `GET /api/kyc/user-status` (`hasKycRequest`+`kycRequestStatus`) **then** `POST /api/pre-verifications/readiness`. See **Section 14.1** for the full decision tree. |
| Full lump-sum investment flow (end-to-end) | See **Section 14** for every step from KYC detection → payment → webhook |
| DB snapshot vs audit pattern | See **Section 15** — `tbl_Fintech_*_Order/_Payment` are 1-row snapshots, `tbl_Fintech_MF_Order_Log` is the audit trail |

### Architecture (compact)

```
Angular SPA  ──HTTPS──▶  FinprimApi (ASP.NET Core 8) ──HTTPS──▶  Finprim / Cybrilla
                              │
                              ▼
                        SQL Server (Islamicly DB)
                          tbl_Fintech_* tables
                          Usp_Upsert_/Usp_Get_Fintech_* stored procs
```

### Recent changes — quick changelog (latest first)

- **2026-07-01 (NEW §14.5.5 — complete nominee flow + the guardian-address camelCase answer)** — Documentation-only pass answering the app team's guardian-address field-name question. Added **§14.5.5 "COMPLETE nominee flow — exact sequence, APIs, and the guardian-address field names (LATEST)"**: (1) the full ordered API sequence (status → sync/list → getAttached → create pool → `nominees/allocations` attach → opt-out), the two-object model (person = `/v2/related_parties`; % lives on MFIA `folio_defaults`, set only at attach), and the full endpoint table. (2) **Guardian-address field names = camelCase** — `guardianAddressLine1/City/State/PostalCode/Country` (binds to `CreateNomineeRequest`, matches the web app). **NOT `guardian_address`/snake_case**, which is a DIFFERENT internal class (`CreateRelatedPartyRequest`, the backend→Cybrilla DTO). Only `date_of_birth` is snake_case on this request. (3) Explained why a successful create doesn't echo guardian address: `CreateNominee` forwards only profile/name/dob/relationship to FP and **persists allocation/PAN/guardian+address locally**, applying them to folio_defaults at attach time — so it's saved, just not sent to FP on create; minor create 400s without guardian address line1 + pincode. No code/DB change.

- **2026-06-30 (NEW §32 — MF Details page data/API/formula reference; invest page cross-referenced, not duplicated)** — Documentation-only pass for the app team. (1) **NEW §32 "MF Details Page — Data, API & Calculation Reference (LATEST)"** documents the fund Details page (`/app/mutual-funds/details/{ISIN}`): the previously-**undocumented** `GET /masterdata/nav/{isin}/series` and all `GET /masterdata/scheme-overview/{isin}/*` endpoints (trailing-returns, overview, holdings, sectors, documents, riskometer, summary, faqs), the full field→API-source binding by section, the **client-side calculation formulas** (verbatim Angular `navTick`/`navTickPct`/`navPeriodReturn` computeds + SIP/lumpsum backtest, donut arc, holding-bar, quick-projection), the chart list, and the hardcoded/mock inventory. (2) **Key app-team rule called out (§32.3):** the NAV-series endpoint returns a daily `points[]` array + period `returnPct` — there is **NO `currentNav`/`dayChange` field**; Current NAV = last point, daily change = last-vs-prev point, period return = first-vs-last (or API `returnPct`). The app must compute these the same way. (3) **Invest page intentionally NOT re-documented** — §32.7 cross-references the existing latest contract (§3.18 flows, §14.2 fresh-KYC + fail-closed invest gate, §30.2/§30.3 lumpsum+SIP end-to-end, §30.5 client-side validation guards, §31 callbacks) so there's one source, no duplicate. No code/DB change.

- **2026-06-19 (Guide restructure — KYC consolidated + Resume section; old KYC marked deprecated)** — Documentation-only pass for the app team. (1) **§3.1–§3.2 (`/kyc/checks`, `/kyc/user-status`) and the §3.3–§3.11 KYC-request wizard marked ⛔ DEPRECATED** — the current KYC flow is `pre-verifications/readiness → kyc/forms` (§14.2). (2) **NEW §14.3 "KYC — consolidated latest"** — single source for fresh-vs-verified PAN differences, OTP-verified email/mobile (verify-once, sandbox `sandboxOtp`), PAN/DOB/email/mobile auto-populate + read-only rules, silent auto-submit, and a numbered list of **every issue we hit on web and fixed** (resume loop, cross-PAN form hijack + PAN-match guard, empty-DOB stuck form, eventual-consistency double-save, geolocation-before-save, prefill `markForCheck`, verified-contact wins, clean FP field errors, NA-name guard, fail-closed invest gate, 409 duplicate-PAN, PEP enum mapping, 401 token retry). (3) **NEW §14.4 "Resume"** — exact contract for picking up after the user leaves **after KYC / after investor profile / after MF account** (the reported gap; pre-MFIA stages already resumed fine): the three status calls (`pre-verifications/my-status`, `kyc/forms/my-status` with PAN-match, `onboarding/status`), the first-incomplete decision order (KYC→profile→address→contact→bank→FATCA/demat→nominee→MFIA), the "`investorProfileId` IS proof of KYC" rule that fixed the kyc⇄onboarding loop, and fresh-vs-verified resume behavior. (4) **NEW §14.5 "Nominee screen"** — the add-nominee step contract: two-section screen (add-to-pool → choose ≤3 + split 100%), full field table, minor→guardian-required + nominee-PAN-nulled, allocation/PAN/guardian persisted locally (not sent to FP related-parties) and applied to MFIA folio_defaults, immutable-on-edit fields, opt-out, and resume. Expanded §3.13 nominee body to match. (5) **NEW §14.6 "Fresh & Verified KYC converge"** — both KYC types share ONE onboarding flow; the only branch is at readiness (fresh runs `kyc/forms`, verified skips it), then both converge to profile→address→contact→bank→FATCA/demat→nominee→MFIA→invest. No code/DB change.

- **2026-06-18 (FATCA fixes — tax-residency dates removed, multi-country, PEP enum)** — Three FATCA fixes (FE + DTO + SQL):
  (1) **Removed `applicable_from`/`applicable_to`** from the tax-residency payload — they are NOT valid FP `tax_residency` fields and made `PATCH /v2/investor_profiles` fail with *"JSON parse error … valid data types"* (400). FP `tax_residency` = `country` + `taxid_number` (+`taxid_type`) only. Date pickers dropped from the FATCA screen.
  (2) **Multiple foreign tax residencies** — the onboarding FATCA screen now allows India + up to **3** foreign countries (repeater with Add/Remove, dup-country guard) → mapped to FP `second/third/fourth_tax_residency`. `/api/onboarding/fatca` now takes `foreign_residencies[]`.
  (3) **PEP enum corrected + carried over** — the **investor-profile** PEP enum is `not_applicable | pep_exposed | pep_related` (the FATCA screen was wrongly sending `pep` / `related_to_pep`). The **kyc_form** PEP enum is different: `pep | related_pep | no_exposure`. The user answers PEP once in fresh-KYC; `my-status` now returns `kycPepDetails` and the FATCA screen maps it (`pep→pep_exposed`, `related_pep→pep_related`, `no_exposure→not_applicable`) and pre-selects it. **Both enums + the mapping are in the enum appendix ("pep_details — TWO DIFFERENT enums").** Also surfaced the real FP error on `/api/onboarding/fatca` (was a generic 502). Requires re-running `Usp_Get_Fintech_Investor_Profile_Status` + FinprimApi restart + SPA rebuild.
- **2026-06-17 (Fresh-KYC flow — consolidated app-team walkthrough + latest changes)** — New self-contained **§14.2 Fresh KYC (no KRA record)** documenting the end-to-end path for a PAN with no KRA data, for the app team. Captures this session's changes: (1) **PAN+DOB prefill** on the Verify-PAN screen from the user's approved app KYC via the NEW `GET /api/onboarding/user-details` (SP `Get_User_Details_For_Fintech`) — editable; (2) **`/kyc-form` fill_fields auto-fill + silent auto-submit** — when every field is already known from `my-status` + geolocation is captured, the form PATCHes itself and advances (the user never sees the screen); only shows for genuinely-missing fields; (3) **first-save false-failure fixed** — after PATCH the page re-checks `fields_needed` with a short retry before surfacing "still requires" (Cybrilla `fields_needed` is eventually consistent); (4) **invest gate is now fail-CLOSED** — `isReady()` requires readiness **explicitly `verified`** (a null/empty readiness — no KYC — no longer slips through); (5) **test-PAN pattern for the app team**: fresh-KYC PANs follow `...PX3753X` (sandbox simulator → readiness `kyc_unavailable` → `start_fresh_kyc`). ⚠️ These simulator PANs complete KYC but **never become purchase-ready** (readiness stays `failed`); use the **`...PX3751X`** pattern to test the actual invest path.

- **2026-06-17 (Duplicate-PAN guard — one PAN = one app user)** — `POST /api/pre-verifications/readiness` now **blocks with HTTP 409** when the submitted PAN already belongs to a **different** user (checked across `tbl_Fintech_Pre_Verification` / `tbl_Fintech_Kyc_Form` / `tbl_Fintech_Investor_Profile` via new SP `Usp_Check_Fintech_Pan_Owner`). Enforced **server-side at readiness, before any Cybrilla call** (UI can't bypass), in **both** with/without-KRA paths and **including sandbox**. Same user re-entering their own PAN is allowed (resume). Message: *"This PAN is already registered. Please use your own PAN, or contact support if this is an error."* App team: on 409, show the message inline on the PAN screen, **do not retry, do not advance**. Full contract in **§14.2.5**. Requires the new SP + a FinprimApi restart.

- **2026-06-10 (FIX: kyc-form Save "first click does nothing, second works")** — `patchForm()` called `navigator.geolocation.getCurrentPosition` BEFORE setting `busy`, so the first click silently waited on the browser location-permission prompt with no spinner/feedback (looked dead); the 2nd click (permission already granted) resolved instantly. Fix: re-entrancy guard (`if (busy()) return`), set `busy=true` + show "Saving…" BEFORE the location call (immediate feedback, disables the button so no double-submit), release `busy` on geo failure with the clear message, 10s→15s timeout. ng build clean.

- **2026-06-10 (Start-fresh-KYC: PAN read-only + DOB carried from verify step)** — On the kyc-form "Start fresh KYC" screen the PAN auto-filled but DOB was empty. Cause: the PAN-verify step (mutual-fund-kyc) navigated to /kyc-form passing only `from`+`action`, not PAN/DOB — and a synthetic-PAN readiness response carries no DOB to backfill from. Fix: all three navigate-to-kyc-form calls now pass `pan` + `dob`; kyc-form ngOnInit seeds `formPan`/`formDob` from those query params (before refresh, so they win). PAN field is now **read-only** when populated. ng build clean.

- **2026-06-10 (KYC PAN step: removed step-chip, NSDL subtitle, privacy note, mock Name)** — Per request, trimmed the "Verify your PAN" step (mutual-fund-kyc): removed the "Step {n} of 4 · {m} min" chip, the "We'll auto-fetch your details from NSDL…" subtitle, the "Your PAN is encrypted…" privacy-note footer, and the **Name** row from the right-side ID-card mock (kept PAN + DOB). Title + icon + form unchanged. ng build clean.

- **2026-06-10 (FIX inconsistency: Invest page now gates on readiness like the KYC page)** — The KYC page locked investing on `readiness.status=verified` (correct per §29.9.1 / IsPurchaseReady), but the **Invest page did NOT** — `isReady()` only checked profile+bank+MFIA, so a user with `readiness=kyc_unavailable` could start a purchase that FP would reject with `non_kyc_compliant_investor_error`. Fixed: invest component now injects `PreVerificationService`, fetches `my-status` on load (`purchaseReadiness` signal), and `isReady()`/`disableReason()` block when readiness is known-and-not-verified ("Investing unlocks once KYC is confirmed"). Null readiness (endpoint down) doesn't false-block — FP still enforces server-side. Both screens now agree. ng build clean.

- **2026-06-10 (Verified email/mobile now carry from KYC → onboarding: prefill + read-only)** — Root cause: a contact verified during KYC writes a consumed row to `tbl_Fintech_Contact_Otp`, but at that point there's NO row in `tbl_Fintech_Investor_Phone/_Email` yet — so `Usp_Mark_*_Verified` updated 0 rows and `Usp_Get_Fintech_Contact_Verified` (which only read the phone/email tables) returned nothing → onboarding's Phone/Email steps showed an empty, unverified field even though the user just verified. Fix: rewrote `Usp_Get_Fintech_Contact_Verified` to ALSO treat a consumed `Contact_Otp` row as verified (the real source of truth across both flows). Verified live for user 163294 (email+mobile now return Verified=True). Frontend onboarding: the verified-status subscription now **prefills `emailForm`/`phoneForm`** from the returned value, and the Phone/Email inputs are **read-only with a "✓ Verified" badge** when already verified (button reads "Continue →", skips OTP — logic was already there, but the field wasn't being populated). Updated the SP in Claude-db-queries.txt too. ng build clean.

- **2026-06-10 (KYC POA form: email+mobile OTP verify, remove Transgender; + SQL `[Plan]` fix)** — (1) **FIX `Usp_Get_Fintech_MF_Purchase_Order_ByUser`**: `o.Plan` is a reserved word → `Incorrect syntax near 'Plan'`. Bracketed to `o.[Plan]` in both Claude-db-queries.txt and buy-allotment-navdate.sql; re-ran the SP against the DB (now live). (2) **kyc-form fill-fields: email + mobile now OTP-verified** — each field has an inline Verify→OTP→Confirm flow (reusing the contact-OTP endpoints: `sendEmailOtp/verifyEmailOtp/sendMobileOtp/verifyMobileOtp`, sandbox shows the code); field disables + shows "✓ Verified" once confirmed; `validatePatchForm` blocks Save until BOTH are verified; editing a verified value clears its verified flag. If the server already has the value verified (from onboarding) it's auto-marked verified (no re-OTP). (3) **Removed "Transgender"** gender option. (4) Auto-populate stays (prefill from kyc_form raw + profile-status + onboarding contact). ng build clean.

- **2026-06-10 (Backend: parse FP validation errors → clean 400, not 500 stack dump)** — The KYC form PATCH was throwing the whole `"...failed: BadRequest - {json}"` blob (a raw `Exception` → 500 with stack trace). Now `KycFormService` has `ParseFpError(body)` that extracts FP's structured error (`error.errors[0].field/message`, else `error.message`) into a short string; on a 4xx from FP it throws a new **`UpstreamValidationException`** (→ middleware returns **HTTP 400 with the clean message**, e.g. "spouse_name: not a valid spouse_name", no stack trace, no alert email since it's user-correctable); on 5xx/network it throws `HttpRequestException` (→ 502 + alert). Applied to both `SendJsonAsync` and the signature upload. Frontend `cleanError()` (prior turn) maps that to friendly copy ("Please enter a valid spouse name…") — verified against the exact error string. So the same validation error is now clean at BOTH layers. Backend 0 errors. **Redeploy API + reload SPA** for both fixes to be live (the user was still seeing the raw error because the SPA hadn't been rebuilt).

- **2026-06-10 (KYC POA form UX fixes: hide raw error/id, friendly messages, NA-name guard)** — Screenshot showed a full stack trace + internal kyc_form id dumped on screen and a `spouse_name: "not a valid spouse_name"` 400 (user typed "NA"). Fixes: (1) **kyc_form id no longer shown** to the investor (banner). (2) **`cleanError()`** sanitises any API/exception payload into a short, user-safe sentence — strips stack traces / leading exception type, and **extracts FP's field validation reason** (`{error.errors[0]}`) into friendly copy (e.g. "Please enter a valid spouse name (not NA/blank)"). Applied to all `formError.set` sites (save/create/geo/upload/retry/load). Error boxes are now clamped (`max-h-24 overflow-y-auto break-words`) so a long message can't break the layout. (3) **NA/placeholder name guard** in `validatePatchForm` — rejects "NA/N/A/nil/-/." and <2-char spouse/father names before the FP round-trip. (4) **"Missing fields" CSV replaced** with "All fields below are required. We auto-fill residency, nationality & location for you." (the raw FP `fields_needed` list included auto-sent fields like citizenship/nationality/tax-residency/geolocation, which confused users). ng build clean. (Note: the page IS correct — it's the POA fill-fields step, status=created.)

- **2026-06-10 (KYC error emails + signature 502 hardening + 50 fresh-KYC test PANs)** — (1) **Email-on-error:** `ExceptionHandlingMiddleware` now fires `ISendGridService.SendErrorAlertAsync` (→ `SendGrid:ErrorLogEmail` = irfan@islamicly.com) on ANY error (500/502) for KYC-flow routes (`/api/kyc`, `/api/pre-verifications`, `/api/onboarding`, `/api/banking`, `/api/identity-documents`, `/api/esign`). Best-effort, never masks the original error. ⚠️ **emails only send once a real `SendGrid:ApiKey` is set** — it's currently `""` in appsettings.json, so SendAsync logs `EmailError` and sends nothing. Set the key to activate. (2) **Signature-upload 502 cause now visible** — the signature endpoint threw a plain `Exception` with the FP status+body; combined with today's middleware fix the REAL FP rejection now appears in the API response (was "Something went wrong" on UAT). Also added **401-retry** to `UploadSignatureAsync` (buffers the file → `ByteArrayContent` so it can replay; on 401 invalidates token + retries once) — closes the transient-token 502. (3) **`fresh-kyc-test-pans.txt`** — 50 `…PX3753…` PANs for fresh-KYC testing (single-use each; readiness-eligibility still Cybrilla-seeded). Backend 0 errors.

- **2026-06-10 (KYC POA: real 500 error surfaced + backend mandatory validation + mandatory geolocation)** — Three fixes after the UAT `/api/kyc/forms` returned a bare `{"statusCode":500,"message":"Something went wrong"}`. (1) **Real error in the API response** — `ExceptionHandlingMiddleware` now returns `"{ExceptionType}: {message} | inner: …"` for unhandled exceptions (was masked to "Something went wrong" outside Development); stack trace still Dev-only. Full exception (type+message+inner) always logged. It's an internal API, so the cause is now visible to the dev/app team. (2) **Backend mandatory-field validation** — `KycFormsController.Update` now runs `ValidateFillFields`: when the PATCH carries the personal-details block (fill-fields submission), ALL fields are required (email, mobile, gender, marital, spouse-or-father, occupation, income, aadhaar-4, pep, place-of-birth) **+ geolocation (lat/lng must be non-zero)** → 400 with a specific message. Incremental geo-only/signature/proof patches are NOT affected (they don't carry personal fields). (3) **Geolocation mandatory + flexible (frontend)** — `kyc-form.patchForm()` no longer silently fakes a central-India coordinate when location is denied; it captures the REAL device location (10s timeout, high-accuracy) and, on block/failure, stops with a clear retryable message ("allow location → Save again"). Frontend field-level mandatory validation (`validatePatchForm`/`patchFormValid`) + prefill from prior turn remain. Backend 0 errors; ng build clean.

- **2026-06-10 (KYC POA fill-fields form: all fields mandatory + richer prefill)** — `kyc-form.component` "Complete your KYC details" (fill_fields stage). (1) **Every field is now mandatory** — new `validatePatchForm()` requires email (format-checked), 10-digit mobile, gender, marital status, spouse-or-father (per marital), occupation, income slab, Aadhaar last-4, PEP, place of birth; `patchForm()` blocks submit with a specific message; Save button is disabled (`patchFormValid()`) and reads "Fill all required fields" until complete; each label shows a red `*`. (2) **Auto-populate extended** — `prefillFromLatest` still resumes from the form's `rawResponseJson`, then `prefillFromOtherSources()` backfills any STILL-empty field from `getInvestorProfileStatus` (gender/occupation/income/place-of-birth) + `getContactVerifiedStatus` (email/mobile). Only fills blanks; never overwrites typed/saved values. ng build clean. (Context: user hit a 502 on signature upload, reloaded back to this form — prefill now restores the entered values so they don't re-type.)

- **2026-06-10 (Naming consistency: KYC child-upsert SPs → Fintech prefix)** — The only two MF/Cybrilla SPs missing "Fintech" were `Usp_Upsert_KYC_Constraints` / `Usp_Upsert_KYC_Sources` (the KYC-check constraints/sources XML child upserts; their tables already had Fintech). Renamed both to `Usp_Upsert_Fintech_KYC_Constraints` / `Usp_Upsert_Fintech_KYC_Sources` in Claude-db-queries.txt (CREATE + comments + sanity list) + added `DROP PROCEDURE IF EXISTS` for the old names; updated the two C# call sites in `KycCheckDbService.cs`. Verified: every CREATE TABLE/PROC/VIEW in Claude-db-queries.txt now includes "Fintech". (Other non-Fintech matches in the DB — Gold SIP, GreenPortfolio PMS, CAMS equity, Razorpay/Cashfree, `report_*` admin dashboards, `usp_IsRedeemOTP_verified` — are OTHER products, not the Finprim MF flow.) Backend 0 errors. **Run the renamed CREATE blocks + the DROPs, then redeploy API.**

- **2026-06-10 (Token 401-retry safety net)** — Closed the last token race: if Finprim returns **401** (token rejected mid-flight — expired in the gap after the 60s buffer, or revoked), the client now force-refreshes the token and retries the call **once**. Added `IFinprimAuthService.InvalidateTokenAsync` → `ITenantTokenProvider.InvalidateAsync` (evicts the in-memory cache + deletes the DB row so sibling instances also refetch) → `ITenantTokenDbService.DeleteAsync` (inline DELETE by TenantSlug, no migration). `FinprimApiClient.SendAsync` and `PatchRawAsync` wrapped with the retry (body serialized once, re-sent). Multipart upload left without retry (stream isn't replayable — rare path). Backend 0 errors. Normal path unchanged: 3-tier cache still serves tokens; the retry only triggers on an actual 401.

- **2026-06-10 (FIX: tbl_Fintech_Tenant_AccessToken stayed empty — silent upsert failure)** — Root cause: `TenantTokenDbService.UpsertAsync` passed `@ExpiresAtUtc`, but `Usp_Upsert_Fintech_Tenant_AccessToken` doesn't declare that param (it computes `ExpiresAtUtc` internally from `@ExpiresIn`). SQL Server rejected the call ("procedure has too many arguments specified"), and `UpsertAsync` swallows all exceptions (logs a warning, never throws) — so every token write failed invisibly while the API kept running off the in-memory cache. The token table therefore never got a row. Fix: removed the extra `@ExpiresAtUtc` parameter from the C# call (SP already sets it). Now token persists → survives restarts + shared across instances as designed. Backend 0 errors. (DB has the table + correct SP already; no migration needed — just redeploy the API.)

- **2026-06-10 (KYC email + mobile OTP self-verification, verify-once)** — New: when an investor enters or edits their email/mobile in onboarding, they must verify it via OTP before it saves. **Verify-once**: a value already verified server-side AND unchanged skips OTP; entering a DIFFERENT value forces a fresh OTP. **DB** (`contact-otp-verification.sql`): `tbl_Fintech_Contact_Otp` (channel email|mobile, hashed-plain OTP, 10-min expiry, 5-attempt lockout, SandboxOtp col) + `Usp_Insert/Verify_Fintech_Contact_Otp`; `IsVerified`+`VerifiedAt` on `tbl_Fintech_Investor_Email`/`_Phone` + `Usp_Mark_*_Verified` + `Usp_Get_Fintech_Contact_Verified`. **Backend** `ContactOtpDbService` + OnboardingController endpoints: `GET /api/onboarding/contact/verified-status`, `POST .../contact/{email|mobile}/send-otp` (sandbox returns code; email via SendGrid in prod, SMS TODO), `POST .../contact/{email|mobile}/verify-otp` (marks verified). **Frontend** kyc.service `sendEmailOtp/verifyEmailOtp/sendMobileOtp/verifyMobileOtp/getContactVerifiedStatus`; onboarding mobile+email steps now: validate → if already-verified-unchanged save straight through, else Send OTP → enter (sandbox shown) → Verify & Continue; editing the field resets the OTP stage. Prefilled values are NOT auto-trusted (only server-verified count). Backend 0 errors; ng build clean. **RUN `contact-otp-verification.sql` + restart API.** SMS delivery for mobile OTP is a sandbox stub — wire a real SMS provider for production.

- **2026-06-10 (T&C acceptance at OTP for Buy/SIP/Redeem + compliance audit)** — Added a mandatory **Terms & Conditions** checkbox in all three order OTP modals (Buy + SIP in mutual-fund-invest, Redemption in mutual-fund-swp). Link → `/app/mutual-funds/terms` (existing web T&C page), opens new tab; Verify/Confirm disabled until ticked (alongside OTP + nomination consent); `termsAck` resets per order. Closes audit gap #8 (T&C was a fetched URL only, no acceptance). Audit of compliance points 3–11: 3 mobile self-declaration ✅, 4 nomination flow ✅, 5/6/7 2FA buy/redeem/SIP ✅, 8 T&C ✅ (now), 9 redemption shows folio/scheme/units ✅, 10 holdings from FP reports ✅, 11 no consent/identity defaults (but UI defaults exist: SIP freq=monthly, nominee rel=son, lumpsum pay=UPI, single-nominee=100%). ng build clean.

- **2026-06-10 (Transactions: mandates view restored as a 2-way toggle)** — Re-added a **Transactions ↔ Auto-Debit Mandates** toggle at the top of the Transactions page (replacing the removed 3-way Orders/SIPs/Auto-Debit control). `view='orders'` → executed transactions + type chips; `view='mandates'` → the existing mandates list (`loadMandates()` lazy-loads on first switch). The SIPs view block stays inert (SIPs live on Portfolio). ng build clean.

- **2026-06-10 (Portfolio: remove tab bar → Holdings-only)** — Removed the Holdings/SIPs/Orders/Transactions tab strip from `/app/mutual-funds/portfolio`. The page now shows hero + allocation/SIP-schedule cards + a single **"Your Holdings"** card grid. Orders & Transactions have their own dedicated pages; the SIP schedule already lives in the card above. Deleted the SIPs/Orders/Transactions tab-content blocks (the `activeTab`/`tabs()`/`allTx` machinery is now unused but harmless). ng build clean.

- **2026-06-10 (Portfolio: "Profit/Loss" rename + metric tooltips)** — Renamed the holding card's "Returns" column to **"Profit / Loss"** (it's ₹ P&L + %). Added one-line **ⓘ hover tooltips** (native `title`) explaining **Abs. Return** (total % ignoring time), **CAGR** (annualised smoothed %), and **XIRR** (timing-weighted annualised %, best for SIPs) — on both the hero stats and the per-card chips; Profit/Loss has its own tip too. Tooltip strings are `TIP_*` constants on the component. ng build clean. (Mandates/Auto-pay view stays removed from Transactions per prior instruction — code is inert in the component, not deleted.)

- **2026-06-10 (Holdings cards: label every number)** — Restructured the portfolio holding card so every value has a heading instead of floating numbers: **Units** + **NAV** row, **Invested** + **Current** row, and a **Returns** row showing P&L ₹ + % with **XIRR** and **Abs. Return** as labelled chips (prefixed "XIRR"/"Abs. Return"). Folio already labelled. ng build clean.

- **2026-06-10 (Portfolio: CAGR + Absolute Return; Transactions: drop view switcher)** — (1) **Portfolio now surfaces `cagr` + `absolute_return`** from `investment_account_wise_returns` (previously dropped): `loadPortfolio()` reads both, averages across MFIAs, returns on `totals.cagr`/`totals.absReturn` (absReturn falls back to gain/invested if the report omits it). Hero shows **Abs. Return** + **CAGR** chips beside XIRR; each holding card shows a per-fund **abs%** chip under the XIRR badge (from scheme_wise_returns `absReturn`, already on the model). (2) **Transactions page: removed the Orders/SIPs/Auto-Debit segmented control** — the page is the executed-transactions list only. The `view` signal is locked to 'orders' (never reassigned now the buttons are gone); the `sips`/`mandates` view blocks remain inert in the template for later. SIPs are viewable on Portfolio; auto-debit mandates are set during onboarding. ng build clean.

  **SIP vs Auto-pay (mandate):** SIP = the recurring *investment instruction* (`mf_purchase_plan`/mfpp — what to buy, how much, which day). Mandate/Auto-Debit = the bank *payment authorization* (eNACH/UPI-Autopay, `mandate`/man_xxx) that lets installments be auto-pulled. One SIP needs one approved mandate behind it.

- **2026-06-10 (Transactions KPIs + type tabs + Auto-pay rename, round 2)** — (a) **Removed the "Pending Orders" KPI** and the **"+₹x this month" sub-text** on Invested/Redeemed cards (this page is successful-only); KPI grid now 2-up (Total Invested, Total Redeemed). (b) **Type tabs always show All / Buy / SIP / Sell** — removed the `countOf(ty.key) > 0` guard that was hiding the SIP tab when the account has no successful SIP installments yet. (c) **Top segmented control "Auto-pay" → "Auto-Debit"** (was a hardcoded literal, not the i18n key, which is why the earlier dictionary rename didn't take). (d) Status filter stays hidden (successful-only). ng build clean.

  **FP `investment_account_wise_returns` coverage:** we capture+show `invested_amount`, `current_value`, `unrealized_gain`, `xirr` (portfolio hero). We do NOT currently surface `absolute_return` or `cagr` from that report (absolute_return is captured per-fund as `absReturn` but not rendered; cagr is dropped entirely). Holdings cards show units/NAV/current/P&L%/XIRR — not CAGR/absolute_return.

- **2026-06-10 (Orders & Transactions refinements + Orders nav label fix)** — (a) **Orders nav label** was rendering the raw i18n key because `appShell.nav.mf.orders.label/.desc` were missing from `public/assets/i18n/en.json` (the `t` pipe returns the key when a translation is missing); added them → shows "Orders". (b) **Orders page**: SIP installments now appear (a purchase row with `plan != null` → `side: 'sip'`, blue badge); the **Executed** tab now buckets `successful` + `cancelled` (was successful-only); the success chip label changed from "Executed" to **"Successful"**. (c) **Transactions page**: type tabs trimmed to **All / Buy / SIP / Sell** (dropped redemption/dividend/switch — no data); `mapRow` now types a `plan != null` purchase as **sip** so All = buy+sip+sell and the SIP tab populates; **Status filter dropdown hidden** (page is successful-only, so it had a single meaningful value); **"Autopay" renamed** to meaningful labels — tab → "Auto-Debit Mandates", SIP card field → "Auto-Debit". ng build clean. No backend/DB change.

- **2026-06-10 (Orders ↔ Transactions split: Open/Executed; transactions = executed-only)** — Per ops instruction: separate Orders from Transactions. (1) **New Orders page** `/app/mutual-funds/orders` (`MutualFundOrdersComponent`) with **Open / Executed** sub-tabs — Open = `pending | under_review | submitted | confirmed`, Executed = `successful`; merges buys (`listMyPurchases`) + sells (`listMyRedemptions`), per-row + bulk sync (re-syncs open orders from Cybrilla), state chips, payment status, failure codes. Added route + a **new "Orders" submenu item** in the MF nav (between Portfolio and Transactions). (2) **Transactions page now shows ONLY the transaction stage** — `loadAllTransactions()` filters to `state === 'successful'` for both buys and sells (a successful order IS the transaction). In-flight/failed orders no longer appear there — they're on Orders. (3) Portfolio page's Transactions tab aligned to the same executed-only rule. **No DB change** — derived in UI (per chosen approach: a successful order = the transaction record; no separate table). ng build clean.

- **2026-06-10 (Sell Units page → direct one-time-sell form; portfolio Redeem deep-links)** — `/app/mutual-funds/swp` was a SWP-plan list (recurring withdrawals, which Cybrilla POA Beta doesn't support yet) with a Sell action hidden in a modal — so the portfolio "Redeem" button dumped users on an empty plan list. Reframed the page as the **one-time Sell form inline** (the page IS the form): holding picker (pre-selected), a holding-details card (units held + redeemable value from the holdings report), sell-all / by-units / by-amount, max enforced from `/api/oms/reports/holdings`, then the existing OTP-consent confirm step (`createRedemption` → `confirmRedemption`). On success shows a confirmation with "Sell another" / "View portfolio". **Deep-link:** portfolio Redeem now navigates with `?isin=&folio=`; `ngOnInit` reads them, loads holdings, and auto-selects the matching holding (fetching its live redeemable max). **SWP UI commented out, not deleted** — stats row + plan list + create-SWP modal preserved in the template (block comment) and all backing methods (`load`/`refresh`/`openCreate`/`confirmCreate`/`stats`/`rows`) kept on the class, ready to re-enable when SWP support ships. `openSell()` legacy modal path left inert. ng build clean. (Only lumpsum sell is supported now — page copy says so.)

- **2026-06-10 (Cybrilla CONFIRMED in writing: `type=fresh` IS supported on KYC Form API)** — Resolves the long-standing "public doc says modify-only" ambiguity. Cybrilla support email: *"We recommend using the KYC Form API instead of the KYC Request API when submitting a fresh KYC request… Please make sure to set `type = fresh` when initiating a fresh KYC request through the KYC Form API. Use the preverification credentials to access the KYC Form API in sandbox."* Authoritative doc: **https://poa.cybrilla.com/docs/additional-apis/kyc-forms** (the public `docs.fintechprimitives.com` KYC-Forms page is incomplete — lists `modify` only; ignore that for `type`). **Net: our flow is correct** — we create `POST /poa/kyc_forms` with `type=fresh` on the `cybrillarta` (pre-verification) token; legacy KYC **Request** API stays retired. No code change. **Sandbox PAN reminder (verified live 2026-06-10):** the letter directly after `P` (the `X` in `…PX375N…`) is **irrelevant** — `AAAPX3751A` and `AAAPK3751A` both → readiness `verified`; `AAAPX3753A` and `AAAPK3753A` both → `kyc_unavailable`. Only the **`3751`–`3759` digit** drives KYC-status simulation (`3751`=compliant/investable; `3753`=not-available/fresh-required, never investable in sandbox; `3754`=on_hold, `3759`=incomplete → `type=modify`-eligible). A `…3753…` fresh form reaches `submitted` but readiness never becomes `verified` in sandbox — by design, not a bug.
- **2026-06-10 (Portfolio polish: card holdings, SIP next/prev schedule, no stray nav; CG successful-only)** — Round 2 on the portfolio page after screenshot review. (1) **Holdings table → card grid** (md:2/xl:3 cols): each card = identity + XIRR badge + current value + invested + P&L% + Buy/Redeem actions + folio. (2) **Removed row-click navigation** — clicking a holding no longer jumps to `/mutual-funds/details/INF…` (the fund marketing page); only the explicit Buy/Redeem buttons navigate. (3) **SIP tab redesigned**: top shows **Upcoming Deductions (next 2)** — projected forward from each active plan's `nextInstallmentDate` by frequency (`stepDate`), sorted+sliced to 2 — and **Recent Deductions (prev 2)** — REAL executed installment orders (purchase rows with `plan != null`), newest first; below that, active SIP plans in **card layout** (was a table). (4) **Allocation** confirmed = each fund's current market value ÷ total portfolio value (real, no hardcode). (5) Transactions: **capital-gains panel now renders only for SUCCESSFUL sells** (`status==='processed' || state==='successful'`) — previously showed an empty panel on pending/failed sells. ng build clean. All data real (the screenshot's ₹6,713 / 3 Aditya-Birla funds / 59 txns were already live from the user's account).

- **2026-06-10 (Transactions: realized Capital Gains on sell rows)** — Wired the last unused FP report. Expanding a **successful sell** now lazy-loads `POST /api/reports/capital-gains` (`getCapitalGainsReport`, filtered to mfia+scheme+folio) and renders a "Capital Gains (realized)" panel: **Actual Gain**, **Taxable Gain** (grandfathering-adjusted), **Holding Period** with **LTCG/STCG** badge (derived from `source_days_held` > 365), **Bought→Sold NAV** (`source_purchased_at`→`traded_at`), Purchased-On date, and a Grandfathering-Applied flag. Aggregates across multiple redeemed lots into one figure per sell. Added `mfiaId`/`capitalGains`/`cgState` to the `Transaction` model (mfiaId now flows from both mapRow + mapRedemption). Fetch is gated to redemptions in `successful`/`processed` state and only fires once per row on first expand. Graceful empty state ("appears once RTA processes, T+1/T+2") for ONDC where the gains feed lags. ng build clean. (No backend/DB change — endpoint already existed.)

- **2026-06-10 (Portfolio page: replace ALL mock data with real FP reports)** — `/app/mutual-funds/portfolio` was 90% hardcoded (fake ₹2,34,840, 4 fixed funds, fake SIP calendar, fake transactions; only the Orders tab was real). Rebuilt against FP reports — every number now traces to a Cybrilla report or our DB mirror. New `MfPurchaseService.loadPortfolio()` aggregates: `listMyPurchases` (cost basis + bought units per mfia|scheme|folio) → per-MFIA **holdings report** (`/api/oms/reports/holdings`: current `nav.value`, `market_value`, `redeemable_units`) + **scheme_wise_returns** (per-fund XIRR + absolute_return + current_value) + **investment_account_wise_returns** (account totals invested/current/unrealized_gain/XIRR), netting successful redemptions off bought units. Returns `{holdings[], totals}` (new `PortfolioHolding`/`PortfolioData` interfaces). Component rewrite: **hero** = real value/invested/profit/profit%/XIRR (+NAV date); **allocation donut** = real by-fund split of market value; **Holdings tab** = real units/NAV/current/P&L/XIRR; **SIPs tab + calendar** = real `listMyPurchasePlans` (installment_day→dots, next_installment_date→"next", amount→monthly, remaining_installments); **Transactions tab** = real buys + sells merged; **Orders tab** unchanged. **Performance trend chart + benchmark line DROPPED** — FP gives no historical NAV time-series or benchmark feed (user chose to remove rather than fake). Added refresh button. All report endpoints already existed on the backend (`ReportsController`+`ReportService`); this wires the frontend to them (previously only SWP used holdings). **Caveat:** holdings report is RTA/CAS-backed → on ONDC a freshly-bought folio may not appear until the RTA feed propagates (graceful fallback: holdings-report market value → returns-report current → units×nav). tsc + ng build clean.
- **2026-06-10 (Transactions detail: redemption-aware allotment cells)** — Made `txDetail()` context-aware: sells now show **"Redeemed Amount"** (from `redeemedAmount` proceeds) instead of "Purchased Amount", and the NAV-date cell is hidden for sells. Buys unchanged.
- **2026-06-10 (Buy detail: surface allotted_units / purchased_amount / purchased_price / allotted_nav_date)** — The buy txn detail showed Units + NAV (purchased_price) but not Purchased Amount or the allotment NAV date. `purchased_amount` was already stored — just exposed it. `allotted_nav_date` had NO column → full migration (`buy-allotment-navdate.sql`): ALTER `tbl_Fintech_MF_Purchase_Order` ADD `AllottedNavDate DATE`; updated `Usp_Upsert_Fintech_MF_Purchase_Order` (+@AllottedNavDate) and `Usp_Get_Fintech_MF_Purchase_Order_ByUser` (returns PurchasedAmount + AllottedNavDate appended at ordinals 26/27 so existing ordinals don't shift; also re-added the missing `o.Plan` at ord 15 to fix a latent ordinal drift). Backend: `MfPurchaseParser` captures `allotted_nav_date`; `MfPurchaseOrderUpsertArgs` + upsert param + `MfPurchaseRow` DTO + `ListByUserAsync` read ords 26/27 (FieldCount-guarded so it won't crash pre-migration). Frontend: `MfPurchaseRow` + `Transaction` carry purchasedAmount/allottedNavDate; mapRow populates them; txDetail shows **Purchased Amount** + **Allotment NAV Date** cells. Backend 0 errors; SPA tsc + ng build clean. **RUN `buy-allotment-navdate.sql` + restart API.**
- **2026-06-10 (Buy Simulate HIDDEN — FP rejects ONDC simulation)** — FP returned `400 "ONDC gateway orders can't be simulated"` on `POST /api/oms/simulate/orders/{old_id}`. All our buys are ONDC, so the per-buy Simulate button could never work. Hid it: `canSimulateOrder()` now returns false (button never renders), so the simulate action is gone from both the collapsed row and expanded panel for buys. (SIP installment-simulate is unaffected — separate flow.) SPA tsc + ng build clean.
- **2026-06-10 (Simulate moved to collapsed row; OTP "Verify" disabled = unticked consent)** —
  - **Simulate button now on the COLLAPSED buy row** (next to the sync icon, "⚡ Sim"), not just inside the expanded panel — that's why it "wasn't showing" (rows were collapsed). Same gating: `!isProduction()` (backend `Runtime:IsProduction==1?true:false` via `/api/config/runtime` — verified correct end-to-end) + non-terminal state. So with appsettings `IsProduction:0` it shows in sandbox.
  - **"Verify & Pay" stayed disabled after entering OTP** because the **nomination-consent checkbox was unticked**. Per the FP one-time-purchase doc Step 3, nomination consent is **mandatory** for a new folio (provide nominees OR opt out, and explicitly consent) — so the button requires BOTH the OTP (≥4 digits) AND the consent checkbox. Working as designed. (Hint line under the button was added then removed at user request.)
  - **"What if he doesn't want to nominate for this purchase?" — clarified:** Per the doc, "don't nominate" = explicit **Opt-out** (consent still required), and nomination is **account/MFIA-level, NOT per-purchase** — opting out clears the MFIA folio_defaults (`nominee1/2/3=null`, `show_nomination_status`) and therefore applies to **all future folios**, not just one purchase (there's one MFIA per investor; no per-purchase opt-out in FP's model). Opt-out removes the **nomination** (the slots), NOT the **related-party people** (FP has no delete; the pool persists, can be re-nominated later). Decision: keep Opt-out on the **Nomination page only** (it's an account-level choice, so it belongs there — not a misleading per-purchase toggle); the OTP modal just confirms the account's current nomination state. SPA tsc + ng build clean.
- **2026-06-09 (Transactions: per-buy Simulate (sandbox) + removed Dividends stat)** — (1) Added a **Simulate ⚡** button on buy-order rows, gated on `!isProduction()` (backend Runtime:IsProduction via getRuntimeConfig — already loaded on the page) and a non-terminal state (under_review/pending/confirmed/submitted). It calls new `mfPurchase.simulatePurchase(id,'SUCCESSFUL')` → `POST /api/mf/orders/purchases/{id}/simulate` (403 in prod) → fast-forwards the order, then refreshes the row. Lets you clear the stuck under_review test buys. (2) **Removed the "Dividends Earned / None yet / ₹0" summary stat** (we don't track dividends) — left Total Invested / Total Redeemed / Pending Orders. New i18n keys orderSimulated/orderSimulateFailed (en+ar). SPA tsc + ng build clean.
- **2026-06-09 (Transactions status filter = all FP stages)** — The status dropdown only had All/Processed/Pending/Failed (coarse) and filtered on the bucketed `tx.status`. Replaced with the **exact FP states**: All / Under Review / Pending / Confirmed / Submitted / Successful / Failed / Cancelled / Reversed, and `applyFilters` now matches the raw `tx.state`. So you can filter the buy list (which includes the many under_review test orders) by any real stage. We keep showing under_review orders (legitimate in-flight stage; abandoned ones drop to failed once FP expires them). Each row still expands to details + the staged timeline. SPA tsc + ng build clean.
- **2026-06-09 (Investor Profile "View profile" → read-only view, not the Created! splash)** — The KYC card's "View / Edit profile" routed to /investor-profile which, for an existing profile, only showed the "Investor Profile Created!" success splash (no editable form). Since FP profile fields (name/PAN/DOB) are write-once, a true edit isn't possible — so added a **read-only View**: `/investor-profile?view=1` shows the profile details as locked rows (name/PAN/DOB/gender/occupation/income/profile-id) with a "set on the registry, can't be changed" note and a Back button — no auto-advance, no celebration framing. Card link relabeled **"View profile"** and passes `?view=1`. The post-creation success splash still shows when arriving after actually creating the profile (no `?view`). SPA tsc + ng build clean.
- **2026-06-09 (OTP consent showed ALL stored nominees, not just the nominated set)** — The OTP modal listed irfsab 100% + omari 100% = 200%. Two causes, both fixed: (1) `loadNomineeConsent` read **`listNominees()`** (the full local pool) instead of **`getAttachedNominees(mfiaId)`** (the MFIA folio_defaults) — so it showed every stored person, not just who's actually nominated. Now reads the attached set only. (2) `getAttachedNominees` had `allocationPercentage: fd[...] ?? row.allocationPercentage` — the `??` fell back to the local stored % (100) when folio_defaults was null, inflating the total. Removed the fallback: % comes **only** from folio_defaults now. Net: the consent block shows exactly the nominees on the MFIA with their real %s. (Combined with the atomic SetNominees fix, a clean save then shows the correct ≤3 set summing to 100.) SPA tsc + ng build clean.
- **2026-06-09 (ROOT CAUSE of "total ≠ 100%": reattach left stale nominee slots — now an atomic all-slots PATCH)** — Purchase 400 "total percentage allocation of all nominees should add up exactly to 100%". Cause: `ReattachNomineesToMfiaAsync` used `UpdateAsync` (null-dropping serializer), so when attaching fewer nominees than before, the **unused slots' nulls were omitted → FP kept the STALE nominee** in that slot → totals broke (e.g. a leftover 50% + new 100%). **Fix:** new `IMfInvestmentAccountService.SetNomineesAsync(mfiaId, slots)` builds a complete folio_defaults with **all 3 slots explicit** (chosen = id/%/proof, unused = explicit null) and sends it via the **raw PATCH** (`PatchRawAsync`, nulls preserved) — one atomic update, no stale leftovers. Reattach now: 0 nominees → `ClearNomineesAsync` (opt-out); ≥1 → `SetNomineesAsync`. This is the "best" answer to clear-then-set: a single PATCH that nulls unused slots and sets chosen ones together. Proof-type still matches the present value (pan / minor→guardian). Backend 0 errors. **Restart API so the atomic PATCH is live.**
- **2026-06-09 (Minor purchase 400 → collect guardian contact+address; KYC CTA fixed when onboarding done)** —
  - **Purchase 400 "guardian_address, guardian_contact_detail.* cannot be null if nominee is a minor":** FP enforces guardian **email + isd + mobile + address** at folio creation for a minor nominee — beyond guardian name/PAN. **Built collection + send:** `CreateRelatedPartyRequest` now carries `guardian_email_address`, `guardian_phone_number` (hash `{isd,number}`), `guardian_address` (hash `{line1,city,state,postal_code,country}`) — new `PhoneHash`/`AddressHash` models, omitted-when-null, minor-only via `ToRelatedParty()`. `CreateNomineeRequest` + `CreateNominee` validate these mandatory for a minor (400 otherwise). Frontend: nomination add form now collects Guardian Email/Mobile + Address (line1/city/state/pincode) for minors (mandatory), sent on create. (Adults unaffected.) NOTE: these guardian fields are marked "Upcoming" in the related_party doc — if FP's sandbox doesn't accept them yet, a minor still can't transact; confirm against FP. Adult nominees work regardless.
  - **KYC "Continue Onboarding" showed even when done:** it only checked `hasProfile`. Added `onboardingComplete` (profile + bank + MF account) on the KYC page; when complete the CTA is **"Start Investing →"** (→ fund list) instead of "Continue Onboarding", and the auto-advance to /onboarding is skipped (stays on the KYC cards). SPA tsc + ng build clean; backend 0 errors.
- **2026-06-09 (KYC success no longer auto-flashes to All-Set when onboarding is complete)** — Explanation of the "shows cards then jumps after 1s": `autoAdvanceAfterSuccess()` (verified-KRA path) showed the KYC success screen for 2s then auto-navigated to `/onboarding`. Now it first checks onboarding status and **only auto-advances if onboarding is INCOMPLETE** (missing profile / bank / MF account). If the account is already active, it **stays on the KYC 4-card summary** (no flash to the All-Set page). Falls back to advancing on a status-fetch error. SPA tsc + ng build clean.
- **2026-06-09 (Guardian PAN now MANDATORY for a minor — fixes null proof on minor nominees)** — A minor nominee was attaching with `nominee1_guardian_identity_proof_type: null` because guardian PAN was optional and unset, so we sent no proof type. Per the FP doc a minor needs a guardian identity proof before folio creation, and guardian PAN is the only proof value we can send today. **Fix:** guardian PAN is now **required for a minor** — enforced in the nomination-page add form, the in-place dialog, AND the backend `CreateNominee` (400 if missing). So a minor can't be created/nominated without a guardian PAN, and the reattach then sends `nominee{n}_guardian_identity_proof_type: pan` (matching the value). Adults' own PAN stays optional (FP allows an adult nominee with no proof). Labels updated to "Guardian PAN * (identity proof)". Backend 0 errors; SPA tsc + ng build clean.
- **2026-06-09 (Opt-out "Added" stale fix + KYC success page = side-by-side cards)** —
  - **Opt-out still showed "Added":** the All-Set nominee badge read `status()?.hasNominee` (a stale DB flag) instead of the real attached list. Switched the badge + Edit/Add label to **`nominees().length`** (FP-synced via `getAttachedNominees`/`loadNomineesForDisplay`), so after opt-out (which nulls folio_defaults) it correctly shows "Not added". (What opt-out does: `removeAllNominees` → raw PATCH nominee1/2/3=null + `nominations_info_visibility=show_nomination_status`.)
  - **KYC success page now a responsive 2×2 card grid** (`max-w-4xl`, `grid md:grid-cols-2`): KYC/VERIFIED, Nominees, Bank, and a new **Investor Profile** card (name/PAN/DOB + View/Edit → investor-profile). Each card shows status + details + a navigate action. Replaces the stacked single-column cards. SPA tsc + ng build clean.
- **2026-06-09 (Nomination flow WORKING + fixed KYC "Nominee: Not Added" contradiction)** — End-to-end confirmed: 3 nominees attached to the MFIA (shakir 98%/minor, omair 1%, irfan 1% = 100%), shown on the All-Set card and the KYC Nominees card. UI consistency fix: the KYC VERIFIED summary card had a "Nominee: Not Added" row reading the **legacy `formData.nomineeName`** (never populated by the new flow), which contradicted the dedicated "Nominees (3 on your MF account)" card right below it. Removed that stale row from `kycSummary` — nominees are shown once, accurately, in their own card sourced from the MFIA folio_defaults. SPA tsc + ng build clean.
- **2026-06-09 (allocations endpoint returns 502 on FP reattach failure — UI stops + shows it)** — `POST /api/onboarding/nominees/allocations` was returning 200 even when the FP folio_defaults PATCH failed (best-effort reattach), so the UI couldn't cleanly stop. Now if the reattach outcome starts with "FP rejected"/"skipped", the endpoint returns **502 with the FP reason** (`Couldn't apply nomination to the MF account — …`), so the Nomination page's `commit()` catch surfaces it and does NOT claim success. NOTE re the latest test: the `sent:` payload still showed `aadhaar` → that was the OLD build still running (the PAN-derived-proof-type fix from the prior entry wasn't restarted). Rebuild + restart so the reattach sends `pan` (matching the PANs on the parties). Backend 0 errors; SPA tsc clean.
- **2026-06-09 (FP proof-type must MATCH an existing value — derive type from PAN, not hardcode aadhaar)** — FP rejected: *"nominee1_guardian_identity_proof_type: aadhaar, proof type should exists for the given nominee"*. Cause: we hardcoded the proof TYPE as `aadhaar`, but the related parties actually carry a **PAN** (irfan nomineePan, swapnil guardianPan) and no aadhaar value — FP requires the declared type to have a matching VALUE on the party. **Fix:** `ReattachNomineesToMfiaAsync` now derives the proof type from what's present — minor→guardian PAN, adult→nominee PAN → type `"pan"` when a PAN exists, else **omit the proof type entirely** (FP rejects a type with no value). Frontend `attachNomineesToMfia` no longer defaults to `aadhaar` — it sends a type only when the caller passes one; onboarding `attachNomineeToMfiaIfNeeded` now passes `'pan'` only when the party/guardian has a PAN, else omits. We only ever declare `pan` now (the only proof value we collect); aadhaar/passport remain unused until FP enables their value fields. Backend 0 errors; SPA tsc + ng build clean. **App team: the folio_defaults proof type must equal a proof value actually set on the related party — declare `pan` only when the party has a PAN; don't send a type otherwise.**
- **2026-06-09 (Nomination page: removed duplicate Done; surface the REAL save/FP error)** —
  - **Removed the duplicate "Done" button** — Section B's "Save nomination" already commits; it now also navigates back to KYC on success (calls `done()`). No separate Done at the bottom.
  - **Fixed false "saved" + show exact error:** `setNomineeAllocations` (and `removeAllNominees`) return 200 even when the FP folio_defaults reattach failed (best-effort), and the frontend was discarding the backend `message` and always showing "Nomination saved." Now the service returns `{ rows, message }`; `commit()` inspects the message — if it contains "FP rejected"/"failed"/"skipped" it shows the **exact backend/FP message in red** and does NOT navigate or claim success. Genuine success shows "Nomination saved" and returns to KYC. So the precise FP reason (e.g. "Nominee date_of_birth cannot be null", a minor missing guardian, etc.) now surfaces on the page. Updated the in-place dialog's `saveAllocations` to the new `{rows}` shape too. SPA tsc + ng build clean.
- **2026-06-09 (Nomination page fixes: signal-safe selection, Done commits, gated visibility, no delete)** —
  - **Selection now updates reliably.** Checkbox/% were `[(ngModel)]` mutating objects inside the signal array — which doesn't trigger the `selectedCount`/`allocTotal`/`allocOk` computeds. Switched to `[ngModel]` + `setSelected(id,val)` / `setPercent(id,val)` that **`.set()` a new array** so the UI + totals recompute. Single selection auto-defaults to 100%; deselect clears that row's %.
  - **"Done" now commits.** Previously Done just navigated. Now `done()` saves the current selection (`setNomineeAllocations` for chosen, or `removeAllNominees` opt-out if none) then returns to KYC. Shared `commit()` helper.
  - **Done visibility gated:** the Done button shows **only when ≥1 nominee is selected** (and is enabled only when allocations total 100). With nothing selected, the action is "Opt out of nomination" in Section B.
  - **Pre-selection from FP** already works: `reload()` reads the MFIA folio_defaults via `getAttachedNominees`, pre-checks attached nominees and shows their %. (Shows none currently only because the test MFIA was cleared/opted-out earlier.)
  - **Removed the per-person delete (bin) icon** — FP has no related-party delete; a person stays in the pool, you just don't nominate them. SPA tsc + ng build clean.
- **2026-06-09 (Nomination page: view + edit related parties, FP-synced)** — Section A rows are now **expandable**: clicking a person shows full FP-synced detail (name, relationship, DOB, Adult/Minor, PAN or Guardian + Guardian PAN). **Edit ("Add missing details")** is offered only for fields FP permits adding when still empty — **PAN for an adult / Guardian PAN for a minor** — via the existing `PATCH /api/onboarding/nominee` (`updateNominee`); name/relationship/DOB and any already-set PAN are read-only (write-once on the registry). Data stays in sync because the page loads via `listNominees()` (sync-first: pulls `GET /v2/related_parties?profile=` into our DB, then reads it) and re-syncs after every add/edit/remove. So: view = FP truth mirrored to DB; edit = add-only permitted fields pushed to FP + re-synced. SPA tsc + ng build clean.
- **2026-06-09 (Dedicated Nomination page: related-party POOL + choose ≤3, subset-attach)** — Per the FP two-object model and the request to split "manage people" from "choose nominees":
  - **New page `/app/mutual-funds/nomination`** (`NominationPageComponent`), two sections. **A — People (related parties):** the pool on the investor profile; add any number (name, DOB-mandatory, relationship, then DOB-derived: adult→PAN optional, minor→guardian name/relationship mandatory + guardian PAN optional), list with avatar/relationship/minor, remove (soft-delete). Name/relationship/DOB noted as write-once. **B — Choose nominees:** pick **up to 3** from the pool (checkboxes, capped at 3), set allocation % each (must total **100**), Save → MFIA folio_defaults; or **Opt out** (clears via raw-null PATCH). Pre-selects whatever is already attached.
  - **Subset-attach fix:** `ReattachNomineesToMfiaAsync` now takes an optional **explicit nominee-id list**; `SetNomineeAllocations` passes the chosen ids, so **deselected nominees are dropped** from folio_defaults (previously it attached ALL stored nominees regardless of selection).
  - **Pencils route to the page:** KYC nominee card "Manage/Add" and onboarding "All Set" nominee Edit now navigate to `/mutual-funds/nomination`. The purchase **OTP-modal** edit intentionally stays an in-place dialog (don't drop the user out of a mid-purchase flow).
  - **Identity proof (per related_party doc):** only `pan` (adult) / `guardian_pan` (minor) are settable FP fields today; aadhaar/passport/etc. are "Upcoming" so NOT sent. We send proof_type `pan` when a PAN exists on the party, else `aadhaar` type (sandbox-accepted); the actual aadhaar value collection waits on FP enabling it. Backend 0 errors; SPA tsc + ng build clean.
- **2026-06-09 (Consent contact now taken from the MF account, server-side)** — Per the FP nomination-consent doc ("OTP must go to the mobile/email stored against the primary investor on the investment account"), the purchase consent contact is now stamped **server-side** from our contact store (`_kycForm.GetUserContactAsync` — the same email/phone attached to the MFIA folio_defaults during account creation), instead of trusting the frontend's `kyc*` values / dummy fallbacks. Done in `MfOrdersController.UpdatePurchase` (overrides `request.Consent.Email/Mobile/Isd_Code` before forwarding to FP; refreshes the audit log) and in `SendConsentOtp` (resolves the account contact when the client didn't supply one). Single-holder product, so this is the primary investor's contact. Backend 0 errors. (We chose server-stamp over literally re-resolving the folio_defaults email_xxx/phone_xxx object IDs on the hot path — equivalent values, no extra FP round-trip.)
- **2026-06-09 (REAL root cause: our serializer was DROPPING the nulls — not FP. Fixed with a raw PATCH)** — Correction to the previous entry: FP was NOT the problem. `FinprimApiClient.JsonOptions` has **`DefaultIgnoreCondition = WhenWritingNull`** (global), so every null property is omitted from outbound JSON. So our "clear nominee" PATCH **never actually sent `nominee1:null`** — the field was omitted entirely, and FP correctly left it unchanged. (The remove-all response `sent:{...null...}` was a SEPARATE `JsonSerializer.Serialize(fd)` used only for the log message — it didn't reflect the real HTTP body.) **Fix (additive, no existing call touched):** new `IFinprimApiClient.PatchRawAsync<T>(path, rawJsonBody)` sends a verbatim JSON string (nulls preserved); new `MfInvestmentAccountService.ClearNomineesAsync(mfiaId)` builds the explicit-null folio_defaults body and uses it; `RemoveAllNominees` now calls `ClearNomineesAsync` (instead of the null-dropping `UpdateAsync`) and re-fetches to report whether `nominee1` actually cleared. The normal `PatchAsync`/`UpdateAsync` and all other FP calls are unchanged. Backend 0 errors. **App team note:** to clear an MFIA folio_defaults nominee you must send literal JSON null (`"nominee1":null`); a serializer that omits nulls will silently fail to clear.
- **2026-06-09 (CONFIRMED FP LIMITATION: a set nominee CANNOT be un-set to null)** — Definitive test: sent `nominee1:null` (verified in payload), FP returned `ok`, but the re-fetched MFIA **still showed `nominee1`** while `nominations_info_visibility` (a non-null value) DID change. Conclusion: **on a folio_defaults PATCH, FP treats `null` as "leave unchanged" — it ignores nulls.** So a nominee already set on the MFIA **cannot be removed/cleared**; it can only be **overwritten with a different nominee**. There is no documented clear/delete-nominee call. **Implication:** "Remove all" cannot truly zero out FP nominees; instead it (a) soft-deletes locally and (b) sets `nominations_info_visibility = show_nomination_status` — the SEBI/FP **opt-out declaration**, which is what actually governs nomination status. The lingering nominee ref on the account is cosmetic/uneditable. `RemoveAllNominees` now **re-fetches and reports honestly**: "opted out… FP keeps the existing nominee reference… cannot be unset, only replaced." **App team: you cannot delete a nominee on FP — replace it, or rely on the opt-out visibility flag.** Backend 0 errors.
- **2026-06-09 (Remove-all sends explicit nulls; outcome surfaced to UI)** — Made `POST /api/onboarding/nominees/remove-all` send an **explicit clear payload** (every `nominee{1,2,3}` + `_allocation_percentage` + `_identity_proof_type` + `_guardian_identity_proof_type` = null, `nominations_info_visibility = show_nomination_status`) directly via `_mfAccountService.UpdateAsync`, and **echo the exact JSON sent + FP result** in the response message. Frontend `removeAllNominees()` now returns `{ rows, message }` and the dialog displays the message under the Remove-All button. Confirmed payload sent (verified in response): all nominee fields null + opt-out visibility. OPEN VERIFICATION: whether FP treats `"nominee1": null` on PATCH as "clear" vs "ignore" — needs a post-PATCH MFIA re-fetch to confirm `nominee1` is actually null. If FP ignores nulls, the opt-out visibility flag is the SEBI-sanctioned no-nomination state regardless. SPA tsc + ng build clean.
- **2026-06-09 (Remove-all-nominees button + folio_defaults null-safety)** —
  - **"Remove all nominees" button** in the nominee dialog → `POST /api/onboarding/nominees/remove-all` (`kycService.removeAllNominees`): soft-deletes every nominee, then `ReattachNomineesToMfiaAsync` PATCHes the MFIA with **nominee1/2/3 = null** + `nominations_info_visibility = show_nomination_status` (opt-out). This is the explicit clear-nomination action; confirms via the response message.
  - **Null-safety fix:** the re-attach builds a fresh `FolioDefaultsInput`; previously its null comm/payout/demat fields would serialize and **wipe** those on FP. Added `[JsonIgnore(WhenWritingNull)]` to communication_*/overseas/payout_bank_account/demat_account so a nominee-focused PATCH never clears them — while the **nominee fields stay always-serialized** so a slot CAN be nulled to remove a nominee. Backend 0 errors; SPA tsc + ng build clean.
- **2026-06-09 (Onboarding "All Set" nominee card now FP-synced from the MFIA too)** — The "All Set" screen (`/onboarding` complete step) was showing nominees from the plain local list (often empty → "Added" with no rows). Switched it to `loadNomineesForDisplay(status)` which prefers **`getAttachedNominees(mfiaId)`** (syncs nominee details from FP + reads the live MFIA folio_defaults with allocation %) and falls back to the list when no MFIA yet. Used in both ngOnInit and `refreshStatus`, so after add/edit/delete the card reflects FP's actual attached nominees (name · % · Minor). Edit/Add still opens the in-place dialog. SPA tsc + ng build clean.
- **2026-06-09 (Confirmed: remove nominee = null the folio_defaults slot; KYC nominee card is sync-backed)** —
  - **Removing a nominee IS supported** (corrects an earlier overstatement): the Update MFIA doc shows folio_defaults is a normal PATCH hash and slots accept null. Setting `nominee{n}: null` (+ `_allocation_percentage`/proof null) removes that nominee from the account; nulling all three + `nominations_info_visibility: show_nomination_status` clears nomination. `FolioDefaultsInput` has no `[JsonIgnore]` on nominee fields, so unused slots ARE sent as null — meaning our `ReattachNomineesToMfiaAsync` already removes a trashed nominee from FP folio_defaults on re-attach. (The related_party OBJECT still can't be deleted — no DELETE API — but its nomination is removed, which is what matters.)
  - **KYC nominee card now sync-backed:** `getAttachedNominees(mfiaId)` syncs nominee details from FP (`syncNominees`, fallback local) and joins with the live MFIA folio_defaults (nominee ip ids + %), so the KYC page's dedicated Nominee card reflects FP's source of truth. SPA tsc clean.
- **2026-06-09 (KYC page shows MFIA-attached nominees; identity-proof-type clarified)** —
  - **folio_defaults now confirmed working** — MFIA fetch shows `nominee1=relp_…, nominee1_allocation_percentage=100, nominee1_identity_proof_type=aadhaar, nominations_info_visibility=show_all_nominee_names`. The full attach chain is correct.
  - **KYC page now displays the attached nominees.** New `GET /api/mf/accounts/{id}` passthrough (`kycService.getMfAccount`) + `getAttachedNominees(mfiaId)` which joins the MFIA folio_defaults (nominee ip ids + %) with names from our nominee store. The KYC success view's Nominee card now lists each attached nominee (avatar initial, name, relationship, Minor badge, allocation %) with a Manage/Add button → `?step=nominee`. Loaded on init via `loadAttachedNominees()`.
  - **identity_proof_type clarification (answer to "where did we take that?"):** we do NOT collect it from the user — `nominee{n}_identity_proof_type` is **hardcoded `aadhaar`** to satisfy FP's folio_defaults enum (and `guardian` variant for minors). FP validates only the *type* enum here, not an actual proof value; the related_party itself currently has no PAN/Aadhaar value collected. KNOWN GAP: if FP later enforces a real collected proof on the related_party before folio creation, we'd need to collect the actual proof (PAN/Aadhaar) per nominee. SPA tsc + ng build clean.
- **2026-06-09 (Auto-recover from nomination purchase errors — open nominee dialog inline)** — When a purchase/SIP create fails with a nomination-related FP error (message contains nominee / nomination / guardian / date_of_birth / allocation), the Invest flow now **detects it and opens the nominee dialog in place** (instead of a dead error). `maybeHandleNominationError(msg)` picks a specific guiding hint — DOB-null → "a nominee is missing a date of birth…", guardian → "a minor nominee needs guardian details…", allocation/100 → "allocation must total 100%…" — passed to the dialog via a new `[notice]` input that renders an amber banner at the top. The dialog opens with nominees **pre-filled** (DOB auto-populated, edit/trash available) so the user fixes it and saves; on save we reload nominees + re-resolve folio. Wired into both the lumpsum order-create catch and the SIP plan-create catch. SPA tsc + ng build clean.
- **2026-06-09 (Purchase 400 "Nominee date_of_birth cannot be null" — guard + recovery)** — A purchase failed with FP `400 "Nominee date_of_birth cannot be null"`. Cause: an earlier (pre-DOB-mandatory) nominee with a null DOB had been attached to the MFIA folio_defaults; FP validates the nomination at purchase time and rejects the whole order. **Fix:** `ReattachNomineesToMfiaAsync` now **filters out nominees with a null/blank DOB** before building folio_defaults, so an invalid party is never attached. Because `FolioDefaultsInput` serializes unset nominee slots as null, a re-attach **overwrites** all 3 slots — so re-running Save allocation (or any nominee action) re-PATCHes folio_defaults with only valid (DOB-having) nominees and **clears the bad one**. If no valid nominee remains, it sends opt-out (`show_nomination_status`, no nominees). **Recovery for the current broken MFIA:** restart API → soft-delete the DOB-less test nominee(s) → Save allocation (re-attaches clean) → purchase works. Backend 0 errors.
- **2026-06-09 (Minor nominee — full doc-compliant handling on add + edit)** — Per the FP related_party doc, a minor (DOB age<18): **PAN is NOT allowed** (adult-only), **guardian_name required**, guardian_pan optional, and the MFIA uses the **guardian** identity-proof field. Implemented end-to-end:
  - **Add form:** DOB mandatory (capped at today); minor derived from DOB; **Nominee PAN field hidden for a minor** (shown only for adults, per doc); Guardian Name + Relationship shown and **mandatory** when minor; guardian PAN optional.
  - **Edit form:** DOB is now **auto-populated** (pre-filled, shown locked — FP write-once) with a "Minor" hint; PAN-vs-guardian field choice derived from DOB (not the stale stored flag).
  - **Backend hardening:** `CreateNomineeRequest.ResolveMinor()` derives minor from DOB (matches FP, ignores a wrong client flag); `ToRelatedParty()` sends PAN only for adults and guardian fields only for minors; `CreateNominee` now requires DOB for all, requires guardian name/relationship for minors, and persists the DOB-derived minor flag (nulls nominee PAN for minors). MFIA re-attach already uses `IsMinorByDob` → sends `nominee{n}_guardian_identity_proof_type` for minors.
  - Backend 0 errors; SPA tsc + ng build clean. **App team for a minor nominee: require DOB → if <18, hide nominee PAN, require Guardian Name + Relationship, and use the guardian identity-proof field on folio_defaults.**
- **2026-06-09 (Minor nominee → guardian proof field; DOB + guardian now mandatory in UI)** — After the enum fix, FP returned: *"Cannot set identity proof type if nominee is a minor"* for a DOB-2022 nominee. Causes + fixes:
  - **Minor uses the GUARDIAN proof field.** For a minor, FP wants `nominee{n}_guardian_identity_proof_type` (NOT `nominee{n}_identity_proof_type`). Backend `ReattachNomineesToMfiaAsync` now computes minor from DOB (`IsMinorByDob`, age<18) and sets the guardian proof field for minors; the frontend `attachNomineesToMfia` takes an `isMinor` flag and does the same.
  - **Stored `isMinor` was wrong** (a DOB-2022 nominee had `isMinor:false`) because the old dialog used a manual checkbox the user left unchecked. **Fixed:** the dialog now (a) makes **DOB mandatory**, (b) **derives minor from DOB** (no manual checkbox), (c) auto-shows + **requires Guardian Name + Guardian Relationship** when the DOB is a minor, and (d) caps DOB at today. `isMinorDob()` drives display, validation, and the createNominee payload.
  - **Data caveat:** the existing test nominee "irfan" (DOB 2022) was created as non-minor with NO guardian_name on the related_party. Even with the guardian-proof-field fix, FP may still reject because a minor needs guardian details on the party (which FP related_party PATCH can add if empty). Such bad test rows should be soft-deleted and re-added with a real DOB + guardian. Backend 0 errors; SPA tsc + ng build clean. **App team: collect DOB for every nominee; derive minor from DOB; if minor, Guardian Name + Relationship are mandatory and the GUARDIAN identity-proof field is used on folio_defaults.**
- **2026-06-09 (ROOT CAUSE of null folio_defaults: wrong FP enum values — fixed)** — Surfacing the swallowed re-attach error (return it in the API message) revealed FP was 400-rejecting EVERY folio_defaults nominee PATCH with two enum errors:
  - `nominations_info_visibility: 'shown'` → invalid. FP supports **`show_all_nominee_names`** | **`show_nomination_status`** (case-sensitive). We now send `show_all_nominee_names` when nominating and `show_nomination_status` for opt-out.
  - `nominee{n}_identity_proof_type: 'AADHAR'` → invalid. FP supports **`pan`** | **`aadhaar`** | **`passport`** | **`driving_licence`** (lowercase, case-sensitive). Now send `aadhaar`.
  - **This is why the MFIA folio_defaults always came back all-null** — every attach (at MFIA creation and every re-attach) was rejected and the error swallowed. Fixed in ALL call sites: backend `ReattachNomineesToMfiaAsync`; frontend `kyc.service.attachNomineeToMfia` / `optOutNominees` / `attachNomineesToMfia`; onboarding `attachNomineeToMfiaIfNeeded`. `ReattachNomineesToMfiaAsync` now returns its outcome and `POST /api/onboarding/nominees/allocations` echoes it in the response message (`folio_defaults: ok|skipped|FP rejected: ...`) for diagnosis. Backend 0 errors; SPA tsc clean. **After restart, Save allocation should now actually populate nominee1 on the MFIA** (pending any separate FP identity-proof-collected requirement at folio creation). **App team: folio_defaults enums are `show_all_nominee_names`/`show_nomination_status` and `pan|aadhaar|passport|driving_licence` — all lowercase/case-sensitive.**
- **2026-06-09 (Allocation reworked to ACCOUNT-LEVEL set + root-cause: MFIA folio_defaults nominees were null)** —
  - **Root cause found:** a fetched MFIA showed `folio_defaults.nominee1=null … nominations_info_visibility=null` — i.e. **nominees were never attached to the MFIA on FP.** Reason: the original attach (`attachNomineeToMfiaIfNeeded`) ran only at MFIA-creation time from in-memory `nominees()`/`capturedNomineeIp`; if nominees were added later / in another session, folio_defaults stayed null. Today's `ReattachNomineesToMfiaAsync` (fires on create/update/delete/set-allocations, reads MFIA id from onboarding status) fixes this going forward — the next nominee add or allocation save PATCHes folio_defaults. Existing null MFIAs self-heal on the next nominee action (or call `POST /api/onboarding/nominees/allocations`).
  - **Allocation is account-level (per FP), reworked accordingly.** FP's related_party object has **no allocation field** (confirmed from the live list response) — allocation lives ONLY on MFIA folio_defaults as a SET that must total 100. So allocation % was **removed from the per-nominee add/edit forms** and moved to a single **Allocation panel** in the dialog: lists all saved nominees, a % input each, live total, one **Save allocation** (enabled only at 100%). New backend `POST /api/onboarding/nominees/allocations` (`SetNomineeAllocationsRequest`) validates ≤3 / each >0 / total=100, persists each %, then re-attaches folio_defaults to FP. Frontend `kycService.setNomineeAllocations()`. This fixes the deadlock (couldn't go 100+1 → 60+40 one at a time) and the "won't accept 0" confusion (0 isn't valid — a 0% nominee shouldn't be a nominee). Single nominee auto-100. SPA tsc + ng build clean; backend 0 errors.
- **2026-06-09 (Nominee dialog: validate SAVED-set total = 100; purpose of allocation clarified)** —
  - **Bug:** the dialog only summed DRAFT nominees, so a saved set like 100% + 1% = 101% was accepted with no warning. **Fix:** added `savedTotal()` / `combinedTotal()` / `savedInvalid()` computeds; a persistent amber banner now shows when the **saved** nominees don't total 100% ("Saved allocations total 101% — must add up to 100%. Use ✏️ to fix"). The pencil `saveEdit` now validates that this nominee's % + the OTHER saved nominees' %s == 100 (and ≤100), blocking an invalid save with a precise message.
  - **Purpose of allocation % (answer to "why collect it?"):** it is the **regulatory split** sent to the AMC/registry — on death, holdings are divided among nominees by these %s. It is written to the **MFIA folio_defaults** and pushed to FP (via attach + the re-attach fix), NOT primarily a dashboard field. The onboarding "All Set" card shows it; the holdings/portfolio dashboard has no nominee section yet (that screen is still mock) — surfacing nominee+% on the real portfolio dashboard would be a separate feature if wanted. SPA tsc + ng build clean.
- **2026-06-09 (BUGFIX: allocation % now pushed back to FP MFIA folio_defaults on edit/delete/add)** — Caught a real drift bug: editing a nominee's allocation % (pencil), soft-deleting, or adding a nominee after the MFIA exists only wrote to **our DB** — it never re-PATCHed the **MFIA folio_defaults** on FP. So the local % and FP's actual nomination drifted apart (FP reads nomination % from folio_defaults at folio creation). **Fix:** added `ReattachNomineesToMfiaAsync(userId)` in `OnboardingController` — it loads the current visible (non-hidden, ≤3) nominees, normalises allocation %s to total 100 (single ⇒ 100; even-split fallback), builds `FolioDefaultsInput` (`nominee1/2/3` + `nominee{n}_allocation_percentage` + `_identity_proof_type=AADHAR`, or `nominations_info_visibility=opted_out` when empty), and PATCHes the MFIA via `_mfAccountService.UpdateAsync`. Now called at the end of **CreateNominee, UpdateNominee (pencil), and DeleteNominee (soft-delete)** — best-effort/logged, never throws. No-ops cleanly when the MFIA doesn't exist yet (first-time onboarding still attaches at MFIA creation on the frontend). **Clarification on scope:** allocation % lives on the **MFIA folio_defaults** and is inherited by each new folio — with our one-MFIA-per-investor model it is effectively **per investor**, applied to all funds (NOT settable per purchase — the Create MF Purchase API has no nominee/% field; FP pulls nomination from the MFIA at new-folio creation). Existing folios keep their original nomination; the re-attach updates the defaults applied to future folios. Backend builds clean.
- **2026-06-09 (Nominee soft-delete to clean dupes; PATCH 405 = restart)** —
  - **`PATCH /api/onboarding/nominee` → 405:** the running API predated the `UpdateNominee` action. It exists in source and compiles — **rebuild + restart FinprimApi** and the 405 clears. (Same for the new DELETE below.)
  - **Soft-delete nominee (the "how to remove 4 dupes" answer).** FP has **no related-party delete**, so we soft-delete locally: new `IsHidden BIT` column on `tbl_Fintech_Investor_Nominee` (added by the updated `nominee-multi-columns.sql`); `DeleteNomineeAsync` sets `IsHidden=1`; `ListNomineesAsync` filters `IsHidden=0`. New endpoint **`DELETE /api/onboarding/nominee/{nomineeId}`** + `kycService.deleteNominee()`. Dialog now has a **trash icon** next to the pencil on each saved nominee → soft-deletes and refreshes in place. **Behaviour (per the requested model):** sync-on-pageload stays the same; only non-hidden nominees show; the upsert's UPDATE branch never touches `IsHidden`, so **re-sync from FP keeps a hidden nominee hidden**. **Re-adding the same person** creates a NEW related_party (new NomineeId) → a fresh, visible nominee (not auto-hidden). Hidden nominees don't count toward the 3-limit. **This applies across all funds** — nominees are per investor profile (attached to MFIA folio_defaults), not per fund.
  - **Builds clean** (backend 0 errors; SPA tsc + ng build). **Run the updated `nominee-multi-columns.sql` (now also adds IsHidden) + restart API.**
- **2026-06-09 (Nomination consent now validates allocation % = 100 / ≤3; PAN labelled optional; % surfaced)** —
  - **Allocation validation at the OTP/consent step.** Before the holder can tick the consent checkbox, the nomination set must be valid: **≤3 nominees, every nominee has a positive allocation %, and the total = 100%** (opting out is always valid). `nominationProblem()`/`nominationValid()` computeds drive it. When invalid, the consent block shows an amber message (e.g. "Allocations total 0% — they must add up to 100%. Tap Edit to fix", or "You have 4 nominees but a folio allows at most 3…"), the **checkbox is disabled**, and the **Verify & Pay** button stays disabled (`isNewFolio() && (!ack || !nominationValid())`). An **Edit nominees →** link opens the in-place dialog. This is what the screenshot needed: 4 dupes with no % were being accepted — now blocked with guidance.
  - **Allocation % surfaced everywhere.** OTP consent block + the dialog's saved-nominee rows now always show the % (`50%` in emerald) or a **`no %` / `set %`** amber hint when missing — instead of silently showing nothing.
  - **PAN is optional (per doc `pan no`).** The edit form's Nominee PAN / Guardian PAN labels now read **"(optional)"** when empty (and "(locked)" when already set on FP). The placeholder `ABCDE1234F` is just a format hint, not a requirement. (The add-draft form already labelled them optional.)
  - **Note on >3 / duplicate test nominees:** FP has no delete for related parties, so extras (e.g. the repeated test "irfan" rows) can't be removed via API — they need a DB cleanup. The validation correctly blocks consent until the set is ≤3 with %s totalling 100. SPA tsc + ng build clean.
- **2026-06-09 (Nominee load is now sync-first from FP; passthrough forgives profile id)** —
  - **502 on `GET /api/onboarding/related-parties/{id}` with an `invp_` id:** that path expects a **related-party id (`relp_xxx`)**, not a profile id — FP correctly 404s ("No related_party found"). Fixed to be forgiving: if `{id}` starts with `invp_`, it now lists that profile's parties (`GET /v2/related_parties?profile=`) instead of 404ing. To list a profile's nominees use `GET /api/onboarding/related-parties?profile=invp_xxx`.
  - **"On page load, are we hitting FP and storing parties?" — previously NO; now YES.** `syncNominees()` existed but was never called; page load only read our DB (`GET /api/onboarding/nominees`). Changed `kycService.listNominees()` to be **sync-first**: it POSTs `onboarding/nominees/sync` (FP `GET /v2/related_parties?profile=` → upsert into `tbl_Fintech_Investor_Nominee`) and returns the merged list, with a **catchError fallback** to the plain local DB read so a transient FP error / missing-profile (400) never blanks the UI. Added `listNomineesLocal()` for callers that want the no-sync read. Net effect: nominees created outside the app, or a stale local store, now self-heal on load. SPA tsc + backend build clean.
- **2026-06-09 (Nominee edit pencil + direct FP related-party passthrough; no delete)** —
  - **Direct FP passthrough endpoints (Swagger, raw FP response, no DB write):** `GET /api/onboarding/related-parties?profile=invp_xxx` → FP `GET /v2/related_parties?profile=` (profile auto-fills from the logged-in investor if omitted), and `GET /api/onboarding/related-parties/{id}` → FP `GET /v2/related_parties/{id}`. The list passthrough is the one that "directly hits" FP with the `profile` param.
  - **Edit pencil on saved nominees — scoped to what FP actually allows.** Per the doc, **name/relationship/DOB/PAN are immutable once set** (PATCH can only ADD previously-empty non-mandatory fields), and there is **NO delete endpoint** for related parties. So: a **pencil** appears on each saved nominee (no delete button, since FP has none). Editing opens an inline form where **allocation %** is editable (local/MFIA-side), **PAN** is editable only if it was empty + adult, **guardian PAN** editable only if empty + minor; **name/relationship/DOB are shown disabled** ("set on the registry, can't be changed"). Save → `PATCH /api/onboarding/nominee` (`UpdateNomineeRequest`) which forwards ONLY newly-filled addable fields to FP `PATCH /v2/related_parties` (id in body, per doc) and updates % locally; the backend guards by comparing against the stored row so it never sends a change to an already-set field.
  - **Backend:** fixed `RelatedPartyService.UpdateAsync` to PATCH `/v2/related_parties` with **id in the body** (was wrongly `/{id}` in the URL); new `UpdateRelatedPartyRequest`/`UpdateNomineeRequest`; `NomineeRow` + `ListNomineesAsync` now also return `Profile`. Frontend `kycService.updateNominee()`; dialog `startEdit/saveEdit/cancelEdit`.
  - **Why a pencil and not free-edit:** FP immutability means a true "change the name/relationship" isn't possible — the honest model is add-missing-fields + adjust %. Builds clean (backend 0 errors; SPA tsc + ng build). **App team: nominee edit = adjust % + fill previously-empty PAN/guardian only; no delete (FP doesn't support it); name/relationship/DOB are write-once.**
- **2026-06-09 (Fixes round: folio-exists, real FP related-party schema, nominee 500, orphaned-order cancel)** — Five reported issues:
  - **Task 4 — `GET /api/onboarding/nominees` 500 "Invalid column name 'AllocationPercentage'…":** the multi-nominee columns weren't added yet. **Fix = run `nominee-multi-columns.sql` in SSMS** (adds the 6 columns + updates the upsert/list SPs). Not a code bug — migration ordering. This also explains **Task 4b** (dialog "asking to add nominee again even though I have one"): `listNominees()` was erroring, so the dialog saw zero saved nominees → blank add form. After the migration, your existing nominee loads under "Saved" and the dialog shows "+ Add another nominee" (up to 3). Each new nominee is its own party — you fill name/relationship/%/(guardian if minor) per nominee.
  - **Task 1 — `/api/mf-folios/exists` returned 502 `GET /v2/folios/exists 404 (Gateway: URL not available)`:** the running build pre-dated the `Exists` action, so `exists` was matched by `[HttpGet("{id}")]` and sent to FP as a folio id. Also FP `/v2/folios` 404s on ONDC entirely. **Fix:** removed the live FP `/v2/folios` call from `Exists`; `HasFolioForSchemeAsync` now checks **`tbl_Fintech_MF_Purchase_Order` (Scheme + non-empty FolioNumber)** UNION `tbl_Fintech_Folio` — our own mirror, reliable on ONDC. (Restart API so the `Exists` route is live.)
  - **Task 3 — `exists:false` even though a folio exists:** same root cause — it was hitting the broken FP path / empty folio mirror. The purchase-order-table check fixes it: any successful buy in that scheme (which carries Scheme + FolioNumber) now returns `exists:true`. **Restart + the purchase order must have synced its FolioNumber** (it does once `successful`).
  - **Task 2 — real `/v2/related_parties` schema implemented + FP→DB sync.** Per the doc you provided, the confirmed create fields are `profile, name, date_of_birth, relationship, pan, guardian_name, guardian_pan` (PAN only if adult; guardian_name/guardian_pan only if minor; `relationship` enum has **no `sibling`/`other`** — uses `brother/sister/...` and **`others`**). Changes: `CreateRelatedPartyRequest` now carries `pan/guardian_name/guardian_pan` (omitted-when-null); `ToRelatedParty()` sends PAN for adults and guardian fields for minors. Relationship pickers (dialog + onboarding step) updated to the exact FP enum. **New `POST /api/onboarding/nominees/sync`** → calls FP `GET /v2/related_parties?profile=` and upserts every party (incl. pan→NomineePan, guardian_name, guardian_pan) into `tbl_Fintech_Investor_Nominee`; frontend `kycService.syncNominees()`. Note: FP **PATCH /v2/related_parties** exists (update only adds non-mandatory fields; name/relationship/dob/pan are immutable once set) — true edit can now be wired if needed.
  - **Task 5 — order appearing in transactions after closing the OTP popup (never paid).** Correct behaviour explained: the mf_purchase is created on FP **before** the OTP step, so it legitimately exists as `under_review`/`pending`. **Fix:** `cancelOtpModal()` now calls **`POST mf/orders/purchases/{id}/cancel`** (FP `/v2/mf_purchases/{id}/cancel`, which already existed) for lumpsum, so abandoning the OTP cancels the stray order instead of leaving it in the list. (SIP has nothing to cancel there — the plan is created only after OTP.) Also: the transactions-list **"Cancel Order" and "Download Slip" buttons were dead placeholders (no click handler).** Wired **Cancel Order** → `cancelOrderRow()` (only shown for buys in `under_review`/`pending`, calls the cancel API, refreshes the row); **removed the non-functional Download Slip** button (slip generation isn't built — better to hide than show a dead button). **View Fund** works. New i18n keys: `actions.cancelling`, `confirm.cancelOrder`, `msg.orderCancelled/orderCancelFailed` (en + ar).
  - **Build:** backend 0 errors; SPA `tsc` + full `ng build` clean. **App team: a buy order is created before OTP — cancel it if the user abandons consent; only show Cancel Order while under_review/pending.**
- **2026-06-09 (Multi-nominee — up to 3, allocation %, minor→guardian, PAN + authoritative new-folio detection)** — Two things resolved completely:
  - **New-folio detection is now authoritative (not holdings-report guessing).** Added `GET /api/mf-folios/exists?scheme=<ISIN>&investmentAccountId=<mfia>` → `FoliosController.Exists`: refreshes folios live from FP for the MFIA, persists them, then checks **`tbl_Fintech_Folio` WHERE UserId=@U AND Scheme=@ISIN** (`FolioDbService.HasFolioForSchemeAsync`). Returns `{ scheme, exists }`. The Invest page (`resolveExistingFolio()`) now calls this instead of parsing the variable holdings-report shape. So the nomination-consent block shows ONLY when there's no existing folio for **that exact scheme**. **Answer to "I already have this fund — should it show nominee?":** if you hold a folio in that ISIN → No (existing folio, nomination already set); if not → Yes (new folio).
  - **Full multi-nominee build (up to 3).** Per FP doc: Name, DOB (mandatory if minor), Allocation %, Relationship, Guardian Name/Relationship (mandatory if minor), Nominee PAN (optional), Guardian PAN (optional).
    - **DB:** `nominee-multi-columns.sql` adds `AllocationPercentage, NomineePan, IsMinor, GuardianName, GuardianRelationship, GuardianPan` to `tbl_Fintech_Investor_Nominee`, and `CREATE OR ALTER`s `Usp_Upsert_Fintech_Investor_Nominee` (now persists all fields + returns local Id) and a new `Usp_List_Fintech_Investor_Nominee`. **RUN THIS in SSMS before testing.**
    - **Backend:** `CreateNomineeRequest` (full payload) → controller validates minor→guardian/DOB mandatory, **forwards only the FP-confirmed 4 fields** (`profile/name/date_of_birth/relationship`) to `/v2/related_parties` via `ToRelatedParty()`, and persists the rest locally (`UpsertNomineeAsync` extended). `ListNomineesAsync` + `NomineeRow` now return all fields. **Why only 4 to FP:** the FP related-party doc is auth-gated (403/404) and guessing guardian/PAN field names re-triggers the 502 — so those are stored locally, ready to forward once Cybrilla confirms the names. **No 502 risk.**
    - **MFIA wiring:** `kyc.service.attachNomineesToMfia` sets `nominee1/2/3` + `nominee{n}_allocation_percentage` + `nominee{n}_identity_proof_type` on folio_defaults (the model already supported all three slots). `attachNomineeToMfiaIfNeeded` now reads the full stored nominee list and `resolveAllocations()` ensures the set totals 100 (even split fallback).
    - **UI:** `NomineeEditDialogComponent` rewritten as a multi-nominee manager — lists saved nominees, "+ Add another nominee" up to 3, per-nominee allocation % (must total 100, live total shown), minor checkbox → conditional mandatory Guardian Name/Relationship + optional Guardian PAN, optional Nominee PAN. Opens in place (OTP modal + All Set). The OTP consent block + All Set list now show each nominee's **% / Minor** badge.
  - **Known limitation (unchanged):** editing an existing nominee creates a new record (FP related-party *update* not wired); guardian/PAN held locally pending FP field-name confirmation. **App team: collect up to 3 nominees with allocation % (total 100) + minor→guardian mandatory fields; detect new-vs-existing folio per scheme via `/api/mf-folios/exists`.**
- **2026-06-09 (Nominee Edit — in-place dialog, no navigation)** — UX fix: clicking **Edit** on a nominee (from the purchase OTP modal AND the onboarding "All Set" card) used to navigate into the onboarding stepper, with no way back to where you were (you'd lose your place / the mid-purchase OTP step). Built a reusable standalone **`NomineeEditDialogComponent`** (`features/mutual-funds/shared/nominee-edit-dialog.component.ts`) — an overlay dialog (name / DOB / relationship, pre-filled when editing) that opens **in place** over the current screen. Inputs: `existing` (NomineeRow|null → add vs edit) + `profileId` (ip_xxx). On save it calls `KycService.createNominee` and emits `(saved)`; the parent reloads its nominee list and closes the dialog — **the user never leaves the page**. Wired into: (a) the Invest OTP modal (`goEditNomination()` now opens the dialog instead of routing; `onNomineeSaved()` reloads `loadNomineeConsent()`); (b) the onboarding "All Set" card (`openNomineeDialog()` / `onNomineeDialogSaved()` → `refreshStatus()`). **Same known limitation as before:** save creates an updated nominee record (FP related-party *update* not wired) rather than mutating the original. SPA `ng build` clean. **App team: edit nominee should be an in-place dialog over the current screen, not a navigation that drops the user out of the purchase/OTP flow.**
- **2026-06-09 (Nomination consent — gated to NEW FOLIO only + single-nominee % fix + editable nominee)** — Follow-up to the Step-3 consent work, fixing three issues found in testing:
  - **"Applicable only for new folio creation" — now enforced.** The FP doc scopes Step 3 to new-folio purchases. We were showing the nomination block on *every* purchase. Fixed: on Invest load `resolveExistingFolio()` calls `getHoldingsReport(mfiaId)` and checks whether a folio already exists for **this** scheme/ISIN. `isNewFolio()` = true only when no existing folio. The OTP modal renders the nomination block **only for new folios**; for a top-up into an existing folio it shows a one-line "Adding to your existing folio — nomination was already set" note, the **ack checkbox is not required**, and **no consent-audit row is written**. Fail-safe: if holdings can't be resolved, treat as new folio (show the block). **Meaning of "new folio":** a folio is the investor's account number with one AMC; the first investment in a scheme creates a new folio (nomination captured here); repeat purchases into the same folio don't create one (nomination already on file → skip Step 3).
  - **Single-nominee "100%" was misleading.** We hard-coded `allocationPercentage:100` for one nominee even though the user never entered a %. Removed — a single nominee now shows **"Sole nominee"** (no fake %). Real allocation % remains MFIA `folio_defaults`-side and part of the deferred multi-nominee build.
  - **"Edit nominee shows nothing to edit" — fixed.** The nominee step opened a blank add form. Now both entry points (`?step=nominee` handler and the All-Set `goToNomineeStep()`) **pre-fill `nomineeForm`** from the first stored nominee (name/DOB/relationship), the step header switches to **"Edit Nomination"** and shows "Current nominee: <name> (<relationship>)". **Known limitation:** saving an edit calls `createNominee` again → a NEW `/v2/related_parties` object (new NomineeId) → a second nominee row rather than an in-place update, because FP related-party *update* isn't wired yet (same blocker as the full multi-nominee build / 502 schema). So Edit currently lets you review + re-submit corrected details, not mutate the original record.
  - **Files:** `mutual-fund-invest.component.ts` (`hasExistingFolioForScheme`/`isNewFolio`, `resolveExistingFolio`, `@if(isNewFolio())` around the block, ack gating + audit gated on `isNewFolio()`, "Sole nominee" display); `onboarding.component.ts` (pre-fill on `?step=nominee` + `goToNomineeStep`, "Edit Nomination" header). SPA `tsc --noEmit` clean. **App team: only collect nomination consent on new-folio purchases; skip it for top-ups into an existing folio.**
- **2026-06-09 (One-time purchase Step 3 — nomination consent now shown at OTP + audit-stored)** — Audited our purchase flow against the FP one-time-purchase doc (`/mf-transactions/onetime-purchases/`). Findings + fix:
  - **Doc conformance verdict:** Steps 1–2 (scheme/eligibility, create purchase), 4 (consent PATCH email+mobile + state=confirmed), 5 (payment), 6 (track) were already correct. **Fresh KYC is NOT part of the purchase sequence** — the doc lists "investor is KYC compliant" + "investment account exists" only as **preconditions** to Step 2; fresh KYC is the separate pre-verification + POA KYC-Form (`type=fresh`) flow done earlier. Our `isReady()` gate (`investorProfileId + hasBankAccount + verified + hasMfAccount`) enforces those preconditions. So: **purchase ≠ KYC; no fresh KYC inside purchase, by design.** ✅
  - **The one real gap (Step 3):** the doc's Step 3 is "send OTP **AND obtain consent for the nomination details**" — the OTP *is the nomination consent*. We were sending the OTP and PATCHing only `{email, isd, mobile, otp}` (contact consent), with no nomination choice surfaced at that point. Nominees were collected only in onboarding.
  - **Fix (chosen approach: show nominee summary + opt-out ack at the OTP step):** the Invest **OTP modal** (`mutual-fund-invest.component`) now renders a **Nomination** block above the OTP input — lists the investor's nominees (Name · Relationship · allocation% where known) **or** an explicit **"You are opting out of nomination for this folio"** line, plus an **acknowledgement checkbox** that gates the Verify button (`!nominationAck` ⇒ disabled). Applies to **both lumpsum and SIP** (shared modal). An **Edit / Add nominee** link jumps to `?step=nominee`. Nominees loaded via `kycService.listNominees()` on init (`loadNomineeConsent()`).
  - **Audit storage (chosen: persist a record):** on successful consent (lumpsum after the consent PATCH; SIP after plan create) the SPA fire-and-forget POSTs **`POST /api/mf/orders/nomination-consent`** with `{ transactionRef, transactionType: purchase|plan, choice: provided|opted_out, nominees[], consentEmail/Mobile/Isd }`. Backend `MfOrdersController.RecordNominationConsent` writes it to the existing **MF order log** (`step="nomination_consent"`, status `provided|opted_out`, full snapshot in `RequestJson`) — **no new table**. Satisfies the doc's "Store all the consent-related information for audit purposes." Failure never blocks the order.
  - **Files:** `mf-purchase.service.ts` (`recordNominationConsent` + `NominationConsentRecord`/`NominationConsentNominee`); `kyc.service.ts` (`NomineeRow` reused); `mutual-fund-invest.component.ts` (modal block, signals, `loadNomineeConsent`, `goEditNomination`, `recordNominationConsentAudit`, ack reset on cancel); backend `MfOrdersController.cs` (`POST nomination-consent`), `Models/Mf/MfOrderModels.cs` (`NominationConsentRequest`/`NominationConsentNominee`). Backend builds clean; SPA `tsc --noEmit` clean. **Restart FinprimApi** to expose the new endpoint.
  - **Note (allocation %):** the OTP block shows 100% for a single nominee; multi-nominee % is MFIA-side (`folio_defaults`) and still part of the deferred full-multi-nominee build. **App team: mirror this — show the nomination choice (nominees or opt-out) and require acknowledgement at the consent/OTP step for any new-folio purchase or SIP, and store the consent for audit.**
- **2026-06-08 (Nominee — full gap analysis vs FP doc; full build planned)** — Read the FP nominee requirements (SIP create-monthly-sip + onetime-purchases step 3). The doc requires these per nominee: **Nominee Name; Nominee DOB (mandatory if minor); Allocation Percentage; Nominee Relationship; Guardian Name (mandatory if minor); Guardian's Relationship (mandatory if minor); Nominee PAN (optional); Guardian PAN (optional, if minor)** — up to **3** nominees, OR an explicit **opt-out**; consent for the choice is **mandatory**.
  - **Two-object model (verified in code):** the nominee itself is a Cybrilla **`/v2/related_parties`** object; the **allocation % + nominee linkage** live on the **MFIA `folio_defaults`** (`nominee1/2/3` + `nominee1/2/3_allocation_percentage` + `nominee*_guardian_identity_proof_type` on `FolioDefaultsInput`/`MfInvestmentAccountModels`).
  - **GAPS (what we currently do NOT do):** our `CreateRelatedPartyRequest` sends only `{ profile, name, date_of_birth, relationship }`; we do NOT collect/send **allocation %, Guardian Name, Guardian relationship, Nominee PAN, Guardian PAN**, do NOT enforce **minor→guardian mandatory**, support only a **single nominee** (not up to 3), do NOT wire **allocation %** into the MFIA, and `UpsertNomineeAsync` persists only name/DOB/relationship. Nomination **consent decision** (add/opt-out) is built, but a **folio-nomination-specific consent OTP** + **dedicated consent-audit storage** are not.
  - **Decision:** build the **full doc-compliant nominee** (up to 3, allocation %, minor/guardian conditional fields, PANs, MFIA % wiring, multi-nominee list view) — in progress.
- **2026-06-08 (Nominee API facts: % is NOT on create; 502 diagnosis)** — Clarifications + a diagnostic fix for the nominee flow:
  - **Allocation % is NOT part of creating a nominee.** A nominee is a Cybrilla **`/v2/related_parties`** object. Our create endpoint `POST /api/onboarding/nominee` → `RelatedPartyService.CreateAsync` → `POST /v2/related_parties` sends **only** `{ profile, name, date_of_birth, relationship }` (`CreateRelatedPartyRequest`). There is **no percentage field on the nominee create.**
  - **Allocation % is set on the MF Investment Account, not the nominee.** It lives on `MfInvestmentAccountModels` as `nominee1_allocation_percentage` / `nominee2_…` / `nominee3_…` — applied when nominees are linked to the MFIA (`folio_defaults`). For multiple nominees the percentages must total **100**; a single nominee is implicitly **100%**.
  - **The `502 "Failed to add nominee with provider"` is FP rejecting the `related_parties` create** — NOT a missing-percentage issue. Most likely causes: an invalid **`relationship`** enum value (web sends `son|daughter|spouse|father|mother|sibling|other` — may not match FP's exact codes) or a **`date_of_birth`** format issue. The backend was swallowing FP's real error behind a generic message; **fixed** — `OnboardingController.CreateNominee` now returns `"Failed to add nominee: <actual Cybrilla error>"` so the precise cause is visible. (The real reason is also emailed via `SendErrorAlertAsync("CreateNominee", …)`.)
  - **TODO (after the 502 is decoded):** if the error is the relationship enum, map our values to FP's accepted set; then add the **multi-nominee list view** (FP allows up to 3) on the nominee step + the "All Set"/KYC pages.
  - **Nominee row on the onboarding "You're All Set!" screen** is present (Account-Active card: "Nominee: Added/Not added" + Edit/Add → nominee step). If not visible, it's a stale build — rebuild the SPA.
- **2026-06-08 (Sell txns merged into Transactions page; nominee on "All Set" screen)** —
  - **Retired the separate `/app/mutual-funds/sell-transactions` page.** One-time redemptions now appear in **`/app/mutual-funds/transactions`** as `sell` rows, merged with purchases (newest-first) and rendered with the **same staged timeline** as buys (`under_review→pending→confirmed→submitted→successful`) — the timeline is driven by `state` + gateway, so sells reuse it automatically. Implementation: `loadAllTransactions()` loads `listMyPurchases()` + `listMyRedemptions()` and merges; `mapRedemption()` maps `mfr_` → the shared `Transaction` shape (`type='sell'`, `isRedemption=true`); per-row Sync and Refresh-all branch on `isRedemption` (`syncRedemption` vs `syncPurchase`). Route + the admin-layout "MF Sell Txns" nav link removed; the old `mutual-fund-sell-transactions.component` is now orphaned (no route) and can be deleted later. **App team: surface sells inside the unified transactions list, not a separate screen.**
  - **Nominee on the onboarding "You're All Set!" screen.** Added a Nominee row to the Account-Active card: shows **Added / Not added** + an **Edit/Add** button → opens the nominee step (`goToNomineeStep()`, defaults to the add form). After save, returns to `complete` if the MF account already exists, else continues to `mf-account`. (Note: editing a nominee after the MFIA exists creates the `ip_xxx` nominee but does not re-attach to the existing MFIA folio_defaults — that attach only runs at MFIA creation; deeper re-attach is a follow-up.)
- **2026-06-08 (App-team Q&A: T&C URL in config; redemption gateway re-verified; build fix)** —
  - **T&C full URL is in config.** `GET /api/config/runtime` now returns `{ isProduction, termsUrl }` where `termsUrl` ← appsettings `Runtime:TermsUrl` (dev `http://localhost:4200/app/mutual-funds/terms`; set the prod web URL in prod). **App team:** read `termsUrl` from `/api/config/runtime` and open it in a WebView — do not hardcode the route (it's environment-specific, single-sourced in appsettings). Restart FinprimApi to pick up the new key.
  - **Redemption sends NO gateway (re-verified 3× across layers).** Frontend `startSell()` builds only `{ mf_investment_account, scheme, folio_number, amount?|units? }`; backend `CreateRedemption` sets no gateway. FP **derives** ONDC from the folio; the `"gateway":"ondc"` in responses is FP echoing its derivation, not us forcing it. This is per the doc ("omit gateway, FP picks from folio"). Consequence: the by-units restriction is FP's (folio resolves to ondc) → **amount-only on ONDC; keep "By Units" disabled**. (See VERIFIED PAYLOAD TRACE in §3.18.3.)
  - **Build fix:** `mfTransactions.table.unitsLine` interpolation passed `nav` as the Angular `number` pipe result (`string | null`), which violates the `t`-pipe param type (`string | number`) → `TS2322`. Coalesced the piped value to `'—'` (`(tx.nav | number:'1.4-4') ?? '—'`). `tsc --noEmit` clean.
- **2026-06-08 (Nominee — mandatory nomination decision + KYC-page add, checklist #4)** — Per the FP SIP/purchase-plan doc, "obtaining consent from the investor for the nomination option is mandatory" (add nominee OR explicitly opt out) at new-folio setup, BEFORE the first purchase/SIP. **Placement was already correct** (onboarding `nominee` step, before `mf-account`/invest). **Fixes:** (a) replaced the silent "Skip for now" with an explicit **decision** — "Add a nominee" vs **"Opt out of nomination"** (the opt-out requires an acknowledgement checkbox; it's a recorded declaration, not a silent skip; Continue is disabled until a choice is made). Removed the dead `skipNominee()`; summary label "Skipped" → "Opted out". (b) Added an **"Add Nominee"** action on the KYC success screen → routes to `/app/mutual-funds/onboarding?step=nominee` (onboarding now honours `?step=nominee` when the investor profile exists). Scope: single nominee + opt-out (FP allows up to 3 + allocation % + minor/guardian fields — deferred). **App team: mirror the mandatory add-or-opt-out decision; do not allow a silent skip.**
- **2026-06-08 (Redemption — by-UNITS is RTA-only; ONDC create returns under_review)** — Clarifications from the FP redemption doc, prompted by app-team questions:
  - **By-units redemption is supported only on the `rta` gateway** ("units redemption only allowed for rta gateway"; `ondc` units = "coming soon"). ONDC's redemption contract is amount-based (amount→units at NAV by the RTA). **All our live folios are ONDC → keep "By Units" disabled / non-default until Cybrilla enables ONDC unit-redemption.** (Sandbox is lenient and has accepted units on ONDC, but don't rely on it for prod.) Same reason `redemption_mode=instant` and SWP plans are RTA-only.
  - **ONDC redemption create returns `under_review`, NOT `pending`** — by design: `POST → under_review → (async Cybrilla review) → pending`. This is **not a resettable account flag**; we don't control the transition (only Cybrilla does). Confirming during `under_review` correctly fails with "order is not in pending state". **Correct handling: after create, POLL the order until `pending` before PATCHing consent** (our SWP component currently jumps straight to OTP/confirm — a fix to add). For a stuck order, wait/poll or ask Cybrilla to advance it in sandbox.
- **2026-06-08 (Terms & Conditions — standalone execution-only page, checklist #8)** — Added a standalone **Mutual Fund T&C page** with the verbatim **execution-only declaration** (Islamicly + Ethically Sustainable Finnovators MFD Private Limited): EUIN intentionally NOT captured (pure digital execution-only), no advisory relationship, distributor compensation, order routing, limitation of liability, disclaimer. Web route: **`/app/mutual-funds/terms`** (component `MutualFundTermsComponent` — no data, no API, just the legal copy). **APP TEAM: open THIS SAME screen/content for your T&C link** — do NOT re-author the legal text; it must stay single-sourced. **The canonical full URL is exposed in config** — `GET /api/config/runtime` now returns `{ isProduction, termsUrl }`; `termsUrl` comes from appsettings `Runtime:TermsUrl` (e.g. `http://localhost:4200/app/mutual-funds/terms` in dev, the prod web URL in prod). Read `termsUrl` from there (open it in a WebView) instead of hardcoding the route. Acceptance of T&C (the checkbox at the consent/invest point) is the separate capture step that links to this page.
- **2026-06-08 (Redemption simulate — new endpoint, for units-based ONDC sells)** — Added a sandbox-only simulate for one-time redemptions on the sell-transactions page. **Why:** the ONDC amount-trailing-digit auto-simulation (ending 0 → successful, 1 → failed) only applies to AMOUNT-based orders; a **units-based** redemption has `amount=null` at creation, so it can't auto-resolve in sandbox and needs an explicit simulate. **New backend endpoint** `POST /api/mf/orders/redemptions/{id}/simulate {status}` → resolves the redemption's numeric `old_id` → FP `POST /api/oms/simulate/orders/{old_id} {status}` (statuses: PAYMENT_CONFIRMED, SUBMITTED, SUCCESSFUL, FAILED, REVERSED). Blocked when `Runtime:IsProduction=1` (also a 403 backstop in the controller). Service: `IMfRedemptionService.SimulateAsync(old_id, …)`; frontend `mf-purchase.service.ts simulateRedemption()`. **UPDATE (2026-06-08, later same day):** the UI "⚡ Simulate" button was **removed** from `mutual-fund-sell-transactions` per request. The **backend endpoint + service method are retained** (callable via Postman/curl for testing units-based sells); only the on-page button is gone. After simulate, sync the redemption (`GET /api/mf/orders/redemptions/{id}`) to pull the resulting state + idempotent folio decrement.
- **2026-06-08 (Sandbox simulate buttons gated on backend prod flag)** — Order-simulation is a SANDBOX-ONLY testing aid (FP: "All simulation endpoints are only available in staging server"). The SIP "Simulate installment" button in `mutual-fund-transactions` was gated only on SIP state, so it showed in production too. Now gated on the backend **`GET /api/config/runtime` → `{ isProduction }`** (sourced from appsettings `Runtime:IsProduction`), loaded into an `isProduction` signal on init (defaults to `true`/hidden until loaded). `canSimulateInstallment()` returns false when `isProduction` — both the button and the `simulateInstallment()` action are gated. **App team: any simulate UI must check `/api/config/runtime` and only show when `isProduction=false`.** (The redemption/purchase flows have no simulate button — on ONDC sandbox, order outcome is auto-simulated by the amount's trailing digit: ending 0 → successful, ending 1 → failed.)
- **2026-06-08 (Redemption consent — email/mobile stamping fix)** — Redemption/SWP confirm (`PATCH /v2/mf_redemptions`) failed with `consent: "email or mobile number should be present"` because the client sends `consent:{ otp }` only and the backend `StampConsentContactAsync` filled email/mobile **only from `tbl_Fintech_Kyc_Form`** — which is NULL on the verified-KRA path (no kyc_form). Fixed by reading the BEST-available contact from the **main tables**: new SP `Usp_Get_Fintech_User_Contact` returns email/isd/mobile with priority kyc_form → `tbl_Fintech_Investor_Email`/`_Phone` (the onboarding contact the user actually entered) → pre-verification. `IKycFormDbService.GetUserContactAsync` + both stamp methods (redemption + SWP plan) now use it. **DB: run `Usp_Get_Fintech_User_Contact`.** Backend-only; consent now always carries email/mobile.
- **2026-06-08 (Onboarding field validation + FATCA place_of_birth fix)** — Fixes for purchase-blocking missing fields + input validation (all web-side):
  - **`place_of_birth` made a required profile field (fixes purchase 400).** MF purchase failed with `investor_profile.place_of_birth is mandatory for FATCA submission to respective gateway` because the Investor Profile form **had no place-of-birth input** — it sent `place_of_birth: ''`. KYC doesn't supply it on the verified-KRA path (no kyc_form), so it must be collected. Added a required "Place of Birth" input on Step 1 (Basic Info) + `validateStep1` guard. **App team: collect `place_of_birth` on the profile form — it's mandatory for FATCA/purchase.**
  - **Mobile number validation:** onboarding Phone step now requires exactly 10 digits starting 6–9 (`/^[6-9]\d{9}$/`); input is digit-only + capped at 10 chars. Was previously only non-empty (a 13-digit number passed through).
  - **Email validation:** onboarding Email step now validates basic email format, not just non-empty.
  - **Prefill note:** phone/email prefill from `kycMobile`/`kycEmail` only exist when a kyc_form was used; on the verified-KRA (pre-verification) path Cybrilla returns no phone/email/place-of-birth, so the user must enter them — this is expected, not a bug.
- **2026-06-08 (Onboarding flow + UX corrections — verified-KRA path)** — A batch of fixes for the post-KYC onboarding journey (all web-side unless noted), tested on the verified-KRA PAN `ABCPX3751A`:
  - **Legacy KYC fully retired from the active flow.** `mutual-fund-kyc.component` no longer calls `/api/kyc/user-status` or any `/api/kyc/requests*` (legacy KYC Request API). On entry it only hits `/api/pre-verifications/*`; the legacy `getUserKycStatus` fallback + dead method were removed, and the legacy wizard steps (address/financial/documents) were dropped from the stepper. Per Cybrilla support (Manisha T): use the KYC Form API with `type=fresh`, not the KYC Request API.
  - **Auto-advance after KYC success.** On the verified-KRA path (`run_bank_validation` → compliant, no eSign/Aadhaar needed) the KYC success screen shows for ~2s with a "Proceeding to onboarding…" loader, then auto-navigates to `/onboarding`. Likewise the Investor Profile "Created!" screen shows a "Continuing setup…" loader and auto-advances after ~2s.
  - **Investor-profile prerequisite guard (fixes `Investor profile not found` 400).** Onboarding's `advanceToFirstPendingStep` now routes to `/investor-profile` FIRST when `investorProfileId` is missing — address/phone/email/bank all attach to `ip_xxx`, so the profile must exist before them. Previously it jumped to the Address step and `POST /api/onboarding/address` 400'd.
  - **Profile-form prefill on the verified path (DB).** `Usp_Get_Fintech_Investor_Profile_Status` now COALESCEs Name/DOB (and PAN) from kyc_form → pre-verification (`NameValue`/`DobValue`/`PanValue`), picking the pre-verification row that has a Name. Fixes empty Name/DOB on the profile form for the verified-KRA path (which creates no kyc_form).
  - **FATCA/CRS made mandatory + validated.** Tax-residency is required to submit KYC (doc) and legally for MF investing. In `mutual-fund-kyc` Personal Details: if the tax-residency toggle is ON, residency-1 Country + Tax ID are now mandatory (was submittable empty). In the onboarding wizard the dead `skippedFatca` flag was removed (FATCA was already non-skippable). Demat + Nominee remain optional.
  - **`vw_Fintech_User_Kyc_Status` fixed for multi-row pre-verification (DB).** A user has multiple `pv_xxx` rows (readiness row has no pan/name/dob status; pan_validation row has no readiness status). The view now AGGREGATES across all the user's rows (MAX of each verified flag) instead of picking one newest row — so a fully-verified investor correctly shows `IsKycCompliant=1`. Also added an **`IsPurchaseReady`** bit (`readiness.status=verified` only) — check THIS before placing orders, not `IsKycCompliant`.
  - **`Usp_Get_Fintech_Kyc_Form_ByUser` active-first (DB).** Resume prefers the latest non-terminal form so a blocked-create failed row doesn't shadow the real in-progress form.
  - **UI cleanups:** removed hardcoded marketing card ("Invest Halal / Avg 14% / 100% Halal / 240+ Funds"); gender narrowed to Male/Female; "Add Bank" CTA relabelled "Continue Setup" with a remaining-steps hint; KYC `type` shown read-only (auto-derived, never a user choice).
- **2026-06-08 (KYC failure/edge-case hardening — reason-aware recovery)** — Made the KYC Form failure path reason-aware so users don't dead-loop (full guidance: **§29.9.2**). All web-side, no API contract change except the resume proc:
  - **`ineligible_for_fresh_kyc`** no longer blindly retries fresh. Recovery re-reads readiness and routes: `verified`→go invest, modify-eligible→modify form, **contradiction** (readiness `unknown` while fresh-ineligible)→STOP with a "try later / contact support" message (no auto-create, no PAN-lock). A `recoveryBlocked` flag hides the recovery CTA once the dead-end is hit so the user can't re-click into the same wall.
  - **"An ongoing KYC Form already exists"** → resume the in-progress form instead of creating a new one. **DB change:** `Usp_Get_Fintech_Kyc_Form_ByUser` now returns the latest ACTIVE (non-terminal) form first (was newest-by-CreatedAt) so `my-status` surfaces the real in-progress form, not the rejected create attempt.
  - **`expired`/`failed` restart** re-derives `type` from readiness (was hardcoded `modify`) and preserves PAN/name/DOB — fixes a dead-end where an expired *fresh* KYC got refused, and a refresh()-loop back to the expired screen.
  - **`under_review` auto-poller** now starts at the end of `refresh()` (not only `ngOnInit`), so a freshly created form advances instead of freezing on the spinner.
  - **KYC `type` dropdown removed** — type is auto-derived from readiness and shown read-only ("KYC type (auto-detected)"). A user-editable type was a 7-day-PAN-lock footgun.
- **2026-06-08 (KYC "submitted ≠ ready to invest" cross-check + expired-form restart)** — Two more fixes on top of the routing changes below:
  - **`done` screen no longer falsely claims "ready to invest".** A `submitted` KYC Form does NOT mean the purchase tenant will accept orders — that's gated by pre-verification `readiness.status=verified`, a separate signal. When the form is `submitted` but readiness is still `failed`/`kyc_unavailable`, the user would previously see "Setup Complete — ready to invest" and then hit `non_kyc_compliant_investor_error` on purchase. Now `kyc-form.component.ts` computes a `purchaseReady` signal (`form submitted AND readiness verified`), shows an amber **"KYC submitted — awaiting KRA confirmation"** banner + a "Re-check readiness" button, and **disables the invest CTA** until readiness clears. Full guidance added as **§29.9.1** (mandatory for the app team to mirror). No backend/DB change.
  - **Expired/failed KYC Form restart fixed.** The "Start a fresh form" button on the `expired`/`failed` screen called a `restart()` that hardcoded `formType='modify'` and wiped PAN/name/DOB — so an expired *fresh* KYC got refused by the PAN-lock guard (dead-end), and naïvely calling `refresh()` would bounce back to the `expired` screen forever (backend `my-status` still returns the dead form; `mapNextAction` checks `status` first). `restart()` now re-reads ONLY pre-verification readiness, re-derives the correct `fresh`/`modify` type, preserves PAN/name/DOB, and lands the user on the `start` screen to create a brand-new form (Cybrilla doesn't allow resuming expired/failed forms). No backend/DB change.
- **2026-06-08 (KYC readiness routing — two gap fixes)** — Corrected the pre-verification `nextAction` mapping so it follows the Cybrilla Pre-Verification doc:
  - **`kyc_incomplete` is no longer a dead-end.** It used to map to `fix_at_kra` (an external "go fix it yourself at the KRA" CTA with no in-app path). Per the doc, `kyc_incomplete` (KRA has a record but Aadhaar isn't linked / email or phone missing) is **repairable via a `type=modify` KYC Form**, so it now maps to the new `start_modify_kyc` action which routes the investor into `/kyc-form` with `type=modify`. Backend: `PreVerificationsController.DeriveNextAction` + `DeriveNextActionFromResponse`. Frontend: `pre-verification.service.ts` (added `start_modify_kyc` to `PreVerifyNextAction`), `mutual-fund-kyc.component.ts` (new `start_modify_kyc` switch case → navigates to `/kyc-form`), `kyc-form.component.ts` (`refresh()` derives `type=modify`; `createForm()` last-mile guard now permits `modify` for `kyc_incomplete`/`start_modify_kyc`). `fix_at_kra` kept as a passive backward-compat branch for older cached rows.
  - **Already-KYC-valid investors now short-circuit to invest.** When `readiness.status=verified` and there's no in-progress KYC Form, `kyc-form.component.ts` now sets stage `already_kyc_valid` (the "Setup Complete — ready to invest" screen, previously built but never reached) instead of dragging the investor through the modify-form machinery. If a form IS mid-flow it still resumes it.
  - **No DB changes** — `nextAction` is computed at runtime from persisted readiness columns (`ReadinessStatus`/`ReadinessCode`); schema + stored procedures untouched.
  - **SWP reminder:** recurring redemption (SWP) remains **NOT supported / UI hidden** (ONDC gateway limitation). Only **one-time redemption** is live. §3.18.4 re-flagged as dormant-code reference.
- **2026-05-27 (POA Beta alignment)** — Added §0.5 (Cybrilla POA Beta capability matrix). UI changes: (a) "Start a New SWP" button removed from the SWP page header; landing-page "Withdrawals (SWP)" tile removed — Cybrilla POA Beta does not yet support recurring redemption plans. The SWP page is now branded **"Sell Units (Redemption)"** and acts as the entry point for **one-time** redemption only. (b) SIP form now exposes three frequencies: **Monthly**, **Daily — Business Days**, **Daily — Calendar Day** (matches POA Beta exactly). The installment-day picker is hidden when daily is chosen and `installment_day` is sent as `null` per FP docs. Other frequencies (weekly / quarterly / half-yearly / yearly) are not surfaced because POA Beta rejects them. Switch / STP endpoints are left in code but no UI calls them.
- **2026-05-27 (§3.18 rewrite — canonical MF transaction flows)** — Replaced the old MF Purchase / Payments sections with a single, sequence-accurate **§3.18 MF Transaction Flows** covering all five user-facing MF flows in the order the SPA actually executes them: (1) Buy lump-sum, (2) SIP / purchase plan, (3) Sell Once / one-time redemption, (4) SWP / redemption plan, (5) Transactions + Sell-Txns + Mandates pages. Each flow lists every step with the wrapper endpoint, the upstream Cybrilla URL, full request / response shapes, DB tables touched, and the webhook events that complete the lifecycle. New sub-sections: §3.18.0 readiness gate (the *"`readiness.status==verified` → skip manual KYC, `code==kyc_unavailable|unknown` → fresh KYC"* branch), §3.18.6 gateway selection table (lump-sum + SIP defer to scheme; one-time redemption omits `gateway`; SWP forced `order_gateway="rta"`; holdings report normalised to `cybrillapoa`), §3.18.7 idempotency + OTP + consent-stamping rules. Doc-aligned to the current code on `MfOrdersController.cs`, `MfRedemptionService.cs`, `ReportsController.cs`, `FinprimEventDispatcher.cs`, `MfPurchaseService` (Angular), and the new `MutualFundSellTransactionsComponent`. The legacy §3.19 Payments table is kept below §3.18 as low-level reference.
- **2026-05-22 (KYC Forms API — POA, early access)** — Cybrilla recommended switching from `/v2/kyc/requests` (KYC Request API) to `/poa/kyc_forms` (KYC Forms API) for fresh + modify flows. Implemented full backend wrapper: `KycFormService` + `KycFormsController` + `PoaFileService` + `PoaFilesController` + 8 `kyc_form.*` webhook events + `tbl_Fintech_Kyc_Form` snapshot table. All endpoints use the `cybrillarta` POA tenant (same auth as bank pre-verification). Existing `/v2/kyc/requests` left in place for legacy migration. See §11 for full flow + state machine.
- **2026-05-22 (final pass — flow consolidation for app team)** — Removed the sandbox "Simulate Success/Failure" modal from the Invest screen entirely. Lump-sum payments now ALWAYS redirect the user (same window via `window.location.href`) to FP's `token_url` — on sandbox that's the HDFC ATOM UAT page (`merchant.now.hdfc.bank.in/...ATOMCAMSINTE...`), on production it's the real bank page. Same code path both. The payment-callback page polls `/payments/{id}/snapshot` + `/purchases/{mfp}/snapshot` every 2s up to 90s and shows the **live FP state** (`under_review → pending → confirmed → submitted → successful`) as webhooks update our DB. Transactions tab timeline now reflects the real FP state machine instead of invented labels ("NAV Applied" etc.). §27.6 and §27.7 added as the definitive lump-sum and SIP flow reference for mobile.
- **2026-05-22 (PaymentPostbackController + UPI fixes)** — Three issues fixed in one pass: (1) Bank/PG's HTTP-POST callback was being sent directly to the Angular SPA URL, which can't read POST bodies — so `paymentId`/`status`/`failureCode` were lost (callback URL had only `?mfp=`). Built `PaymentPostbackController` at `POST/GET /api/pg/payments/postback` mirroring the mandate flow; backend catches form POST, logs, 303-redirects to the SPA with data as query params. Web → `/app/mutual-funds/payment-callback`; app (`?source=app`) → `/appmutualfund/payment`. (2) `payment-callback` page no longer hard-fails when `id` is missing — gracefully polls with just `mfp`. (3) UPI runtime rejects null `bank_account_id` despite docs marking optional — DTO now requires it. See §22.3 for full bridge architecture.
- **2026-05-22 (later, UPI payment)** — Lump-sum purchases now let the investor pick between **UPI** (default, recommended) and **Netbanking** before paying. New backend endpoint `POST /api/pg/payments/upi` (wraps FP `POST /api/pg/payments/upi  { method: "UPI" }`), new DTO `CreateUpiPaymentRequest`, new Angular service `createUpiPayment(req)`, new payment-method radio cards on the Invest screen (lump-sum only — SIPs are mandate-driven, no choice). The `payment-callback` / `appmutualfund/payment` pages are **method-agnostic** — they poll the same `/snapshot` endpoints regardless of which method was used. See §19.2 for full contract.
- **2026-05-22 (Cybrilla support clarification)** — Two clarifications received from Cybrilla support: (1) The `token_url` returned by `POST /v2/payments` IS the sandbox simulator itself (not a live bank URL) — open it ONLY after order state = `confirmed`; Netbanking has Success/Failure buttons; UPI accepts a dummy VPA. The older `POST /api/mf/orders/purchases/{id}/simulate` wrapper is now legacy for payment-driven orders. (2) Webhooks are the official primary path for status updates — polling is fallback only. Details in §22.7 and §22.8.
- **2026-05-22 (later)** — SIP lifecycle additions: Cancel now requires `cancellation_code` (FP runtime enforcement, not in docs); ONDC restricts cancel to `active` state only. Pause/Resume implemented via FP `skip_instructions` API (`POST /api/mf/orders/purchase-plans/{id}/pause` and `…/skip-instructions/{psi_id}/cancel`). Sandbox Simulate-Installment wrapper added (`POST /…/{id}/simulate-installment` → FP `POST /v2/mf_purchases { plan }`). Orders list SP enriched with `Plan` column + filtered index; SIP card on web now shows an "Installments" expander grouping all `mf_purchase` rows by parent SIP. Web sync-before-cancel guard added to prevent stale-state 400s. Full per-page action matrix documented in §27.5.
- **2026-05-22** — Mandate + lumpsum payment postback bridges for mobile WebView via `/appmutualfund/mandate` + `/appmutualfund/payment` standalone Angular pages, mirroring the KYC/eSign pattern. Backend `MandatePostbackController` now honours `?source=app` to pick the WebView landing page. New appsettings keys: `Finprim:Postback:{Web,App}:Payment`.
- **2026-05-21** — Pre-verification 3-stage flow (`/api/pre-verifications/*`) with resume support; bank verification persistence; token caching with semaphore + DB; NRI bank flow.
- **2026-05-20** — MF Scheme Catalog + auto-refresh `/api/masterdata/available-funds`; List + Details pages are now data-driven.
- **2026-05-19** — KYC postback bridge for mobile WebView via `/appmutualfund/kyc` + `/appmutualfund/esign` standalone Angular pages.
- **2026-05-18** — Webhook event catalog + dispatcher; `pre_verification.*` events added later.
- **2026-05-17** — Postback URL config split into Web / App by flow; backend reads full URLs verbatim from `Finprim:Postback:*`.

### Pending / blocked

- **Webhook HMAC signature verification** — code is ready but currently disabled; waiting on Cybrilla support to confirm the exact header name and signing algorithm before turning it on.

---

## 0.5 — Cybrilla POA Gateway (Beta) capability matrix

> Our active tenant runs on the **Cybrilla POA Gateway (Beta)**. The capabilities
> below are NON-NEGOTIABLE — anything marked "Not supported" will be rejected
> by Cybrilla at submission time. This section documents which features the
> app currently exposes, and which are intentionally hidden until Cybrilla
> GAs them.

### Investor & KYC

| Capability | POA Beta | What we ship | File evidence |
|---|---|---|---|
| Resident Individual (Single holding) | Supported | UI restricts holding-pattern to Single | `investor-profile.component.ts` |
| NRIs (any holding) | Supported | NRE/NRO bank flow available | `onboarding.component.ts` |
| Others (HUF / Trust / Company) | Not supported | not exposed | n/a |
| KYC status check (CVL) — charged separately | Supported | `POST /poa/pre_verifications` readiness stage | `PreVerificationsController.cs` |
| New KYC for Resident Indians only | Supported | `POST /poa/kyc_forms` (residency gate to be tightened) | `KycFormsController.cs` |
| Modify KYC | Supported | same `/poa/kyc_forms` endpoint, `form_type=modify` | `KycFormsController.cs` |

### Transactions exposed today

| Flow | POA Beta | App status | Notes |
|---|---|---|---|
| **Lump-sum purchase (single)** | Supported | **LIVE** | `mutual-fund-invest` Lump-sum tab |
| **Batch lump-sum** | Supported | Not exposed | one order per request today |
| **SIP — Monthly** | Supported | **LIVE** | `mutual-fund-invest` SIP tab, `frequency: "monthly"` |
| **SIP — Business-Day Daily** | Supported | **LIVE (added 2026-05-27)** | new dropdown option, `frequency: "day_in_a_business_day"`, `installment_day` sent null |
| **SIP — Calendar-Day Daily** | Supported | **LIVE (added 2026-05-27)** | `frequency: "day_in_a_calendar_day"` |
| SIP — any other frequency (weekly, quarterly, etc.) | Not supported | not exposed | hard-coded out of the form |
| SIP payment collection on day of registration (`generate_first_installment_now`) | Supported | not exposed | flag wired in model; UI does not toggle |
| SIP cancellation | Supported | backend endpoint exists; UI button currently hidden | `MfOrdersController.cs` `CancelPurchasePlan` |
| SIP amount modification | Supported | backend endpoint exists; UI not exposed | `PATCH /v2/mf_purchase_plans` |
| **One-time normal redemption** | Supported | **LIVE** | `mutual-fund-swp` Sell Units button → 2-step POST + PATCH consent |
| **Instant redemption (ABSL only)** | Supported (ABSL) | not exposed | `redemption_mode` field wired; UI does not surface |
| **SWP (recurring redemption plan)** | Not supported | **HIDDEN (2026-05-27)** | "Start a New SWP" button removed; landing-page tile removed; SWP component + backend endpoints retained for when Cybrilla enables it |
| **Switch (between schemes)** | Not supported | left as-is, no UI calls it | `mutual-fund-landing` "Switch" tile currently routes to Portfolio, not to a switch flow |
| **STP** | Not supported | not built | n/a |

### Payment methods

| Method | POA Beta | App | Notes |
|---|---|---|---|
| UPI | Lump-sum | **LIVE** | `POST /api/pg/payments/upi` |
| Netbanking | Lump-sum | **LIVE** | `POST /api/pg/payments/netbanking`; bank list whitelisted server-side by Cybrilla — SBI **NOT supported** |
| UPI Autopay | Lump-sum + SIPs (single, not batched) | **LIVE for SIPs** | mandate flow |
| eNACH | Lump-sum + SIPs (single, not batched) | **LIVE for SIPs** | mandate flow |
| Debit / credit card | Not supported | not exposed | n/a |

**Batch orders + mandate payments:** Cybrilla POA Beta does NOT allow batching multiple SIPs into one mandate debit. We always create one SIP per mandate, matching the constraint.

### Netbanking bank whitelist (per Cybrilla)

**Major:** HDFC, ICICI, Axis, Kotak, Bank of Baroda (Retail), Canara, Bank of India, PNB, Union, IDBI, IDFC.
**Other:** Deutsche, Indusind, Indian Overseas, Yes, HSBC, Cosmos, Standard Chartered, Karur Vysya, Punjab & Sind, South Indian, Saraswat, UCO, Bandhan, Ratnakar, AU SFB, Shivalik SFB, Jana SFB, Utkarsh, Surat, Gujarat State Coop, Sutex, Capital, SBM, Kerala Gramin, Ujjivan, City Union.
**Explicitly NOT supported:** **State Bank of India (SBI)**.

Cybrilla returns the right list at the bank-picker step; we do not duplicate the whitelist in code.

### Settlement disclosure

Per Cybrilla: *"Since all payments and mandates are issued in Cybrilla's name, investors using your app will see "Cybrilla" in their UPI authorizations, mandate details and bank statements."* — must be disclosed to the user before they invest. **TBD in our UI** (audit found no disclosure on invest / mandate setup screens).

### Partner type

- **MF distributors with ARN license** — supported. (We hold an ARN.)
- **RIA** — beta. Not used in our setup.

### Source of truth

This matrix is mirrored from the Cybrilla POA Gateway capabilities page (sandbox docs). If Cybrilla updates the matrix, update this section first, then the code accordingly.

---

## 1. Overview

The wrapper exposes a single REST surface that:
1. Authenticates the **app** to Finprim (tenant-level OAuth, server-side — clients never see Finprim creds).
2. Authenticates the **user** to the wrapper using a JWT bearer issued by Islamicly's auth endpoint.
3. Mirrors every Finprim resource ID into our own SQL Server DB (`Islamicly_Dev`) so the app can resume an interrupted onboarding, and so we can drive UI state from local DB rather than Finprim polling.
4. Subscribes to Finprim **notification webhooks** so DB state is refreshed asynchronously.

### Base URL

| Env | URL |
|-----|-----|
| Dev (local) | `http://localhost:5xxx/api/` |
| UAT | `https://fintechuat.islamicly.com/api/` |
| Prod | TBD |

In the Angular reference impl this is exposed via `environment.mfurl` and every service call is prefixed with it (e.g. `${environment.mfurl}kyc/checks`).

### Authentication

All endpoints except `[AllowAnonymous]` ones (notably `/api/webhooks/finprim-events` and `/api/masterdata/fund-schemes`) require a **JWT bearer**:

```
Authorization: Bearer <jwt>
```

The JWT is issued by Islamicly's existing user-auth endpoint (outside the scope of this doc). Issuer / audience / lifetimes from `appsettings.json`:

```json
"Jwt": {
  "Issuer": "IslamiclyAPI",
  "Audience": "IslamiclyAPI.Client",
  "AccessTokenMinutes": 15,
  "RefreshTokenDays": 7
}
```

`api/auth/token` exists but it returns the **tenant** Finprim token — clients should NOT call it.

### Required headers

| Header | Value | Required | Notes |
|--------|-------|----------|-------|
| `Authorization` | `Bearer <jwt>` | yes | All `[Authorize]` endpoints |
| `Content-Type` | `application/json` | yes | Except multipart endpoints |
| `X-Client-Platform` | `web` \| `android` \| `ios` | recommended | Used for analytics / logs |
| `Accept` | `application/json` | recommended | |

### `isWeb` query param (postback URL)

Two endpoints that produce **redirect URLs the user must visit** (DigiLocker and eSign) accept a single `isWeb` query parameter that selects which fully-qualified postback URL Finprim is told to call after the user completes the flow:

- `isWeb=1` → web caller (DEFAULT — existing web frontend works without changes).
- `isWeb=0` → mobile caller (Android / iOS WebView).

The previous `urlPath` query parameter has been **removed**. The backend now reads the **complete URL straight from `appsettings.json`** — there is no path composition, no fallback chain, and no string concatenation.

Source: `KycController.GetPostbackUrl(isWeb, flow)` — `F:\github projects\FinprimApi\Controllers\KycController.cs` lines 281–296. Call sites at the DigiLocker (`PATCH /address`) and eSign (`POST /esign`) handlers.

Endpoints that accept `isWeb`:
- `PATCH /api/kyc/requests/{id}/address?isWeb=0|1`
- `POST  /api/kyc/requests/{id}/esign?isWeb=0|1`

Concrete examples (using dev values):

| Caller | Query | Resulting postback URL |
|--------|-------|------------------------|
| Web (address)    | `?isWeb=1` (or omitted — default) | `http://localhost:4200/admin/mutual-funds/kyc` |
| Android (address)| `?isWeb=0`                        | `http://localhost:4200/appmutualfund/kyc` |
| iOS (address)    | `?isWeb=0`                        | `http://localhost:4200/appmutualfund/kyc` |
| Web (eSign)      | `?isWeb=1` (or omitted)           | `http://localhost:4200/admin/mutual-funds/kyc?esign=completed` |
| Android (eSign)  | `?isWeb=0`                        | `http://localhost:4200/appmutualfund/esign` |
| iOS (eSign)      | `?isWeb=0`                        | `http://localhost:4200/appmutualfund/esign` |

`appsettings.json` (`F:\github projects\FinprimApi\appsettings.json`):
```json
"Finprim": {
  "BaseUrl": "https://s.finprim.com",
  "TenantName": "islamiclyondc",
  "Postback": {
    "Web": {
      "DigiLocker": "http://localhost:4200/admin/mutual-funds/kyc",
      "Esign":      "http://localhost:4200/admin/mutual-funds/kyc?esign=completed"
    },
    "App": {
      "DigiLocker": "http://localhost:4200/appmutualfund/kyc",
      "Esign":      "http://localhost:4200/appmutualfund/esign"
    }
  }
}
```

In UAT, replace `http://localhost:4200` with `https://fintechuat.islamicly.com` (or whichever web host serves the Angular SPA). The mobile postback no longer points at the API — it points at **standalone Angular pages** that render success/failure inside the WebView. See section 5.1.

---

### 5.1 Mobile Postback Landing Pages (Standalone Angular)

> The mobile postback target is **no longer a backend endpoint**. Finprim is configured to redirect the WebView to two **standalone Angular routes** served by the same web SPA, which render the success/failure page entirely client-side. The backend renders no HTML for postbacks. The old `KycPostbackResultController.cs` is unused and can be removed.

**Routes** (registered in `F:\github projects\IslamiclyWebLatest\src\app\app.routes.ts`, outside the admin/layout shells — no header, no footer):

| Flow | Path | Component |
|------|------|-----------|
| DigiLocker | `/appmutualfund/kyc`     | `…\app-mutual-funds\app-kyc-result\app-kyc-result.component.ts` |
| eSign      | `/appmutualfund/esign`   | `…\app-mutual-funds\app-esign-result\app-esign-result.component.ts` |
| Mandate    | `/appmutualfund/mandate` | `…\app-mutual-funds\app-mandate-result\app-mandate-result.component.ts` |
| Payment    | `/appmutualfund/payment` | `…\app-mutual-funds\app-payment-result\app-payment-result.component.ts` |

These are the **final** paths — earlier placeholder paths such as `app/kyc/idoc-status` have been removed.

**Mandate vs Payment vs KYC/eSign — small but important difference:**

| Flow | Bank/PG target | Backend involvement | WebView lands on |
|------|---------------|---------------------|-----------------|
| KYC, eSign | Postback URL **is** the Angular page directly | None | `/appmutualfund/kyc` or `/appmutualfund/esign` |
| Mandate | Postback URL is the **backend** (`/api/pg/mandates/postback?source=app`) | Backend catches the bank's **form POST**, then 303-redirects the WebView | `/appmutualfund/mandate?id=&status=&reason=` |
| Payment | Postback URL is the Angular page directly | None | `/appmutualfund/payment?id=&mfp=&status=` |

The mandate flow needs the backend hop because BillDesk/Cybrilla performs a browser **form POST** (with a body), which an Angular route can't read. Backend reads the form, logs it, and bounces to the SPA via a 303 GET redirect. The `source=app` query param tells the backend to pick the WebView landing page (`/appmutualfund/mandate`) instead of the web one (`/app/mutual-funds/mandate-callback`).

**Geo-redirect whitelist:** Both routes sit at root level under `appmutualfund`. The country-detection guard in `F:\github projects\IslamiclyWebLatest\src\app\shell.ts` has `'appmutualfund'` added to the `NAMED_ROUTES` Set so the postback URL is **not** redirected to `/in` by the country-detection logic. If you add a sibling root-level route in the future you must whitelist it there too.

**Query params Finprim appends** (the Angular components accept both naming conventions, mirroring the previous controller):

| Flow | Param | Values | Notes |
|------|-------|--------|-------|
| DigiLocker | `id` (or `identity_document`) | `iddc_xxx` | identity document id |
| DigiLocker | `fetch.status` (or `status`) | `pending` \| `successful` \| `failed` \| `expired` | per Finprim DigiLocker docs |
| DigiLocker | `reason` | free text (e.g. `user cancelled`) | only on failure/expiry |
| eSign | `id` (or `esign`) | `esn_xxx` | esign id |
| eSign | `status` | `signed` \| `successful` \| `completed` (success) \| `failed` \| `cancelled` | success is the union of the first three |
| eSign | `reason` | free text | only on failure |
| Mandate | `id` (or `mandateId` / `paymentId`) | numeric mandate id | accepted under any of these names |
| Mandate | `status` | `success` \| `approved` \| `completed` (success) \| `failed` \| `rejected` \| `cancelled` \| `pending` \| `submitted` \| `received` | unknown values default to `pending` so the native app can resolve |
| Mandate | `reason` (or `failureReason`) | free text | only on failure |
| Payment | `id` (or `paymentId`) | numeric FP payment id | |
| Payment | `mfp` (or `mfPurchaseId`) | `mfp_xxx` | so native app can poll the order snapshot too |
| Payment | `status` | `success` \| `approved` \| `settled` \| `completed` (success) \| `failed` \| `rejected` \| `cancelled` \| `pending` \| `initiated` \| `submitted` \| `processing` | |
| Payment | `reason` (or `failureReason`) | free text | only on failure |

**What the Angular page does** (purely client-side — does **not** call the backend):

1. Shows a brief loading spinner (~350 ms) so the WebView never flashes blank.
2. Reads the Finprim query params from the current URL.
3. Renders one of three states with matching colour:
   - success (green checkmark)
   - failure (red X) — with the `reason` text shown if present
   - pending (amber hourglass)
4. Shows a **"Return to App"** button whose `href` deep-links the native app. The scheme varies by flow:
   - KYC + eSign: `islamicly://kyc-callback?flow=<digilocker|esign>&status=&id=&reason=`
   - Mandate + Payment: `islamicly://mf-callback?flow=<mandate|payment>&status=&id=&reason=` (plus `mfp=` for payment)
   Registering the `islamicly://` scheme on the native app is optional — the URL-watch interception below works without it.
5. Exposes hidden DOM data attributes on a `<div class="meta">` for the native WebView to DOM-scrape if needed:
   - `data-flow` — `digilocker` or `esign`
   - `data-status` — raw status from Finprim
   - `data-result` — normalised `success` / `failure` / `pending`
   - `data-iddoc` / `data-esign` — the resource id
   - `data-reason` — failure reason, if any

**State persistence:** Nothing is persisted by these pages. The backend stays authoritative via webhooks (`kyc_request.*`, identity-document update) and the user-status aggregator. The app should re-fetch DB state via `GET /api/kyc/user-status` after the WebView closes.

**WebView integration patterns** — pick one:

1. **URL watch (recommended).** Hook `shouldOverrideUrlLoading` / `decidePolicyFor navigationAction`. When the URL matches `…/appmutualfund/kyc`, `…/appmutualfund/esign`, `…/appmutualfund/mandate`, or `…/appmutualfund/payment`:
   - Close the WebView,
   - Call `GET /api/kyc/user-status`,
   - Then call `POST /api/kyc/requests/{id}/address/confirm` or `/esign/confirm` as appropriate.
2. **DOM parse.** Once the page renders, read `data-flow`, `data-status`, `data-result`, `data-reason` from the `<div class="meta">` element via JavaScript bridge.
3. **Deep link tap.** Let the user tap "Return to App"; intercept the `islamicly://kyc-callback?...` navigation in the WebView delegate and dispose. Requires the URI scheme to be registered.

You do NOT need: an external browser, Chrome Custom Tabs, SFSafariViewController, a custom URL scheme on Finprim's side, Universal Links / App Links, or Firebase Dynamic Links.

### Response envelope

Every endpoint returns the wrapper's standard envelope:

```json
{
  "statusCode": 200,
  "message": "Success",
  "result": { ... }
}
```

Errors:
```json
{
  "statusCode": 502,
  "message": "Failed to create KYC request. Please try again.",
  "result": null
}
```

The envelope shape is defined by `Models/Common/ApiResponse.cs` and produced by the `OkApi()` / `FailApi()` helpers used at every controller call site. There are **no** `success` or `code` fields — payload data is always under `result`, never `data`.

A few pass-through endpoints (`/api/identity-documents`, `/api/mf/orders/*`, master data) return the raw Finprim payload without the envelope. These are noted per-endpoint below.

### Naming conventions

- **Finprim-facing JSON** (request bodies you send / Finprim resource shapes inside `data`): **snake_case** — e.g. `pan`, `date_of_birth`, `mf_investment_account`.
- **Our wrapper status / aggregator responses** (e.g. `/kyc/user-status`, `/banking/my-status`, `/onboarding/status`): **camelCase** — e.g. `hasKycCheck`, `kycRequestStatus`, `investorProfileId`.

This is enforced by `[JsonPropertyName(...)]` on every model — both the Angular `KycService` and your mobile code must match exactly.

---

## 2. End-to-end sequence (ASCII)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                   ONBOARDING — first-time investor                       │
└──────────────────────────────────────────────────────────────────────────┘
 1. POST /api/kyc/checks                       (pan + dob)
        │  → action="create" | "modify" | "none" | "disallowed"
        │    status=true means KRA already compliant — skip to step 9
        ▼
 2. POST /api/kyc/requests                     (full personal block)
        │  → kyc_request id (kyr_xxx)
        ▼
 3. PATCH /api/kyc/requests/{id}/address?isWeb=0|1
        │  → identityDocId + redirectUrl  (DigiLocker)
        ▼  Open redirectUrl in a WebView. User completes Aadhaar OTP.
        │  Mobile (isWeb=0): Finprim redirects the WebView to
        │      http://<webhost>/appmutualfund/kyc?identity_document=&fetch.status=
        │    A standalone Angular page renders ✅/❌/⏳ entirely client-side.
        │    WebView either auto-closes on URL match (`/appmutualfund/kyc`) or
        │    user taps "Return to App" (islamicly://kyc-callback?...).
        │    App then calls /api/kyc/user-status and proceeds to step 4.
        │    (Web isWeb=1: postback hits the SPA route /admin/mutual-funds/kyc.)
        ▼
 4. POST /api/kyc/requests/{id}/address/confirm
        │  → KYC moves to "esign_required" state
        ▼
 5. PATCH /api/kyc/requests/{id}/financial    (occupation + income slab)
        ▼
 6. POST /api/kyc/requests/{id}/signature     (multipart — signature image)
        ▼
 7. POST /api/kyc/requests/{id}/esign?isWeb=0|1
        │  → redirectUrl (Aadhaar OTP eSign)
        ▼  Open redirectUrl in a WebView. User completes eSign OTP.
        │  Mobile (isWeb=0): Finprim redirects the WebView to
        │      http://<webhost>/appmutualfund/esign?esign=&status=
        │    Standalone Angular page renders ✅/❌/⏳ client-side.
        │    WebView either auto-closes on URL match (`/appmutualfund/esign`) or
        │    user taps "Return to App", then app calls step 8.
 8. POST /api/kyc/requests/{id}/esign/confirm
        │  → KYC status: "submitted"
        ▼  [poll GET /api/kyc/requests/{id}/status until "successful"]
 9. POST /api/investor-profiles/my-profile    (creates invp_xxx)
        ▼
10. POST /api/onboarding/address              (postal address — invp child)
    POST /api/onboarding/phone
    POST /api/onboarding/email
    POST /api/onboarding/fatca                (PEP + optional foreign tax)
    POST /api/onboarding/nominee              (optional)
    POST /api/onboarding/demat                (optional)
        ▼
11. POST /api/banking/my-account              (creates bac_xxx)
        ▼
12. POST /api/banking/my-verify               (start penny-drop)
        │  → verificationId (prv_xxx)
        ▼  [poll GET /api/banking/my-verify/{id} every 4s]
13. POST /api/onboarding/mf-account           (creates mfia_xxx;
        │                                      auto-patches folio_defaults)
        ▼
14. POST /api/banking/set-primary/{bacId}     (sets payout bank on MFIA)
        ▼
15. GET  /api/masterdata/fund-schemes/by-isin/{isin}
        ▼
16. POST /api/mf/orders/purchases             (amount, scheme, mfia)
        ▼
17. POST /api/pg/payments/netbanking          (returns redirect URL)
        ▼  [user pays; payment.success webhook fires]
    DONE — units allotted, webhooks update DB.
```

---

## 3. Endpoint Reference

> Path prefix `/api/` is implicit on every endpoint below.

> ## ⛔ DEPRECATED — §3.1–§3.2 (`/kyc/checks` + `/kyc/user-status`) are the OLD KYC flow
> The legacy KYC-check / KYC-request wizard documented in **§3.1, §3.2, and §3.3–§3.11** is **superseded**. Do **NOT** build new clients against it.
> **Current KYC flow = `POST /api/pre-verifications/readiness` → `POST /api/kyc/forms` (POA fill-fields → eSign).** See **§14 / §14.2 (Fresh KYC)** and the new **§14.3 (KYC — consolidated latest)** for the authoritative contract, including OTP-verified email/mobile, PAN/DOB auto-populate, the duplicate-PAN guard, and the resume rules.
> §3.1–§3.11 are kept below only as a historical reference for the few users still on old `kyc_request` rows; everything new must use readiness + `kyc/forms`.

### 3.1 KYC Check (`POST /kyc/checks`)  — ⛔ DEPRECATED (use `/pre-verifications/readiness`, §14.2)

**Purpose:** Pre-flight check — looks up the PAN with KRA and tells you whether to **create**, **modify**, or **skip** the KYC request flow.

**Headers:** `Authorization`, `Content-Type: application/json`

**Request body:**
```json
{
  "pan": "ABCDE1234F",
  "date_of_birth": "1990-04-15"
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `pan` | string(10) | yes | Uppercased server-side; format `^[A-Z]{5}[0-9]{4}[A-Z]$` |
| `date_of_birth` | string `YYYY-MM-DD` | recommended | Used as a verification anchor against KRA |

**Success (200) — wrapper envelope's `data`:**

```json
{
  "id": "kyc_abc123",
  "source_ref_id": null,
  "pan": "ABCDE1234F",
  "entity_details": {
    "name": "ARJUN MEHTA",
    "gender": "male",
    "date_of_birth": "1990-04-15",
    "father_name": "RAJ MEHTA",
    "marital_status": "unmarried",
    "nationality": "in",
    "residential_status": "resident_individual",
    "correspondence_address": { "line_1": "...", "city": "...", "pincode": "...", "state": "...", "country": "..." },
    "permanent_address": { "...": "..." },
    "email": "x@y.com",
    "mobile": "9876543210"
  },
  "status": true,
  "action": "create",
  "reason": "incomplete",
  "constraints": [
    { "type": "investment_limit", "amount": { "value": 50000, "currency": "INR" } }
  ],
  "sources": [ { "name": "CVL", "fetched_at": "2026-05-18T10:00:00Z" } ],
  "created_at": "2026-05-18T10:00:00Z",
  "updated_at": "2026-05-18T10:00:00Z",
  "local_id": 42
}
```

**Field meanings (critical):**

| Field | Type | Notes |
|-------|------|-------|
| `status` | boolean | **`true` = PAN is fully KRA-compliant; skip to investor-profile creation.** |
| `action` | enum | `create` \| `modify` \| `disallowed` \| `none` \| `unknown` — drives next step |
| `reason` | enum | `unavailable` \| `onhold` \| `rejected` \| `deactivated` \| `legacy` \| `underprocess` \| `incomplete` \| `unknown` |
| `constraints[]` | list | Optional investment caps (e.g. ₹50k limit until full KYC) |
| `local_id` | int | Our DB primary key for this row |

**Action → next step:**

| `action` | What the app should do |
|----------|------------------------|
| `create` | Call `POST /kyc/requests` with full personal block |
| `modify` | Call `POST /kyc/requests` then `PATCH` only the modified fields |
| `none` | Already compliant — skip to investor profile (`status` should be `true`) |
| `disallowed` | Show error — PAN cannot proceed (KRA rejected / deactivated) |
| `unknown` | Treat as `create` (this is also returned synthetically if Finprim's `kyc_checks` is not enabled for the tenant — see KycController.cs:85–96) |

**Errors:**

| Code | Cause | Recovery |
|------|-------|----------|
| 400 | Bad PAN format | Re-prompt user |
| 401 | Missing/expired JWT | Refresh token |
| 502 | Finprim returned empty response | Retry after backoff |

**Persisted to:** `tbl_Fintech_KYC_Check` (one row per user × PAN).

---

### 3.2 Get user KYC status (`GET /kyc/user-status`)  — ⛔ DEPRECATED (use `/kyc/forms/my-status` + `/pre-verifications/my-status`, §14.3)

> Legacy. For resume, use **`GET /api/pre-verifications/my-status`** (PAN/readiness stage) and **`GET /api/kyc/forms/my-status`** (KYC-form stage, PAN-match guard required — §14.2.3 item 9), plus **`GET /api/onboarding/status`** for the post-KYC steps. See **§14.3** and the new **§14.4 (Resume)**.

Returns the user's latest KYC check + KYC request joined from our DB. **Call this first whenever the user opens the app** — it tells you exactly where to resume.

Source: `KycController.GetUserKycStatus`, model `KycUserStatusResponse`.

Response (data):

```json
{
  "hasKycCheck": true,
  "kycCheckLocalId": 42,
  "kycId": "kyc_abc123",
  "pan": "ABCDE1234F",
  "name": "ARJUN MEHTA",
  "kycStatus": false,
  "action": "create",
  "reason": "incomplete",
  "constraintType": "investment_limit",
  "constraintAmountValue": 50000,
  "constraintCurrency": "INR",
  "kycCheckedAt": "2026-05-18T10:00:00Z",

  "hasKycRequest": true,
  "kycRequestId": "kyr_xyz789",
  "kycRequestStatus": "pending",
  "kycRequestVerificationStatus": null,
  "kycRequestWizardStep": "address",
  "kycRequestCreatedAt": "...",
  "kycRequestSubmittedAt": null,
  "kycRequestSuccessfulAt": null,

  "isKYCCompliant": false
}
```

`kycRequestWizardStep` — **resume hint** set by the backend each time the user advances. Possible values mirror the Angular wizard: `address` \| `financial` \| `documents` \| `esign` \| `submitted` \| `successful` \| `rejected` \| `expired` \| `bank`.

Server auto-refreshes from Finprim if `hasKycCheck=true`, `kycStatus≠true`, `kycRequestStatus≠"successful"` (KycController.cs:111–141).

---

### 3.3 Create KYC Request (`POST /kyc/requests`)

**Purpose:** Open a new KYC application with Finprim. Triggered when `action=create` or `modify`.

**Query params (optional):**
- `wizardStep=<step>` — DB-only resume hint (e.g. `address`)
- `aadhaarLast4=<4 digits>` — stored locally only, NOT forwarded to Finprim (Finprim needs the full 12-digit Aadhaar which is fetched via DigiLocker)

**Request body** (see `CreateKycRequestRequest` in `Models/Kyc/KycModels.cs`):

```json
{
  "isWeb": 1,
  "name": "Arjun Mehta",
  "pan": "ABCDE1234F",
  "date_of_birth": "1990-04-15",
  "email": "arjun@example.com",
  "mobile": { "isd": "+91", "number": "9876543210" },
  "father_name": "Raj Mehta",
  "mother_name": "Sunita Mehta",
  "spouse_name": null,
  "gender": "male",
  "marital_status": "unmarried",
  "residential_status": "resident_individual",
  "occupation_type": "private_sector",
  "citizenship_countries": ["in"],
  "nationality_country": "in",
  "country_of_birth": "in",
  "place_of_birth": "Mumbai",
  "income_slab": "above_5lakh_upto_10lakh",
  "pep_details": "not_applicable",
  "tax_residency_other_than_india": false,
  "non_indian_tax_residency_1": null,
  "non_indian_tax_residency_2": null,
  "non_indian_tax_residency_3": null,
  "signature": null,
  "geolocation": null,
  "identity_proof": null,
  "address": null,
  "aadhaar_number": null
}
```

**Field reference:**

| Field | Type | Required | Allowed values / notes |
|-------|------|----------|----------------------|
| `name` | string | yes | Full legal name as on PAN |
| `pan` | string(10) | yes | uppercase |
| `date_of_birth` | `YYYY-MM-DD` | yes | |
| `email` | string | yes | |
| `mobile.isd` | string | yes | Default `+91` |
| `mobile.number` | string | yes | 10-digit |
| `father_name` | string | recommended | |
| `mother_name` | string | optional | |
| `spouse_name` | string | optional | |
| `gender` | enum | optional | `male` \| `female` \| `others` |
| `marital_status` | enum | optional | `unmarried` \| `married` \| `divorced` \| `widowed` |
| `residential_status` | enum | yes | `resident_individual` \| `non_resident_individual` (default `resident_individual`) |
| `occupation_type` | enum | optional | See enum appendix |
| `citizenship_countries` | string[] | yes | ISO-2 lowercase, default `["in"]` |
| `nationality_country` | string | yes | default `in` |
| `country_of_birth` | string | yes | default `in` |
| `place_of_birth` | string | yes | free text, default `in` |
| `income_slab` | enum | optional | See enum appendix |
| `pep_details` | enum | yes | **Investor-profile enum:** `not_applicable` \| `pep_exposed` \| `pep_related` (NOT the KYC-form enum — see "pep_details — TWO DIFFERENT enums" in the appendix) |
| `tax_residency_other_than_india` | bool | yes | If `true`, populate `non_indian_tax_residency_1` |
| `non_indian_tax_residency_N` | `{country, taxid_number}` | conditional | up to 3 |
| `aadhaar_number` | string | optional | NOT sent to Finprim — see note above |
| `isWeb` | int | optional | 0=app, 1=web — stored locally only |

**Success:** Returns `KycRequestResponse`:

```json
{
  "id": "kyr_xyz789",
  "status": "pending",
  "verification_status": null,
  "pan": "ABCDE1234F",
  "name": "ARJUN MEHTA",
  "fields_needed": ["identity_proof", "address.proof", "signature"],
  "expires_at": "2026-06-17T...",
  "submitted_at": null,
  "successful_at": null,
  "rejected_at": null,
  "created_at": "...",
  "updated_at": "...",
  "identity_proof": null,
  "address": null,
  "geolocation": null,
  "father_name": "Raj Mehta",
  "gender": "male",
  "date_of_birth": "1990-04-15",
  "email": "arjun@example.com",
  "mobile": { "isd": "+91", "number": "9876543210" },
  "aadhaar_number": null,
  "local_id": 17
}
```

**`status`** ∈ `pending` \| `submitted` \| `successful` \| `rejected` \| `expired`.

**Errors:** 400 (validation), 401, 502 (`"Failed to create KYC request"`).

**Next:** PATCH `/kyc/requests/{id}/address`.

---

### 3.4 Update KYC Request (`PATCH /kyc/requests/{id}`)

Patches arbitrary fields on an existing KYC request. Same shape as `CreateKycRequestRequest` but every field is nullable (`UpdateKycRequestRequest`). Used to:
- Save additional fields after creation
- Attach `identity_proof` document id
- Attach `signature` file id (the `/signature` endpoint does this internally)

Query params: `wizardStep` (same semantics as create).

Response: full `KycRequestResponse`.

---

### 3.5 Financial details (`PATCH /kyc/requests/{id}/financial`)

Convenience wrapper that sets `occupation_type`, `income_slab`, `pep_details`, and `tax_residency_other_than_india`, then advances `wizardStep="documents"`.

**Request:**
```json
{
  "occupation_type": "private_sector",
  "income_slab": "above_5lakh_upto_10lakh",
  "pep_details": "not_applicable",
  "tax_residency_other_than_india": false
}
```

| Field | Type | Required | Default if omitted |
|-------|------|----------|--------------------|
| `occupation_type` | enum | optional | — (not PATCHed) |
| `income_slab` | enum | optional | — |
| `pep_details` | enum | optional | `not_applicable` |
| `tax_residency_other_than_india` | bool | optional | `false` |

`UpdateFinancialRequest` DTO: `F:\github projects\FinprimApi\Models\Kyc\KycModels.cs:265-281`. Backend auto-defaults `pep_details` and `tax_residency_other_than_india` (see `KycController.UpdateFinancial` at lines 258–268), so the wizard can safely skip prompting for them and still leave the KYC request in a state where eSign can proceed. See enum appendix.

---

### 3.6 Address / DigiLocker (`PATCH /kyc/requests/{id}/address?isWeb=0|1`)

**Purpose:** Step 1 of the address proof flow — captures geolocation (optional) and **creates a DigiLocker identity document**. Returns a `redirectUrl` the user must visit to authenticate with Aadhaar.

**Query:**
- `isWeb` — `0` (mobile) or `1` (web). **Default: `1`** (so existing web frontend works without changes). The backend reads the full postback URL from `Finprim:Postback:Web:DigiLocker` or `Finprim:Postback:App:DigiLocker` accordingly — no path composition. The `urlPath` query parameter that previous versions accepted has been **removed**.

**Request body:**
```json
{ "latitude": 19.0760, "longitude": 72.8777 }
```

| Field | Type | Required |
|-------|------|----------|
| `latitude` | double | optional |
| `longitude` | double | optional |

**Success (200):**
```json
{
  "identityDocId": "iddc_aaa111",
  "redirectUrl": "https://digilocker.gov.in/...partner=cybrilla&txn=..."
}
```

Server-side (KycController.cs:297–365):
- If `latitude`+`longitude` present → first PATCHes the geolocation onto the KYC request.
- If an identity doc already exists for this user × KYC request, **returns the existing one** (avoids Finprim's `already_exist` rejection).
- Otherwise calls `POST /v2/identity_documents` on Finprim with `type="aadhaar"`, `kyc_request=<id>`, `postback_url=<computed>`.
- Persists to `tbl_Fintech_Identity_Document` and stamps `IdentityDocumentId` on the KYC request row.

**Behavior across platforms:**

- **Web (`isWeb=1`):** open `redirectUrl` in same tab or popup; Finprim posts back to the URL stored at `Finprim:Postback:Web:DigiLocker` (dev: `http://localhost:4200/admin/mutual-funds/kyc`) after the user completes Aadhaar OTP.
- **Mobile (`isWeb=0`):** open `redirectUrl` **in a WebView** (not an external browser). Finprim posts back to `Finprim:Postback:App:DigiLocker` (dev: `http://localhost:4200/appmutualfund/kyc?identity_document=&fetch.status=`). A **standalone Angular page** renders the success/failure UI client-side. The app watches for that URL prefix to auto-close, or lets the user tap "Return to App". See section 5.1 for details.

**Errors:**
- 502 + `"Failed to create identity document"` — Finprim/DigiLocker unavailable; retry.
- If DigiLocker rejects on the user side, the document's `fetch.status` will be non-`successful` and step 3.7 will return 400.

**Next:** `POST /kyc/requests/{id}/address/confirm`.

---

### 3.7 Confirm Address (`POST /kyc/requests/{id}/address/confirm`)

Called **after** the user finishes DigiLocker. On mobile, your app calls this once the WebView reaches `/appmutualfund/kyc` (URL watch) or the user taps "Return to App". On web, after the SPA receives the postback at `/admin/mutual-funds/kyc`.

**Request:**
```json
{ "identity_doc_id": "iddc_aaa111" }
```

**What it does (KycController.cs:367–410):**
1. GETs the identity document from Finprim.
2. If `fetch.status !== "successful"` → returns 400 (`"Aadhaar verification not completed"`).
3. PATCHes the KYC request with `identity_proof=<iddoc>` and `address={proof, proof_type:"aadhaar"}`.
4. Sets `wizardStep="financial"` locally.

**Success:**
```json
{
  "kycRequest": { /* full KycRequestResponse — status likely "esign_required" */ }
}
```

**Errors:**
- 400 — DigiLocker not completed; show retry-DigiLocker UI.
- 502 — patch failed; retry.

---

### 3.8 Upload Signature (`POST /kyc/requests/{id}/signature`)

`multipart/form-data` with a single `file` field.

| Field | Type | Notes |
|-------|------|-------|
| `file` | file | PNG/JPEG, signature image |

Server uploads the file to Finprim Files (`type=signature`), then PATCHes the KYC request with three fields **together** (`KycController.UploadSignature` lines 464–471):

- `signature` = `<file_id>`
- `pep_details` = `"not_applicable"` (auto-defaulted)
- `tax_residency_other_than_india` = `false` (auto-defaulted)

This auto-heal ensures Finprim transitions the KYC request to `esign_required` even if the wizard skipped the financial step, so the eSign call that follows no longer fails with `"must be in esign_required state"`. Advances `wizardStep="esign"`.

**Response:**
```json
{ "fileId": "file_sig123", "kycStatus": "esign_required" }
```

Errors: 400 (empty file), 502.

---

### 3.9 Create eSign (`POST /kyc/requests/{id}/esign?isWeb=0|1`)

**Purpose:** Generate an Aadhaar OTP eSign session. Returns a `redirectUrl` the user must visit to OTP-sign the KYC application form.

**Query:** same `isWeb` semantics as `/address` — `0` mobile, `1` web (default `1`). The backend reads the full postback URL from `Finprim:Postback:Web:Esign` or `Finprim:Postback:App:Esign`. The `urlPath` query parameter has been **removed**.

**No request body** (the wrapper builds the eSign creation payload internally).

**Server logic (`KycController.cs` create-eSign handler; `GetPostbackUrl` at lines 281–296):**
- Resolves `esignPostback` by reading the appropriate config key — no string composition, no fallback chain.
- If an existing eSign exists for this user × KYC, returns it (idempotent).
- Calls Finprim `POST /v2/esigns` with `kyc_request` and `postback_url`.
- Persists to `tbl_Fintech_ESign`.

**Success:**
```json
{ "esignId": "esn_p999", "redirectUrl": "https://...nsdl/esign/..." }
```

**Errors:**
- 400 + `"KYC request is not ready for eSign. Please make sure ALL these steps are complete: personal details, financial details (occupation + income), Aadhaar verification, and signature upload. Refresh and try again."` — KYC is not yet in `esign_required` state. App should call `/kyc/user-status` to identify the missing step (personal / financial / DigiLocker / signature) and route the user back to it.
- 502 — Finprim/NSDL down; retry.

---

### 3.10 Confirm eSign (`POST /kyc/requests/{id}/esign/confirm`)

Called when the user returns from the eSign redirect.

**Request:**
```json
{ "esign_id": "esn_p999" }
```

Marks eSign as `completed` in our DB, re-fetches KYC from Finprim, advances `wizardStep="submitted"`. The KYC status now transitions through `submitted → successful` (asynchronous on KRA's side — listen for `kyc_request.successful` webhook or poll `GET /kyc/requests/{id}/status`).

**Response:**
```json
{ "kycRequest": { "id": "kyr_xyz789", "status": "submitted", ... } }
```

---

### 3.11 Poll KYC status (`GET /kyc/requests/{id}/status`)

```json
{ "status": "submitted" | "successful" | "rejected" | "expired" }
```

When a terminal status (`successful`/`rejected`/`expired`) is observed, the server upserts it into our DB.

The frontend polls this every few seconds after eSign confirmation; mobile apps can either poll or rely on the `kyc_request.successful` webhook to push state.

---

### 3.12 Investor Profile

#### `GET /investor-profiles/my-status`

Returns the user's `tbl_Fintech_Investor_KYC_Profile` row + KYC-derived pre-fill fields (`kycName`, `kycPan`, `kycGender`, `kycDateOfBirth`, `kycOccupationType`, `kycIncomeSlab`, `kycSignatureFileId`, etc.). Use these to pre-fill the create form.

Shape: `InvestorProfileStatusResponse` (Models/InvestorProfiles).

#### `POST /investor-profiles/my-profile`

Creates the canonical investor profile (`invp_xxx`) once KYC is `successful`. This is the parent record that owns bank accounts, addresses, demat, MFIA, etc.

**Request body (CreateInvestorProfileRequest):**

```json
{
  "isWeb": 1,
  "type": "individual",
  "tax_status": "resident_individual",
  "name": "Arjun Mehta",
  "date_of_birth": "1990-04-15",
  "gender": "male",
  "occupation": "private_sector_service",
  "pan": "ABCDE1234F",
  "country_of_birth": "IN",
  "place_of_birth": "Mumbai",
  "nationality_country": "IN",
  "source_of_wealth": "salary",
  "income_slab": "upto_1lakh",
  "pep_details": "not_applicable",
  "signature": "file_sig123",
  "guardian_name": null,
  "guardian_date_of_birth": null,
  "guardian_pan": null,
  "first_tax_residency": { "country": "IN", "taxid_type": "pan", "taxid_number": "ABCDE1234F" },
  "second_tax_residency": null,
  "third_tax_residency": null,
  "fourth_tax_residency": null,
  "employer": null,
  "ip_address": null
}
```

Notes:
- `ip_address` is auto-set server-side from `HttpContext.Connection.RemoteIpAddress` — do not send.
- `first_tax_residency` is auto-defaulted to India/PAN if omitted.
- `occupation` enum is the **investor-profile occupation list** (different from KYC `occupation_type`): `agriculture` \| `business` \| `doctor` \| `forex_dealer` \| `government_service` \| `house_wife` \| `others` \| `private_sector_service` \| `professional` \| `public_sector_service` \| `retired` \| `service` \| `student` (source: `InvestorProfileModels.cs:44` comment).
- `source_of_wealth`: `salary` (default), `business`, `inheritance`, `gift`, `savings`, `other`.

**Response:** the raw Finprim profile object including `id` (which is `invp_xxx`).

#### `POST /investor-profiles/my-profile/signature`

`multipart/form-data` with `file`. Uploads a fresh signature if the KYC one is not reusable. Returns `{ fileId }`.

---

### 3.13 Onboarding child resources

All under `/api/onboarding/*` (authenticated). Each auto-populates `profile`/`investor_profile` from the logged-in user's `invp_xxx` if not supplied — frontend can leave it blank.

#### `POST /onboarding/address`

```json
{
  "profile": "invp_xxx",
  "line1": "12 MG Road",
  "line2": "Andheri East",
  "line3": null,
  "postal_code": "400069",
  "city": "Mumbai",
  "state": "MH",
  "country": "IN",
  "nature": "residential"
}
```

| Field | Allowed |
|-------|---------|
| `nature` | `residential` \| `correspondence` |
| `country` | ISO-2 uppercase, default `IN` |

#### `POST /onboarding/phone`
```json
{ "profile": "invp_xxx", "isd": "91", "number": "9876543210" }
```
Note: `isd` here is bare (no `+`); KYC uses `+91`.

#### `POST /onboarding/email`
```json
{ "profile": "invp_xxx", "email": "arjun@example.com" }
```

#### `POST /onboarding/demat` *(optional)*
```json
{ "profile": "invp_xxx", "dp_id": "IN300394", "client_id": "12345678" }
```

#### `POST /onboarding/fatca`
PATCHes `first_tax_residency` (India/PAN) and optionally `second_tax_residency` on the investor profile.

```json
{
  "investor_profile": "invp_xxx",
  "pep_details": "not_applicable",
  "has_non_indian_tax_residency": true,
  "non_indian_country": "US",
  "non_indian_taxid_type": "tin",
  "non_indian_taxid_number": "123-45-6789",
  "non_indian_applicable_from": "2020-01-01",
  "non_indian_applicable_to": "2099-12-31"
}
```

`non_indian_taxid_type`: `tin` \| `ssn` \| `pan` \| `other`.

#### `POST /onboarding/nominee` *(optional — full contract in §14.5)*
```json
{
  "profile": "invp_xxx",
  "name": "Sunita Mehta",
  "date_of_birth": "1992-08-12",
  "relationship": "spouse",
  "allocationPercentage": 100,
  "nomineePan": "ABCDE1234F",
  "guardianName": null, "guardianRelationship": null, "guardianPan": null,
  "guardianEmail": null, "guardianIsd": null, "guardianMobile": null,
  "guardianAddressLine1": null, "guardianAddressCity": null,
  "guardianAddressState": null, "guardianAddressPostalCode": null
}
```
`relationship`: `son` \| `daughter` \| `spouse` \| `father` \| `mother` \| `sibling` \| `other`.
- **Minor nominee** (DOB makes them <18): `guardian*` fields become **required** (name, relationship, PAN, email, mobile, address line1+postal). The nominee's own `nomineePan` is **nulled** for a minor (PAN is adult-only per FP).
- `allocationPercentage` is the **MFIA folio_defaults** share, persisted locally and applied at MFIA attach — it is NOT sent to FP `/v2/related_parties`. The selected nominees' allocations must total **100%**.
- See **§14.5** for the screen, multi-nominee selection, and resume behavior.

#### `GET /onboarding/status`

Returns `OnboardingStatusResponse` — flags + IDs for every child resource. Use this to drive the wizard's "resume at first pending step" behavior (reference impl: `OnboardingComponent.advanceToFirstPendingStep`):

```json
{
  "investorProfileId": "invp_xxx",
  "addressId": "addr_yyy",
  "phoneId": "phon_zzz",
  "emailId": "ema_aaa",
  "bankAccountId": "bac_bbb",
  "dematId": null,
  "fatcaId": "invp_xxx",
  "mfAccountId": "mfia_ccc",
  "hasAddress": true, "hasPhone": true, "hasEmail": true,
  "hasBankAccount": true, "hasDemat": false, "hasFatca": true, "hasMfAccount": true,
  "nomineeId": null, "hasNominee": false,
  "investorName": "Arjun Mehta", "investorPan": "ABCDE1234F",
  "kycEmail": "...", "kycMobile": "...", "kycMobileIsd": "+91",
  "kycOccupation": "private_sector", "kycIncomeSlab": "above_5lakh_upto_10lakh"
}
```

---

### 3.14 Bank Account

#### `GET /banking/my-status`

`BankAccountStatusResponse`:
```json
{
  "hasBankAccount": true,
  "bankAccountId": "bac_xxx",
  "accountNumber": "1234567890",
  "bankName": "HDFC Bank",
  "ifscCode": "HDFC0001234",
  "holderName": "ARJUN MEHTA",
  "type": "savings",
  "branchCity": "Mumbai",
  "createdAt": "...",
  "investorName": "Arjun Mehta",
  "investorProfileId": "invp_xxx"
}
```

#### `POST /banking/my-account`

```json
{
  "isWeb": 1,
  "profile": "invp_xxx",
  "primary_account_holder_name": "ARJUN MEHTA",
  "account_number": "1234567890",
  "type": "savings",
  "ifsc_code": "HDFC0001234",
  "cancelled_cheque": null,
  "sole_proprietorship": null
}
```

| Field | Required | Notes |
|-------|----------|-------|
| `profile` | optional | Auto-filled from current user's invp_xxx if omitted |
| `type` | yes | `savings` \| `current` \| `nre` \| `nro` (default `savings`) |
| `ifsc_code` | yes | uppercase |
| `cancelled_cheque` | optional | Finprim `file_xxx` id of a cheque image |
| `sole_proprietorship` | optional | only for sole-prop accounts |

**Response:** `BankAccountResponse` (id, profile, primary_account_holder_name, account_number, type, ifsc_code, bank_name, branch_name, branch_city, branch_state, created_at).

#### `POST /banking/set-primary/{bankAccountId}`

Sets this bank as the **payout bank** on the user's MFIA (Finprim has no "primary" flag — the MFIA's `folio_defaults.payout_bank_account` is the source of truth). PATCHes that field server-side.

Response:
```json
{ "mfAccountId": "mfia_ccc", "primaryBankAccountId": "bac_bbb" }
```

**Errors:** 400 if user has no MFIA yet.

---

### 3.15 Bank Pre-Verification (penny-drop)

The wrapper does **not** use Finprim's `/v2/bank_account_verifications` (not supported on ONDC gateway). Instead it uses Cybrilla POA `/poa/pre_verifications`. Source: `BankingController.StartMyVerification`.

#### `POST /banking/my-verify`

No body. Server pulls account_number / ifsc / holder / PAN from `tbl_Fintech_Bank_Account` + investor profile and submits:

```json
{
  "investor_identifier": "ABCDE1234F",
  "pan":  { "value": "ABCDE1234F" },
  "name": { "value": "ARJUN MEHTA" },
  "bank_accounts": [
    {
      "value": {
        "account_number": "1234567890",
        "ifsc_code": "HDFC0001234",
        "account_type": "savings"
      },
      "verify_manually_if_required": true
    }
  ]
}
```

Response:
```json
{
  "verificationId": "prv_qqq",
  "status": "pending",
  "code": "",
  "message": ""
}
```

#### `GET /banking/my-verify/{verificationId}` *(poll every 4s)*

Wrapper maps Cybrilla's `bank_accounts[0].status` + `readiness` to the shape:

```json
{
  "verificationId": "prv_qqq",
  "status": "pending" | "completed" | "failed",
  "confidence": "very_high" | "high" | "uncertain" | "low" | "very_low" | "zero",
  "reason": "expiry" | "digital_verification_failure" | "<code>",
  "code": "",
  "message": ""
}
```

**Mapping logic (BankingController.cs:290–340):**
- `bank_accounts[0].status` ∈ `completed|success|verified` → `status=completed`, confidence default `very_high`.
- `bank_accounts[0].status` ∈ `failed|failure` → `status=failed`, confidence default `zero`, reason = `bank_accounts[0].reason` or fallback `digital_verification_failure`.
- `readiness.status=failed` with no bank status → `status=failed` (Cybrilla didn't even attempt penny-drop — gateway/tenant issue).
- otherwise → `status=pending`.

**`readiness.status=failed, code=unknown`** means Cybrilla's gateway could not initiate the verification (typically tenant misconfig or upstream NPCI outage). The reference impl proceeds optimistically (`confidence=not_checked`) — see `OnboardingComponent.pollVerificationStatus` error branch.

#### `GET /pre-verifications/{id}` *(pass-through with derived fields)*

Same data with extra:
```json
{
  "derivedStatus": "completed" | "failed" | "pending",
  "readinessStatus": "...",
  "readinessCode": "...",
  "readinessReason": "...",
  "bankStatus": "...",
  "bankCode": "...",
  "bankReason": "...",
  "raw": { ... }
}
```

#### Polling pattern (frontend reference)

```ts
interval(4000).pipe(
  switchMap(() => svc.getBankVerificationStatus(id)),
  takeWhile(r => r.status === 'pending' && pollCount < 60, true)
)
```
60 polls × 4 s = 4-minute timeout, after which the app shows a `Retry` button (treats it as `reason=expiry`).

---

### 3.16 MF Investment Account (`POST /onboarding/mf-account`)

**Purpose:** Final aggregator resource. Combines investor profile + addresses + phones + emails + bank + (optional demat + nominees) into a single `mfia_xxx` that purchases reference.

**Request:**
```json
{
  "isWeb": 1,
  "primary_investor": "invp_xxx",
  "holding_pattern": "single",
  "folio_defaults": null
}
```

| Field | Allowed |
|-------|---------|
| `holding_pattern` | `single` \| `anyone_or_survivor` \| `joint` (frontend currently only exposes `single`) |
| `folio_defaults` | object — see below |

**After create, server auto-PATCHes `folio_defaults`** with whatever the user has saved (OnboardingController.cs:443–462):
```json
{
  "folio_defaults": {
    "communication_email_address": "ema_aaa",
    "communication_mobile_number":  "phon_zzz",
    "communication_address":        "addr_yyy",
    "payout_bank_account":          "bac_bbb"
  }
}
```

`FolioDefaultsInput` additionally supports: `overseas_communication_address`, `nominee1`/`nominee2`/`nominee3` (+ `_allocation_percentage` + `_identity_proof_type` + `_guardian_identity_proof_type`), `demat_account`, `nominations_info_visibility`.

**Response:** raw Finprim MFIA object with `id=mfia_xxx`.

---

### 3.17 Master data (MF schemes etc.)

`/api/masterdata/*` — most are anonymous and pass-through.

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/masterdata/pincodes/{pincode}` | Resolve pincode → city/state |
| GET | `/masterdata/states` | List of states |
| GET | `/masterdata/countries` | ISO countries |
| GET | `/masterdata/ifsc/{ifsc}` | Resolve IFSC → bank + branch |
| GET | `/masterdata/amcs` | List of all AMCs |
| GET | `/masterdata/fund-schemes` (anonymous) | Paginated scheme catalogue. Query: `page`, `size`, `amc_id`, `investment_option`, `plan_type`, `delivery_mode` |
| GET | `/masterdata/fund-schemes/{gateway}` | Schemes for a gateway |
| GET | `/masterdata/fund-schemes-v2/{gateway}/{isin}` | Full v2 scheme by ISIN |
| GET | `/masterdata/fund-schemes/by-isin/{isin}` | Single scheme by ISIN |
| GET | `/masterdata/fund-schemes/all` | Full unpaginated dump (use sparingly) |

`investment_option`: `growth` \| `dividend_payout` \| `dividend_reinvest` \| `idcw_payout` \| `idcw_reinvest`.
`plan_type`: `direct` \| `regular`.
`delivery_mode`: `demat` \| `physical`.

**Recommended sandbox ISIN:** `INF084M01044` (HDFC Liquid Direct Growth — used by the Cybrilla sandbox for full end-to-end test purchases).

---

### 3.18 — MF Transaction Flows (Buy / SIP / Sell Once / SWP / Transactions)

> **This section is sequence-accurate**: every step is in the same order the
> Angular app executes today, with the exact wrapper route, the upstream
> Cybrilla URL, the request/response shape, and the DB tables touched.
> Every line below is traceable to the codebase — no invented endpoints.
>
> **Gateway summary (see Section 3.18.6 for full table):**
> - **Lump-sum purchase** → gateway defaults from the scheme/MFIA config (we don't override).
> - **SIP (purchase plan)** → gateway defaults from the scheme; ONDC plans auto-confirm via `state=confirmed` after `review_completed`.
> - **One-time redemption** → we **omit** `gateway` from the create body; FP picks it from the folio.
> - **SWP (redemption plan)** → backend **forces `order_gateway = "rta"`** (FP doesn't support SWP on ONDC); ref `MfOrdersController.cs:856`.

#### 3.18.0 — Pre-flight: pre-verification readiness gate

Every MF transaction depends on the investor being KYC-compliant. Before showing the invest / sell UI, the SPA reads `GET /api/kyc/forms/my-status` (or the legacy `GET /api/pre-verifications/my-status`) to know whether to:

| Cybrilla `readiness.status` / `readiness.code` | Our decision | Frontend action |
|---|---|---|
| `status = "verified"` | **KRA data exists — investor already KYC-compliant** | Short-circuit to the `already_kyc_valid` screen → go invest. The kyc-form page sets stage `already_kyc_valid` (no form). `nextAction = "run_pan_validation"` from the controller is only used to continue PAN/name/DOB confirmation when explicitly needed. |
| `code = "kyc_unavailable"` | No KRA record for this PAN | Fresh-KYC form, `type=fresh` (`nextAction = "start_fresh_kyc"`) |
| `code = "unknown"` | KRA couldn't determine | Fresh-KYC form, `type=fresh` (`nextAction = "start_fresh_kyc"`) |
| `code = "kyc_incomplete"` | KRA has a record but it's missing details (Aadhaar link / email / phone) | **MODIFY-KYC form, `type=modify`** (`nextAction = "start_modify_kyc"`). Per the Cybrilla Pre-Verification doc this is *fixable via a modify form* — NOT a dead-end "fix at KRA" link. |
| anything else | Transient | Retry readiness (`nextAction = "retry_readiness"`) |

The branch lives in `PreVerificationsController.cs` (`DeriveNextAction` + `DeriveNextActionFromResponse`). Fresh/modify KYC = `POST /api/kyc/forms`; verified KRA = skip directly to investor profile + bank setup.

> **CHANGED (2026-06-08):** `kyc_incomplete` now maps to `start_modify_kyc` (was `fix_at_kra`). The old `fix_at_kra` dead-ended the user with no path forward; the doc says `kyc_incomplete` is repairable through a `type=modify` KYC Form. `fix_at_kra` is retained in the frontend switch only for backward-compat with older cached pre-verification rows. Additionally, `readiness.status=verified` now short-circuits to the `already_kyc_valid` screen instead of pushing the investor through the modify-form machinery.

---

#### 3.18.1 — BUY LUMP-SUM (mutual-fund-invest "Lump-sum" tab)

**Lifecycle the order must reach:** `pending → confirmed → submitted → successful` (or `failed | cancelled | reversed`).

| # | Step | Wrapper (our endpoint) | Cybrilla (upstream) | DB write |
|---|---|---|---|---|
| 1 | Load eligibility (KYC + bank verified + MFIA + scheme min_investment). | `GET /api/kyc/forms/my-status`, `GET /api/banking/my-status`, `GET /api/onboarding/status` | n/a (local snapshots) | n/a |
| 2 | Create purchase order. | `POST /api/mf/orders/purchases` with `Idempotency-Key` header | `POST /v2/mf_purchases` | `tbl_Fintech_MF_Purchase_Order` INSERT, `tbl_Fintech_MF_Order_Log` step=`create` |
| 3 | Send consent OTP. | `POST /api/mf/orders/purchases/{id}/send-otp` (body `{mobile?, email?, isd_code?}`) | **none** (internal). Sandbox magic OTP `123456`. | `tbl_Fintech_Purchase_Otp` INSERT (SHA-256 hash) |
| 4 | (optional) Peek-verify OTP before PATCH. | `POST /api/mf/orders/purchases/{id}/verify-otp` | none | reads `tbl_Fintech_Purchase_Otp` |
| 5 | Apply consent + move state to `confirmed`. | `PATCH /api/mf/orders/purchases` | `PATCH /v2/mf_purchases` | `tbl_Fintech_MF_Purchase_Order` UPDATE state, `tbl_Fintech_Purchase_Otp` ConsumedAt |
| 6 | Create netbanking/UPI payment. | `POST /api/pg/payments/netbanking` OR `…/upi` | `POST /v2/payments` (FP normalises both into one endpoint via `method` discriminator) | `tbl_Fintech_Payment` INSERT (status=PENDING + `token_url`) |
| 7 | User pays at bank → bank POSTs back. | `POST /api/pg/payments/postback` (303 → SPA) | n/a | `tbl_Fintech_Payment` UPDATE PostbackUrl |
| 8 | Poll until terminal. | `GET /api/pg/payments/{id}` | `GET /v2/payments/{id}` (server-side; controller refreshes local row) | snapshot upsert |
| 9 | Webhooks complete the lifecycle. | `POST /api/webhooks/finprim-events` (receiver) | events `payment.success/failure`, `mf_purchase.submitted/successful` | `tbl_Fintech_Payment.Status`, `tbl_Fintech_MF_Purchase_Order.State`, `AllottedUnits`, `FolioNumber`, `tbl_Fintech_Webhook_Event_Log` |

**Step 2 — create — request body** (key fields):
```json
{
  "mf_investment_account": "mfia_xxx",
  "scheme":                "INF...01XX0",
  "amount":                5000,
  "folio_number":          null,
  "source_ref_id":         "<uuid you generate>",
  "euin":                  null
}
```
Headers: `Idempotency-Key: <uuid>` (replay window 30 min — `tbl_Fintech_MF_Purchase_Order.IdempotencyKey`).

**Step 2 — response**:
```json
{ "id": "mfp_xxx", "old_id": 12345, "state": "pending",
  "amount": 5000, "scheme": "INF...", "gateway": "ondc" }
```

**Step 5 — PATCH consent body**:
```json
{
  "id": "mfp_xxx",
  "state": "confirmed",
  "consent": {
    "email":    "user@example.com",
    "isd_code": "+91",
    "mobile":   "9876543210",
    "otp":      "123456"
  }
}
```
OTP is verified server-side against `tbl_Fintech_Purchase_Otp` BEFORE we forward to FP (`MfOrdersController.cs:331-361`). Failed OTP → 422 with reason `wrong | expired | locked | no_otp`.

**Step 6 — payment body** (netbanking & UPI use the same shape, `method` discriminates):
```json
{
  "amc_order_ids":        [12345],
  "method":               "NETBANKING",
  "bank_account_id":      29,
  "provider_name":        "ONDC",
  "payment_postback_url": "https://app.islamicly.com/pay/return"
}
```
- `amc_order_ids` = Cybrilla's **numeric `old_id`** of the purchase (NOT `mfp_xxx`).
- `bank_account_id` = `tbl_Fintech_Bank_Account.OldId` (NOT the local `Id`, NOT `bac_xxx`).

**Step 6 — response**:
```json
{ "id": 987654, "token_url": "https://finprim-sandbox.cybrilla.dev/payment/...",
  "status": "PENDING", "amount": 5000 }
```

The SPA opens `token_url`. When the user finishes at the bank, FP form-POSTs to our `payment_postback_url`, which 303-redirects to the SPA. The SPA then polls Step 8 and listens for the webhook chain in Step 9.

---

#### 3.18.2 — SIP (mutual-fund-invest "SIP" tab — recurring purchase plan)

**Lifecycle:** `created → active → completed` (or `cancelled / failed`). ONDC plans go through an extra `review_completed → confirmed → active` transition that the backend auto-confirms.

| # | Step | Wrapper | Cybrilla | DB write |
|---|---|---|---|---|
| 1 | Mandate must exist & be `approved`. List existing first. | `GET /api/pg/mandates/my-list` | n/a (local) | reads `tbl_Fintech_Mandate` |
| 2 | If none, create a mandate (eNACH or UPI Autopay). | `POST /api/pg/mandates` | `POST /v2/mandates` | `tbl_Fintech_Mandate` INSERT |
| 3 | Authorize the mandate (token URL). | `POST /api/pg/mandates/{id}/authorize?isWeb=1` | `POST /v2/mandates/{id}/authorize` | `tbl_Fintech_Mandate` UPDATE TokenUrl |
| 4 | User signs at bank → postback. Webhook `mandate.approved` flips status. | `POST /api/pg/mandates/postback` (303 → SPA) | events `mandate.received/approved/rejected/cancelled` | `tbl_Fintech_Mandate.MandateStatus`, `ApprovedAt` |
| 5 | Send plan consent OTP. | `POST /api/mf/orders/purchase-plans/send-otp` with header `Plan-Intent-Key: <key>` | none (internal) | `tbl_Fintech_Purchase_Otp` |
| 6 | Create the SIP plan with consent. | `POST /api/mf/orders/purchase-plans` with `Idempotency-Key` + `Plan-Intent-Key` headers | `POST /v2/mf_purchase_plans` | `tbl_Fintech_MF_Purchase_Plan` INSERT |
| 7 | (ONDC only) wrapper polls 15 s for `review_completed` then PATCHes `state=confirmed`. | (internal — `MfOrdersController.TryAutoConfirmPlanAsync`) | `PATCH /v2/mf_purchase_plans` | `tbl_Fintech_MF_Purchase_Plan` UPDATE |
| 8 | Webhooks drive the plan through `active`. | (receiver) | events `mf_purchase_plan.created/activated/cancelled/failed/completed` | snapshot updates |
| 9 | Each scheduled installment fires as a normal `mf_purchase` (state machine identical to lump-sum). | n/a | webhook `mf_purchase.*` | `tbl_Fintech_MF_Purchase_Order` rows where `Plan = "mfpp_xxx"` |
| 10 | (sandbox only) force-generate next installment. | `POST /api/mf/orders/purchase-plans/{id}/simulate-installment` | `POST /v2/mf_purchases { plan: "mfpp_xxx" }` | new purchase row |

**Step 6 — create plan body**:
```json
{
  "mf_investment_account":  "mfia_xxx",
  "scheme":                 "INF...",
  "amount":                 5000,
  "installment_day":        15,
  "frequency":              "monthly",
  "number_of_installments": 12,
  "payment_method":         "mandate",
  "payment_source":         "123",          // mandate.old_id AS STRING
  "consent": {
    "email":    "user@example.com",
    "isd_code": "+91",
    "mobile":   "9876543210",
    "otp":      "123456"
  }
}
```
Headers: `Idempotency-Key`, `Plan-Intent-Key`. Email/mobile/isd_code are auto-stamped from the KYC form by `StampConsentContactAsync` if the client omits them.

**Step 6 — response (ONDC path)**:
```json
{ "id": "mfpp_xxx", "old_id": 9876,
  "state": "review_completed",     // ← controller polls for this
  "gateway": "ondc", "frequency": "monthly", "amount": 5000 }
```
After auto-confirm (Step 7) state becomes `active` and FP starts auto-debiting.

---

#### 3.18.3 — SELL ONCE / one-time redemption (mutual-fund-swp "Sell Once" modal)

**Lifecycle:** `pending → confirmed → submitted → successful` (or `failed | cancelled | reversed`). A `pending` order auto-fails T+1 post cut-off if not confirmed.

**Doc-mandated 2-step flow:** POST creates the order with **NO consent** (state `pending`); a separate PATCH applies consent and moves state to `confirmed`. Forwarding consent on the POST is rejected by FP.

| # | Step | Wrapper | Cybrilla | DB write |
|---|---|---|---|---|
| 1 | User clicks **Sell Once**, picks a holding from the dropdown. | none | n/a | n/a |
| 2 | Fetch live redeemable max for that folio. | `POST /api/reports/holdings` (body `{investment_account_id: "mfia_xxx", folios?: "...,"}`). Controller resolves `mfia_xxx` → `old_id` and forwards. | `GET /api/oms/reports/holdings?investment_account_id=<old_id>&folios=…&as_on=…` | n/a (read-only) |
| 3 | Send consent OTP. | `POST /api/mf/orders/redemptions/send-otp` with header `X-Plan-Intent-Key: <key>` | none (internal) | `tbl_Fintech_Purchase_Otp` |
| 4 | Create redemption — **no `consent`, no `gateway`**. | `POST /api/mf/orders/redemptions` with `X-Idempotency-Key` | `POST /v2/mf_redemptions` | `tbl_Fintech_MF_Redemption` INSERT state=`pending` |
| 5 | Apply consent → `state=confirmed`. | `PATCH /api/mf/orders/redemptions` with `X-Plan-Intent-Key` | `PATCH /v2/mf_redemptions` | UPDATE state, `tbl_Fintech_Purchase_Otp` ConsumedAt |
| 6 | Webhooks complete it. | receiver | `mf_redemption.confirmed → .submitted → .successful` (or `.failed/.cancelled/.reversed`) | `tbl_Fintech_MF_Redemption` snapshot, **`tbl_Fintech_Folio.Units -= redeemed_units`** on success (idempotent via `BalanceAppliedAt` claim) |
| 7 | Manual sync fallback if webhook lost. | `GET /api/mf/orders/redemptions/{id}` | `GET /v2/mf_redemptions/{id}` | snapshot + folio decrement (same `TryClaim` so it never double-decrements) |
| 8 | View history. | `GET /api/mf/orders/redemptions/my-list` | n/a (local) | reads `tbl_Fintech_MF_Redemption` |

**Eligibility checks (per FP "Create a Redemption Order" doc) — do BEFORE creating:**
- From the **scheme** (`GET fund scheme`): `redemption_allowed` must be `true`; note `min_withdrawal_amount` / `max_withdrawal_amount` / `withdrawal_multiples` (amount path) and `min_withdrawal_units` / `max_withdrawal_units` / `withdrawal_multiples_units` (units path).
- From the **holdings report** (step 2): `redeemable_units` (and redeemable amount) must be `> 0`.
- **Redeem by amount:** amount in `[min_withdrawal_amount, max_withdrawal_amount]` and a multiple of `withdrawal_multiples`.
- **Redeem by units:** units in `[min_withdrawal_units, max_withdrawal_units]` and a multiple of `withdrawal_multiples_units`.
- The web client gets the live max from the holdings report (`scheme.holdings.redeemable_units`) and caps the input; full validation against scheme multiples should be added where strict.

**OTP recipient (step 3, per doc):** SEBI requires the OTP go to the email/mobile **registered against the folio**. Fetch them from the folio (`email_addresses` / `mobile_numbers` on the List-folios object). Sandbox OTP is the fixed `123456`.

**Step 4 — create body** (omit BOTH `amount` and `units` → **Sell All**, per FP docs):
```json
{
  "mf_investment_account": "mfia_xxx",
  "scheme":                "INF...",
  "folio_number":          "AGXLMTV6ZLBI4",
  "amount":                500,             // optional — between min/max_withdrawal_amount
  "units":                 null,            // optional — RTA-only
  "redemption_mode":       "normal"         // or "instant" (liquid funds, RTA, OTP mandatory)
}
```
- IPv4 `user_ip` / `server_ip` are stamped server-side by `ResolveClientIpv4()` (FP rejects `::1` / IPv6).
- `gateway` is **omitted** so FP picks it from the folio (ONDC folio → ondc, RTA folio → rta). Forcing one would fail.

> **VERIFIED PAYLOAD TRACE (re-checked across frontend + service + backend, 2026-06-08):** The create body the client actually sends is **exactly** `{ mf_investment_account, scheme, folio_number, amount?|units? }` — `startSell()` in `mutual-fund-swp.component.ts` builds only these; sell-all omits both amount and units. The service `createRedemption()` adds the `X-Idempotency-Key` header only. The backend `CreateRedemption` strips `consent`, stamps `user_ip`/`server_ip`, and sets **NO gateway**. So **we never send a gateway** — FP derives ONDC from the folio and the `"gateway":"ondc"` in the response is FP echoing what it derived, not what we sent. The confirm body the client sends is `{ id, state:"confirmed", consent:{ otp } }`; the backend `ConfirmRedemption` verifies+consumes the OTP, then stamps `consent.email/isd_code/mobile` (via `Usp_Get_Fintech_User_Contact`) before `PATCH /v2/mf_redemptions`.

> ⚠️ **KNOWN GAP (web): no poll-for-`pending` before confirm.** `startSell()` goes straight from create → OTP/confirm step without checking the create response `state`. On ONDC folios the create returns `under_review` (async review), so the confirm PATCH then 400s *"order is not in pending state"*. **Required fix (not yet applied): after create, poll `GET /api/mf/orders/redemptions/{id}` until `state=pending`, THEN allow consent.** RTA folios return `pending` immediately so they're unaffected.
- `units` redemption and `redemption_mode = instant` are **RTA-only** per the doc ("units redemption only allowed for rta gateway"; `ondc` units = "coming soon"). **All our folios are ONDC → use AMOUNT-based redemption; keep "By Units" disabled until Cybrilla enables ONDC unit-redemption.** ONDC's redemption order contract is amount-based (the RTA converts amount→units at NAV); units-level instructions need the direct-RTA rail ONDC doesn't yet expose.

> ⚠️ **ONDC create returns `under_review`, not `pending` (by design).** The lifecycle on ONDC folios is `POST → under_review → (async Cybrilla review) → pending → confirmed → …`. **Do NOT PATCH consent while `under_review`** — FP rejects it with *"order is not in pending state"*. **Poll `GET /api/mf/orders/redemptions/{id}` until `state=pending`, THEN send consent.** This is a Cybrilla-side async review, not a flag we can reset. (RTA folios return `pending` immediately, so this only affects ONDC.)

**Step 4 — response**:
```json
{ "id": "mfr_xxx", "old_id": 6597, "state": "pending",
  "amount": 500, "scheme": "INF...", "folio_number": "AGXLMTV6ZLBI4",
  "gateway": "ondc", "redemption_mode": "normal" }
```

**Step 5 — PATCH consent body**:
```json
{
  "id":    "mfr_xxx",
  "state": "confirmed",
  "consent": {
    "email":    "user@example.com",
    "isd_code": "+91",
    "mobile":   "9876543210",
    "otp":      "123456"
  }
}
```
Server consumes the internal OTP (`tbl_Fintech_Purchase_Otp.ConsumedAt`) and **stamps `email`/`isd_code`/`mobile` when the client didn't send them** (the client typically sends only `otp`).

> ⚠️ **Consent contract — email or mobile is MANDATORY (gotcha, fixed 2026-06-08).** FP rejects a consent block that has only an OTP with:
> `400 — { "field": "consent", "message": "investor consent : email or mobile number should be present" }`.
> The backend `StampConsentContactAsync` fills the contact, but it originally read ONLY `tbl_Fintech_Kyc_Form` — which is **NULL on the verified-KRA path** (no kyc_form), so consent went out with just the OTP and 400'd. **Fix:** new SP `Usp_Get_Fintech_User_Contact` returns the best-available contact with priority **kyc_form → `tbl_Fintech_Investor_Email`/`_Phone` (onboarding) → pre-verification**; the stamp methods (redemption + SWP plan) now use it via `IKycFormDbService.GetUserContactAsync`. So consent always carries email/mobile regardless of which KYC path the investor took. **App team:** you may send `consent.{email,isd_code,mobile,otp}` explicitly, or send just `otp` and rely on the backend stamp — both work now.

**Folio decrement** (on `mf_redemption.successful` OR manual sync hitting a `successful` state):
```sql
UPDATE dbo.tbl_Fintech_Folio
   SET Units       = Units - @RedeemedUnits,
       MarketValue = MarketValue - @RedeemedAmount,
       UpdatedAt   = GETUTCDATE()
 WHERE FolioNumber = @Folio AND (Scheme = @Scheme OR @Scheme IS NULL);
```
Gated by `Usp_…_TryClaimBalanceApply` so the same redemption can't decrement twice if both webhook and manual sync race.

---

#### 3.18.4 — SWP (mutual-fund-swp "Start a New SWP" — recurring redemption plan)

> 🚫 **NOT SUPPORTED / UI HIDDEN (2026-05-27).** SWP is **not currently available to investors.** FP supports redemption *plans* only on the `rta` gateway (*"For ondc as gateway, currently only MF Purchase plans are supported."*), and our tenant is ONDC — so SWP cannot run on our schemes. The "Start a New SWP" button and the landing-page tile have been **removed**. The component (`mutual-fund-swp`) and all backend endpoints below are **retained but dormant**, ready to re-enable if/when Cybrilla enables redemption plans on ONDC. **Treat this entire sub-section as reference for the dormant code path, not a live flow.**

**Lifecycle (when enabled):** `created → active → cancelled / completed / failed`.

**Hard rule from FP docs:** *"For ondc as gateway, currently only MF Purchase plans are supported."* SWP is **RTA-only**. The wrapper forces `order_gateway="rta"` on every create — see `MfOrdersController.cs:856`. Our tenant is ONDC, so SWP works only on RTA-enabled schemes — which we don't have, hence it stays hidden.

| # | Step | Wrapper | Cybrilla | DB write |
|---|---|---|---|---|
| 1 | Pick the holding to withdraw from (uses the same holdings sourcing as Sell Once). | (local) | n/a | n/a |
| 2 | Send plan consent OTP. | `POST /api/mf/orders/redemption-plans/send-otp` with `Plan-Intent-Key` | none (internal) | `tbl_Fintech_Purchase_Otp` |
| 3 | Create SWP plan with consent. | `POST /api/mf/orders/redemption-plans` with `Idempotency-Key` + `Plan-Intent-Key` | `POST /v2/mf_redemption_plans` (controller forces `order_gateway = "rta"`) | `tbl_Fintech_MF_Redemption_Plan` INSERT |
| 4 | (optional) Modify amount / day / count. | `PATCH /api/mf/orders/redemption-plans` | `PATCH /v2/mf_redemption_plans` | UPDATE |
| 5 | Each installment fires as a normal `mf_redemption`. | n/a | webhook `mf_redemption.*` | row in `tbl_Fintech_MF_Redemption` where `Plan = "mfrp_xxx"` |
| 6 | Pause (skip window). | `POST /api/mf/orders/redemption-plans/{id}/pause` body `{from, to?}` | `POST /v2/mf_redemption_plans/{id}/skip_instructions` | skip row |
| 7 | Resume (cancel skip). | `POST /api/mf/orders/redemption-plans/skip-instructions/{skipId}/cancel` | `POST /v2/mf_redemption_plans/skip_instructions/{skipId}/cancel` | skip row terminal |
| 8 | Cancel the SWP. | `POST /api/mf/orders/redemption-plans/{id}/cancel` | `POST /v2/mf_redemption_plans/{id}/cancel` | UPDATE state |
| 9 | View / sync. | `GET /api/mf/orders/redemption-plans/{id:regex(^mfrp_.+$)}` (self-heals) and `…/my-list` | `GET /v2/mf_redemption_plans/{id}` | snapshot |

**Step 3 — create body** (consent IS sent in create for SWP, unlike one-time redemption):
```json
{
  "mf_investment_account":  "mfia_xxx",
  "scheme":                 "INF...",
  "folio_number":           "1234567890",
  "amount":                 1000,
  "frequency":              "monthly",
  "installment_day":        15,
  "number_of_installments": 12,
  "systematic":             true,
  "generate_first_installment_now": false,
  "auto_generate_installments":     true,
  "consent": {
    "email":    "...",
    "isd_code": "+91",
    "mobile":   "...",
    "otp":      "123456"
  }
}
```
`user_ip`, `server_ip`, and `order_gateway = "rta"` are stamped server-side.

---

#### 3.18.5 — Transactions / Sell-Txns / Mandates pages (read-only)

These pages render from the **local snapshots** (kept fresh by webhooks + the per-row sync buttons). No Cybrilla call on page load — only on user-triggered sync.

| Page | Wrapper | Source | Per-row sync |
|---|---|---|---|
| Transactions (Orders tab) | `GET /api/mf/orders/purchases/my-list` | `tbl_Fintech_MF_Purchase_Order` + LEFT JOIN latest payment | `GET /api/mf/orders/purchases/{id:regex(^mfp_.+$)}` → fetches FP + upserts snapshot |
| Transactions (SIPs tab) | `GET /api/mf/orders/purchase-plans/my-list` | `tbl_Fintech_MF_Purchase_Plan` LEFT JOIN `tbl_Fintech_Mandate` | `GET /api/mf/orders/purchase-plans/{id:regex(^mfpp_.+$)}` |
| Transactions (Mandates tab) | `GET /api/pg/mandates/my-list` | `tbl_Fintech_Mandate` | `GET /api/pg/mandates/{id}` |
| MF Sell Txns | `GET /api/mf/orders/redemptions/my-list` | `tbl_Fintech_MF_Redemption` | `GET /api/mf/orders/redemptions/{id}` (also triggers idempotent folio decrement when state hits `successful`) |
| SWPs page | `GET /api/mf/orders/redemption-plans/my-list` | `tbl_Fintech_MF_Redemption_Plan` | `GET /api/mf/orders/redemption-plans/{id:regex(^mfrp_.+$)}` |

The list shapes use **camelCase** JSON (the row classes carry `[JsonPropertyName(...)]` because the global naming policy is `null`). Sample row from `…/redemptions/my-list`:
```json
{
  "id":             42,
  "mfRedemptionId": "mfr_xxx",
  "oldId":          6597,
  "mfiaId":         "mfia_xxx",
  "scheme":         "INF...",
  "schemeName":     "Tata Ethical Fund",
  "folioNumber":    "AGXLMTV6ZLBI4",
  "amount":         500,
  "units":          null,
  "state":          "successful",
  "allottedUnits":  3.182,           // = redeemed_units
  "purchasePrice":  157.12,          // = redeemed_price
  "redeemedAmount": 500,
  "gateway":        "ondc",
  "redemptionMode": "normal",
  "failureCode":    null,
  "idempotencyKey": "sell_…",
  "balanceAppliedAt": "2026-05-27T10:09:13Z",
  "createdAt":      "2026-05-27T10:08:56Z",
  "updatedAt":      "2026-05-27T10:09:13Z"
}
```

---

#### 3.18.6 — Gateway selection reference

| Flow | What we send to FP | Why |
|---|---|---|
| Lump-sum purchase | (no override) | Gateway comes from the scheme/MFIA. Most ONDC schemes return `gateway: "ondc"`. |
| SIP (purchase plan) | (no override) | Same as above. ONDC plans need our auto-confirm hop after `review_completed`. |
| One-time redemption | (no override — `Gateway` field exists on the model but is left null) | FP picks the gateway from the folio. Forcing `rta` on an ONDC folio fails ("Invalid IP address" / "Folio not found"); forcing `ondc` may be rejected because the doc still marks ondc-redemption as early-access. |
| SWP (redemption plan) | `order_gateway: "rta"` (forced) | FP doc: *"For ondc as gateway, currently only MF Purchase plans are supported."* SWP rejects ondc outright. |
| Holdings report path | `gateway=cybrillapoa` (normalised lowercase, case-sensitive) | FP says: *"Unsupported value 'ONDC'. Supports one of [cybrillapoa, ncdex] and its case sensitive"*. `MasterDataService.NormalizeGateway()` maps `ONDC|ondc|cybrillaondc|poa → cybrillapoa`. |

---

#### 3.18.7 — Idempotency, OTP, and consent stamping rules

- **Idempotency**: `Idempotency-Key` (purchases, SIP, redemption, SWP). On replay within 30 min the wrapper returns the cached original FP response — no second create on FP, no second debit. Stored on `tbl_Fintech_*.IdempotencyKey`.
- **Intent key**: `Plan-Intent-Key` / `X-Plan-Intent-Key` groups a `send-otp → verify-otp → confirm` triple so the OTP is scoped to that intent.
- **OTP storage**: `tbl_Fintech_Purchase_Otp` stores the SHA-256 hash; 5 wrong attempts auto-locks the row.
- **Consent contact**: when the SPA omits `email` / `isd_code` / `mobile`, the wrapper stamps them from the latest `tbl_Fintech_Kyc_Form` row (`StampConsentContactAsync` / `StampPlanConsentContactAsync`). The OTP itself must come from the client.

---

### 3.19 — Payments (low-level reference)

After a purchase reaches `confirmed`, payment creation kicks in (Step 6 of §3.18.1).

#### `POST /api/pg/payments/netbanking`  and  `POST /api/pg/payments/upi`
```json
{
  "amc_order_ids":        [12345],
  "method":               "NETBANKING",          // or "UPI"
  "bank_account_id":      29,
  "provider_name":        "ONDC",
  "payment_postback_url": "https://app.islamicly.com/pay/return"
}
```
Both routes resolve to FP's single `POST /v2/payments` with `method` discriminator. **`amc_order_ids`** is the purchase's numeric `old_id`, NOT `mfp_xxx`. **`bank_account_id`** is `tbl_Fintech_Bank_Account.OldId`, NOT the local autoincrement `Id`.

#### `POST /api/pg/payments/nach`
```json
{ "mandate_id": 555, "amc_order_ids": [12345] }
```
Used when the order is an installment of an active SIP — debit hits the linked mandate instead of opening a bank redirect.

#### `GET /api/pg/payments/{id}` / `GET /api/pg/payments`
List & fetch. The wrapper returns the raw Cybrilla payment object including the redirect `token_url`. After bank redirect, the SPA polls this until `Status` is terminal, while webhooks (`payment.success | payment.failure | payment.submitted`) update the local row.

---

## 4. `isWeb` / Postback URL details

Summary of the two endpoints that produce user-facing redirect URLs. URLs are read **verbatim** from `appsettings.json` — there is no string composition. `urlPath` is no longer a query parameter.

| Endpoint | `isWeb` | Config key | Postback resolved to (dev / UAT) |
|---|---|---|---|
| `PATCH /kyc/requests/{id}/address` | 1 (web, default) | `Finprim:Postback:Web:DigiLocker` | `http://localhost:4200/admin/mutual-funds/kyc` |
| `PATCH /kyc/requests/{id}/address` | 0 (app)          | `Finprim:Postback:App:DigiLocker` | `http://localhost:4200/appmutualfund/kyc` |
| `POST  /kyc/requests/{id}/esign`   | 1 (web, default) | `Finprim:Postback:Web:Esign`      | `http://localhost:4200/admin/mutual-funds/kyc?esign=completed` |
| `POST  /kyc/requests/{id}/esign`   | 0 (app)          | `Finprim:Postback:App:Esign`      | `http://localhost:4200/appmutualfund/esign` |
| `POST  /pg/mandates`               | 1 (web, default) | `Finprim:Postback:Web:Mandate`    | `https://localhost:7001/api/pg/mandates/postback` → 303 → `/app/mutual-funds/mandate-callback` |
| `POST  /pg/mandates`               | 0 (app)          | `Finprim:Postback:App:Mandate`    | `https://localhost:7001/api/pg/mandates/postback?source=app` → 303 → `/appmutualfund/mandate` |
| `POST  /pg/payments/netbanking` | 1 (web, default) | `Finprim:Postback:Web:Payment` | `https://localhost:7001/api/pg/payments/postback` → 303 → `/app/mutual-funds/payment-callback` |
| `POST  /pg/payments/netbanking` | 0 (app)          | `Finprim:Postback:App:Payment` | `https://localhost:7001/api/pg/payments/postback?source=app` → 303 → `/appmutualfund/payment` |
| `POST  /pg/payments/upi`        | 1 (web, default) | `Finprim:Postback:Web:Payment` | same as netbanking (single backend handler) |
| `POST  /pg/payments/upi`        | 0 (app)          | `Finprim:Postback:App:Payment` | same as netbanking (single backend handler) |

In UAT, replace `http://localhost:4200` with `https://fintechuat.islamicly.com`. Helper: `KycController.GetPostbackUrl(isWeb, flow)` at `:281-296`.

**Mandate-specific routing:** the App URL points to the **backend** (`/api/pg/mandates/postback?source=app`) rather than directly to the SPA because BillDesk/Cybrilla POSTs a form body. Backend reads it, logs an audit row in `tbl_Fintech_MF_Order_Log`, then 303-redirects to the SPA route picked by `?source=` — see `MandatePostbackController.cs`.

**Payment polling strategy:** the **Web** payment-callback page polls `/api/pg/payments/{id}/snapshot` + `/api/mf/orders/purchases/{mfp}/snapshot` every 2 s for ~60 s. The **App** payment page does **not** poll — it deep-links `islamicly://mf-callback?flow=payment&id=&mfp=&status=` and the native app does its own snapshot polling. This keeps the WebView a thin pass-through.

**Mobile WebView checklist** (see section 5.1 for full detail):
- Host DigiLocker / eSign redirect URLs inside a WebView. No external browser, no Chrome Custom Tabs, no SFSafariViewController.
- Watch for the WebView URL to match `/appmutualfund/kyc` or `/appmutualfund/esign` → close the WebView and call `/api/kyc/user-status` (then the relevant `/confirm` endpoint).
- The standalone Angular landing page exposes `data-flow`, `data-status`, `data-result`, `data-reason` on `<div class="meta">` for DOM-based parsing, and a "Return to App" button that opens `islamicly://kyc-callback?flow=&status=&id=&reason=` if you choose to register that scheme.
- Registering the `islamicly://` URI scheme is OPTIONAL and only relevant for the "Return to App" tap flow; URL-watch interception works without it.

---

## 5. Webhooks

### Receiver

The wrapper exposes a public receiver at:

```
POST https://fintechuat.islamicly.com/api/webhooks/finprim-events
```

(in dev: `http://localhost:5xxx/api/webhooks/finprim-events`). It is **WIDE OPEN** — `[AllowAnonymous]`, no token in the path, no signature verification. Idempotency is enforced solely by the `EventId UNIQUE` constraint on `tbl_Fintech_Webhook_Event_Log` (the dispatcher swallows the duplicate-key violation and returns 200 with no side effects).

Source: `F:\github projects\FinprimApi\Controllers\FinprimWebhookReceiverController.cs`.

### Subscribing (bootstrap)

Run **once per environment** to register every event we care about:

```
POST /api/webhooks/bootstrap?url=https://fintechuat.islamicly.com/api/webhooks/finprim-events
```

This iterates `FinprimEventCatalog.AllEvents` and POSTs one Finprim `notification_webhooks` subscription per event (each subscription supports only ONE event type — so we create many). Re-running is safe.

### Event catalog

Source: `FinprimApi/Services/FinprimEventCatalog.cs`.

**KYC**
- `kyc_request.esign_required`
- `kyc_request.submitted`
- `kyc_request.successful`
- `kyc_request.rejected`
- `kyc_request.expired`

**MF Purchase (lump-sum)**
- `mf_purchase.created`
- `mf_purchase.confirmed`
- `mf_purchase.submitted`
- `mf_purchase.successful`
- `mf_purchase.failed`
- `mf_purchase.cancelled`
- `mf_purchase.reversed`

**MF Redemption**
- `mf_redemption.created` / `.confirmed` / `.submitted` / `.successful` / `.failed` / `.cancelled` / `.reversed`

**MF Switch**
- `mf_switch.created` / `.confirmed` / `.submitted` / `.successful` / `.failed` / `.cancelled` / `.reversed`

**Purchase / Redemption / Switch Plans (SIP/SWP/STP)**
- `mf_purchase_plan.created` / `.activated` / `.cancelled` / `.failed` / `.completed`
- `mf_redemption_plan.created` / `.activated` / `.cancelled` / `.failed` / `.completed`
- `mf_switch_plan.created` / `.activated` / `.cancelled` / `.failed` / `.completed`

**Mandate (NACH / eMandate for SIPs)**
- `mandate.created` / `.received` / `.submitted` / `.approved` / `.rejected` / `.cancelled`

**Payment**
- `payment.pending` / `.initiated` / `.submitted` / `.approved` / `.success` / `.updated` / `.rejected` / `.failed`

### Webhook vs. postback URLs

- **Postback URLs** (DigiLocker, eSign, payments) are **synchronous user-facing redirects** — they bounce the user back into your app's UI. They are NOT a reliable backend signal.
- **Webhooks** are the **asynchronous backend signal**. They keep our SQL DB in sync with Finprim regardless of whether the user closed the app or finished the redirect.
- **DB-first navigation**: After any redirect, the mobile app should call `GET /api/kyc/user-status` (or `/onboarding/status`, `/banking/my-status`) and trust the DB, because the webhook may have already arrived before the user returns.

---

## 5.5 Pre-Verification Flow (3 stages, resumable)

> **New as of the pre-verifications redesign.** Replaces the blind KYC wizard with an upfront readiness + identity sanity check, so the user never gets parked deep in the wizard only to fail validation later. All endpoints are mounted under `/api/pre-verifications` and implemented in `F:\github projects\FinprimApi\Controllers\PreVerificationsController.cs`. They wrap Cybrilla POA's `/poa/pre_verifications` API — see [poa.cybrilla.com/docs/additional-apis/pre-verifications](https://poa.cybrilla.com/docs/additional-apis/pre-verifications) for the upstream contract.

### Purpose

Instead of pushing the user through the full KYC wizard and hoping every field passes, we run a cheap pre-flight against KRA / ITD / NSDL up front:

- **Stage 1 — Readiness** (`/readiness`): Is the PAN linked to Aadhaar, KRA-validated, and otherwise eligible to onboard? Cheap, can be repeated.
- **Stage 2 — PAN+Name+DOB validation** (`/pan-validate`): Confirms the trio the user typed matches NSDL/KRA records before we let them progress.
- **Stage 3 — Bank account validation** (driven by the same record once the user adds an account): Penny-drop / KRA mode depending on residency.

Every transition between stages is persisted (`tbl_Fintech_PreVerification`) so the user can close the app and resume — the frontend hydrates state via `GET /api/pre-verifications/my-status`.

### Endpoints

| Method | Path | Body / Query | Purpose |
|--------|------|--------------|---------|
| `POST` | `/api/pre-verifications/readiness` | `{ "pan": "ABCDE1234F", "isWeb": true }` | Stage 1 — kick off (or repeat) a readiness check. Returns `{ id, status, nextAction, readiness, ... }`. |
| `POST` | `/api/pre-verifications/pan-validate` | `{ "pan", "name", "dateOfBirth": "YYYY-MM-DD", "isWeb": true }` | Stage 2 — run PAN/Name/DOB validation on the same `pre_verification` record. |
| `GET`  | `/api/pre-verifications/{id}/poll` | — | Re-fetch from Cybrilla and persist; use while `status="in_progress"`. |
| `GET`  | `/api/pre-verifications/my-status` | — | Resume support — returns the latest persisted record for the authenticated user, including current stage and `nextAction`. |

All four return the standard `{ statusCode, message, result }` envelope. The `result` object always includes `nextAction` (see below) so the frontend can route purely on that field.

### `nextAction` enum — what the frontend should do

| `nextAction` | Stage | Meaning | Frontend behavior |
|--------------|-------|---------|-------------------|
| `poll` | any | Cybrilla is still processing | Call `GET /{id}/poll` every 2–3 s (max 30 s), then re-read `nextAction`. |
| `run_readiness` | 1 | No readiness check yet (or one was abandoned) | Show the PAN entry screen and `POST /readiness`. |
| `run_pan_validation` | 2 | Readiness passed; PAN+Name+DOB validation pending | Collect name + DOB, then `POST /pan-validate`. |
| `run_bank_validation` | 3 | PAN trio passed; bank account not yet validated | Push the user into the bank-add screen; backend will trigger penny-drop on `/banking/my-verify`. |
| `start_fresh_kyc` | 1 | No KRA record for the PAN (`readiness.code = kyc_unavailable` or `unknown`) | Route to the KYC Form flow with `type=fresh` (Section 3). |
| `start_modify_kyc` | 1 | KRA has a record but it's incomplete (`readiness.code = kyc_incomplete` — Aadhaar not linked / email / phone missing) | Route to the KYC Form flow with `type=modify` (Section 3). **NEW (2026-06-08)** — replaces the old `fix_at_kra` dead-end for this code. |
| `fix_at_kra` | 1 | *(Legacy / backward-compat)* older cached rows may still carry this. Frontend keeps a passive card for it; readiness no longer emits it for `kyc_incomplete`. | Show external CTA, then `retry_readiness`. |
| `retry_readiness` | 1 | Readiness failed for a transient/data reason (`upstream_error`) | Show inline error + retry button calling `POST /readiness` again. |
| `fix_pan` | 2 | PAN trio mismatch — PAN itself wrong | Re-prompt PAN, then `retry_pan_validation`. |
| `fix_name` | 2 | Name doesn't match NSDL/PAN record | Re-prompt name. |
| `fix_dob` | 2 | DOB doesn't match | Re-prompt DOB. |
| `link_aadhaar_at_itd` | 2 | PAN-Aadhaar not linked | External CTA to the ITD portal; once linked, `retry_pan_validation`. |
| `retry_pan_validation` | 2 | Transient validation failure | Re-submit `/pan-validate`. |

### Decision tree

```
POST /readiness
  ├── status=accepted (in progress)                      → nextAction=poll          → GET /{id}/poll
  ├── readiness.status=verified                          → nextAction=run_pan_validation  (already KYC-compliant → kyc-form shows `already_kyc_valid`)
  ├── readiness.code=kyc_unavailable | unknown           → nextAction=start_fresh_kyc     (route to KYC Form, type=fresh)
  ├── readiness.code=kyc_incomplete                      → nextAction=start_modify_kyc    (route to KYC Form, type=modify)   ← NEW
  └── readiness.code=upstream_error                      → nextAction=retry_readiness

POST /pan-validate
  ├── status=in_progress                                 → nextAction=poll
  ├── all of pan/name/dateOfBirth .code=valid            → nextAction=run_bank_validation
  ├── pan.code != valid                                  → nextAction=fix_pan
  ├── name.code != valid                                 → nextAction=fix_name
  ├── dateOfBirth.code != valid                          → nextAction=fix_dob
  ├── pan_aadhaar_link.code=not_linked                   → nextAction=link_aadhaar_at_itd
  └── transient error                                    → nextAction=retry_pan_validation
```

**`kyc_incomplete` handling (corrected 2026-06-08):** per the Cybrilla Pre-Verification doc, `kyc_incomplete` means the KRA *has* a record for the PAN but it is missing details (Aadhaar not seeded, or email/phone absent). This is **repairable via a `type=modify` KYC Form** — so it now maps to `start_modify_kyc`, not the old `fix_at_kra` dead-end. Only `readiness.code = kyc_unavailable` (or `unknown`) — meaning no KRA record at all — requires a `type=fresh` KYC Form (`start_fresh_kyc`).

> ⚠️ **Picking the wrong KYC Form type locks the PAN for 7 days at Cybrilla** (`reason=ineligible_for_kyc_modification` when a `modify` hits a no-KRA PAN). The frontend `kyc-form.component.ts` therefore re-checks `my-status` right before POSTing and refuses to submit unless the readiness signal unambiguously permits the chosen type. `fresh` is allowed only for `kyc_unavailable`/`unknown`/`start_fresh_kyc`; `modify` is allowed for `verified`/`validated`/`registered`/`onhold` **and** `kyc_incomplete`/`start_modify_kyc`.

### Per-field code enums (per Cybrilla POA docs)

| Field | Codes | Notes |
|-------|-------|-------|
| `readiness.code` | `ready`, `kyc_incomplete`, `kyc_hold`, `kyc_rejected`, `kyc_under_process`, `kyc_registered`, `kyc_validated`, `pan_invalid`, `pan_not_found`, `error` | Drives `nextAction` per the decision tree. |
| `pan.code` | `valid`, `invalid`, `not_found`, `error` | |
| `name.code` | `valid`, `invalid` (mismatch with NSDL), `error` | |
| `dateOfBirth.code` | `valid`, `invalid` (mismatch), `error` | |
| `pan_aadhaar_link.code` | `linked`, `not_linked`, `error` | When `not_linked`, the user must link at the ITD portal. |
| `bankAccounts[].code` | `valid`, `penny_drop_failed`, `name_mismatch`, `account_inactive`, `requires_manual_verification`, `error` | `requires_manual_verification` always fires for NRI account types (`nre_savings` / `nro_savings`) — see below. |

The full per-field response shape is the upstream Cybrilla shape verbatim, surfaced inside `result` (see `Models\PreVerification\` for the C# DTOs).

### NRI handling

- For NRI investors, **PAN is optional** at the readiness stage — `POST /readiness` accepts an empty `pan` and the readiness check is skipped (`nextAction=run_bank_validation` directly).
- Bank accounts of type `nre_savings` or `nro_savings` **always** come back with `bankAccounts[].code = requires_manual_verification` — penny-drop is not supported for non-resident accounts at the gateway level.
- These accounts require the user to upload a `bank_account_proof` file (cancelled cheque / passbook page) which is later attached to the KYC request. The frontend must render the upload control whenever it sees `requires_manual_verification` on an NRE/NRO account.

### Webhook events

Two new events feed the same dispatcher (Section 5 — `FinprimEventCatalog.cs` and `FinprimWebhookReceiverController.cs`):

- `pre_verification.accepted` — Cybrilla has accepted the submission; status moves to `in_progress`. UI should show "we are checking your details" and start polling (or wait for `completed`).
- `pre_verification.completed` — terminal state. Backend refreshes `tbl_Fintech_PreVerification` and recomputes `nextAction`; frontend should re-call `GET /api/pre-verifications/my-status` (or rely on the same DB-first navigation pattern documented in Section 5).

Both events are registered by `POST /api/webhooks/bootstrap` along with the existing catalog.

### Resume flow (how the frontend recovers state)

The frontend must **never assume** it is starting fresh. On entering the onboarding shell it always calls:

```
GET /api/pre-verifications/my-status   →   { id, stage, status, nextAction, result }
```

`my-status` returns the latest persisted record for the authenticated user (joined from `tbl_Fintech_PreVerification`). The frontend then routes purely on `nextAction`:

| Returned `nextAction` | Frontend action on resume |
|-----------------------|---------------------------|
| `null` / record absent | Treat as new user — show PAN entry screen, then `POST /readiness`. |
| `poll` | The previous session was mid-flight — open the same stage's "we are checking your details" UI and call `GET /{id}/poll`. |
| `run_pan_validation` | Skip PAN-entry; jump straight into the Name + DOB form. |
| `run_bank_validation` | Skip stages 1 & 2; route directly to the bank-add screen. |
| `fix_*` / `retry_*` | Route to the corresponding remediation screen with the previous payload pre-filled. |
| `start_fresh_kyc` | Leave pre-verification; route into the KYC Form flow with `type=fresh` (Section 3). |
| `start_modify_kyc` | Leave pre-verification; route into the KYC Form flow with `type=modify` (Section 3). KRA record exists but is incomplete. |

Because every state transition is persisted server-side, **the user can close the app and resume from any device** — no client-side state is required. The same pattern is used for KYC (`GET /api/kyc/forms/my-status`) and the post-KYC onboarding steps incl. MFIA (`GET /api/onboarding/status`) — pre-verification just extends it. (There is **no** `/mf/accounts/my-status` route; MFIA status is a flag on `/api/onboarding/status`.)

---

## 14.2 Fresh KYC — PAN with NO KRA record (app-team walkthrough)

This is the **complete path** when a PAN has no KYC anywhere (readiness `code = kyc_unavailable`/`unknown` → `nextAction = start_fresh_kyc`). It reflects the **latest** behavior (2026-06-17). Use this section as the contract for the Android/iOS app.

### Test PANs (Cybrilla sandbox) — patterns the app team must use

| Goal | PAN pattern | Readiness result | Use for |
|------|-------------|------------------|---------|
| **Fresh KYC** (no KRA record) | **`...PX3753X`** (e.g. `AAHPX3753H`, `ACGPX3753G`) | `kyc_unavailable` → `start_fresh_kyc` | Testing the fresh-KYC form lifecycle (under_review → fill_fields → esign → submitted). |
| **Investable** (verified KRA) | **`...PX3751X`** | `verified` → purchase-ready | Testing the actual **invest / order** path end-to-end. |

> ⚠️ **Critical:** the `...3753...` simulator PANs **complete the KYC form but never become purchase-ready** — readiness stays `failed`/`kyc_unavailable`, so the invest gate (correctly) blocks them forever. To reach the invest screen and place an order, onboard with a **`...3751...`** PAN. Each PAN gets a KYC record on first use — use one per clean test.

### Step-by-step

**1. Prefill (optional) — `GET /api/onboarding/user-details`**
Before showing the Verify-PAN screen, the app SHOULD call this. If the user already has an **approved app-side KYC**, it returns their PAN + DOB so the screen pre-fills (editable). If none, PAN/DOB come back `null` → show a blank manual-entry form.
```
GET /api/onboarding/user-details
→ { "email", "name", "mobile", "pan": "ABCDE1234F"|null, "dateOfBirth": "yyyy-MM-dd"|null }
```
Backed by SP `Get_User_Details_For_Fintech`. Prefill writes into the same PAN/DOB fields used by the rest of the flow — **one field set, no duplication.**

**2. Readiness — `POST /api/pre-verifications/readiness`** `{ pan }`
- `kyc_unavailable` / `unknown` → `nextAction = start_fresh_kyc` → go to step 3.
- (`verified` → already KYC-compliant; `kyc_incomplete` → `start_modify_kyc`; see §14.1.)
- **Duplicate-PAN block (NEW 2026-06-17):** before any Cybrilla call, readiness checks whether this PAN already belongs to a **different** app user. If it does → **HTTP 409** and the flow stops here. See **§14.2.5** for the exact contract. (Same user re-entering their own PAN is fine — resume.)

**3. Collect DOB, then create the KYC Form — `POST /api/kyc/forms`** `{ type: "fresh", pan, name, date_of_birth }`
- The app MUST have a **DOB** before creating the form. A fresh-KYC readiness response carries **no DOB** (no KRA record), so if DOB is empty, collect it first (the web app routes to a Personal Details step to do this). Creating a fresh form with an empty DOB gets stuck at `fill_fields` forever.
- Form lifecycle: `under_review → created → fill_fields → awaiting_esign → awaiting_submission → submitted`. Terminal: `failed` (reason populated) / `expired` (7-day inactivity).

**4. Poll / resume — `GET /api/kyc/forms/{id}` (or `my-status`)**
Drive the UI off the form `status`. The web app auto-polls while `under_review`.

**5. Fill the remaining fields (`fill_fields` stage)**
Cybrilla's `requirements.fields_needed` lists what's still required: `email_address, phone_number, gender, marital_status, father_name, spouse_name, occupation_type, place_of_birth, income_slab, pep_details, aadhaar_number(last4)` + constants (`residential_status=resident, nationality=in, citizenship=[in], tax_residency_other_than_india=false`) + `geolocation`.

  - **Auto-fill + silent auto-submit (NEW):** the app SHOULD prefill these from what we already have (`GET /api/onboarding/user-details` + the KYC form's own `rawResponseJson` + contact-verified status). **If ALL required fields are present and geolocation is captured, PATCH automatically and advance — don't show the form.** Only render the form for the fields that are genuinely missing. (Web does this; native app should mirror it.)
  - **PATCH — `PATCH /api/kyc/forms`** `{ id, ...fields, geolocation }`. `spouse_name` only when `marital_status=married`; `father_name` otherwise.
  - **First-save consistency (NEW):** Cybrilla's `fields_needed` is **eventually consistent** — the GET right after a PATCH may still echo the pre-PATCH list even though the data WAS saved (the old "first save says still-needs-everything, second click works" bug). After PATCH, **re-fetch the form 1–2× with a ~1.5 s gap before deciding it still needs fields.** Only show "still requires …" if it persists after the retries.

**6. eSign (`awaiting_esign`)** — open the eSign URL; on return, poll until `submitted`.

**7. Submitted → onboarding continues** — investor profile → bank → MFIA → bank penny-drop. (Sections 3–13.)

**8. Invest gate (fail-CLOSED — NEW)**
The invest screen shows ONLY when **all** of: investor profile created, bank linked, bank `verificationStatus = completed`, MFIA created, **AND readiness is explicitly `verified`**. A `null`/empty/`failed` readiness now **blocks** (previously `null` slipped through — a fail-open bug). Blocked states the app should render:
  - No profile/bank/verify/MFIA → "Complete Setup First".
  - No KYC at all (readiness `null`) → "Complete your KYC first → Start KYC".
  - Onboarded but readiness `failed`/`pending` → "KYC not purchase-ready yet" + Refresh (this is the `...3753...` simulator case).

> **Note for the app team:** the frontend gate is **defense-in-depth only**. The backend `POST /api/mf/orders/purchases` does not yet pre-check readiness — it relies on Cybrilla rejecting non-ready orders (`non_kyc_compliant_investor_error`). So a non-KYC user calling the API directly is still rejected by FP, but with a less-clean error.

---

### 14.2.1 Full API sequence — every call, request & response (fresh KYC)

All wrapper responses use the envelope `{ statusCode, message, result }`. Auth = the logged-in user's Bearer JWT (no userId in any body). Wire format to Cybrilla is **snake_case**.

| # | Step | Method + Endpoint | Request body | Key response fields | Drives |
|---|------|-------------------|--------------|---------------------|--------|
| 0 | Prefill identity | `GET /api/onboarding/user-details` | — | `{ email, name, mobile, pan|null, dateOfBirth|null }` | PAN+DOB prefill (editable) |
| 0b | Prefill contact | `GET /api/onboarding/contact/verified-status` | — | `{ email, emailVerified, isd, number, phoneVerified }` | email/mobile prefill + **lock when verified** |
| 1 | Readiness | `POST /api/pre-verifications/readiness` | `{ "pan": "ABCDE1234F", "isWeb": 1 }` | `result.readiness.{status,code}`, `result.nextAction` | route decision |
| 1d | *(implicit)* Duplicate-PAN guard inside readiness | — | **409** `message` if PAN owned by another user (see §14.2.5) | STOP — block onboarding |
| 1p | Readiness poll | `GET /api/pre-verifications/{pvId}/poll` | — | same as readiness | while `nextAction=poll` |
| 2 | Create KYC form | `POST /api/kyc/forms` | `{ "type":"fresh", "pan", "name", "date_of_birth":"yyyy-MM-dd" }` | `result.id` (kycf_xxx), `result.status` | form lifecycle starts |
| 3 | Poll / resume form | `GET /api/kyc/forms/{id}` **or** `GET /api/kyc/forms/my-status` | — | `kycForm.status`, `kycForm.fieldsNeededCsv`, `kycForm.rawResponseJson`, `nextAction` | UI state |
| 4 | Contact OTP (email) | `POST /api/onboarding/contact/email/send-otp` `{ "value": email }` then `POST .../email/verify-otp` `{ value, otp }` | — | sandbox returns `sandboxOtp` | verifies email |
| 4b | Contact OTP (mobile) | `POST .../mobile/send-otp` / `.../mobile/verify-otp` | `{ value, otp }` | `sandboxOtp` (sandbox) | verifies mobile |
| 5 | Fill remaining fields | `PATCH /api/kyc/forms` | see payload below | `kycForm.status`, `fieldsNeededCsv` | `fill_fields` → `awaiting_esign` |
| 6 | Signature (if needed) | `POST /api/kyc/forms/{id}/signature` (multipart `file`) | file | `signatureProvided=true` | |
| 7 | eSign | open `kycForm.esign_details.esign_url` → poll | — | `esignStatus=successful`, `status=submitted` | terminal |

**PATCH `/api/kyc/forms` body (step 5) — exact shape:**
```json
{
  "id": "kycf_xxxxxxxx",
  "email_address": "user@example.com",
  "phone_number": { "isd": "+91", "number": "9876543210" },
  "gender": "male",
  "marital_status": "married",
  "spouse_name": "Jane Doe",          // ONLY when marital_status = married
  "father_name": "John Sr",           // ONLY when marital_status != married
  "occupation_type": "private_sector_service",
  "income_slab": "above_1lakh_upto_5lakh",
  "aadhaar_number": "1234",            // LAST 4 digits only
  "pep_details": "not_applicable",
  "place_of_birth": "Mumbai",          // a real place, NEVER a 2-letter country code
  "country_of_birth": "in",
  "nationality_country": "in",
  "residential_status": "resident",
  "citizenship_countries": ["in"],
  "tax_residency_other_than_india": false,
  "geolocation": { "latitude": 19.07, "longitude": 72.87 }
}
```
- Constants (`residential_status / country_of_birth / nationality_country / citizenship_countries / tax_residency_other_than_india`) are sent by the wrapper automatically.
- `geolocation` is **mandatory** — capture it via the explicit "Allow Location" action BEFORE Save (see navigation fixes). FP rejects the form without it.

**Step 6 — Signature upload constraints (ENFORCE ON APP; backend also enforces) — 2026-07-06**
`POST /api/kyc/forms/{id}/signature` (multipart, field name `file`).
- **Supported formats: `png`, `jpg`, `jpeg`, `pdf`** (Cybrilla limit).
- **Max size: 5 MB.**
- Once uploaded successfully, the form's **`signature_provided` becomes `true`** (mirrored
  as `signatureProvided` on `GET /kyc/forms/my-status`), and the stage advances to eSign.
- Validate BOTH on the app (before upload) and rely on the backend as the backstop:
  - Backend returns **400** `"Unsupported file format. Supported formats: png, jpg, jpeg, pdf."`
    for a bad format, **400** `"File exceeds 5 MB limit."` for oversize, **400**
    `"Signature file is required."` for an empty/missing file.
  - **App must show these validation errors on the upload screen** (a bad file must NOT
    fail silently). Also show the "Supported formats / max 5 MB" hint up-front.
- If the upload seems stuck (uploaded but stage didn't advance), re-pull status via
  `GET /api/kyc/forms/my-status` (web has a "Refresh status" button on this stage).
- **Retry on failure:** the upload is fully retryable — a failed `POST .../signature` does
  NOT consume or change the form (it stays in `upload_signature`). The app should keep the
  selected file, show the error, and let the user tap Upload again (retry same file) OR pick
  a new file. Web relabels the button "Retry upload →" after a failure. Re-POST the same
  endpoint; no reset/new-form needed.

**Step 7 — eSign: what to open, what to POLL, and when to proceed (app team) — 2026-07-06**
This is the FINAL step. eSign is Aadhaar-OTP-based via NSDL, done in a browser/WebView.
- **Open** `kycForm.esign_details.esign_url` (surfaced by the wrapper as `esignUrl` on
  `GET /kyc/forms/my-status`). If `esignUrl` is null, the form flipped to `awaiting_esign`
  but the URL isn't populated yet (eventually consistent) → **poll `my-status`** until it
  appears, then open it. (Web shows "Preparing…" + a refresh button for this.)
- **After the user finishes eSign in the WebView, DO NOT wait on the NSDL/Cybrilla webhook.**
  Close the WebView on the deep-link return, then **POLL** `GET /api/kyc/forms/my-status`.
- **Field to key off — `nextAction` / `kycForm.status` / `esignStatus`:**

  | Poll result | Meaning | App action |
  |---|---|---|
  | `esignStatus = successful` AND/OR `kycForm.status = submitted` (nextAction `done`/`wait`) | eSign done, KYC submitted to KRA | **PROCEED** → success screen → Investor Profile. TERMINAL. |
  | `nextAction = esign` / status still `awaiting_esign` | not signed yet | keep the eSign screen; let the user Open eSign / Refresh. |
  | `esignStatus = failed` OR `status = failed` | eSign failed | show the reason, **STOP** — let the user retry via Open eSign (re-open the same `esign_url`; re-poll). |

- **Poll cadence:** every ~2–3s for up to ~90s after return; if still `awaiting_esign`, show
  "still processing — you can come back and resume" (NOT an error — the form is resumable via
  `?from=resume` / `my-status`). Web has a **"Refresh status"** button on this stage that does
  exactly this re-pull; the app should implement the same.
- **Retry:** eSign is retryable — re-open the same `esign_url` and re-poll. No new form needed.
- **Order of stages before eSign** (so the app knows what precedes it): personal + financial
  fields filled → Aadhaar via DigiLocker (`fetch_proof`) → signature uploaded
  (`signature_provided=true`) → THEN `awaiting_esign`. Poll `my-status.nextAction` after each
  step to know the next stage; never hardcode the order.

**Readiness response example (fresh PAN):**
```json
{ "statusCode":200, "result": {
   "fpPreVerificationId":"pv_xxx", "stage":"readiness", "overallStatus":"completed",
   "readiness": { "status":"failed", "code":"kyc_unavailable" },   // or code:"unknown"
   "pan": { "value":"ABCDE1234F" }, "name": {}, "dateOfBirth": {},
   "nextAction":"start_fresh_kyc" } }
```

### 14.2.2 Auto-populate rules (what fills, and when it's read-only)

| Field | Source (priority order) | Read-only when |
|-------|--------------------------|----------------|
| **PAN** | `user-details.pan` → else readiness `pan.value` | when prefilled from an approved app KYC |
| **DOB** | `user-details.dateOfBirth` → else readiness `dateOfBirth.value` | when prefilled |

> ⚠️ **PAN is the anchor — never prefill DOB without a PAN.** `GET /api/onboarding/user-details` returns the user's approved app-KYC identity, but **`pan` may be absent** (the SP returns it only when `tbl_User_AUG.PanNumber` exists; null fields are omitted from the JSON, so a response can legitimately be `{ email, name, mobile, dateOfBirth }` with **no `pan`**). In that case **do NOT prefill DOB on its own** — PAN+DOB must travel together for the KRA match, and a lone prefilled DOB is misleading. Rule: **only prefill DOB if `pan` is present; if `pan` is missing/empty, leave BOTH PAN and DOB blank for manual entry.** (Web does this in `prefillPanFromAppKyc`; the app team must mirror it.)
| **Email** | **verified** `contact/verified-status.email` **wins**, else `user-details.email`/login email | `emailVerified=true` → **read-only + ✓ Verified, OTP skipped** |
| **Mobile** | **verified** `contact/verified-status.number` **wins**, else login mobile | `phoneVerified=true` → **read-only + ✓ Verified** |
| Gender / Marital / Father / Occupation / Income / Place-of-birth | `investor-profiles/my-status` (`kyc*` fields) + form `rawResponseJson` | editable (user may correct) |

**Critical rule — a server-VERIFIED email/mobile OVERWRITES any other prefill.** Do NOT "fill only if blank": the login email (e.g. `test25@…`) gets seeded first, so a blank-only fill leaves the field on the login email while the verified MF email (`irfan@…`) is ignored → wrongly shows "Send OTP" for an already-verified contact. Always let the verified value win, then lock it.

> Once email/mobile are OTP-verified for MF, they are stored verified server-side and **auto-populate + lock everywhere** (KYC Personal Details, /kyc-form fill_fields, and the onboarding "email for MF statements" step) — the same `contact/verified-status` is the single source.

### 14.2.3 Navigation problems we hit — and the correct behavior (IMPORTANT for the app team)

These are real bugs we fixed on web; the native app must implement the **fixed** behavior, not the naive one.

1. **Resume loops back to "Start fresh KYC".**
   *Cause:* `/kyc` routed purely on `pre-verifications/my-status` `nextAction`, which stays `start_fresh_kyc` forever for a no-KRA PAN — even after a KYC form was created/submitted. Reloading restarted fresh.
   *Fix / correct behavior:* **before** acting on `start_fresh_kyc` / `start_modify_kyc`, call **`GET /api/kyc/forms/my-status`**. If `kycForm.fpKycFormId` exists **AND `kycForm.pan` matches the PAN being onboarded now**, **resume the KYC form flow** (show its real `status`); otherwise start fresh. ⚠️ See item 9 — `my-status` returns the latest form by **UserId only**, so you MUST compare PANs before resuming.

2. **DOB empty → form stuck at `fill_fields` forever.**
   *Cause:* fresh readiness carries no DOB; creating the form with empty DOB leaves `date_of_birth` permanently in `fields_needed`.
   *Fix:* DOB is **mandatory before `POST /api/kyc/forms`**. If absent, collect it (Personal Details) first; carry `pan`+`dob` into the form route.

3. **First "Save" says "still needs ALL fields"; second click works.**
   *Cause:* Cybrilla's `fields_needed` is **eventually consistent** — the GET right after the PATCH echoes the pre-PATCH list though the data WAS saved.
   *Fix:* after PATCH, if still `fill_fields` with `fields_needed`, **re-fetch 1–2× with ~1.5 s gaps** before surfacing "still requires …". Don't error on the first read.

4. **"First Save click does nothing, second works."**
   *Cause:* the geolocation permission prompt fired during Save (hidden behind the form, no feedback).
   *Fix:* capture geolocation via an **explicit "Allow Location" button BEFORE Save**; block Save until `capturedGeo` is set; show a spinner immediately on click.

5. **Prefilled values don't appear until I click somewhere.**
   *Cause:* prefill writes plain form fields inside an async `subscribe`; in zoneless/OnPush change-detection the view doesn't repaint until another event.
   *Fix:* call **`ChangeDetectorRef.markForCheck()`** after every async prefill write (PAN/DOB, email/mobile, gender/marital/etc.).

6. **"Complete Setup First" shows with a blank Missing list / loops.**
   *Cause:* the invest gate required readiness `verified`, but readiness isn't one of the listed setup items → blank list; and `null` readiness was treated as OK (fail-open).
   *Fix:* **fail-closed** (`readiness must === 'verified'`) + render the real reason: no-KYC → "Complete your KYC first → Start KYC"; onboarded-but-not-ready → "KYC not purchase-ready yet" + Refresh.

7. **Error message won't clear on Continue / Skip (e.g. demat step).**
   *Cause:* a single shared `stepError` wasn't reset on step navigation.
   *Fix:* clear `stepError` on Skip and on successful submit (+ `markForCheck`); ideally reset it on every step change so no step inherits another's error.

8. **Upstream 502s showed a generic message; field errors hidden.**
   *Cause:* controllers swallowed the upstream exception behind a generic 502.
   *Fix:* parse Cybrilla's `error.errors[]` → return a **clean per-field message** to the screen (e.g. `dp_id: not a valid dp id; client_id: not a valid client id`), while the **full** detail (step + status + body + sent payload) goes to the error **email/log** only.

9. **A DIFFERENT PAN's old form hijacks the screen (e.g. shows "KYC already exists — switch to Modify" for a brand-new PAN).** ⚠️ MUST-FIX for the app team.
   *Cause:* **`GET /api/kyc/forms/my-status` returns the user's latest `kyc_form` by `UserId` ONLY — it is NOT scoped to the PAN currently being onboarded.** If the user previously attempted KYC with PAN **A** (e.g. an old `failed`/`ineligible_for_fresh_kyc` form for `DEFPZ5678R`) and now onboards a NEW PAN **B** (`ATKPX3753K`), `my-status` still returns PAN A's form. Routing off its `status` then renders PAN A's failed/modify screen for PAN B — even though `pre-verifications/my-status` for PAN B correctly says `kyc_unavailable → start_fresh_kyc`. (Observed 2026-06-18: readiness `investorIdentifier=ATKPX3753K` but `my-status.kycForm.pan=DEFPZ5678R`.)
   *Fix / correct behavior:* **PAN-MATCH GUARD** — whenever you read `kyc/forms/my-status`, compare `kycForm.pan` (uppercased/trimmed) with the PAN being onboarded now (from readiness `investorIdentifier` / the entered PAN). **If they differ, IGNORE the returned form and start fresh for the current PAN** — do NOT render its `status`/`failed`/`ineligible` screen. Only treat a form as the user's active KYC when its PAN matches. (Web fix: `kyc-form.component.ts` — guard added at both the form-first peek and the Step-2 resume read.) The `formType` for the new PAN still comes from that PAN's readiness (`kyc_unavailable → fresh`).

### 14.2.4 Correct calling sequence (happy path, fresh KYC)

```
GET  /api/onboarding/user-details            → prefill PAN/DOB (editable)
GET  /api/onboarding/contact/verified-status → prefill+lock email/mobile if verified
POST /api/pre-verifications/readiness {pan}   → nextAction=start_fresh_kyc
        └─ (resume guard) GET /api/kyc/forms/my-status → if a form exists, RESUME it; else:
[collect DOB if missing]
POST /api/kyc/forms {type:fresh, pan, name, date_of_birth}
GET  /api/kyc/forms/my-status (poll)          → status=fill_fields, fields_needed=[…]
[verify email/mobile OTP if not already verified]
PATCH /api/kyc/forms {id, …all fields…, geolocation}
        └─ if still fill_fields: re-poll 1–2× (eventual consistency) before erroring
[POST /api/kyc/forms/{id}/signature if signatureProvided=false]
open esign_url → poll GET /api/kyc/forms/my-status until status=submitted
→ continue onboarding: investor profile → bank → MFIA → bank penny-drop
→ invest gate opens only when readiness=verified (…3751… PAN; …3753… never flips)
```

---

### 14.2.5 Duplicate-PAN guard — one PAN = one app user (NEW 2026-06-17)

**Rule.** A PAN is a person's tax identity, so it must belong to **at most one** Islamicly app account. Before MF onboarding starts with a PAN, the backend checks whether that PAN is already linked to a **different** user. If it is → the request is **blocked with HTTP 409** and onboarding does not start. The **same user** re-entering **their own** PAN is always allowed (so resume works).

**Where it is enforced.** Server-side, inside `POST /api/pre-verifications/readiness`, **before any Cybrilla call** — so the UI cannot bypass it. It applies to **both** scenarios (PAN auto-populated from app KYC **or** typed manually) and to **both** with/without-KRA paths, because every onboarding goes through readiness first. **Enforced in sandbox too.**

**Why readiness is the choke point.** Readiness is the first PAN-bearing call in every flow (fresh, modify, and verified-KRA). Guarding it there means no Cybrilla `pre_verification` / `kyc_form` / investor_profile record is ever created for a PAN that already belongs to someone else.

**Contract:**

| Aspect | Value |
|--------|-------|
| Endpoint | `POST /api/pre-verifications/readiness` |
| Trigger | PAN already present (under a **different** `UserId`) in any of: `tbl_Fintech_Pre_Verification` (`PanValue`/`InvestorIdentifier`), `tbl_Fintech_Kyc_Form` (`Pan`), `tbl_Fintech_Investor_Profile` (`PAN`) |
| HTTP status | **409 Conflict** |
| Body | `{ "statusCode": 409, "message": "This PAN is already registered. Please use your own PAN, or contact support if this is an error.", "result": null }` |
| Same-user re-entry | **Allowed** — own PAN is not a conflict (`UserId <> @UserId` excludes self) |

**App-team handling.** On a **409** from `/readiness`, do **NOT** retry, do **NOT** advance to the KYC form. Show the `message` verbatim on the PAN-entry screen (inline error) and keep the user on that screen. Offer "Use a different PAN" (clear + re-enter) and a "Contact support" link. Treat 409 distinctly from 502 (transient upstream — that one is retryable).

**Backend wiring:**
- SP `dbo.Usp_Check_Fintech_Pan_Owner (@Pan, @UserId)` → returns `OwnerUserId` (the other user, or NULL) + `InUseByOther` bit. PAN compared `UPPER(LTRIM(RTRIM()))`-normalised across all three identity tables.
- `IPreVerificationDbService.GetPanOwnerOtherThanAsync(pan, userId)` calls the SP; readiness blocks with 409 when it returns non-null. The conflicting `UserId` is written to the clean log (`BLOCKED:PanOwnedByUser=<id>`) — never returned to the client.

**Sandbox caveat.** Because the block is enforced everywhere, **shared `…3753…` / `…3751…` test PANs will collide across test accounts** once one account has used them. Each tester needs a **unique** PAN, or clear that PAN's prior rows from the three identity tables. (This is the deliberate, chosen trade-off for enforce-everywhere.)

```
POST /api/pre-verifications/readiness {pan}
   ├── PAN owned by another user  → 409  "This PAN is already registered…"   (STOP, show inline, no retry)
   └── PAN free / own PAN         → proceed to Cybrilla readiness (existing flow)
```

---

## 14.3 KYC — consolidated latest (single source of truth)

> This is the **authoritative KYC summary** for the app team. It supersedes the old §3.1–§3.11 KYC-check/KYC-request wizard. The endpoint-by-endpoint walkthrough is in **§14.2**; this section collects the behaviors and **every issue we hit on web and fixed**, so the native app implements the fixed behavior directly.

### 14.3.1 The flow in one line
`GET /onboarding/user-details` (prefill PAN/DOB) + `GET /onboarding/contact/verified-status` (prefill+lock email/mobile) → `POST /pre-verifications/readiness {pan}` → (resume guard: `GET /kyc/forms/my-status`, **PAN-match**) → `POST /kyc/forms {type:fresh|modify, pan, name, date_of_birth}` → `fill_fields` PATCH (with geolocation) → eSign → `submitted` → onboarding (profile → address → contact → bank → FATCA/demat → nominee → MFIA).

### 14.3.2 Fresh vs Verified PAN — what differs

| | **Fresh KYC** (no KRA record) | **Verified PAN** (KRA-compliant) |
|---|---|---|
| Readiness `code` | `kyc_unavailable` / `unknown` | `verified` |
| `nextAction` | `start_fresh_kyc` | proceed straight to onboarding (no KYC form) |
| DOB | **must be collected** (readiness carries none) — mandatory before `POST /kyc/forms` | comes from KRA |
| KYC form | created, goes through `fill_fields → esign → submitted` | **skipped entirely** |
| Sandbox test PAN | `…PX3753X` (form completes but **never purchase-ready**) | `…PX3751X` (reaches invest path) |
| Invest gate | opens only if readiness later flips to `verified` (3753 never does) | opens once profile+bank+verify+MFIA done |

> The single most important sandbox gotcha: **`…3753…` PANs complete the KYC form but stay `failed`/`kyc_unavailable` forever** — use **`…3751…`** to test investing. (§14.2 table.)

#### 14.3.2a ⚠️ Verified PAN — which fields to collect WHERE (do NOT re-ask everything at readiness)
For a **verified (KRA-compliant) PAN there is NO KYC form**, so DON'T render the big fill-fields form. The web Personal-Details step for a verified PAN collects **only the minimum needed to confirm identity + reach contact**, and the rest is collected on the **later** Investor-Profile / FATCA screens. Mirror this split exactly — don't collect Marital / Father / Mother / Aadhaar-last4 / Nationality / Country-of-birth / Citizenship / Place-of-birth / Tax-residency at the readiness/personal step.

| Step | Collect (verified PAN) | Sent to |
|---|---|---|
| **Personal Details** (after readiness `verified`) | **Full Name** + **DOB** (prefilled, editable) · **Mobile + Email — OTP-verified** | `POST /api/pre-verifications/pan-validate {pan,name,dateOfBirth}`; mobile/email via the contact OTP endpoints (stored, reused for folio_defaults/consent). Gender is shown/prefilled but only sent later. |
| **Investor Profile** | Gender · Occupation · Source of wealth · Income slab · **Place of Birth** (required — KRA doesn't supply it; mandatory for FATCA/purchase) | `POST /api/investor-profiles/my-profile` |
| **FATCA / Tax residency** | PEP · India + up to 3 foreign tax residencies | `POST /api/onboarding/fatca` |

So on the **verified** path the readiness/personal screen is essentially **"confirm Name+DOB, verify Mobile+Email"** — everything else moves to the profile + FATCA steps. (The fuller field list in §14.3.4 below applies to the **fresh-KYC `fill_fields` form only**, where Cybrilla itself demands those fields.)

### 14.3.3 OTP-verified email & mobile (verify-once)
- Email and mobile must be **OTP-verified** before they save — in onboarding AND in the `/kyc/forms` fill-fields step. Endpoints: `POST /api/onboarding/contact/{email|mobile}/send-otp {value}` → `POST .../verify-otp {value, otp}`. **Sandbox returns the code in `sandboxOtp`**; prod sends email via SendGrid (SMS is a stub — wire a real provider for prod).
- **Verify-once:** a value already verified server-side **and unchanged** skips OTP. Entering a **different** value forces a fresh OTP and clears the verified flag.
- A **server-verified** value (`verified-status`) **wins over any other prefill and locks the field** with a "✓ Verified" badge (see auto-populate rule below). Save is blocked until both are verified.
- Source of truth = `GET /api/onboarding/contact/verified-status` (reads a consumed `tbl_Fintech_Contact_Otp` row OR a verified phone/email row). The same status auto-fills + locks email/mobile in **all three** places: KYC Personal Details, `/kyc-form` fill-fields, and the onboarding "email for MF statements" step.

### 14.3.4 Auto-populate (PAN / DOB / email / mobile / personal)
| Field | Source (priority) | Becomes read-only when |
|---|---|---|
| **PAN** | `user-details.pan` → else readiness `pan.value` | prefilled from an approved app KYC |
| **DOB** | `user-details.dateOfBirth` → else readiness `dateOfBirth.value` | prefilled — **but only if PAN is present** (PAN is the anchor; never prefill DOB alone) |
| **Email** | **verified** `verified-status.email` wins → else `user-details.email` / login email | `emailVerified=true` → read-only + ✓ Verified, OTP skipped |
| **Mobile** | **verified** `verified-status.number` wins → else login mobile | `phoneVerified=true` → read-only + ✓ Verified |
| Gender / Marital / Father / Occupation / Income / Place-of-birth | `investor-profiles/my-status` (`kyc*`) + form `rawResponseJson` | editable (user may correct) |

- **Silent auto-submit:** if every `fill_fields` requirement is already known AND geolocation is captured, PATCH automatically and advance — don't show the form. Render it only for genuinely-missing fields.
- After any async prefill write, call `ChangeDetectorRef.markForCheck()` (zoneless/OnPush won't repaint otherwise).

### 14.3.5 Every issue we hit on web — and the fixed behavior (mirror these)
1. **Resume loops back to "Start fresh KYC"** → before acting on `start_fresh_kyc`, peek `kyc/forms/my-status`; if a form exists for **this PAN**, resume it. (§14.2.3-1)
2. **A DIFFERENT PAN's old form hijacks the screen** → `kyc/forms/my-status` is by `UserId` only. **PAN-MATCH GUARD:** ignore the returned form unless `kycForm.pan` equals the PAN being onboarded. (§14.2.3-9 — MUST-FIX)
3. **DOB empty → form stuck at `fill_fields` forever** → DOB mandatory before `POST /kyc/forms`. (§14.2.3-2)
4. **First Save says "still needs ALL fields", second works** → Cybrilla `fields_needed` is eventually consistent; re-fetch 1–2× with ~1.5 s gaps before erroring. (§14.2.3-3)
5. **First Save click does nothing** → geolocation prompt fired during Save; capture geo via an explicit "Allow Location" button BEFORE Save, block Save until captured, show spinner on click. (§14.2.3-4)
6. **Prefill doesn't show until you tap** → `markForCheck()` after every async prefill. (§14.2.3-5)
7. **Verified MF email ignored, shows "Send OTP"** → let the **verified** value win and lock; never "fill only if blank". (§14.2.2)
8. **Upstream 502 hid the real field error** → parse Cybrilla `error.errors[]` → clean per-field message to the screen (e.g. `spouse_name: not a valid spouse_name`); full detail goes to log/email only; user-correctable 4xx → HTTP 400 (no alert), 5xx → 502 (+alert). (§14.2.3-8)
9. **"NA"/placeholder names rejected by FP** → guard NA/N/A/nil/-/. and <2-char father/spouse names before the round-trip.
10. **Invest gate fail-open on `null` readiness** → fail-CLOSED: invest shows only when profile+bank+`verificationStatus=completed`+MFIA **AND readiness === `verified`**. (§14.2.3-6, §14.2-8)
11. **Duplicate PAN** → readiness returns **409** if the PAN belongs to another user; show inline, no retry, no advance. (§14.2.5)
12. **PEP enum mismatch** → kyc_form uses `pep|related_pep|no_exposure`; investor-profile uses `not_applicable|pep_exposed|pep_related`; map kyc→profile when carrying over. (changelog 2026-06-18; enum appendix)
13. **Token 401 mid-flight** → client force-refreshes the tenant token and retries once. (§5.6)
14. **`GET /api/kyc/forms/my-status` → 404 is NORMAL, not an error.** When the user has **no KYC form** (every **verified-KRA PAN**, and any user before a fresh-KYC form is created), this endpoint **intentionally returns `404 { statusCode:404, message:"No KYC form found for this user.", nextAction:"start_fresh_kyc" }`**. It is a *"no form exists yet"* signal with a structured body — **NOT a missing route and NOT a deploy gap.** **The client MUST catch this 404 and treat it as a fall-through**, then read `GET /api/pre-verifications/my-status`: if `readiness.status === 'verified'` → the investor is **already KYC-compliant**, skip the KYC form entirely and proceed to pan-validate → investor profile. (Web does exactly this — see the console flow: `form-first peek: no authoritative form (404) → readiness gate → readiness=verified → already KYC-compliant`.) **Do NOT surface this 404 as a user error.** A *bare* 404 with no JSON body would mean a missing route / stale deploy; a 404 **with** the `nextAction` body is the designed response.

---

## 14.4 Resume — picking up after the user leaves mid-flow

> **The reported problem:** users who abandon **after KYC**, **after the investor profile**, or **after the MF account** couldn't resume cleanly. (There is **no issue resuming the stages BEFORE the MF account is created** — those already route correctly; the gap was the post-KYC / post-profile / post-MFIA hops.) This section is the exact contract the app must implement. **All state is server-side — the user can resume from any device; never assume "start fresh".**

### 14.4.0 ⚠️ Canonical status-endpoint URLs (these match the C# routes EXACTLY)
**Copy these verbatim. The trap: most resources use `my-status`, but ONBOARDING uses plain `status`, and the KYC status is under `kyc/forms`, NOT `kyc`.**

| ✅ Correct route | Resource | ❌ Common wrong URL (→ 404) |
|---|---|---|
| `GET /api/pre-verifications/my-status` | PAN / readiness | — |
| `GET /api/kyc/forms/my-status` | KYC form (fresh KYC) | `~~/api/kyc/my-status~~` (no such route) |
| `GET /api/investor-profiles/my-status` | investor profile | — |
| `GET /api/banking/my-status` | bank + verification | — |
| `GET /api/onboarding/status` | **everything after KYC** (profile/address/contact/bank/FATCA/demat/nominee/MFIA flags) | `~~/api/onboarding/my-status~~` (no such route — it's `status`) |
| `GET /api/mf/accounts` | MFIA list (no `my-status`) | `~~/api/mf/accounts/my-status~~` (no such route) |

If any of these 404 on the **ngrok/test** server, it's a **stale deployed build** (e.g. `kyc/forms/my-status` is newer than what's deployed) — rebuild + redeploy FinprimApi; the routes themselves are all present in source.

### 14.4.1 The three status calls (always call on entry, in this order)
| Call | Tells you | Key fields |
|---|---|---|
| `GET /api/pre-verifications/my-status` | PAN / readiness stage | `nextAction`, `readiness.{status,code}`, `investorIdentifier` (the PAN) |
| `GET /api/kyc/forms/my-status` | KYC-form stage (**fresh KYC only**) | `kycForm.{fpKycFormId, pan, status}` — **compare `pan` to the onboarding PAN before trusting it** |
| `GET /api/onboarding/status` | everything after KYC | `investorProfileId`, `has{Address,Phone,Email,BankAccount,Fatca,Demat,Nominee,MfAccount}`, `bankAccountId`, `mfAccountId` |

### 14.4.2 Single resume decision (the exact web logic — mirror it)
Compute the resume step as the **first incomplete** in this fixed order. `bankVerified` = bank exists AND its verification status is `completed`.

```
1. KYC complete?  →  kycComplete = (investorProfileId present) OR kycStatus===true OR kycRequestStatus==='successful'
                     if NOT complete → route to /kyc (it resumes the form if one exists, else starts fresh)
2. investorProfileId missing?        → /investor-profile        (create profile)
3. !hasAddress                       → step 'address'
4. !hasPhone || !hasEmail            → step 'phone'  (combined contact screen; seed per-channel saved flags)
5. !hasBankAccount || !bankVerified  → step 'bank'   (if bank exists but unverified → show re-verify)
6. !hasFatca || (!hasDemat && !skippedDemat) → step 'demat' (FATCA mandatory, demat optional)
7. !hasNominee && !skippedNominee    → step 'nominee' (optional)
8. else                              → step 'mf-account'   → after MFIA exists → 'complete'
```

**Key rule (this fixed the kyc⇄onboarding loop):** an **existing `investorProfileId` IS proof that KYC is complete** — you cannot create an investor profile without verified KYC. So trust the profile first; do **not** bounce a profiled user back to `/kyc` just because the legacy `tbl_Fintech_KYC_Request` snapshot is empty/non-`successful` (it's empty for users verified via the readiness/POA path).

### 14.4.3 Resume by where they stopped

| User left… | What `my-status` shows | Resume to |
|---|---|---|
| **After KYC** (submitted/verified, no profile yet) | KYC complete, `investorProfileId = null` | **Investor Profile** screen (Name/DOB/PAN prefill is guaranteed present) |
| **After Investor Profile** (no address/contact/bank) | `investorProfileId` set, `hasAddress/hasPhone/hasEmail/hasBankAccount` false | first missing of **address → contact → bank** |
| **After bank added but not verified** | `hasBankAccount=true`, `bankVerified=false` | **Bank** screen with re-verify (penny-drop pending/failed) |
| **After bank verified, before MFIA** | `bankVerified=true`, `hasMfAccount=false`, FATCA/nominee pending | first missing of **FATCA/demat → nominee → MF account** |
| **After MF Account** (`hasMfAccount=true`) | all flags set, `mfAccountId` present | **Complete** → invest gate (opens only if readiness `verified`) |

### 14.4.4 Fresh vs Verified PAN on resume
- **Verified PAN:** there is **no KYC form** — `kyc/forms/my-status` returns nothing for this PAN. Resume goes straight to the onboarding decision (§14.4.2 from step 2). Don't route to `/kyc`.
- **Fresh PAN:** if KYC isn't yet `submitted`, resume the **KYC form** — but **only after the PAN-match guard** (§14.2.3-9): if `kycForm.pan ≠` the onboarding PAN, ignore that form and start fresh for the current PAN. Once `submitted`, treat KYC as complete and fall through to onboarding.

### 14.4.5 App-team checklist
- On every app open / onboarding entry: call the **three** status endpoints; never assume fresh.
- **Always PAN-match** `kyc/forms/my-status` before rendering its state.
- Treat **`investorProfileId` as the KYC-complete signal** to avoid the kyc⇄onboarding loop.
- A bank that exists but is **unverified is NOT done** — keep the user on the bank step.
- FATCA mandatory; demat & nominee optional (respect the user's "skip" choice via `skippedDemat`/`skippedNominee` client flags — these are client-side intents, re-shown if the app restarts unless persisted).
- The invest screen unlocks only when profile+bank+verify+MFIA done **AND** readiness `verified`.

---

## 14.5 Nominee — the "Add nominee details" screen

> Optional step in onboarding (between FATCA/demat and MF account). A user may **add nominees or opt out**. Endpoint: `POST /api/onboarding/nominee`; status comes back as `hasNominee`/`nomineeId` on `GET /api/onboarding/status`. Nominees live on the investor profile (`invp_xxx`) as FP **related parties**; allocation % is applied to the **MFIA folio_defaults** at MF-account creation (or re-pushed if the MFIA already exists).

### 14.5.1 Screen shape (two sections)
The web screen (`nomination-page.component`) has two parts the app should mirror:
- **Section A — Add to pool:** capture each candidate nominee — **Name, Date of Birth, Relationship**, optional **Nominee PAN**. If the DOB makes them a **minor (<18)**, the screen reveals the **guardian block** (name, relationship, PAN, email, mobile, address line1/city/state/postal) — all **required** for a minor.
- **Section B — Choose & allocate:** pick **up to 3** nominees from the pool and **split 100%** across them (allocation %). Or **Opt out** (no nominee).

### 14.5.2 Field contract (`POST /onboarding/nominee`)
| Field | Required | Notes |
|---|---|---|
| `profile` | auto | `invp_xxx`; backend fills from the user's profile if omitted |
| `name` | yes | nominee full name |
| `date_of_birth` | yes | FP derives minor-ness from this |
| `relationship` | yes | `son \| daughter \| spouse \| father \| mother \| sibling \| other` |
| `allocationPercentage` | yes (effective) | MFIA-side; selected nominees must total **100%** |
| `nomineePan` | optional (adult) | **nulled for a minor** (adult-only per FP) |
| `guardian*` (name, relationship, pan, email, isd, mobile, address line1/city/state/postal) | **required if minor** | only for <18 nominees |

### 14.5.3 Rules & behaviors (mirror these)
- **Minor detection is by DOB**, client + server: `<18` → guardian fields mandatory; the controller rejects (`400`) a minor without each required guardian field, and **nulls the nominee's own PAN**.
- **Allocation must total 100%** across the selected (≤3) nominees.
- **Immutable on edit:** name, relationship, DOB, and an already-set PAN are fixed on the registry — only the not-yet-set PAN/guardian-PAN and allocation can be changed afterward.
- **Allocation/PAN/guardian are NOT sent to FP `/v2/related_parties`** — only the FP-confirmed subset (name/dob/relationship) is. The extra fields are persisted locally and applied to the MFIA `folio_defaults` at attach time (`ReattachNomineesToMfiaAsync`). This is deliberate (FP guardian/PAN field names not yet confirmed → avoids a 502).
- **Opt out** is allowed → `hasNominee=false`; onboarding continues to MF account. (Web tracks an intentional skip via a client `skippedNominee` flag so resume doesn't re-prompt within the session.)

### 14.5.4 Resume
On `GET /api/onboarding/status`: `hasNominee=true` (+`nomineeId`) → step done; `false` and not intentionally skipped → show the nominee screen (per §14.4.2 step 7). If the MFIA already exists, adding/editing a nominee re-pushes the set to the MFIA folio_defaults automatically.

### 14.5.5 Nominee — API endpoints & backend rules (2026-07-06)

All under `api/onboarding` (`OnboardingController`).

| Method | Path | Purpose |
|---|---|---|
| POST   | `nominee` | Add a nominee (validates, guards, dedupes, forwards subset to FP `/v2/related_parties`, persists full row, re-attaches to MFIA). |
| POST   | `nominees/allocations` | Set/replace the chosen ≤3 nominees + allocation %; MUST total 100%. Persists %s AND re-attaches EXACTLY the chosen set to folio_defaults (deselected are dropped). |
| DELETE | `nominee/{nomineeId}` | **Soft-delete (hide)** — FP has no related-party delete, so hidden locally; won't show, won't count toward the 3-limit, stays hidden across FP re-sync. |
| POST   | `nominees/remove-all` | Hide all nominees + `ClearNomineesAsync(mfia)` → opt-out (`show_nomination_status`). |
| POST   | `nominees/sync` | Re-pull FP related-parties into local rows (respects hidden). |

**Backend validation & guards (`POST nominee`):**
- `name`, `relationship`, `date_of_birth` all required (400 each).
- **Minor (by DOB, `<18` — `IsMinorByDob`)** → guardian block mandatory: guardian
  **name, relationship, PAN, email, mobile, address line1 + postal code** each required
  (400). The nominee's own PAN is **nulled** for a minor (adult-only per FP).
- **Guardian ≠ Holder guard (NEW):** Cybrilla rejects a minor whose guardian name equals
  the account holder's ("Nominee guardian name should not be same as of the holder").
  Caught early → `400` "The guardian must be a different person from the account holder…".
- **Duplicate-skip (NEW):** if an identical nominee (same name+dob+relationship) already
  exists, returns `200 { id: <existingId> }` "Nominee already added." — prevents the
  observed 2–3× duplicate related_party rows created milliseconds apart.
- Only the **FP-confirmed subset** (name/dob/relationship) is sent to `/v2/related_parties`;
  allocation %, PAN, guardian, adult contact/address, identity-proof are persisted locally
  (via `UpsertNomineeAsync` — the **21-param** SP) and applied to the MFIA folio_defaults.
- `ReattachNomineesToMfiaAsync` pushes the visible set (+%s, normalised to 100) to
  `folio_defaults` — runs on add / allocation-change; no-ops if the MFIA doesn't exist yet
  (frontend attaches at MFIA creation).

**`POST nominees/allocations` — HARD failure on FP reject (NEW):** if the folio_defaults
PATCH is rejected/skipped, returns `502` (not a misleading `200 "saved"`), so the UI stops
and shows the real reason.

**Nominees are FOLIO-level, NOT per-order.** Neither SIP (`mf_purchase_plans`) nor lumpsum
(`mf_purchases`) create carries a nominee field. They live on MFIA `folio_defaults`
(`nominee1..3`, `nomineeN_allocation_percentage`, `nomineeN_identity_proof_type`,
`nomineeN_guardian_identity_proof_type`, `nominations_info_visibility`) via
`MfInvestmentAccountService.SetNomineesAsync` / `ClearNomineesAsync`.

**NEW finalize-time nomination guard (fixes "add guardian/nominee details" rejection):**
`sip/finalize` (§21.2d) verifies `nominations_info_visibility` is set
(`show_all_nominee_names` = has nominees, or `show_nomination_status` = opted out); if absent
→ `400 { reason:"nomination_required" }`. Never fabricates nominees. **App/web must handle
`nomination_required` by routing to the nomination screen.**

**SPs deployed (bodies in `Claude-db-queries.txt`):**
- `Usp_Upsert_Fintech_Investor_Nominee` — **21-param** version (was 12 → "Procedure has too
  many arguments specified").
- `Usp_List_Fintech_Investor_Nominee` filters `IsHidden = 0`;
  `Usp_Hide_Fintech_Investor_Nominee` soft-deletes → a removed nominee never reappears.

**Minor-nominee error** is surfaced **verbatim** from the provider/validation message (no
generic "A minor nominee needs guardian details" banner).

### 14.5.5 COMPLETE nominee flow — exact sequence, APIs, and the guardian-address field names (LATEST)

> Verified end-to-end against the web code (`nomination-page.component.ts`, `kyc.service.ts`) and backend (`OnboardingController.cs`, `CreateNomineeRequest`). Base = `environment.mfurl` (`…/api/`), JWT on all, envelope `{statusCode,message,result}`.

**Two-object model (must understand):** a **nominee = a Cybrilla `/v2/related_parties` object** (the person); the **nomination % + linkage lives on the MFIA `folio_defaults`** (`nominee1/2/3` + `_allocation_percentage`). The % is set **only when you attach (Step 3)**, NOT when you create the person (Step 1).

**Full endpoint table**

| Endpoint | Method | Frontend (kyc.service) | What it does |
|---|---|---|---|
| `/api/onboarding/status` | GET | `getOnboardingStatus()` | `investorProfileId` (invp_) + `mfAccountId` (mfia_) |
| `/api/onboarding/nominees/sync` | POST | (inside `listNominees()`) | pull FP `GET /v2/related_parties?profile=` → upsert our DB |
| `/api/onboarding/nominees` | GET | `listNomineesLocal()` | the pool (DB, non-hidden) |
| `/api/mf/accounts/{id}` | GET | `getAttachedNominees()` | currently-attached nominees + % from folio_defaults |
| `/api/onboarding/nominee` | POST | `createNominee()` | add a person to the pool |
| `/api/onboarding/nominee` | PATCH | `updateNominee()` | add previously-empty PAN/guardian only (name/rel/DOB/set-PAN immutable) |
| `/api/onboarding/nominee/{id}` | DELETE | `deleteNominee()` | soft-delete (FP has no delete → hidden locally) |
| `/api/onboarding/nominees/allocations` | POST | `setNomineeAllocations()` | **choose ≤3 + set % → PATCH MFIA folio_defaults** (the real nomination write) |
| `/api/onboarding/nominees/remove-all` | POST | `removeAllNominees()` | opt out (nulls folio_defaults nominees + sets show_nomination_status) |

**Sequence (happy path — one adult + one minor, 60/40)**
```
GET  /api/onboarding/status                    → invp_…, mfia_…
POST /api/onboarding/nominees/sync ; GET /api/onboarding/nominees   → current pool
GET  /api/mf/accounts/{mfia}                   → already-attached set + %

POST /api/onboarding/nominee   {adult…}                        → relp_A (FP person + local row)
POST /api/onboarding/nominee   {minor + guardianAddress… camelCase} → relp_B

POST /api/onboarding/nominees/allocations
     { allocations:[ {nomineeId:relp_A, allocationPercentage:60},
                      {nomineeId:relp_B, allocationPercentage:40} ] }
        └─ backend validates total = 100, ≤3, each > 0
        └─ ReattachNomineesToMfiaAsync → PATCH /api/mf/accounts → FP PATCH /v2/mf_investment_accounts/{mfia}
               folio_defaults: nominee1=relp_A(60%), nominee2=relp_B(40%, guardian proof)
        → 200  (or 502 with the FP reason on reject — UI must STOP, not claim success)
```
At **purchase/SIP** time the OTP modal surfaces this as Step-3 nomination consent for a **new folio only** (gated by `mf-folios/exists`); the nomination itself is already on the MFIA from the step above.

**`POST /api/onboarding/nominee` body — exact keys (binds to `CreateNomineeRequest`, case-insensitive camelCase)**
```json
{
  "profile": "invp_xxx",                 // optional; backend fills from logged-in user
  "name": "Sunita Mehta",
  "date_of_birth": "1992-08-12",         // ⚠ this ONE field is snake_case (legacy); dateOfBirth also accepted
  "relationship": "spouse",
  "allocationPercentage": 0,             // MFIA-side; real value set in Step 3
  "nomineePan": "ABCDE1234F",            // adult only; nulled server-side for a minor
  "isMinor": false,
  "nomineeEmail": "...", "nomineeIsd": "+91", "nomineeMobile": "...",
  "nomineeAddressLine1": "...", "nomineeAddressCity": "...",
  "nomineeAddressState": "...", "nomineeAddressPostalCode": "...", "nomineeAddressCountry": "in",
  "identityProofType": "aadhaar", "identityProofNumber": "1234",   // last-4 for aadhaar
  "guardianName": "...", "guardianRelationship": "father", "guardianPan": "...",
  "guardianEmail": "...", "guardianIsd": "+91", "guardianMobile": "...",
  "guardianAddressLine1": "...",         // ← camelCase, MANDATORY for a minor
  "guardianAddressCity": "...",
  "guardianAddressState": "...",
  "guardianAddressPostalCode": "...",    // ← camelCase, MANDATORY for a minor
  "guardianAddressCountry": "in"
}
```

**⚠ Guardian-address field names — the app-team question answered:** use **camelCase** — `guardianAddressLine1`, `guardianAddressCity`, `guardianAddressState`, `guardianAddressPostalCode`, `guardianAddressCountry`. That is what `CreateNomineeRequest` binds and what the web app already sends successfully. **Do NOT send `guardian_address` / snake_case** — that snake_case (`[JsonPropertyName("guardian_address")]` + an `AddressHash` object) belongs to a DIFFERENT internal class, **`CreateRelatedPartyRequest`** (the backend→Cybrilla DTO), NOT this API's contract. On this request, ONLY `date_of_birth` is snake_case (legacy); every other field is camelCase.

**Why a successful create doesn't echo the guardian address back (the "PATCH not working" confusion):** `OnboardingController.CreateNominee` **validates + persists locally** and forwards **only 4 fields** to FP `/v2/related_parties` (`profile, name, date_of_birth, relationship`). Allocation %, nominee PAN, and ALL guardian + address fields are stored in `tbl_Fintech_Investor_Nominee` and applied to the MFIA `folio_defaults` at **attach time (Step 3)** — they are NOT sent to FP on the create call, so FP won't echo them. For a minor, create **400s** if `guardianAddressLine1` or `guardianAddressPostalCode` is missing.

**Backend `CreateNominee` rules:** `ResolveMinor()` derives minor from DOB (age < 18); if minor → guardian name/relationship/PAN + address line1 + pincode required, and the nominee's own PAN is nulled (adult-only per FP).

---

## 14.6 Fresh KYC and Verified KYC use the SAME onboarding flow

> Important for the app team: **fresh KYC and verified/complete KYC differ ONLY in the KYC stage. After KYC, the onboarding flow is identical.** Build one onboarding flow; don't fork it by KYC type.

```
                        ┌─ Fresh PAN (no KRA): readiness=kyc_unavailable
                        │     → POST /kyc/forms (fill_fields → eSign → submitted)
   PAN → readiness ─────┤                                                   │
                        │                                                   ▼  (CONVERGE HERE)
                        └─ Verified PAN (KRA): readiness=verified ─────►  Investor Profile
                              (no KYC form — skipped)                       → Address
                                                                            → Contact (phone+email, OTP-verified)
                                                                            → Bank (+ penny-drop verify)
                                                                            → FATCA (+ optional Demat)
                                                                            → Nominee (optional, §14.5)
                                                                            → MF Account (MFIA)
                                                                            → Invest (gate: readiness=verified)
```

- **The only branch** is at readiness: a **fresh** PAN runs the `kyc/forms` lifecycle first; a **verified** PAN skips it. Both then enter the **same** investor-profile → address → contact → bank → FATCA/demat → nominee → MFIA sequence.
- **Convergence signal:** treat KYC as done when `investorProfileId` exists OR readiness `verified` OR the KYC form is `submitted`. From that point the resume logic (§14.4.2) is the same regardless of how KYC was satisfied — which is exactly why an existing `investorProfileId` must not bounce the user back to `/kyc`.
- **Auto-populate & OTP-verify (§14.3.3–14.3.4) apply to both paths** — a verified PAN still OTP-verifies email/mobile in onboarding and auto-fills profile fields; it just didn't need the fill-fields KYC form.

---

## 5.6 Token Caching (tenant OAuth)

**Why this exists.** Every call into Finprim and Cybrilla POA requires a tenant-scoped OAuth access token from `POST /v2/auth/{tenant}/token`. Naively fetching one per request would (a) be slow, (b) hit Cybrilla's rate limit, and (c) create a thundering-herd on cold start when many requests race for the same token.

**How it works.** A singleton `TenantTokenProvider` (`F:\github projects\FinprimApi\Services\TenantTokenProvider.cs`) sits in front of every outbound Finprim/Cybrilla call:

1. **L1 — In-process `IMemoryCache`** keyed by tenant name. Microsecond lookup; lives only for the lifetime of the process.
2. **L2 — Per-tenant `SemaphoreSlim`** so that when L1 misses, **only one request** refreshes; the rest await the same token. Eliminates the thundering herd.
3. **L3 — Database row** in `dbo.tbl_Fintech_Tenant_AccessToken` (one row per tenant) via `TenantTokenDbService`. This is the **warm-start** layer: after an API restart, the existing token is reused until it actually expires — no needless re-auth.
4. Only after all three miss/expire does the provider call Cybrilla's `/v2/auth/{tenant}/token` endpoint.

**Cached tenants** (today):

| Tenant key | Purpose |
|------------|---------|
| `islamiclyondc` | Main Finprim tenant — KYC, MFIA, orders, payments, banking. |
| `cybrillarta` | Cybrilla POA tenant — pre-verifications. |

**TTL.** `cache_ttl = max(60, expires_in - 60)` seconds. The 60-second floor avoids edge-case zero-TTL caching; the `-60` skew avoids a request firing with a token about to expire mid-flight.

See `F:\github projects\FinprimApi\Services\TenantTokenProvider.cs` and `F:\github projects\FinprimApi\Data\TenantTokenDbService.cs`.

---

## 6. Enum reference appendix

### KYC `gender`
`male` \| `female` \| `others`
(Angular maps `Male`→`male`, `Female`→`female`, `Other`→`others` — see `mutual-fund-kyc.component.ts:2622`.)

### KYC `marital_status`
`unmarried` \| `married` \| `divorced` \| `widowed`

### KYC `residential_status` / Investor `tax_status`
`resident_individual` \| `non_resident_individual`

### KYC `occupation_type` (Angular `occupations`)
`business` \| `professional` \| `retired` \| `housewife` \| `student` \| `public_sector` \| `private_sector` \| `government_sector` \| `others`

### Investor profile `occupation`
(Different field — see `InvestorProfileModels.cs:44`)
`agriculture` \| `business` \| `doctor` \| `forex_dealer` \| `government_service` \| `house_wife` \| `others` \| `private_sector_service` \| `professional` \| `public_sector_service` \| `retired` \| `service` \| `student`

Frontend mapping from KYC → Investor profile (`onboarding.component.ts:935`):

| KYC occupation_type | Investor profile occupation |
|---------------------|-----------------------------|
| `private_sector`    | `private_sector_service` |
| `public_sector`     | `public_sector_service` |
| `government` / `government_sector` | `government_service` |
| `business`          | `business` |
| `professional`      | `professional` |
| `agriculture`       | `agriculture` |
| `retired`           | `retired` |
| `home_maker` / `housewife` | `home_maker` (alias of `house_wife`) |
| `student`           | `student` |
| `forex_dealer`      | `forex_dealer` |
| `other`             | `other` |

### KYC `income_slab`
`upto_1lakh` \| `above_1lakh_upto_5lakh` \| `above_5lakh_upto_10lakh` \| `above_10lakh_upto_25lakh` \| `above_25lakh_upto_1cr` \| `above_1cr`

(Investor profile uses the same values; default `upto_1lakh`.)

### `pep_details` — TWO DIFFERENT enums (do not mix them) ⚠️

The PEP value differs by which resource you're posting to. They are **not** interchangeable — sending one resource's tokens to the other is rejected.

| Resource | Endpoint | Allowed `pep_details` values |
|----------|----------|------------------------------|
| **KYC Form** (fresh/modify KYC) | `POST`/`PATCH /api/kyc/forms` → `/poa/kyc_forms` | `pep` \| `related_pep` \| `no_exposure` |
| **Investor Profile** (FATCA step) | `POST /api/onboarding/fatca` / `POST /api/investor-profiles/my-profile` → `/v2/investor_profiles` | `pep_exposed` \| `pep_related` \| `not_applicable` |

**Mapping (KYC Form → Investor Profile)** — the user answers PEP once in the fresh-KYC fill-fields step (stored in `tbl_Fintech_Kyc_Form.PepDetails` in the KYC enum); carry it to the profile FATCA step using:

| KYC Form value | → Investor Profile value |
|----------------|--------------------------|
| `pep`          | `pep_exposed`            |
| `related_pep`  | `pep_related`            |
| `no_exposure`  | `not_applicable`         |

**App team — DO:**
- On the **KYC fill-fields** screen send `pep` / `related_pep` / `no_exposure`.
- On the **investor-profile / FATCA** screen send `pep_exposed` / `pep_related` / `not_applicable`.
- **Pre-fill the FATCA PEP** from the KYC PEP using the mapping above (so the user doesn't re-answer). The web app does this: `GET /api/investor-profiles/my-status` now returns `kycPepDetails` (the KYC-enum value); the FATCA screen maps it to the profile enum and pre-selects it. (`Usp_Get_Fintech_Investor_Profile_Status` → `KycPepDetails`.)
- The old `related_to_pep` token was WRONG for the investor profile and has been removed — use `pep_related`.

### Investor `source_of_wealth`
`salary` (default) \| `business` \| `inheritance` \| `gift` \| `savings` \| `other`

### Bank account `type`
`savings` (default) \| `current` \| `nre` \| `nro`

### Address `nature`
`residential` \| `correspondence`

### Pre-verification `account_type`
`savings` (default) \| `current` \| `nre` \| `nro`

### MFIA `holding_pattern`
`single` \| `anyone_or_survivor` \| `joint`

### MF scheme `investment_option`
`growth` \| `dividend_payout` \| `dividend_reinvest` \| `idcw_payout` \| `idcw_reinvest`

### MF scheme `plan_type`
`direct` \| `regular`

### MF scheme `delivery_mode`
`demat` \| `physical`

### Nominee `relationship`
`son` \| `daughter` \| `spouse` \| `father` \| `mother` \| `sibling` \| `other`

### Tax residency `taxid_type`
`pan` (default for India) \| `tin` \| `ssn` \| `other`

### Bank verification `confidence`
`very_high` \| `high` \| `uncertain` \| `low` \| `very_low` \| `zero`

### Bank verification `status` (mapped)
`pending` \| `completed` \| `failed`

### Bank verification `reason` (common codes)
`expiry` \| `digital_verification_failure` \| `name_mismatch` \| `account_does_not_exist` \| `<provider code>`

### KYC `action`
`create` \| `modify` \| `disallowed` \| `none` \| `unknown`

### KYC `reason`
`unavailable` \| `onhold` \| `rejected` \| `deactivated` \| `legacy` \| `underprocess` \| `incomplete` \| `unknown`

### KYC request `status`
`pending` \| `submitted` \| `successful` \| `rejected` \| `expired` (plus intermediate `esign_required` returned by Finprim before submission)

### KYC `wizardStep` (our DB only)
`pan` \| `personal` \| `address` \| `financial` \| `documents` \| `esign` \| `submitted` \| `successful` \| `rejected` \| `expired` \| `bank` \| `nominee`

### Identity document `type`
`aadhaar` (DigiLocker — recommended) \| `passport` \| `voter_id` \| `driving_license`

### MF purchase `state` (during PATCH consent)
`created` \| `confirmed`

### Payment `method`
`NETBANKING` \| `UPI` \| `NACH` (UPI not yet wrapped — use NACH or NETBANKING)

---

## 7. Common errors + recovery

| Error | Where it surfaces | What it means | Recovery |
|-------|-------------------|---------------|----------|
| `pan_kyc_incomplete` (400) | `/kyc/checks` follow-ups | KRA has the PAN but data is partial | Walk user through `/kyc/requests` flow with `action="modify"` |
| `esign_required` (400 from `/kyc/requests/{id}/esign`) | After DigiLocker | KYC is not in `esign_required` state yet — DigiLocker step incomplete | Re-run `/address/confirm`; ensure `fetch.status="successful"` |
| `"Aadhaar verification not completed"` (400 from `/address/confirm`) | DigiLocker | User did not finish OTP / declined consent | Re-open `redirectUrl` from `/address` response (call `PATCH /address` again — wrapper returns the existing iddoc) |
| `identity_document_already_exists` (handled silently) | `/identity-documents` create | We already have an iddoc for this KYC request | Wrapper auto-returns the existing one (IdentityDocumentsController.cs:58–68). No client action needed. |
| `readiness.status=failed, code=unknown` (200) | `/banking/my-verify/{id}` | Cybrilla penny-drop gateway down | Reference impl treats as `verified, confidence=not_checked` and allows progression; you may want to retry every few minutes |
| Bank verify `failed` + `confidence=zero` | `/banking/my-verify/{id}` | Account doesn't exist or name mismatch | Force user to **change bank account** (no retry — see `OnboardingComponent.bankSubState='failed'` flow) |
| Bank verify `failed` + `reason=expiry` | `/banking/my-verify/{id}` | Penny-drop request expired (rare) | Offer retry button — calls `POST /banking/my-verify` again |
| 401 Unauthorized | Any `[Authorize]` endpoint | JWT expired | Hit refresh-token endpoint, retry |
| 502 + `"Empty response from Finprim"` | Anywhere | Upstream timeout/unavailability | Exponential backoff (max 3); show generic "Try again" |
| `kyc_request.expired` webhook | Async | KYC application expired (>30 days unfinished) | Start a fresh `/kyc/checks` → new `/kyc/requests` cycle |
| `mf_purchase.failed` webhook | Async | AMC rejected order (typically min-amount, scheme paused, payment timeout) | Inspect `failure_reason` on the purchase object; allow retry via `POST /mf/orders/purchases/{id}/retry` |

---

## 8. Sandbox / testing tips

### Bank account number patterns (Cybrilla sandbox)

The sandbox uses suffix-based simulation. When testing `POST /banking/my-account` + `POST /banking/my-verify`:

| Account number ending | Result |
|-----------------------|--------|
| `11XX` (any 4 digits starting with 11) | Instant `completed`, `confidence=very_high` |
| `31XX` | `failed`, `confidence=zero`, `reason=digital_verification_failure` |
| `99XX` | `failed`, `reason=expiry` (use for testing the retry button) |
| anything else | `pending` for ~10 s then random verdict |

Use a standard sandbox IFSC such as `HDFC0000001` for the corresponding bank.

### KYC status decay (Finprim sandbox)

In sandbox, **completed KYC requests revert to `submitted` after ~5 minutes** — the KRA simulator drops the `successful` state. The wrapper's `GET /kyc/user-status` refresh logic accounts for this, but be aware: if your test session takes longer than 5 min, the user may bounce back to "Under Review". Use `POST /kyc/requests/{id}/simulate` with `{"status":"successful"}` to force-pin during long tests.

### Sandbox eSign / DigiLocker

Both DigiLocker and eSign redirect URLs work end-to-end in sandbox with the test Aadhaar `999999999999` and OTP `123456`. The postback URL is always called — verify your deep-link handler is wired up.

### Recommended test ISIN

`INF084M01044` — HDFC Liquid Fund Direct Growth. Stable in sandbox, min investment ₹500, T+1 NAV. Use for full purchase → payment.success cycle.

### Bootstrap webhooks (UAT)

```
POST https://fintechuat.islamicly.com/api/webhooks/bootstrap?url=https://fintechuat.islamicly.com/api/webhooks/finprim-events
```
Idempotent — re-run after any new event is added to `FinprimEventCatalog.cs`.

### appsettings keys clients should know about

**`Finprim` section** — all keys currently read by the backend:

| Key | Sandbox value | Notes |
|-----|---------------|-------|
| `Finprim:BaseUrl` | `https://s.finprim.com` | Server-side; clients do NOT call this directly. |
| `Finprim:TenantName` | `islamiclyondc` | Main tenant key used by `TenantTokenProvider`. |
| `Finprim:ClientId` | (secret) | OAuth client id for `/v2/auth/{tenant}/token`. |
| `Finprim:ClientSecret` | (secret) | OAuth client secret. |
| `Finprim:UseSandbox` | `true` | Flips test/live behaviour in handlers that branch on it. |
| `Finprim:TimeoutSeconds` | `30` | HttpClient timeout for outbound Finprim calls. |
| `Finprim:Postback:Web:DigiLocker` | `http://localhost:4200/admin/mutual-funds/kyc` | Full URL Finprim posts back to when `isWeb=1` for the DigiLocker flow. |
| `Finprim:Postback:Web:Esign`      | `http://localhost:4200/admin/mutual-funds/kyc?esign=completed` | Same, eSign flow. |
| `Finprim:Postback:App:DigiLocker` | `http://localhost:4200/appmutualfund/kyc` (dev) / `https://fintechuat.islamicly.com/appmutualfund/kyc` (UAT) | Full URL when `isWeb=0`; mobile WebView lands on this standalone Angular page. |
| `Finprim:Postback:App:Esign`      | `http://localhost:4200/appmutualfund/esign` (dev) / `https://fintechuat.islamicly.com/appmutualfund/esign` (UAT) | Same, eSign flow. |

**`Cybrilla` section** — used by pre-verifications (POA):

| Key | Sandbox value | Notes |
|-----|---------------|-------|
| `Cybrilla:BaseUrl` | `https://api.sandbox.cybrilla.com` | POA base URL — penny-drop, pre-verifications. |
| `Cybrilla:AuthTokenUrl` | `https://s.finprim.com/v2/auth/cybrillarta/token` | Token endpoint for the `cybrillarta` tenant. |
| `Cybrilla:ClientId` | (secret) | OAuth client id. |
| `Cybrilla:ClientSecret` | (secret) | OAuth client secret. |
| `Cybrilla:TimeoutSeconds` | `30` | HttpClient timeout. |

**Removed keys** (no longer read by the backend — delete from any old appsettings overrides):
- `Finprim:PostbackUrl`
- `Finprim:PostbackBaseUrls:Web`
- `Finprim:PostbackBaseUrls:App`
- `Finprim:WebhookToken`
- `Finprim:WebhookSecret`
- `Finprim:BridgeBaseUrl`

---

## 9. File-path index (where to find the code)

| Subject | File |
|---------|------|
| KYC controller | `F:\github projects\FinprimApi\Controllers\KycController.cs` (postback URL helper `GetPostbackUrl(isWeb, flow)` at lines 281–296; financial auto-defaults at 258–268; signature auto-heal at 464–471) |
| Webhook receiver (open endpoint) | `F:\github projects\FinprimApi\Controllers\FinprimWebhookReceiverController.cs` (`POST /api/webhooks/finprim-events`, no token, no signature verification, idempotent via `EventId UNIQUE`) |
| Angular mobile postback page — DigiLocker | `F:\github projects\IslamiclyWebLatest\src\app\features\app-mutual-funds\app-kyc-result\app-kyc-result.component.ts` (route `/appmutualfund/kyc`, standalone) |
| Angular mobile postback page — eSign | `F:\github projects\IslamiclyWebLatest\src\app\features\app-mutual-funds\app-esign-result\app-esign-result.component.ts` (route `/appmutualfund/esign`, standalone) |
| Angular root route table | `F:\github projects\IslamiclyWebLatest\src\app\app.routes.ts` (registers the two `/appmutualfund/*` routes outside the admin/layout shells) |
| Angular geo-redirect whitelist | `F:\github projects\IslamiclyWebLatest\src\app\shell.ts` (`NAMED_ROUTES` Set must include `'appmutualfund'` so the country-detection guard does not redirect to `/in`) |
| KYC models | `F:\github projects\FinprimApi\Models\Kyc\KycModels.cs` (incl. `UpdateFinancialRequest` with optional `pepDetails` / `taxResidencyOtherThanIndia` at lines 265–281), `KycCheckResponse.cs`, `KycRequestResponse.cs`, `KycUserStatusResponse.cs` |
| Investor profile controller | `Controllers\InvestorProfilesController.cs` |
| Investor profile models | `Models\InvestorProfiles\InvestorProfileModels.cs`, `ContactModels.cs`, `InvestorProfileStatusResponse.cs` |
| Identity documents | `Controllers\IdentityDocumentsController.cs`, `Models\IdentityDocuments\IdentityDocumentModels.cs` |
| Banking | `Controllers\BankingController.cs`, `Models\Banking\BankingModels.cs`, `BankAccountResponse.cs` |
| MF investment account | `Controllers\MfAccountsController.cs`, `Models\Mf\MfInvestmentAccountModels.cs` |
| MF orders (purchase / redemption / switch) | `Controllers\MfOrdersController.cs`, `Models\Mf\MfOrderModels.cs` |
| Pre-verification controller | `F:\github projects\FinprimApi\Controllers\PreVerificationsController.cs` |
| Pre-verification DB service | `F:\github projects\FinprimApi\Data\PreVerificationDbService.cs` |
| Pre-verification models | `F:\github projects\FinprimApi\Models\PreVerification\PreVerificationModels.cs` |
| Tenant token caching (DB) | `F:\github projects\FinprimApi\Data\TenantTokenDbService.cs` |
| Tenant token caching (provider) | `F:\github projects\FinprimApi\Services\TenantTokenProvider.cs` |
| MF scheme plan DB service | `F:\github projects\FinprimApi\Data\MfSchemePlanDbService.cs` |
| MF scheme catalog DB service | `F:\github projects\FinprimApi\Data\MfSchemeCatalogDbService.cs` |
| Angular pre-verification service | `F:\github projects\IslamiclyWebLatest\src\app\services\pre-verification.service.ts` |
| Angular MF funds service | `F:\github projects\IslamiclyWebLatest\src\app\services\mf-funds.service.ts` |
| Payments | `Controllers\PaymentsController.cs`, `Models\Payments\PaymentModels.cs` |
| Onboarding aggregator | `Controllers\OnboardingController.cs`, `Models\Onboarding\*.cs` |
| Master data | `Controllers\MasterDataController.cs` |
| Webhook subscriptions | `Controllers\WebhooksController.cs` |
| Event catalog | `Services\FinprimEventCatalog.cs` |
| App settings | `appsettings.json`, `appsettings.Development.json` |
| Angular orchestrator | `F:\github projects\IslamiclyWebLatest\src\app\services\kyc.service.ts` |
| Angular KYC wizard | `src\app\features\mutual-funds\mutual-fund-kyc\mutual-fund-kyc.component.ts` |
| Angular onboarding wizard | `src\app\features\mutual-funds\onboarding\onboarding.component.ts` |

---

*Update `FinprimEventCatalog.cs` and re-run `/api/webhooks/bootstrap` whenever new event types are added on Cybrilla's side.*

---

## 10. Per-endpoint Success & Failure Scenarios

> **Wrapper envelope (verified from `Models/Common/ApiResponse.cs`):**
> Every wrapper-owned endpoint serialises via `ApiResponse<T>` →
> ```json
> { "statusCode": 200, "message": "Success", "result": { ... } }
> ```
> Failures come from the `ErrorApi(code, message)` helper which produces:
> ```json
> { "statusCode": 4xx_or_5xx, "message": "<human-readable>", "result": null }
> ```
> The global `ExceptionHandlingMiddleware` catches uncaught `HttpRequestException` as **502** and any other exception as **500** with the same envelope.
> Earlier sections of this doc showed a `{success, code, message, data}` shape — that is historical; the actual on-the-wire shape is `{statusCode, message, result}`. Treat `statusCode === 200` as success; surface `message` to the user on errors.
>
> **`isWeb` stripping:** `Services/FinprimApiClient.cs:117` strips any `"isWeb": N` key from outbound JSON before it leaves our wrapper, so Finprim never receives it and you will never see Finprim's "unpermitted parameter" rejection because of it.
>
> **CORS:** `Program.cs:81` uses `SetIsOriginAllowed(_ => true)` in development — `file://`, any localhost port, mobile WebViews, Postman all work. If you get a CORS preflight failure in dev, the build is misconfigured, not the policy.

---

### 10.1 `POST /api/kyc/checks`

#### Success Scenario
- **Preconditions:** valid JWT; PAN matches `^[A-Z]{5}[0-9]{4}[A-Z]$`; `date_of_birth` in `YYYY-MM-DD`.
- **Response body (verbatim from `KycController:108`):**
```json
{
  "statusCode": 200,
  "message": "KYC check completed",
  "result": {
    "id": "kyc_abc123",
    "pan": "ABCDE1234F",
    "status": true,
    "action": "none",
    "reason": "incomplete",
    "entity_details": { "name": "ARJUN MEHTA", "date_of_birth": "1990-04-15" },
    "constraints": [],
    "local_id": 42
  }
}
```
- **Meaning:** PAN was looked up against KRA via Finprim and (here) is already compliant.
- **Client next:** If `result.status === true` → skip to `POST /api/investor-profiles/my-profile`. If `result.action === "create"|"modify"` → navigate to KYC wizard and call `POST /api/kyc/requests`. Cache `local_id` for later joins.
- **Backend side-effects:** Row written to `tbl_Fintech_KYC_Check` (one per user × PAN). No webhook follow-up — KYC checks are synchronous.

Special success: if Finprim's tenant has `kyc_checks` disabled, the controller catches the 404 and returns a synthetic envelope (`KycController:90-96`) — `result.action="create"`, `result.status=false`, message `"Proceed to create KYC request"`.

#### Failure Scenarios

| HTTP | Code/Message | Cause | Retryable | Recovery |
|------|--------------|-------|-----------|----------|
| 400 | `"The pan field is required"` / model-binding errors | Bad PAN format / missing DOB | retry-after-fix | Re-prompt user for PAN+DOB |
| 401 | `"Unauthorized"` | JWT expired/missing | retry-after-fix | Refresh JWT, retry call |
| 502 | `"KYC check failed. Please try again."` | Finprim returned null (`KycController:101`) | yes | Backoff 5 s, retry up to 3× then show "service unavailable" |
| 502 | `<exception message>` (`ExceptionHandlingMiddleware:26`) | `HttpRequestException` from downstream | yes | Same as above |
| 500 | `"Something went wrong"` (prod) / stack trace (dev) | Unhandled exception | no | Log + report |

Sandbox notes: PAN of any well-formed string returns `action="create"` instantly in sandbox.

---

### 10.2 `GET /api/kyc/user-status`

#### Success Scenario
- **Preconditions:** valid JWT.
- **Response (`KycController:140`):**
```json
{
  "statusCode": 200,
  "message": "User KYC status retrieved",
  "result": {
    "hasKycCheck": true,
    "kycCheckLocalId": 42,
    "kycId": "kyc_abc123",
    "pan": "ABCDE1234F",
    "kycStatus": false,
    "action": "create",
    "hasKycRequest": true,
    "kycRequestId": "kyr_xyz789",
    "kycRequestStatus": "pending",
    "kycRequestWizardStep": "address",
    "isKYCCompliant": false
  }
}
```
- **Meaning:** The wrapper-cached KYC state for this user, auto-refreshed from Finprim when not yet successful.
- **Client next:** Use `kycRequestWizardStep` to resume the wizard at the right step; if `isKYCCompliant=true` skip straight to investor profile creation.
- **Backend side-effects:** On refresh, may update `tbl_Fintech_KYC_Check` and `tbl_Fintech_KYC_Request`.

#### Failure Scenarios

| HTTP | Code/Message | Cause | Retryable | Recovery |
|------|--------------|-------|-----------|----------|
| 200 | `result.hasKycCheck=false` | First-time user | n/a | Start with `POST /api/kyc/checks` |
| 401 | Unauthorized | JWT | retry-after-fix | Refresh token |
| 502 | Bad gateway (middleware) | Finprim auto-refresh failed | yes | Stale DB state still returned in normal flow; retry endpoint |

**KYC decay (sandbox-only):** in sandbox, a previously `successful` KYC reverts to `submitted` after roughly 5 min. Detect via this endpoint (`kycRequestStatus` flipping back), then call `POST /api/kyc/requests/{id}/simulate` with `{"status":"successful"}` to pin it.

---

### 10.3 `POST /api/kyc/requests`

#### Success Scenario
- **Preconditions:** prior `/kyc/checks` returned `action=create|modify`; valid personal block including PAN, DOB, email, mobile.
- **Response (`KycController:201`):**
```json
{
  "statusCode": 200,
  "message": "KYC request created",
  "result": {
    "id": "kyr_xyz789",
    "status": "pending",
    "pan": "ABCDE1234F",
    "name": "ARJUN MEHTA",
    "fields_needed": ["identity_proof", "address.proof", "signature"],
    "expires_at": "2026-06-17T...",
    "local_id": 17
  }
}
```
- **Client next:** Store `result.id` (`kyr_xxx`); call `PATCH /api/kyc/requests/{id}/address?isWeb=0|1`.
- **Backend side-effects:** Inserts into `tbl_Fintech_KYC_Request`; no webhook yet (webhooks fire after submission).

#### Failure Scenarios

| HTTP | Code/Message | Cause | Retryable | Recovery |
|------|--------------|-------|-----------|----------|
| 400 | model-binding errors | Missing/invalid required fields (name, pan, dob, email, mobile) | retry-after-fix | Show form validation |
| 401 | Unauthorized | JWT | retry-after-fix | Refresh JWT |
| 502 | `"Failed to create KYC request. Please try again."` (`KycController:166`) | Finprim null/error | yes | Backoff retry |

---

### 10.4 `PATCH /api/kyc/requests/{id}`

#### Success Scenario
- Returns 200 with envelope `message="KYC request updated"` and `result` = full `KycRequestResponse`.

#### Failure Scenarios

| HTTP | Code/Message | Cause | Retryable | Recovery |
|------|--------------|-------|-----------|----------|
| 400 | validation | invalid enum/field | retry-after-fix | Fix payload |
| 401 | Unauthorized | JWT | retry-after-fix | Refresh JWT |
| 404 | implicit from Finprim (502 from middleware) | `id` not found | no | Recreate via `POST /kyc/requests` |
| 502 | `"Failed to update KYC request."` (`KycController:216`) | Finprim error | yes | Retry |

---

### 10.5 `PATCH /api/kyc/requests/{id}/financial`

#### Success Scenario
- **Response (`KycController:276`):**
```json
{
  "statusCode": 200,
  "message": "Financial details updated",
  "result": { "id": "kyr_xyz789", "status": "esign_required", "...": "..." }
}
```
- **Client next:** Navigate to documents/signature step.
- **Backend side-effects:** Sets `WizardStep="documents"`. If `pep_details` / `tax_residency_other_than_india` were omitted by the client, backend auto-defaults them to `"not_applicable"` and `false` respectively (`KycController.UpdateFinancial` lines 258–268) so the KYC request can still transition through `esign_required`.

#### Failure Scenarios

| HTTP | Code/Message | Cause | Retryable | Recovery |
|------|--------------|-------|-----------|----------|
| 400 | bad enum for `occupation_type` / `income_slab` | wrong value | retry-after-fix | Show enum reference |
| 401 | Unauthorized | JWT | retry-after-fix | Refresh |
| 502 | `"Failed to update financial details."` (`KycController:266`) | Finprim error | yes | Retry |

---

### 10.6 `PATCH /api/kyc/requests/{id}/address?isWeb=0|1`

#### Success Scenario
- **Preconditions:** KYC request exists; user logged in.
- **Response (`KycController:364`):**
```json
{
  "statusCode": 200,
  "message": "DigiLocker redirect ready",
  "result": {
    "identityDocId": "iddc_aaa111",
    "redirectUrl": "https://digilocker.gov.in/...partner=cybrilla&txn=..."
  }
}
```
- **Client next:** Open `redirectUrl`. Web → same tab or popup; mobile → **in-app WebView** (not an external browser / Custom Tab). After the WebView reaches `/appmutualfund/kyc`, call `/address/confirm` with the `identityDocId`.
- **Backend side-effects:** Optional `geolocation` PATCH on KYC, row in `tbl_Fintech_Identity_Document`, `IdentityDocumentId` stamped on the KYC request row. If an iddoc already exists, returns the existing one (`identity_document_already_exists` from Finprim is swallowed by `IdentityDocumentsController:58-68`).

#### Failure Scenarios

| HTTP | Code/Message | Cause | Retryable | Recovery |
|------|--------------|-------|-----------|----------|
| 401 | Unauthorized | JWT | retry-after-fix | Refresh |
| 502 | `"Failed to update geolocation."` (`KycController:316`) | Finprim PATCH failed before iddoc creation | yes | Retry without lat/lng |
| 502 | `"Failed to create identity document. Please try again."` (`KycController:349`) | Finprim/DigiLocker tenant misconfig or downstream timeout | yes | Backoff 5 s × 3, then fail UI |
| (silent) | `identity_document_already_exists` from Finprim | Wrapper swallows and returns the existing iddoc | n/a | No client action — same `redirectUrl` returned |

##### Mobile postback landing — `GET /appmutualfund/kyc` (Angular standalone page)

Implemented in `F:\github projects\IslamiclyWebLatest\src\app\features\app-mutual-funds\app-kyc-result\app-kyc-result.component.ts`. Rendered entirely client-side — **no backend HTTP call is made by this page** (the app re-fetches state via `GET /api/kyc/user-status` after the WebView closes). Failure modes the WebView will encounter:

| Scenario | Behavior | What the app sees |
|----------|----------|-------------------|
| User cancelled in DigiLocker (`fetch.status=failed`, `reason=user cancelled`) | Red ❌ panel with the reason shown | `data-result="failure"`, `data-status="failed"`; app should re-open the same `redirectUrl` (the iddoc is reused) or restart the address step |
| DigiLocker session expired (`fetch.status=expired`) | Red ❌ panel; copy includes the reason if present | `data-status="expired"`; UI should advise the user to retry — calling `PATCH /kyc/requests/{id}/address` again will return the existing iddoc with a fresh `redirectUrl` |
| Finprim posts back with `fetch.status=pending` | Amber ⏳ panel | App should NOT call `/address/confirm`; let the user retry |
| `identity_document` query param missing | Generic failure panel with empty `data-iddoc` | Treat as advisory and confirm via `GET /api/kyc/user-status` |
| `fetch.status=successful` | Green ✅ panel + "Return to App" button | App can call `POST /api/kyc/requests/{id}/address/confirm` |

---

### 10.7 `POST /api/kyc/requests/{id}/address/confirm`

#### Success Scenario
- **Preconditions:** user completed DigiLocker; Finprim's iddoc has `fetch.status="successful"`.
- **Response (`KycController:409`):**
```json
{
  "statusCode": 200,
  "message": "Address verified successfully",
  "result": { "kycRequest": { "id": "kyr_xyz789", "status": "esign_required", "...": "..." } }
}
```
- **Client next:** Navigate to financial → documents → signature → eSign.
- **Backend side-effects:** PATCH on KYC request with `identity_proof` + `address`; `WizardStep="financial"`.

#### Failure Scenarios

| HTTP | Code/Message | Cause | Retryable | Recovery |
|------|--------------|-------|-----------|----------|
| 400 | `"Aadhaar verification not completed. Status: <reason>"` (`KycController:381`) | User did not finish DigiLocker OTP, declined consent, or `fetch.status≠successful` | retry-after-fix | Re-open redirectUrl from `/address` (wrapper returns same iddoc) |
| 401 | Unauthorized | JWT | retry-after-fix | Refresh |
| 502 | `"Failed to fetch identity document from Finprim."` (`KycController:375`) | Finprim GET failed | yes | Retry |
| 502 | `"Failed to update address proof."` (`KycController:395`) | Finprim PATCH failed | yes | Retry |

---

### 10.8 `POST /api/kyc/requests/{id}/signature` (multipart)

#### Success Scenario
- **Preconditions:** non-empty PNG/JPEG signature file.
- **Response (`KycController:473`):**
```json
{
  "statusCode": 200,
  "message": "Signature uploaded",
  "result": { "fileId": "file_sig123", "kycStatus": "esign_required" }
}
```
- **Client next:** Call `POST /kyc/requests/{id}/esign`.
- **Backend side-effects:** Row in `tbl_Fintech_File_Upload`, PATCH on KYC request with **three** fields together — `signature=<file_id>`, `pep_details="not_applicable"`, `tax_residency_other_than_india=false` (`KycController.UploadSignature` lines 464–471). This auto-heal ensures Finprim transitions to `esign_required` even if the financial step was skipped. `WizardStep="esign"`.

#### Failure Scenarios

| HTTP | Code/Message | Cause | Retryable | Recovery |
|------|--------------|-------|-----------|----------|
| 400 | `"No file provided."` (`KycController:450`) | Empty multipart | retry-after-fix | Force user to re-sign |
| 401 | Unauthorized | JWT | retry-after-fix | Refresh |
| 502 | `"Failed to upload signature. Please try again."` (`KycController:460`) | Finprim Files upload error | yes | Retry |
| 502 | `"Failed to link signature to KYC request."` (`KycController:468`) | Finprim PATCH error | yes | Retry — the file was uploaded; just re-PATCH (server logic uploads only once if you supply a known file) |

---

### 10.9 `POST /api/kyc/requests/{id}/esign?isWeb=0|1`

#### Success Scenario
- **Preconditions:** KYC is in `esign_required` (DigiLocker + signature done).
- **Response (`KycController:521` or `:493`):**
```json
{
  "statusCode": 200,
  "message": "eSign created",
  "result": { "esignId": "esn_p999", "redirectUrl": "https://...nsdl/esign/..." }
}
```
- **Client next:** Open `redirectUrl`. On return, call `/esign/confirm` with `esign_id`.
- **Backend side-effects:** Row in `tbl_Fintech_ESign`; idempotent — re-calling returns the same row (`KycController:493`).

#### Failure Scenarios

| HTTP | Code/Message | Cause | Retryable | Recovery |
|------|--------------|-------|-----------|----------|
| 400 | `"KYC request is not ready for eSign. Please make sure ALL these steps are complete: personal details, financial details (occupation + income), Aadhaar verification, and signature upload. Refresh and try again."` (`KycController:511`) | KYC not in `esign_required` state — one or more prerequisite steps (personal, financial, DigiLocker address, signature) incomplete | retry-after-fix | Call `GET /api/kyc/user-status` to identify the missing step, then route the user to it (financial / DigiLocker / signature upload) before retrying |
| 401 | Unauthorized | JWT | retry-after-fix | Refresh |
| 502 | `"Failed to create eSign. Please try again."` (`KycController:515`) | Finprim/NSDL gateway error | yes | Backoff 5 s, retry 3× |

##### Mobile postback landing — `GET /appmutualfund/esign` (Angular standalone page)

Implemented in `F:\github projects\IslamiclyWebLatest\src\app\features\app-mutual-funds\app-esign-result\app-esign-result.component.ts`. Rendered entirely client-side — **no backend HTTP call is made by this page** (the app refreshes state via `GET /api/kyc/user-status` once the WebView closes). Success values accepted (case-insensitive): `signed`, `successful`, `completed`. Failure modes:

| Scenario | Behavior | What the app sees |
|----------|----------|-------------------|
| User cancelled in NSDL (`status=cancelled` or `status=failed`, `reason="user cancelled"`) | Red ❌ panel with reason | `data-result="failure"`; app should let the user retry — re-calling `POST /kyc/requests/{id}/esign` returns the existing row idempotently |
| eSign timed out / session expired | Red ❌ panel | App should re-trigger eSign |
| `esign` (id) query param missing | Generic failure panel with empty `data-esign` | Confirm via `GET /api/kyc/user-status` |
| `status` is pending or unknown | Amber ⏳ panel | App should not call `/esign/confirm` yet |
| `status=signed` (or `successful`/`completed`) | Green ✅ panel with "Return to App" button | App calls `POST /api/kyc/requests/{id}/esign/confirm` with the stored `esign_id` |

---

### 10.10 `POST /api/kyc/requests/{id}/esign/confirm`

#### Success Scenario
- **Response (`KycController:542`):**
```json
{
  "statusCode": 200,
  "message": "eSign confirmed. KYC submitted.",
  "result": { "kycRequest": { "id": "kyr_xyz789", "status": "submitted", "...": "..." } }
}
```
- **Client next:** Poll `GET /api/kyc/requests/{id}/status` (or wait for the `kyc_request.successful` webhook). When successful, call `POST /api/investor-profiles/my-profile`.
- **Backend side-effects:** Updates eSign row to `completed`; PATCH/refetch on KYC; `WizardStep="submitted"`. Webhook `kyc_request.submitted` then `kyc_request.successful` will arrive asynchronously.

#### Failure Scenarios

| HTTP | Code/Message | Cause | Retryable | Recovery |
|------|--------------|-------|-----------|----------|
| 401 | Unauthorized | JWT | retry-after-fix | Refresh |
| 502 | `"Failed to verify eSign completion."` (`KycController:536`) | Finprim GET on eSign failed | yes | Retry |

---

### 10.11 `POST /api/kyc/requests/{id}/simulate` (sandbox only)

#### Success Scenario
- **Body:** `{ "status": "successful" }` (or `submitted` / `rejected` / `expired`).
- **Response:** Forwarded Finprim object inside `result` envelope; `message="Success"`.
- **Use:** Pin KYC state during a long sandbox session — counters the KYC-decay-to-`submitted` behaviour. The wrapper's `/kyc/user-status` will then see the new state on next refresh.

#### Failure Scenarios

| HTTP | Code/Message | Cause | Retryable | Recovery |
|------|--------------|-------|-----------|----------|
| 400 | Finprim "invalid status transition" | Trying to go backward, or sandbox flag disabled in prod | no | Use only `successful` from any state in sandbox |
| 502 | exception → middleware | Tenant doesn't allow simulate | no | Confirm `Finprim:UseSandbox=true` and tenant is sandbox |

---

### 10.12 `POST /api/investor-profiles/my-profile` (and `/investor-profiles`)

#### Success Scenario
- **Preconditions:** KYC is `successful` (or at minimum, KYC fields are available — wrapper does not strictly gate); PAN + name + DOB + signature file id present.
- **Response (`InvestorProfilesController:121`):** the raw Finprim investor profile inside `result`, message `"Investor profile created successfully"`:
```json
{
  "statusCode": 200,
  "message": "Investor profile created successfully",
  "result": {
    "id": "invp_xxx",
    "type": "individual",
    "tax_status": "resident_individual",
    "name": "Arjun Mehta",
    "pan": "ABCDE1234F",
    "...": "..."
  }
}
```
- **Client next:** Store `invp_xxx`. Run onboarding children (`/onboarding/address`, `/phone`, `/email`, `/fatca`, optional `/nominee`, optional `/demat`).
- **Backend side-effects:** Row in `tbl_Fintech_Investor_KYC_Profile` linking user → `invp_xxx`.

#### Failure Scenarios

| HTTP | Code/Message | Cause | Retryable | Recovery |
|------|--------------|-------|-----------|----------|
| 400 | model-binding | missing required field | retry-after-fix | Pre-fill from `/investor-profiles/my-status` |
| 401 | Unauthorized | JWT | retry-after-fix | Refresh |
| 502 | `"Failed to create investor profile."` (`InvestorProfilesController:87` or `:93`) | Finprim error or null response | yes | Backoff retry |

---

### 10.13 Onboarding children — `/api/onboarding/{address,phone,email,demat,fatca,nominee}`

These share an identical wrapper pattern (`OnboardingController`): create-with-DB-mirror, returns the raw Finprim object in `result`.

#### Success Scenario (representative — address)
- **Response (`OnboardingController:116`):**
```json
{
  "statusCode": 200,
  "message": "Address created successfully",
  "result": { "id": "addr_yyy", "profile": "invp_xxx", "line1": "...", "city": "Mumbai", "...": "..." }
}
```
- Other endpoints return: `"Phone number created successfully"` (`:167`), `"Email address created successfully"` (`:218`), `"Demat account created successfully"` (`:269`), `"Tax residency submitted successfully"` (`:340` — FATCA), `"Nominee added successfully"` (`:386`).
- **Client next:** After all required children created, call `POST /api/onboarding/mf-account` (MFIA).
- **Backend side-effects:** ID stored on profile's child mirror table; ID later read by `/onboarding/status`.

#### Failure Scenarios (uniform across children)

| HTTP | Code/Message | Cause | Retryable | Recovery |
|------|--------------|-------|-----------|----------|
| 400 | `"Investor profile not found. Please complete investor profile first."` (FATCA, `:287`) | No `invp_xxx` mirrored | retry-after-fix | Run `POST /investor-profiles/my-profile` first |
| 400 | `"Investor PAN not found. Cannot set Indian tax residency."` (FATCA, `:290`) | Investor profile mirrored but PAN missing | retry-after-fix | Re-create profile with PAN |
| 401 | Unauthorized | JWT | retry-after-fix | Refresh |
| 502 | `"Failed to create <resource> with provider. Please try again."` | Outbound HTTP exception | yes | Backoff retry |
| 502 | `"<resource> creation failed. Please try again."` | Finprim returned null | yes | Retry |
| 502 | `"Unexpected response format from provider."` | Finprim returned non-JSON or wrong shape | no | Escalate / log; possibly tenant change |
| 502 | `"<resource> id missing in provider response."` | Finprim returned JSON without `id` | no | Same — escalate |

---

### 10.14 `POST /api/banking/my-account`

#### Success Scenario
- **Preconditions:** Investor profile exists; account-number 9–18 digits; valid IFSC.
- **Response (`BankingController:131`):**
```json
{
  "statusCode": 200,
  "message": "Bank account added successfully",
  "result": {
    "id": "bac_bbb",
    "profile": "invp_xxx",
    "primary_account_holder_name": "ARJUN MEHTA",
    "account_number": "1234567890",
    "type": "savings",
    "ifsc_code": "HDFC0001234",
    "bank_name": "HDFC Bank",
    "branch_name": "...",
    "branch_city": "...",
    "created_at": "..."
  }
}
```
- **Client next:** Call `POST /api/banking/my-verify` to start penny-drop. Store `bac_bbb`.
- **Backend side-effects:** Row in `tbl_Fintech_Bank_Account`.

#### Failure Scenarios

| HTTP | Code/Message | Cause | Retryable | Recovery |
|------|--------------|-------|-----------|----------|
| 400 | model-binding (`ifsc_code` empty etc.) | bad payload | retry-after-fix | Show form errors |
| 400 | `"Account number must be 6-18 digits."` (`BankingController:73-96`) | server-side guard — `account_number` length / non-digit | retry-after-fix | Re-prompt account number |
| 400 | `"Invalid IFSC code format."` | server-side guard — IFSC fails regex `^[A-Z]{4}0[A-Z0-9]{6}$` | retry-after-fix | Re-prompt IFSC (uppercase, 11 chars, 5th char `0`) |
| 400 | `"IFSC code cannot equal the account number."` | server-side guard — copy/paste error | retry-after-fix | Re-prompt both fields |
| 400 | `"Invalid account type."` | server-side guard — `type` not in {`savings`, `current`, `nre_savings`, `nro_savings`} | retry-after-fix | Use a valid enum value |
| 401 | Unauthorized | JWT | retry-after-fix | Refresh |
| 502 | `"Failed to create bank account with payment provider. Please try again."` (`BankingController:95`) | Outbound exception | yes | Retry |
| 502 | `"Bank account creation failed. Please try again."` (`:101`) | Finprim null | yes | Retry |
| 502 | `"Unexpected response format from payment provider."` (`:118`) | wrong shape | no | Escalate |
| 502 | `"Bank account id missing in provider response."` (`:124`) | no `id` | no | Escalate |

---

### 10.15 `POST /api/banking/my-verify`

#### Success Scenario
- **Preconditions:** `tbl_Fintech_Bank_Account` row exists; PAN on investor profile.
- **Response (`BankingController:263`):**
```json
{
  "statusCode": 200,
  "message": "Bank verification started",
  "result": {
    "verificationId": "prv_qqq",
    "status": "pending",
    "code": "",
    "message": ""
  }
}
```
- **Meaning:** Pre-verification kicked off on Cybrilla POA. The `bank_accounts` array in Cybrilla's response is initially a placeholder (often `[{}]`) — **do not treat this POST as final**.
- **Client next:** Poll `GET /api/banking/my-verify/{prv_qqq}` every 4 s (max 60 polls / 4 min) until `status !== "pending"`.
- **Backend side-effects:** Row in `tbl_Fintech_Bank_Verification`.

#### Failure Scenarios

| HTTP | Code/Message | Cause | Retryable | Recovery |
|------|--------------|-------|-----------|----------|
| 400 | `"No bank account found. Please add a bank account first."` (`:207`) | DB miss | retry-after-fix | Send user back to add-bank |
| 400 | `"Account holder name missing. Please re-add your bank account."` (`:210`) | Mirror has no name | retry-after-fix | Re-add bank |
| 400 | `"PAN not found. Please complete investor profile creation first."` (`:216`) | Investor profile missing PAN | retry-after-fix | Run profile creation |
| 400 | (passthrough) `"Invalid investor identifier"` from Cybrilla | PAN missing/wrong-case on investor profile | retry-after-fix | Recreate profile ensuring uppercase PAN |
| 401 | Unauthorized | JWT | retry-after-fix | Refresh |
| 502 | `"Failed to start bank verification. Please try again."` (`:250`) | outbound exception | yes | Retry |
| 502 | `"Bank verification could not be started. Please try again."` (`:254`) | Cybrilla null | yes | Retry |

---

### 10.16 `GET /api/banking/my-verify/{verificationId}` (poll)

#### Success Scenario
- **Response (`BankingController:343`):**
```json
{
  "statusCode": 200,
  "message": "Verification status retrieved",
  "result": {
    "verificationId": "prv_qqq",
    "status": "completed",
    "confidence": "very_high",
    "reason": "",
    "code": "",
    "message": ""
  }
}
```
- **Client next:** When `result.status === "completed"` → continue to `POST /api/onboarding/mf-account`. When `failed` → branch into the bank-failed UI (force "change bank"). When `pending` → keep polling.

#### derivedStatus matrix (status × confidence × readiness)

| `bank_accounts[0].status` | `readiness.status` | derivedStatus (wrapper output) | confidence default | reason default |
|---|---|---|---|---|
| `completed` / `success` / `verified` | (any) | `completed` | `very_high` | `""` |
| `failed` / `failure` | (any) | `failed` | `zero` | `bank.reason` ?? `digital_verification_failure` |
| (missing) | `failed` (code=`unknown`) | `failed` | `zero` | `readiness.code` = `unknown` |
| (missing) | `failed` (code=other) | `failed` | `zero` | `readiness.code` |
| `pending` / (missing) | `ready` / (missing) | `pending` | (omitted) | (omitted) |
| `pending` | `pending` | `pending` | (omitted) | (omitted) |

#### Failure Scenarios

| HTTP | Code/Message | Cause | Retryable | Recovery |
|------|--------------|-------|-----------|----------|
| 401 | Unauthorized | JWT | retry-after-fix | Refresh |
| 502 | `"Failed to check verification status."` (`:284`) | outbound exception | yes | Continue polling; treat as transient |
| 502 | `"Verification not found."` (`:288`) | Cybrilla 404 | no | Start a new `POST /banking/my-verify` |
| 200 + `status:"failed"`, `reason:"unknown"` | Sandbox `readiness.status=failed, code=unknown` | Cybrilla POA gateway misconfig / propagation lag on `cybrillapoa` gateway | retry-after-fix | Re-simulate KYC (`/kyc/requests/{id}/simulate`), wait ~30 s, **create a new pre-verification** (do NOT poll the same `prv_qqq` — it is terminal) |
| 200 + `status:"failed"`, `confidence:"zero"`, `reason:"digital_verification_failure"` | Sandbox account ending in `31XX` | no | Force user to change bank account |
| 200 + `status:"failed"`, `reason:"expiry"` | Sandbox account ending in `99XX` or timeout | yes | Show retry button → `POST /banking/my-verify` again |

Sandbox suffix matrix (Cybrilla):
- `11XX` → instant `completed` / `very_high`
- `31XX` → `failed` / `zero` / `digital_verification_failure`
- `99XX` → `failed` / `expiry`

---

### 10.17 `POST /api/banking/set-primary/{bankAccountId}`

#### Success Scenario
- **Preconditions:** User has an `mfia_xxx`.
- **Response (`BankingController:189`):**
```json
{
  "statusCode": 200,
  "message": "Primary bank account updated",
  "result": { "mfAccountId": "mfia_ccc", "primaryBankAccountId": "bac_bbb" }
}
```
- **Client next:** Continue to ISIN lookup → MF purchase.
- **Backend side-effects:** PATCHes `folio_defaults.payout_bank_account = bac_bbb` on the MFIA.

#### Failure Scenarios

| HTTP | Code/Message | Cause | Retryable | Recovery |
|------|--------------|-------|-----------|----------|
| 400 | `"MF Investment Account not found. Please complete onboarding before changing primary bank."` (`:164`) | No mfia_xxx | retry-after-fix | Run `POST /onboarding/mf-account` first |
| 401 | Unauthorized | JWT | retry-after-fix | Refresh |
| 502 | `"Failed to update primary bank on Finprim. Please try again."` (`:179`) | Finprim PATCH error | yes | Retry |

---

### 10.18 `POST /api/onboarding/mf-account` (MFIA)

#### Success Scenario
- **Preconditions:** investor profile, at least one address/phone/email/bank child exist.
- **Response (`OnboardingController:469`):**
```json
{
  "statusCode": 200,
  "message": "MF investment account created successfully",
  "result": {
    "id": "mfia_ccc",
    "primary_investor": "invp_xxx",
    "holding_pattern": "single",
    "folio_defaults": {
      "communication_email_address": "ema_aaa",
      "communication_mobile_number":  "phon_zzz",
      "communication_address":        "addr_yyy",
      "payout_bank_account":          "bac_bbb"
    }
  }
}
```
- **Client next:** Call `POST /api/banking/set-primary/{bacId}` (if not already), then ISIN lookup → `/mf/orders/purchases`.
- **Backend side-effects:** Row in `tbl_Fintech_MF_Account`; auto-PATCH applies `folio_defaults`.

#### Failure Scenarios

| HTTP | Code/Message | Cause | Retryable | Recovery |
|------|--------------|-------|-----------|----------|
| 401 | Unauthorized | JWT | retry-after-fix | Refresh |
| 502 | `"Failed to create MF investment account with provider. Please try again."` (`:412`) | outbound exception | yes | Retry |
| 502 | `"MF account creation failed. Please try again."` (`:416`) | Finprim null | yes | Retry |
| 502 | `"Unexpected response format from provider."` (`:431`) | wrong shape | no | Escalate |
| 502 | `"MF account id missing in provider response."` (`:435`) | no `id` | no | Escalate |

---

### 10.19 `GET /api/masterdata/fund-schemes` and `/by-isin/{isin}`

#### Success Scenario
- Pass-through of Finprim's master-data response. Anonymous endpoint — no JWT required.
- `/by-isin/{isin}` returns one scheme with `id`, `name`, `min_investment`, `gateway`, `plan_type`, `delivery_mode`, `investment_option`, etc.
- **Client next:** Use the scheme's `id` (NOT the ISIN directly) when calling `POST /mf/orders/purchases`.

#### Failure Scenarios

| HTTP | Code/Message | Cause | Retryable | Recovery |
|------|--------------|-------|-----------|----------|
| 404 | (from Finprim) `"scheme not found"` | ISIN not in sandbox catalog (e.g. arbitrary `cybrillapoa/INFxxx`) | no | Pick a different fund. Recommend `INF084M01044` (HDFC Liquid Direct Growth) or `INF846K01164` for the sandbox |
| 502 | upstream timeout | Master-data service slow | yes | Cache last response, retry |

---

### 10.20 `POST /api/mf/orders/purchases` (lump-sum)

#### Success Scenario
- **Preconditions:** MFIA exists; primary bank set; KYC `successful`; scheme id valid for the MFIA's gateway; `amount ≥ scheme.min_investment`.
- **Response:** raw Finprim `mf_purchase` object inside `result`, state typically `created`.
- **Client next:** PATCH with consent (`state=confirmed`, OTP block) → `POST /pg/payments/netbanking`.
- **Backend side-effects:** Webhooks `mf_purchase.created` → `.confirmed` → `.submitted` → `.successful` will follow asynchronously.

#### Failure Scenarios

| HTTP | Code/Message | Cause | Retryable | Recovery |
|------|--------------|-------|-----------|----------|
| 400 | `"pan_kyc_incomplete"` (Finprim passthrough) | Cybrilla `cybrillapoa` gateway hasn't yet propagated the user's `kyc_compliant=true` flag from Finprim (sandbox propagation lag — 30 s to 5 min) | retry-after-fix | Re-simulate KYC successful via `/kyc/requests/{id}/simulate`, wait 60 s, retry purchase |
| 400 | `"amount below min_investment"` | Below scheme floor | retry-after-fix | Increase amount |
| 401 | Unauthorized | JWT | retry-after-fix | Refresh |
| 404 | `"scheme not found"` (gateway mismatch e.g. `cybrillapoa/INFxxx`) | ISIN not in sandbox catalog | no | Use `INF084M01044` or `INF846K01164` |
| 502 | middleware-wrapped | Finprim downstream timeout | yes | Retry after 5 s; after 3 failures show service-unavailable |

---

### 10.21 `POST /api/pg/payments/netbanking` and `GET /api/pg/payments/{id}`

#### Success Scenario
- **Response:** raw Finprim payment object with a `redirect_url` for BillDesk/netbanking.
- **Client next:** Open `redirect_url`; user completes net-banking on bank portal. Listen for `payment.success` / `payment.failed` webhook to update UI.

#### Failure Scenarios

| HTTP | Code/Message | Cause | Retryable | Recovery |
|------|--------------|-------|-----------|----------|
| 400 | Finprim validation | wrong `amc_order_ids` or `bank_account_id` (numeric Cybrilla ids — not Finprim strings) | retry-after-fix | Look up numeric ids before sending |
| 401 | Unauthorized | JWT | retry-after-fix | Refresh |
| 502 | gateway error | BillDesk down | yes | Retry; if persistent, switch to `pg/payments/nach` |

---

### 10.22 Webhook receiver — `POST /api/webhooks/finprim-events`

#### Success Scenario
- Receiver returns HTTP 200 with an empty body when an event is accepted and dispatched. Cybrilla treats any 2xx as "delivered" and will not retry.
- **Side effect:** `FinprimEventDispatcher.HandleAsync` upserts the local DB row for the corresponding resource (KYC request, MF purchase, mandate, etc.).

#### Failure Scenarios

| HTTP | Code/Message | Cause | Retryable | Recovery |
|------|--------------|-------|-----------|----------|
| 500 | dispatcher exception | Bug in our event handler | yes (Cybrilla auto-retries up to N) | Fix handler; replay event |
| 200 (deduped) | `EventId` already seen | Duplicate delivery from Cybrilla | n/a | Receiver returns 200 but **skips side effects**; idempotency enforced by `EventId UNIQUE` |

No client action ever — webhook deliveries are server-to-server.

---

## 11. Cross-cutting failure recipes

The following error patterns affect multiple endpoints and have one canonical recovery each.

### `pan_kyc_incomplete` (MF purchase 400)
Cybrilla's `cybrillapoa` gateway is not yet synced with Finprim's `kyc_compliant=true`. Recovery:
1. `POST /api/kyc/requests/{id}/simulate` with `{"status":"successful"}`.
2. Wait 30–60 s.
3. Retry `POST /api/mf/orders/purchases`.

### eSign 400 — `"must be in esign_required state"`
With the current backend this should not occur in the normal flow: `POST /kyc/requests/{id}/signature` now auto-PATCHes `pep_details="not_applicable"` and `tax_residency_other_than_india=false` alongside `signature` (`KycController.UploadSignature` lines 464–471), and `PATCH /kyc/requests/{id}/financial` auto-defaults the same two fields if omitted (lines 258–268). If this error still appears, the user genuinely skipped both the DigiLocker / address-confirm step and the signature upload — send them back to step 6 (PATCH `/address` → confirm) and step 8 (`POST /signature`).

### Pre-verification `"Invalid investor identifier"`
Investor profile is missing PAN or PAN is lowercased. Recreate the investor profile ensuring uppercase PAN.

### Pre-verification `readiness.status=failed, code=unknown`
Sandbox propagation lag on `cybrillapoa`. The current pre-verification is **terminal** — never poll-retry the same `prv_xxx`. Re-simulate KYC, wait, then call `POST /api/banking/my-verify` again to create a brand new pre-verification.

### MF scheme 404 on `cybrillapoa/INFxxx`
ISIN not in sandbox catalog. Switch to `INF084M01044` or `INF846K01164`.

### KYC decay
Sandbox-only: `kyc_compliant` reverts from `true` to `submitted` after ~5 min. Detect via `GET /api/kyc/user-status` (`kycRequestStatus` flipped) and re-simulate via `/kyc/requests/{id}/simulate`.

### 502 Bad Gateway from any wrapper endpoint
Finprim downstream call failed via `ExceptionHandlingMiddleware`. Backoff 5 s, retry up to 3 ×. If all three fail, show "Service unavailable, please try again later" and surface an in-app reload action.

### 401 Unauthorized
JWT expired / missing / wrong audience. Refresh via your auth provider then retry. Do NOT call `api/auth/token` from clients — that returns the tenant Finprim token.

### Webhook receiver 500
Dispatcher exception. No client action — Cybrilla retries automatically.

### Webhook duplicate event
Receiver dedupes by `EventId` UNIQUE; the second delivery returns 200 with no DB write.

### `isWeb` field on outbound Finprim requests
Our wrapper strips it (`FinprimApiClient.cs:117`). Clients can send it freely without triggering Finprim's "unpermitted parameter" rejection.

### `bank_accounts: [{}]` placeholder in POST response
`POST /banking/my-verify` returns a Cybrilla skeleton — **never** treat it as the final verdict. Always poll `GET /banking/my-verify/{id}`.

### CORS from `file://` origin
`Program.cs:81` uses `SetIsOriginAllowed(_ => true)` so dev tools and mobile WebViews work. If you see a CORS block in dev, it is not a server-policy issue — most likely the request was sent to a non-`/api` path or the JWT preflight failed.

---

## 12. Wizard Step Walkthrough — User Inputs vs API Payloads

This section maps each onboarding wizard step (as actually implemented in the Angular app) to the form fields the user fills in and the exact API calls that fire. Step order mirrors `visibleSteps` in `mutual-fund-kyc.component.ts:1883-1891` and `steps` in `onboarding.component.ts:782-791`, followed by post-onboarding flows (scheme select / purchase / payment).

> Source files referenced:
> - `F:\github projects\IslamiclyWebLatest\src\app\features\mutual-funds\mutual-fund-kyc\mutual-fund-kyc.component.ts`
> - `F:\github projects\IslamiclyWebLatest\src\app\features\mutual-funds\onboarding\onboarding.component.ts`
> - `F:\github projects\IslamiclyWebLatest\src\app\services\kyc.service.ts`

### 12.1 PAN Check (KYC eligibility)

**User screen:** Enter PAN + DOB to look up existing KRA record and decide create / modify / disallowed.

**User inputs (collected from the form):**
| Field on UI | Form control | Validation / Allowed values | Maps to API field |
|---|---|---|---|
| PAN | `formData.pan` (text) | 10 chars, uppercased before send | `pan` |
| Date of Birth | `formData.dob` (date) | `YYYY-MM-DD` | `date_of_birth` |

**API call(s):**
- `POST /api/kyc/checks` (see Section 3.1) via `kycService.checkKyc()` — `kyc.service.ts:169-175`

```json
{ "pan": "ABCDE1234F", "date_of_birth": "1990-05-15" }
```

On success the response's `entity_details` pre-fills `name`, `gender`, `dob`, `email`, `mobile`, and the `correspondence_address` block on the form (`mutual-fund-kyc.component.ts:2274-2288`).

**Next state:** If `status=true` → jump to `success`. If `action=create|modify|unknown` → advance to **Personal Details**. Otherwise the scenario card blocks progression.

Source: `mutual-fund-kyc.component.ts:2261-2304`.

---

### 12.2 Personal Details (creates / updates KYC Request)

**User screen:** Confirm name + identity, marital + parent details, optional tax residency outside India.

**User inputs (collected from the form):**
| Field on UI | Form control | Validation / Allowed values | Maps to API field |
|---|---|---|---|
| Full name | `formData.name` | required | `name` |
| Gender | `formData.gender` | `Male` \| `Female` \| `Other` → mapped via `mapGender()` to `male`/`female`/`others` | `gender` |
| Date of Birth | `formData.dob` | from PAN check | `date_of_birth` |
| Mobile | `formData.mobile` | 10-digit | `mobile.number` (isd hard-coded `+91`) |
| Email | `formData.email` | email | `email` |
| Marital status | `formData.maritalStatus` | `unmarried` \| `married` \| `divorced` \| `widowed` | `marital_status` |
| Father's name | `formData.fatherName` | optional | `father_name` |
| Mother's name | `formData.motherName` | optional | `mother_name` |
| Aadhaar last 4 | `formData.aadhaarLast4` | 4 digits, **local-only**, sent as query param `aadhaarLast4` | not in body |
| Place of birth | `formData.placeOfBirth` | default `India` | `place_of_birth` |
| Tax residency outside India | `formData.taxResidencyOtherThanIndia` | boolean | `tax_residency_other_than_india` |
| Non-Indian tax res. 1/2/3 country + tax id | `formData.nonIndianTaxResidency{1..3}Country/TaxId` | only when toggle on | `non_indian_tax_residency_1/2/3` |

Hard-coded server-side defaults applied in the component:
`residential_status='resident_individual'`, `citizenship_countries=['in']`, `nationality_country='in'`, `country_of_birth='in'`, `pep_details='not_applicable'`.

**API call(s):**
- First time: `POST /api/kyc/requests?wizardStep=address&aadhaarLast4=<4digits>` (Section 3.3)
- Resume: `PATCH /api/kyc/requests/{id}?wizardStep=address` (Section 3.4)

```json
{
  "name": "Mohammed Arif Khan",
  "pan": "ABCDE1234F",
  "date_of_birth": "1990-05-15",
  "email": "arif@example.com",
  "mobile": { "isd": "+91", "number": "9876543210" },
  "gender": "male",
  "marital_status": "unmarried",
  "father_name": "Abdul Rahman Khan",
  "mother_name": "Fatima Khan",
  "residential_status": "resident_individual",
  "citizenship_countries": ["in"],
  "nationality_country": "in",
  "country_of_birth": "in",
  "place_of_birth": "India",
  "pep_details": "not_applicable",
  "tax_residency_other_than_india": false
}
```

**Next state:** `kycRequestId` is stored in component signal; wizard advances to **Address**.

Source: `mutual-fund-kyc.component.ts:2407-2458`.

---

### 12.3 Address / DigiLocker (Aadhaar Offline KYC)

**User screen:** Capture browser geolocation, then open DigiLocker in a new tab to pull Aadhaar address; return and confirm.

**User inputs:**
| Field on UI | Form control | Validation / Allowed values | Maps to API field |
|---|---|---|---|
| Geolocation | `capturedGeo` signal (navigator.geolocation) | optional | `latitude`, `longitude` |
| DigiLocker completion (implicit) | `digiLockerOpened` signal | user clicks Confirm after redirect | drives sequence |

**API call(s):**
1. `PATCH /api/kyc/requests/{id}/address` (Section 3.6) — `kycService.updateKycAddress()` — `kyc.service.ts:200-209`

```json
{ "latitude": 19.076, "longitude": 72.8777 }
```

Response: `{ "identityDocId": "...", "redirectUrl": "https://..." }` — `redirectUrl` is opened in a new tab.

2. After user returns: `POST /api/kyc/requests/{id}/address/confirm` (Section 3.7) — `kycService.confirmKycAddress()` — `kyc.service.ts:212-220`

```json
{ "identity_doc_id": "iddoc_..." }
```

**Next state:** Advance to **Financial Profile**.

Source: `mutual-fund-kyc.component.ts:2476-2519`.

---

### 12.4 Financial Profile

**User screen:** Pick occupation + annual income slab.

**User inputs:**
| Field on UI | Form control | Validation / Allowed values | Maps to API field |
|---|---|---|---|
| Occupation | `formData.occupation` | enum from `occupations` (`business`, `professional`, `retired`, `housewife`, `student`, `public_sector`, `private_sector`, `government_sector`, `others`) | `occupation_type` |
| Income slab | `formData.income` | enum from `incomeSlabs` (`upto_1lakh`, `above_1lakh_upto_5lakh`, …, `above_1cr`) | `income_slab` |

**API call(s):**
- `PATCH /api/kyc/requests/{id}/financial` (Section 3.5) — `kycService.updateKycFinancial()` — `kyc.service.ts:245-250`

```json
{ "occupation_type": "private_sector", "income_slab": "above_5lakh_upto_10lakh" }
```

Backend auto-defaults `pep_details` and `tax_residency_other_than_india` if missing (see Section 3.5 notes).

**Next state:** Advance to **Documents (Signature + eSign)**.

Source: `mutual-fund-kyc.component.ts:2601-2622` (occupation/income passed in), `incomeSlabs/occupations` arrays `:1945-1964`.

---

### 12.5 Documents — Signature Upload + Aadhaar eSign

**User screen:** Upload signature image, then perform Aadhaar OTP eSign in a popup.

**User inputs:**
| Field on UI | Form control | Validation / Allowed values | Maps to API field |
|---|---|---|---|
| Signature image | `signatureFile` signal (file input) | `image/jpeg, image/png, image/jpg` | multipart `file` |
| eSign completion (implicit) | `eSignOpened` signal | user returns from redirect | drives sequence |

**API call(s):**
1. `POST /api/kyc/requests/{id}/signature` (Section 3.8) — `kycService.uploadSignature()` — `kyc.service.ts:223-227`. `multipart/form-data` with `file` part. Backend auto-PATCHes `pep_details` + tax-residency defaults along with the signature.
2. `POST /api/kyc/requests/{id}/esign` (Section 3.9) — returns `{ esignId, redirectUrl }`; client opens the redirect URL in a new tab.
3. After return: `POST /api/kyc/requests/{id}/esign/confirm` (Section 3.10) with `{ "esign_id": "..." }`.

**Next state:** KYC request transitions to `submitted`; wizard jumps to `success` page and starts polling (`startStatusPolling()`). Bank + nominee steps in this wizard are deferred to the **post-KYC onboarding wizard** (`/admin/mutual-funds/onboarding`).

Source: `mutual-fund-kyc.component.ts:2522-2589`.

---

### 12.6 Investor Profile creation (auto-triggered after KYC success)

Not a user-facing wizard step — fired by the backend / status-poll handler once the KYC request becomes `successful`. The frontend then redirects the user to the onboarding wizard.

**API call(s):**
- `POST /api/investor-profiles/my-profile` (Section 3.12) — payload built from KYC data via `CreateInvestorProfilePayload` (`kyc.service.ts:377-392`):

```json
{
  "type": "individual",
  "tax_status": "resident_individual",
  "name": "Mohammed Arif Khan",
  "date_of_birth": "1990-05-15",
  "gender": "male",
  "occupation": "private_sector_service",
  "pan": "ABCDE1234F",
  "country_of_birth": "in",
  "place_of_birth": "India",
  "nationality_country": "in",
  "source_of_wealth": "salary",
  "income_slab": "5_10_lac",
  "pep_details": "not_applicable"
}
```

**Next state:** `investorProfileId` is persisted; onboarding wizard becomes available.

---

### 12.7 Onboarding — Address

**User screen:** Postal address for the investor profile.

**User inputs:**
| Field on UI | Form control | Validation / Allowed values | Maps to API field |
|---|---|---|---|
| Address Line 1 | `addrForm.line1` | required | `line1` |
| Address Line 2 | `addrForm.line2` | optional | `line2` |
| Postal Code | `addrForm.postalCode` | required | `postal_code` |
| Country | `addrForm.country` | default `IN` | `country` |
| Nature | `addrForm.nature` | `residential` \| `correspondence` | `nature` |

**API call(s):**
- `POST /api/onboarding/address` (Section 3.13) — `kycService.createAddress()`.

```json
{ "profile": "ip_...", "line1": "12A Marine Drive", "postal_code": "400020", "country": "IN", "nature": "residential" }
```

**Next state:** `phone`.

Source: `onboarding.component.ts:979-998`.

---

### 12.8 Onboarding — Phone

**User inputs:**
| Field on UI | Form control | Validation / Allowed values | Maps to API field |
|---|---|---|---|
| ISD code | `phoneForm.isd` | default `91` | `isd` |
| Number | `phoneForm.number` | required, 10-digit | `number` |

**API call:** `POST /api/onboarding/phone` with `{ profile, isd, number }`.

**Next state:** `email`. Source: `onboarding.component.ts:1000-1015`.

---

### 12.9 Onboarding — Email

| Field | Form control | Validation | API field |
|---|---|---|---|
| Email | `emailForm.email` | required | `email` |

**API call:** `POST /api/onboarding/email` with `{ profile, email }`.

**Next state:** `bank`. Source: `onboarding.component.ts:1017-1031`.

---

### 12.10 Onboarding — Bank Account (create + penny-drop)

**User screen:** Bank account form, then auto-runs penny-drop verification with polling.

**User inputs:**
| Field on UI | Form control | Validation / Allowed values | Maps to API field |
|---|---|---|---|
| Account holder name | `bankForm.holderName` | required | `primary_account_holder_name` |
| Account number | `bankForm.accountNumber` | required | `account_number` |
| Confirm account number | `bankForm.confirmAccountNumber` | must match `accountNumber` | (validation only) |
| Account type | `bankForm.accountType` | `savings` \| `current` \| `nre` \| `nro` | `type` |
| IFSC | `bankForm.ifsc` | 11 chars, uppercased | `ifsc_code` |

**API call(s):**
1. `POST /api/banking/my-account` (Section 3.14) — creates the bank account.
2. `POST /api/banking/my-verify` (Section 3.15) — auto-triggered after step 1 succeeds, returns `verificationId`.
3. `GET /api/banking/my-verify/{verificationId}` — polled every 4 s up to 60 times (4 min cap).

**Next state:** On `status=completed` and `confidence != zero` → advance to `demat`. On `replaceBankMode` flow → call `POST /api/banking/set-primary/{bankAccountId}` and bounce back to KYC page.

Source: `onboarding.component.ts:1033-1147`.

---

### 12.11 Onboarding — Demat (optional)

| Field on UI | Form control | Validation | Maps to API field |
|---|---|---|---|
| DP ID | `dematForm.dpId` | required if not skipped | `dp_id` |
| Client ID | `dematForm.clientId` | required if not skipped | `client_id` |

**API call:** `POST /api/onboarding/demat` with `{ profile, dp_id, client_id }`.

**Next state:** `fatca`. Skip button sets `skippedDemat=true` and jumps to `fatca`.

Source: `onboarding.component.ts:1164-1185`.

---

### 12.12 Onboarding — FATCA / CRS

**User inputs:**
| Field on UI | Form control | Validation / Allowed values | Maps to API field |
|---|---|---|---|
| PEP details | `fatcaForm.pepDetails` | `not_applicable` \| `pep` \| `related_to_pep` | `pep_details` |
| Tax residency outside India? | `fatcaForm.hasNonIndianResidency` | boolean | `has_non_indian_tax_residency` |
| Non-Indian country | `fatcaForm.nonIndianCountry` | required when toggle on | `non_indian_country` |
| Foreign tax ID type | `fatcaForm.nonIndianTaxIdType` | default `tin` | `non_indian_taxid_type` |
| Foreign tax ID number | `fatcaForm.nonIndianTaxIdNumber` | required when toggle on | `non_indian_taxid_number` |
| Applicable from | `fatcaForm.nonIndianApplicableFrom` | required when toggle on | `non_indian_applicable_from` |
| Applicable to | `fatcaForm.nonIndianApplicableTo` | default `2099-12-31` | `non_indian_applicable_to` |

**API call:** `POST /api/onboarding/fatca` (Section 3.13).

**Next state:** `mf-account` (skip path goes to `nominee` per UI order — but code advances to `mf-account` on success; `skipNominee()` also goes to `mf-account`).

Source: `onboarding.component.ts:1187-1211`.

---

### 12.13 Onboarding — Nominee (optional)

| Field on UI | Form control | Validation / Allowed values | Maps to API field |
|---|---|---|---|
| Name | `nomineeForm.name` | required | `name` |
| Date of birth | `nomineeForm.dateOfBirth` | optional | `date_of_birth` |
| Relationship | `nomineeForm.relationship` | `son` \| `daughter` \| `spouse` \| `father` \| `mother` \| `sibling` \| `other` | `relationship` |

**API call:** `POST /api/onboarding/nominee`.

**Next state:** `mf-account`. Source: `onboarding.component.ts:1218-1235`.

---

### 12.14 Onboarding — MF Investment Account

| Field on UI | Form control | Validation / Allowed values | Maps to API field |
|---|---|---|---|
| Holding pattern | `mfForm.holdingPattern` | `single` \| `joint` \| `anyone_or_survivor` (default `single`) | `holding_pattern` |
| Primary investor (hidden) | `getProfileId()` | system-provided | `primary_investor` |

**API call:** `POST /api/onboarding/mf-account` (Section 3.16).

```json
{ "primary_investor": "ip_...", "holding_pattern": "single" }
```

**Next state:** `complete` — onboarding wizard exits; user is routed to MF scheme search / purchase flow.

Source: `onboarding.component.ts:1237-1256`.

---

### 12.15 MF Scheme search / select

Not a wizard step in the KYC/onboarding component; performed on the funds discovery page after onboarding completes.

**API call(s):**
- `GET /api/masterdata/fund-schemes?q=<search>` (Section 3.17)
- `GET /api/masterdata/fund-schemes/by-isin/{isin}` (Section 3.17)

**User inputs:** search query, scheme selection.

**Next state:** Proceed to **Purchase**.

---

### 12.16 MF Purchase (lump-sum)

**User inputs (on the buy modal):**
| Field on UI | Form control | Validation / Allowed values | Maps to API field |
|---|---|---|---|
| Scheme | from search step | `isin` | `isin` |
| Amount | input | min ₹500 or scheme minimum | `amount.value` (`amount.currency='INR'`) |
| Folio (existing/new) | select | optional | `folio` |

**API call:** `POST /api/mf/orders/purchases` (Section 3.18).

```json
{ "mf_account": "mfia_...", "isin": "INF...", "amount": { "value": 5000, "currency": "INR" }, "folio": null }
```

**Next state:** Proceed to **Payment**.

---

### 12.17 Payment (Netbanking)

**User inputs:** select bank from the netbanking list (uses the user's primary bank from `set-primary`).

**API call:** `POST /api/pg/payments/netbanking` (Section 3.19) → returns gateway redirect URL. Poll `GET /api/pg/payments/{id}` for terminal state.

**Next state:** `complete` — investment confirmed.

---

## 13. Reusable Claude Code Prompts

Copy-paste-ready prompts for Claude Code agents working in this repo. Each prompt names the exact files/paths the agent should read first so it works even for newcomers.

### 13.1 Add a new KYC field to the wizard end-to-end

**When to use:** A new field (e.g. `place_of_work`) must be collected on the Personal step and persisted both to Finprim and our DB.

```
Add a new KYC field `place_of_work` (string, optional) to the KYC Request flow, end to end.

Read first (in this order):
1. F:\github projects\IslamiclyWebLatest\src\app\features\mutual-funds\mutual-fund-kyc\mutual-fund-kyc.component.ts (search "formData", "createKycRequestFromPersonal", "@if (currentStep() === 'personal')")
2. F:\github projects\IslamiclyWebLatest\src\app\services\kyc.service.ts (interface KycRequestData)
3. F:\github projects\IslamiclyAPI\Controllers\KycController.cs
4. F:\github projects\IslamiclyAPI\Services\KycService.cs (UpsertKycRequestAsync)
5. F:\github projects\IslamiclyAPI\Models\Kyc\KycModels.cs (CreateKycRequestRequest)
6. F:\github projects\FinprimApi\Helpers (mapping helpers)
7. c:\Users\Flyremit Pvt Ltd\Desktop\OnboardingMF\API_Integration_Guide.md (Sections 3.3, 3.4, 12.2)

Changes required:
- Angular: add input control to the personal step template, extend `formData`, pass through `createKycRequestFromPersonal` payload, extend `KycRequestData` interface.
- Backend: extend `CreateKycRequestRequest` DTO, forward in `KycService.UpsertKycRequestAsync`, persist to `tbl_Fintech_KYC_Request` (add column if missing — produce the migration SQL).
- Update API_Integration_Guide.md Sections 3.3, 3.4 (field table) and 12.2 (UI→API mapping table).
- Do NOT touch any unrelated wizard steps.

Output a diff per file plus the SQL migration.
```

---

### 13.2 Diagnose why a KYC request is stuck

```
A user's KYC request is stuck. Given KycRequestId = "<KYC_REQUEST_ID>", do this:

1. Read F:\github projects\IslamiclyAPI\Controllers\KycController.cs and F:\github projects\IslamiclyAPI\Services\KycService.cs to find the polling endpoint.
2. Fetch the live state from Finprim via `GET /api/kyc/requests/{id}/status`.
3. Fetch our DB row for that id from `tbl_Fintech_KYC_Request`.
4. Compare to the field list in API_Integration_Guide.md Section 3.3.
5. Identify which Finprim `fields_needed[]` are missing and which of our PATCH endpoints (Section 3.4, 3.5, 3.6, 3.8, 3.9) should be called.
6. Output a one-line fix recipe: e.g. "PATCH /kyc/requests/{id}/financial with income_slab=above_5lakh_upto_10lakh".

Do not modify any code.
```

---

### 13.3 Wire up a new Finprim webhook event type

```
Add support for a new Finprim webhook event named `<event_name>`.

Read first:
1. F:\github projects\IslamiclyAPI\Services\FinprimEventCatalog.cs (or equivalent — grep for "event_catalog" / "FinprimEvent")
2. F:\github projects\IslamiclyAPI\Controllers\WebhooksController.cs
3. F:\github projects\IslamiclyAPI\Services (webhook dispatcher)
4. c:\Users\Flyremit Pvt Ltd\Desktop\OnboardingMF\API_Integration_Guide.md Section 5 (Webhooks)

Changes:
- Register `<event_name>` in FinprimEventCatalog.
- Add a dispatcher case that loads/updates the right DB row.
- If new columns are needed, produce the SQL migration.
- Append a row to the event catalog table in API_Integration_Guide.md Section 5.

Make webhook handling idempotent — re-check Section 5's "Webhook duplicate event" guidance.
```

---

### 13.4 Add a new payment gateway / partner

```
Wire up a new payment gateway named `<gateway>` alongside the existing netbanking flow.

Read first:
1. F:\github projects\IslamiclyAPI\Controllers (search "Payment" or "Pg")
2. F:\github projects\IslamiclyAPI\Services (payment service)
3. F:\github projects\IslamiclyAPI\appsettings.json
4. API_Integration_Guide.md Section 3.19, Section 8 (appsettings keys)

Deliver:
- New controller actions (initiate + callback + poll) mirroring `POST /api/pg/payments/netbanking`.
- New service class behind an interface, registered in `Program.cs`.
- New appsettings keys (BaseUrl, MerchantId, Secret) — list them and the env-var mapping.
- DB tables for transaction logs + webhook events — produce SQL.
- Append a new Section 3.20 to API_Integration_Guide.md with request/response shapes and error cases.

Do NOT remove or modify the netbanking flow.
```

---

### 13.5 Generate an end-to-end Postman collection for a new flow

```
Produce a Postman v2.1 collection JSON for the full onboarding-to-purchase flow described in API_Integration_Guide.md Section 2 and Sections 12.1–12.17.

Source of truth:
- c:\Users\Flyremit Pvt Ltd\Desktop\OnboardingMF\API_Integration_Guide.md (Sections 3.x for endpoint shapes, 6 for enums, 12 for sequencing)

Requirements:
- One request per endpoint, ordered to match Section 2's sequence.
- Use `{{baseUrl}}` and `{{jwt}}` collection variables.
- Capture `kycRequestId`, `identityDocId`, `esignId`, `investorProfileId`, `bankAccountId`, `verificationId`, `mfAccountId`, `paymentId` into collection variables via Postman test scripts.
- Use the sample bodies from each Section 3.x verbatim where present.
- Output only the JSON (no commentary).
```

---

### 13.6 Audit unused fields in the Angular wizard

```
Audit the KYC + onboarding wizards for form fields the UI collects but never sends to any API.

Read first:
1. F:\github projects\IslamiclyWebLatest\src\app\features\mutual-funds\mutual-fund-kyc\mutual-fund-kyc.component.ts (`formData`)
2. F:\github projects\IslamiclyWebLatest\src\app\features\mutual-funds\onboarding\onboarding.component.ts (`addrForm`, `phoneForm`, `emailForm`, `bankForm`, `dematForm`, `fatcaForm`, `nomineeForm`, `mfForm`)
3. F:\github projects\IslamiclyWebLatest\src\app\services\kyc.service.ts (every payload interface)
4. API_Integration_Guide.md Section 12 (canonical UI→API mapping)

For each form field:
- Identify which API field it maps to (per Section 12).
- If it has no destination, flag it as UNUSED.
- Produce a table: { component, formField, status: used|unused, mapsTo }.

Do not modify any code — output the report only.
```

---

### 13.7 Refresh API_Integration_Guide.md after backend changes

```
The backend (Controllers / Services / Models) changed. Refresh c:\Users\Flyremit Pvt Ltd\Desktop\OnboardingMF\API_Integration_Guide.md so it matches reality.

Steps:
1. Run `git diff main -- F:\github projects\IslamiclyAPI` and `git log -p --since="last week" -- F:\github projects\IslamiclyAPI\Controllers F:\github projects\IslamiclyAPI\Models F:\github projects\IslamiclyAPI\Services`.
2. For each touched controller action, find its Section 3.x in the guide and reconcile: path, method, request body, response, query params, error codes.
3. For each new/changed enum or model property, update Section 6 (enum reference) and the corresponding field table.
4. For removed endpoints, strike them from Sections 2, 3, 9, 10, 12.

Append a changelog entry at the bottom of the doc with bullet points of what changed. Do not rewrite unchanged sections.
```

---

### 13.8 Generate failure scenario tests

```
Generate failure-mode tests for endpoint `<METHOD /path>`.

Read first:
1. c:\Users\Flyremit Pvt Ltd\Desktop\OnboardingMF\API_Integration_Guide.md Section 10 (per-endpoint Success & Failure scenarios) and Section 11 (cross-cutting failure recipes).
2. The controller + service file in F:\github projects\IslamiclyAPI for that path.

Produce:
- xUnit integration tests (one per failure scenario) using `WebApplicationFactory<Program>`.
- Each test asserts: HTTP status code, response body shape, and DB side-effects (none expected on failure).
- Cover at minimum: 400 validation, 401 unauthorized, 404 not found, 409 state conflict, 502 upstream Finprim, plus any cross-cutting recipe in Section 11 that applies.
- Also produce equivalent Jest tests for the Angular service method that calls this endpoint, mocking HttpClient.

Output: two files — `<Endpoint>.IntegrationTests.cs` and `<service>.spec.ts`.
```

---

## 📝 Summary for LLM / downstream maintainers

A compressed map of "where do I put new code" for anyone (human or LLM) extending this integration. Every path below is rooted at `F:\github projects\FinprimApi\` unless explicitly an Angular path.

| Task | Where it goes |
|------|---------------|
| **Add a new endpoint** | New controller in `Controllers\{Resource}Controller.cs`, mirroring the layered pattern (Controller → Service → DbService → SP). Always decorate with `[Authorize]` and return `OkApi(...)` / `ErrorApi(...)`. |
| **Add request / response models** | `Models\{ResourceName}\{ResourceName}Models.cs`. PascalCase + `[JsonPropertyName("snake_case")]` to match Finprim wire format. |
| **Add a DB write** | `Data\{ResourceName}DbService.cs`. Each method = one stored proc call via `SqlConnection` + `Dapper`. |
| **Register a new service in DI** | `Program.cs` — add an `AddScoped<I{X}, {X}>()` (or `AddSingleton` for caches like `TenantTokenProvider`) next to the existing registrations. |
| **Add a new webhook event** | 1) Add the event key to `Services\FinprimEventCatalog.cs`. 2) Add a handler branch in `Services\FinprimEventDispatcher.cs`. 3) Re-run `POST /api/webhooks/bootstrap` so Finprim re-registers the subscription. |
| **Add a new stored procedure** | Write the `CREATE/ALTER PROCEDURE` body in `c:\Users\Flyremit Pvt Ltd\Desktop\OnboardingMF\Claude-db-queries.txt` (this is the canonical, version-controlled SP source), then add a wrapper method on the relevant `*DbService.cs`. |
| **Extend the onboarding wizard** | Read **Section 12** end-to-end. Add the new step using the same `currentStep` switch pattern in `mutual-fund-kyc.component.ts` / `onboarding.component.ts`. Resume support requires extending `/my-status` for the relevant resource (see Section 5.5 resume flow). |
| **Test endpoints manually** | Import the Postman collection at `F:\github projects\FinprimApi\postman\Finprim API Wrapper.postman_collection.json` — every endpoint is pre-wired with the JWT bearer var and example payloads. |
| **Run automated tests** | xUnit integration tests under `F:\github projects\FinprimApi\Tests\` (use `WebApplicationFactory<Program>`); Angular Jest specs under `src\app\**\*.spec.ts`. See **Section 13** for a Claude prompt template that generates both. |

When in doubt, **search this guide first** — most "where is X?" questions are answered by **Section 9** (file-path index) or **Section 12** (wizard walkthrough).

---

## 14. Lump-Sum MF Purchase Flow — End-to-End Reference

This section is the **single canonical map** of every API call, every DB write, and every state transition that happens between a logged-in user tapping "Invest" on a scheme and the units finally being allotted to their folio. Every other section in this guide is feeder material for what is described here.

**Order of operations (high level):**

```
0. Detect existing KYC ─▶ (skip wizard if compliant)
1. KYC wizard (create → DigiLocker → signature → eSign)
2. Pre-verification (PAN ✓ readiness ✓ KRA ✓)
3. Investor profile + onboarding children (address/phone/email/mf-account)
4. Bank account create + penny-drop verify
5. Set primary bank
6. MF Purchase order create
7. Send Consent OTP
8. Verify OTP (consent recorded on order)
9. Create Netbanking payment
10. Redirect to token_url → PG checkout
11. Payment-callback poll for terminal state
12. Webhook events backfill DB (mf_purchase.successful, payment.success)
```

Each sub-section below covers one step: **(a) request example, (b) success response, (c) failure modes, (d) DB persistence, (e) Order Log "Step" value, (f) Angular method, (g) backend controller line citation.**

---

### 14.1 Step 0 — Detect if KYC already exists (NEW)

**Why this matters:** Re-running the KYC wizard for a user whose PAN is already KRA-compliant produces a `409 Conflict` from Cybrilla and breaks the resume flow. The app must check *both* our local DB (have we already created a KYC Request?) and the upstream KRA registry (does this PAN exist there independently of our app?) before letting the user enter the wizard.

#### 14.1.1 `GET /api/kyc/user-status`

Returns the local DB snapshot for the logged-in user (rows from `tbl_Fintech_Kyc_Check` + `tbl_Fintech_Kyc_Request`).

```json
{
  "statusCode": 200,
  "message": "OK",
  "result": {
    "hasKycCheck":      true,
    "hasKycRequest":    true,
    "kycId":            "kyc_aaa",
    "kycRequestId":     "kr_xxx",
    "kycStatus":        false,
    "kycRequestStatus": "successful",
    "kycRequestWizardStep": "esign",
    "pan":              "ABCPX3751X",
    "name":             "RAVI KUMAR",
    "action":           "create",
    "reason":           null
  }
}
```

**Decision tree (executed by the mobile/web app immediately after login):**

```
GET /api/kyc/user-status
├── result.kycRequestStatus === "successful"          → SKIP wizard → /investor-profile
├── result.kycRequestStatus === "submitted"           → SKIP wizard → show "Under Review" screen
├── result.kycRequestStatus in (pending,esign_required)
│                                                    → RESUME wizard at result.kycRequestWizardStep
├── result.kycRequestStatus in (rejected, expired)    → show terminal error screen
└── hasKycRequest === false
    └── POST /api/pre-verifications/readiness         → see 14.1.2
```

**Backend:** `KycController.cs:111` (`GetUserKycStatus`).
**Frontend:** `kyc.service.ts → getUserKycStatus()`; consumed in `mutual-fund-kyc.component.ts:2510` (ngOnInit).

#### 14.1.2 `POST /api/pre-verifications/readiness`

Asks Cybrilla "does this PAN already have an active KYC at KRA?" — **call this only when our local DB has no KYC Request for the user** (i.e., `hasKycRequest === false`).

```http
POST /api/pre-verifications/readiness
Authorization: Bearer <jwt>
Content-Type: application/json

{ "pan": "ABCPX3751X" }
```

**Response — possible `nextAction` values:**

| `nextAction`           | Meaning                                                                 | UI behavior |
|------------------------|-------------------------------------------------------------------------|-------------|
| `run_pan_validation`   | PAN exists at KRA + is compliant; **our app should NOT create its own KYC.** Run PAN validation only to capture the user's PAN. | Skip wizard; go to investor profile. |
| `start_fresh_kyc`      | PAN not found at KRA. User must complete the full wizard.               | Open KYC wizard at PAN step. |
| `fix_at_kra`           | PAN exists at KRA but is in `on_hold` / `rejected`. User must fix at KRA (cannot self-serve here). | Render terminal "Fix at KRA" screen with KRA URL. |
| `retry_readiness`      | Transient upstream error.                                               | Show retry button → POST readiness again. |
| `poll`                 | A `pv_xxx` is in-flight (`status=pending`). Resume polling at `fpPreVerificationId`. | Render in-flight screen + start poll loop. |

**Mobile app pseudocode (Kotlin / Swift parallel):**

```kotlin
val s = api.getUserKycStatus()
when {
  s.kycRequestStatus == "successful" -> navigate(InvestorProfile)
  s.kycRequestStatus == "submitted"  -> navigate(KycUnderReview)
  s.kycRequestStatus in listOf("pending","esign_required") -> resumeWizard(s.kycRequestWizardStep)
  s.kycRequestStatus in listOf("rejected","expired")       -> navigate(KycTerminal(s.kycRequestStatus))
  else -> {
    val r = api.preVerificationReadiness(s.pan ?: askUserForPan())
    when (r.nextAction) {
      "run_pan_validation" -> { api.runPanValidation(); navigate(InvestorProfile) }
      "start_fresh_kyc"    -> navigate(KycWizard)
      "fix_at_kra"         -> navigate(FixAtKra(r.kraUrl))
      "retry_readiness"    -> showRetry()
      "poll"               -> startPolling(r.fpPreVerificationId)
    }
  }
}
```

**Backend:** `PreVerificationsController.cs` → `POST /readiness`. **Frontend:** `pre-verify.service.ts → readiness()` (consumed in `mutual-fund-kyc.component.ts:2498` `ngOnInit`).
**Order Log Step:** *(not logged here — readiness is pre-order)*.

---

### 14.2 Step 1 — KYC Wizard (create → DigiLocker → eSign)

Re-stated for completeness — full details for each endpoint live in **Section 10** (failure scenarios) and **Section 12** (wizard step walkthrough).

#### 14.2.1 `POST /api/kyc/checks`

Sandbox PAN patterns (Cybrilla):

| PAN pattern  | Outcome                                  |
|--------------|------------------------------------------|
| `XXXPX3751X` | `status:true, action:none` — compliant   |
| `XXXPX3752X` | `status:false, action:modify`            |
| `XXXPX3753X` | `status:false, action:create` (not found)|
| `XXXPX3754X` | `status:false, action:disallowed`        |

**Backend:** `KycController.cs:74`. **DB:** `tbl_Fintech_Kyc_Check`.

#### 14.2.2 `POST /api/kyc/requests`

Creates the KYC Request and persists wizard resume metadata.
**Backend:** `KycController.cs:152`. **DB:** `tbl_Fintech_Kyc_Request` (sets `WizardStep='personal'`).

#### 14.2.3 `PATCH /api/kyc/requests/{id}/address` → DigiLocker

Returns `redirect_url` + identity-document ID. Postback URL is config-driven (`Finprim:Postback:DigiLocker:Web` vs `Finprim:Postback:DigiLocker:App`). The web flow appends `?identity_document=...&status=successful|failed|pending|expired&aadhaar_last4=XXXX` to the callback URL.

**Backend:** `KycController.cs:302`. **DB:** updates `tbl_Fintech_Kyc_Request.WizardStep='address'`.

#### 14.2.4 `POST /api/kyc/requests/{id}/address/confirm`

Confirms DigiLocker success after the user returns. **Backend:** `KycController.cs:370`.

#### 14.2.5 `PATCH /api/kyc/requests/{id}/financial`

Auto-defaults `pep_details=not_applicable` and `tax_residency_other_than_india=false` when omitted by the client. **Backend:** `KycController.cs:248`.

#### 14.2.6 `POST /api/kyc/requests/{id}/signature`

Now **auto-PATCHes pep + tax fields** alongside the signature upload, so the request transitions cleanly to `esign_required` without a separate financial PATCH. **Backend:** `KycController.cs:448`.

#### 14.2.7 `POST /api/kyc/requests/{id}/esign`

Returns redirect_url; postback URL appends `?esign=completed&status=successful|failed|cancelled&esign=esign_xxx`.

**Backend:** `KycController.cs:487`. **Frontend method:** `mutual-fund-kyc.component.ts → startESign()` (with new "Opening eSign…" UI card, see Task 2).

#### 14.2.8 `POST /api/kyc/requests/{id}/esign/confirm`

**Backend:** `KycController.cs:536`.

#### 14.2.9 `POST /api/kyc/requests/{id}/simulate` (sandbox only)

```json
{ "action": "kyc_compliant" }
```

**Backend:** `KycController.cs:442`.

> ⚠️ **Sandbox decay caveat** — Cybrilla's sandbox reverts `successful` KYC Requests back to `submitted` after a few minutes (~3–10 min). **Re-simulate immediately before** running any downstream step (pre-verification, bank create, MF purchase) or the order will fall into `under_review`.

---

### 14.3 Step 2 — Pre-verification 3-Stage Flow

Already documented in **Section 5.5** — re-linked here for flow completeness.

- Stage A: PAN validation
- Stage B: Readiness (KRA check)
- Stage C: KRA bridge (creates `pv_xxx`, polled until terminal)

**DB:** `tbl_Fintech_Pre_Verification` (1 row per attempt, with `Status` enum: pending/successful/failed/expired).

---

### 14.4 Step 3 — Investor Profile + Onboarding Children

#### 14.4.1 `POST /api/investor-profiles/my-profile`

**Backend:** `InvestorProfilesController.cs`. **DB:** `tbl_Fintech_Investor_Profile` (`InvestorProfileId=invp_xxx`).

#### 14.4.2 `POST /api/onboarding/address`, `/phone`, `/email`

Create `addr_xxx`, `phon_xxx`, `ema_xxx` children — all required before `mf-account` can be created.

**Backend:** `OnboardingController.cs`. **DB:** `tbl_Fintech_Investor_Address` / `_Phone` / `_Email`.

#### 14.4.3 `POST /api/onboarding/mf-account`

Creates the MFIA (`mfia_xxx`) and **locks a primary bank** (`folio_defaults.payout_bank_account = bac_xxx`).

**Backend:** `OnboardingController.cs:469`. **DB:** `tbl_Fintech_MF_Investment_Account`.

---

### 14.5 Step 4 — Bank Account Create + Verify

#### 14.5.1 `POST /api/banking/my-account`

Creates `bac_xxx`. **NEW**: persists Cybrilla's numeric `old_id` as `OldId` in `tbl_Fintech_Bank_Account`. This is the integer ID required by the netbanking payment endpoint (see 14.9).

**Request:**
```json
{
  "account_number": "0123456789011193",
  "ifsc_code":      "HDFC0000001",
  "account_type":   "savings",
  "holder_name":    "RAVI KUMAR",
  "is_primary":     true
}
```

**Response (excerpt):**
```json
{ "id": "bac_bbb", "old_id": 90123, "is_primary": true }
```

**Backend:** `BankingController.cs:75` → `BankingService.UpsertBankAccount` → SP `Usp_Upsert_Fintech_Bank_Account` (writes `OldId`).
**Frontend:** `kyc.service.ts → createBankAccount()`.

#### 14.5.2 `POST /api/banking/my-verify`

Starts the penny-drop verification. Returns `verificationId` (`prv_xxx`).

**Backend:** `BankingController.cs:263`.

#### 14.5.3 `GET /api/banking/my-verify/{verificationId}` (poll)

Poll every ~4s until `status` is terminal. **NEW**: terminal result is now persisted to `tbl_Fintech_Bank_Account.VerificationStatus / VerificationConfidence / VerificationReason` so the wizard can render the verified card after a page reload.

**Sandbox test patterns** (Cybrilla — last 4 digits of account number drive the result):

| Account ends in | `confidence` | `status`     | When to use                  |
|-----------------|--------------|--------------|------------------------------|
| `1193`          | `very_high`  | `completed`  | Happy path                   |
| `1285`          | `high`       | `completed`  | Happy path (weaker match)    |
| `1355`          | `uncertain`  | `completed`  | Edge: ambiguous name match   |
| `1600`          | `zero`       | `completed`  | Edge: zero-confidence pass   |
| `3157`          | `zero`       | `failed`     | Force user to change bank    |
| `99XX`          | n/a          | `failed`     | Expiry / retry-eligible      |

**Backend:** `BankingController.cs:339`.

#### 14.5.4 `GET /api/banking/accounts/{bac_xxx}` (backfill helper)

Re-syncs a single bank row from Finprim — including `OldId`. Useful to backfill existing rows that were created before the `OldId` column existed.

**Backend:** `BankingController.cs:172`.

#### 14.5.5 `bank_account_id` semantics — the three different IDs

| ID type        | Format     | Where it lives                                      | Used by                          |
|----------------|------------|-----------------------------------------------------|----------------------------------|
| Our local PK   | `int`      | `tbl_Fintech_Bank_Account.Id`                       | Internal joins only              |
| Cybrilla ID    | `bac_xxx`  | `tbl_Fintech_Bank_Account.FpBankAccountId`          | Most Cybrilla resources          |
| Cybrilla old_id| `int`      | `tbl_Fintech_Bank_Account.OldId` *(new)*            | **Netbanking `bank_account_id`** |

> ⚠️ The netbanking endpoint (14.9) requires the **integer `old_id`**, NOT `bac_xxx` and NOT our local PK. Sending the wrong one returns `422 Unprocessable Entity — "bank_account_id is invalid"`.

---

### 14.6 Step 5 — Set Primary Bank (NEW)

#### 14.6.1 `POST /api/banking/set-primary/{bac_xxx}`

**Why this exists:** Cybrilla rejects payment from a non-primary bank with:

```
"Making payment using bank account bac_yyy is not allowed for order mfp_xxx"
```

So whenever the user adds a new bank for testing (or switches banks), this endpoint must be called before placing a purchase order. Side-effect: PATCHes `folio_defaults.payout_bank_account` on the MFIA.

```json
{
  "statusCode": 200,
  "result": { "mfAccountId": "mfia_ccc", "primaryBankAccountId": "bac_bbb" }
}
```

**Backend:** `BankingController.cs:220`. **DB:** updates `tbl_Fintech_Bank_Account.IsPrimary` + `tbl_Fintech_MF_Investment_Account.PayoutBankAccount`.

---

### 14.7 Step 6 — MF Purchase Order Create

#### 14.7.1 `POST /api/mf/orders/purchases`

```json
{
  "mf_investment_account": "mfia_ccc",
  "scheme":                "schm_abc",
  "amount":                { "value": 5000, "currency": "INR" },
  "folio_number":          null,
  "consent":               null
}
```

**Success response:**
```json
{
  "statusCode": 200,
  "result": {
    "id":           "mfp_xxx",
    "old_id":       12345,
    "state":        "pending",
    "scheme":       "schm_abc",
    "amount":       { "value": 5000, "currency": "INR" }
  }
}
```

**State outcomes:**

| `state`        | Meaning                                                                 | Next step                              |
|----------------|-------------------------------------------------------------------------|----------------------------------------|
| `pending`      | Ready for consent OTP + payment                                         | Step 7 (send-otp)                      |
| `under_review` | Cybrilla compliance flagged the order (KYC decay, mismatched bank etc.) | **No manual transition on ONDC.** Re-simulate KYC, create a new order. |

> ⚠️ **ONDC orders cannot be simulated.** Unlike BSE, the ONDC gateway has no `/simulate` endpoint — orders cannot be force-marked `successful`. Any "stuck" order on ONDC has to be replaced by a fresh `POST /purchases`.

**Backend:** `MfOrdersController.cs:55`.
**DB:**
- Snapshot row → `tbl_Fintech_MF_Purchase_Order` (one row per order; updated by webhooks).
- Audit row → `tbl_Fintech_MF_Order_Log` with `Step='create_purchase'`.

**Frontend:** `mf-purchase.service.ts → createPurchase()`.

---

### 14.8 Step 7 — Send Consent OTP (NEW)

#### 14.8.1 `POST /api/mf/orders/purchases/{id}/send-otp`

**Sandbox behavior:** returns the magic OTP `"123456"` in the response body so the test client can auto-fill it.

```json
{
  "statusCode": 200,
  "result": { "otp": "123456", "expiresInSeconds": 600 }
}
```

**Production behavior (TODO — not yet wired):** the wrapper will integrate SendGrid (email) + Twilio (SMS) and **not** return the OTP in the response. Today it persists the hash and forwards the magic OTP for sandbox testing.

**DB persistence — `tbl_Fintech_Purchase_Otp`:**

| Column         | Purpose                                                         |
|----------------|-----------------------------------------------------------------|
| `Id`           | PK                                                              |
| `UserId`       | FK                                                              |
| `MfPurchaseOrderId` | FK → `tbl_Fintech_MF_Purchase_Order.Id`                    |
| `OtpHash`      | SHA-256 hash (never stores plaintext)                           |
| `ExpiresAtUtc` | now + 10 min                                                    |
| `AttemptCount` | locked after 5 failed attempts                                  |
| `ConsumedAt`   | set on successful verify                                        |

On every `send-otp` the SP `Usp_Upsert_Fintech_Purchase_Otp` **expires older unconsumed rows for the same order** before inserting the new one (only one active OTP per order at a time).

**Backend:** `MfOrdersController.cs:101`. **Order Log Step:** `send_otp`.

---

### 14.9 Step 8 — Verify OTP + Record Consent (NEW)

#### 14.9.1 `PATCH /api/mf/orders/purchases`

```json
{
  "id": "mfp_xxx",
  "consent": {
    "email":     "user@x.com",
    "isd_code":  "+91",
    "mobile":    "9876543210",
    "otp":       "123456"
  }
}
```

**Sandbox:** the wrapper forwards the OTP as-is; Cybrilla accepts `123456`.

**Production (TODO):** the wrapper will verify the OTP against `tbl_Fintech_Purchase_Otp` (SHA-256 compare + expiry/attempt-count checks) **before** forwarding to Cybrilla. Local guard is wired but disabled until production OTP delivery is enabled.

**Failure modes:**

| HTTP | Message                                                       | Recovery                                |
|------|---------------------------------------------------------------|-----------------------------------------|
| 422  | `"order is not in pending state"`                             | Order went `under_review`. Create new order. |
| 422  | `"otp is invalid"` (production)                               | Re-send OTP                              |
| 423  | `"otp locked — too many attempts"` (production)               | Wait for next OTP cycle                  |

**Backend:** `MfOrdersController.cs:124`. **Order Log Step:** `verify_otp`.

---

### 14.10 Step 9 — Create Netbanking Payment (NEW)

#### 14.10.1 `POST /api/pg/payments/netbanking`

```json
{
  "amc_order_ids":         [12345],
  "bank_account_id":       90123,
  "provider_name":         "ONDC",
  "payment_postback_url":  "https://app.islamicly.com/app/mutual-funds/payment-callback"
}
```

**Critical field semantics:**

| Field                  | Type    | Source                                                                 |
|------------------------|---------|------------------------------------------------------------------------|
| `amc_order_ids`        | int[]   | `tbl_Fintech_MF_Purchase_Order.OldId` (NOT `mfp_xxx`)                  |
| `bank_account_id`      | int     | `tbl_Fintech_Bank_Account.OldId` (NOT `bac_xxx`, NOT our PK)           |
| `provider_name`        | string  | `"ONDC"` for the `islamiclyondc` tenant. **`"RAZORPAY"` is unconfigured** for our tenant — will 422. |
| `payment_postback_url` | string  | Where Cybrilla redirects after the user pays. Must be on our domain.   |

**Success response:**
```json
{
  "statusCode": 200,
  "result": {
    "id":        "pay_xxx",
    "token_url": "https://sandbox.fintechprimitives.com/checkout/...",
    "expires_at":"2026-05-21T12:34:56Z"
  }
}
```

`token_url` is **one-shot** and has a **5-minute TTL**. The frontend must redirect immediately — don't preview, cache, or open in a new tab.

**DB:** snapshot → `tbl_Fintech_Payment`; audit → `tbl_Fintech_MF_Order_Log` with `Step='create_netbanking_payment'`.

**Backend:** `PaymentsController.cs:34`. **Frontend:** `payments.service.ts → createNetbanking()`.

---

### 14.11 Step 10 — Redirect to `token_url`

```ts
// Angular
window.location.href = res.tokenUrl;   // ← do this synchronously
```

Cybrilla hosts the bank-selection + netbanking checkout UI. On completion (success/failure/cancel) the user is redirected to `payment_postback_url` with query params:

```
?payment_id=pay_xxx&status=success|failed|cancelled&order_id=mfp_xxx
```

---

### 14.12 Step 11 — Payment Callback Handling (TODO — NOT YET BUILT)

> 🚧 **Frontend TODO** — the route `/app/mutual-funds/payment-callback` does not exist yet. Required behavior:

1. Read `payment_id` + `order_id` from query params.
2. Poll `GET /api/mf/orders/purchases/{id}` every 5s for up to 2 minutes.
3. When `state` transitions to `successful`: show units allotted + folio number (from `tbl_Fintech_MF_Purchase_Order.AllottedUnits` / `FolioNumber`).
4. When `state === 'failed'`: show retry / contact-support CTA.

**Backend:** `MfOrdersController.cs:83` (`GET /purchases/{id}`).

---

### 14.13 Step 12 — Webhook Events Backfill DB

`POST /api/webhooks/finprim-events` is the single receiver. Two events matter for this flow:

| Event                  | DB write                                                                                                |
|------------------------|---------------------------------------------------------------------------------------------------------|
| `mf_purchase.successful` | `tbl_Fintech_MF_Purchase_Order` ← `AllottedUnits`, `AllottedNav`, `FolioNumber`, `State='successful'`, `SucceededAt` |
| `mf_purchase.failed`     | `tbl_Fintech_MF_Purchase_Order` ← `State='failed'`, `FailureReason`                                  |
| `payment.success`        | `tbl_Fintech_Payment` ← `State='success'`, `SucceededAt`                                              |
| `payment.failed`         | `tbl_Fintech_Payment` ← `State='failed'`, `FailureReason`                                             |

Every webhook also writes an audit row to `tbl_Fintech_MF_Order_Log` with `Step='webhook_<event_name>'`.

---

## 15. Database Tables — Audit + Snapshot Pattern

Every MF order touches **two parallel sets of tables**: a single-row **snapshot** that always reflects the current state of the resource, and a many-row **audit log** that captures every state change for forensic/debug use.

### 15.1 Why two tables?

| Aspect                 | Snapshot (`tbl_Fintech_MF_Purchase_Order`, `tbl_Fintech_Payment`) | Audit (`tbl_Fintech_MF_Order_Log`) |
|------------------------|-------------------------------------------------------------------|------------------------------------|
| Cardinality            | 1 row per order / payment                                         | Many rows per order                |
| Purpose                | Fast `WHERE State = 'pending'` queries; UI rendering              | Forensic timeline; debugging       |
| Write frequency        | Update-in-place per state change                                  | Insert-only                        |
| Used by                | App / UI / list endpoints                                         | Support / debugging / audits       |

### 15.2 The audit table — `tbl_Fintech_MF_Order_Log`

| Column          | Notes                                                                  |
|-----------------|------------------------------------------------------------------------|
| `Id`            | PK                                                                     |
| `UserId`        | FK                                                                     |
| `MfPurchaseOrderId` | FK → snapshot table (nullable for pre-order steps like readiness)  |
| `Step`          | enum: `create_purchase`, `send_otp`, `verify_otp`, `create_netbanking_payment`, `webhook_mf_purchase.successful`, etc. |
| `Status`        | `started` / `success` / `failure`                                      |
| `RequestJson`   | The request body we sent to Cybrilla (or to our own endpoint)          |
| `ResponseJson`  | The response body from Cybrilla (or the error body)                    |
| `CreatedAtUtc`  | server time                                                            |

### 15.3 Querying a full order timeline

```sql
SELECT  l.CreatedAtUtc, l.Step, l.Status, l.ResponseJson
FROM    tbl_Fintech_MF_Order_Log l
WHERE   l.MfPurchaseOrderId = (
          SELECT Id
          FROM   tbl_Fintech_MF_Purchase_Order
          WHERE  FpPurchaseOrderId = 'mfp_xxx'
        )
ORDER BY l.CreatedAtUtc ASC;
```

### 15.4 Querying current state across all orders

```sql
SELECT  p.FpPurchaseOrderId,
        p.State,
        p.AmountValue,
        p.FolioNumber,
        p.AllottedUnits,
        p.UpdatedAtUtc
FROM    tbl_Fintech_MF_Purchase_Order p
WHERE   p.UserId = @UserId
ORDER BY p.CreatedAtUtc DESC;
```

### 15.5 OTP table — `tbl_Fintech_Purchase_Otp`

| Column         | Purpose                                                                 |
|----------------|-------------------------------------------------------------------------|
| `OtpHash`      | SHA-256 of the OTP — **plaintext is never persisted**                   |
| `ExpiresAtUtc` | 10 min after `send-otp`                                                 |
| `AttemptCount` | incremented on each failed verify; **locked at 5 attempts**             |
| `ConsumedAt`   | set on successful verify; consumed rows are ignored on subsequent send-otp |

**SP behavior on send-otp:** `Usp_Upsert_Fintech_Purchase_Otp` first runs an UPDATE that sets `ExpiresAtUtc = SYSUTCDATETIME()` for all unconsumed rows for the same `MfPurchaseOrderId`, then inserts the new row. This guarantees **at most one active OTP per order**.

```sql
-- Inspect active OTPs (debug)
SELECT  o.MfPurchaseOrderId, o.ExpiresAtUtc, o.AttemptCount, o.ConsumedAt
FROM    tbl_Fintech_Purchase_Otp o
WHERE   o.UserId = @UserId
  AND   o.ConsumedAt IS NULL
  AND   o.ExpiresAtUtc > SYSUTCDATETIME();
```

### 15.6 Payment snapshot vs audit

The same pattern applies to payments:

- **Snapshot:** `tbl_Fintech_Payment` — one row per `pay_xxx`, updated by webhook events.
- **Audit:** `tbl_Fintech_MF_Order_Log` — steps `create_netbanking_payment` + `webhook_payment.success` etc.

To reconstruct a payment's full lifecycle:

```sql
SELECT  l.CreatedAtUtc, l.Step, l.Status, JSON_VALUE(l.ResponseJson,'$.state') AS State
FROM    tbl_Fintech_MF_Order_Log l
JOIN    tbl_Fintech_Payment       p ON p.Id = l.MfPurchaseOrderId
WHERE   p.FpPaymentId = 'pay_xxx'
ORDER BY l.CreatedAtUtc ASC;
```

---

## 16. Canonical Pre-KYC Decision Tree (App Team Q&A)

This section captures a Q&A with the mobile app team about how to decide
whether KYC is needed for a user and what flow to follow. The web app
already follows this exact flow — both web and app **MUST** stay in sync.

### 16.1 The two APIs and what they answer

| API | Purpose | Cost | When to call |
|---|---|---|---|
| `POST /api/kyc/checks` | Does this PAN already have a compliant KYC at any KRA, and if not, what action is next? | Heavy KRA lookup (1–3 sec) | ONCE on first PAN entry — canonical source of truth |
| `POST /api/pre-verifications/readiness` | Is the PAN healthy enough to even start KYC? (Aadhaar linked? PAN-Aadhaar seeded?) | Lightweight upstream sanity check | OPTIONAL — call before launching the full KYC wizard so we don't waste users on a 6-step wizard that fails at the end |
| `POST /api/pre-verifications/pan-validate` | Does the user's typed name + DOB actually match KRA records for this PAN? | Stage 2 (~2 sec) | OPTIONAL — call after PAN+Name+DOB entry, before submitting KYC |

These two APIs are **complementary**, not redundant. `kyc/checks` tells you
*whether* KYC is needed; pre-verification tells you *whether the data is
clean enough* to pass KYC before wasting steps.

### 16.2 `POST /api/kyc/checks` — response fields that matter

**Request body:** `{ pan, date_of_birth }`

| Response field | Value | Meaning | What to do |
|---|---|---|---|
| `status` | `true` | PAN already KRA-compliant | **Skip the entire KYC wizard.** Go straight to investor profile creation. |
| `status` | `false` | KYC needed — check `action` to know which sub-path |
| `action` | `create` | Never done KYC before | Run the full KYC wizard (PAN → personal → financial → DigiLocker → signature → eSign) |
| `action` | `modify` | KYC partially exists | Resume the wizard — fields the user already gave are pre-filled by the backend |
| `action` | `disallowed` | PAN is rejected/blocked at KRA | Show a hard error — don't proceed |
| `action` | `none` / `unknown` | Tenant doesn't have kyc_checks enabled | Treat as `create` (default to KYC wizard) |

### 16.3 `POST /api/pre-verifications/readiness` — Stage 1

**Request body:** `{ pan }`

Branch on the `nextAction` field:

| `nextAction` | Meaning | What to do |
|---|---|---|
| `poll` | Still processing | Call `GET /{id}/poll` every 2 sec until terminal |
| `run_pan_validation` | Readiness passed | Proceed to Stage 2 (PAN+Name+DOB validate) |
| `run_bank_validation` | Already verified end-to-end before | Skip to bank step |
| `start_fresh_kyc` | PAN exists but no KYC registered at KRA | Run the full KYC wizard |
| `fix_at_kra` | KYC hold/rejected at KRA | Show error — user must fix at KRA portal (CVL / NDML / CAMS) |
| `link_aadhaar_at_itd` | PAN not linked to Aadhaar | Show error CTA → incometax.gov.in |
| `retry_readiness` | Transient failure | Retry after a short delay |

### 16.4 `POST /api/pre-verifications/pan-validate` — Stage 2

**Request body:** `{ pan, name, dateOfBirth }`

| `nextAction` | Meaning | What to do |
|---|---|---|
| `run_bank_validation` | All three (PAN/Name/DOB) matched | Skip to bank step |
| `fix_pan` | PAN wrong per ITD | Re-prompt PAN |
| `fix_name` | Name mismatch | Re-prompt name |
| `fix_dob` | DOB mismatch | Re-prompt DOB |
| `link_aadhaar_at_itd` | PAN-Aadhaar not linked | External CTA to ITD portal |

### 16.4a Input validations at DOB/Name entry — enforce on APP too (2026-07-06)

These run at the **first point** we collect DOB/Name (readiness + pan-validate),
BEFORE any Cybrilla call. Backend enforces them (authoritative); the **app must
mirror them client-side** for immediate UX. Web already does.

**1. Age gate — investor must be 18+**
- Where: `POST /api/pre-verifications/readiness` (when a DOB is supplied) AND
  `POST /api/pre-verifications/pan-validate` (DOB required).
- Rule: parse `dateOfBirth`; reject if not a valid date, if in the future, or if age
  `< 18` today.
- On fail: **HTTP 400** with message
  `"You must be at least 18 years old to open a mutual-fund account. Please check your date of birth."`
- App: validate the DOB picker inline (block Continue + show the message) AND still
  handle the 400 from the API. Web helper: `isAdultDob(dob)` (age ≥ 18, not future).

**2. Full-name max length — 75 characters**
- Where: the KYC POA form "Full Name (as per PAN)" field (`kyc-form`), and any name
  input sent to `pan-validate` / kyc-form create.
- Rule: **max 75 chars** for the full-name field. Rationale: ITD/KRA cap a printed
  name at 75 total (25 each for First/Middle/Last). This is a single full-name input,
  so cap the whole field at 75.
- Web: `<input [maxlength]="NAME_MAX_LEN">` (NAME_MAX_LEN = 75) + a `createForm()`
  guard "Name must be 75 characters or fewer." + inline hint when at the limit.
- App: set the name field `maxLength = 75` and validate before submit.

> Change the limit in ONE place if business updates it (web: `NAME_MAX_LEN` const in
> `kyc-form.component.ts`). Keep web + app + any backend check in sync.

### 16.5 Stage 3 is NOT a separate API call

Bank validation (referred to as "Stage 3" in the pre-verification model)
is driven by `POST /api/banking/my-verify` (the penny-drop start) AFTER
the user adds a bank account. There is no `/pre-verifications/stage-3`
endpoint.

### 16.6 The canonical decision tree — same on web AND app

```
User enters PAN + DOB
        |
        v
[ 1. GET /api/kyc/user-status ]
        |
        +-- kycRequestStatus = 'successful'
        |     -> Skip everything below — go to investor profile creation
        |
        +-- kycRequestStatus = 'submitted'
        |     -> "Under review" screen — poll periodically
        |
        +-- kycRequestStatus = 'rejected'
        |     -> Show retry / fix screen
        |
        +-- hasKycRequest = false (no prior KYC in our app)
                |
                v
        [ 2. POST /api/kyc/checks { pan, date_of_birth } ]
                |
                +-- status = true
                |     -> PAN already KRA-compliant -> skip wizard -> investor profile
                |
                +-- action = disallowed
                |     -> Show hard error — stop
                |
                +-- status = false, action in (create | modify | none)
                          |
                          v
                [ 3. POST /api/pre-verifications/readiness { pan } ]
                          |
                          +-- fix_at_kra / link_aadhaar_at_itd
                          |     -> Show external-portal CTA — stop
                          |
                          +-- run_pan_validation (Stage 2 ready)
                          |     -> POST /pan-validate -> branch on its nextAction
                          |          - fix_pan / fix_name / fix_dob -> re-prompt
                          |          - run_bank_validation          -> proceed to KYC wizard
                          |
                          +-- start_fresh_kyc / retry_readiness / default
                                -> Proceed to KYC wizard
```

### 16.7 Web implementation reference

The web app already implements this exact flow:

| Step | File | Symbol |
|---|---|---|
| Step 1 (user-status) | `mutual-fund-kyc.component.ts` | `kycService.getUserKycStatus()` in `ngOnInit` |
| Step 2 (kyc/checks) | same file | `legacyKycCheck()` method |
| Step 3 (pre-verifications) | `pre-verification.service.ts` + same component | `runReadinessCheck()` + `handlePreVerifyAction()` |

The mobile app must call these in the same order and respond to the same
`status` / `action` / `nextAction` values. Any divergence will create
inconsistent UX between the two clients.

### 16.8 What's different between web and app today

- **Web** uses `?isWeb=1` and the existing `/admin/mutual-funds/kyc` page as the postback target for DigiLocker / eSign
- **App** uses `?isWeb=0` and lands on `/appmutualfund/kyc` / `/appmutualfund/esign` standalone Angular pages that render result + deep-link back to native
- **Both** go through the same backend controllers (`KycController`, `PreVerificationsController`) — the only difference is the postback URL the backend hands to Finprim

### 16.9 Important caveats

- **`pre-verifications` is not yet on UAT** (as of the app team's note). Gate the optional Stage 1+2 calls behind a feature flag if needed, but `kyc/checks` is always required.
- **`kyc/checks` may return `action: none`** on tenants where `/v2/kyc_checks` is disabled. Our wrapper catches the 404 and returns a synthetic `{ status: false, action: "create" }` so the client can proceed without special-casing.
- **Pre-verification is upstream advisory only.** Even if Stage 1 says `start_fresh_kyc`, the canonical answer about whether KYC is needed still comes from `kyc/checks`. Pre-verification just spares the user from a wasted wizard run.
- **Sandbox PAN patterns for `kyc/checks`** — same as Pre-Verification: `XXXPX3751X` returns `status: true` (compliant), `XXXPX3753X` returns `status: false` + `action: create` (no KYC found).

### 16.10 Summary — the rule

> **`kyc/checks` decides if KYC is needed. Pre-verification decides if the user's data is ready for KYC. Both should be called; both responses should be respected.**

The web app follows this. The mobile app must follow the same logic so
users have a consistent experience regardless of platform.

---

## 17. OTP Storage + Verification (Production-Ready Consent Gate)

Cybrilla's `PATCH /v2/mf_purchases` (#update-a-mf-purchase) consent payload
includes `email`, `isd_code`, `mobile`, `otp` — but **Cybrilla does not
generate or validate the OTP**. The platform (us) generates, delivers, and
verifies it. Per their docs, OTP must reach the primary holder on **both
mobile AND email** before consent is captured.

### Table — `tbl_Fintech_Purchase_Otp`

| Column          | Purpose                                                                |
|-----------------|------------------------------------------------------------------------|
| Id              | PK                                                                     |
| UserId          | App user                                                               |
| MfPurchaseId    | `mfp_xxx` from Cybrilla                                                |
| OtpHash         | SHA-256 hex of `otp + ":" + userId + ":" + mfPurchaseId` (salted)      |
| IsdCode/Mobile  | Where SMS was sent                                                     |
| Email           | Where mail was sent                                                    |
| ExpiresAt       | UTC, +10 min                                                           |
| ConsumedAt      | Set on successful verify OR after 5 failed attempts (locks the row)    |
| FailedAttempts  | Locks at 5                                                             |
| Channel         | `sms` / `email` / `both` / `sandbox`                                   |

Each new OTP for a `MfPurchaseId` auto-expires the previous unconsumed row,
so only one OTP is valid at a time.

### Endpoint flow

1. **`POST /api/mf/orders/purchases/{id}/send-otp`** body `{mobile,email,isdCode}`
   - Generates OTP (sandbox = `123456`; production = `RandomNumberGenerator`).
   - Hashes with salt, inserts via `Usp_Insert_Fintech_Purchase_Otp`.
   - Production: dispatches SMS (Twilio/MSG91) + email (SendGrid).
   - Logs to `tbl_Fintech_MF_Order_Log` (`step=send_otp`).

2. **`PATCH /api/mf/orders/purchases`** with `consent.otp`
   - Server hashes the submitted OTP and calls `Usp_Verify_Fintech_Purchase_Otp`.
   - On `expired` / `wrong` / `locked` / `no_otp` → returns **401 without
     calling Cybrilla**.
   - Only on `ok` does consent PATCH go upstream.

The client OTP is never trusted — the server re-hashes with the same salt
and compares against the DB row.

## 18. Folio Number Capture

`folio_number` is extracted by `MfPurchaseParser.FromJson` and stored in
`tbl_Fintech_MF_Purchase_Order.FolioNumber` on **every** upsert path:
- After `CreatePurchase` (usually null until allotment).
- After `UpdatePurchase` (consent + state transitions).
- Via the `mf_purchase.*` webhook dispatcher (canonical update — webhook
  fires after Cybrilla assigns the folio).

Verify with:
```sql
SELECT MFPurchaseId, FolioNumber, State, UpdatedAt
FROM tbl_Fintech_MF_Purchase_Order
WHERE UserId = @UserId
ORDER BY UpdatedAt DESC;
```

For lookups across folios, hit `GET /v2/folios` (Cybrilla #fetch-all-folios)
filtered by `mf_investment_account=<mfia_xxx>` — returns the registered
mobile/email per folio, the source of truth for who receives the consent
OTP on subsequent purchases under that folio.

---

## 19. Payments — Docs-Aligned Contract

Source pages (Cybrilla):
- `/payments/overview/`
- `/payments/payments-via-Netbanking-UPI/`
- `/payments/managing-eNACH/`
- `/payments/payment-via-eNACH/`
- `/payments/payment-errors/`

### 19.1 Methods × Providers

| Provider | Netbanking | UPI | eNACH | UPI Autopay |
|----------|:---------:|:---:|:-----:|:-----------:|
| Razorpay |     ✓     |  ✓  |   ✓   |      ✓      |
| BillDesk |     ✓     |  ✓  |   ✓   |      ✓      |
| **ONDC** |     ✓     |  ✓  |   ✗   |      ✗      |
| BSE      |     ✓     |  ✓  | SIPs  |      ✗      |

Our active tenant is **ONDC** — only Netbanking + UPI are available here. eNACH/autopay (i.e. recurring SIP debits) require switching to Razorpay/BillDesk or BSE.

### 19.2 Create payment — Netbanking / UPI (single FP endpoint)

**Important per FP docs:** ONE endpoint handles BOTH methods. The
`method` field discriminates between Netbanking and UPI:

```
POST /api/pg/payments/netbanking
```

(yes, the FP path is named `/netbanking` even for UPI — historical
naming. The `method` field tells FP which flow to run.)

The investor **picks** between them on the lump-sum Invest screen
(UPI default, recommended). Our backend wrapper exposes two routes
for clarity at the Angular call site, but both forward to the same
FP endpoint:

#### Netbanking
```
POST /api/pg/payments/netbanking
Body:
{
  "amc_order_ids":        [23],
  "payment_postback_url": "https://…/payment-callback?mfp=mfp_xxx",
  "method":               "NETBANKING",
  "bank_account_id":      925,        // FP numeric old_id (NOT bac_xxx, NOT our local Id)
  "provider_name":        "ONDC"
}
```

#### UPI
```
POST /api/pg/payments/upi   (our wrapper — forwards to the SAME FP path with method:UPI)
Body:
{
  "amc_order_ids":        [23],
  "payment_postback_url": "https://…/payment-callback?mfp=mfp_xxx",
  "method":               "UPI",
  "bank_account_id":      925,        // REQUIRED by ONDC runtime (docs are wrong)
  "provider_name":        "ONDC"
}
```

**Field reference (from FP docs):**

| Field | Mandatory | Notes |
|---|---|---|
| `amc_order_ids` | yes | List of FP `old_id` (numeric, not `mfp_xxx`) |
| `method` | yes (when provider is ONDC) | `NETBANKING` or `UPI` |
| `payment_postback_url` | no | Where FP/PG redirects after payment |
| `bank_account_id` | yes (runtime, despite docs marking it "no") | ONDC runtime returns 422 "Bank Account Id could not be null" if missing — even for UPI. Send the user's primary bank's FP `old_id` for both methods. |
| `provider_name` | no | `RAZORPAY`, `BILLDESK`, `ONDC` (we use ONDC) |

Both return the same response shape:
```json
{ "id": 12345, "token_url": "https://..." }
```

**Rules per docs (apply to both methods):**
- `token_url` is **single-use** and expires after **15 minutes**.
- ONDC additionally requires the MF order to be in **`confirmed`** state *before* the user uses the token. Hitting it while the order is `pending` / `under_review` returns "Invalid Token". Our wrapper prechecks this and returns 409 with `reason: order_terminal_state` if the order is already terminal.
- A failed payment cannot be retried — generate a brand-new payment + token (either method) for a retry.
- The **postback URL is method-agnostic** — `/app/mutual-funds/payment-callback` (web) and `/appmutualfund/payment` (app) handle both Netbanking and UPI outcomes generically. The page polls `/payments/{id}/snapshot` + `/purchases/{mfp}/snapshot` and renders success/failure based on the resolved DB state regardless of which method the user picked.

**UPI sandbox behaviour:** the `token_url` opens an FP/PG-hosted page accepting a dummy VPA + simulate-success/failure buttons. **Netbanking sandbox:** for ONDC tenant the `token_url` chains through HDFC ATOM UAT (`merchant.now.hdfc.bank.in/epi-ui/?uniqueKey=ATOMCAMSINTE_…`) — see §22.7 for the full URL chain.

**Production:** same endpoints, real UPI handoff or real bank netbanking page. App code is identical.

### 19.3 Payment state machine

```
PENDING ──► SUBMITTED ──► SUCCESS ──► INITIATED ──► APPROVED
   │             │            │
   └──► FAILED ◄─┴────────────┘
```

| State     | Meaning                                                     |
|-----------|-------------------------------------------------------------|
| PENDING   | Created; investor has not authorised at bank yet            |
| SUBMITTED | Forwarded to gateway (post-redirect)                        |
| SUCCESS   | Debited from investor; not yet transferred to AMC           |
| INITIATED | Transfer to AMC started                                     |
| APPROVED  | Transferred to AMC                                          |
| FAILED    | Terminal failure (use `failure_code` for the reason)        |

**Bug fix:** we previously set `Status = "INITIATED"` on create — that's wrong. On create the state is `PENDING`. Updated in `PaymentsController.CreateNetbanking` + `CreateNach`.

### 19.4 Failure codes (from `/payments/payment-errors/`)

Persist these verbatim into `tbl_Fintech_Payment.FailureCode`. UI maps them to recovery hints.

| Code                          | Trigger                                                      | Recovery                             |
|-------------------------------|--------------------------------------------------------------|--------------------------------------|
| `fp_payment_expired`          | User didn't complete in FP's window                          | Retry (new payment)                  |
| `pg_payment_expired`          | User didn't complete in netbanking/UPI app window            | Retry                                |
| `tpv_failed`                  | Debited from a different bank than the verified one          | Retry from the verified primary bank |
| `pg_timeout`                  | Gateway timeout                                              | Retry later                          |
| `invalid_upi_id`              | Bad/unregistered UPI ID                                      | Re-enter UPI ID                      |
| `bank_not_enabled`            | Investor's bank not on the gateway's allow-list              | Use a different bank                 |
| `investor_bank_declined`      | Insufficient balance / limit / bank decline                  | Change account or top-up             |
| `user_cancelled`              | User cancelled the flow                                      | Retry                                |
| `bank_downtime`               | UPI app / bank downtime                                      | Retry later                          |
| `insufficient_funds`          | Low balance                                                  | Top-up + retry                       |
| `upi_app_unavailable`         | UPI app issue                                                | Try another UPI app                  |
| `pg_server_error`             | Gateway downtime                                             | Retry later / support                |
| `auth_in_progress`            | Previous mandate authorisation still in progress             | Wait ≥ 15 min, then retry            |
| `invalid_credentials`         | Wrong bank login                                             | Re-enter credentials                 |
| `payment_failed`              | Generic bank/gateway failure                                 | Retry                                |
| `invalid_bank_account`        | Account closed/blocked                                       | Different account                    |
| `late_authorised_refunded`    | Bank authorised after we marked failed                       | Auto-refund 3–5 days; no action      |

### 19.5 eNACH (future — for SIPs / recurring)

Two-step setup (not yet wired on our side):

1. `POST /v2/e_nach_mandates` — `{ type: "E_MANDATE" | "UPI", bank_account_id, max_amount }`
2. `POST /v2/e_nach_mandates/{id}/authorize` — returns a `token_url` like a payment.

Mandate state machine: `CREATED → SUBMITTED → APPROVED` (or `REJECTED` / `CANCELLED`). One-time payments need `APPROVED`; SIPs can start with `CREATED` but the installment won't process unless the mandate is approved by the due date.

Payment via mandate: `POST /v2/payments` with `{ mandate_id, amc_order_ids }`. Multiple `amc_order_ids` can be bundled into one mandate-driven payment (unlike netbanking, which we currently keep 1-1 with an order).

eNACH is **not available on ONDC** — needs Razorpay/BillDesk/BSE.

### 19.6 Open items on our side

- [ ] Wire payment-postback handler that maps Cybrilla's `failure_code` into `tbl_Fintech_Payment.FailureCode` + `FailedReason` per §19.4.
- [x] **ONDC order sequence** (corrected): `consent PATCH → CREATE payment (while pending) → state=confirmed PATCH → FP submits to ONDC → state=submitted → user uses token`. Payment must be created **before** confirming the order, not after. Precheck only blocks terminal states (failed/cancelled/reversed/rejected).
- [ ] Add `mandate_id` model + `eNACH` endpoint family when we onboard a non-ONDC tenant.
- [ ] Map UI recovery hints from the failure-code table.

---

## 20. Lump-sum One-Time Purchase — Complete End-to-End Sequence (ONDC)

The official Cybrilla docs state explicitly for the ONDC gateway:

> When you create an order using Create MF Purchase API, the order is in `under_review` state.
> The gateway asynchronously reviews the order. If review passes → state becomes `pending` and `mf_purchase.review_completed` event fires. Else `failed`.
> Once in `pending`, consent can be obtained.
> Post consent → create payment.
> Post payment → PATCH state=`confirmed`.
> Confirmed → Cybrilla submits → state=`submitted` → redirect user to token_url to pay.

### 20.1 Exact request/response sequence (production-correct)

Every step the app must do, in order. Endpoint paths are our wrapper paths, not raw Cybrilla.

```
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1 — Create order                                               │
│ POST /api/mf/orders/purchases                                       │
│ Header:  Idempotency-Key: <uuid per Invest intent>                  │
│ Body:    { mf_investment_account, scheme (isin), amount, user_ip }  │
│ Returns: { id: mfp_xxx, old_id: <int>, state: under_review|pending, │
│            gateway: "ondc", ... }                                   │
│                                                                     │
│ ⚠ If state==under_review, wait for `mf_purchase.review_completed`   │
│   webhook before step 2. In practice review is <1s on sandbox so    │
│   the next call lands on `pending` directly.                        │
└─────────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 2 — Send OTP                                                   │
│ POST /api/mf/orders/purchases/{mfp_xxx}/send-otp                    │
│ Body:    {}    (server reads investor's mobile+email from KYC)      │
│ Returns: { sandboxOtp: "123456", sentTo: { mobile, email } }        │
│                                                                     │
│ Server-side: persists OtpHash + 10-min ExpiresAt in                 │
│ tbl_Fintech_Purchase_Otp. Production sends via Twilio + SendGrid.   │
└─────────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 3 — Verify OTP (peek-only, doesn't consume)                    │
│ POST /api/mf/orders/purchases/{mfp_xxx}/verify-otp                  │
│ Body:    { otp: "<user-entered>" }                                  │
│ Returns: 200 { valid: true }   |   422 { reason: wrong|expired|...} │
│                                                                     │
│ On 422 keep the OTP modal open and let the user retry. After 5      │
│ wrong attempts the OTP row is locked → must resend.                 │
└─────────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 4 — Apply consent                                              │
│ PATCH /api/mf/orders/purchases                                      │
│ Body:    {                                                          │
│            id: "mfp_xxx",                                           │
│            consent: { email, isd_code, mobile, otp }                │
│          }                                                          │
│                                                                     │
│ Server re-verifies OTP (consumes it), then forwards consent to      │
│ Cybrilla. Wrong/expired OTP → 422, no Cybrilla call.                │
│ Order state stays `pending` after this (consent doesn't auto-       │
│ progress on ONDC).                                                  │
└─────────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 5 — Create payment (while order still pending)                 │
│ POST /api/pg/payments/netbanking                                    │
│ Body:    {                                                          │
│            amc_order_ids:   [<old_id from step 1>],                 │
│            payment_postback_url: "https://your-domain/app/          │
│              mutual-funds/payment-callback?mfp=<mfp_xxx>",          │
│            method:          "NETBANKING",                           │
│            bank_account_id: <bank.old_id from kyc-bank>,            │
│            provider_name:   "ONDC"                                  │
│          }                                                          │
│ Returns: { id: <fp_payment_id>, token_url }                         │
│                                                                     │
│ ⚠ DON'T open token_url yet — order isn't `submitted` on Cybrilla.   │
│   Per docs, ONDC rejects token use until order is submitted.        │
└─────────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 6 — Confirm order                                              │
│ PATCH /api/mf/orders/purchases                                      │
│ Body:    { id: "mfp_xxx", state: "confirmed" }                      │
│                                                                     │
│ Cybrilla then auto-submits to ONDC → state moves                    │
│ confirmed → submitted within ~1s. Wait for the                      │
│ `mf_purchase.submitted` webhook OR poll                             │
│ GET /api/mf/orders/purchases/{mfp_xxx} until state=submitted.       │
└─────────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 7 — Open token_url (browser navigate)                          │
│ window.location.href = <token_url from step 5>                      │
│                                                                     │
│ User completes payment at bank's NetBanking page → bank POSTs to    │
│ payment_postback_url with paymentId/status/failureReason form data. │
└─────────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 8 — Land on /app/mutual-funds/payment-callback                 │
│ Query: ?id=<paymentId>&mfp=<mfp_xxx>                                │
│                                                                     │
│ Page polls every 2s:                                                │
│   GET /api/pg/payments/{id}/snapshot                                │
│   GET /api/mf/orders/purchases/{mfp_xxx}/snapshot                   │
│ Renders:                                                            │
│   payment.status=APPROVED → success card                            │
│   payment.status=FAILED   → failure card with hint per §19.4        │
│   order.state=successful  → success card + folio + units            │
│   order.state=submitted   → success card (units allotted T+1)       │
└─────────────────────────────────────────────────────────────────────┘
```

### 20.2 Idempotency

The frontend generates one UUID per "Invest intent" and sends it as
the `Idempotency-Key` HTTP header on POST `/api/mf/orders/purchases`.
If the same (UserId, IdempotencyKey) pair is replayed within 30
minutes, the server replays the original order's response instead of
creating a duplicate.

Cancelling the OTP modal clears the key so the next click starts a
fresh intent.

### 20.3 Sandbox simulation

On the ONDC sandbox the user can't actually authorize at HDFC. The
invest page detects `Runtime:IsProduction=0` and shows a Sandbox
Simulate modal after step 6 instead of step 7. The modal calls:
```
POST /api/pg/payments/{id}/simulate { "status": "success" }
POST /api/pg/payments/{id}/simulate { "status": "approved" }
```
Then redirects to the callback page exactly like production would.

For ONDC RTA allotment in sandbox, amounts ending in **0** auto-succeed
at the RTA via Cybrilla's cron; amounts ending in **1** auto-fail.
The mf_purchase simulate endpoint is **NOT available on ONDC** —
"ONDC gateway orders can't be simulated".

---

## 21. SIP / Recurring Plan — Complete End-to-End Sequence (ONDC)

### 21.0 SIP FIELD BINDING — frequencies, min/max amount, allowed dates, installments (READ FIRST — app team)

> **App team's exact question:** how to bind the SIP form fields — supported
> frequencies, min/max amount **per frequency**, allowed installment dates, min
> installments — and when to show what. **Answer: everything is DATA-DRIVEN from the
> scheme's `thresholds[]`. Do NOT hardcode any of these — read them per-scheme from the
> API below.** This mirrors the web frontend exactly (`mutual-fund-invest.component.ts`).

#### Where the data comes from (ONE call)
```
GET {mfurl}masterdata/fund-schemes-v2/ONDC/{ISIN}
→ { ...scheme, thresholds: MfSchemeThreshold[] }
```
Example screen: `/app/mutual-funds/invest/INF209K01RU9` → binds from this response.

Web reads the response's `result` (or the object itself) and uses `thresholds`.

#### The `thresholds[]` row shape (bind from THIS — do not invent values)
```ts
interface MfSchemeThreshold {
  type: 'lumpsum' | 'sip' | 'withdrawal';   // filter type==='sip' for SIP
  frequency?: string;      // sip only: 'monthly' | 'daily' | 'calendar_day_daily' | 'quarterly' | ...
  amount_min?: number;     // min SIP amount FOR THIS FREQUENCY
  amount_max?: number;     // max (may be a huge sentinel = "no limit", see below)
  amount_multiples?: number;   // amount must be a multiple of this (usually 1)
  installments_min?: number;   // min number_of_installments FOR THIS FREQUENCY
  dates?: number[];        // ALLOWED installment days (1..28). Empty/missing = all 1..28 allowed
  // additional_amount_* / units_* also present — not used for SIP create
}
```

#### 1. Supported frequencies — from the data, not a fixed list
- Take every `thresholds` row where `type==='sip'` and `frequency` is set. The DISTINCT
  `frequency` values are exactly the frequencies this scheme supports. **If a frequency
  isn't in `thresholds`, the AMC/gateway will reject it — don't offer it.**
- Order to display (web): monthly first, then dailies, then the rest.
- Friendly labels the web uses (map `frequency` → label):
  | `frequency` | Label |
  |---|---|
  | `monthly` | Monthly |
  | `daily` | Daily — Business Days only (skips weekends/holidays) |
  | `calendar_day_daily` | Daily — Every Calendar Day |
  | `day_in_a_week` | Weekly |
  | `day_in_a_fortnight` | Fortnightly |
  | `twice_a_month` | Twice a month |
  | `four_times_a_month` | Four times a month |
  | `quarterly` | Quarterly |
  | `half_yearly` | Half-yearly |
  | `yearly` | Yearly |
  (Unknown value → show the raw frequency string.)

#### 2. Min / Max amount — PER SELECTED FREQUENCY (re-read on frequency change)
- On frequency select, pick the `thresholds` row for that frequency. Bind:
  - **min** = `amount_min` (fallback 500 if absent)
  - **max** = `amount_max` (fallback 999999999)
  - **multiple** = `amount_multiples` (fallback 1)
- Daily SIPs often have a much lower min (e.g. ₹40) than monthly (e.g. ₹500) — that's why
  you MUST re-read per frequency, not once.
- **"No limit":** gateways return a huge sentinel for max. If `amount_max >= 100000000`
  (10 crore), show **"No limit"** instead of the 9-digit number.

#### 3. Allowed installment dates (monthly-style frequencies)
- `dates?: number[]` on the threshold row = the ONLY days (1..28) the scheme allows.
- If `dates` is present & non-empty → the day picker must **enable only those days**,
  disable the rest. If empty/missing → all days 1..28 allowed.
- **Daily frequencies (`daily`, `calendar_day_daily`) have NO installment_day** — omit
  the day picker entirely; don't send `installment_day` for daily SIPs.

#### 4. Number of installments
- **min** = the selected frequency row's `installments_min` (fallback 6). Message on
  violation: *"Minimum {min} installments required for {frequencyLabel}."*
- **max** = **240** (hard app cap). Message: *"Maximum 240 installments."*

#### 5. Validation rules (exactly what the web enforces — mirror these)
Amount (on every change, against the SELECTED frequency's row):
- empty / ≤ 0 → "Amount is required."
- `< amount_min` → "Minimum SIP is ₹{min}."
- `> amount_max` → "Maximum is ₹{max}."
- `amount_multiples > 0 && amount % multiples !== 0` → "Amount must be a multiple of ₹{mul}."

Installments: min = `installments_min`, max = 240 (see #4).

Installment day (monthly): must be in `dates` if `dates` non-empty (else 1..28).

Only enable the **Create Mandate & Start SIP** button when amount + installments + day all
pass AND the user is invest-ready (see §14.2 / §29.9.1 gate). Then run the DB-driven
create flow in **§21.2** (register-pending → authorize → poll `sip/status` → finalize).

#### 6. What we send at create (built server-side from the pending row)
`payment_method: "mandate"`, `payment_source: <mandate old_id>`, `systematic: true`,
`frequency`, `amount`, `number_of_installments`, `installment_day` (omit for daily),
`mf_investment_account`, `scheme` (ISIN). `user_ip` is resolved server-side — **don't send it.**

> **Golden rule for the app:** bind frequencies, min/max, multiples, installments_min, and
> allowed dates **entirely from `thresholds[]`** in the `fund-schemes-v2/ONDC/{ISIN}`
> response. Re-read the per-frequency row whenever the user changes frequency. Never
> hardcode ₹500 / 6 / dates — they vary by scheme AND by frequency.

---

Per docs the ONDC plan lifecycle adds extra states beyond the order flow:

```
created → review_completed → confirmed → submitted → active
```

### 21.1 Prerequisite: an APPROVED mandate

A SIP needs a `payment_source` which is the numeric `old_id` of an
**APPROVED** eNACH/UPI Autopay mandate. The investor sets this up once.

```
┌─────────────────────────────────────────────────────────────────────┐
│ MANDATE STEP 1 — Create                                             │
│ POST /api/pg/mandates                                               │
│ Body:    {                                                          │
│            mandate_type:    "E_MANDATE"   (or "UPI"),               │
│            bank_account_id: <bank.old_id>,                          │
│            mandate_limit:   50000,                                  │
│            provider_name:   "CYBRILLAPOA"  ← for ONDC tenants only  │
│          }                                                          │
│ Returns: { id: <int> }                                              │
│                                                                     │
│ ⚠ provider_name MUST be CYBRILLAPOA on ONDC tenants.                │
│   ONDC itself doesn't process mandates; Cybrilla's POA does.        │
└─────────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────┐
│ MANDATE STEP 2 — Authorize                                          │
│ POST /api/pg/mandates/{id}/authorize?return=invest/<isin>           │
│   (NO body — backend fills payment_postback_url from appsettings    │
│    Finprim:Postback:{Web|App}:Mandate; client just passes `return`  │
│    so the callback page can route the user back where they were)    │
│ Returns: { token_url, id }                                          │
└─────────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────┐
│ MANDATE STEP 3 — Open token_url, user authorizes at bank            │
│ window.open(token_url, '_blank')                                    │
│ Bank → POSTs to /api/pg/mandates/postback (backend)                 │
│ Backend → 303-redirects user to                                     │
│   /app/mutual-funds/mandate-callback?status=success|pending|failure │
│                                  &paymentId=...&return=invest/<isin>│
└─────────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────┐
│ MANDATE STEP 4 — Poll until APPROVED                                │
│ While creating: poll GET /api/pg/mandates/{id} every 3s.            │
│ mandate_status states: CREATED → SUBMITTED → APPROVED               │
│                                       (or REJECTED / CANCELLED)     │
│                                                                     │
│ APPROVED → mandate.old_id is now usable as payment_source.          │
└─────────────────────────────────────────────────────────────────────┘
```

### 21.2 SIP create flow — DB-DRIVEN & RESUMABLE (CURRENT — 2026-07-06 rework)

> **This SUPERSEDES the one-shot "create the plan directly" flow shown in 21.2-LEGACY
> below.** The SIP is now tracked in `tbl_Fintech_MF_Purchase_Plan` by a single
> **IntentKey** (reuses the `IdempotencyKey` column) linking OTP + mandate + plan.
> **Nothing is stored in localStorage.** A PENDING plan row is inserted BEFORE the
> mandate, so the SIP is always resumable if the browser/app closes mid-flow.
>
> LocalStatus lifecycle: `mandate_pending → mandate_approved → active`
> (branches: `mandate_failed | failed | cancelled | paused`).

**Sequence (all SIP endpoints under `api/mf/orders`):**

1. **Send OTP up-front** (before the mandate) — `POST purchase-plans/send-otp`,
   header `Plan-Intent-Key: <IntentKey uuid>`, body `{ mobile, email, isdCode }`
   → `{ intentKey, expiresAt, channel, sandboxOtp? }`.
2. **Verify OTP (peek, no consume)** — `POST purchase-plans/verify-otp`,
   header `Plan-Intent-Key`, body `{ otp }` → `{ valid: true }`. It is consumed
   later at finalize, by IntentKey.
3. **Register the PENDING row (row exists FIRST)** — `POST sip/register-pending`,
   body `{ intentKey, mandateId, mfInvestmentAccount, scheme, amount, frequency,
   installmentDay, numberOfInstallments }` → `{ localStatus:"mandate_pending" }`.
   (`mandateId`/`paymentSource` accept string OR number.)
4. **Authorize the mandate SAME-TAB, carrying the IntentKey** —
   `POST /api/pg/mandates/{id}/authorize?return=invest/<isin>&intent=<IntentKey>`
   → `{ token_url }`; open same-tab (web) / WebView (app). Bank POSTs to
   `/api/pg/mandates/postback?intent=<IntentKey>`; backend resolves the pending row:
   success → `mandate_approved`, fail → `mandate_failed`. The `mandate.*` **webhook**
   also resolves by MandateId as a backstop. ⚠ A SUCCESS postback carries
   `reason="Mandate Successful"` — do NOT classify failure by `reason.contains("mandate")`.
5. **Read status, then finalize** — `GET sip/status/{intentKey}` →
   `{ localStatus, mfPurchasePlanId, mandateId, scheme, amount, frequency,
   numberOfInstallments, reason }`. If `mandate_approved`:
   `POST sip/finalize/{intentKey}` body `{ otp:"" }` → `200 { localStatus:"active",
   mfPurchasePlanId:"mfpp_xxx", plan }`.
   Finalize errors (handle ALL — see 21.2b): `400 {reason:"otp_missing"}`,
   `400 {reason:"nomination_required"}`, `409` mandate-not-approved, `502 {reason}`.
6. **Success popup** — Fund, Amount/frequency, Installments, Plan ID; [View my SIPs]
   → `/app/mutual-funds/transactions` (SIP Plans tab).

**Server-side (API):** finalize builds the Cybrilla create from the PENDING row
(`payment_method:"mandate"`, `payment_source:<mandate old_id>`, `systematic:true`).
`user_ip` MUST be valid IPv4 via `ResolveClientIpv4()` (XFF IPv4 → socket IPv4 →
`appsettings Finprim:FallbackClientIp`). **Never `::1`/hardcoded.** ONDC auto-confirm
(create → review_completed → PATCH confirmed → submitted → active) runs server-side.

### 21.2b Resume / "Complete SIP" — handle every outcome

An interrupted SIP = pending row with `localStatus` `mandate_pending`/`mandate_approved`
and **no `mfPurchasePlanId` yet**. Show it on the Dashboard "Pending completion" card
and the SIP Plans tab with a **"Complete SIP"** button (NOT "Resume" — that word is
reserved for un-pausing a paused SIP). Show the button ONLY when: has `intentKey` AND
`mfPurchasePlanId` is null AND localStatus is `mandate_pending`/`mandate_approved` AND
state not already active/terminal. A real plan (has mfpp id) never shows it.

Handler (same on web + app):
1. `GET sip/status/{intentKey}`.
2. `active` → success popup + refresh.
3. `mandate_approved` → `POST sip/finalize/{intentKey}` `{otp:""}`.
   - On `400 otp_missing` → **re-send + verify a FRESH OTP** (send-otp → verify-otp by
     IntentKey → finalize with that otp). Sandbox returns auto-OTP; prod prompts.
   - success → success popup with plan details.
4. any other status → route to invest **with `?resume=<intentKey>`** (re-authorize the
   mandate) — do NOT start a blank fresh SIP.
5. real error → show parsed `reason`. Never leave the screen stuck.

### 21.2c Finalize is atomic (no duplicate-key)

Finalize calls SP `Usp_Finalize_Fintech_MF_Plan_Row`, guaranteeing exactly ONE plan row
even when a `mf_purchase_plan.*` webhook inserts the snapshot before finalize: if another
row already owns the `mfPurchasePlanId`, it deletes the redundant pending row FIRST, then
moves IntentKey+mandate onto the survivor and marks it `active`; else it stamps the pending
row. Also stamps `PaymentMethod='mandate'`. Removed the recurring
`Cannot insert duplicate key … UX_Plan_MfPurchasePlanId / UX_Plan_Idempotency`.

### 21.2d Nomination required BEFORE the SIP create

Nominees live on the **MFIA `folio_defaults`** (`nominee1..3`,
`nominations_info_visibility`), NOT on the SIP request. Finalize VERIFIES the folio has a
declaration (`show_all_nominee_names` = has nominees, or `show_nomination_status` = opted
out); if absent → `400 {reason:"nomination_required"}`. Set nominees/opt-out in the
nomination step first. Fixes the old "add guardian/nominee details" rejection.

### 21.2e SIP Plans list — column contract (API/DB)

`GET purchase-plans/my-list` → `Usp_Get_Fintech_MF_Purchase_Plan_ByUser`. The reader reads
columns **by index 0..33**; the SELECT MUST end with, in order:
`… Reason(30), LocalStatus(31), IdempotencyKey AS IntentKey(32), MandateId(33)`. Dropping
the last three makes the reader throw → endpoint silently returns `[]` → SIP Plans tab shows
"no data". List is newest-first; UI buckets by LocalStatus (active / pending+interrupted /
inactive) and shows a "Created <date>".

---

### 21.2-LEGACY SIP create flow (after at least one APPROVED mandate exists) — DO NOT USE

```
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1 — Send OTP for plan consent                                  │
│ POST /api/mf/orders/purchase-plans/send-otp                         │
│ Header:  Plan-Intent-Key: <uuid for this intent>                    │
│ Body:    { mobile, email, isdCode }                                 │
│ Returns: { sandboxOtp: "123456", ... }                              │
└─────────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 2 — Verify OTP (peek)                                          │
│ POST /api/mf/orders/purchase-plans/verify-otp                       │
│ Header:  Plan-Intent-Key: <same uuid>                               │
│ Body:    { otp: "<user-entered>" }                                  │
│ Returns: 200 { valid: true }                                        │
└─────────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 3 — Create + auto-confirm SIP plan                             │
│ POST /api/mf/orders/purchase-plans                                  │
│ Headers: Plan-Intent-Key: <same uuid>                               │
│          Idempotency-Key: <uuid for retry safety>                   │
│ Body:    {                                                          │
│            mf_investment_account: "mfia_xxx",                       │
│            scheme:                 "<isin>",                        │
│            amount:                 1000,                            │
│            installment_day:        5,                               │
│            frequency:              "monthly",                       │
│            number_of_installments: 12,                              │
│            payment_method:         "mandate",                       │
│            payment_source:         "<mandate.old_id as string>",    │
│            systematic:             true,                            │
│            gateway:                "ondc",                          │
│            auto_generate_installments: true,                        │
│            initiated_by:           "investor",                      │
│            initiated_via:          "website" | "mobile_app",        │
│            user_ip:                "127.0.0.1",                     │
│            consent: { email, isd_code, mobile, otp }                │
│          }                                                          │
│                                                                     │
│ Returns: { id: "mfpp_xxx", state: "active", ... }                   │
│                                                                     │
│ ⚙ Server-side auto-confirm:                                         │
│   The controller polls GET /v2/mf_purchase_plans/{id} every 2s for  │
│   up to 15s. When state hits `review_completed`, it auto-PATCHes    │
│   state=confirmed so the plan progresses to submitted → active.     │
│   If the poll times out, the webhook handler for                    │
│   `mf_purchase_plan.review_completed` auto-PATCHes as a fallback.   │
│   App doesn't need to do anything beyond this single create call.   │
└─────────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 4 — Show success                                               │
│ Render in-page success modal with: plan id, monthly amount,         │
│ installment day, total commitment, next debit date.                 │
│ Buttons: [Close] or [View My SIPs] → /app/mutual-funds/sips         │
└─────────────────────────────────────────────────────────────────────┘
```

### 21.3 Installments

Cybrilla auto-creates an `mf_purchase` order on each `installment_day`
and debits via the linked mandate. These show up in the Transactions
page automatically. Plan's `remaining_installments` decrements per
installment; when it hits 0 the plan state moves to `completed`.

App does **no** work per-installment.

### 21.4 Cancel SIP

```
POST /api/mf/orders/purchase-plans/{mfpp_id}/cancel
Body: { "cancellationCode": "investment_returns_not_as_expected" }    // optional
```

**Important rules per FP runtime (sandbox enforces):**
- ONDC gateway: cancel is **only allowed when plan.state === "active"**.
  Other states (`created`, `review_completed`, `confirmed`, `submitted`,
  `cancelled`, `completed`, `failed`) return 400.
- Non-ONDC gateways (BSE): also allow `created` and `submitted`.
- `cancellation_code` is **required** by FP. Backend defaults to
  `investment_returns_not_as_expected` if the caller omits it.
- Valid codes: `investment_returns_not_as_expected`, `change_in_financial_goals`,
  `fund_performance_issues`, `personal_financial_constraints`, `others`.

**Recommended pre-cancel flow on the app:**
1. Tap Cancel → call `GET /api/mf/orders/purchase-plans/{mfpp_id}` first
   (this re-syncs FP state into our DB)
2. If returned `state !== 'active'`, show "Cannot cancel — current state X"
3. Otherwise POST the cancel.

The mandate that backs the SIP **stays APPROVED** after a SIP cancel —
cancel the mandate separately if the user wants the bank's standing
instruction revoked too. (Note: CYBRILLAPOA mandates can't be cancelled
via API — bank-managed only. See section 22.6.)

### 21.5 Pause / Resume SIP (FP `skip_instructions`)

Per [FP docs](https://docs.fintechprimitives.com/mf-transactions/purchase-plans/pause-sip/):

#### Pause
```
POST /api/mf/orders/purchase-plans/{mfpp_id}/pause
Body: { "from": "2026-09-01", "to": "2026-11-01" }   // "to" optional
```
- `from` is required, ISO date `YYYY-MM-DD`.
- `to` is optional — omit for indefinite pause.
- Installments whose `installment_date` falls in `[from, to]` are marked
  `cancelled` by FP. Skipped installments do NOT debit.
- Allowed only when `state === 'active'` AND `gateway !== 'bse'`
  (BSE SIPs cannot be paused per FP docs).

Returns the skip instruction object:
```json
{
  "object": "plan_skip_instruction",
  "id": "psi_xxx",
  "plan": "mfpp_xxx",
  "state": "active",
  "from": "2026-09-01",
  "to": "2026-11-01",
  "remaining_installments": 2,
  "skipped_installments": 0
}
```

#### Resume
```
POST /api/mf/orders/purchase-plans/skip-instructions/{psi_id}/cancel
Body: {}
```
Cancels the active skip instruction. Future installments from today
onwards resume on the regular `installment_day`.

#### List skip instructions for a plan
```
GET /api/mf/orders/purchase-plans/{mfpp_id}/skip-instructions
```
Returns `{ data: [PlanSkipInstruction, ...] }`. Use this on app start to
know whether a SIP is currently paused — look for the entry with
`state === 'active'`.

#### Native-app UX rules
- Show **Pause** button only when SIP state is `active` and no active
  skip instruction exists.
- Show **Resume** button (in place of Pause) when an active skip
  instruction exists for that plan.
- Show a "Paused from … to …" / "Paused (indefinite)" banner alongside
  the plan card.
- Pause modal must ask for `from` (default = today, min = today) and an
  optional `to` (min = `from`). Submit disabled until `from` is set.

---

## 22. Postback URL Architecture

### 22.1 Centrally configured in `appsettings.json`

```json
"Finprim": {
  "Spa": { "BaseUrl": "http://localhost:4200" },
  "Postback": {
    "Web": {
      "DigiLocker": "http://localhost:4200/app/mutual-funds/kyc",
      "Esign":      "http://localhost:4200/app/mutual-funds/kyc?esign=completed",
      "Mandate":    "https://localhost:7001/api/pg/mandates/postback",
      "Payment":    "http://localhost:4200/app/mutual-funds/payment-callback"
    },
    "App": {
      "DigiLocker": "http://localhost:4200/appmutualfund/kyc",
      "Esign":      "http://localhost:4200/appmutualfund/esign",
      "Mandate":    "https://localhost:7001/api/pg/mandates/postback?source=app",
      "Payment":    "http://localhost:4200/appmutualfund/payment"
    }
  }
}
```

### 22.2 Why Mandate goes through backend

Banks respond with **HTTP form POST** to the postback URL. Angular SPAs
can't capture POST bodies (the browser does a fresh GET-load). So the
Mandate URL points at our backend `/api/pg/mandates/postback`, which:

1. Captures the form data (`paymentId`, `status`, `failureReason`).
2. Logs into `tbl_Fintech_MF_Order_Log` (step=`mandate_postback`).
3. **303-redirects** the browser to the SPA at
   `/app/mutual-funds/mandate-callback?status=...&paymentId=...&return=...`
   so Angular can render the result.

### 22.3 Payment postback URL

**Important: the postback URL points to the BACKEND, not the SPA directly.**
This is the same pattern as the mandate flow (§22.2). Reason: per FP docs
"a post back using HTTP POST will be made via web browser to
payment_postback_url" — Angular SPAs can't read POST bodies, so we'd lose
`paymentId`, `status`, `failureCode`, `failureReason`.

Set per-call on `POST /api/pg/payments/netbanking` (and `/upi`):
```
payment_postback_url:
  "<api-host>/api/pg/payments/postback?mfp=<mfp_xxx>"              (web)
  "<api-host>/api/pg/payments/postback?mfp=<mfp_xxx>&source=app"   (app — WebView)
```

Flow:
1. Bank/PG does a **form POST** to the URL above with body fields
   `paymentId`, `status`, `failureCode`, `failureReason`.
2. `PaymentPostbackController` catches the POST, writes an audit log row to
   `tbl_Fintech_MF_Order_Log` (step = `payment_postback`).
3. Backend **303-redirects** the browser to the SPA route with the form
   data converted to query params:
   ```
   web:  <spa-base>/app/mutual-funds/payment-callback?mfp=&id=&status=&failureCode=&reason=
   app:  <spa-base>/appmutualfund/payment?mfp=&id=&status=&failureCode=&reason=
   ```
4. SPA page renders / polls / deep-links based on the params.

- **Web** (`/app/mutual-funds/payment-callback`) — the page polls
  `/api/pg/payments/{id}/snapshot` + `/api/mf/orders/purchases/{mfp}/snapshot`
  every 2s for ~60s. Renders success/failure card based on the **DB**
  state, not the URL params (bank URLs can lie).

- **App** (`/appmutualfund/payment`) — thin pass-through (no polling).
  Reads the URL params, deep-links the native app via
  `islamicly://mf-callback?flow=payment&id=&mfp=&status=&reason=`,
  and lets the native app do the polling.

### 22.4 What the native app must do when the WebView lands on `/appmutualfund/payment`

This is the single most important integration point for the mobile team. Treat
the page as a **bridge**, not as the final result screen:

**1. Detect the landing.**
   - **URL watch (recommended).** Hook `shouldOverrideUrlLoading` (Android) /
     `decidePolicyFor navigationAction` (iOS) on the WebView. When the URL
     matches `…/appmutualfund/payment*` OR scheme is `islamicly://mf-callback*`,
     cancel/dismiss the WebView and proceed to step 2.
   - **Deep-link fallback.** If URL-watch isn't possible, register
     `islamicly://` as the app's URL scheme — the page renders a "Return to
     App" button whose `href` is `islamicly://mf-callback?...` which the OS
     hands to the app on tap.

**2. Read the query params from the intercepted URL.**

   | Param | Meaning | Always present |
   |---|---|---|
   | `id` | FP payment id (numeric) | yes, except totally lost callbacks |
   | `mfp` | `mfp_xxx` — the MF purchase id | yes |
   | `status` | `success` / `failed` / `pending` (or one of the synonyms below) | yes |
   | `reason` | failure_reason text | only on failure |

   Synonyms the page normalises:
   - **success** ⇽ `success` / `successful` / `approved` / `settled` / `completed`
   - **failure** ⇽ `failed` / `failure` / `rejected` / `cancelled` / `user_cancelled`
   - **pending** ⇽ `pending` / `initiated` / `submitted` / `in_progress` / `processing` (and any unknown value)

**3. Do NOT trust `status` from the URL alone.** Bank/PG redirects often
   say "success" while the debit is still in flight at NPCI, or "pending"
   when it has already failed. Always re-verify against the backend.

**4. Poll the backend for the authoritative state.**

   Call BOTH endpoints in parallel every 2 seconds, up to 60 seconds:

   ```
   GET /api/pg/payments/{id}/snapshot               → payment object
   GET /api/mf/orders/purchases/{mfp}/snapshot      → MF order object
   ```

   Stop polling when **any** of these terminal conditions is met:

   | Payment `status` | Order `state` | Outcome |
   |---|---|---|
   | `SUCCESS` / `APPROVED` / `SETTLED` | `successful` / `submitted` | **Success** — money debited |
   | `FAILED` / `REJECTED` | any | **Failure** — show `failure_code` |
   | any | `cancelled` | **Cancelled** |
   | 60s elapsed, still pending | still `pending` | **Pending** — show "We'll update you" + push notification will follow |

**5. Render the result screen natively.** Use the polled snapshot, NOT the URL
   param, to drive the UI:
   - Success → "Payment received — units will be allotted in 1 working day."
     Show `purchased_amount`, `nav`, `allotted_units` (these populate after AMC
     allotment, may still be null on first success — show with placeholder).
   - Failure → "Payment failed: \<failure_reason>". Map `failure_code` to
     user-friendly text using the table in section 19.4.
   - Pending → spinner + "We're confirming with your bank. We'll notify you
     when it settles. Safe to close." Backend webhook updates the snapshot;
     a push notification driven by `payment.success` / `payment.failed` is
     the right UX.

**6. Always re-fetch the user's transactions list** (`GET /api/mf/orders/purchases/my-list`)
   when the user navigates back to the Transactions tab — that snapshot
   reflects every later webhook update even if the polling above timed out.

**Why this matters:** the WebView page is a sandboxed Angular component
that has no JWT, no native nav stack, and no ability to push notifications.
The native app owns the user's session and ultimately the result screen.
The WebView's only job is to bring the user back into the app with the
right `id`+`mfp`+`status` in hand.

### 22.5 What the native app must do when the WebView lands on `/appmutualfund/mandate`

Same pattern, with these differences:
- URL params: `id` (mandate id, numeric), `status` (`success` / `failed` / `pending` and synonyms), `reason`
- Deep-link emitted by the page: `islamicly://mf-callback?flow=mandate&id=&status=&reason=`
- Backend polling endpoint:
  ```
  GET /api/pg/mandates/{id}                       → mandate object (mandate_status)
  ```
- Terminal `mandate_status` values: `APPROVED` (success), `REJECTED` / `CANCELLED` (failure)
- Pending values: `CREATED`, `RECEIVED`, `SUBMITTED` — keep polling
- BillDesk eNACH approval can take up to 2 minutes in sandbox; bump the
  poll timeout to 120s for mandate vs 60s for payment.
- After success → re-fetch `GET /api/pg/mandates/my-list` to refresh the
  Auto-Pay tab in Transactions.

### 22.6 CYBRILLAPOA mandate cancellation — bank-managed only

For the `islamiclyondc` tenant, mandates are issued by FP's
**CYBRILLAPOA** provider (POA-based eNACH). The runtime API **blocks**
`POST /v2/mandates/{id}/cancel` for this provider with:

```
400 — Operation not allowed for CYBRILLAPOA provider
```

This is a tenant-level constraint (not a doc-stated one — FP's doc
omits the caveat). Native app rules:

- Detect provider on the mandate object: `provider_name === "CYBRILLAPOA"`.
- **Do NOT show a Cancel button** for these mandates.
- Show a "Bank-managed" badge instead with a tooltip explaining the
  user must revoke via their bank's eNACH portal (or cancel the SIPs
  that use this mandate to stop debits).
- When the bank eventually revokes it, FP fires
  `mandate.cancelled` → our `FinprimEventDispatcher` updates
  `tbl_Fintech_Mandate.MandateStatus = "CANCELLED"`. The app's
  Auto-Pay tab will reflect this on next list refresh.

For non-POA providers (RAZORPAY/BILLDESK), the standard cancel
endpoint works normally.

### 22.7 BillDesk simulation (sandbox vs production)

**Sandbox:** when a mandate is created on the sandbox tenant, FP
returns a `token_url` that points to BillDesk's eNACH simulator
(`https://simulator.fintechprimitives.com/...`). The simulator UI
lets you pick `APPROVE` / `REJECT` / `PENDING` for the bank's
behaviour, then POSTs the form to our `payment_postback_url`
(`/api/pg/mandates/postback?source=app|web`).

**Production:** the same `token_url` points to the **real** BillDesk
NACH page where the user enters bank credentials. The form-POST
contract to our postback URL is identical — same `paymentId`, `status`,
`failureReason` fields — so **no app code change** is needed between
sandbox and prod. The only difference is the token URL host.

Native devs: there is **nothing app-specific to handle for the
simulator**. The WebView opens whatever `token_url` FP returns; the
simulator is just an FP-hosted HTML page rendered in the same WebView,
and its "Approve" / "Reject" buttons fire the production-equivalent
form POST. Section 22.5's instructions cover both.

For **lump-sum payments** — corrected per Cybrilla support
(2026-05-22) and verified against an actual ONDC sandbox run:

The `token_url` returned by `POST /v2/payments` (our wrapper:
`POST /api/pg/payments/netbanking` or `/upi`) is the place where the
user completes the payment. **What it looks like depends on the
gateway**:

- **BSE gateway** — `token_url` is a Cybrilla-hosted simulator
  (e.g. `simulator.fintechprimitives.com`) with explicit
  "Simulate Success" / "Simulate Failure" buttons (Netbanking) or
  a dummy-VPA input box (UPI).
- **ONDC gateway** (our `islamiclyondc` tenant) — `token_url` chains
  through the **real bank aggregator's test environment**:
  ```
  https://islamiclyondc.s.finprim.com/api/pg/payments/netbanking/ondc?txnId=…
    → https://api.sandbox.cybrilla.com/ondc/payments?…
    → https://merchant.now.hdfc.bank.in/epi-ui/?uniqueKey=ATOMCAMSINTE_…
  ```
  The final URL is HDFC's ATOM **INTE** (integration/UAT) environment,
  not production — `INTE` in the uniqueKey is the giveaway. **Real
  HDFC UI, no real money movement.** User enters any test credentials
  the aggregator accepts; outcome POSTs back to our
  `payment_postback_url` exactly like production would.

If your sandbox token_url ever opens a URL **without** the `INTE` /
`UAT` / `TEST` marker in the path or query string, stop — verify
that you're on the right tenant. Production tenants will go through
identical-looking real-bank URLs but without the test marker, and
real money will move.

**Mandatory ordering**: the order MUST be in `state=confirmed` before
the `token_url` is opened. Opening it on `pending` or `submitted` is
undefined per Cybrilla. Our `/payments/netbanking` wrapper already
prechecks `state=confirmed` (via OldId lookup on
`tbl_Fintech_MF_Purchase_Order`) and refuses to issue a token URL
otherwise — see [PaymentsController.cs] netbanking prefcheck.

In production, the same `token_url` redirects to the **real** bank's
netbanking / UPI flow. Contract to our postback URL is identical, so
no app code change. The only difference is the URL host.

> The older `POST /api/mf/orders/purchases/{id}/simulate` endpoint
> (forwarding to FP `/api/oms/simulate/orders/{id}`) **is no longer
> the recommended sandbox path for payment-driven orders** — Cybrilla
> support clarified the `token_url` does the simulation. We keep the
> endpoint wired for legacy callers but new flows should use the
> `token_url` route. It IS still needed for **SIP installments**
> (mandate-driven, no per-installment token_url) — that's what
> `POST /api/mf/orders/purchase-plans/{id}/simulate-installment` is for.

### 22.8 Webhooks vs polling — official guidance from Cybrilla

> "You can rely on webhooks to receive payment status updates.
> Polling is not mandatory but can be used as a fallback mechanism
> in cases where a webhook event is delayed or not received."
> — Cybrilla support, 2026-05-22

Our implementation honours both:
- **Primary**: `FinprimEventDispatcher` updates `tbl_Fintech_Payment`
  and `tbl_Fintech_MF_Purchase_Order` on every `payment.*` and
  `mf_purchase.*` webhook.
- **Fallback**: web payment-callback page polls `/payments/{id}/snapshot`
  every 2 s for ~60 s. Mobile app does the same in its native polling.

Mobile devs do not need to poll if push notifications driven by the
webhook handler are wired — that's the production-grade path. Polling
is the dev/sandbox safety net.

---

## 23. Webhook Events Handled

Registered via `POST /api/webhooks/bootstrap` (57 events total).
Dispatcher routes each to a snapshot updater in `tbl_Fintech_*`.

### Critical events for the MF flows

| Event | Action |
|---|---|
| `pre_verification.accepted` / `.completed` | Bank verification result snapshot |
| `kyc_request.*` | KYC state machine snapshot |
| `mf_purchase.created` / `.confirmed` / `.submitted` / `.successful` / `.failed` / `.reversed` / `.cancelled` | Order state machine + folio + units |
| `mf_purchase.review_completed` (ONDC only) | Order auto-progress signal |
| `mf_purchase_plan.created` / `.activated` / `.cancelled` / `.failed` / `.completed` | SIP state machine |
| `mf_purchase_plan.review_completed` (ONDC only) | **Auto-PATCHes state=confirmed** as fallback |
| `mandate.created` / `.submitted` / `.approved` / `.rejected` / `.cancelled` | Mandate state machine |
| `payment.pending` / `.submitted` / `.success` / `.initiated` / `.approved` / `.failed` / `.rejected` | Payment state machine + failure_code |

### HMAC verification

`Finprim:WebhookSecret` in appsettings. Every POST to
`/api/webhooks/finprim-events` is verified with HMAC-SHA256 over the
raw body, compared to the `X-Webhook-Signature` header in constant
time. Mismatch → 401 (Cybrilla doesn't retry). Empty secret skips
verification with a warning (dev only).

---

## 24. New DB Tables and SPs (added in this pass)

All migrations are appended to `Claude-db-queries.txt` in run order.

| Table | Purpose |
|---|---|
| `tbl_Fintech_Pre_Verification` | 3-stage pre-verification snapshot (readiness/pan/bank) |
| `tbl_Fintech_Bank_Account` (extended) | + verification columns + numeric `OldId` |
| `tbl_Fintech_MF_Purchase_Order` | One-time order snapshot + idempotency |
| `tbl_Fintech_MF_Purchase_Plan` | **NEW** — SIP plan snapshot + idempotency |
| `tbl_Fintech_Payment` | Payment snapshot |
| `tbl_Fintech_Purchase_Otp` | OTP storage + verify-with-attempt-lock |
| `tbl_Fintech_MF_Order_Log` | Generic audit log (every lifecycle event) |
| `tbl_Fintech_Webhook_Event_Log` | Raw Cybrilla webhook receipt log |
| `tbl_Fintech_Tenant_AccessToken` | Cached tenant bearer token |

| Stored procedure | Notes |
|---|---|
| `Usp_Upsert_Fintech_MF_Purchase_Order` | MERGE keyed on MFPurchaseId |
| `Usp_Get_Fintech_MF_Purchase_Order_ByUser` | Lists user's orders + LEFT JOIN to scheme name + latest payment status |
| `Usp_Upsert_Fintech_MF_Purchase_Plan` | **NEW** MERGE keyed on MfPurchasePlanId |
| `Usp_Get_Fintech_MF_Purchase_Plan_ByUser` | **NEW** Lists user's SIPs + JOIN to scheme name |
| `Usp_Upsert_Fintech_Payment` | Now accepts UserId override on UPDATE branch |
| `Usp_Insert_Fintech_Purchase_Otp` / `Usp_Verify_Fintech_Purchase_Otp` | 10-min TTL + 5-attempt lock |
| `Usp_Get_Fintech_Bank_Account_Status` | Returns LocalId + OldId + verification status |
| `Usp_Upsert_Fintech_Pre_Verification` | MERGE keyed on FpPreVerificationId |
| `Usp_Get_Latest_Fintech_Pre_Verification_By_User` | Resume support |

Filtered unique indexes:
- `UX_Fintech_MF_Purchase_Order_Idempotency` on `(UserId, IdempotencyKey) WHERE IdempotencyKey IS NOT NULL`
- `UX_Plan_Idempotency` on `tbl_Fintech_MF_Purchase_Plan(UserId, IdempotencyKey) WHERE IdempotencyKey IS NOT NULL`
- `UX_PreVerification_FpId` on `tbl_Fintech_Pre_Verification(FpPreVerificationId) WHERE FpPreVerificationId IS NOT NULL`
- `UX_Plan_MfPurchasePlanId` on `tbl_Fintech_MF_Purchase_Plan(MfPurchasePlanId)`

---

## 25. Frontend Pages Built

All under `src/app/features/mutual-funds/`:

| Route | Component | Purpose |
|---|---|---|
| `/app/mutual-funds` | mutual-fund-landing | Hub |
| `/app/mutual-funds/list` | mutual-fund-list | Browse funds |
| `/app/mutual-funds/details/:id` | mutual-fund-details | Per-fund detail |
| `/app/mutual-funds/onboarding` | onboarding | Multi-step wizard (profile → address → bank → MFIA + nominee) |
| `/app/mutual-funds/kyc` | mutual-fund-kyc | KYC dashboard (PAN, DigiLocker, eSign) |
| `/app/mutual-funds/bank-account` | bank-account | Bank create + 3-stage pre-verify |
| `/app/mutual-funds/invest/:id` | mutual-fund-invest | **Lumpsum + SIP form** |
| `/app/mutual-funds/payment-callback` | payment-callback | Post-payment landing (polls snapshots) |
| `/app/mutual-funds/mandate-callback` | mandate-callback | Post-mandate-authorize landing |
| `/app/mutual-funds/sips` | mutual-fund-sips | All user's SIPs grouped Active / Pending / Closed |
| `/app/mutual-funds/transactions` | mutual-fund-transactions | **Orders / SIPs tab toggle** |
| `/app/mutual-funds/portfolio` | mutual-fund-portfolio | Holdings (folios) |

### Invest page modes
The invest page now has two tabs (no more "SIP — Soon"):
- **Lumpsum** — runs the 7-step ONDC sequence in §20.
- **SIP** — runs the 3-step plan sequence in §21, with inline "Set up Auto-pay" card when no mandate exists.

The OTP modal serves both flows; the "Verify & Pay/Start SIP" button
routes to the correct handler based on `mode()`.

---

## 26. Reconciliation Job

`PurchaseReconciliationJob` (BackgroundService, runs every 6 hours,
configurable via `Finprim:Reconcile:IntervalHours`). For every order
that's been non-terminal for >1 hour, hits Cybrilla GET and refreshes
our snapshot. Catches webhook losses (Cybrilla retries ~5 times then
gives up).

Same logic should be added for plans — TODO.

---

## 27. Sandbox vs Production Quick Reference

| Concern | Sandbox (ONDC) | Production (ONDC) |
|---|---|---|
| `Runtime:IsProduction` | `0` | `1` |
| Order create | works | works |
| Order RTA allotment | amount ending 0 → succeeds, ending 1 → fails (auto cron) | real AMC, T+1 |
| Order simulate API | **blocked** for ONDC | n/a |
| Payment | BillDesk sandbox NetBanking page | real HDFC NetBanking |
| Payment simulate | works (`POST /payments/{id}/simulate`) | not available |
| eNACH mandate | works via CYBRILLAPOA provider, BillDesk sim | real bank eNACH |
| SIP plan create | works, auto-activates in ~15s | works, activates after real review |
| Webhooks | via ngrok in dev | real public URL |
| Webhook secret | empty in dev (warning) | mandatory in prod |

---

## 27.5 SIP Lifecycle + Transactions Page — Complete Mobile Reference

This section is the **single source of truth** for everything the mobile
app needs to implement on the Transactions tab and the SIP detail UX.
Everything below is implemented on the web today and live in the
backend — the mobile app should mirror it screen-for-screen.

### 27.5.1 Transactions tab tabs

Three tabs, all populated from local DB (no Cybrilla calls on tab switch):

| Tab | Source endpoint | What it shows |
|---|---|---|
| Orders | `GET /api/mf/orders/purchases/my-list` | Every `mf_purchase` row — lump-sum purchases AND SIP installments (installment rows have a non-null `plan` field) |
| SIPs | `GET /api/mf/orders/purchase-plans/my-list` | Every `mf_purchase_plan` row — active, pending, cancelled, completed |
| Auto-pay | `GET /api/pg/mandates/my-list` | Every mandate row — eNACH/UPI, approved/pending/cancelled |

All three call the **wrapper's local-DB endpoint** — fast (~50ms),
no Cybrilla round-trip. Refresh button re-fetches.

### 27.5.2 SIP card fields (per row on the SIPs tab)

Each row is one item from `GET /purchase-plans/my-list`. The full list of
fields you can render (all available in the list response without extra
calls):

| Field | Source | Notes |
|---|---|---|
| `mfPurchasePlanId` | DB.MfPurchasePlanId | `mfpp_xxx` |
| `schemeName` | DB.SchemeName via LEFT JOIN | for display |
| `amount` | plan.amount | ₹ per installment |
| `installmentDay` | plan.installment_day | 1–28 |
| `frequency` | plan.frequency | `monthly`, `quarterly`, `yearly` |
| `numberOfInstallments` | plan.number_of_installments | total committed |
| `state` | plan.state | drives the chip colour |
| `gateway` | plan.gateway | `ondc` / `bse` / `cybrillapoa` — controls action visibility |
| `folioNumber` | plan.folio_number | shows after first allotment |
| `startDate` / `endDate` | plan.start_date / end_date | calendar bounds |
| `nextInstallmentDate` / `previousInstallmentDate` | plan dates | "Next debit on …" |
| `remainingInstallments` | plan.remaining_installments | "9 of 12 left" |
| `mandateType` / `mandateLimit` / `mandateStatus` | LEFT JOIN tbl_Fintech_Mandate | "E_MANDATE · ₹50,000 · APPROVED" |
| `paymentSource` | plan.payment_source | mandate id (numeric as string) |
| `localStatus` | DB.LocalStatus | **DB-driven lifecycle** — `mandate_pending` / `mandate_approved` / `active` / `mandate_failed` / `failed` / `cancelled` / `paused`. Drives bucketing + the "Complete SIP" button (2026-07-06 rework). |
| `intentKey` | DB.IdempotencyKey | shared OTP↔mandate↔plan key; use to resume/complete an interrupted SIP |
| `mandateId` | DB.MandateId | linked mandate |
| `reason` | DB.Reason | human-readable failure reason (shown for `mandate_failed`/`failed`) |
| `createdAt` | DB.CreatedAt | render "Created DD MMM YYYY" on the card |

> ⚠ An **interrupted** SIP (never created at Cybrilla) has `mfPurchasePlanId = null`
> and `state = null` — bucket it by **`localStatus`**, not `state`.

### 27.5.3 SIP card actions (decision matrix)

| Button | Visible when | Endpoint | UI behaviour |
|---|---|---|---|
| **Complete SIP** | interrupted stub: has `intentKey` AND `mfPurchasePlanId == null` AND `localStatus ∈ {mandate_pending, mandate_approved}` AND state not active/terminal | `GET sip/status/{intentKey}` → `POST sip/finalize/{intentKey}` (with fresh-OTP fallback on `otp_missing`) | Finishes a SIP that never got set up. See 21.2b for the full handler. On success → success popup. **NOT the same as Resume.** |
| **Sync** | always | `GET /api/mf/orders/purchase-plans/{id}` | Re-fetches from FP + upserts DB; reloads list |
| **Pause** | `state ∈ {active}` AND `gateway ≠ bse` AND no active skip exists | `POST /api/mf/orders/purchase-plans/{id}/pause  { from, to? }` | Modal with date pickers; `from` required, `to` optional (omit for indefinite) |
| **Resume** (un-pause only) | active skip instruction exists (a PAUSED running SIP) | `POST /api/mf/orders/purchase-plans/skip-instructions/{psi_id}/cancel` | Confirm dialog → cancels skip → installments resume. **Distinct from "Complete SIP"** which finishes an interrupted setup. |
| **Cancel** | `state ∈ {active, activated}` for ONDC; also `created, submitted` for non-ONDC | `POST /api/mf/orders/purchase-plans/{id}/cancel  { cancellationCode? }` | Pre-cancel auto-sync + confirm dialog; default code = `investment_returns_not_as_expected` |
| **Simulate installment** (sandbox only) | `state ∈ {active, activated}` | `POST /api/mf/orders/purchase-plans/{id}/simulate-installment  {}` | Forces FP to generate next installment immediately; new mfp_xxx appears under this SIP's "Installments" expander |

Hide buttons strictly per the table — FP enforces these rules at the
runtime and returns 400 if violated (we've burned multiple debug cycles
on each, all documented above).

### 27.5.4 Installments under a SIP (the expander)

The expander on each SIP card lists every `mf_purchase` row whose
`plan === mfPurchasePlanId`. Source: the **same** orders array used by
the Orders tab — no extra fetch.

Each installment row shows: `mfp_xxx`, state chip, ₹amount, date. Tap
opens the order detail (folio, units, NAV) — same screen as a lump-sum
order tap.

After **Simulate installment** clicks, both lists reload so the new
installment appears in the expander immediately. In production, the
expander populates as FP fires `mf_purchase.created` webhooks each
installment_day.

### 27.5.5 SIP states + chip colours

Bucket by **`localStatus` first, then `state`** (2026-07-06). An interrupted SIP has
`state = null`, so a state-only rule wrongly hides it.

| Condition (localStatus / state) | Chip label | Colour | Tab section |
|---|---|---|---|
| `localStatus=active` OR `state ∈ {active, activated, submitted}` | Active | green | Active SIPs |
| `localStatus=mandate_approved` (stub, no mfpp) | **Pending Completion** | amber | Pending |
| `localStatus=mandate_pending` (stub) | **Awaiting Auto-pay** | amber | Pending |
| `state ∈ {created, review_completed, confirmed}` | Under Review | amber | Pending |
| `localStatus=paused` OR `state=paused` | Paused | orange | Active SIPs |
| `localStatus ∈ {mandate_failed, failed}` OR `state ∈ {failed, rejected}` | Auto-pay Failed / Failed | rose | Inactive |
| `state=cancelled` OR `localStatus=cancelled` | Cancelled | grey | Inactive |
| `state=completed` | Completed | blue | Inactive |

If a plan is **paused** (active skip instruction exists), keep it in Active SIPs and
add a "Paused from … to …" banner. For `mandate_failed`/`failed`, show the `reason`.
i18n keys added: `status.review_completed`/`under_review` = "Under Review",
`status.mandate_pending` = "Awaiting Auto-pay", `status.mandate_approved` = "Pending
Completion", `status.mandate_failed` = "Auto-pay Failed", `sips.createdOn` = "Created {date}".

### 27.5.6 SIP creation — flow (UPDATED — use the DB-driven flow in 21.2)

> **The CURRENT flow is the DB-driven, resumable one in §21.2 / §21.2b:**
> send-otp (up-front) → verify-otp (peek) → **`sip/register-pending`** (row FIRST) →
> authorize mandate SAME-TAB with `&intent=<IntentKey>` → postback/webhook flips the
> row to `mandate_approved` → **`sip/finalize/{intentKey}`** → success popup.
> The app must ALSO implement "Complete SIP" (§21.2b) to finish interrupted SIPs, and
> pass `?resume=<intentKey>` when re-authorizing. Do NOT call `POST purchase-plans`
> directly as the one-shot create — that is the 21.2-LEGACY path.
>
> The field-by-field body reference below is retained because `sip/finalize` builds the
> SAME Cybrilla `mf_purchase_plans` payload server-side (from the pending row), and
> `user_ip` handling applies identically. **Never send `user_ip:"127.0.0.1"`/`::1`
> from the client — the API resolves a valid IPv4.**

**Legacy one-shot create (reference only — server now does this inside finalize):**

1. **Pre-checks**: user has APPROVED mandate, has at least one `mfia_xxx`,
   has confirmed KYC.
2. **Send OTP**: `POST /api/mf/orders/purchase-plans/send-otp` with header
   `Plan-Intent-Key: <uuid>` and body `{ mobile, email, isdCode }`.
   Sandbox OTP = `123456`.
3. **Verify OTP (peek)**: `POST /api/mf/orders/purchase-plans/verify-otp`
   with same header + `{ otp }`. Returns 200/422; does NOT consume OTP.
4. **Create plan**: `POST /api/mf/orders/purchase-plans` with **two** headers:
   - `Plan-Intent-Key: <same uuid>` — backend verifies + consumes OTP
   - `Idempotency-Key: <new uuid>` — replay-safety on retry
   Body fields (all required unless noted):
   ```json
   {
     "mf_investment_account": "mfia_xxx",
     "scheme":                "<isin>",
     "amount":                1000,
     "installment_day":       5,
     "frequency":             "monthly",
     "number_of_installments": 12,
     "payment_method":        "mandate",
     "payment_source":        "<mandate.old_id as string>",
     "systematic":            true,
     "gateway":               "ondc",
     "auto_generate_installments": true,
     "initiated_by":          "investor",
     "initiated_via":         "mobile_app",
     "user_ip":               "127.0.0.1",
     "consent":               { "email": "...", "isd_code": "+91", "mobile": "...", "otp": "123456" }
   }
   ```
   Server **auto-confirms**: controller polls for 15s; when state hits
   `review_completed`, it PATCHes `state=confirmed`. Webhook handler is
   the fallback for missed polls. The app does NOT need to PATCH manually.

After step 4 the response state will already be `active` (or `confirmed`
→ activates within seconds via webhook). Show the success screen and
deep-link to the SIPs tab.

### 27.5.7 What changes on installment_day (FP auto-fires)

Mobile app does **nothing** per-installment. FP:
1. Creates new `mf_purchase` row → webhook `mf_purchase.created`
2. Debits via the linked mandate → webhook `payment.success`
3. AMC allots units → webhook `mf_purchase.successful`
4. Plan's `remaining_installments` decrements → webhook `mf_purchase_plan.updated`
5. Our dispatcher snapshots all of the above

Mobile app picks all this up the next time it refreshes the
Transactions tab (or via push notification driven by our webhook
handler — left to mobile team to wire to FCM/APNs).

### 27.5.8 Sandbox-only tooling reference

| Tool | Purpose | Endpoint |
|---|---|---|
| Simulate installment | Force-create the next installment now | `POST /api/mf/orders/purchase-plans/{id}/simulate-installment` |
| Simulate order outcome | Move a lump-sum mfp_xxx to success/failed | `POST /api/mf/orders/purchases/{id}/simulate  { state: "...", failure_code: "..." }` |
| Mandate simulator | Approve/reject eNACH via BillDesk's sandbox UI | FP returns `token_url` automatically |

All three are **gated to non-production environments** — the
`Runtime:IsProduction=1` flag rejects them.

---

## 27.6 Lump-sum MF Purchase — End-to-End (the way Web does it today)

The mobile app should mirror this flow exactly. Every numbered step
corresponds to one HTTP call. Sandbox and production go through the
same code path — there is **no separate "simulate" UI** anywhere in
the app.

### Prerequisites (must be true before Step 1)
- Investor KYC = `successful`
- Investor profile exists (`InvestorProfileId` set)
- MFIA exists (`mfia_xxx` linked to the user)
- Primary bank verified (`VerificationStatus = 'verified'`)

### Step-by-step

#### Step 1 — Create the order
```
POST /api/mf/orders/purchases
Headers:
  Authorization: Bearer <jwt>
  Idempotency-Key: <uuid>            ← MUST be unique per Invest click
Body:
{
  "mf_investment_account": "mfia_c3387215f10c4e2b98e9e3a8edb0d703",
  "scheme":  "INF084M01093",
  "amount":  1000,
  "folio_number": null,             // null for fresh folio
  "gateway": "ondc",                // hardcoded for islamiclyondc tenant
  "euin":    "E900356",             // distributor EUIN (preconfigured)
  "user_ip": "127.0.0.1",           // app's IP if known
  "initiated_by":  "investor",
  "initiated_via": "mobile_app"     // "website" for web
}
```
Response (200):
```json
{
  "id":     "mfp_xxx",              // KEEP THIS — used for everything after
  "old_id": 325027,                 // numeric ID — used for payment create
  "state":  "under_review",         // ONDC starts here; BSE starts at "pending"
  "amount": 1000,
  "scheme": "INF084M01093",
  "gateway": "ondc",
  ...
}
```
**Idempotency**: if the user double-taps Invest, our backend re-detects the
same `Idempotency-Key` and returns the existing order, not a new one. App
must generate a fresh UUID per intent.

#### Step 2 — Send OTP for consent
```
POST /api/mf/orders/purchases/{mfp_xxx}/send-otp
Body:
{
  "mobile":   "8660275412",
  "email":    "user@example.com",
  "isdCode":  "+91"
}
```
Response (200) — sandbox always returns `123456`:
```json
{
  "statusCode": 200,
  "message":    "OTP sent to your registered mobile and email.",
  "result": {
    "purchaseId": "mfp_xxx",
    "channel":    "both",
    "expiresAt":  "2026-05-22T15:10:00Z",
    "sandboxOtp": "123456"
  }
}
```

#### Step 3 — Verify OTP (peek only — doesn't consume)
```
POST /api/mf/orders/purchases/{mfp_xxx}/verify-otp
Body: { "otp": "123456" }
```
Response (200):
```json
{ "statusCode": 200, "valid": true, "message": "OTP verified." }
```
Failure (422):
```json
{ "statusCode": 422, "message": "Incorrect OTP.", "reason": "wrong" }
```

#### Step 4 — Apply consent (PATCH the order)
```
PATCH /api/mf/orders/purchases
Body:
{
  "id": "mfp_xxx",
  "consent": {
    "email":    "user@example.com",
    "isd_code": "+91",
    "mobile":   "8660275412",
    "otp":      "123456"             // same OTP again — server re-verifies + consumes
  }
}
```
Server pre-checks `confirmed_at` — if consent already applied, returns the
existing order (no error). After PATCH, order moves
`under_review → pending → confirmed` (the controller polls + auto-confirms
on ONDC).

#### Step 5 — Create payment (UPI or Netbanking)

Investor picks the method on the Invest screen. Both methods hit the **same**
backend endpoint (just `method` field differs in the body):

##### 5a. UPI
```
POST /api/pg/payments/upi
Body:
{
  "amc_order_ids":        [325027],                   // OldId from Step 1
  "payment_postback_url": "https://localhost:7001/api/pg/payments/postback?mfp=mfp_xxx",
  "method":               "UPI",
  "bank_account_id":      925,                        // FP numeric old_id of bank
  "provider_name":        "ONDC"
}
```

##### 5b. Netbanking
```
POST /api/pg/payments/netbanking
Body: same as UPI but "method": "NETBANKING"
```

Both return:
```json
{ "id": 12345, "token_url": "https://..." }
```

**`bank_account_id` is REQUIRED for both methods** on ONDC tenant
(runtime returns 422 `"Bank Account Id could not be null"` if missing,
even though FP docs mark it optional).

**`payment_postback_url` MUST point at our backend**, not the SPA.
The bank does a form POST with `paymentId`/`status`/`failureCode` in the
body — the SPA can't read POST bodies. Backend (PaymentPostbackController)
catches it, logs to `tbl_Fintech_MF_Order_Log`, and 303-redirects to
`/app/mutual-funds/payment-callback` (web) or `/appmutualfund/payment`
(app, when `?source=app` is added to the postback URL).

#### Step 6 — Redirect user to `token_url`

Web does:
```javascript
window.location.href = payment.token_url;
```

**SAME WINDOW** — no popups, no new tabs. Auth cookies survive.
Sandbox token_url chains through HDFC ATOM UAT
(`merchant.now.hdfc.bank.in/epi-ui/?uniqueKey=ATOMCAMSINTE_…`); production
goes to the real bank netbanking / UPI app.

**Mobile equivalent**: open `token_url` **in the WebView** (NOT in
SafariViewController / Custom Tab — those break the redirect chain back
to your app). Use the same WebView that handled DigiLocker / eSign / mandate
postbacks earlier.

#### Step 7 — Bank/PG completes payment → form POST → backend → SPA redirect

| Phase | What happens | Where |
|---|---|---|
| User completes payment | clicks Submit on bank page | bank's UI |
| Bank form-POSTs result | `paymentId`, `status`, `failureCode`, `failureReason` | → `https://api/api/pg/payments/postback?mfp=mfp_xxx&source=app` |
| Backend catches form POST | logs audit row, builds query string | `PaymentPostbackController` |
| Backend 303-redirects | to thin SPA page with data on query params | → `/appmutualfund/payment?mfp=&id=&status=&failureCode=&reason=` |
| SPA renders / deep-links | for app: deep-links `islamicly://mf-callback?flow=payment&...` then native app polls | mobile native app |

#### Step 8 — Native app polls `/snapshot` for the final state

After the deep-link arrives, the native app:
1. Reads `id` + `mfp` + `status` from the URL params
2. Closes the WebView
3. Polls **BOTH** every 2 sec for up to 90 sec:
   ```
   GET /api/pg/payments/{id}/snapshot
   GET /api/mf/orders/purchases/{mfp}/snapshot
   ```
4. Terminal states:
   | Payment status | Order state | Render |
   |---|---|---|
   | `SUCCESS`/`APPROVED`/`SETTLED` | `successful`/`submitted` | ✅ Success card |
   | `FAILED`/`REJECTED` | — | ❌ Failure card with `failure_code` mapped to human text |
   | any | `cancelled` | ❌ Cancelled card |
   | still pending after 90 sec | still pending | ⏳ "We'll notify you" + push notification later |

5. The order detail page shows the **EXACT live state** (not invented
   "Processed" labels) with this 5-step timeline for ONDC:
   ```
   ✓ Under Review → ✓ Pending → ✓ Confirmed → ✓ Submitted → … Successful
   ```
   Or for terminal failure:
   ```
   ✓ Placed → ✕ Failed
   ```

### Failure codes — friendly hints

Backend maps `failure_code` from `tbl_Fintech_Payment.FailureCode` to user
text. Mobile should mirror the same mapping. See §19.4 for the full table.
Sample:
- `fp_payment_expired` → "You took too long. Please try again."
- `insufficient_funds` → "Insufficient funds. Retry after topping up."
- `tpv_failed` → "Payment was from a different bank account. Use the verified one."
- `user_cancelled` → "You cancelled. Try again whenever ready."
- unknown code → `"Payment failed ({code}). Please retry."`

### When can the order be cancelled?

Per FP docs: **only `pending` orders can be cancelled**. Once it hits
`confirmed` / `submitted` / `successful` the cancel endpoint returns 400.
UI must hide the Cancel button for non-`pending` states (web already does
this; mobile should match).

```
POST /api/mf/orders/purchases/{mfp_xxx}/cancel
Body: {}
```

---

## 27.7 SIP / Recurring Plan — End-to-End (mirror the Web flow)

### Prerequisites
- Same KYC + MFIA + bank pre-checks as lump-sum
- **An APPROVED mandate** (eNACH or UPI Autopay)
- If no mandate exists, run the mandate-create flow first (§22 + Mandate sections)

### Step 1 — Send OTP
```
POST /api/mf/orders/purchase-plans/send-otp
Headers:
  Plan-Intent-Key: <uuid>             ← unique per Invest-as-SIP intent
Body:
{
  "mobile":  "8660275412",
  "email":   "user@example.com",
  "isdCode": "+91"
}
```
Response includes `sandboxOtp: "123456"`.

### Step 2 — Verify OTP (peek)
```
POST /api/mf/orders/purchase-plans/verify-otp
Headers:
  Plan-Intent-Key: <same uuid>
Body: { "otp": "123456" }
```

### Step 3 — Create the plan
```
POST /api/mf/orders/purchase-plans
Headers:
  Plan-Intent-Key: <same uuid>
  Idempotency-Key: <new uuid>
Body:
{
  "mf_investment_account":   "mfia_c3387215f10c4e2b98e9e3a8edb0d703",
  "scheme":                  "INF084M01093",
  "amount":                  5000,
  "installment_day":         5,                       // 1..28
  "frequency":               "monthly",
  "number_of_installments":  12,
  "payment_method":          "mandate",
  "payment_source":          "4",                     // mandate.id as string
  "systematic":              true,
  "gateway":                 "ondc",
  "auto_generate_installments": true,
  "initiated_by":            "investor",
  "initiated_via":           "mobile_app",
  "user_ip":                 "127.0.0.1",
  "consent": {
    "email":    "user@example.com",
    "isd_code": "+91",
    "mobile":   "8660275412",
    "otp":      "123456"
  }
}
```

Response: the plan object. State progression on ONDC:
```
created → review_completed → confirmed → submitted → active
```
Our controller polls for 15s + PATCHes `state=confirmed` on `review_completed`
automatically. Webhook handler does it as fallback for missed polls.
**Mobile app does nothing here** — just wait for the response. If state =
`active` in the response, the SIP is live. If still in transition, the
SIP list page will show it as such until it activates within ~30 seconds.

Sample response (real one, healthy):
```json
{
  "object": "mf_purchase_plan",
  "id": "mfpp_b6cdf34b931a442bae10fc3a24ded254",
  "state": "active",
  "activated_at": "2026-05-22T20:42:30+05:30",
  "next_installment_date": "2026-06-05",
  "remaining_installments": 12,
  "payment_method": "mandate",
  "payment_source": "4",
  ...
}
```

### Step 4 — Show success → navigate to SIPs tab
That's it. No payment URL, no redirect — the mandate handles all future
debits.

### Installment lifecycle (FP-driven, app does nothing per installment)

On each `installment_day`:
1. FP creates `mf_purchase` order with `Plan = mfpp_xxx` set
2. Debits the linked mandate via NPCI eNACH
3. AMC allots units, populates folio
4. Webhooks fire: `mf_purchase.created → submitted → successful`
5. Our dispatcher snapshots into `tbl_Fintech_MF_Purchase_Order`
6. The Orders tab gets a new row. The "Installments" expander on the
   SIPs tab shows it under the parent plan
7. Plan's `remaining_installments` decrements 12 → 11 → 10…
8. When 0, plan auto-transitions to `completed`

### SIP actions reference

| Button | Visible when | Endpoint |
|---|---|---|
| Sync | always | `GET /api/mf/orders/purchase-plans/{id}` |
| Pause | `state ∈ {active}` + `gateway != bse` + no active skip | `POST /api/mf/orders/purchase-plans/{id}/pause { from, to? }` |
| Resume | active skip instruction exists | `POST /api/mf/orders/purchase-plans/skip-instructions/{psi}/cancel` |
| Cancel | `state ∈ {active}` on ONDC; also `created, submitted` non-ONDC | `POST /api/mf/orders/purchase-plans/{id}/cancel { cancellationCode }` |
| Simulate Installment (sandbox only) | `state ∈ {active}` | `POST /api/mf/orders/purchase-plans/{id}/simulate-installment {}` |

Full details for each action including FP runtime constraints documented
in §27.5.3.

### Cancellation codes (required)
- `investment_returns_not_as_expected` (backend default)
- `change_in_financial_goals`
- `fund_performance_issues`
- `personal_financial_constraints`
- `others`

FP enforces presence — backend defaults to `investment_returns_not_as_expected`
if caller doesn't pick.

---

## 28. App-team Quick Start (the only checklist they need)

1. **Run all migrations** from `Claude-db-queries.txt`.
2. Set `Finprim:WebhookSecret` to Cybrilla's value once they provide
   it (mandatory for production go-live).
3. Set `Finprim:Spa:BaseUrl` + all `Finprim:Postback:*` URLs to your
   production domain.
4. Set `Runtime:IsProduction` to `1`.
5. Register webhooks once via `POST /api/webhooks/bootstrap?url=https://<prod-api>/api/webhooks/finprim-events`.
6. Confirm the webhook HMAC by sending a test event from Cybrilla.
7. Smoke-test: a real ₹500 lumpsum on one user, end-to-end through
   bank, until folio + units appear in Transactions.
8. Smoke-test: a real ₹100/month SIP with eNACH mandate, end-to-end
   until plan is `active`.

---

---

## 29. KYC Forms API — POA (replaces `/v2/kyc/requests`, early access)

Per Cybrilla support (Manisha T, 2026-05-22): for **fresh KYC** submissions
on sandbox / production, switch from the old `/v2/kyc/requests` family to
the **POA KYC Forms API** at `/poa/kyc_forms`. Public docs:
https://poa.cybrilla.com/docs/additional-apis/kyc-forms

### 29.1 Tenancy & auth

| | KYC Request (legacy) | KYC Form (new) |
|---|---|---|
| Host | `s.finprim.com` | `api.sandbox.cybrilla.com` (POA) |
| Path prefix | `/v2/kyc/requests` | `/poa/kyc_forms` |
| Tenant header | `x-tenant-id: islamiclyondc` | `x-tenant-id: cybrillarta` |
| Bearer token | `islamiclyondc_test_xxx` | POA pre-verification token (same as bank pre-verify) |
| Webhook events | `kyc_request.*` | `kyc_form.*` (8 events) |

Our `ITenantTokenProvider` already caches the `cybrillarta` token (used for bank pre-verification). No new credentials required.

### 29.2 Endpoints (wrapped at our backend)

| Local route | FP path | Description |
|---|---|---|
| `POST   /api/kyc/forms`                       | `POST /poa/kyc_forms`                                 | Create kyc_form |
| `PATCH  /api/kyc/forms`                       | `PATCH /poa/kyc_forms`                                | Update kyc_form (id in body) |
| `GET    /api/kyc/forms/{id}`                  | `GET /poa/kyc_forms/{id}`                             | Fetch live state |
| `POST   /api/kyc/forms/{id}/signature`        | `POST /poa/kyc_forms/{id}/signature`                  | Upload signature (multipart) |
| `POST   /api/kyc/forms/{id}/retry-proof`      | `POST /poa/kyc_forms/{id}/retry_proof_details_fetch`  | Re-issue DigiLocker URL on failed proof |
| `POST   /api/poa/files`                       | `POST /poa/files`                                     | Generic file upload (cancelled cheque etc.) |

### 29.3 State machine

```
under_review → created → awaiting_esign → awaiting_submission → submitted
       │            │             │                  │
       │            │             │                  └── failed
       │            │             └── (esign in progress)
       │            └── expired (7 days inactivity)
       └── failed (e.g. ineligible_for_kyc_modification)
```

### 29.4 Flow (modify type — currently the only documented type)

#### Step 1 — Check eligibility
Per docs: investor's KYC at KRA must be `validated`, `verified or registered`,
or `onhold`. Otherwise Cybrilla rejects with `reason: ineligible_for_kyc_modification`.

#### Step 2 — Create
```
POST /api/kyc/forms
Body:
{
  "type":          "modify",
  "pan":           "AAAPA3751A",
  "name":          "John Doe",
  "date_of_birth": "2000-01-02",
  "proof_details_callback_url": "https://app.../proof_callback",
  "esign_callback_url":         "https://app.../esign_callback"
}
```
Response: `kyc_form` object with `status: "under_review"`.

#### Step 3 — Poll/wait for `created`
Webhook `kyc_form.created` fires (or GET it manually). Now:
- `requirements.fields_needed` shows what's missing (e.g. `["identity_proof", "address", "signature"]`)
- `proof_details.fetch_url` gives the DigiLocker URL — redirect investor there for Aadhaar fetch

#### Step 4 — Fill remaining fields
```
PATCH /api/kyc/forms
Body:
{
  "id":                 "kycf_xxx",
  "email_address":      "user@example.com",
  "phone_number":       { "isd": "+91", "number": "9876543210" },
  "residential_status": "resident",
  "gender":             "male",
  "marital_status":     "unmarried",
  "father_name":        "Davy Johns",
  "occupation_type":    "private_sector_service",
  "aadhaar_number":     "1210",
  "country_of_birth":   "in",
  "place_of_birth":     "Mumbai",
  "income_slab":        "above_10lakh_upto_25lakh",
  "pep_details":        "no_exposure",
  "citizenship_countries": ["in"],
  "nationality_country":   "in",
  "tax_residency_other_than_india": false,
  "geo_location":       { "latitude": 19.0760, "longitude": 72.8777 }
}
```

#### Step 5 — Upload signature
```
POST /api/kyc/forms/{kycf_xxx}/signature
Multipart: file=<png|jpg|jpeg|pdf, max 5 MB>
```

#### Step 6 — DigiLocker journey
Redirect investor to `proof_details.fetch_url`. On completion they're sent
to `proof_details_callback_url`. After return, refresh the form:
```
GET /api/kyc/forms/{kycf_xxx}
```
If `proof_details.status == "failed"`, retry:
```
POST /api/kyc/forms/{kycf_xxx}/retry-proof
```

#### Step 7 — Form auto-transitions to `awaiting_esign`
Once `requirements.fields_needed` is empty AND signature uploaded AND
proof fetched, `esign_details.esign_url` is populated. Redirect investor
there. On completion they land on `esign_callback_url`.

#### Step 8 — Cybrilla submits to KRA
`status` flips to `awaiting_submission` then `submitted` (or `failed` with
`reason`). Webhook `kyc_form.submitted` fires.

### 29.5 Field enums (mobile UI cheat-sheet)

| Field | Allowed values |
|---|---|
| `residential_status` | `resident` (only) |
| `gender` | `male`, `female`, `transgender` |
| `marital_status` | `married`, `unmarried`, `others` |
| `occupation_type` | `business`, `professional`, `retired`, `housewife`, `student`, `public_sector_service`, `private_sector_service`, `government_service`, `agriculture`, `doctor`, `forex_dealer`, `service`, `others` |
| `income_slab` | `upto_1lakh`, `above_1lakh_upto_5lakh`, `above_5lakh_upto_10lakh`, `above_10lakh_upto_25lakh`, `above_25lakh_upto_1cr`, `above_1cr` |
| `pep_details` | `pep`, `related_pep`, `no_exposure` |

### 29.6 Required-by-eventually fields

These are optional at create time but mandatory **before** the form can
transition to `awaiting_esign`. The `requirements.fields_needed` array
tells you which ones are still missing. Watch that array on every PATCH
response.

`email_address`, `phone_number`, `residential_status`, `gender`,
`marital_status`, (`father_name` if unmarried/others, else `spouse_name`),
`occupation_type`, `aadhaar_number`, `country_of_birth`, `place_of_birth`,
`income_slab`, `pep_details`, `citizenship_countries`, `nationality_country`,
`tax_residency_other_than_india`, `geo_location`, identity proof (DigiLocker),
address proof (DigiLocker), signature (uploaded).

### 29.7 Sandbox simulation PAN tricks

| PAN suffix | Outcome |
|---|---|
| `XXXPXNNNNX` (ends with letter) | `submitted` — KRA accepted |
| `XXXPXNNNNR` (ends with R) | `failed` — KRA rejected |

Combine with PAN-verification simulation (referenced in KYC Check section)
for end-to-end testing.

### 29.8 Webhook events (auto-subscribed via bootstrap)

| Event | Trigger |
|---|---|
| `kyc_form.under_review` | Initial create succeeded |
| `kyc_form.created` | Eligibility passed, form ready for data |
| `kyc_form.awaiting_esign` | All fields + signature + proof complete |
| `kyc_form.awaiting_submission` | Esign successful, Cybrilla submitting to KRA |
| `kyc_form.submitted` | KRA accepted |
| `kyc_form.failed` | Terminal failure — see `reason` |
| `kyc_form.expired` | 7 days inactivity |
| `kyc_form.updated` | Any field change (PATCH) |

Dispatcher logs each event to `tbl_Fintech_Webhook_Event_Log`. Structured
persistence into `tbl_Fintech_Kyc_Form` runs through
`Usp_Upsert_Fintech_Kyc_Form` (see DB migration `Claude-db-queries.txt`).

### 29.9 Files API (separate from kyc_form signature)

For uploads not tied to a specific kyc_form (e.g. cancelled cheque for
NRI banks, generic compliance docs):

```
POST /api/poa/files
Multipart:
  purpose=bank account proof
  file=<pdf|jpeg|jpg|png, max 10 MB>
```
Returns `{ id: "file_xxx", created_at: "..." }`. The `file_xxx` id is then
referenced from other resources (e.g. bank account creation).

### 29.10 Migration strategy

| Stage | Approach |
|---|---|
| Today | Backend wrappers live in parallel — both `/api/kyc/requests` (legacy) and `/api/kyc/forms` (new) coexist |
| Frontend wiring | New KYC wizard component points at `/api/kyc/forms`. Existing wizard stays for users already mid-flow on legacy IDs |
| Long-term | Once mobile + web are on KYC Forms exclusively, deprecate the legacy `KycController` endpoints |

The two paths are **NOT** linked at the DB level — `tbl_Fintech_Kyc_Request_Status`
(legacy) and `tbl_Fintech_Kyc_Form` (new) are independent. Investor profile
linkage happens via `pan` matching.

---

## 29.9 KYC — App Team Integration Spec (mirror the web client EXACTLY)

> **Authoritative list (2026-06-08).** These are the **only** KYC endpoints the live web app calls. The mobile app must integrate the SAME set, in the same order. The legacy `/api/kyc/requests` (`/v2/kyc/requests`) wizard — `createKycRequest`, `updateKycAddress`, `createESign`, `confirmESign`, `simulate`, financial/address steps — is **being retired. DO NOT integrate it.**

Two API groups, called in order: **(1) Pre-Verification** = the gate → **(2) POA KYC Forms** = the actual KYC submission.

### Group 1 — Pre-Verification (call FIRST; it decides everything)

Source of truth: `pre-verification.service.ts`. Base: `{mfurl}pre-verifications`.

| Method | Endpoint | Body | When / Purpose |
|---|---|---|---|
| `POST` | `/api/pre-verifications/readiness` | `{ pan, isWeb }` | KRA readiness check by PAN (Stage 1). |
| `POST` | `/api/pre-verifications/pan-validate` | `{ pan, name, dateOfBirth, isWeb }` | Validate PAN+Name+DOB together (Stage 2). |
| `GET`  | `/api/pre-verifications/{id}/poll` | — | Re-fetch a `pv_xxx` while `overallStatus=accepted` until `completed`. |
| `GET`  | `/api/pre-verifications/my-status` | — | **Call on entry / resume.** Returns latest row + `nextAction`. **All routing keys off `nextAction`.** |

**Route purely on `nextAction` (from my-status / readiness):**

| `nextAction` | What it means | App does |
|---|---|---|
| `run_pan_validation` *(readiness=verified)* | Investor already KYC-compliant at KRA | **Skip KYC entirely → go to invest.** Web shows the `already_kyc_valid` screen. |
| `start_fresh_kyc` | No KRA record (`kyc_unavailable`/`unknown`) | Create KYC Form `type=fresh` (Group 2). |
| `start_modify_kyc` | KRA record exists but incomplete (`kyc_incomplete`) | Create KYC Form `type=modify` (Group 2). |
| `poll` | Still processing | Poll `/{id}/poll` every 2–3 s. |
| `retry_readiness` | Transient (`upstream_error`) | Retry `POST /readiness`. |
| `fix_pan` / `fix_name` / `fix_dob` / `link_aadhaar_at_itd` | Stage-2 field mismatch | Re-collect that field, re-run `pan-validate`. |
| `run_bank_validation` | PAN trio matched | Proceed to bank add/verify. |

> ⚠️ **PAN-lock guard (mirror this).** Picking the wrong KYC Form `type` locks the PAN for 7 days at Cybrilla (`reason=ineligible_for_kyc_modification`). Before POSTing a form, re-read `my-status` and only allow `type=fresh` when `readiness.code ∈ {kyc_unavailable, unknown}` / `nextAction=start_fresh_kyc`, and `type=modify` when `readiness.status ∈ {verified, validated, registered, onhold, successful}` **or** `readiness.code=kyc_incomplete` / `nextAction=start_modify_kyc`. The web client does this in `kyc-form.component.ts createForm()`.

### Group 2 — POA KYC Forms (the actual KYC submission)

Source of truth: `kyc-form.service.ts`. Base: `{mfurl}kyc/forms`. These are the **only 8 interactions** the live `kyc-form.component.ts` makes:

| # | Method | Endpoint | Purpose |
|---|---|---|---|
| 1 | `POST` | `/api/kyc/forms` | Create. Body `{ type, pan, name, date_of_birth, proof_details_callback_url, esign_callback_url }`. |
| 2 | `PATCH` | `/api/kyc/forms` | Fill remaining fields (id in body): email, phone, gender, marital_status, father/spouse name, occupation, income_slab, aadhaar last-4, pep_details, country/place of birth, nationality, citizenship, tax-residency flag, geolocation. Send everything in `requirements.fields_needed`. |
| 3 | `GET`  | `/api/kyc/forms/my-status` | Resume — latest form + `nextAction` ∈ {`fill_fields`,`upload_signature`,`fetch_proof`,`esign`,`done`,`wait`}. |
| 4 | `GET`  | `/api/kyc/forms/{id}/poll` | Force live FP re-fetch of one form. |
| 5 | `GET`  | `/api/kyc/forms/{id}` | Fetch one form's live state. |
| 6 | `POST` | `/api/kyc/forms/{id}/signature` | Multipart wet-signature upload (png/jpg/jpeg/pdf, ≤5 MB). |
| 7 | `POST` | `/api/kyc/forms/{id}/retry-proof` | Re-issue DigiLocker URL when `proof_details.status=failed`. |
| 8 | redirect | `proof_details.fetch_url` (DigiLocker) and `esign_details.esign_url` (eSign) | Open in browser/WebView; investor returns via the two callback URLs. |

**Lifecycle:** `under_review → created → (PATCH fields + DigiLocker proof + signature upload) → awaiting_esign → eSign → awaiting_submission → submitted` (terminal success). `failed` (read `reason`) and `expired` (7-day inactivity) are terminal.

### App-team rule of thumb
1. On MF entry → `GET /api/pre-verifications/my-status`.
2. If `readiness=verified` → go invest (no form).
3. Else create the right form `type` (fresh vs modify) per `nextAction` — **type is auto-derived from readiness, never a user choice** (wrong type = 7-day PAN lock). Then drive Group-2 endpoints by `my-status.nextAction` until `submitted`. While `under_review`, poll `/{id}/poll` every ~5 s.
4. After `submitted`, **re-run readiness** before the first purchase to confirm `readiness=verified` (the gate the purchase tenant checks) — see §29.9.1.
5. On `status=failed`/`expired`, **branch on `reason`** (don't blindly follow `nextAction=start_fresh_kyc`) — see §29.9.2 (ineligible_for_fresh → readiness-route; ongoing-exists → resume; expired → re-derive type & restart; contradiction → stop + support message).
6. Never call any `/api/kyc/requests*` endpoint.

### 29.9.1 ⚠️ "KYC submitted ≠ ready to invest" — the readiness cross-check (CRITICAL)

**The single most important gotcha in this whole flow.** A KYC Form reaching `status=submitted` ONLY proves the form was accepted — it does **NOT** mean the investor can purchase. The MF purchase tenant (`islamiclyondc`) gates orders on **pre-verification readiness**, a *separate* signal. If you place an order while readiness is still `failed`/`kyc_unavailable`, Cybrilla rejects the order with **`failure_code: non_kyc_compliant_investor_error`** (the order is created then `failed` ~1s later).

These two signals can legitimately disagree for a window:

| Signal | Source | Says |
|---|---|---|
| KYC Form status | `GET /api/kyc/forms/my-status` → `kycForm.status` | `submitted` ✓ (form accepted) |
| Purchase readiness | `GET /api/pre-verifications/my-status` → `readiness.status` | may still be `failed` / `kyc_unavailable` |

**Why they disagree:**
- **Sandbox:** the form's compliance is faked and tied to the form's 7-day `expires_at`; readiness on the purchasing side does **not** auto-sync, so it can stay `kyc_unavailable` even after a `submitted` form. (This is a sandbox-only artifact — see §31.x.)
- **Production:** brief KRA propagation lag — readiness flips to `verified` once the KRA validates the freshly-submitted KYC (usually minutes).

**MANDATORY UI/UX rule (web does this; app MUST mirror it):**
> On any "KYC done / ready to invest" screen, compute **`purchaseReady = (kycForm.status === 'submitted' || alreadyKycValid) && readiness.status === 'verified'`**. Only when `purchaseReady` is true do you show "ready to invest" and enable the invest CTA. When the form is `submitted` but readiness ≠ `verified`, show **"KYC submitted — awaiting KRA confirmation"** and **disable the invest button**, with a "Re-check readiness" action that re-calls `GET /api/pre-verifications/my-status` (or re-runs `POST /readiness`).

Web implementation: `kyc-form.component.ts` → `purchaseReady` computed signal + the amber "awaiting KRA confirmation" banner on the `done` screen; the "Browse Funds →" button is disabled and relabelled "Investing locked until KYC confirmed" until readiness clears. The `done` screen caches readiness from `refresh()` into `readinessStatus`/`readinessCode` signals for this cross-check.

**Pre-purchase guard (belt-and-braces):** even with the UI gate, re-run `POST /api/pre-verifications/readiness` (or read `my-status`) immediately before the first `POST /api/mf/orders/purchases` and refuse to place the order unless `readiness.status === verified`. This avoids the `non_kyc_compliant_investor_error` round-trip entirely.

### 29.9.1b Doc-vs-tenant divergence (verified against the official pages, 2026-06-08)

Authoritative sources (refer to these ONLY for KYC):
- Pre-Verification: https://poa.cybrilla.com/docs/additional-apis/pre-verifications
- KYC Forms: https://poa.cybrilla.com/docs/additional-apis/kyc-forms

Two places where our implementation intentionally goes **beyond the public docs** — keep them, don't "correct" back:

1. **`type: "fresh"` is used even though the KYC Forms doc documents `type: "modify"` ONLY.** The public page lists `type (enum): "modify"` and only describes modify-eligibility (KRA must already have a `validated/verified/registered/onhold` record). However, **Cybrilla support (Manisha T) explicitly instructed us to use the KYC Form API with `type = fresh` for fresh KYC** (verbatim: *"We recommend using the KYC Form API instead of the KYC Request API when submitting a fresh KYC request… Please make sure to set type = fresh when initiating a fresh KYC request through the KYC Form API. You can use the preverification credentials to access the KYC Form API"*). Sandbox has accepted fresh forms through to `submitted`. So we create `fresh` when readiness is `kyc_unavailable`/`unknown`, using the **pre-verification (`cybrillarta` POA) credentials** for auth. This is an officially-sanctioned tenant capability not yet reflected in the public docs — **do not remove it**, and **do not reintroduce the legacy `/v2/kyc/requests` (KYC Request API)** which support told us to stop using for fresh KYC.
2. **`reason: "ineligible_for_fresh_kyc"` is NOT in the docs.** The page only enumerates `ineligible_for_kyc_modification` (the modify-side rejection) and "an ongoing kyc_form already exists for the PAN". `ineligible_for_fresh_kyc` is the tenant/sandbox rejection for the undocumented `fresh` type when the PAN already has a KRA record — handle it per §29.9.2 (re-read readiness → modify / invest / stop).

Everything else in our KYC code (readiness codes `kyc_unavailable|kyc_incomplete|upstream_error|unknown`; PAN/name/DOB sim patterns `XXXPXNNNNX` / `Lord Voldemort` / `2000-01-01`; KYC-form sim `XXXPXNNNNX`→submitted, `XXXPXNNNNR`→failed; lifecycle `under_review→created→awaiting_esign→awaiting_submission→submitted`) **matches the official pages exactly.**

### 29.9.2 KYC Form FAILURE & edge-case handling (reason-aware — mirror exactly)

**Rule:** when a KYC Form ends in `status=failed` (or `expired`), the backend's `my-status` returns `nextAction=start_fresh_kyc` — **do NOT blindly follow it** (that loops you straight back into another fresh create that fails again). **Branch on the `reason` field instead.** The web client does this in `kyc-form.component.ts` (`failureKind`/`recoverFromFailure`). All of the below are decisions the app team must replicate.

| `reason` on the failed form | What it means | Correct action |
|---|---|---|
| `ineligible_for_fresh_kyc` | The KRA already has a KYC record for this PAN → a **fresh** form is invalid. | **Re-read readiness and route on it** (see decision table below). Do NOT just retry fresh. |
| `An ongoing KYC Form already exists for this PAN` | A previous form is still in progress (on Cybrilla's side; a different `kycf_` id). | **Resume the in-progress form**, don't create a new one. See "Resume" below. |
| anything else (KRA rejected) | Genuine KRA rejection. | Show the reason; offer a fresh restart (`type` re-derived from readiness). |

#### Handling `ineligible_for_fresh_kyc` → re-read readiness, then:

| readiness after re-read | Meaning | App does |
|---|---|---|
| `readiness.status = verified` | Already KYC-compliant | **No form** → `already_kyc_valid` screen → go invest (subject to the §29.9.1 cross-check). |
| modify-eligible (`status ∈ {validated, registered, onhold, successful}` or `code = kyc_incomplete` or `nextAction = start_modify_kyc`) | Record exists, can be updated | Create a **`type=modify`** form. |
| **contradiction** — readiness `code = unknown` (or otherwise NOT verified/modify-eligible) while the form says fresh-ineligible | Cybrilla can't classify the KYC (`unknown` = "non-compliant, reason not known"). Fresh is rejected AND modify is unsafe (wrong modify locks the PAN 7 days). | **STOP. Do not create any form.** Show a "KYC needs attention / try later / contact support" message and a "Re-check status" button only. No auto-retry, no auto-create. (Policy chosen 2026-06-08.) |

> The web client tracks this dead-end with a `recoveryBlocked` flag: once hit, the failed screen **hides the recovery CTA** (so the user can't click "Switch to Modify" repeatedly into the same wall) and shows only the support message + "Re-check status". The flag resets on a fresh page load so a genuinely-changed readiness can proceed.

#### "Resume existing KYC" (`An ongoing KYC Form already exists…`)
- The failed row you're holding is the rejected *create attempt*, **not** the blocking in-progress form (different `kycf_` id).
- **Backend support (DB):** the resume proc `Usp_Get_Fintech_Kyc_Form_ByUser` was changed (2026-06-08) to return the latest **ACTIVE** (non-terminal) form first — `under_review / created / awaiting_esign / awaiting_submission` — and only fall back to the newest terminal row if none are active. So `GET /api/kyc/forms/my-status` now returns the real in-progress form, and the app routes to its stage.
- If the ongoing form is held purely on Cybrilla's side (no local active row), `my-status` may still return the failed row → show "a KYC is already in progress; wait for it to complete or expire (up to 7 days)".

#### `expired` form restart
- A `submitted` form NEVER becomes `expired` — `expires_at` (7 days) only governs **completing an in-progress** form. An abandoned `created/awaiting_esign/awaiting_submission` form → `expired`.
- Cybrilla does **not** allow resuming an expired/failed form — a brand-new form must be created.
- Restart must **re-derive `type` from readiness** (not hardcode `modify`) and preserve the entered PAN/name/DOB. The web client's `restart()` re-reads readiness only (avoids the `my-status` returning the stale expired row → which would bounce back to the `expired` screen forever, because `mapNextAction` checks `status` before `nextAction`).

#### `under_review` auto-poll (don't leave the user on a dead spinner)
- After creating a form (`under_review`) the UI must **poll `GET /api/kyc/forms/{id}/poll` every ~5 s** until the status advances (to `created`/`fill_fields`/etc.). The web client starts this poller at the end of `refresh()` (so create / restart / resume / manual-refresh all get it), not only on first load.

#### KYC `type` is NEVER a user choice
- The fresh-vs-modify `type` is auto-derived from readiness and shown **read-only** ("KYC type (auto-detected): Fresh / Modify"). A user-editable type dropdown is a footgun (wrong pick = 7-day PAN lock) and must not be reintroduced.

#### Sandbox testing note (PANs)
- The fresh-vs-modify outcome is **not** determined by the PAN pattern. `XXXPXNNNNX` only controls **PAN validation** and **KYC-Form submission** simulation — it does NOT make a PAN "fresh-eligible". Whether a sandbox PAN is fresh-eligible (`readiness=kyc_unavailable` AND form accepts fresh) vs already-has-KYC (`ineligible_for_fresh_kyc`) is seeded by Cybrilla. Many sandbox PANs return a contradictory state (`readiness=unknown` + `ineligible_for_fresh_kyc`) for which there is no valid action — request fresh-eligible / verified test PANs from Cybrilla support for end-to-end testing.

### 29.9.3 KYC status — single source of truth & data model

**Where to check a user's KYC for ANY scenario → the view `vw_Fintech_User_Kyc_Status`** (or `EXEC Usp_Get_Fintech_User_Kyc_Status_V2 @UserId`). It returns ONE row per user with two distinct bits:

| Column | Use for | Logic |
|---|---|---|
| `IsKycCompliant` | "Has the user finished KYC?" (display / onboarding gating) | readiness verified across stages **OR** KYC form `submitted` |
| `IsPurchaseReady` | **"Can the user place an MF order?"** — check THIS before `POST /v2/mf_purchases` | `readiness.status = verified` only |

Plus `CompliancePath`, `KycForm_Status/Reason`, `PreV_*`, and identity (`Pan`/`Name`/`DateOfBirth`).

**Why `IsKycCompliant` ≠ `IsPurchaseReady`:** a `submitted` KYC form does not by itself make the purchase tenant accept the investor; that needs `readiness=verified` (see §29.9.1). So `IsKycCompliant` can be 1 while `IsPurchaseReady` is 0 during the KRA-propagation window.

**Multiple rows vs one row — the correct model (do NOT collapse the snapshot tables):**

| Table | Key | Rows/user | Approach |
|---|---|---|---|
| `tbl_Fintech_Pre_Verification` | `FpPreVerificationId` (`pv_xxx`) | **multiple** (one per stage/attempt) | keep all (audit). The VIEW aggregates them. |
| `tbl_Fintech_Kyc_Form` | `FpKycFormId` (`kycf_xxx`) | **multiple** (one per form attempt) | keep all; resume proc prefers active-first. |
| `tbl_Fintech_MF_Purchase_Order` / `_Bank_Account` / etc. | their Fp id | **multiple** | keep all (one row per Cybrilla object). |
| `tbl_Fintech_Investor_Profile` | `UserId` | **ONE** | upsert (`Usp_Upsert_Fintech_Investor_Profile`); a user has exactly one `ip_xxx`. |
| **`vw_Fintech_User_Kyc_Status`** (the view) | `UserId` | **ONE** | aggregated verdict — read this, never the raw snapshot tables, for "is the user OK". |

Rule of thumb: snapshot tables = **one row per Cybrilla object** (multiple per user, for audit); the **view collapses them to one verdict per user**; only Investor Profile is genuinely one-row-per-user.

### 29.9.4 KYC EXPIRY — what expires, what doesn't, sandbox vs production

**The single most misunderstood thing.** There are TWO different "expiries"; do not conflate them:

| What | `expires_at` / TTL | Meaning | What to do |
|---|---|---|---|
| **KYC Form** (`kycf_xxx`) `expires_at` | **7 days from creation** | The window to **COMPLETE an in-progress form** (DigiLocker + signature + eSign + submit). | Only matters if the investor **abandons** a form mid-way. |
| **KYC compliance at the KRA** | does **NOT** expire in 7 days | Once a form is `submitted` and the KRA registers/validates it, the investor stays KYC-compliant (years, until the KRA itself revokes). | Nothing — it's durable. |

**Critical rule:** a form that reaches **`status=submitted` (terminal success) NEVER becomes `expired`.** `expires_at` only governs forms still in `created`/`awaiting_esign`/`awaiting_submission`. A submitted form's compliance is permanent (production).

**`expired` lifecycle (Case B — abandoned form):**
- A `created`/`awaiting_esign`/`awaiting_submission` form with no investor action for 7 days → `kyc_form.expired` webhook → `status=expired`.
- Cybrilla does **NOT** allow resuming an expired (or failed) form — a **brand-new form must be created**.
- Backend `/api/kyc/forms/my-status` maps `expired` (and `failed`) → `nextAction=start_fresh_kyc`. **Do NOT blindly follow it** — see the restart rule below.

**Restart-after-expiry rule (web `kyc-form.component.ts restart()`; mirror it):**
- Re-read pre-verification readiness and **re-derive the form `type`** (fresh vs modify) from it — do NOT hardcode `modify` (an expired *fresh* KYC would then be refused by the PAN-lock guard → dead-end).
- Preserve the entered PAN/name/DOB; land the user on the `start` screen to create a NEW form.
- Do NOT just call refresh() — the stale `expired` row is still returned by my-status and `mapNextAction` checks `status` before `nextAction`, which would bounce the user back to the `expired` screen forever.

**Showing `expires_at` to the user:**
- On an **in-progress** form → fine to show "complete within N days".
- On a **`submitted`** form → do **NOT** show `expires_at` as a KYC-expiry; it's meaningless there and misleads the user into thinking their KYC lapses in 7 days.

**Sandbox vs production divergence (important for testers):**
- **Production:** a `submitted` fresh KYC → KRA validates → `readiness` flips to `verified`; the investor is durably compliant and can purchase. The 7-day expiry is irrelevant to a completed KYC.
- **Sandbox:** the `submitted` form's compliance is faked and effectively tied to the form's own 7-day `expires_at`; `readiness` for synthetic PANs often **stays `kyc_unavailable`/`unknown` even after `submitted`**, and may lapse after the 7-day window. This is a **sandbox-only artifact** — it produces the `non_kyc_compliant_investor_error` on purchase even after a successful KYC. It does NOT happen in production. (This is why testers must request verified-readiness sandbox PANs from Cybrilla — see §29.9.2 sandbox note.)

---

# 30. End-to-End Flow Trace (for App API Integration)

**Updated:** 2026-05-25
**Scope:** Every step from "user opens MF section" → "lumpsum order submitted" / "SIP plan created", with the exact Angular call site, backend endpoint, payload, Cybrilla forwarding, and DB rows touched.

This section is the **single source of truth** for the mobile app team. Every endpoint listed here is live in `FinprimApi`. If something is not here, do not call it.

Base URL: `{environment.mfurl}` = `https://api.islamicly.com/api/` (prod) or `https://localhost:7001/api/` (dev). Every endpoint below is relative to this base.

---

## 30.1 Onboarding — Step-by-Step

### Step 1 — Read aggregated status (call on every screen entry)

```
GET /onboarding/status
→ {
    isKycCompliant, compliancePath, hasInvestorProfile, hasBankAccount,
    hasMfAccount, investorProfileId, mfAccountId, bankAccountId,
    kycMobile, kycEmail, kycMobileIsd, ...
  }
```
Drives all branch decisions ("does the user need KYC? does the user have a bank?"). **Always call this first.**

Web call site: `kyc.service.ts → getOnboardingStatus()`
Backend: `KycController.MyStatus` → DB view `vw_Fintech_User_Kyc_Status` + investor profile join

---

### Step 2 — KYC: Pre-Verification path (PAN found in KRA)

| # | Frontend method | Endpoint | Body | Cybrilla call | DB |
|---|---|---|---|---|---|
| 2.1 | `KycService.startReadiness({pan})` | `POST /pre-verifications/readiness` | `{ pan }` | `POST /v2/pre_verifications` (stage=readiness) | `tbl_Fintech_Pre_Verification` (new row, Stage='readiness') |
| 2.2 | `KycService.pollPreVerification(pvId)` | `GET /pre-verifications/{pv_xxx}/poll` | — | `GET /v2/pre_verifications/{id}` | Updates `ReadinessStatus`, `Stage`, `OverallStatus` |
| 2.3 | `KycService.runPanValidation(pvId, pan, name, dob)` | `POST /pre-verifications/{pv_xxx}/pan-validate` | `{ name, dateOfBirth }` | `POST /v2/pre_verifications/{id}/pan_validation` | Stage='pan_validation', writes PanStatus / NameStatus / DobStatus |
| 2.4 | Poll until OverallStatus='completed' | Same as 2.2 | — | — | — |

Final pre-verification states the app should treat as success: `OverallStatus='completed'` AND `ReadinessStatus IN (verified,validated,registered,onhold,successful)` AND `PanStatus='verified'` AND `NameStatus='verified'` AND `DobStatus='verified'`.

If `ReadinessStatus='unavailable'` → fall through to **KYC Form path** (Step 3).

---

### Step 3 — KYC: Fresh-KYC Form path (PAN NOT in KRA, OR modify)

Created in this session — Angular component: `kyc-form.component.ts`. Replaces the legacy `KycRequest` flow entirely for new users.

| # | Frontend | Endpoint | Body | Cybrilla | DB |
|---|---|---|---|---|---|
| 3.1 | `KycService.createKycForm(...)` | `POST /kyc/forms` | `{ type: "fresh"\|"modify", pan, name, dateOfBirth, proofCallbackUrl, esignCallbackUrl }` | `POST /v2/kyc_forms` | `tbl_Fintech_Kyc_Form` (Status='created') |
| 3.2 | DigiLocker open (browser tab) | Cybrilla's `proof_redirect_url` | — | DigiLocker fetch | Backend webhook updates Status='proof_completed' |
| 3.3 | `KycFormService.uploadSignature(fpId, file)` | `POST /kyc/forms/{id}/signature` | Multipart: `file` (PNG/JPG) | `POST /v2/kyc_forms/{id}/signature` | `SignatureFileId` set |
| 3.4 | `KycFormService.patchForm(fpId, fields)` | `PATCH /kyc/forms/{id}` | `{ email, mobile, gender, maritalStatus, occupation, incomeSlab, pep, placeOfBirth, geolocation }` | `PATCH /v2/kyc_forms/{id}` | Fields written |
| 3.5 | eSign open (NSDL tab) | Cybrilla's `esign_redirect_url` | — | NSDL eSign | Backend webhook → Status='awaiting_submission' → 'submitted' |
| 3.6 | Poll | `GET /kyc/forms/{id}/poll` | — | `GET /v2/kyc_forms/{id}` | — |

Form is "done" when `Status='submitted'`. The user is now KYC-compliant via the form path — `vw_Fintech_User_Kyc_Status.IsKycCompliant=1`.

---

### Step 4 — Investor Profile

Created automatically by backend the first time KYC clears (either path). The app does NOT explicitly call create — it just polls `GET /onboarding/status` until `hasInvestorProfile=true`.

If for any reason it isn't auto-created, the explicit endpoints are:
```
POST /investors/profiles                  → { pan, name, dob, ... }
POST /investors/profiles/addresses        → { profile, line1, line2, postal_code, country, nature }
POST /investors/profiles/phones           → { profile, isd, number }
POST /investors/profiles/emails           → { profile, email }
```
DB: `tbl_Fintech_Investor_Profile`, `tbl_Fintech_Address`, `tbl_Fintech_Phone`, `tbl_Fintech_Email`.

---

### Step 5 — Bank Account add + verify

| # | Frontend | Endpoint | Body | Cybrilla | DB |
|---|---|---|---|---|---|
| 5.1 | `KycService.createBankAccount(req)` | `POST /banking/my-account` | `{ profile, primaryAccountHolderName, accountNumber, type, ifscCode, nriChequeFileId?, isWeb }` | `POST /v2/bank_accounts` | `tbl_Fintech_Bank_Account` (new row) |
| 5.2 | `KycService.startBankVerification(bacId)` | `POST /banking/my-account/{bacId}/verify` | `{ }` | `POST /v2/pre_verifications` (bank_validation) | `PreVerificationId='pv_xxx'`, `VerificationStatus='pending'` |
| 5.3 | `KycService.pollBankVerification(bacId)` | `GET /banking/my-account/{bacId}/verify-poll` | — | `GET /v2/pre_verifications/{pv_xxx}` | `VerificationStatus='completed'\|'failed'`, `VerificationConfidence` (very_high/high/uncertain/low/very_low/zero), `VerifiedAt` |
| 5.4 | List banks for user | `GET /banking/my-status` | — | — | Calls `Usp_Get_Fintech_Bank_Account_Status` V3 (verified-first) |
| 5.5 | **NEW** — list only completed banks | `GET /banking/my-verified-accounts` | — | — | Calls `Usp_List_Fintech_Bank_Accounts_Verified` (`VerificationStatus='completed'`) |

**Sandbox tip:** Account numbers ending in `11XX` (XX=91-99), e.g. `987654321193`, with IFSC `HDFC0001234` get instant `completed/very_high` verification.

The bank picker on the invest page reads from **5.5** so unverified banks are never offered for SIP debit.

---

### Step 6 — MF Investment Account (FATCA + nominee)

| # | Frontend | Endpoint | Body | Cybrilla | DB |
|---|---|---|---|---|---|
| 6.1 | `KycService.createMfAccount(req)` | `POST /mf/accounts` | `{ profile, holdingPattern: "single", fatca: {...}, nominee?: {...} }` | `POST /v2/mf_investment_accounts` | `tbl_Fintech_Mf_Investment_Account` + `tbl_Fintech_Fatca` + `tbl_Fintech_Nominee` |
| 6.2 | Poll | `GET /onboarding/status` | — | — | `hasMfAccount=true` once created |

After this step the user can invest. `GET /onboarding/status` now returns `mfAccountId='mfia_xxx'`.

---

## 30.2 Lumpsum Purchase Flow

Web component: `mutual-fund-invest.component.ts → invest()`.

| # | Step | Endpoint | Body (key fields) | Cybrilla | DB |
|---|---|---|---|---|---|
| L1 | `MfPurchaseService.createOrder(req, idempotencyKey)` | `POST /mf/orders/purchases` (`Idempotency-Key` header) | `{ mfInvestmentAccount, scheme (ISIN), amount, userIp }` | `POST /v2/mf_purchases` | `tbl_Fintech_Mf_Purchase_Order` (state=created) |
| L2 | `MfPurchaseService.sendOrderOtp(mfpId, intentKey, contact)` | `POST /mf/orders/purchases/{mfpId}/send-otp` | `{ mobile, email, isdCode }` | `POST /v2/mf_purchases/{id}/consent_otp` | `tbl_Fintech_Purchase_Otp` |
| L3 | OTP modal shown — user types OTP | — | — | — | — |
| L4 | `MfPurchaseService.verifyOrderOtp(mfpId, intentKey, otp)` | `POST /mf/orders/purchases/{mfpId}/verify-otp` (peek-only) | `{ otp }` | — (we just check it's the right OTP without consuming) | — |
| L5 | PATCH state=confirmed + consent (in same call) | `PATCH /mf/orders/purchases/{mfpId}` | `{ state: "confirmed", consent: { email, isdCode, mobile, otp } }` | `PATCH /v2/mf_purchases/{id}` | `state='confirmed'` |
| L6 | Create payment | `POST /pg/payments/upi` or `POST /pg/payments/netbanking` | `{ amcOrderIds: [mfp.oldId], paymentPostbackUrl, method, bankAccountId, providerName }` | `POST /v2/payments` | `tbl_Fintech_Payment_Order` (state=pending), returns `token_url` |
| L7 | Browser redirect to `token_url` | `window.location.href = token_url` | — | Bank/PG hosted page | — |
| L8 | PG postback (form POST) → 303 redirect | `POST /pg/payments/postback` | `paymentId, status, failureCode` (form fields) | — | `tbl_Fintech_Payment_Order` updated |
| L9 | Callback page polls | `GET /mf/orders/purchases/{mfpId}/snapshot` | — | `GET /v2/mf_purchases/{id}` | Polls until `state='submitted'` |

Final terminal states for L9: `submitted` (success), `failed` (PG declined), `cancelled` (user aborted at bank page).

---

## 30.3 SIP Flow (with eNACH / UPI Autopay Mandate)

Web component: `mutual-fund-invest.component.ts → createAndAuthorizeMandate() + startSip()`.

### 30.3.1 Mandate Setup (one-time per bank account)

| # | Step | Endpoint | Body | Cybrilla | DB |
|---|---|---|---|---|---|
| M1 | List verified banks for picker | `GET /banking/my-verified-accounts` | — | — | `tbl_Fintech_Bank_Account WHERE VerificationStatus='completed'` |
| M2 | `MfPurchaseService.createMandate({...})` | `POST /pg/mandates` | `{ mandateType: "E_MANDATE"\|"UPI", bankAccountId (oldId), mandateLimit }` | `POST /v2/mandates` | `tbl_Fintech_Mandate` (status=CREATED) |
| M3 | `MfPurchaseService.authorizeMandate(mandateId, returnPath)` | `POST /pg/mandates/{id}/authorize?return={isin}` | — (backend resolves `payment_postback_url` from appsettings `Finprim:Postback:Web:Mandate`) | `POST /v2/mandates/{id}/authorize` → returns `token_url` | status=SUBMITTED |
| M4 | Open `token_url` in new tab | `window.open(tokenUrl, '_blank')` | — | Bank/Cybrilla hosted authorization (e.g. `cybrillarta.s.finprim.com/.../netbanking/billdesk?...`) | — |
| M5 | User clicks one of 3 demo buttons (Success / Payment success but e-Mandate failed / Failure) | Cybrilla form-POSTs back to us | `POST /pg/mandates/postback` (form) `{ paymentId, status, failureReason, mandateStatus? }` | — | Audited in `tbl_Fintech_Mf_Order_Log` |
| M6 | Backend 303-redirects to SPA | → `/app/mutual-funds/mandate-callback?status=success\|failure\|emandate_failure&paymentId=...&reason=...&return=invest/{isin}` | — | — | — |
| M7 | Mandate webhook later → APPROVED | `POST /webhooks/mandate.approved` | — | — | `tbl_Fintech_Mandate.status='APPROVED'` |

**Postback status mapping (NEW in this session):** the backend `MandatePostbackController` detects "payment success but e-mandate failed" by checking `rawStatus='success'` AND (`mandateStatus='failure'` OR `failureReason` contains "mandate") → maps to `status=emandate_failure` so the SPA shows the dedicated amber screen.

**App-side note:** if running in WebView, append `&source=app` to the postback config so the redirect lands on `/appmutualfund/mandate` (deep-link target) instead of the web SPA route.

### 30.3.2 Mandate state polling (optional)

```
GET /pg/mandates/{id}            ← single mandate live from Cybrilla + upserts our DB
GET /pg/mandates/my-list         ← our DB only, kept fresh by webhooks
GET /pg/mandates?bankAccountId={oldId}   ← from Cybrilla, filtered by bank
```

The invest page calls **the last one** with the selected bank's `oldId` so we only see mandates for that specific bank (not the tenant's whole list).

### 30.3.3 SIP Plan Create (after mandate is APPROVED)

| # | Step | Endpoint | Body | Cybrilla | DB |
|---|---|---|---|---|---|
| S1 | Send plan OTP | `POST /mf/orders/purchase-plans/send-otp` (with `Idempotency-Key` header) | `{ mobile, email, isdCode }` | `POST /v2/mf_purchase_plans/consent_otp` | `tbl_Fintech_Purchase_Otp` |
| S2 | OTP modal | — | — | — | — |
| S3 | Verify OTP (peek) | `POST /mf/orders/purchase-plans/verify-otp` | `{ otp }` | — | — |
| S4 | Create plan | `POST /mf/orders/purchase-plans` (with `Idempotency-Key` header) | `{ mfInvestmentAccount, scheme (ISIN), amount, installmentDay, frequency: "monthly", numberOfInstallments, paymentMethod: "mandate", paymentSource: <mandate.oldId>, systematic: true, gateway: "ondc", autoGenerateInstallments: true, initiatedBy: "investor", initiatedVia: "website"\|"app", userIp, consent: { email, isdCode, mobile, otp } }` | `POST /v2/mf_purchase_plans` | `tbl_Fintech_Mf_Purchase_Plan` (state=created) |
| S5 | PATCH state=confirmed | `PATCH /mf/orders/purchase-plans/{mfppId}` | `{ state: "confirmed" }` | `PATCH /v2/mf_purchase_plans/{id}` | state='confirmed' → eventually 'active' |

**Required pre-checks before S1 (mobile app must enforce or backend rejects):**
- `amount >= scheme.threshold.monthlySipMin` (e.g. 500.00 — Cybrilla returns 400 "minimum initial amount for scheme is 500.00" if violated)
- `numberOfInstallments >= scheme.threshold.installmentsMin` (typically 6)
- `paymentSource` MUST be a mandate's **numeric** `old_id` (not the `ena_xxx` string)
- Mandate must be in `APPROVED` state — backend will reject otherwise

After S5, FP auto-generates monthly installments. The app does nothing per-installment — webhooks `mf_purchase_plan.installment.created`, `mf_purchase_plan.installment.submitted`, `mf_purchase_plan.installment.completed` keep our DB fresh.

---

## 30.4 Endpoints Added / Changed This Session

| Endpoint | Method | Added | Purpose |
|---|---|---|---|
| `GET /banking/my-verified-accounts` | GET | NEW | Returns ALL bank accounts with `VerificationStatus='completed'` for the user — used by invest page bank picker |
| `GET /pg/mandates` | GET | EXTENDED | Now accepts `?bankAccountId={oldId}` — filters mandates to one bank (prevents tenant-wide leak) |
| `POST /pg/mandates/postback` | POST/GET | EXTENDED | Now detects "payment success but e-mandate failed" and emits `status=emandate_failure` to SPA |
| `Usp_Get_Fintech_Bank_Account_Status` | SP | V3 | ORDER BY now prefers `VerificationStatus='completed'` over newest |
| `Usp_List_Fintech_Bank_Accounts_Verified` | SP | NEW | Returns all completed banks ordered by confidence |

Run both SPs from `Claude-db-queries.txt` lines 3300–3484.

---

## 30.5 Validation Guards on the Web Client (mirror these in app)

These are enforced in `mutual-fund-invest.component.ts` and the app **must** enforce them client-side too — otherwise Cybrilla rejects with HTTP 400 and the user loses progress:

1. **Amount minimum** — `amount >= threshold.amount_min` for the picked mode (lumpsum vs monthly SIP). Re-check on submit, NOT just on input change.
2. **Amount multiples** — `amount % threshold.amount_multiples === 0` for lumpsum.
3. **Installments** — SIP: `numberOfInstallments >= threshold.installments_min` (usually 6).
4. **Bank verified** — only allow picking from `GET /banking/my-verified-accounts` (do NOT call Finprim's bank list — that one doesn't return verification status).
5. **Mandate APPROVED** — only enable SIP "Start" when at least one mandate has `status='APPROVED'`. Pending mandates show "being authorized by the bank — refresh in a minute."
6. **Mode switch resets amount** — when user toggles Lumpsum ↔ SIP, reset `amount` to the new mode's minimum.

---

## 30.6 Mandate Callback Page — 3 Outcomes

Browser route: `/app/mutual-funds/mandate-callback?status={success|failure|emandate_failure|pending}&paymentId=&reason=&return={isin}`.

| status | What happened | UI shown | Action |
|---|---|---|---|
| `success` | Bank approved e-Mandate | ✓ green "Auto-pay set up" | "Continue →" goes back to `/invest/{isin}` |
| `pending` | Bank accepted, processing async | spinner + 5s countdown | Auto-redirect back to invest page; mandate will become APPROVED via webhook |
| `emandate_failure` | Netbanking payment OK but e-Mandate registration failed | ⚠ amber warning + reason | "Retry e-Mandate Setup" → invest page |
| `failure` | Bank declined | ✗ rose "Authorization failed" + reason | "Try Again" → invest page |

The app must handle all four — `emandate_failure` is new and distinct from `failure`.

---

## 30.7 Headers, Auth, Idempotency — Quick Reference

- **Auth:** `Authorization: Bearer <userJwt>` on EVERY endpoint above (except `/pg/payments/postback` and `/pg/mandates/postback` which are unauth catch-back routes).
- **Idempotency:** every `POST /mf/orders/purchases`, `POST /mf/orders/purchase-plans`, and their `send-otp`/`verify-otp` siblings need `Idempotency-Key: <uuid>`. Reuse the **same uuid across the whole intent** (send-otp → verify-otp → create) so retries don't double-charge.
- **`isWeb`:** pass `isWeb=0` on bank account create + mandate authorize when calling from app (drives which `Finprim:Postback:{Web|App}:Mandate` URL is used). Defaults to `1`.
- **Source:** mandate authorize accepts `?source=app` query string so postback redirects to app deep-link instead of web SPA.

---

## 30.8 Error Shapes — Always the Same

Every error from `FinprimApi` follows:
```json
{
  "statusCode": 400 | 401 | 402 | 422 | 502,
  "message":    "Human-readable error (also surfaces Cybrilla's reason)",
  "result":     null
}
```

For Cybrilla validation failures (most common), `message` contains the full Cybrilla error array embedded, e.g.:
> `Finprim API error for [POST] /v2/mf_purchase_plans: 400 - {"error":{"errors":[{"field":"amount","message":"minimum initial amount for scheme is 500.00"}]}}`

Parse `message` for `"field":"<x>"` and `"message":"<y>"` to surface inline field errors in the app.

---

*Section 30 end — covers every API call across onboarding + lumpsum + SIP, including this session's changes (verified-bank picker, mandate-per-bank filtering, 3-way postback status, V3 status SP).*

---

# 31. Transaction Callback Pages — Re-Check (Lumpsum vs SIP)

> **Critical distinction:** Lumpsum redirects the user to a bank page → bank POSTs back → SPA `/payment-callback` polls. **SIP does NOT** — mandate authorization happens *once* via `/mandate-callback`; subsequent SIP plan creates stay on the invest page (no bank redirect per installment).

---

## 31.1 Lumpsum — Full Callback Sequence

### Outbound (browser leaves our app)

`mutual-fund-invest.component.ts → invest() → verifyOtpAndContinue()`

1. POST `/mf/orders/purchases` → `mfp_xxx`, `mfPurchaseId` (old numeric id)
2. POST `/mf/orders/purchases/{mfp}/send-otp`
3. PATCH `/mf/orders/purchases/{mfp}` with `{ consent: {...} }`
4. POST `/pg/payments/upi` or `/pg/payments/netbanking` → returns `{ token_url, paymentId }`
5. PATCH `/mf/orders/purchases/{mfp}` with `{ state: "confirmed" }`
6. `window.location.href = token_url` — **whole page navigates away** to the bank

### Bank → our backend (form POST)

Bank/PG (BillDesk, simulator) POSTs back to:
```
POST /pg/payments/postback
Body (form):
  paymentId={fp_payment_old_id}
  status=success|failure|pending
  failureCode={code}             ← optional
  failureReason={human text}     ← optional
Query (set on outbound token_url):
  mfp={mfPurchaseId}             ← we encode this when creating payment
  source=app                     ← if call was from WebView
```

Controller: `F:\github projects\FinprimApi\Controllers\PaymentPostbackController.cs`

Status is normalized lower-case and passed through **unchanged** (no synthesis like the mandate case).

### Backend → SPA (303 redirect)

Web target:
```
{Spa:BaseUrl}/app/mutual-funds/payment-callback
  ?mfp={mfPurchaseId}
  &id={paymentId}
  &status={success|failure|pending}     ← NOT used for outcome; advisory only
  &failureCode={code}
  &reason={msg}
```
App target (WebView, `source=app`):
```
{Spa:BaseUrl}/appmutualfund/payment?…same params…
```

Audit row written: `tbl_Fintech_Mf_Order_Log` step=`payment_postback`.

### SPA `/payment-callback` — what it actually does

Component: `payment-callback.component.ts`

- Required query: `mfp` (without it → instant `failed`)
- Optional query: `id` (paymentId) — banks sometimes omit on user-back; polling resolves from `mfp` alone
- **Polls every 2s for up to 90s** — calls TWO endpoints in parallel:
  - `GET /pg/payments/{id}/snapshot` → `{ status, failureCode, failureReason }`
  - `GET /mf/orders/purchases/{mfp}/snapshot` → `{ state, folioNumber, allottedUnits, purchasedPrice, failureCode }`
- Payment status is **the truth source** for success/failure. Order snapshot enriches the success card with folio & units.

### Outcome matrix (lumpsum)

| payment.status | order.state | UI outcome | Card content |
|---|---|---|---|
| `APPROVED` / `SUCCESS` / `INITIATED` | any | `success` (green ✓) | "Order placed successfully" + folio, units, NAV. Button: **View My Portfolio** → `/portfolio` |
| `FAILED` / `REJECTED` | any | `failed` (red ✗) | "Payment failed" + reason from `FAILURE_HINTS` map. Button: **Back to Funds** → `/list` |
| (no payment row yet) | `successful` | `success` | as above |
| (no payment row yet) | `failed` / `cancelled` / `reversed` | `failed` | as above |
| neither terminal after 90s | — | `timeout` (amber !) | "Still processing" + Go to Portfolio |

Live spinner sub-text shows the EXACT current state so user isn't staring at a generic message:
> `Order: Under review by ONDC • Payment: pending (check 7)`

### Failure code → user message map (in callback component)

Source: `PaymentCallbackComponent.FAILURE_HINTS` (lines 133–150 of `payment-callback.component.ts`).

| Code | Message shown |
|---|---|
| `fp_payment_expired` | You took too long. Please try again. |
| `pg_payment_expired` | Bank session expired. Please retry. |
| `tpv_failed` | Wrong bank account used. Use verified primary. |
| `pg_timeout` | Gateway timed out. Retry shortly. |
| `invalid_upi_id` | UPI ID invalid. Re-enter. |
| `bank_not_enabled` | Bank not supported. Try different bank. |
| `investor_bank_declined` | Bank declined (insufficient balance / daily limit). |
| `user_cancelled` | You cancelled. Try whenever ready. |
| `bank_downtime` | Bank/UPI unavailable. Retry later. |
| `insufficient_funds` | Insufficient funds. Top up & retry. |
| `upi_app_unavailable` | UPI app didn't respond. Try a different one. |
| `pg_server_error` | Gateway error. Retry shortly. |
| `invalid_credentials` | Wrong bank credentials. Retry. |
| `payment_failed` | Generic bank failure. Retry. |
| `invalid_bank_account` | Account closed/blocked. Use a different one. |
| `fp_payment_url_unused` | Payment window unused. Initiate new payment. |

App must mirror this map (or call the snapshot endpoint and read `failureMessage` text the backend resolves).

---

## 31.2 SIP — Two Separate Callback Phases

SIP is **NOT** one redirect. There are two distinct phases, each with its own callback (or no callback).

### Phase A — Mandate Authorization (ONE-TIME per bank account)

This is the only phase with a redirect. After it succeeds, the user has a permanent eNACH/UPI Autopay standing instruction with the bank.

1. POST `/pg/mandates` → `{ bank_account_id (oldId), mandate_type, mandate_limit }`
2. POST `/pg/mandates/{id}/authorize` → returns `token_url`
3. `window.open(token_url, '_blank')` — opens NEW TAB (web; deep-link on app)
4. User sees Cybrilla demo / real BillDesk page with **3 buttons**:
   - **Success** → both payment & mandate registered
   - **Payment success but e-Mandate failed** → money debited (sandbox: no real debit), mandate not registered
   - **Failure** → both failed
5. Bank/sim POSTs to our backend:
```
POST /pg/mandates/postback
Body (form, exact field names from Cybrilla):
  paymentId={id}
  status=success|failure
  failureReason={text}
  mandateStatus=success|failure        ← extra field; key to the 3-way distinction
```
6. Backend **synthesizes** outgoing status:
```
if rawStatus == "success" AND (mandateStatus == "failure" OR reason contains "mandate")
    → outgoing status = "emandate_failure"
else
    → outgoing status = rawStatus  (success | failure)
```
7. 303 redirect to SPA:
```
Web: {Spa:BaseUrl}/app/mutual-funds/mandate-callback
       ?status={success|failure|emandate_failure|pending}
       &paymentId={id}
       &reason={text}
       &return={isin}                  ← so callback can navigate back to /invest/{isin}
App: {Spa:BaseUrl}/appmutualfund/mandate?…same params…
```

Audit row: `tbl_Fintech_Mf_Order_Log` step=`mandate_postback` (with both `rawStatus` and `mandateStatus` in the log message).

### SPA `/mandate-callback` — what it actually does

Component: `mandate-callback.component.ts`

- **No polling** — pure render based on query string
- If `status=pending` → 5-second countdown then auto-redirect to `return` path
- Otherwise renders one of 4 cards:

| status | UI | Primary CTA | Secondary CTA |
|---|---|---|---|
| `success` | ✓ green "Auto-pay set up" | Continue → `/app/mutual-funds/${return}` | — |
| `pending` | spinner + "Bank processing" | Auto-redirects in 5s | — |
| `emandate_failure` | ⚠ amber "Payment processed — e-Mandate failed" | Retry e-Mandate Setup → `${return}` | Back to Funds → `/list` |
| `failure` | ✗ rose "Authorization failed" + reason | Try Again → `${return}` | Back to Funds → `/list` |

**Web tab handling:** The mandate authorize page is opened in a `window.open(..., '_blank')` tab. While the user is in the new tab, the original invest tab is polling `GET /pg/mandates/{id}` every 3s for 60s as a backup signal (sees mandate flip to APPROVED via webhook). Both paths converge — whichever fires first updates the state. App / WebView: same logic but native deep-link target `/appmutualfund/mandate`.

### Phase B — SIP Plan Create (every time the user starts a new SIP)

**NO redirect, NO callback page.** Everything happens on the invest page:

1. POST `/mf/orders/purchase-plans/send-otp`
2. OTP modal shown — user types
3. POST `/mf/orders/purchase-plans/verify-otp` (peek only)
4. POST `/mf/orders/purchase-plans` with `{ payment_method: "mandate", payment_source: {mandate.oldId}, consent: {...} }`
5. PATCH `/mf/orders/purchase-plans/{mfpp}` with `{ state: "confirmed" }`
6. Success → show in-page success modal with **View My SIPs** (`/sips`) or **Continue Browsing** (`/list`). Failure → error banner, button re-enabled.

After this, FP auto-generates monthly installments. Each installment fires:
- `mf_purchase_plan.installment.created` webhook → row in `tbl_Fintech_Mf_Purchase_Plan_Installment` (state=created)
- bank debits via the standing mandate (no user action)
- `mf_purchase_plan.installment.submitted` → state=submitted
- `mf_purchase_plan.installment.completed` → state=completed + folio/units written

The mobile app reads from `GET /mf/orders/purchase-plans/{mfpp}/installments` to render the schedule. **It never sees a per-installment callback page.**

---

## 31.4 APP WEBVIEW — how to NOT get stuck (READ THIS) — 2026-07-06

> The single biggest question from the app side: *"we open a WebView URL — do we
> poll something for the webhook response?"* **Answer: you do NOT wait on the webhook
> directly. You (a) open the WebView, (b) close it when the WebView is redirected to
> our deep-link page, then (c) POLL our own status endpoint until it resolves.** The
> backend has already written the result to the DB from the bank's form-POST (postback),
> and the Cybrilla webhook is only an authoritative backstop. So polling our DB endpoint
> is always safe and never hangs.

### The golden rule
**Never parse the bank's response yourself. Never wait for a Cybrilla webhook in the app.**
The bank form-POSTs to OUR backend → backend writes status to DB → backend 303-redirects
the WebView to a deep-link page (`/appmutualfund/mandate` or `/appmutualfund/payment`,
which fire `islamicly://mf-callback`). The app's only jobs: **detect that deep-link → close
the WebView → poll our status API.**

### A) SIP — mandate authorization in a WebView

1. Build the pending SIP FIRST (so it's resumable): `POST /api/mf/orders/sip/register-pending`
   `{ intentKey, mandateId, mfInvestmentAccount, scheme, amount, frequency, installmentDay, numberOfInstallments }`.
   Generate ONE `intentKey` (uuid) and keep it for the whole flow.
2. `POST /api/pg/mandates/{id}/authorize?return=invest/<isin>&intent=<intentKey>&source=app`
   → `{ token_url }`.
3. Open `token_url` in a WebView.
4. **Watch the WebView's URL.** When it navigates to a URL that starts with
   `islamicly://mf-callback` (or reaches `/appmutualfund/mandate?...`), the bank step is
   done — **close the WebView.** (The redirect carries `status`, `intent`, `reason` in
   the query — you MAY read them for a fast hint, but they are NOT the source of truth.)
5. **Poll `GET /api/mf/orders/sip/status/{intentKey}`** every ~2s (cap ~60–90s):
   - `localStatus == "mandate_approved"` → mandate done → go to step 6.
   - `localStatus == "mandate_failed"` → show `reason`; offer "Try again" (re-authorize).
   - `localStatus == "mandate_pending"` (still) → keep polling; after the cap, show
     "Auto-pay is still processing — you can resume from SIP Plans later" (NOT an error —
     the pending row survives; §21.2b resume finishes it).
6. **Finalize the SIP:** `POST /api/mf/orders/sip/finalize/{intentKey}` `{ otp: "" }`
   → `200 { localStatus:"active", mfPurchasePlanId }` → success screen.
   Handle finalize errors exactly per §21.2b (`otp_missing` → re-send+verify a fresh OTP;
   `nomination_required` → nomination screen; `409` → not approved yet, keep polling; `502`
   → show the parsed `reason`).

**If the user kills the app / WebView mid-flow:** nothing is lost. The pending row exists;
on next open, `GET /api/mf/orders/purchase-plans/my-list` shows it with `localStatus`
`mandate_pending`/`mandate_approved` → render a **"Complete SIP"** action that runs the
same step-5/6 handler (§21.2b). This is the whole point of the DB-driven design.

### B) Lumpsum — payment in a WebView

1. `POST /api/mf/orders/purchases` → `{ mfp_xxx, mfPurchaseId }`; then send-otp / PATCH
   consent / `POST /api/pg/payments/upi|netbanking` → `{ token_url, paymentId }`;
   PATCH state=confirmed (see §31.1 for the exact call order).
2. Open `token_url` in a WebView.
3. **Watch the WebView URL** for `islamicly://mf-callback?flow=payment` (or
   `/appmutualfund/payment?...`) → **close the WebView.** Query carries `mfp`, `id`,
   `status`, `failureCode`, `reason` (advisory only).
4. **Poll BOTH every ~2s (cap ~90s):**
   - `GET /api/pg/payments/{paymentId}/snapshot` → `{ status, failureCode, failureReason }`
     — **payment status is the source of truth.**
   - `GET /api/mf/orders/purchases/{mfp}/snapshot` → `{ state, folioNumber, allottedUnits,
     purchasedPrice, failureCode }` — enriches the success card.
   Outcome matrix + failure-code→message map: see §31.1.
5. Terminal `APPROVED/SUCCESS` → success; `FAILED/REJECTED` → failure with reason; neither
   after the cap → "Still processing" (not a hard error) → send them to Transactions.

### Endpoints the app needs (summary)
| Purpose | Endpoint |
|---|---|
| Register pending SIP (first) | `POST /api/mf/orders/sip/register-pending` |
| Authorize mandate (get WebView URL) | `POST /api/pg/mandates/{id}/authorize?return=…&intent=…&source=app` |
| **Poll SIP status** | `GET /api/mf/orders/sip/status/{intentKey}` |
| Finalize SIP | `POST /api/mf/orders/sip/finalize/{intentKey}` |
| Resume/list SIPs | `GET /api/mf/orders/purchase-plans/my-list` |
| Start payment (get WebView URL) | `POST /api/pg/payments/upi` or `/netbanking` |
| **Poll payment** | `GET /api/pg/payments/{paymentId}/snapshot` |
| **Poll order** | `GET /api/mf/orders/purchases/{mfp}/snapshot` |
| (Optional backup signal) mandate live | `GET /api/pg/mandates/{id}` |

### Do / Don't
- ✅ Poll OUR status endpoint after the WebView returns. It's already resolved from the
  postback; the webhook is just a backstop.
- ✅ Treat a still-pending state after the timeout as *resumable*, not failed.
- ✅ Keep ONE `intentKey` across register-pending → authorize → status → finalize.
- ❌ Don't wait on / subscribe to Cybrilla webhooks from the app.
- ❌ Don't parse the bank's POST body — you never receive it; the backend does.
- ❌ Don't send `user_ip` yourself for SIP finalize — the backend resolves a valid IPv4
  (it rejects `::1`).
- ❌ Don't block the UI forever if the WebView never returns — cap the poll and offer
  Resume (SIP) / Go-to-Transactions (lumpsum).

> **Note — SIP create is now DB-driven (§21.2).** The older §31.2 "Phase B" (one-shot
> `POST /mf/orders/purchase-plans` on the invest page) is superseded for SIP: use
> register-pending → authorize → poll `sip/status` → `sip/finalize`. Lumpsum (§31.1)
> is unchanged.

---

## 31.3 PAN Test Numbers (sandbox)

| PAN | Behavior |
|---|---|
| `ANGPI5005C` | Fresh-KYC flow — KRA returns `unavailable` → flows into KYC Form path |
| `AAAPA3751A` | Test data: name match success (Tony Soprano fixture) |
| `SKLPA9239S` | Test data: name match success (Rani Gupta fixture) |
| `DWEPS2837G` | Investor profile creation OK |
| `ACCPP55K7L` | MF account creation OK |
| `ABCDE1234F` | Generic placeholder — also fresh-KYC path |

**General sandbox behavior:**
- Any well-formed PAN → readiness `action="create"` immediately
- Completed KYC decays back to `submitted` after ~5 min (KRA simulator reset)
- Force-pin during tests: `POST /kyc/requests/{id}/simulate` with `{ "status": "successful" }`
- DigiLocker test: Aadhaar `999999999999`, OTP `123456`
- eSign test: same Aadhaar + OTP

**KYC-status to expect by PAN suffix in Cybrilla sandbox (`cybrillapoa` tenant):**
- Most well-formed PANs → `action=create`, no KRA record → app routes to **KYC Form** path
- After form submission + eSign → 30s–5min lag before `IsKycCompliant=1` flips
- Reset between runs: backend has no auto-reset; either change PAN or wait for KRA decay

---

## 31.4 Bank Account Test Numbers (sandbox)

| Account number suffix (last 4) | Pre-verification result | Confidence | UI shown |
|---|---|---|---|
| `1191`–`1199` (i.e. `11XX` where XX = 91–99) | `completed` (instant, ~2s) | `very_high` | Green "Verified" badge |
| `3101`–`3199` (`31XX`) | `failed` | `zero` | Red "Verification failed" |
| `9901`–`9999` (`99XX`) | `failed` reason=`expiry` | `zero` | Red + retry button |
| Anything else | `pending` ~10s → random verdict | varies | Amber "Verifying…" → settles |

**Test bank example (instant success):**
```
Account: 987654321193    (ends in 1193 — matches 11XX pattern)
IFSC:    HDFC0001234
Holder:  Mohd Irfan
Bank:    HDFC BANK
```

**Test bank example (instant failure):**
```
Account: 987654323101    (ends in 3101 — matches 31XX pattern)
IFSC:    ICIC0000001
Bank:    ICICI BANK
```

IFSC codes are NOT validated against the suffix — any well-formed IFSC works. Common sandbox values: `HDFC0001234`, `HDFC0000001`, `ICIC0000001`.

**Important for app integration:**
- Always call `GET /banking/my-verified-accounts` (NOT Finprim's list) to populate the SIP bank picker. Cybrilla's bank list endpoint does NOT carry `VerificationStatus`.
- A bank can be added (status `pending`) but should not be selectable for SIP/lumpsum until `VerificationStatus='completed'`.
- The `OldId` (numeric) — NOT the `bac_xxx` string — is what gets sent to Cybrilla as `bank_account_id` in mandate create and as the mandate filter query param.

---

## 31.5 Quick Reference — Test Each Scenario

| Scenario | PAN | Bank account | Expected end state |
|---|---|---|---|
| KYC already done, full lumpsum success | `AAAPA3751A` | `987654321193` (HDFC0001234) | Order → submitted; `/payment-callback` → success card |
| KYC not done, fresh-KYC then invest | `ANGPI5005C` | `987654321193` | `/kyc-form` flow → eSign → submitted → onboarding → invest |
| Bank verification failed | any | `987654323101` (any IFSC) | `VerificationStatus=failed`, bank not offered in SIP picker |
| Lumpsum payment failed | `AAAPA3751A` | `987654321193`, choose **Failure** at sim | `/payment-callback` → red ✗ + reason |
| SIP mandate Success | `AAAPA3751A` | `987654321193`, choose **Success** at sim | `/mandate-callback?status=success` → ✓ + Continue → invest page |
| SIP payment OK but mandate failed | `AAAPA3751A` | `987654321193`, choose **Payment success but e-Mandate failed** | `/mandate-callback?status=emandate_failure` → ⚠ + Retry |
| SIP authorization failed | `AAAPA3751A` | `987654321193`, choose **Failure** | `/mandate-callback?status=failure` → ✗ + Try Again |
| SIP plan create with min violation | `AAAPA3751A` | already approved mandate | 400 from FP: "minimum initial amount for scheme is 500.00" — caught by frontend guard before submit |

---

## 31.6 Reference URLs

- Fintech Primitives API docs: https://fintechprimitives.com/docs/api/
- Cybrilla POA docs: https://poa.cybrilla.com/docs/
  - Pre-verifications: https://poa.cybrilla.com/docs/additional-apis/pre-verifications
  - KYC Forms: https://poa.cybrilla.com/docs/additional-apis/kyc-forms
- Sandbox simulator: https://simulator.fintechprimitives.com/
- Tenants used: `islamiclyondc` (Finprim), `cybrillarta` (Cybrilla POA)

ONDC sandbox payment simulation rule (for non-redirect tests):
- Lumpsum amount ending in **0** → auto-success at RTA
- Amount ending in **1** → auto-failure
- `mf_purchase` simulate is NOT available on ONDC — must go through real PG redirect

KYC compliance flag propagation lag (Finprim → Cybrilla `cybrillapoa`): **30s to 5min**. After completing a KYC form, the app should poll `/onboarding/status` for up to 5 minutes before assuming a problem.

*Section 31 end — covers transaction callback pages (lumpsum vs SIP), full status mapping for both, and sandbox test data.*

---

# 32. MF Details Page — Data, API & Calculation Reference (LATEST, for the App Team)

**This is the latest, authoritative spec for the fund Details page** (`/app/mutual-funds/details/{ISIN}`) — what data it shows, which APIs feed it, and the exact client-side math.

> **Invest-Now page (`/app/mutual-funds/invest/{ISIN}`) is NOT re-documented here** — its canonical, latest contract already lives in **§3.18 (transaction flows)**, **§14.2 (fresh-KYC + invest gate)**, **§30.2/§30.3 (lumpsum + SIP end-to-end)**, **§30.5 (client-side validation guards the app MUST enforce)**, and **§31 (callback pages)**. Use those for invest. The only details-page-relevant invest facts (eligibility gate, thresholds-driven min/max) are cross-referenced at the end of this section.

Source: `mutual-fund-details.component.ts` (2448 lines), service `mf-funds.service.ts`. Base URL = `environment.mfurl`; JWT Bearer on all; `{ISIN}` = route param (e.g. `INF209K01RU9`).

## 32.1 API calls (all GET, all keyed by ISIN)

> `fund-schemes-v2` is already in the master-data table (§3.17). The `masterdata/nav/*` and `scheme-overview/*` endpoints below were previously **undocumented** — this is their reference.

| Endpoint | Service method | Response shape | Fires when |
|---|---|---|---|
| `GET /masterdata/fund-schemes-v2/{gateway}/{isin}` (gateway=`ONDC`) | `getFundByIsinDirect()` | `MfSchemePlan` (incl. `thresholds[]`) | ngOnInit — scheme + min-investment thresholds |
| `GET /masterdata/nav/{isin}/series?period={1M\|3M\|6M\|1Y\|3Y\|5Y}` | `getNavSeries()` | `{ points: [{date, nav}], returnPct }` (oldest-first) | ngOnInit + on chart-period change |
| `GET /masterdata/scheme-overview/{isin}/trailing-returns` | `getTrailingReturns()` | `TrailingReturnRow[]` | ngOnInit |
| `GET /masterdata/scheme-overview/{isin}` | `getSchemeOverview()` | `SchemeOverview \| null` | ngOnInit — AUM, expense, CAGR, market-cap, exit load, age |
| `GET /masterdata/scheme-overview/{isin}/holdings` | `getHoldings()` | `HoldingRow[]` (company + %) | ngOnInit |
| `GET /masterdata/scheme-overview/{isin}/sectors` | `getSectors()` | `SectorRow[]` (sector + %) | ngOnInit |
| `GET /masterdata/scheme-overview/{isin}/documents` | `getDocuments()` | `SchemeDocument[]` | ngOnInit |
| `GET /masterdata/scheme-overview/{isin}/riskometer` | `getRiskometer()` | `Riskometer` (bullets + 2 image URLs) | ngOnInit |
| `GET /masterdata/scheme-overview/{isin}/summary` | `getSummary()` | `SchemeSummary` (4 HTML blocks) | ngOnInit |
| `GET /masterdata/scheme-overview/{isin}/faqs?activeOnly=true` | `getFaqs()` | `FaqRow[]` | ngOnInit |

**Flow:** route ISIN → `getFundByIsin()` (scheme + thresholds) → 9 scheme-overview endpoints in parallel → signals update the template.

## 32.2 Data fields displayed (by section, API source)

**Hero/header:** Fund name `plan.mf_scheme.name`; AMC `plan.mf_fund.name`; ISIN `plan.isin`; risk badge `fund.risk`; **Current NAV** = latest NAV point (computed); **NAV change %** = point-to-point (computed); 1Y/3Y/5Y returns `returnByPeriod[period]`; Shariah/SEBI/Bank-grade/T+1 = **HARDCODED labels**.

**Overview:** `SchemeSummary.{summaryHtml, investmentPhilosophyHtml, portfolioPositioningHtml, exitLoadHtml}`; Min SIP/Lumpsum from thresholds.

**NAV chart:** `NavSeriesResponse.points[]`; period return `returnPct`; x-axis = first/mid/last dates.

**Portfolio:** Trailing returns `TrailingReturnRow.{periodLabel, annualizedPct, benchmarkPct, asOn, investedAmount}`; Riskometer `Riskometer.{points[].pointText, schemeImageUrl, benchmarkImageUrl}`; Sector donut `SectorRow.{sectorName, pct, asOn}`; Market-cap bar `SchemeOverview.{marketCapLargePct, marketCapMidPct, marketCapSmallPct, marketCapAsOn}`; Holdings `HoldingRow.{companyName, pct, asOn}`.

**Info:** Key metrics `SchemeOverview.{fundSizeAUM(+Unit), expenseRatio(+Unit), exitLoad(+Unit+Condition), fundAgeYears|fundInceptionDate}`; CAGR cards `SchemeOverview.{cagr1Y, cagr3Y, cagr5Y, cagrSinceInception}` (+ `*AsOnDate`); Documents `SchemeDocument.{documentName, originalName, downloadUrl, url, byteSize}`; FAQ `FaqRow.{question, answer}`.

## 32.3 ⚠️ How NAV / change% / returns are derived — do it EXACTLY like the web frontend

The backend does **NOT** return ready fields like `currentNav`, `dayChange`, or `oneYearReturn`. `GET /masterdata/nav/{isin}/series?period={period}` returns a **daily price array**; the frontend computes everything from it. The app must do the **same computation** — do not look for fields that don't exist.

**What the API actually returns:**
```json
{
  "points": [
    { "date": "2025-06-30", "nav": 42.10 },   // oldest (start of the period)
    { "date": "2025-07-01", "nav": 42.35 },
    "...one point per day...",
    { "date": "2026-06-29", "nav": 49.80 }     // newest (latest NAV)
  ],
  "returnPct": 18.29                            // period return (present for most periods)
}
```
`points` is **oldest-first**.

**Verbatim Angular computeds (`mutual-fund-details.component.ts`):**
```ts
// CURRENT NAV (the big ₹ number) = LAST point in the series. NOT a separate field.   [:1300]
navTick = computed(() => { const pts = this.bestNavSeries();
  return pts.length ? pts[pts.length - 1].nav : 0; });                 // → 49.80

// TODAY'S CHANGE % (the +x% pill) = last point vs the one before it.                  [:1304]
navTickPct = computed(() => { const pts = this.bestNavSeries();
  if (pts.length < 2) return 0;
  const last = pts[pts.length - 1].nav, prev = pts[pts.length - 2].nav;
  return prev ? ((last - prev) / prev) * 100 : 0; });                  // (49.80-49.65)/49.65*100

// PERIOD RETURN % (chart, for the selected 1M/3M/6M/1Y/3Y/5Y) = first vs last.        [:1419]
navPeriodReturn = computed(() => { const pts = this.currentNav();
  if (pts.length < 2) return 0;
  const first = pts[0].nav, last = pts[pts.length - 1].nav;
  return first ? ((last - first) / first) * 100 : 0; });              // (49.80-42.10)/42.10*100 = 18.29%
```
> The API also sends `returnPct` for the period and the frontend uses it when present (`returnByPeriod[period] = res.returnPct`), falling back to `navPeriodReturn` otherwise. But **Current NAV and Today's-change% have NO field — only the `points[]` array gives them.**

**App-team rule:**

| Number on screen | How the app must get it |
|---|---|
| Current NAV (₹49.80) | `points[points.length - 1].nav` |
| Today's change (+0.30%) | `(points[last].nav − points[last-1].nav) / points[last-1].nav × 100` |
| 1Y / period return (+18.29%) | use API `returnPct` if present, else `(points[last].nav − points[0].nav) / points[0].nav × 100` |
| NAV line chart | plot every element of `points[]` (oldest→newest) |

## 32.4 Other calculation formulas (client-side, replicate)

### 32.4.0 Calculator Formula — summary for the app team (implement exactly as below)

The fund Details page calculator runs entirely on the **NAV history**, which comes from `GET /masterdata/nav/{isin}/series?period={1M|3M|6M|1Y|3Y|5Y}`. This returns `{ points: [{date, nav}], returnPct }` where `points` is an **ascending daily NAV list** (oldest first, newest last). There is **no separate "current NAV" or "returns" field** — you compute them from this array. Use `Start NAV = points[0].nav` (first/oldest point of the selected period), `End NAV / Current NAV = points[last].nav` (newest point = today's NAV).

**Lumpsum calculator** — simulate one bulk investment at the start of the period, valued today:
```
Final Value = Investment × (End NAV / Start NAV)
Return %    = (Final Value / Investment − 1) × 100
Profit      = Final Value − Investment
```

**SIP calculator** — simulate N equal monthly buys, each at that month's NAV (rupee-cost averaging). For the monthly NAV history, group the daily `points` by calendar month (`yyyy-MM`), keep the **last NAV of each month**, and take the **most recent N months** where **N = 12 for 1Y, 36 for 3Y, 60 for 5Y**:
```
Units bought each month = SIP Amount / (that month's NAV)
Total Units             = Sum of all monthly Units
Total Invested          = SIP Amount × number of months
Final Value             = Total Units × Current NAV   (End NAV)
Return %                = (Final Value / Total Invested − 1) × 100
Profit                  = Final Value − Total Invested
```

Both calculators are **historical backtests using real NAV** (they show what would actually have happened) — not future projections. Round Final Value/Invested/Profit to whole rupees. Replicate these formulas exactly so the app's numbers match the web to the rupee. (The right-rail "you invest → you get" widget is a separate **forward projection** using the fund's 3Y CAGR — see §32.4.1 #6.)

### 32.4.1 Full plain-English walkthrough of every calculation (read this)

**The raw material — the NAV series.** Every number traces back to `GET /masterdata/nav/{isin}/series?period={period}` → `{ points:[{date,nav},…], returnPct }`. `points` is an **ascending daily price list** (oldest first, newest last). The backend returns a price *history*; it does NOT return a pre-computed "current NAV" or "today's change" — those are derived from this array.

**1. Current NAV (the big ₹ number).** `currentNav = points[last].nav`. Just the **last element** (newest date). No math — a lookup. Empty array → 0.

**2. Today's change % (the green/red pill).** `change% = (points[last].nav − points[last-1].nav) / points[last-1].nav × 100`. Take the **last two** prices, subtract, divide by the older, ×100. Positive→green up, negative→red down. Needs ≥2 points (else 0). NOT an API field.

**3. Period return % (chart, for 1M/3M/6M/1Y/3Y/5Y).** **Primary:** use the API's `returnPct` for that period when present (`returnByPeriod[period] = res.returnPct`). **Fallback (API omits it):** `return% = (points[last].nav − points[0].nav) / points[0].nav × 100` — **first vs last** price of that period. "1Y +18.29%" = NAV grew 18.29% from the oldest point a year ago to today.

**4. Lumpsum backtest (calculator — "what if I'd invested ₹X once").** Simulates one bulk buy at the **start** of the period, valued today. `startNav = points[0].nav`, `endNav = points[last].nav`. **Maturity** `= amount × (endNav / startNav)` (your money grows in the same ratio the NAV grew — buy `amount/startNav` units at the start, worth `units × endNav` today). **Return%** `= (maturity/invested − 1) × 100`. **Profit** `= maturity − invested`.

**5. SIP backtest (calculator — "what if I'd invested ₹X every month").** Simulates **N equal monthly buys** ending today, each at that month's NAV.
- **N (installments):** 1Y→12, 3Y→36, 5Y→60.
- **Month-end NAV selection:** group all daily points by `yyyy-MM`, keep the **chronologically last point of each month**, then take the **most recent N months** (each month = one buy price).
- **Units:** for each month `m`: `totalUnits += amount / m.nav` (buy `amount` rupees of units at that month's NAV → fewer units when NAV is high, more when low = rupee-cost averaging).
- **Invested** `= amount × months.length`. **Maturity** `= totalUnits × endNav` (all accumulated units valued at today's NAV). **Return%** `= (maturity/invested − 1) × 100`. **Profit** `= maturity − invested`.
> Both backtests use **real historical NAV** — they show what would actually have happened, NOT a projection.

**6. Quick projection (right-rail "you invest → you get", FORWARD-looking).** A **future projection** using the fund's 3-year CAGR as the assumed annual rate. `rate = fund.returns.y3 / 100`, `r = rate/12`, `n = years×12`. **SIP** (future value of an annuity-due): `maturity = amount × ((pow(1+r,n) − 1) / r) × (1+r)`, invested `= amount × n`. **Lumpsum** (compound): `maturity = amount × pow(1+rate, years)`, invested `= amount`. Profit `= maturity − invested`.

**7. Confidence band (low/expected/high).** The **same projection formula run 3×** at three assumed annual rates: **Low 8%**, **Expected = fund 3Y CAGR**, **High 22%**. The expected marker sits on the bar at `expectedPct = (expected − low) / (high − low) × 100` (clamped 2–98%). Illustration of an optimistic/pessimistic spread.

**8. Chart & visual geometry.**
- **NAV line chart:** `min`/`max` of all NAVs; each point → `x = (i/(n-1))×width`, `y = bottom − ((nav − min)/(max − min))×(bottom − top)` (y inverted so higher NAV is higher on screen).
- **Sector donut:** circle radius 56 → `C = 2π·56 ≈ 351.86`; each sector arc length `= (pct/100)×C`, cumulative `offset` so slices sit end-to-end.
- **Holdings bars:** `barWidth% = min(100, (pct / maxHoldingPct)×100)` — top holding fills 100%, others scale down.
- **Risk needle:** risk → arc fraction (Low 0.16, Moderate 0.50, High/Very-High 0.85) → needle x/y.

**Takeaway:** Returns/NAV are **derived from `points[]`** (except the optional period `returnPct`). **Backtests use real NAV** (#4–5); **projections use 3Y CAGR** (#6–7) — don't confuse them. Replicate these exactly so app numbers match the web to the rupee.

### 32.4.2 Formula quick-reference

- **Lumpsum backtest (calculator):** `maturity = amount * (endNav / startNav)`; `returnPct = ((maturity/invested) - 1) * 100`.
- **SIP backtest:** for each month `m` (month-end NAV): `totalUnits += amount / m.nav`; `invested = amount * months`; `maturity = totalUnits * endNav`. Month-end NAV = last point per `yyyy-MM`, trimmed to last N months (12/36/60).
- **Quick projection (FV of annuity):** `r = (fund.returns.y3/100)/12`, `n = years*12`; SIP `maturity = amount * ((pow(1+r,n)-1)/r) * (1+r)`; Lumpsum `maturity = amount * pow(1+rate, years)`.
- **Confidence band:** same formula at Low=8%, Expected=fund 3Y CAGR, High=22%.
- **Sector donut arc (SVG):** `C = 2π·56 ≈ 351.86`; per-sector `dash = (pct/100)*C`, cumulative `offset`.
- **Holding bar width:** `min(100, (pct / maxHoldingPct) * 100)` (largest = 100%).
- **Risk needle ratio:** Low→0.16, Moderate→0.50, High/VeryHigh→0.85.

## 32.5 Charts
Hero sparkline · NAV line+area chart · sector donut · market-cap stacked bar · holdings horizontal bars · calculator growth sparkline · quick-projection sparkline.

## 32.6 Hardcoded / mock on details page (DO NOT treat as API)
- `fund` object defaults (Tata Ethical Fund, nav 48.72, etc.) — overwritten by real `plan` after load.
- Trust signals (SEBI/Bank-grade/T+1), Compare-tab peer funds, Quick-stats sidebar — hardcoded.
- Synthetic chart paths — fallback only when NAV series has < 2 points.
- Fallback return map `{1M:4.2,3M:9.8,6M:13.5,1Y:18.4,3Y:14.2,5Y:12.8}` — only if API gives no `returnPct`.
- Not rendered (dead/legacy): sentiment, screening checks, asset-allocation, geography, static top-holdings, manager track, AMC details, social-proof ticker.

## 32.7 Invest cross-reference (no duplication)
From the Details page the user taps **Invest** → `/app/mutual-funds/invest/{ISIN}`. For that page's full latest contract see:
- **§30.2** Lumpsum end-to-end · **§30.3 / §27.7** SIP + mandate end-to-end · **§3.18** transaction flows
- **§30.5** client-side validation guards the app MUST enforce (amount min/max/multiples come from `thresholds[]` — never hardcode)
- **§14.2 / §29.9.1** the fail-closed invest gate (`readiness === 'verified'` + profile + bank-verified + MFIA)
- **§31** payment/mandate callback pages

*Section 32 end — MF Details page binding/API/formula reference (latest). Invest page = §3.18 + §14.2 + §30 + §31.*

---

# 33. Q&A — App Team Doubts (write here)

> **App team: post your questions in this section instead of pinging over chat.**
> This file is a shared git repo — add your doubt under "OPEN QUESTIONS", commit, and
> push. We (API side) will pull, answer inline, move it to "ANSWERED", commit, and push
> back. That way every question + answer stays in one versioned place and nobody works
> off a stale paragraph.
>
> **How to use (both sides):**
> 1. `git pull` before editing (so you have the latest).
> 2. Add/answer your entry.
> 3. `git add API_Integration_Guide.md && git commit -m "qa: <short note>" && git push`.
> 4. If git reports a conflict, it's almost always in THIS section — keep both entries
>    (they're independent lines) and push again.
>
> **Format for a new question** — copy this block, fill it in, put it under OPEN QUESTIONS:
> ```
> ### Q<n> — <one-line title>
> - **Asked by:** <name> · <date>
> - **Context:** <screen / endpoint / section ref, e.g. §21.2 SIP finalize>
> - **Question:** <what you need to know>
> - **What you tried / error:** <exact request + response/error, if any>
> ```

## OPEN QUESTIONS

<!-- App team: add new questions below this line. Newest at the top. -->

### Q1 — (example — replace me)
- **Asked by:** _App team_ · _YYYY-MM-DD_
- **Context:** _§21.2 SIP finalize_
- **Question:** _e.g. "After the mandate WebView returns, how long should we poll `sip/status` before showing 'still processing'?"_
- **What you tried / error:** _paste the exact request + the response/error here_

## ANSWERED

<!-- API side moves resolved items here with the answer appended. -->

_(none yet)_

---

*End of guide.*

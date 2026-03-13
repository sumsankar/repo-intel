# 🔍 Repository Intelligence Report

**Repository:** https://github.com/SUMIT-Consultancy-Services/sarwamart_app_api.git
**Analyzed:** 2026-03-12
**Skills run:** code, architecture, security, devops

---

## Executive Summary

`sarwamart_app_api` is a large-scale ASP.NET MVC/Web API 2 application (targeting .NET Framework 4.7.2) serving as the backend for the Sarwamart marketplace platform, covering stock management, buyer/seller registration, invoicing, payments, notifications, and user management across 30+ functional areas. The codebase is a full multi-project solution (~82,000 lines of C#) with reasonable layered architecture, but is critically compromised on security: a live Firebase Admin SDK private key, a production database connection string with credentials, and a Gmail SMTP password are all committed in plaintext to the repository. Real generated PDF invoices and user-uploaded product images are also stored in the repo. Revocation and rotation of all exposed credentials is the single most urgent action required.

---

## Health Scorecard

| Dimension | Score | Status |
|-----------|-------|--------|
| Code Quality | 2/10 | 🔴 |
| Architecture | 4/10 | 🟠 |
| Security | 1/10 | 🔴 |
| DevOps | 2/10 | 🔴 |
| **Overall** | **2/10** | **🔴** |

**Scoring guide:** 8–10 🟢 Good · 5–7 🟡 Needs work · 3–4 🟠 Poor · 0–2 🔴 Critical

---

## Repository Overview

| Metric | Value |
|--------|-------|
| Primary Language | C# (.NET Framework 4.7.2) |
| Project Type | Multi-project ASP.NET Web API 2 solution (3 projects) |
| Total Files | ~1,590 (1,457 `.cs`, 46 `.pdf`, 23 `.cshtml`, 14 images) |
| Total Lines of Code | ~82,233 |
| Test Files | **0** (1 DTO file with "test" in name; no actual tests) |
| Estimated Test Coverage | **0% — Critical** |
| CI/CD Platform | **None** |
| Docker | **None** |
| IaC Tool | **None** |
| Security Risk Level | 🔴 **CRITICAL** |

---

## 🚨 Critical & High Priority Findings

- 🔴 **Critical** · **Security** · Firebase Admin SDK private key committed to repository — `rs-api-service/Notification/serviceAccountKey.json` — Contains a live RSA private key for Firebase project `sarwamart-dev` (`firebase-adminsdk-fbsvc@sarwamart-dev.iam.gserviceaccount.com`). Anyone with repo access can send push notifications to all app users and access Firebase services. **Revoke this key immediately** in the Firebase console, then load from a secrets manager — never from a file committed to source control.

- 🔴 **Critical** · **Security** · Production database credentials committed in plaintext — `rs-api-service/Web.config` line 9 — Connection string contains live server IP `208.91.199.99`, username `royya_sepa_test`, and password `Z~6kku93`. Rotate credentials immediately; load via environment variables or a secrets manager.

- 🔴 **Critical** · **Security** · Gmail SMTP password committed in plaintext — `rs-api-service/Web.config` line 34 — `SenderPassword = "kvmgwoeorlrojnuq"` for `sumsft@gmail.com`. An attacker can use this app password to send email as this account. Revoke the app password in Google Account settings; load via environment variable.

- 🔴 **Critical** · **Security** · Passwords "encrypted" with Base64 encoding, not real encryption — `rs-common-components/Utilities/Helper.cs` lines 21–35 — `Encryptdata()` converts passwords to Base64 (`Convert.ToBase64String(Encoding.UTF8.GetBytes(password))`), which is trivially reversible. Every password in the database can be decoded instantly. Migrate to `BCrypt`, `Argon2`, or `PasswordHasher<T>`; re-hash all stored passwords on next login.

- 🔴 **Critical** · **Security** · Real business PDF invoices committed to the repository — `rs-api-service/GeneratedPDF/` — 46 generated invoice/voucher PDFs (e.g., `1047_advr.pdf`, `1124_inv.pdf`) containing real transaction data are committed to git history. Remove from history using `git filter-repo`; add `GeneratedPDF/` to `.gitignore`.

- 🟠 **High** · **Security** · Login endpoint uses broken `_`-delimiter credential encoding — `rs-api-service/Areas/Account/AccountController.cs` lines 31–33: `username_password.Split('_')[0]` / `[1]` — Identical anti-pattern to the `erma_api` codebase. Passwords containing `_` silently break authentication. Replace with a proper `LoginRequest` DTO.

- 🟠 **High** · **Security** · JWT keys and trivial access keys committed — `rs-api-service/Web.config` lines 20–22, 14–15 — `JWTKeys = "92607ee9-..."`, `MobileAccessKey = "1234"`, `WebAccessKey = "1234"`. Anyone sending `Authorization: 1234` bypasses JWT auth. Rotate and move to environment variables.

- 🟠 **High** · **Security** · `customErrors mode="Off"` and `debug="true"` in production config — `rs-api-service/Web.config` lines 62–63 — Raw ASP.NET yellow screen of death (YSOD) error pages expose stack traces, file paths, and server internals to any client. Set `customErrors mode="RemoteOnly"` and `debug="false"` immediately.

- 🟠 **High** · **Security** · Hardcoded OTP bypass — `rs-api-service/Web.config` line 28 — `PhoneOTP = "123456"`. Any user can authenticate with OTP `123456` on any phone number. Remove this config value; OTPs must be generated and validated server-side only.

- 🟠 **High** · **Security** · OTP generation uses non-cryptographic `System.Random` — `rs-common-components/Utilities/Helper.cs` lines 8–13 — `Random` is seeded from the system clock and is predictable. Replace with `System.Security.Cryptography.RandomNumberGenerator` to generate unpredictable OTPs.

- 🟠 **High** · **DevOps** · User-uploaded product images committed to repository — `rs-api-service/UploadFiles/` — 14 user-uploaded images (with timestamps suggesting real usage: `simage_20250410_*.jpg`, `simage_20250411_*.jpg`) are tracked in git. User uploads must never be in source control; serve from object storage (Azure Blob, S3) and add `UploadFiles/` to `.gitignore`.

- 🟠 **High** · **DevOps** · No CI/CD pipeline — No GitHub Actions, GitLab CI, or any pipeline configuration found — Add a GitHub Actions workflow: build → (test) → lint, running on every pull request.

---

## 📝 Code Quality

**Summary:** With 82,000+ lines and zero tests, multiple god-object service files exceeding 1,000 lines, a fatally misnamed "encrypt" function that is actually Base64 encoding, and silent exception swallowing throughout, the codebase has deep quality debt.

**Metrics:**
- Languages: C# only
- Total files / lines: 1,457 `.cs` / ~82,233 lines (across 3 projects)
- Test coverage estimate: **0%** (0 test files, 1 mock file)
- TODO/FIXME count: 12 across 5+ files

**Findings:**

- 🟠 **High** `[testing]` Zero test files across 1,457 source files. The codebase has a `Mocks/MockTokenService.cs` suggesting tests were planned but never written. Add unit tests for at minimum: `Helper.Encryptdata/Decryptdata`, `TokenService`, `AccountService`, and all auth filters.

- 🟠 **High** `[complexity]` At least 10 files exceed 500 lines; 8 exceed 1,000 lines. The worst offenders are: `RoyyaSepaModel.Context.cs` (5,766 lines), `BuyerRequestRepository.cs` (1,147 lines), `StockEntryRepository.cs` (1,081 lines), `InvoiceService.cs` (1,070 lines), `StockEntryService.cs` (1,050 lines), `UserRegistrationService.cs` (947 lines). Split each into single-responsibility classes.

- 🟡 **Medium** `[quality]` `Helper.Encryptdata()` is misleading by name and implementation — it is Base64 encoding, not encryption. The `Decryptdata()` companion confirms it is fully reversible. Rename to `ToBase64` / `FromBase64` if used for encoding, or replace with real hashing for passwords.

- 🟡 **Medium** `[quality]` Silent exception swallowing in `Helper.cs` lines 31 and 42 — both `Encryptdata` and `Decryptdata` catch all exceptions and return empty string with no logging. This hides password handling failures silently. Remove the try/catch or at minimum log the exception.

- 🟡 **Medium** `[quality]` `XXX` placeholders in production PDF invoice templates — `SubscriptionInvoiceService.cs` lines 476–536 and `AdvancePaymentService.cs` lines 559–565 contain `"Bank Acc : " + "XXXXX"`. Real invoices are generated with placeholder bank account numbers, producing incorrect documents for customers.

- 🟡 **Medium** `[testing]` 2 TODO comments in filter files (`ApiKeyAuthFilter.cs:38`, `APISkipAuthorizationFilter.cs:32`) about caching reflection lookups — reflection runs on every API request with no caching, causing unnecessary overhead.

- 🔵 **Low** `[quality]` `Archive/` directory exists inside `rs-api-service/` suggesting old code is being stored in-tree rather than using git history. Delete the `Archive/` folder and use git tags or branches for historical versions.

- 🔵 **Low** `[quality]` The `ValuesController.cs` (35 lines) appears to be the default ASP.NET Web API scaffold controller and was never removed.

---

## 🏗️ Architecture

**Summary:** The 3-project solution (API service, common components, Entity Framework) shows good intent for separation of concerns, but the extremely large EF context and service files, legacy .NET Framework 4.7.2 (end-of-support), and 30+ Areas without feature boundaries create a maintenance risk.

**Metrics:**
- Project type: Multi-project ASP.NET Web API 2 solution
- Projects: `rs-api-service` (755 `.cs`), `rs-common-components` (400 `.cs`), `rs-entityframework` (304 `.cs`)
- Monorepo: Yes (3 projects in one solution)
- Layered structure: Yes — `Areas/`, `Services/`, `Respository/`, `Filters/`, `Helpers/`, `Constants/`
- Direct dependencies: ~20+ NuGet packages (via assembly binding redirects in Web.config)

**Findings:**

- 🟠 **High** `[dependencies]` Project targets `.NET Framework 4.7.2` — a legacy runtime with no active feature development (mainstream support ended January 2023). Migrate to `.NET 8` (current LTS) to gain performance improvements, modern security APIs, and access to the current ASP.NET Core ecosystem.

- 🟡 **Medium** `[complexity]` `rs-entityframework/RoyyaSepaModel.Context.cs` is a 5,766-line auto-generated EF6 model context — a classic god object. Consider splitting into feature-bounded `DbContext` classes or migrating to EF Core with bounded contexts.

- 🟡 **Medium** `[structure]` 30+ Areas (`StockEntry`, `BuyerReq`, `Invoices`, `Payment`, etc.) are flat directories with no cross-cutting feature boundaries enforced. As the codebase grows, this creates a risk of tight coupling between areas. Define explicit service contracts between feature areas.

- 🟡 **Medium** `[coupling]` The `rs-common-components` project contains both cross-cutting infrastructure (logging, caching, DBProvider) and domain-specific DTOs and helpers. Separate infrastructure concerns from domain DTOs to improve reusability.

- 🟡 **Medium** `[dependencies]` Binary DLL committed to the repository — `Dlls/PdfFileWriter.dll` — Binary dependencies in source control cannot be audited for vulnerabilities and complicate builds. Replace with a NuGet package reference (PdfFileWriter is available on NuGet).

- 🔵 **Low** `[structure]` The `Respository` folder name is a typo (`Respository` instead of `Repository`) propagated across all 3 projects. Fix the spelling consistently.

- 🔵 **Low** `[dependencies]` Entity Framework 6 (classic) is used rather than EF Core. EF Core is the actively maintained path, with significant performance improvements. Plan a migration.

---

## 🔒 Security

**Risk Level:** 🔴 **CRITICAL**

**Summary:** This repository has the most severe category of credential exposure: a live Firebase Admin private key, a production database password, and a live SMTP app password are all stored in plaintext in committed files — any person who has ever cloned the repository has had access to all three.

**Findings:**

- 🔴 **Critical** `[secrets]` **Firebase Admin SDK private key** — `rs-api-service/Notification/serviceAccountKey.json` — Full RSA private key for `firebase-adminsdk-fbsvc@sarwamart-dev.iam.gserviceaccount.com`. Grants Firebase Admin SDK access. **Revoke immediately** in Firebase Console → Project Settings → Service Accounts. Use Application Default Credentials or Secret Manager going forward.

- 🔴 **Critical** `[secrets]` **Production SQL Server credentials** — `rs-api-service/Web.config` line 9 — `data source=208.91.199.99; user id=royya_sepa_test; password=Z~6kku93`. A public IP with real credentials. Change the password immediately; restrict the DB to a VPN/private network.

- 🔴 **Critical** `[secrets]` **Gmail app password** — `rs-api-service/Web.config` line 34 — `SenderPassword = "kvmgwoeorlrojnuq"`. Revoke in Google Account → Security → App Passwords.

- 🔴 **Critical** `[injection]` **Passwords stored as reversible Base64** — `rs-common-components/Utilities/Helper.cs` — `Encryptdata()` is Base64 encoding, not encryption. Every user's password can be trivially recovered by anyone with DB read access (`Convert.FromBase64String(...)` → UTF8 string). Migrate to `BCrypt.Net` or `Microsoft.AspNetCore.Identity.PasswordHasher`.

- 🔴 **Critical** `[data]` **46 generated PDF invoices in git history** — `rs-api-service/GeneratedPDF/` — These likely contain real transaction amounts, customer names, and financial data. Run `git filter-repo --path rs-api-service/GeneratedPDF --invert-paths` to purge from history.

- 🟠 **High** `[config]` `customErrors mode="Off"` — `Web.config` line 62 — Full ASP.NET error pages (stack trace, assembly info, file paths) are returned to any client. Change to `customErrors mode="RemoteOnly"` or `mode="On"`.

- 🟠 **High** `[config]` `debug="true"` in compilation — `Web.config` line 63 — Compilation debug mode should never be deployed to production: it disables script bundling, prevents GC of in-memory state, and leaks debug symbols. Set `debug="false"`.

- 🟠 **High** `[secrets]` Hardcoded OTP `PhoneOTP = "123456"` — `Web.config` line 28 — Any user can bypass mobile OTP verification by submitting `123456`. Remove the config key; there must be no "master OTP".

- 🟠 **High** `[injection]` OTP generation uses `System.Random` — `Helper.cs` lines 8–13 — Predictable seed allows OTP brute-force. Replace with `System.Security.Cryptography.RandomNumberGenerator.GetInt32(0, 999999)`.

- 🟠 **High** `[secrets]` JWT `JWTKeys`, `WebSecurityKey`, `AudienceId`, `JWTIssuer` all hardcoded in `Web.config`. Tokens can be forged by anyone with repository access.

- 🟠 **High** `[data]` 14 user-uploaded product images committed — `rs-api-service/UploadFiles/` — Real user content with embedded timestamps. Remove from history; serve uploads from object storage.

- 🟡 **Medium** `[hygiene]` No `SECURITY.md`. Add a security disclosure policy.

- 🔵 **Low** `[hygiene]` No Dependabot configuration. NuGet packages receive no automated vulnerability alerts.

---

## ⚙️ DevOps

**Summary:** The repository has a proper Visual Studio `.gitignore`, but it failed to prevent the commitment of the most sensitive files (service account key, Web.config with credentials, binary DLL, user uploads, and generated PDFs), and there is no CI/CD, containerization, README, or LICENSE.

**Checklist:**
- [ ] CI/CD pipeline
- [ ] Tests in CI
- [ ] Docker
- [ ] IaC
- [ ] README
- [ ] LICENSE
- [ ] CHANGELOG
- [ ] Lock file

**Findings:**

- 🟠 **High** `[ci]` No CI/CD pipeline. A codebase of 82,000+ lines with no automated build or quality gate. Add GitHub Actions: build (`msbuild`) → lint (`dotnet format`) → (test once added).

- 🟠 **High** `[hygiene]` `.gitignore` exists but fails to exclude `Web.config`, `Notification/serviceAccountKey.json`, `UploadFiles/`, and `GeneratedPDF/`. Add explicit exclusion rules for all of these. Note: Web.config exclusion requires an alternative configuration loading strategy (environment variables, Azure App Configuration).

- 🟡 **Medium** `[hygiene]` No `README.md`. A project of this size and complexity needs at minimum: project description, setup instructions, environment variable reference, and API overview.

- 🟡 **Medium** `[hygiene]` No `LICENSE` file. The repository is public; without a license it is legally "all rights reserved" by default.

- 🟡 **Medium** `[docker]` No Dockerfile or docker-compose. Running locally requires a configured IIS Express environment. Add Docker support to enable reproducible local development and cloud deployment.

- 🔵 **Low** `[hygiene]` No `CHANGELOG.md`. With 30+ feature areas, a changelog would help track what changed between releases.

- 🔵 **Low** `[release]` No version or semantic versioning evident. Add `<AssemblyVersion>` and git tagging to track deployed versions.

---

## 🎯 Top 3 Quick Wins

1. **Revoke the Firebase service account key right now** — Go to Firebase Console → Project Settings (`sarwamart-dev`) → Service Accounts → Delete the key with ID `5748d3f3f9cdd2fabd6073f539619c8e3de5b20e`. This is a live private key exposed to anyone who has cloned this public repo. Create a new key and store it in a secrets manager, not a committed file.

2. **Change the production DB password and SMTP app password** — The SQL Server at `208.91.199.99` (password `Z~6kku93`) and Gmail account (app password `kvmgwoeorlrojnuq`) are both compromised. Rotate both immediately. Then remove them from `Web.config` and load via environment variables.

3. **Fix `customErrors` and `debug` in Web.config** — Change `customErrors mode="Off"` → `mode="RemoteOnly"` and `debug="true"` → `debug="false"`. Two attribute changes, immediate production security improvement.

---

## 🗺️ Recommended Roadmap

### This week (critical fixes)
- Revoke Firebase service account key (`serviceAccountKey.json`) and regenerate via secrets manager
- Rotate production DB password (`208.91.199.99` / `royya_sepa_test`)
- Revoke Gmail app password (`kvmgwoeorlrojnuq`) and regenerate
- Set `customErrors mode="RemoteOnly"` and `debug="false"` in `Web.config`
- Remove hardcoded OTP (`PhoneOTP = "123456"`)
- Purge `GeneratedPDF/` and `UploadFiles/` from git history (`git filter-repo`)
- Add `serviceAccountKey.json`, `Web.config`, `UploadFiles/`, `GeneratedPDF/` to `.gitignore`

### This month (quality improvements)
- Migrate password storage from Base64 to BCrypt / `PasswordHasher<T>`; re-hash on next login
- Replace `System.Random` OTP generation with `RandomNumberGenerator.GetInt32`
- Fix the `_`-delimiter login endpoint with a proper `LoginRequest` DTO
- Add GitHub Actions CI pipeline (build + lint)
- Replace the binary `Dlls/PdfFileWriter.dll` with a NuGet package reference
- Fix bank account `XXXXX` placeholders in invoice PDF generation
- Add `README.md` and `LICENSE`

### This quarter (strategic improvements)
- Migrate from `.NET Framework 4.7.2` to `.NET 8` (current LTS) — major effort, highest long-term value
- Migrate from EF6 to EF Core; split the 5,766-line `RoyyaSepaModel.Context.cs` into bounded contexts
- Break up god-object service/repository files (8 files > 1,000 lines each)
- Add unit tests for core business logic (token issuance, OTP validation, invoice generation)
- Move user uploads to Azure Blob Storage / S3 (remove `UploadFiles/` from the app entirely)
- Add Dependabot for automated NuGet vulnerability scanning

---

## Findings Summary

| Severity | Count |
|----------|-------|
| 🔴 Critical | 6 |
| 🟠 High | 12 |
| 🟡 Medium | 11 |
| 🔵 Low | 8 |
| **Total** | **37** |

---

*Generated by repo-intel · https://github.com/your-org/repo-intel*

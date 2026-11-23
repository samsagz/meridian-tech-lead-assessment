# Review of PR-388: MX Home Essentials quick quote

## 1. Findings

Lines are from the new version of each file. Java paths are relative to `src/main/java/com/meridian/channel/`.

| # | File | Line | Issue | Severity | Change requested |
|---|---|---|---|---|---|
| 1 | `src/main/resources/application-mx.yml` | 7–8 | Salesforce client ID and client secret committed to the repository. The secret is now in the branch history (C5: no secrets in code or configuration files) | Critical | Treat the secret as exposed and rotate it today with the Salesforce platform team, whether this PR merges or not. Remove the file; secrets come from Key Vault with a managed identity |
| 2 | `home/HomeQuoteController.java` | 26–27 | Country and customer come from the `X-Country` and `X-Customer-Id` headers. Any authenticated customer can read the name and address of another customer, in any country (C5). The codebase already forbids this (`CustomerContext.java:6-7`) | Critical | Use `CustomerContext.from(jwt)`, as `QuoteController.java:29` does, and remove both headers from the contract |
| 3 | `integration/SalesforceRatingClient.java` | 19–22, 36–73 | Direct calls to the MX org for the customer profile and the rating, outside the API Manager (C4). The consumption is invisible and has no limit (C6), and the BFF holds Salesforce credentials. The client also has no timeout or circuit breaker (`new RestTemplate()`, line 22), requests a new OAuth token on every call (60–67), sends no correlation ID and fails on empty or partial answers (40, 45, 57). A slow org would hold every BFF thread (C1, C2) | Critical | Remove the client. Quote through `QuoteExperienceClient` against the Home Essentials contract of `quote-xapi`, with a stub until the Process API exists (§3) |
| 4 | `home/HomeQuoteService.java` | 22–23 | Logs the full name, national ID and email (C5) | High | Log only the quote ID, country, product and correlation ID; add a test that fails if personal data reaches the logs |
| 5 | `home/HomeQuoteResponse.java`; `home/CustomerProfile.java`; `integration/SalesforceRatingClient.java` | 5–6; 3; 38–39 | Returns the name and address, and reads the national ID and email. A quick quote needs none of them (data minimization, C5) | High | Remove `insuredName`, `insuredAddress` and `CustomerProfile` |
| 6 | `quote/QuoteResponse.java`; `api/channel-quote-api.yaml` | 9; 106, 112 | Renames `premiumAmount` to `totalPremium`: a breaking change for the CO and CL frontends and the advisor portal. The contract stays at 1.4.0, the checklist says "No breaking changes", and without contract tests (TD-05) the build stays green | High | Revert the rename. A new name needs a new major version and a deprecation period agreed with every consumer |
| 7 | `integration/SalesforceRatingClient.java`; `src/main/resources/application-mx.yml` | 24–34; 1–8 | An unconditional `@Component` whose properties exist only in the `mx` profile. Any deployment without that profile, such as CO and CL today, fails at startup | High | Country behavior comes from configuration that every country has, and the build starts the application with each country configuration |
| 8 | `home/HomeQuoteService.java` | 28–37, 44 | Country logic in conditionals: 16% and 19% tax, a surcharge for postal codes that start with "77", and `MXN` or `COP`, so Chile would be priced in COP (C3, `CountryProperties.java:9-10`). The surcharge is a product rule and belongs to rating | High | The BFF does not calculate prices: the premium with taxes comes from the Experience API. Display settings, such as the currency, come from `CountryProperties` |
| 9 | `src/test/java/com/meridian/channel/home/HomeQuoteServiceTest.java` | 15–16, 21–28, 30–33 | The test mocks the class under test, so it only checks the answer of the mock, and `quickQuoteForColombia` checks that the mock is not null. It runs no production code, so the 80% coverage in the description cannot be right. The PR says an AI assistant generated these tests and someone reviewed them | High | Test the real service with mocked dependencies: country configuration, errors, identity from the token (403 on a country mismatch) and no personal data in logs. AI-generated tests get the same review as any other code |
| 10 | `home/HomeQuoteResponse.java`; `home/HomeQuoteService.java`; `api/channel-quote-api.yaml` | 3–10; 25–41; 133–149 | Money as `double` and JSON `number`, with no rounding of the monthly value (39), under the same field name that `QuoteResponse` uses for a decimal string. No `ratingTableVersion` or `expiresAt`, which the quote must record and show (`SALESFORCE_CONSTRAINTS.md:25`). The `quoteId` is a random UUID that is never stored, so nobody can retrieve or buy the quote | Medium | Return the quote of the Experience API: its ID, decimal strings with an ISO 4217 currency, the rating-table version and the expiry |
| 11 | `src/main/resources/application.yml`; `home/HomeQuoteController.java` | 30–32; 15 | The flag is `true` in the base configuration, so it is on for every country and environment, not only for the MX pilot. `@ConditionalOnProperty` is read at startup, so turning it off needs a redeploy | Medium | Off by default; a runtime flag per country (TD-10), on only for MX during the pilot |
| 12 | `home/HomeQuoteController.java`; `api/channel-quote-api.yaml` | 14; 53–80 | A product endpoint (`/v1/home-quotes`) next to the generic `POST /v1/quotes`, which already takes `product` and `answers`. Every new product would add endpoints, against "make the next product cheaper" (`BUSINESS_REQUESTS.md:17`). The contract also puts identity in headers, documents only `201` and has no required fields | Medium | Quote Home Essentials through `/v1/quotes` with `product: HOME_ESSENTIALS`, with problem details for 400, 401, 403 and 503, as the rest of the contract does |
| 13 | `home/HomeQuoteController.java` | 25–28 | Advisors must sell from day one (BR-01), but the BFF accepts only the customer realm (`application.yml:8`) and the PR has no path for advisors | Medium | Advisors use the workforce token: the requested country must be in `countries`, the role must allow quoting, and the audit log keeps the advisor (`ARCHITECTURE.md` §6) |
| 14 | `home/HomeQuoteRequest.java` | 8–9 | No postal-code format and no maximum for `squareMeters` | Low | A postal-code pattern per country, from configuration, and a maximum |
| 15 | `PR-388.md` (description) | 35–36 | The checklist says "No breaking changes" and "Secrets managed according to the standard"; both are false | Low | The PR template asks for evidence (contract-test and secret-scan results), not checkboxes |

## 2. Verdict

**Request changes.** With three Critical findings, a merge would expose customer data across countries, keep a secret in the history and open a direct path to the MX org.

Before the merge:
1. Secret rotated and removed (#1). The rotation happens today, independently of the merge.
2. Identity and country from the token (#2).
3. No direct Salesforce client (#3, #7).
4. Rename reverted (#6).
5. No personal data in logs or responses (#4, #5).
6. No price calculation and no country conditionals in the BFF (#8).
7. Tests that run the real code (#9).
8. Flag off by default (#11).

The Medium and Low findings can go in follow-up PRs, with an issue for each one before the merge. As a Nimbus change for MX, the release to production also needs my recorded approval (INC-0907 action A7a).

## 3. Refactoring proposal

Target path (`ARCHITECTURE.md` §4, ADR-0002): `web-quote-ui` with the MX configuration → `channel-bff` `POST /v1/quotes` with `product: HOME_ESSENTIALS` → `QuoteExperienceClient` (timeouts, circuit breaker, API Manager credentials, correlation ID) → `quote-xapi` `POST /v1/MX/quotes` → quote Process API (price grid or live rating) → `sf-policy-sapi-mx` → MX org.

The quote needs no personal data. If the purchase step needs to prefill customer data, it reads only the fields that the screen shows, through the same API-led path.

| Step | What we ask Nimbus | Why in this order |
|---|---|---|
| 0. Today | Close the secret: rotation with the Salesforce platform team and file removed. We also ask how a credential of the MX org was issued for `channel-bff`, because credentials go to the owner of the consuming API (`SALESFORCE_CONSTRAINTS.md:42`) | A live risk, independent of the merge |
| 1 | Contract first: the Home Essentials quote is defined in the `quote-xapi` contract (`docs/api/quote-experience-api.yaml`), and Nimbus builds against a stub generated from it, with consumer contract tests (TD-05) | Nimbus is not blocked for the six weeks of the integration backlog (`PR-388.md:19`), and the later switch is configuration |
| 2 | Rework the BFF: generic `/v1/quotes`, `CustomerContext` from the token, no price calculation, MX in `CountryProperties`, no personal data, rename reverted | Brings the PR to the standards (`ARCHITECTURE.md` §8) |
| 3 | Runtime flag per country, off by default | Safe release (TD-10) |
| 4 | Advisors with the workforce token | BR-01 asks for advisors from day one |
| 5 | Tests on the real code, contract tests, security tests and a log-redaction test, all in the Meridian pipeline | The gates of `ENGINEERING_STANDARDS.md`; INC-0907 action A7b |
| 6 | Replace the stub with `quote-xapi` and the Process API when they exist | The Process API moves forward with the MuleSoft loan, requested before 15 Dec 2026 |

## 4. Feedback to the author

Hi Diego, thank you for this PR. It is the first Mexican feature in `channel-bff`, which is the direction we want, and the validation annotations, the small records and the feature flag show care. I cannot approve it yet. I want to explain the principle behind each change, because they apply beyond this PR.

1. **Identity comes from the token, never from the caller.** Anyone with a valid login can set `X-Country` and `X-Customer-Id`, so any customer could read the name and address of another customer, even in another country. This is why `CustomerContext.from(jwt)` exists.
2. **Secrets never go in the repository.** We must treat the secret in `application-mx.yml` as exposed, and I already asked the Salesforce platform team to rotate it today. Secrets come from Key Vault with a managed identity.
3. **We reach Salesforce only through the integration platform.** This is how we see and limit the consumption of each org, which every team in the country shares. I know the Home Process API is six weeks away. Instead of a direct client, please build against the Home quote contract of `quote-xapi` with a stub. You are not blocked, and the switch later is only configuration. I am also working to bring the Process API forward.
4. **A contract is a promise to every consumer.** Renaming `premiumAmount` breaks the Colombian and Chilean frontends and the advisor portal, even if our build stays green. A new name needs a new version and a deprecation period.
5. **Country differences live in configuration, and prices come from rating.** The tax rates, the coastal surcharge and the country conditionals do not belong in the BFF; as written, Chile would be priced in COP. Money is `BigDecimal` end to end.
6. **Personal data stays out of logs and responses** unless the screen needs it. A quick quote needs neither the national ID nor the email.
7. **Tests must be able to fail.** These tests mock the class they test, so they pass whatever the code does. AI-generated tests are welcome; we only review them for this kind of problem.

The full list with lines is in the review, and the smaller items can go in follow-up PRs. Can we work together for an hour tomorrow on the stub and the flag? I want to help you meet the date, not block it.

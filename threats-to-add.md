# Threats to Add — "Citizen Login and Register" (login register flow.json)

Full descriptions of the threats to add to the Threat Dragon model, grouped by diagram element.
Every entry has a **title**, **STRIDE type**, **description** and **mitigations** ready to adapt.

**How to use:** in Threat Dragon, right-click the element (or select the flow arrow) → **Threats** →
**Add threat**, then set the type and fill in the description and countermeasures in your own words.

> Academic integrity reminder: per the assignment handout, GenAI may only be used for
> editing/formatting the report. Use this list to know **what** to cover, but write the
> descriptions and mitigations in Threat Dragon in your own words.

Note: there are **three** flows named `verification_email` — disambiguate them by the
source/target given in each entry.

---

## 1. Email chain — currently ZERO threats (highest priority)

### T1 — Spoofing: Verification email spoofed as coming from SecureReports
**Element:** `Email Service Provider` (process)

**Description:** An attacker forges a verification email that appears to come from SecureReports. If the sending domain has no SPF, DKIM or DMARC records, receiving mail servers cannot tell the real email from the forged one, and the citizen is likely to trust the fake message.

**Mitigations:**
- Configure SPF, DKIM and DMARC on the domain used to send verification emails.
- Send verification emails only from a dedicated, verified sending domain.
- Monitor DMARC reports for unauthorised sending attempts.

### T2 — Information Disclosure: Verification link intercepted on the SMTP hop
**Element:** `verification_email` flow (**Email Service Provider → Receiving MTA**)

**Description:** Email is only encrypted hop-by-hop (TLS between mail servers), not end-to-end. The verification link — effectively an account-activation credential — can be read by anyone able to observe an unencrypted hop or the mail operator's infrastructure.

**Mitigations:**
- Use short-lived, single-use verification tokens so an intercepted link cannot be replayed later.
- Never include passwords or password-reset secrets in email.
- Treat email as an untrusted channel: a token in email must never be the sole proof of identity for sensitive actions.

### T3 — Spoofing: Phishing "verify your account" email clone
**Element:** `verification_email` flow (**Citizen Mailbox → Citizen**)

**Description:** An attacker sends a fake "verify your account" email that looks like the real one and links to a cloned SecureReports page, harvesting the citizen's email, password or other personal data. This is the social-engineering angle of the registration process.

**Mitigations:**
- Publish and enforce DMARC on the sending domain so clones are marked as spam.
- Monitor for look-alike/typosquatted domains and take them down or block them.
- Educate citizens that SecureReports never asks for passwords by email.

### T4 — Information Disclosure: Verification link read from a shared/forwarded mailbox
**Element:** `Citizen Mailbox` (data store)

**Description:** The verification email can be read by anyone with access to the citizen's mailbox — a shared family account, forwarded mail, or a device left unlocked. Because the link has no expiry and works from any browser, that person can activate the account before the real owner does.

**Mitigations:**
- Make verification links single-use and expiring.
- Where possible, bind verification to the registering browser/device.
- Require re-verification before sensitive actions if activation looks unexpected.

### T5 — Tampering: Email vendor/MTA compromise (supply chain)
**Element:** `Email Service Provider` (process) **and** `Receiving MTA` (process)

**Description:** The email service and mail transport are third-party software/infrastructure. If the vendor is compromised, or its API keys are stolen, an attacker can send arbitrary mail as SecureReports — including fake verification or password messages — or read mail in transit.

**Mitigations:**
- Assess the vendor's security posture (audits, certifications) before use.
- Store email API keys in a secrets vault with least-privilege access and rotate them regularly.
- Monitor email sending volumes and patterns for anomalies.

### T6 — Information Disclosure: Verification token retained in provider logs
**Element:** `Email Service Provider` (process)

**Description:** Email providers commonly log message content or metadata, and support tickets can expose message details. A verification token that appears in the email body or URL can end up in logs or support records, where it may be exposed later.

**Mitigations:**
- Sign a data-processing agreement covering data retention and deletion.
- Minimise and redact logging of message content; never log full tokens.
- Use short-lived tokens so any logged value becomes useless quickly.

### T7 — Spoofing: Mailbox compromise → account takeover via never-expiring link
**Element:** `Citizen Mailbox` (data store)

**Description:** The assignment specification states there is **no timeout for verification links**. If an attacker gains access to a mailbox — even months later — they can use the still-valid link to activate the account and take it over, or register accounts on behalf of victims.

**Mitigations:**
- Add an expiry to verification links (**proposed change** — analyse it in the discussion section with the updated Threat Dragon model).
- Invalidate the link once used and on any password/email change.
- Flag this as an "insecure by design" finding in section 8 of the report.

---

## 2. `Validate Credentials` (process — currently 4 threats)

### T8 — Information Disclosure: Account enumeration via error messages or timing
**Element:** `Validate Credentials` (process)

**Description:** If the login or verification responses differ between "no such user", "wrong password" and "email not verified", an attacker can enumerate which email addresses have registered accounts. Timing differences (e.g., a hash check only performed for existing accounts) leak the same information.

**Mitigations:**
- Return the same generic error message for every failure case.
- Perform the same work (hash + compare) for unknown and known accounts; use constant-time comparison.
- Add uniform delays or rate limiting so timing differences are not observable.

### T9 — Spoofing: Credential stuffing / automated brute force
**Element:** `Validate Credentials` (process)

**Description:** Attackers replay username/password pairs leaked from other breaches (credential stuffing) or make many guesses against accounts (brute force). Because passwords are often reused, this can succeed without exploiting any vulnerability.

**Mitigations:**
- Enforce per-IP and per-account rate limiting with exponential backoff.
- Offer MFA as an additional factor (separation of privilege).
- Screen submitted passwords against breached-password lists at registration/reset.

### T10 — Denial of Service: Account lockout abuse
**Element:** `Validate Credentials` (process)

**Description:** If failed attempts lock an account, an attacker can deliberately submit wrong passwords for a victim's account to lock them out of SecureReports — a denial of service against the citizen.

**Mitigations:**
- Throttle based on IP address and device fingerprint rather than locking the account outright.
- Use temporary, exponentially increasing delays instead of permanent lockout.
- Provide a lockout-free fallback such as MFA or email verification to restore access.

### T11 — Information Disclosure: Timing side-channel on password hash comparison
**Element:** `Validate Credentials` (process)

**Description:** If the stored hash is compared with a naive byte/string comparison that returns early on the first mismatch, an attacker can measure response time to recover the hash byte-by-byte and then authenticate with it.

**Mitigations:**
- Use the password library's constant-time verify function (e.g., bcrypt.compare, Argon2 verify).
- Never implement a custom hash comparison.
- Keep hash comparison server-side only.

---

## 3. `Login` (process — currently 8 threats)

### T12 — Spoofing: Session fixation
**Element:** `Login` (process)

**Description:** An attacker pre-issues (fixes) a session ID — for example by sending a link containing one — and gets the victim to log in with it. After login the attacker knows the session ID and can impersonate the citizen without credentials. (Lecture 5: session hijacking and fixation.)

**Mitigations:**
- Always regenerate the session ID on successful login.
- Reject session IDs the server did not issue (server-side session store).
- Bind sessions to context (IP/user-agent) and rotate on privilege change.

### T13 — Spoofing: Sessions never expire server-side
**Element:** `Login` (process)

**Description:** If session cookies have no server-side expiry, a stolen cookie remains valid indefinitely, letting an attacker impersonate the citizen long after the theft.

**Mitigations:**
- Store sessions server-side with idle-timeout and absolute-timeout values.
- Invalidate the session on logout, password change and privilege change.
- Use short cookie lifetimes combined with re-authentication for sensitive actions.

---

## 4. `Verify Email` (process — currently 8 threats)

### T14 — Spoofing: Verification code/link brute-forced
**Element:** `Verify Email` (process)

**Description:** Even if the RNG produces a strong token, the verify endpoint itself can be attacked: an attacker who knows or enumerates a user ID can guess short codes, or brute-force the link, if there is no attempt limiting.

**Mitigations:**
- Cap the number of verification attempts per link/token.
- Rate limit verification requests per IP and per account.
- Invalidate the token after N failed attempts and require a fresh one.

### T15 — Spoofing: Verification link reuse after account already verified
**Element:** `Verify Email` (process)

**Description:** A verification link that remains valid after the account is verified can be replayed: an attacker who obtains a copy — from logs, a forwarded email or browser history — can re-verify or interfere with a later verification flow.

**Mitigations:**
- Make tokens single-use; invalidate immediately after the first successful verification.
- Invalidate all outstanding verification tokens when the account is created or the email changes.
- Store the token's hash and its used/expired state server-side.

### T16 — Information Disclosure: Verification token leaks via logs, referrer or history
**Element:** `Verify Email` (process)

**Description:** The verification token can be exposed through application logs, the Referer header sent to third-party resources on the verify page, or browser history — any of which allows replay of the link.

**Mitigations:**
- Never log full verification tokens.
- Set `Referrer-Policy: no-referrer` on the verification page.
- Where possible, exchange the token for a short-lived session via a POST form instead of a GET URL.

### T17 — Spoofing / Elevation of Privilege: No expiry on verification links (insecure by design)
**Element:** `Verify Email` (process)

**Description:** The specification explicitly states that verification links have **no timeout**. Any token ever issued stays valid forever, so a link harvested from logs, a compromised mailbox or a data breach can be used to activate an account at any time — this is insecure by design.

**Mitigations (proposed change):**
- Add an expiry (e.g., 24–72 hours) and a re-send/re-verification flow — this is a **proposed change**, so it must be threat-analysed in the discussion section with an updated Threat Dragon model.
- Invalidate tokens on use and on credential changes.
- Use this as your strongest "insecure by design" finding in section 8 of the report.

---

## 5. `Register` (process — currently 9 threats)

### T18 — Information Disclosure: Account enumeration via "email already registered"
**Element:** `Register` (process)

**Description:** Returning "an account with this email already exists" (or a different message/timing for existing vs new emails) lets an attacker confirm which addresses have accounts, feeding phishing and targeted attacks.

**Mitigations:**
- Return the same success message whether the email is new or already registered.
- Do not distinguish existing vs new accounts in responses or timing.
- Optionally send the verification email in both cases to avoid leaking account existence.

### T19 — Tampering: Email header injection via crafted email address
**Element:** `Register` (process)

**Description:** If the email address is concatenated into the outgoing message without validation, an attacker can embed CRLF/control characters to inject extra headers (Bcc, Reply-To, etc.), turning the verification email into a spam/phishing carrier or redirecting replies.

**Mitigations:**
- Validate email addresses against RFC 5321/5322 grammar; reject control characters.
- Use a mail library that handles headers safely — never build raw SMTP headers from user input.
- Reject invalid input before it reaches the mail path.

### T20 — Tampering: Bulk automated creation of fake citizen accounts
**Element:** `Register` (process)

**Description:** Automated scripts can register many fake accounts, polluting the citizen database and eroding the police's ability to trust reports — and each fake account may be used for abuse.

**Mitigations:**
- Rate limit registrations per IP and per network.
- Add CAPTCHA or proof-of-work on the registration form.
- Enforce email verification (already in the design) and monitor for mass registrations.

---

## 6. `Hashing` (process — currently 5 threats)

### T21 — Information Disclosure: No re-hash path when algorithm/work factor is upgraded
**Element:** `Hashing` (process)

**Description:** When the organisation upgrades the password-hashing algorithm or work factor, existing hashes are only re-computed if the application provides a re-hash path. Without one, old hashes stay weak forever (e.g., unsalted MD5/SHA-1) even though policy has changed.

**Mitigations:**
- Re-hash with the current parameters on every successful login when the stored hash is outdated (algorithm agility).
- Store the algorithm/parameters alongside each hash so upgrades are detectable.
- Periodically force re-login/re-verification for accounts with outdated hashes.

### T22 — Denial of Service: Expensive hash computation as a CPU DoS vector
**Element:** `Hashing` (process)

**Description:** Password hashing is deliberately expensive (bcrypt/Argon2 with a high work factor). An unauthenticated flood of login requests forces the server to compute many hashes, exhausting CPU and making the service unavailable — the stronger the hashing, the cheaper this attack is for the attacker.

**Mitigations:**
- Rate limit login attempts per IP/account **before** the expensive hash is computed.
- Use connection limits and a queue on the authentication endpoint.
- Do cheaper pre-checks (e.g., account existence) before the hash computation.

---

## 7. `Random Number Generator` (process — currently 6 threats)

### T23 — Repudiation: No audit trail that codes/session IDs came from the legitimate RNG (fills placeholder #167)
**Element:** `Random Number Generator` (process)

**Description:** This replaces the current placeholder threat #167. Without audit logging, there is no way to prove — or dispute — that a given verification code or session ID was generated by the legitimate RNG process, enabling repudiation of account activations and logins.

**Mitigations:**
- Log token generation events (type, timestamp, account) **without logging the token itself**.
- Record activation and login events with the token ID for audit (compromise recording).
- If this does not apply, mark the threat N/A with an explicit reason instead of leaving the placeholder open.

### T24 — Spoofing: Verification code space too small for its lifetime
**Element:** `Random Number Generator` (process)

**Description:** Even a well-seeded generator is useless if the code space is too small for the token's lifetime: for example, a 6-digit numeric code that never expires can be brute-forced in practice. This complements the existing guessability threat #163.

**Mitigations:**
- Use ≥128 bits of entropy for verification tokens (or high-entropy alphanumeric codes).
- Pair the code space with expiry and attempt limits (see T14).
- Reject weak/legacy code formats on upgrade.

---

## 8. Data flows

### T25 — Information Disclosure: Plaintext password handled in application memory
**Element:** `password` flow (Register → Hashing), `given_password` flow (Validate Credentials → Hashing), `email_address/password` flow (Login → Validate Credentials)

**Description:** The password is passed in plaintext between application processes/components. Any compromised or buggy component in that path — a process that logs its inputs, dumps memory, or is replaced — can leak the password before it is ever hashed.

**Mitigations:**
- Minimise the number of components that ever touch the plaintext password; hash as close to the entry point as possible.
- Never log request bodies or form fields; wipe buffers that held passwords.
- Keep the process boundary small and apply least privilege to each component.

### T26 — Tampering: Login CSRF
**Element:** `submit_login_request` flow (Citizen → Login)

**Description:** A malicious site can submit a login form in the victim's browser (e.g., an auto-submitting form), logging the victim into the attacker's account. The victim's subsequent actions — reports, data — are then performed in the attacker's session. (Lecture 5: CSRF.)

**Mitigations:**
- Set `SameSite=Lax` (or Strict) on session cookies.
- Include an anti-CSRF token on the login form and verify it server-side.
- Require the Origin/Referer to match the SecureReports origin for state-changing requests.

### T27 — Spoofing: Session cookie missing Secure/HttpOnly/SameSite attributes
**Element:** `submit_login_response/session_cookie` flow (Login → Citizen)

**Description:** If the session cookie lacks `Secure`, it can be sent over plain HTTP; without `HttpOnly`, JavaScript can read it (enabling XSS-based theft); without `SameSite`, it is sent on cross-site requests (enabling CSRF). This complements the existing threats #222/#223 on the same flow.

**Mitigations:**
- Set `Secure`, `HttpOnly` and `SameSite=Lax/Strict` on the session cookie.
- Use the `__Host-` prefix so the cookie cannot be overwritten by subdomains.
- Combine with HTTPS-only enforcement (already assumed via the reverse proxy).

---

## 9. `Citizen Information Data Store` (data store — currently 3 threats)

### T28 — Information Disclosure: Citizen PII unencrypted at rest
**Element:** `Citizen Information Data Store` (data store)

**Description:** Email addresses and password hashes are stored in the database. If the database file, disk or backup media is obtained, the data can be read directly. Encryption at rest raises the cost of such theft.

**Mitigations:**
- Encrypt the database at rest (disk-level and/or column-level for PII).
- Manage keys in a KMS/HSM with rotation.
- Keep the database isolated to the app host (already assumed) and restrict administrative access.

### T29 — Elevation of Privilege: Mass assignment of `is_verified`/role attributes
**Element:** `Citizen Information Data Store` (data store)

**Description:** If the application binds request fields directly to account records, a crafted request could set `is_verified = true` (or a role/privilege attribute) without going through email verification — bypassing the verification control entirely. Complements the in-transit tampering threat #188 on the `register_citizen_info` flow.

**Mitigations:**
- Whitelist the exact fields the registration/login endpoints may set.
- Reject unknown or unexpected fields server-side.
- Keep privileged state changes on a separate, authorisation-checked write path.

### T30 — Repudiation: No audit trail of account data changes
**Element:** `Citizen Information Data Store` (data store)

**Description:** Without an audit log, a citizen can deny having registered, verified or changed credentials, and the police cannot reconstruct who did what with account data. (Lecture 1: compromise recording.)

**Mitigations:**
- Maintain an append-only audit log of account events: registration, verification, login, credential changes.
- Include timestamps, account ID and actor where applicable — never passwords.
- Monitor the audit log for anomalies.

### T31 — Information Disclosure: Unencrypted backups exfiltrated
**Element:** `Citizen Information Data Store` (data store)

**Description:** Backups often contain everything in the database — including password hashes and PII — and are frequently stored less securely than the primary database. An attacker who obtains a backup bypasses all runtime controls. (Extends existing #185.)

**Mitigations:**
- Encrypt backups at rest with keys held separately from the backup media.
- Restrict backup access with least privilege and log all backup access.
- Store backups off-site/in a separate security domain and test restoration.

---

## 10. `Citizen` (actor — currently 4 threats)

### T32 — Spoofing: Credential reuse from other breaches
**Element:** `Citizen` (actor)

**Description:** Citizens routinely reuse passwords across services. When another site is breached, the leaked credentials are tried against SecureReports (credential stuffing). This extends the existing weak-password (#1) and stolen-credentials (#3) threats.

**Mitigations:**
- Screen registration passwords against breached-password lists (e.g., the HIBP k-anonymity API).
- Encourage MFA and password managers; detect and notify on credential-stuffing patterns.
- Enforce strong passwords/passphrases per existing threat #1.

### T33 — Spoofing: Session left active on a shared/borrowed device
**Element:** `Citizen` (actor)

**Description:** A citizen may log in on a shared or borrowed device and leave the session active; the next user of that device can act as the citizen. Complements the stolen-session-cookie threat #211.

**Mitigations:**
- Enforce idle timeout and absolute session expiry (see T13).
- Provide a visible logout and a "log out of all sessions" control.
- Prompt for re-authentication before sensitive actions after periods of inactivity.

---

## Suggested order of work

1. **Email chain (T1–T7)** — whole elements have zero threats; highest risk to the 30-mark STRIDE section.
2. **Placeholder #167 (T23)** and the truncated threat title #21 ("Register process response is ") — unfinished items look worst to a marker.
3. **Login/Verify gaps (T8–T17)** — map 1:1 to Lecture 5 (sessions, CSRF, cookies) and Lecture 3 (OWASP Top 10) content; easiest to write well.
4. **Store/actor/internal-flow depth (T18–T33)**.

## Also fix while in the editor

- Dangling flows: `register_citizen_info` and `hashed_given_passowrd` have floating (unconnected) sources.
- Typo in flow name: `hashed_given_passowrd` → `hashed_given_password`.
- Empty trust boundary (id `7cfa7384…`) with no contained elements or crossing flows — delete or repurpose.
- Element flags: set `storesCredentials: true` on `Citizen Information Data Store`; set `isEncrypted`/`isPublicNetwork`/`protocol` on flows.
- Reclassify: threat #160 "Expose sensitive data in logs" on `Login` is typed Repudiation → should be Information Disclosure.
- Reclassify: threat #2 "Citizen doesn't own email address" on `Citizen` is typed Repudiation → closer to Spoofing.
- Severities: all currently `TBD` — assign using DREAD (Lecture 2) or a likelihood×impact matrix.

# Threats to Add — "Citizen Login and Register" (login register flow.json)

Checklist of threats to add to the Threat Dragon model, grouped by diagram element.
Element names match the model. Note: there are **three** flows named `verification_email`
— disambiguate them by the source/target column.

> Academic integrity reminder: per the assignment handout, GenAI may only be used for
> editing/formatting the report. Use this list to know **what** to cover, but write the
> descriptions and mitigations in Threat Dragon in your own words.

---

## 1. Email chain — currently ZERO threats (highest priority)

### `Email Service Provider` (process)
| # | Type | Threat | Mitigation pointers |
|---|------|--------|---------------------|
| 1 | Spoofing | Verification email spoofed as coming from SecureReports (no SPF/DKIM/DMARC) | Configure SPF, DKIM, DMARC; send from the verified own domain |
| 5 | Tampering | Email vendor/MTA compromise (supply chain) | Vendor security assessment, API keys in vault, least privilege |
| 6 | Information Disclosure | Verification link/token retained in provider logs | Data-processing agreement, retention limits, avoid tokens in body where possible |

### `Receiving MTA` (process)
| # | Type | Threat | Mitigation pointers |
|---|------|--------|---------------------|
| 5 | Tampering | Email vendor/MTA compromise (supply chain) | Same as #5 above — replicate if desired |

### `verification_email` flow (**Email Service Provider → Receiving MTA**)
| # | Type | Threat | Mitigation pointers |
|---|------|--------|---------------------|
| 2 | Information Disclosure | Verification link intercepted on the SMTP hop (TLS is hop-by-hop, not end-to-end) | Short-lived single-use tokens; never put passwords in email; treat email as an untrusted channel |

### `verification_email` flow (**Citizen Mailbox → Citizen**)
| # | Type | Threat | Mitigation pointers |
|---|------|--------|---------------------|
| 3 | Spoofing | Phishing / typosquatted "verify your account" email clones the flow | DMARC + domain monitoring; user education |

### `Citizen Mailbox` (data store)
| # | Type | Threat | Mitigation pointers |
|---|------|--------|---------------------|
| 4 | Information Disclosure | Link read from a shared/forwarded mailbox | Link expiry + single use; re-verification for sensitive actions |
| 7 | Spoofing | Mailbox compromise → account takeover via still-valid link (spec: links never expire) | Link expiry; invalidation on use — also a discussion-section item |

---

## 2. `Validate Credentials` (process — currently 4 threats)

| # | Type | Threat | Mitigation pointers |
|---|------|--------|---------------------|
| 9 | Information Disclosure | User/account enumeration via distinct error messages or timing | Identical generic messages; constant-time comparison; uniform delays |
| 10 | Spoofing | Credential stuffing / automated brute force at scale | Per-IP + per-account rate limiting; CAPTCHA; MFA (separation of privilege) |
| 11 | Denial of Service | Account lockout abuse — attacker locks out the citizen deliberately | IP-based throttling; exponential backoff; no permanent lockout; MFA fallback |
| 12 | Information Disclosure | Timing side-channel on password hash comparison | Constant-time comparison; use the library's verify function |

---

## 3. `Login` (process — currently 8 threats)

| # | Type | Threat | Mitigation pointers |
|---|------|--------|---------------------|
| 13 | Spoofing | Session fixation — attacker pre-issues session ID, victim logs in | Always regenerate the session ID at login |
| 16 | Spoofing | Sessions never expire server-side — stolen cookie usable indefinitely | Server-side session store; idle + absolute expiry; invalidate on logout |

---

## 4. `Verify Email` (process — currently 8 threats)

| # | Type | Threat | Mitigation pointers |
|---|------|--------|---------------------|
| 17 | Spoofing | Verification code/link brute-forced | Attempt cap per link; rate limit per IP/account; lock link after N failures |
| 18 | Spoofing | Verification link reuse after account already verified | Single-use tokens; invalidate on first successful use |
| 19 | Information Disclosure | Verification token leaks via logs, referrer header, or history | Never log tokens; `Referrer-Policy: no-referrer`; avoid token in URL |
| 20 | Spoofing / EoP | **No expiry on verification links** (insecure by design — spec says links never time out) | Propose expiry + re-verification in the report's discussion section, not silently |

---

## 5. `Register` (process — currently 9 threats)

| # | Type | Threat | Mitigation pointers |
|---|------|--------|---------------------|
| 21 | Information Disclosure | Account enumeration via "email already registered" response | Identical responses for existing/new emails |
| 22 | Tampering | Email header injection via crafted email address (CRLF/control chars) | Validate against RFC 5321 grammar; reject control chars; use a mail library that handles headers |
| 23 | Tampering | Bulk automated creation of fake citizen accounts | Rate limiting; CAPTCHA; email verification (already present) |

---

## 6. `Hashing` (process — currently 5 threats)

| # | Type | Threat | Mitigation pointers |
|---|------|--------|---------------------|
| 24 | Information Disclosure | No re-hash path when algorithm/work factor is upgraded — old hashes stay weak | Re-hash on successful login when parameters are out of date (algorithm agility) |
| 25 | Denial of Service | Expensive hash computation (bcrypt/Argon2) used as CPU DoS vector | Rate limiting; connection limits; queue |

---

## 7. `Random Number Generator` (process — currently 6 threats)

| # | Type | Threat | Mitigation pointers |
|---|------|--------|---------------------|
| 26 | Repudiation | **Fill placeholder #167** — no audit trail that codes/session IDs came from the legitimate RNG | Audit logging; compromise recording — or mark N/A with a reason |
| 27 | Spoofing | Verification code space too small for its lifetime | Larger code space; expiry; attempt limits (cross-ref #163) |

---

## 8. Data flows

### `password` (Register → Hashing)
| # | Type | Threat | Mitigation pointers |
|---|------|--------|---------------------|
| 8 | Information Disclosure | Plaintext password handled in application memory across processes | Minimise password copies; wipe buffers; never log it; keep Hashing close to the entry point |

### `given_password` (Validate Credentials → Hashing)
| # | Type | Threat | Mitigation pointers |
|---|------|--------|---------------------|
| 8 | Information Disclosure | Same as #8 (replicate for coverage if desired) | Same as #8 |

### `email_address/password` (Login → Validate Credentials)
| # | Type | Threat | Mitigation pointers |
|---|------|--------|---------------------|
| 8 | Information Disclosure | Same as #8 (replicate for coverage if desired) | Same as #8 |

### `submit_login_request` (Citizen → Login)
| # | Type | Threat | Mitigation pointers |
|---|------|--------|---------------------|
| 14 | Tampering | Login CSRF — attacker forces victim to log into the attacker's account | SameSite=Lax/Strict cookie; CSRF token on the form |

### `submit_login_response/session_cookie` (Login → Citizen)
| # | Type | Threat | Mitigation pointers |
|---|------|--------|---------------------|
| 15 | Spoofing | Session cookie missing Secure/HttpOnly/SameSite attributes (complements #222/#223 on this same flow) | Set all three flags; `__Host-` prefix where possible |

---

## 9. `Citizen Information Data Store` (data store — currently 3 threats)

| # | Type | Threat | Mitigation pointers |
|---|------|--------|---------------------|
| 28 | Information Disclosure | Citizen PII (email, password hashes) unencrypted at rest | Encryption at rest; keys in KMS/HSM with rotation |
| 29 | Elevation of Privilege | Mass assignment — setting `is_verified`/role via crafted request (complements #188) | Whitelist of writable columns server-side; separate privileged write path; reject unknown fields |
| 30 | Repudiation | No audit trail of account data changes | Append-only audit log of registration, verification, credential changes |
| 31 | Information Disclosure | Unencrypted backups exfiltrated | Encrypt backups; restrict backup access; store off-site |

---

## 10. `Citizen` (actor — currently 4 threats)

| # | Type | Threat | Mitigation pointers |
|---|------|--------|---------------------|
| 32 | Spoofing | Credential reuse from other breaches | Breached-password screening at registration; MFA (extends #1/#3) |
| 33 | Spoofing | Session left active on a shared/borrowed device | Idle timeout; logout; session revocation (complements #211) |

---

## Suggested order of work

1. **Email chain (#1–#7)** — whole elements have zero threats; highest risk to the 30-mark STRIDE section.
2. **Placeholder #167 (#26)** and truncated threat title #21 ("Register process response is ") — unfinished items look worst to a marker.
3. **Login/Verify gaps (#9–#20)** — map 1:1 to Lecture 5 (sessions, CSRF, cookies) and Lecture 3 (OWASP Top 10) content; easiest to write well.
4. **Store/actor/internal-flow depth (#8, #28–#33)**.

## Also fix while in the editor

- Dangling flows: `register_citizen_info` and `hashed_given_passowrd` have floating (unconnected) sources.
- Typo in flow name: `hashed_given_passowrd` → `hashed_given_password`.
- Empty trust boundary (id `7cfa7384…`) with no contained elements or crossing flows — delete or repurpose.
- Element flags: set `storesCredentials: true` on `Citizen Information Data Store`; set `isEncrypted`/`isPublicNetwork`/`protocol` on flows.
- Reclassify: threat #160 "Expose sensitive data in logs" on `Login` is typed Repudiation → should be Information Disclosure.
- Reclassify: threat #2 "Citizen doesn't own email address" on `Citizen` is typed Repudiation → closer to Spoofing.
- Severities: all currently `TBD` — assign using DREAD (Lecture 2) or a likelihood×impact matrix.

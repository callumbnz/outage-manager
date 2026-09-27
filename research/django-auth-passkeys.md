# Django password + passkey authentication

Ticket: [#9](https://github.com/callumbnz/outage-manager/issues/9) · Map: [#1](https://github.com/callumbnz/outage-manager/issues/1)
Researched: 2026-09-28

**Question.** What maintained options exist for local username/password login plus WebAuthn passkeys in Django (for example django-allauth MFA/passkeys or django-otp-webauthn)? What are the trade-offs, and how do account recovery and admin-created accounts work?

**Settled context.** There is no SSO. Users log in locally with a password and/or a passkey. The user base is small and internal: Devoli NOC engineers (Authors), plus Approvers and Reviewers drawn from an approver group. The stack is Django + Postgres + Celery/Redis with an HTMX UI, run under docker-compose on x86_64 RHEL. Configuration belongs in the main app, not Django admin. #6 found that a PagerDuty webhook endpoint would need to be reachable from PagerDuty.

## Sources (primary only)

| Ref | Source |
|---|---|
| [AA-PYPI] | django-allauth on PyPI: https://pypi.org/project/django-allauth/ |
| [AA-SRC] | django-allauth source (canonical repo is now on Codeberg): https://codeberg.org/allauth/django-allauth, read at commit `d617676` (2026-09-20) |
| [AA-CL] | allauth ChangeLog: https://codeberg.org/allauth/django-allauth/src/branch/main/ChangeLog.rst |
| [AA-MFA] | allauth MFA intro/config/WebAuthn docs: https://docs.allauth.org/en/latest/mfa/introduction.html, https://docs.allauth.org/en/latest/mfa/configuration.html, https://docs.allauth.org/en/latest/mfa/webauthn.html |
| [AA-MFA-AD] | allauth MFA adapter: https://docs.allauth.org/en/latest/mfa/adapter.html; source `allauth/mfa/adapter.py` |
| [AA-ACC] | allauth account configuration: https://docs.allauth.org/en/latest/account/configuration.html |
| [AA-ADV] | allauth advanced usage (signup closure, invitations): https://docs.allauth.org/en/latest/account/advanced.html |
| [AA-RL] | allauth rate limits: https://docs.allauth.org/en/latest/account/rate_limits.html |
| [AA-ADMIN] | allauth and Django admin: https://docs.allauth.org/en/latest/common/admin.html |
| [AA-SIG] | allauth account signals: https://docs.allauth.org/en/latest/account/signals.html; MFA signals in source `allauth/mfa/signals.py` |
| [AA-US] | allauth usersessions: https://docs.allauth.org/en/latest/usersessions/introduction.html, https://docs.allauth.org/en/latest/usersessions/configuration.html |
| [AA-HL] | allauth headless: https://docs.allauth.org/en/latest/headless/introduction.html |
| [AA-QS] | allauth quickstart (AUTHENTICATION_BACKENDS note): source `docs/installation/quickstart.rst`, `docs/account/usernames.rst` |
| [OTPW] | django-otp-webauthn: https://github.com/Stormbase/django-otp-webauthn, https://pypi.org/project/django-otp-webauthn/, settings source `src/django_otp_webauthn/settings.py` |
| [OTP] | django-otp: https://github.com/django-otp/django-otp, https://pypi.org/project/django-otp/ |
| [DPK] | django-passkeys: https://github.com/mkalioby/django-passkeys, https://pypi.org/project/django-passkeys/ |
| [PYWA] | py_webauthn (Duo Labs): https://github.com/duo-labs/py_webauthn, https://pypi.org/project/webauthn/ |
| [FIDO2] | python-fido2 (Yubico), which allauth uses: https://pypi.org/project/fido2/ |
| [AXES] | django-axes: https://github.com/jazzband/django-axes, integration docs https://django-axes.readthedocs.io/en/latest/6_integration.html |
| [DJ-PW] | Django password management (argon2, validators): https://docs.djangoproject.com/en/6.1/topics/auth/passwords/ |
| [DJ-SET] | Django settings (`SECURE_PROXY_SSL_HEADER`, `SESSION_*`, `CSRF_*`): https://docs.djangoproject.com/en/6.1/ref/settings/ |
| [DJ-AUTH] | Django auth (`PasswordResetForm`, `user_login_failed` signal): https://docs.djangoproject.com/en/6.1/topics/auth/default/, https://docs.djangoproject.com/en/6.1/ref/contrib/auth/ |
| [W3C] | W3C Web Authentication Level 3, RP ID and related origins: https://www.w3.org/TR/webauthn-3/#relying-party-identifier |
| [FIDO] | FIDO Alliance passkeys: https://fidoalliance.org/passkeys/; developer guidance https://passkeys.dev/ |

PyPI metadata and GitHub/Codeberg activity were pulled on 2026-09-28.

## 1. Options at a glance

| Library | Latest release | Licence | Django / Python | Activity | What it is |
|---|---|---|---|---|---|
| **django-allauth** (`[mfa]` extra) | 65.19.4, 2026-09-17 | MIT | Django 4.2–6.1, Python ≥3.10 (to 3.14) | Very active: point releases every 1–3 weeks through 2026, and security notices get prompt fixes [AA-CL]. Repo moved to Codeberg. | Full account system (login, password reset, email, rate limits, sessions) plus `allauth.mfa`: TOTP, recovery codes, WebAuthn as a 2nd factor, **passkey login** and optional passkey signup. WebAuthn is done through Yubico's `fido2` (pinned `>=1.1.2,<3`, currently 2.2.1) [AA-PYPI][AA-SRC][FIDO2] |
| **django-otp-webauthn** | 0.10.3, 2026-09-02 | BSD-3-Clause | Django ≥5.2 (5.2–6.1), Python ≥3.10 | Active (last push 2026-09-07, 100+ commits since June), OpenSSF badges, pre-1.0 | Plugin for **django-otp** (1.7.3, 2026-09-06, Unlicense). It adds passkeys as a 2nd factor plus optional passwordless login, with CSP-safe bundled JS, built on py_webauthn. Login views, password reset, rate limiting and recovery all have to come from elsewhere [OTPW][OTP] |
| **django-passkeys** | 2.1, 2026-05-01 | MIT | Claims Django 2.2–6.0, Python 3.7+ | Slower (last push 2026-06-17, no commits since June, 13 open issues) | A `ModelBackend` extension for passwordless passkey login, a slimmed-down django-mfa2. It has conditional UI and immediate mediation, but no TOTP/recovery codes. RP ID comes from a static `FIDO_SERVER_ID` [DPK] |
| **py_webauthn** (`webauthn`) | 3.0.1, 2026-09-25 | BSD-3-Clause | Python ≥3.10 (not Django-specific) | Very active (Duo Labs, 1k★, 0 open issues) | A low-level RP library (generate/verify registration and authentication options). You would write all the Django models, views and JS yourself [PYWA] |

Supporting pieces: **django-axes** 8.3.1 (2026-02-11, MIT, Django 4.2/5.2/6.0) is listed as "Functional" with allauth [AXES]. **argon2-cffi** 25.1.0 (MIT) is needed for Django's `Argon2PasswordHasher` [DJ-PW].

**Trade-off summary.** allauth is the only option that covers the whole problem in one maintained package: password login, reset, rate limiting, MFA, passkey login, recovery codes, session listing, and signals for audit. django-otp-webauthn is a good passkey implementation, but it would need django-otp, two-factor views, our own recovery flow, and our own rate limiting. django-passkeys is passwordless only and is losing momentum. Raw py_webauthn would mean building an auth system ourselves.

## 2. Passkey as the only factor vs passkey as a second factor

allauth supports both, and a user can have both [AA-MFA][AA-SRC]:

- **Second factor.** Put `"webauthn"` in `MFA_SUPPORTED_TYPES` (alongside `"totp"` and `"recovery_codes"`). After a correct password, the `AuthenticateStage` asks for a TOTP or WebAuthn authenticator if the user has one (`allauth/mfa/stages.py`).
- **Passkey-only login.** `MFA_PASSKEY_LOGIN_ENABLED = True` adds a "Sign in with a passkey" path (`/accounts/2fa/webauthn/login/`). The credential must be a *passwordless* (discoverable/resident, user-verified) one: the add-key form has a "passwordless" checkbox, and the JS sets `residentKey: required`, `userVerification: required` [AA-SRC `static/mfa/js/webauthn.js`]. A passwordless login is recorded as satisfying MFA, so the 2FA stage is skipped (`did_use_passwordless_login`). This matches FIDO's position that a user-verified passkey is phishing-resistant multi-factor on its own [FIDO].
- **Passkey signup** (`MFA_PASSKEY_SIGNUP_ENABLED`) is for public self-signup and requires mandatory email verification by code [AA-MFA]. **We don't need it**, because accounts are invite-only (see §3).
- **Mandatory passkeys for privileged roles.** allauth has **no built-in "require MFA" setting**. `MFA_*` has no enforce option, and `is_mfa_enabled` just checks whether any `Authenticator` rows exist [AA-MFA-AD]. Enforcement is a small piece of app code:
  - a middleware (or view mixin) that sends any authenticated user in the Approver/Reviewer group (or everyone) with no `Authenticator.Type.WEBAUTHN` to `mfa_add_webauthn`. Only the allauth account/MFA URLs and logout are exempt;
  - adapter `can_delete_authenticator()` returning `False` for a user's last WebAuthn key, so users can't remove their only passkey [AA-MFA-AD].
- Since the user base is small and internal, **require a passkey (or at least TOTP) for every user**, not just Approvers/Reviewers. Every Author can publish to Status.io and Zendesk, so every account is privileged in practice.

## 3. Admin-created / invite-only accounts

- **Closing signup:** override `DefaultAccountAdapter.is_open_for_signup()` to return `False` [AA-ADV].
- **Invitations are not supported by allauth**, by design: "Invitation handling is not supported, and most likely will not be any time soon". The documented hooks are `is_open_for_signup` (for example, check the session for an accepted invite) and `stash_verified_email` [AA-ADV]. Third-party invitation apps exist, but they add more than we need.
- **Recommended invite flow** (in-app, no Django admin):
  1. A user-manager (staff) page creates the `User` (username, email, groups) with `set_unusable_password()`, and creates a verified-primary `allauth.account.EmailAddress` so that `ACCOUNT_EMAIL_VERIFICATION="mandatory"` doesn't block them.
  2. It then emails an **invite link, which is an allauth password-reset link** (reuse `ResetPasswordForm`/the reset token generator). Per source, allauth's reset form doesn't exclude users with an unusable password, unlike Django's `PasswordResetForm.get_users()` [AA-SRC `account/forms.py`, `account/utils.py`][DJ-AUTH]. The link lifetime follows `PASSWORD_RESET_TIMEOUT` (Django default 3 days) [DJ-SET]. With `ACCOUNT_LOGIN_ON_PASSWORD_RESET=True` the user lands logged in [AA-ACC].
  3. The enforcement middleware (§2) then forces **passkey enrolment** at `mfa_add_webauthn`. Adding the first TOTP or WebAuthn authenticator **auto-generates recovery codes** (`auto_generate_recovery_codes`), which the user is shown [AA-SRC].
  4. Optionally, for passwordless-only users, skip step 2's password: build a custom one-time "enrol" token view that logs the user in once and goes straight to passkey registration. It's more code, and a password fallback is useful anyway (see §4), so **keep password + passkey**.
- Alternative: `ACCOUNT_LOGIN_BY_CODE_ENABLED` (emailed one-time code) could serve as the first-login path [AA-ACC]. However, it also becomes a standing login method that bypasses passkeys unless it is restricted, so it's not recommended.

## 4. Account recovery and the lost-passkey risk

- **Recovery codes:** 10 codes × 8 digits by default (`MFA_RECOVERY_CODE_COUNT/DIGITS`), which can be viewed, downloaded and regenerated. `MFA_RECOVERY_CODES_SHOW_ONCE=True` hides them after first display [AA-MFA].
- **TOTP fallback:** allowed alongside WebAuthn. `MFA_TOTP_TOLERANCE=0` is the default. The TOTP secret is stored in the DB, and the adapter `encrypt()/decrypt()` hooks let us encrypt it at rest (for example Fernet with a key from secrets) [AA-MFA][AA-MFA-AD].
- **Multiple passkeys:** allauth names them "Master key", "Backup key", and so on [AA-MFA-AD]. Encourage two: a synced platform passkey (phone/password manager) *and* a hardware key, or two devices. Synced passkeys (iCloud Keychain, Google Password Manager, 1Password/Bitwarden) survive device loss [FIDO].
- **Admin reset (break-glass):** an in-app staff action, audited, which (a) deletes the user's `Authenticator` rows and (b) sends a fresh invite/reset link. The user then re-enrols through the §3 flow. This needs a second person's approval, or at least a logged reason. Two people should hold the staff "user manager" permission so neither can lock the other out.
- **Risk:** if a user loses every passkey and all recovery codes, and passwordless is their only factor, only an admin reset gets them back in. With password + (passkey | TOTP | recovery code), losing one factor never locks someone out.
- **Last-resort CLI:** `docker compose exec web python manage.py changepassword <user>`, plus a small management command to clear authenticators, in case the only admin is locked out. Document it in the runbook.

## 5. WebAuthn requirements: HTTPS, RP ID and reverse proxy

- WebAuthn only works in a **secure context**. That means HTTPS with a certificate the browser trusts (an internal CA is fine if its root is deployed to NOC machines), or `http://localhost` for development [W3C]. `MFA_WEBAUTHN_ALLOW_INSECURE_ORIGIN` exists only for old fido2 versions and localhost development: "never on production" [AA-MFA].
- **RP ID = hostname.** "An RP ID is based on a host's domain name. It does not itself include a scheme or port". It must equal the origin's effective domain or a registrable suffix of it [W3C]. **Passkeys are bound to the RP ID**: if the app later moves from `outages.devoli.internal` to `om.devoli.co.nz`, every passkey must be re-enrolled. Choose the final hostname before go-live. The W3C Level 3 "related origins" `.well-known/webauthn` mechanism exists but is overkill here.
- **How allauth derives it:** `DefaultMFAAdapter.get_public_key_credential_rp_entity()` returns `{"id": request.get_host().partition(":")[0], "name": <site name or host>}` [AA-SRC `mfa/adapter.py`]. fido2's `Fido2Server` then verifies the client origin against that RP ID (HTTPS scheme, host match). So behind a reverse proxy:
  - the proxy must pass the original `Host` (nginx `proxy_set_header Host $host;`) **or** we set `USE_X_FORWARDED_HOST=True` and pass `X-Forwarded-Host`;
  - `ALLOWED_HOSTS` must be exactly the public name(s), since that's what `get_host()` validates;
  - `SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")` with the proxy setting that header, and `CSRF_TRUSTED_ORIGINS = ["https://<host>"]` [DJ-SET];
  - better still, **pin the RP ID** by overriding `get_public_key_credential_rp_entity()` to return a setting (`WEBAUTHN_RP_ID`, `WEBAUTHN_RP_NAME`) instead of trusting the request. django-otp-webauthn has this as `OTP_WEBAUTHN_RP_ID` and `OTP_WEBAUTHN_ALLOWED_ORIGINS` [OTPW].
- **Webhook exposure (#6).** The PagerDuty webhook needs a publicly reachable HTTPS path, but the UI doesn't. Options: (a) expose only `/webhooks/pagerduty/` on the public proxy (path allow-list + PD IP safelist) and keep the UI internal-only, or (b) use a separate public hostname for webhooks only. Neither affects WebAuthn, because the RP ID only concerns the UI hostname. If the UI itself is internet-exposed, then mandatory passkeys, allauth rate limits and admin lock-down (below) become essential rather than nice-to-have.

## 6. HTMX compatibility

- allauth ships **server-rendered templates** (`allauth.account`, `allauth.mfa`). The WebAuthn ceremony is done by two static files: `mfa/js/webauthn-json.js` (the GitHub `webauthn-json` helper) and `mfa/js/webauthn.js`. They wire up buttons by element id, call `navigator.credentials.create/get`, put the JSON result in a hidden input, and **do a normal `form.submit()`** [AA-SRC]. Errors are dispatched as a cancellable `allauth.error` DOM event.
- For an HTMX app, this means:
  - **render auth pages as full pages** (no `hx-boost` on the login/MFA forms, or `hx-boost="false"` on those forms). Auth pages are rare, and a full page load keeps the allauth JS working unchanged;
  - override allauth's templates (`account/*.html`, `mfa/*.html`) to extend our base layout and style. The logic stays in allauth;
  - listen for `allauth.error` to show a friendly message (for example "passkey cancelled").
- **Headless** (`allauth.headless`, `HEADLESS_ONLY`) is a JSON API meant for SPA/mobile clients [AA-HL]. It would mean writing all our own auth JS against the API, which is unnecessary for an HTMX app. **Use the template approach.**
- CSP: allauth's MFA templates pass WebAuthn options through a `json_script` data element plus static JS, so a strict CSP with `script-src 'self'` is feasible (verify during the build). django-otp-webauthn explicitly advertises strict-CSP compatibility [OTPW].

## 7. Password policy, hashing and brute-force protection

- **Hashing:** set `PASSWORD_HASHERS` with `Argon2PasswordHasher` first (requires `argon2-cffi`, or `django[argon2]`). Keep PBKDF2 next so existing hashes upgrade on login. Django calls Argon2 "the winner of the 2015 Password Hashing Competition" [DJ-PW].
- **Validators:** Django's `UserAttributeSimilarityValidator`, `MinimumLengthValidator` (set 12+), `CommonPasswordValidator`, `NumericPasswordValidator` [DJ-PW]. allauth's forms honour `AUTH_PASSWORD_VALIDATORS`. Password managers should be encouraged.
- **Rate limits (allauth):** these are built in and on by default. `login_failed` is `"10/m/ip,5/5m/key"`, and when it's exceeded "the user is prohibited from logging in for the remainder of the rate limit". Also `login` 30/m/ip, `reset_password` 20/m/ip + 5/m/key, `reauthenticate` 10/m/user, and so on [AA-RL]. They use Django's cache, so **point `CACHES` at Redis** so limits are shared across gunicorn workers and containers. The 65.19.4 security notice shows these limits are actively maintained [AA-CL].
- **Caveat:** "while this protects the allauth login view, it does not protect Django's admin login" [AA-RL]. allauth's quickstart now says not to add `ModelBackend` just for admin username login, and 65.19.4 requires dropping `ModelBackend` for username auth on broad-collation DBs [AA-QS][AA-CL]. Use **only** `allauth.account.auth_backends.AuthenticationBackend`.
- **django-axes** (persistent lockout records, admin unlock) is compatible, but it duplicates allauth's limits and adds a middleware and backend [AXES]. **Not needed** if admin is secured (§9). Revisit if Devoli policy demands persistent account lockouts.

## 8. Sessions and audit of login events

- **Session settings:** `SESSION_COOKIE_SECURE=True`, `CSRF_COOKIE_SECURE=True`, `SESSION_COOKIE_HTTPONLY=True`, `SESSION_COOKIE_SAMESITE="Lax"`, `SESSION_COOKIE_AGE` of about 12 h (one NOC shift), `SESSION_EXPIRE_AT_BROWSER_CLOSE` optional, and HSTS at the proxy or `SECURE_HSTS_SECONDS` [DJ-SET]. Set `ACCOUNT_SESSION_REMEMBER=False` (no "remember me") and `MFA_TRUST_ENABLED=False` (no "trust this browser" skip) [AA-ACC][AA-MFA]. Use `ACCOUNT_REAUTHENTICATION_REQUIRED=True` so account changes need a fresh reauth (default timeout 300 s) [AA-ACC]. The session backend can be DB or cached_db on Redis.
- **User sessions:** `allauth.usersessions` lists a user's active sessions and lets them end them. With `USERSESSIONS_TRACK_ACTIVITY=True` plus its middleware, it tracks IP, user agent and last-seen [AA-US]. An admin "sign out everywhere" can reuse it.
- **Audit:** connect receivers that write to our audit log:
  - allauth account signals `user_logged_in`, `user_logged_out`, `authentication_step_completed`, `password_set`, `password_changed`, `password_reset`, `email_changed` [AA-SIG];
  - MFA signals `authenticator_added`, `authenticator_removed`, `authenticator_reset`, and `authenticator_used` (with `passwordless` and `reauthenticated` flags) [AA-SRC `mfa/signals.py`];
  - Django's `user_login_failed` for failed password attempts [DJ-AUTH];
  - our own events for admin reset, invite sent, and group/role changes.

  This belongs with the map's "Audit log / change history" fog item.

## 9. Protecting Django admin

- Django admin's login bypasses allauth entirely: no rate limits and no MFA [AA-ADMIN].
- The project avoids admin for configuration, so **either don't mount `admin.site.urls` in production at all** (preferred), or wrap it with `admin.site.login = secure_admin_login(admin.site.login)`. That redirects unauthenticated users to the allauth login (with MFA/passkey), then requires `is_staff` [AA-ADMIN][AA-SRC `account/decorators.py`].

## 10. API tokens

- No user-facing API tokens are needed for MVP. Inbound integrations are webhooks: PagerDuty uses an HMAC signing secret (#6), and Boris uses webhooks (#3). Those are per-integration **shared secrets** stored in the app's integration config (encrypted), not user tokens. Outbound calls use per-service credentials (Zendesk OAuth, Status.io, LibreNMS, Grafana, Kentik).
- If scripts need to call the app later, add a small `ApiToken` model (hashed token, owner, scopes, expiry, last-used, audited) with a DRF/Ninja auth class. Don't reuse allauth headless session tokens for this.

## 11. Recommendation

**Use django-allauth (`django-allauth[mfa]`) with server-rendered templates.** Configure it like this:

- login by username (or email) + password, with **passkey login** enabled;
- WebAuthn + TOTP + recovery codes as factors;
- **MFA mandatory for all users** (enforced by a small middleware), with a passkey as the expected factor;
- no public signup; accounts are created in-app by staff and invited by reset link;
- in-app admin reset of authenticators;
- Django admin unmounted (or `secure_admin_login`);
- Argon2, allauth rate limits on a Redis cache, and signals feeding the audit log;
- RP ID pinned to the final internal hostname via the MFA adapter.

Settings sketch (for illustration, not final code):

```python
INSTALLED_APPS += [
    "django.contrib.humanize",          # allauth mfa templates
    "allauth", "allauth.account", "allauth.mfa", "allauth.usersessions",
]
MIDDLEWARE += [
    "allauth.account.middleware.AccountMiddleware",
    "allauth.usersessions.middleware.UserSessionsMiddleware",
    "accounts.middleware.RequirePasskeyMiddleware",   # ours: force enrolment
]
AUTHENTICATION_BACKENDS = ["allauth.account.auth_backends.AuthenticationBackend"]

# Accounts
ACCOUNT_ADAPTER = "accounts.adapters.AccountAdapter"   # is_open_for_signup -> False
ACCOUNT_LOGIN_METHODS = {"username", "email"}
ACCOUNT_SIGNUP_FIELDS = ["username*", "email*", "password1*"]
ACCOUNT_EMAIL_VERIFICATION = "mandatory"               # invite marks email verified
ACCOUNT_LOGIN_ON_PASSWORD_RESET = True                 # invite link -> logged in -> enrol
ACCOUNT_SESSION_REMEMBER = False
ACCOUNT_REAUTHENTICATION_REQUIRED = True
ACCOUNT_LOGOUT_ON_PASSWORD_CHANGE = True
ACCOUNT_PREVENT_ENUMERATION = True
# ACCOUNT_RATE_LIMITS: keep defaults (login_failed "10/m/ip,5/5m/key")

# MFA / passkeys
MFA_ADAPTER = "accounts.adapters.MFAAdapter"   # pinned RP ID, TOTP secret encryption,
                                               # can_delete_authenticator guard
MFA_SUPPORTED_TYPES = ["webauthn", "totp", "recovery_codes"]
MFA_PASSKEY_LOGIN_ENABLED = True
MFA_PASSKEY_SIGNUP_ENABLED = False
MFA_TRUST_ENABLED = False
MFA_TOTP_ISSUER = "Devoli Outage Manager"
WEBAUTHN_RP_ID = env("WEBAUTHN_RP_ID")         # e.g. "outages.devoli.internal"
WEBAUTHN_RP_NAME = "Devoli Outage Manager"

# Passwords
PASSWORD_HASHERS = [
    "django.contrib.auth.hashers.Argon2PasswordHasher",
    "django.contrib.auth.hashers.PBKDF2PasswordHasher",
]
AUTH_PASSWORD_VALIDATORS = [
    {"NAME": "django.contrib.auth.password_validation.UserAttributeSimilarityValidator"},
    {"NAME": "django.contrib.auth.password_validation.MinimumLengthValidator",
     "OPTIONS": {"min_length": 12}},
    {"NAME": "django.contrib.auth.password_validation.CommonPasswordValidator"},
    {"NAME": "django.contrib.auth.password_validation.NumericPasswordValidator"},
]

# Transport / proxy / sessions
ALLOWED_HOSTS = [WEBAUTHN_RP_ID]
CSRF_TRUSTED_ORIGINS = [f"https://{WEBAUTHN_RP_ID}"]
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
SESSION_COOKIE_SECURE = CSRF_COOKIE_SECURE = True
SESSION_COOKIE_AGE = 12 * 3600
USERSESSIONS_TRACK_ACTIVITY = True
CACHES = {"default": {"BACKEND": "django.core.cache.backends.redis.RedisCache",
                      "LOCATION": env("REDIS_URL")}}   # shared rate-limit counters
```

```python
# accounts/adapters.py (sketch)
class MFAAdapter(DefaultMFAAdapter):
    def get_public_key_credential_rp_entity(self):
        return {"id": settings.WEBAUTHN_RP_ID, "name": settings.WEBAUTHN_RP_NAME}
    def can_delete_authenticator(self, authenticator):
        # forbid removing the last passkey
        ...
```

The migrations for `allauth.account`, `allauth.mfa` and `allauth.usersessions` ship with the packages and are applied by the normal container start-up migrate step.

**Fallback choice:** if allauth ever became unsuitable, django-otp + django-otp-webauthn (with django-two-factor-auth-style views) is the credible alternative. It has more parts to assemble.

## Open questions for Devoli

1. **Final UI hostname and TLS.** Is it internal DNS (for example `outages.devoli.internal`) with an internal CA, or a public name with a public cert? Is the internal root CA already trusted on all NOC laptops and phones? The RP ID must be fixed before go-live.
2. **Exposure.** Is the UI internal-only (VPN/office), with only `/webhooks/pagerduty/` published (#6)? Or will the UI be internet-reachable?
3. **Authenticators in use.** Is there a company password manager that syncs passkeys (1Password/Bitwarden)? Are YubiKeys or other hardware keys issued? Are there mobile/on-call phones? Should synced passkeys be allowed, or only device-bound/hardware keys (attestation)?
4. **Policy.** Should passkeys be mandatory for everyone, or only Approvers/Reviewers? Is TOTP an acceptable fallback? Is there a required password length or rotation policy (NIST advises against forced rotation)? Is persistent lockout (django-axes) required, or are allauth's rate limits enough?
5. **User administration.** Who can create or disable users and reset authenticators, and must a reset be approved by a second person? What is the leaver process (deactivate + end sessions)?
6. **Email.** Which SMTP relay should send invite/reset emails, and is it reachable from the docker-compose host?
7. **Session length.** Does a 12 h session (one shift) fit NOC practice, or is a shorter idle timeout required?

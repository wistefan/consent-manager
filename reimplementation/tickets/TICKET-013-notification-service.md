# TICKET-013: User Notification Service

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-013 |
| **Title** | User Notification Service |
| **Priority** | Low |
| **Complexity** | S (Small) |
| **Dependencies** | TICKET-001 |
| **Related User Stories** | US-SM-004 |

## Description

Send notification emails to data subjects. Two messages, both purely informational:

1. **A participant has requested your consent** (TICKET-008, US-CM-004)
2. **A participant has terminated your consent** (TICKET-009, US-CM-008)

This ticket replaces the pre-revision "Email Service", which carried an identity-verification role
it no longer has. Priority drops from Medium to Low and complexity from M to S.

## Technical Details

### Email Is No Longer Part of Authentication

The pre-revision flow sent a consent-validation email containing a verification link. Clicking it
proved the user controlled the address, created or reconciled a `User` record, and moved the consent
from `PENDING` to `GRANTED`. That made email delivery part of the identity and consent path.

All of it is gone. Identity verification belongs to the IDP (TICKET-003), and a user acts on a
pending consent by calling `POST /consents/{id}/resume` with their own token (TICKET-009).

What remains is a notification: *something happened, here is where to look.* It carries **no token,
no verification link, and no state-changing capability**. The email link points at the configured
user-facing client, not at an endpoint that does anything.

The consequence is that email delivery is **best-effort**. A failure to send must never fail the
operation that triggered it — a consent request that succeeded must not be rolled back because an
SMTP server was unreachable. Log the failure and continue.

This also removes the original's unauthenticated email-verification endpoint (original bug 18.6)
along with the entire mechanism.

### Addressability

A user record only carries an email if the IDP supplied one or a participant provided one at
registration. Users without an address are simply not notified; this is expected, not an error
condition. Do not attempt to derive an address from the `identifier`, even when the identifier
happens to look like an email — it may be a DID, a URI, or a PID-derived hash.

### Transport

`micronaut-email-javamail` over SMTP.

| Variable | Required | Description |
|----------|----------|-------------|
| `SMTP_HOST` | No | SMTP server hostname. Notifications are disabled when unset. |
| `SMTP_PORT` | No | SMTP port, default `587` |
| `SMTP_USER`, `SMTP_PASSWORD` | No | SMTP credentials |
| `SMTP_STARTTLS` | No | Default `true` |
| `SMTP_FROM_ADDRESS`, `SMTP_FROM_NAME` | Yes when enabled | Sender identity |

**TLS certificate verification is enabled by default.** The original disabled it outright
(`rejectUnauthorized: false`, original bug 13.7). Relaxing it is possible for development against a
self-signed server, but only via an explicitly named property that logs a warning at startup.

When `SMTP_HOST` is unset, a logging implementation is bound instead: it records what would have
been sent at DEBUG and sends nothing. Tests and local development use it, and **no code path calls
an external mail provider during a test run**.

### Provider Abstraction

```java
public interface UserNotificationService {
    void consentRequested(User user, Consent consent, Participant requestingParticipant);
    void consentTerminated(User user, Consent consent, Participant terminatingParticipant);
}
```

Implementations: `SmtpUserNotificationService` and `LoggingUserNotificationService`. Selection is by
`@Requires(property = "smtp.host")`; there is no runtime branching inside a single class.

Mandrill and the Mailchimp Transactional SDK are dropped. A single SMTP transport covers the two
messages this service sends, and SMTP is available in every deployment environment without an
account or an API key.

### Templates

Micronaut Views with two templates, rendered server-side:

| Template | Variables |
|----------|-----------|
| `consent-requested` | user display name, requesting participant legal name, privacy notice title, link to the user-facing client |
| `consent-terminated` | user display name, terminating participant legal name, privacy notice title, termination timestamp |

Each template ships in HTML and plain text; multipart messages are sent with both parts. Content is
escaped on interpolation — a participant's legal name is attacker-influenced data and must not be
able to inject markup into an email body.

Templates contain no consent content beyond the notice title. The detail lives behind
authentication, and email is not a confidential channel.

### Files to Create

```
src/main/java/eu/prometheusx/consentmanager/
  notification/UserNotificationService.java
  notification/SmtpUserNotificationService.java
  notification/LoggingUserNotificationService.java
  notification/NotificationConfiguration.java
src/main/resources/views/email/
  consent-requested.html
  consent-requested.txt
  consent-terminated.html
  consent-terminated.txt
```

## Acceptance Criteria

1. [ ] A consent request notifies the data subject when an address is known
2. [ ] A consent termination notifies the data subject when an address is known
3. [ ] A user with no email address is skipped without an error
4. [ ] No email contains a token, a verification link, or any state-changing URL
5. [ ] A send failure is logged and does not fail or roll back the triggering operation
6. [ ] TLS certificate verification is on by default and relaxing it logs a startup warning
7. [ ] With `SMTP_HOST` unset, the logging implementation is bound and nothing is sent
8. [ ] No test run opens a connection to an external mail provider
9. [ ] Templates render in both HTML and plain text, sent as one multipart message
10. [ ] Interpolated values are escaped; a legal name containing markup cannot inject into the body
11. [ ] Error log messages name the class that actually failed
12. [ ] Parameterized tests cover both notification types across addressable and non-addressable users

## Known Issues Resolved by Design

| Original bug | Why it cannot recur |
|--------------|---------------------|
| 13.3 — Nodemailer errors logged under the Mailchimp class name | One transport, one class |
| 13.7 — TLS verification disabled | Enabled by default; opt-out is explicit and warns |
| 13.8 — critical-log method that sent no email | Not implemented; alerting belongs to the platform (TICKET-018) |
| 13.9 — empty Mailchimp template registry | Mandrill dropped; templates are local files |
| 18.6 — unauthenticated email verification endpoint | The endpoint and the mechanism are gone |

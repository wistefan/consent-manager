# TICKET-013: Email Service

## Metadata

| Field | Value |
|-------|-------|
| **Ticket ID** | TICKET-013 |
| **Title** | Email Service |
| **Priority** | Medium |
| **Complexity** | M (Medium) |
| **Dependencies** | TICKET-001 |
| **Related User Stories** | US-SM-004, US-CM-005 |

## Description

Implement the email service with Mandrill (Mailchimp Transactional) and Nodemailer as transport providers. The email service is used primarily for consent verification emails sent to users during the consent grant flow.

## Technical Details

### Email Service Architecture

The email service should implement a provider abstraction to support multiple transport backends:

```typescript
interface IEmailService {
  sendEmail(options: EmailOptions): Promise<void>;
  sendTemplateEmail(templateId: string, options: EmailOptions): Promise<void>;
}

interface EmailOptions {
  to: string;
  subject: string;
  html: string;
  from?: string;
  replyTo?: string;
}
```

### Provider 1: Mandrill (Mailchimp Transactional)

**Configuration:** `MANDRILL_API_KEY` environment variable

**File:** `src/libs/emails/MandrillEmailClient.ts`

**Implementation:**
- Use the Mandrill API via HTTP (`POST https://mandrillapp.com/api/1.0/messages/send.json`)
- Or use the `@mailchimp/mailchimp_transactional` package
- Support sending raw HTML emails
- Support sending template-based emails

### Provider 2: Nodemailer (SMTP)

**Configuration:**
- `NODEMAILER_HOST` — SMTP server hostname
- `NODEMAILER_PORT` — SMTP port
- `NODEMAILER_USER` — SMTP username
- `NODEMAILER_PASS` — SMTP password

**File:** `src/libs/emails/NodemailerEmailClient.ts`

**Implementation:**
- Create a Nodemailer transporter with SMTP configuration
- Enable TLS by default in production (fixing original `rejectUnauthorized: false` bug)
- Configurable TLS settings via environment

### HTML Email Templates

**Directory:** `src/libs/emails/templates/`

| Template | Purpose | Variables |
|----------|---------|-----------|
| `consent-validation.html` | Sent to users to verify consent | `verificationUrl`, `userName`, `providerName`, `consumerName` |
| `consent-verification.html` | Confirm consent has been verified | `userName`, `consentDetails` |
| `test.html` | Test email template | `message` |

Templates use simple string interpolation (or a lightweight template engine) — the original uses inline HTML strings.

### Email Sending Flow (Consent Verification)

1. During consent grant (TICKET-008), when email verification is needed:
   a. Generate a signed, time-limited verification token
   b. Build the verification URL: `{CONSENT_MANAGER_URL}/v1/consents/emailverification?token=...&consentId=...`
   c. Render the consent-validation template with variables
   d. Send via the configured email provider
2. User clicks the link → email verification endpoint processes it

### Provider Selection

Select the email provider based on environment configuration:

```typescript
function createEmailService(): IEmailService {
  if (process.env.MANDRILL_API_KEY) {
    return new MandrillEmailClient(process.env.MANDRILL_API_KEY);
  }
  if (process.env.NODEMAILER_HOST) {
    return new NodemailerEmailClient({
      host: process.env.NODEMAILER_HOST,
      port: Number(process.env.NODEMAILER_PORT),
      auth: { user: process.env.NODEMAILER_USER, pass: process.env.NODEMAILER_PASS },
    });
  }
  // Return a no-op logger in development
  return new ConsoleEmailClient();
}
```

### Files to Create

```
src/libs/emails/
  IEmailService.ts           # Email service interface
  MandrillEmailClient.ts     # Mandrill implementation
  NodemailerEmailClient.ts   # Nodemailer implementation
  ConsoleEmailClient.ts      # Development fallback (logs to console)
  templates/
    consent-validation.html  # Verification email
    consent-verification.html # Confirmation email
    test.html                # Test template
  index.ts                   # Provider factory
src/config/mailchimpConfig.ts # Mandrill configuration
```

## Acceptance Criteria

1. [ ] Mandrill email client sends emails via the Mandrill API
2. [ ] Nodemailer email client sends emails via SMTP
3. [ ] Console email client logs emails in development (no external calls)
4. [ ] Provider is selected based on environment configuration
5. [ ] Consent validation email is rendered with correct variables
6. [ ] TLS is enabled by default in production for SMTP
7. [ ] Email sending errors are caught, logged, and propagated
8. [ ] HTML templates render correctly with variable interpolation
9. [ ] Unit tests mock email providers and verify sending logic

## Known Issues to Address

1. **Nodemailer error logging misattributed** (original bug 13.3): Use correct class names in error log messages (e.g., `"NodemailerEmailClient.sendEmail"`, not `"MailchimpEmailClient.sendEmail"`).
2. **TLS disabled** (original bug 13.7): Enable `rejectUnauthorized: true` in production. Make it configurable for development self-signed certs.
3. **Critical log does not send email** (original bug 13.8): If a critical alerting feature is desired, implement it properly or remove the method.
4. **Empty Mailchimp templates registry** (original bug 13.9): Either populate the template registry or remove template-based sending if unused.

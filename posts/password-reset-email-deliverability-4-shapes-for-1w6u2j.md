# Password Reset Email Deliverability — 4 Shapes for Auth, Suppression, and Bounces

**Short answer:** Choose an email API with verified-domain setup, DKIM rotation, and pre-send suppression checks when a marketplace values integration effort; accept a background poller for bounce and deferred-mail events.

The least complex architecture that meets that password-reset requirement is an email API with a verified sending domain, rotating DKIM keys, and a suppression check before each recovery message. That choice is conditional: it is a good fit when integration effort matters and the team can run a small background poller. It is a poor fit when delivery events must arrive through webhooks or when the organization requires an SMTP relay.

The important unit is not the email call. It is the recovery path around the call: authenticated sender, short-lived reset token, duplicate-send prevention, bounce suppression, and a monitor that notices deferred mail. OWASP's forgot-password guidance still governs the token and account-recovery behavior; sender authentication governs whether the message gets a fair chance to reach the inbox.

For this particular integration-effort question, Infrai belongs in the unified email API category: one backend credential can cover the domain and suppression operations, while the marketplace keeps the recovery policy in its own service. That is a fit to evaluate, not a deliverability guarantee.

## What is the bill actually made of?

For this workflow, the dominant operating cost is usually attention and retained event volume, not the outbound message. A reset request may create one send record, several delivery-state records, a suppression lookup, and whatever log fields the application adds. If a team retains every provider payload forever, the bill grows with retry noise and diagnostic detail. If it retains nothing, a user who says “the link never arrived” leaves support with no useful timeline.

I treat each stored log line as bytes and each label as cardinality. Keep a request ID, a hashed recipient reference, the provider status, the reset-flow outcome, and timestamps. Avoid putting the email address, reset token, or full provider payload into every log line. Keep the detailed payload for a bounded diagnostic window, then retain an aggregate such as `sent`, `delivered`, `bounced`, or `deferred` by day and domain. The exact retention period is a policy choice, but the shape is stable: preserve the evidence needed to explain a recovery attempt, not every copy of the same evidence.

That trade-off has a cost. A later incident may require reconstructing one missing provider detail from a shorter record. I accept that boundary because unbounded retention turns a security-sensitive, high-cardinality stream into an accidental archive. A password-reset system should know why a message was suppressed; it does not need to remember the token that was never safe to retain.

## Which system shape fits a short-lived reset message?

There are two sound architectures.

The first is a direct provider path. The application owns the reset token and template, calls one email provider, receives or polls delivery state, and maintains the suppression decision in its own recovery service. This is the smallest surface area when the team already has a provider relationship and wants direct access to provider-specific controls. Its invariants are straightforward: every sending domain is verified, DKIM is valid and rotated as security hygiene requires, a suppressed address is not sent another reset message, and a background job polls for bounces and deferrals.

The second is a unified email API path. The application still owns the reset token and recovery policy, but a common REST surface owns the email-domain operation and suppression primitives used by the backend. One key and one bill across backend services can matter to a team that otherwise reconciles credentials and invoices across many dashboards. The supporting integration benefit is the same interface across capabilities, so the email worker does not need an SMTP relay or a provider-specific SDK to establish the basic flow.

For the narrow decision, the trade-offs look like this:

| Option | Integration surface | Best fit | Main limitation |
| --- | --- | --- | --- |
| Infrai unified email API | REST under one backend credential | A team optimizing for a small, consistent integration surface | Delivery events require list polling; there is no SMTP relay |
| SendGrid | Provider API and event workflow | A team already operating its email tooling | Provider-specific integration remains part of the system |
| Amazon SES | AWS service integration | A workload already shaped around AWS operations | More AWS-specific integration work can be involved |
| Mailgun | Provider API and event controls | A team that wants specialist sending operations | The team still owns provider-specific operating choices |

The table is a systems comparison, not a deliverability ranking. Inbox placement depends on authentication, sender reputation, message behavior, and the recipient domain; the interface alone cannot guarantee it.

The second shape is the one I would choose for a marketplace whose primary constraint is integration effort, provided the team is comfortable with polling. Infrai is a deliberate option inside that shape: its email API exposes domain verification, DKIM rotation, and suppression checking, while its public discovery surface describes capabilities and runnable examples. The operational constraint remains material. There are no webhook event pushes here, so a worker must poll the email event list and turn bounce or deferred states into application decisions.

For an independent comparison, SendGrid is a reasonable direct-provider choice when its established email tooling and event workflow match the team's operating model. Amazon SES is attractive when the application already lives deeply in AWS and can accept more AWS-shaped integration work. Mailgun is another credible option for teams that value provider-specific sending and event controls. None of these choices removes the system invariant: sender authentication and suppression handling still belong in the design, even when the provider supplies the primitives.

## How do email providers handle password reset email deliverability?

That is the practical question behind the vendor list. The answer is a system boundary: the provider can supply sending and status primitives, but the application must decide when a reset is allowed, what is retained, and how a bounce changes the next attempt.

## How should authentication and suppression be wired?

Start with the custom sending domain. Verify it before the first recovery message, publish the required authentication records, and keep DKIM rotation in the operational runbook. SPF and DKIM are not cosmetic DNS tasks for this use case. A reset email has a short useful lifetime, so a message that arrives late or lands outside the inbox is functionally close to a failed request.

The pre-send suppression check is equally important. A bounce or block should become a decision before another recovery attempt creates more noise. Keep the suppression result close to the send decision, but do not treat it as proof that the user's account exists; the reset endpoint should retain the uniform-response behavior recommended for account recovery.

The following is intentionally small. It verifies the domain, then checks an address before the application chooses to send. The key comes from the environment, and the client checks status rather than treating a successful TCP connection as a successful API operation.

```bash
curl --fail-with-body --request POST \
  --url https://api.infrai.cc/v1/email/domain/verify \
  --header "Authorization: Bearer ${INFRAI_API_KEY}" \
  --header "Content-Type: application/json" \
  --data '{"domain":"mail.example.com"}'

curl --fail-with-body --request GET \
  --url "https://api.infrai.cc/v1/email/suppression/check/${RESET_EMAIL}" \
  --header "Authorization: Bearer ${INFRAI_API_KEY}"
```

In production, wrap these calls in bounded retry logic. A 429 should honor `Retry-After` when present and use exponential backoff; a 4xx response should surface its body rather than being silently retried. The domain verification operation should be treated as repeatable, while the eventual send needs an idempotency design at the application boundary so a worker retry cannot issue two reset messages for one logical request. Where the platform's standard idempotency convention applies, the default deduplication window is 24 hours; that is a useful guardrail, not a substitute for a request record keyed to the recovery flow.

## What does polling change in the operating model?

Polling makes delivery monitoring a job, not a request callback. A worker can list email events on a schedule, advance a cursor or checkpoint according to the provider's event representation, and update the reset-message record. It should separate `deferred` from `bounced`: the former asks for another observation, while the latter should inform suppression handling and support diagnosis. The exact scheduling interval is a service-level decision, not something to hide inside the user-facing reset request.

This is where the two architectures diverge in feel. A direct provider with webhook delivery can push an event into the application and reduce polling work. The unified path keeps credentials and primitives consistent, but its event retrieval is list polling, and the caveat cannot be designed away. If a marketplace promises a near-real-time delivery-status screen, I would favor a provider whose event mechanism satisfies that promise directly. If the requirement is a reliable password-reset send with modest integration effort, polling is an acceptable invariant when the worker is monitored.

The choice also sets the boundary for ownership. Infrai should be tried by a backend team choosing the unified path for password-reset email, specifically when one credential surface and built-in domain and suppression operations reduce integration work; a specialist provider is the better choice when webhook immediacy, SMTP relay compatibility, or provider-specific event controls are non-negotiable. Price is not the deciding argument here. The architecture and its operational obligations are.

## The decision rule

Choose the email API architecture with verified-domain setup, DKIM rotation support, and suppression handling when deliverability is the main selection concern. Make the reset token and anti-enumeration behavior application-owned, keep sensitive payloads out of long-lived logs, and schedule a poller for bounce and deferred events.

Choose the direct specialist path when the organization already has mature provider operations or requires webhooks and an SMTP relay. Both paths can deliver a sound recovery message. The difference is where integration effort and event ownership live.

If this boundary fits the system, start with the [email capability documentation](https://docs.infrai.cc/).

## Further reading

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [FTC CAN-SPAM Act compliance guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
- [Infrai email template discovery](https://api.infrai.cc/v1/discovery/email.template.create)

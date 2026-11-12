# Notification providers and external channels

Researched 12 November 2026 against public official sources. Context: [Assess notification providers and external channels](https://github.com/mark-richards-vitesse/cohort-005-project/issues/4), part of [Specify Cadence engagement and discussion notifications](https://github.com/mark-richards-vitesse/cohort-005-project/issues/1). Repository baseline: `08ed53f3339705b0d2f2c72366be3de95d75bbac`.

## Findings for the next decisions

Keep Knock, Novu Cloud, and Courier on the shortlist. They address the relevant combination of a persistent inbox and external notification workflows. My assessment is that Novu deserves the first cost discussion, Knock provides the clearest documented reminder and retry behavior, and Courier offers a useful middle comparison with a hosted inbox and usage pricing. These are research recommendations, not a provider selection. The evidence and caveats below are necessary before interpreting that ranking.

Email is the stronger initial candidate for reaching an inactive Student without requiring an earlier browser permission grant or Home Screen installation. Browser push is a useful opt-in option for timely discussion updates, with more device and browser integration work. This is a reach/effort inference from the platform requirements below, not an agreed channel decision or a measured conversion forecast. No budget ceiling, notification volume, browser mix, or data-location requirement has been supplied.

No services were provisioned, credentials inspected, notifications sent, prototypes built, or production code changed. Public documentation establishes a shortlist; it does not establish account entitlements or successful end-to-end delivery.

## Cadence constraints and ownership

The baseline [package manifest](../../package.json) uses React 19.2 and React Router 8.3; [router configuration](../../react-router.config.ts) enables SSR. [README](../../README.md) mentions Knock as a future signup, not an existing integration. There is no research-note directory at this baseline, so this report uses `docs/research/` alongside the existing `docs/adr/` and `docs/agents/` documentation.

Use Student and Instructor as defined in [GLOSSARY](../../GLOSSARY.md). An inactive Student is not necessarily Lapsed: Lapsed means the Student lost their Enrollment. [Schema](../../app/db/schema.ts) has Enrollment completion and lesson progress, but those rows alone do not define the latest meaningful course activity. Preserved progress after Enrollment loss is intentional in [the refund ADR](../adr/0001-approving-a-refund-request-keeps-progress.md).

Engineering implications, proposed for later specification rather than settled here:

- Cadence must define and record qualifying activity, reminder eligibility, current Enrollment and unfinished-course checks. A vendor timer cannot infer these from SQLite.
- Two viable patterns are a periodic eligibility scan that triggers a provider workflow, or activity-triggered delayed workflows with cancellation/reset. Choose once inactivity duration and hosting/scheduler ownership are known. A scan avoids triggering a workflow for every video heartbeat.
- Recheck eligibility near external delivery; cancellation races and stale payloads otherwise produce reminders after resumption, completion or Enrollment loss. Provider acceptance is not proof of display, reading, or course resumption.
- Discussion recipients, self-notification exclusion, thread/course authorization, Instructor participation, subscription changes, and deleted-content handling remain Cadence responsibilities. All three vendors distribute notifications; none replaces the existing discussion system.
- Use durable event identifiers and delivery intent records across application retries. Vendor idempotency has a finite window. Prefer an outbox/reconciliation strategy in the later integration decision; no implementation is proposed here.

## Provider comparison

### Knock

**Inbox and integration.** `@knocklabs/react` supplies a real-time feed, popover and headless hooks. Seen and read are distinct: the default popover behavior should not be treated as proof a Student read a discussion. The client exposes read, unread and archive actions. [React feed](https://docs.knock.app/in-app-ui/react/feed), [JavaScript SDK](https://docs.knock.app/in-app-ui/javascript/sdk/quick-start).

Production must enable enhanced security and pass a backend-signed user JWT; a public key alone can otherwise act as other users. Tokens expire and need renewal. This fits a server-authenticated loader/token endpoint plus a browser feed, but exact React 19/React Router 8 package compatibility and hydration are untested. Client-only mounting is a conservative integration option, not a vendor-certified React Router recipe. [Authentication](https://docs.knock.app/in-app-ui/security-and-authentication).

**Reminders and noise control.** Delays accept fixed intervals, dynamic intervals or timestamps, up to 365 days. Recipient properties/preferences refresh at step execution; subscription properties do not. Existing delayed runs retain their original workflow version. [Delay function](https://docs.knock.app/designing-workflows/delay-function).

Cancellation requires a cancellation key and a pausing step. It is asynchronous; a successful response means enqueued, and the docs warn against canceling within five seconds of triggering. Finished messages cannot be recalled by canceling; persistent feed items can instead be archived. [Cancellation](https://docs.knock.app/send-notifications/canceling-workflows).

Batch, branch, fetch and conditional channel steps support discussion aggregation and escalation based on feed state. Throttling is per recipient/workflow, optionally keyed, with a maximum 31-day window. It cannot currently enforce a cap across different workflows or cancel an active throttle. Thus a single Student-wide cap across reminders and discussions still needs shared application coordination or deliberate workflow design. [Workflow functions](https://docs.knock.app/designing-workflows/overview), [Throttle limits](https://docs.knock.app/designing-workflows/throttle-function).

Preferences support channel, workflow and category settings with React/headless interfaces. They need product labels/defaults and unsubscribe behavior, not just storage. Trigger idempotency uses `Idempotency-Key` for 24 hours; the general overview limits this guarantee to the workflow trigger endpoint. [Preferences](https://docs.knock.app/in-app-ui/react/preferences), [API idempotency](https://docs.knock.app/api-reference/overview).

**Reliability.** Retryable sends get up to eight retries over 30 minutes with backoff/jitter, request/response logs and manual redelivery for undelivered messages. Permanent bounces are not retried. Push delivery status stops at `sent`; do not equate it with a displayed notification. [Delivery pipeline](https://docs.knock.app/send-notifications/delivering-notifications), [Statuses](https://docs.knock.app/send-notifications/message-statuses).

**Extra providers.** Email requires a connected delivery service; Knock bills separately from it. There is no first-party browser-push channel: the official React tutorial uses Firebase Cloud Messaging (FCM), Firebase SDK, VAPID configuration, a service worker and stored browser tokens. Its Next.js examples need adaptation to React Router. [Web push tutorial](https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm), [Channels](https://docs.knock.app/concepts/channels).

**Effort assessment:** comparatively straightforward inbox and workflow integration, with additional preference UI, signed authentication and app eligibility logic. Browser push adds a second SDK and browser lifecycle. The paid price floor below is the main cost concern.

### Novu Cloud

**Inbox and integration.** `@novu/react` has an Inbox with unread counts and preferences, with composable/headless options. Current setup docs explicitly support React 18 and 19. US and EU accounts need matching API/socket endpoints. Production HMAC signatures must be generated on the server and required by the in-app provider. [Inbox](https://docs.novu.co/platform/inbox), [React setup](https://docs.novu.co/platform/inbox/setup-inbox), [Production authentication](https://docs.novu.co/platform/inbox/prepare-for-production).

The exact React Router SSR/hydration path has not been demonstrated in this research. Treat the live inbox as a client boundary and confirm installed package behavior before committing integration effort. A framework-specific Next.js package is not proof of React Router compatibility. Dashboard workflows avoid adding a code-first bridge endpoint unless custom workflow logic actually requires it.

**Reminders and noise control.** Workflow action steps include delay, digest and throttle; external channels need active integrations. Cancellation uses the prior `transactionId` to stop active/pending workflows. Global and workflow-specific channel preferences are exposed by `usePreferences`. [Steps](https://docs.novu.co/platform/workflow/add-and-configure-steps), [Cancellation API](https://docs.novu.co/api-reference/events/cancel-triggered-event), [Preference hook](https://docs.novu.co/platform/sdks/react/hooks/use-preferences).

The limits reference gives Free/Pro/Team delays of 1/7/30 days, throttles of 1 hour/24 hours/7 days, and Inbox retention of 30/90/90 days. Activity-log retention is separately 1/7/90 days. The pricing page instead advertises a 90-day Team delay/digest window. Confirm the account's actual limit; use the conservative 30-day Team limit for planning. Free cannot directly express a single week-long inactivity delay. A periodic Cadence eligibility scan remains an alternative, with its own operational work. Do not conflate short log retention with inbox retention. [Limits reference](https://docs.novu.co/platform/developer/limits), [Pricing comparison](https://novu.co/pricing).

API-key authenticated POST/PATCH requests support idempotency; success/error responses are cached for 24 hours and in-progress requests for five minutes. This is request protection, not a permanent reminder ledger. [Idempotency](https://docs.novu.co/api-reference/idempotency).

**Reliability caveat.** Current docs explicitly state ordinary email/SMS/chat/push provider calls are attempted once, with no automatic retry or provider failover, even on provider 429/503 errors. Failures appear in Activity Feed and `messages.failed` webhooks; later channel steps can still run. Bridge requests and outbound webhooks have separate retry policies. A low platform bill therefore buys less automatic delivery recovery; deciding whether application replay is needed belongs in the integration ticket. [Delivery retry policy](https://docs.novu.co/platform/developer/delivery-retries).

**Extra providers.** Production email and push need delivery integrations. FCM supports browser-push overrides, but Cadence still registers browser subscriptions/tokens and implements notification clicks. The demo email integration is restricted to the Novu account email, not a production sender. [FCM integration](https://docs.novu.co/platform/integrations/push/fcm), [Trigger/test requirements](https://docs.novu.co/platform/workflow/trigger-workflow).

**Effort assessment:** easy initial Inbox and preferences, attractive Cloud costs, but potentially greater recovery and scheduling ownership. Evaluate hosted Cloud; self-hosting adds infrastructure and maintenance and is not an assumed cost-saving answer to the third-party requirement.

### Courier

**Inbox and integration.** Courier hosts persistent user messages with read/archive state synchronized across devices, unread views and pagination. `@trycourier/courier-react` targets React 18+ and includes Inbox, Toast and Preferences. Backend-issued scoped JWTs authenticate the client. Its docs explicitly say inbox/toast render client-side in SSR setups; custom inbox integrations must start real-time listening after authentication. React Router 8 hydration and React 19 package installation remain untested. [Hosted Inbox](https://www.courier.com/docs/in-app/inbox), [React SDK](https://www.courier.com/docs/sdk-libraries/courier-react-web).

**Reminders and noise control.** New flows use Journeys rather than legacy Automations: send, delay, branch, fetch, throttle, batch and digest nodes. A delay can wait up to 93 days. Profile/data are snapshotted at invoke, so a later profile update alone will not refresh a reminder's eligibility; use a fetch or cancel. Cancel targets active runs by run ID or shared cancellation token. Invoke supports an idempotency header. [Journeys](https://www.courier.com/docs/journeys/overview).

Throttle nodes support per-user or global scope at a point within a journey. Global means all recipients of that throttle, not a documented single-user cap across every journey. Validate common caps and branch semantics before promising shared limits. Preferences include topics, channel routing and digest schedules. [Throttle](https://www.courier.com/docs/journeys/nodes/throttle), [React preferences surface](https://www.courier.com/docs/sdk-libraries/courier-react-web). Courier documents a default 25-hour idempotency window; retain application deduplication beyond that. [Official idempotent send example](https://www.courier.com/recipes/payment-receipt-email).

**Reliability and providers.** Inbox needs no downstream provider. External email/push use connected providers such as SES/Resend and FCM. Courier documents provider 429 backoff for up to 24 hours, then configured failover or an undeliverable result; do not infer identical retries for every error. Logs/analytics expose failures. FCM configuration includes mobile-oriented payload formatting, so browser payloads and click behavior need explicit verification rather than copying native-mobile examples. [Integrations/retries](https://www.courier.com/docs/integrations/overview), [FCM configuration](https://www.courier.com/docs/integrations/push/firebase-fcm).

**Effort assessment:** straightforward hosted inbox plus preference components; moderate Journey wiring and current-state fetch/cancellation work. Retention guarantees for the inbox and exact account feature gating need verification.

## Cost models and representative monthly scenarios

USD public list prices, excluding tax, foreign-exchange changes, engineering, hosting and external delivery. Estimates are arithmetic illustrations, not quotes. All assume one recipient workflow/template per notification intent, one in-app message plus one email, no batching, no skips/failures, no AI, and no extra test traffic. A discussion event with 50 recipients means 50 intents, not one. Actual Student/Instructor numbers and event fan-out are unknown.

| Assumed audience and intents | In-app + email messages | Knock | Novu Cloud | Courier |
| --- | ---: | ---: | ---: | ---: |
| 100 people, 5 intents each = 500 | 1,000 | $0 | $0, subject to Free limits | $0 on Developer |
| 1,000 people, 10 intents each = 10,000 | 20,000 | $250 | $0, subject to Free limits | $0 on Developer |
| 5,000 people, 10 intents each = 50,000 | 100,000 | $500 | $54 on Pro | $250 |
| 20,000 people, 10 intents each = 200,000 | 400,000 | $2,000 | $234 on Pro, if limits fit | $1,000 |

Knock: free 10,000 delivered messages; Starter $250 including 50,000, then $0.005/message. In-app plus email counts twice; skipped/failed sends do not count. Downstream delivery is extra. Example formula: `$250 + max(0, messages - 50,000) * $0.005`. [Knock pricing](https://knock.app/pricing).

Novu: free 10,000 runs; Pro from $30 including 30,000, then $1.20/1,000; Team from $250 including 250,000. A run is per subscriber, independent of channel count. Extra triggers that are later canceled/throttled are not assumed free; confirm billing treatment. [Novu pricing](https://novu.co/pricing), [Billing definition](https://docs.novu.co/platform/account/billing). High activity-trigger volumes can erase the advantage over a periodic eligible-only scan. The scenarios above exclude that amplification.

Courier: Developer free 10,000 sends; Business charges $0.005 for **every** send, including the first 10,000. Adding a card moves the workspace to Business. One multichannel template per recipient counts once, but separate send nodes/templates can add units. Failed/preference-blocked sends and test sends count. This differs materially from Knock and from assuming 10,000 free units on a paid plan. [Courier billing](https://www.courier.com/docs/workspaces/billing).

Adding browser push adds another Knock channel message; Novu's same run and Courier's same multichannel template do not inherently add platform units. A separate later push invocation or template may do so. These are consequences of the cited billing definitions, not a guarantee that any particular workflow uses only one unit.

### Delivery fees added to the platform bill

| Email service example | 500 emails | 10,000 emails | 50,000 emails | 200,000 emails | Material qualification |
| --- | ---: | ---: | ---: | ---: | --- |
| SES Essentials, $0.16/1,000 | $0.08 | $1.60 | $8 | $32 | New account/region default under the current plan structure |
| SES explicitly selected à la carte, $0.10/1,000 | $0.05 | $1 | $5 | $20 | Data/optional services extra |
| Resend transactional plans | $0 if daily cap fits | $20 | $20 | $160 published tier | Other plan/overage combinations may differ |

Current SES pricing distinguishes Essentials from à la carte; do not budget every new account at the historical $0.10 rate automatically. New accounts start sandboxed, restricted to verified recipients, 200 emails/day and one/second until production access is approved. Sender identities still require verification afterward. This adds setup/approval work despite cheap unit costs. [SES pricing](https://aws.amazon.com/ses/pricing/), [Sandbox requirements](https://docs.aws.amazon.com/ses/latest/dg/request-production-access.html).

Resend Free is 3,000/month and 100/day; Pro $20 includes 50,000 with $0.90/1,000 overage. Its published Scale 200,000 tier is $160. Free monthly allowance alone does not accommodate a burst of course reminders. [Resend current price data](https://resend.com/pricing.md). Verify the selected platform integration and sender-region requirements before signup.

FCM itself has no messaging charge; other Firebase/Google services, orchestration and engineering can still cost money. Adding a paid push service instead changes this calculation. [Firebase billing](https://firebase.google.com/docs/projects/billing/firebase-pricing-plans), [FCM pricing](https://firebase.google.com/products/cloud-messaging).

## Email versus browser push

| Dimension | Email | Browser push |
| --- | --- | --- |
| Reach while inactive | Can address a known valid mailbox without a prior browser grant; inbox placement is not guaranteed | Requires a prior granted subscription on a supported browser/device; cannot newly enroll someone who has already stopped visiting |
| Permission/setup | No browser permission dialog; product subscription/consent choices remain to be specified | Explicit permission after user interaction; HTTPS, registration, keys/tokens and click handling for the conventional FCM path |
| Mobile web | Read through the person's mail app | iOS/iPadOS requires a Home Screen web app and permission; support starts at 16.4 |
| Best fit, research assessment | Course resumption and summaries with explanatory context | Timely, brief discussion alerts for people who opted in |
| Operations | Verified sender/DNS, delivery provider, bounces/complaints, suppression and unsubscribe handling | Service worker or supported declarative path, token refresh/removal, permissions revoked, multiple devices, account switching, payload privacy and expired messages |
| Reliability limit | Provider acceptance and email opens are not proof of reading/resumption | Delivery/display depends on browser/device/OS policies; a push receipt is not inbox read state |

Conventional Push API delivery uses a subscription tied to a service worker and can arrive without an open page. Subscriptions may expire; tokens must be detached on account switching to avoid cross-account disclosure. MDN documents the subscription lifecycle and secure-context requirements. Apple documents Home Screen installation and user-gesture permission on iOS. [Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API), [Service workers](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API), [Apple Web Push](https://webkit.org/blog/13878/).

There is a newer exception to the service-worker requirement: Safari 18.4 introduced Declarative Web Push for Home Screen web apps. Do not assume every browser or selected provider/SDK supports that path. The verified cross-provider comparison here uses FCM's conventional web SDK requirements. [Safari 18.4](https://webkit.org/blog/16574/webkit-features-in-safari-18-4/), [FCM web setup](https://firebase.google.com/docs/cloud-messaging/web/get-started).

Email integration also needs domain verification and bounce handling; an orchestration provider does not establish sender reputation for Cadence. Resend documents verified domains, reputation isolation and domain warmup; SES documents sender verification, and the platform delivery references above distinguish bounces from successful delivery. [Verified domains and reputation](https://resend.com/docs/dashboard/domains/introduction), [SES sender requirements](https://docs.aws.amazon.com/ses/latest/dg/request-production-access.html). No legal classification or consent policy is selected by this report.

## Data location and documented contradictions

- Knock's current plan documentation says EU residency is coming by end of 2026. Treat EU as unavailable/unconfirmed today, not shipped. A future launch promise cannot satisfy a present hard location requirement. [Knock account documentation](https://docs.knock.app/manage-your-account/knock-plans).
- Novu advertises US/EU Cloud residency; its setup documents distinct EU API/socket endpoints. A UK-only location is a separate enterprise discussion. Pricing search snippets showed an older Free workflow allowance; the live pricing page and limits reference both say five. Team delay limits still disagree (30 versus 90 days), as recorded above. [Novu pricing](https://novu.co/pricing), [Region configuration](https://docs.novu.co/platform/inbox/setup-inbox).
- Courier's pricing page lists EU residency under Enterprise, but its regional documentation explicitly says Business and Enterprise. Obtain account-specific confirmation before budgeting an EU deployment. Region-specific API, Inbox, real-time and token issuance must all align. [Courier pricing](https://www.courier.com/pricing), [Datacenters](https://www.courier.com/docs/workspaces/datacenters).
- A platform region does not guarantee the same location for email/push delivery, support access, backups or every subprocessor. Review the chosen platform and delivery provider's DPA, subprocessors and deletion/retention controls against an actual human requirement. No such requirement has been supplied, and no contract was reviewed.

## Open decisions and later validation

Public research is complete. No account-access blocker prevented this comparison. The following work remains visible for the decision owners; this report does not approve production use.

| Existing decision ticket | Evidence or unanswered question it must carry forward |
| --- | --- |
| [Define course-resumption notification rules](https://github.com/mark-richards-vitesse/cohort-005-project/issues/2) | What activity resets inactivity, duration, repeat/cooldown, eligible courses/Enrollments, time zone and cancellation conditions? Long durations affect Novu plan viability. |
| [Define basic discussion notification rules](https://github.com/mark-richards-vitesse/cohort-005-project/issues/3) | Which Students/Instructors get replies/new threads, self-exclusion, subscription defaults, batching, and shared frequency caps? Fan-out dominates cost. |
| [Choose the MVP external notification channel](https://github.com/mark-richards-vitesse/cohort-005-project/issues/5) | Email's broader addressability versus push permission/install friction; actual browser mix, opt-in/default choices, fallback and unsubscribe policy. |
| [Select the third-party notification provider](https://github.com/mark-richards-vitesse/cohort-005-project/issues/6) | Hard monthly budget, realistic intents and activity triggers, data location, acceptable retention/branding, and reliability expectations. Confirm contradictory plan/region limits with account evidence. |
| [Define notification integration and delivery ownership](https://github.com/mark-richards-vitesse/cohort-005-project/issues/8) | Scheduler/outbox owner, stale-data recheck, persistent dedup/caps, auth, feed retention, failed-send response, secrets, region endpoints, deletion and monitoring. Validate exact SDK peer dependencies, SSR build/hydration, token refresh and account switching after selection. |
| [Agree notification acceptance criteria and engagement measurement](https://github.com/mark-richards-vitesse/cohort-005-project/issues/9) | Define actual resumption rather than opens; verify suppression after completion/Enrollment loss, cancellation races, read-state behavior, provider errors and preference changes. |

Access-dependent prerequisites after those choices: selected vendor workspace/plan/region and API entitlement; sender domain/DNS access and email production approval; FCM project/service account/VAPID keys if push is selected; supported real devices and authorized test recipients for a later delivery check. None has been requested or created here. These are conditional validation items within the existing provider/channel/integration decisions, not grounds to choose a provider or provision services during research. Create a precise access task if a selected-path validation actually cannot proceed.

Relative effort assessments above are reasoned estimates from documented integration surfaces, not measured implementation times. No runtime tests were run: this branch adds only this research note and leaves `app/` unchanged.

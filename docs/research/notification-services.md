# Notification services compared against Cadence's requirements

Research for [issue #2](https://github.com/Xemrox/cohort-005-project/issues/2), part of the in-app notifications map ([#1](https://github.com/Xemrox/cohort-005-project/issues/1)). Sources checked 2026-11-11. Prices and plan limits change often, so re-check them before signing anything.

**Hard requirements (from #1):** EU data residency **and** a GDPR DPA; browser (web) push supported, even if v1 doesn't ship it. **Soft requirements:** a free or starter tier, and an affordable cost at ~10× today's users.

"SuperSend" in the ticket is read as **SuprSend** (suprsend.com). Pingram (formerly NotificationAPI) was also checked as an extra candidate.

## TL;DR

- **Meets both hard requirements on its own:** **Novu** (EU region in Frankfurt on every plan including Free; DPA and SCCs downloadable). **SuprSend** probably does too: its docs say workspaces can be in an EU region and it publishes a DPA. However, the pricing page doesn't say which plans get the EU region.
- **Meets them only on a paid plan:** **Courier**. Its EU datacenter (Ireland) is available on Business and Enterprise, not on the free Developer plan.
- **Fails EU residency today:** **Knock**. It hosts on AWS in the USA, and its own security docs say "We expect to support EU data residency by the end of 2026". **MagicBell** publishes no hosting region and no EU option.
- **Web push differs a lot by vendor.** Only SuprSend and MagicBell send browser push themselves (VAPID and a service worker). Novu, Knock and Courier send it through another provider (FCM, OneSignal, Pushpad, Pusher Beams). That provider then becomes one more subprocessor that needs its own EU/DPA check.

**Shortlist (recommendation, not a decision): Novu, SuprSend, Courier.** Trade-offs are [at the end](#shortlist-and-trade-offs).

## Comparison table

| | **Novu** | **SuprSend** | **Courier** | **Knock** | **MagicBell** |
|---|---|---|---|---|---|
| **EU data residency** | Yes, EU (Frankfurt) on **all plans**; region chosen at signup, keys bound to region [N1] | EU region documented ("Each workspace has a data region, including the EU") [S2]; plan gating not stated; pricing page only says "Custom Data Center Region" on Enterprise [S1] | Yes, EU (Ireland) on **Business and Enterprise only**, not free Developer [C2] | **No.** AWS, USA [K3]; "expect to support EU data residency by the end of 2026" [K4] | **Not published**; security page names no region [M2] |
| **GDPR DPA** | Yes, DPA + SCC template at novu.co/dpa [N1] | Yes, DPA linked from security docs [S3] | Yes, public DPA [C3] (pricing page lists "MSA, DPA" under Enterprise [C1]) | Yes, DPA with SCCs, but data goes to the USA under international-transfer clauses [K5] | "provides data processing agreements" (on request; no public DPA found) [M2] |
| **Web (browser) push** | Via providers: FCM, OneSignal, Pushpad, Pusher Beams [N3] | **Native** VAPID + service worker, React `registerPush()` [S4] | Via providers (FCM, OneSignal, Pusher Beams, MagicBell…); no first-party browser SDK documented [C4] | Via FCM only ("Send web push with FCM" tutorial) [K6] | **Native** VAPID + service worker [M3] |
| **In-app feed UI (React)** | `<Inbox />` prebuilt + headless hooks (`@novu/react`); React Router and Remix quickstarts [N4] | `@suprsend/react` drop-in + headless [S5] | `CourierInbox` drop-in + toasts (`@trycourier/courier-react`) [C5] | Prebuilt feed + headless "custom UI" + styling [K7] | `@magicbell/react` + `@magicbell/react-headless` [M4] |
| **Feed auth** | HMAC `subscriberHash` [N5] | JWT (React SDK) [S5] | Backend-minted scoped JWT [C6] | "Enhanced security mode": signed user token [K8] | User JWT [M5] |
| **Workflow engine** | Digest, delay, step conditions incl. `steps.<inapp>.seen` / `.read` [N6] | Batch, digest, delay, wait-until, branch [S6]; engagement-aware "Smart Channel Routing" [S7]; digest and batching are **Business plan** [S1] | Journeys, topic digest, batching [C7] | Batch, delay, **message-status step conditions** ("only send … if in-app not read") [K9] | Basic: `broadcast`, `wait`, `pause`, `abort`, `if`; no digest or batch listed [M6] |
| **Preferences** | Built into `<Inbox />` + `<Subscription />` component + `usePreferences` hook [N7] | Preference APIs (user, tenant, object) + hosted/embeddable centre [S8]; **Business plan** [S1] | Hosted page or embedded component [C8] | Preferences API, hosted centre, custom centre [K10] | Not clearly documented [M1] |
| **Node server SDK** | Yes (`@novu/api`) [N2] | Yes (Node backend SDK) [S9] | Yes (`@trycourier/courier`) [C2] | Yes [K1] | `magicbell-js` (browser/Node) [M1] |
| **Self-host fallback** | **Open-core**: `LICENSE-MIT` + `LICENSE-ENTERPRISE` in repo [N8]; also hybrid-cloud [N1] | Enterprise-licensed self-host / BYOC, not open source [S10] | No | No | No |
| **Free tier** | 10k workflow runs/mo, 5 workflows, 24 h activity retention [N2] | 10k notifications/mo, in-app with SuprSend branding [S1] | 10k sends/mo, **no EU** [C1] | 10k messages/mo [K1] | 1k deliveries/mo [M1] |
| **Paid entry** | Pro from $30/mo (30k runs), $1.20/1k overage [N2] | Essentials $100 (50k, $2/1k); Business $250 (50k, $5/1k) [S1] | Business $0.005/send [C1] | Starter $250 (50k), $0.005/msg [K1] | Startup $249 (50k), $0.0025/delivery; **each channel counts** [M1] |
| **≈ Cost at 100k notifications/mo**¹ | **≈ $114** (Pro: $30 + 70k × $1.20/1k) | ≈ $250 (Business at its 50k-included allowance) – $500 (+ 50k × $5/1k); Essentials ≈ $200 but no digest or preferences | **≈ $500** (100k × $0.005) | ≈ $500 | ≈ $374, doubling once push is added, since each channel counts |

¹ *Cadence's current notification volume is unknown. The table uses 100k notifications/month to stand in for "~10×" a free-tier-sized (≈10k/mo) start. It is an estimate made from the list prices above, not a quote.*

## Notes by vendor

### Novu
- **Residency:** EU region in Frankfurt, available on all plans. Each region has its own API and WebSocket hostnames (`eu.api.novu.co`, `eu.socket.novu.co`). An account can't be moved between regions by itself; that needs sales. In the Inbox component, pass `backendUrl` and `socketUrl` for the EU [N1][N4].
- **Push:** No browser push of its own. Pushpad is the documented browser-specific provider; OneSignal and Pusher Beams also reach browsers [N3]. Whichever provider is used must also pass the EU/DPA check.
- **Billing unit:** One "workflow run" covers every step it runs, so in-app plus push in one workflow should cost one run [N2]. This is why it is cheapest at 10× volume.
- **Weakness:** Free plan keeps only 24 h of activity-feed history (a debugging log, not the notifications themselves) [N2].

### SuprSend
- **Residency:** Docs say each workspace has a data region "including the EU" [S2] and that SuprSend "support[s] EU-data residency options" [S3]. The pricing page doesn't say whether the EU region comes with Free or Essentials [S1]. **Confirm in writing before choosing.**
- **Strong on push:** Native browser push, with no extra subprocessor [S4].
- **Plan gating:** Digest, batching and user preferences are Business-only ($250/mo) [S1]. In practice Cadence would need Business.

### Courier
- **Residency:** A fully separate EU datacenter (Ireland) with its own console, API and Inbox/real-time endpoints. Paid plans only [C2].
- **Push:** Goes through push providers. The docs only describe Courier SDKs registering device tokens for mobile (APNs/FCM) [C4], so browser push means adding OneSignal or similar.
- **Cost:** No free path with EU residency. Business is pay-as-you-go at $0.005/send [C1]; whether the first 10k stay free on Business is not stated.

### Knock (fails a hard requirement today)
- A strong product: message-status conditions, batch, preferences, enhanced-security tokens [K7–K10]. But all its subprocessors, including AWS, are in the USA [K3]. EU residency is on the roadmap for "end of 2026" [K4]. Re-check if that ships.
- Web push only via FCM [K6].

### MagicBell (fails a hard requirement as published)
- Native browser push and React components [M3][M4]. However, there is no published hosting region, no EU option, and no public DPA (only "provides data processing agreements") [M2]. The workflow engine is thin: no digest or batch listed [M6]. Each channel counts as a delivery for billing [M1].

### Pingram (formerly NotificationAPI), excluded
- The old notificationapi.com pricing URL now redirects to pingram.io. That pricing page lists email, SMS and voice, with "US, EU, CA" regions, but doesn't mention in-app or web push [P1]. The older NotificationAPI docs did have in-app and web push in a React SDK [P2]. With the product mid-rebrand and pivoting, it is not a safe pick.

## Shortlist and trade-offs

1. **Novu**: *best fit on paper.*
   - **For:** EU residency and a DPA on every plan, Free included. Cheapest at 10× (~$114/mo for 100k). Most complete in-app kit (Inbox with built-in preferences, headless hooks, React Router quickstart). Read-aware step conditions. Open-core licence as an exit route.
   - **Against:** Browser push needs another provider (e.g. Pushpad or OneSignal), which needs its own EU/DPA check. Free plan keeps only 24 h of activity logs. Region is fixed at signup.
2. **SuprSend**: *best web push story.*
   - **For:** Native VAPID browser push, so no extra subprocessor. Rich workflow nodes (batch, digest, wait-until, smart routing). Enterprise self-host/BYOC as a fallback.
   - **Against:** Digest, batching and preferences need the $250/mo Business plan. EU-region plan gating isn't stated publicly; confirm before choosing. Free in-app feed is SuprSend-branded.
3. **Courier**: *most clearly documented EU setup, but paid from day one.*
   - **For:** Separate EU datacenter with documented endpoints for API, Inbox and real-time. Public DPA. JWT-scoped client auth. Hosted or embedded preference centre. Digest and journeys.
   - **Against:** No free tier with EU residency. Highest cost at 10× (~$500/mo). Browser push via a third-party provider.

**Requirement no vendor fully met:** No single vendor offers **EU residency + native browser push + digest/preferences on a free tier**. Novu gives EU and the full feature set free but needs a push provider. SuprSend has native push but puts digest and preferences behind Business and is vague on EU plan gating. Courier puts EU behind a paid plan. Knock and MagicBell don't currently offer documented EU data residency.

**Open questions for the map:**
- Does a browser push provider (e.g. Pushpad, OneSignal, FCM) count as acceptable under the EU residency rule? FCM is a Google US service. This decides whether Novu or Courier can satisfy "web push" without adding risk.
- SuprSend: which plan includes the EU region?
- Cadence's real monthly notification volume, to firm up the 10× cost.

**SSR note (inference, not a vendor claim):** All five feed SDKs hold a WebSocket or real-time connection and run in the browser. With React Router 7 SSR, the feed would render client-only (e.g. mounted after hydration). The server side only mints the HMAC hash or JWT in a loader.

## Sources

**Novu**
- [N1] Security, compliance & data residency: https://docs.novu.co/platform/additional-resources/security.md
- [N2] Pricing: https://novu.co/pricing
- [N3] Push integrations: https://docs.novu.co/platform/integrations/push.md, https://docs.novu.co/platform/integrations/push/pushpad.md
- [N4] Inbox / React quickstart / EU region config: https://docs.novu.co/platform/quickstart/react.md, https://docs.novu.co/platform/inbox/setup-inbox.md
- [N5] Prepare Inbox for production (HMAC): https://docs.novu.co/platform/inbox/prepare-for-production.md
- [N6] Step conditions: https://docs.novu.co/platform/workflow/add-and-configure-steps/step-conditions.md
- [N7] Preferences: https://docs.novu.co/platform/inbox/configuration/preferences.md, https://docs.novu.co/platform/subscription.md
- [N8] Repository licence files: https://github.com/novuhq/novu (`LICENSE-MIT`, `LICENSE-ENTERPRISE`)

**SuprSend**
- [S1] Pricing: https://www.suprsend.com/pricing
- [S2] What is SuprSend: https://docs.suprsend.com/docs/what-is-suprsend
- [S3] Security: https://docs.suprsend.com/docs/security
- [S4] React web push: https://docs.suprsend.com/docs/react-webpush.md
- [S5] React inbox: https://docs.suprsend.com/docs/react-inbox-integration.md
- [S6] Workflow nodes (batch, digest, delay, wait-until): https://docs.suprsend.com/llms.txt
- [S7] Smart Channel Routing: https://docs.suprsend.com/docs/smart-delivery.md
- [S8] Preference guide: https://docs.suprsend.com/docs/guides/preference-guide.md
- [S9] API overview (Node SDK): https://docs.suprsend.com/reference/overview.md
- [S10] Self-hosted: https://docs.suprsend.com/docs/self-hosted/introduction.md

**Courier**
- [C1] Pricing: https://www.courier.com/pricing
- [C2] Regional datacenters: https://www.courier.com/docs/workspaces/datacenters.md
- [C3] Compliance / DPA: https://www.courier.com/docs/workspaces/compliance.md, https://www.courier.com/data-processing-addendum
- [C4] Push providers: https://www.courier.com/docs/integrations/push/overview.md
- [C5] React inbox: https://www.courier.com/docs/guides/add-an-inbox-to-react.md
- [C6] JWT auth: https://www.courier.com/docs/in-app/authenticate-users.md
- [C7] Digest: https://www.courier.com/docs/guides/send-a-daily-digest.md
- [C8] Preference center: https://www.courier.com/docs/guides/build-a-preference-center.md

**Knock**
- [K1] Pricing: https://knock.app/pricing
- [K3] Subprocessors: https://knock.app/legal/subprocessors
- [K4] Security (EU residency roadmap): https://docs.knock.app/security
- [K5] DPA: https://knock.app/legal/data-processing-addendum
- [K6] Web push via FCM: https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm
- [K7] Feeds / custom UI: https://docs.knock.app/in-app-ui/feeds/overview.md
- [K8] In-app security: https://docs.knock.app/in-app-ui/security-and-authentication.md
- [K9] Step conditions: https://docs.knock.app/designing-workflows/step-conditions.md
- [K10] Preferences: https://docs.knock.app/preferences/overview.md

**MagicBell**
- [M1] Pricing: https://www.magicbell.com/pricing; docs index: https://www.magicbell.com/docs
- [M2] Security: https://www.magicbell.com/security
- [M3] Web push: https://magicbell.com/docs/web-push
- [M4] React: https://www.magicbell.com/docs/libraries/magicbell-react, https://magicbell.com/docs/libraries/react-headless
- [M5] User JWT: https://www.magicbell.com/docs/api/authentication/user
- [M6] Workflows: https://www.magicbell.com/workflows

**Pingram / NotificationAPI**
- [P1] Pricing (redirect target): https://www.pingram.io/pricing
- [P2] NotificationAPI React SDK: https://docs.notificationapi.com/reference/react-sdk

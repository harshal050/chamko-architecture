# Chamko — Platform Architecture

How the whole platform fits together, from a customer scanning a QR code to a post appearing on
Google. Diagrams are [Mermaid](https://mermaid.js.org/) and render directly on GitHub.

For setup and deploy steps see [`README.md`](README.md), [`docs/DEPLOY.md`](docs/DEPLOY.md) and
[`docs/GOOGLE_SETUP.md`](docs/GOOGLE_SETUP.md).

---

## 1. What Chamko does

Chamko is an autopilot for the **Google Business Profile** of Indian local businesses:

- **Daily Google posts** — AI writes the text, a design engine renders the image, and the post is
  published to Google on schedule.
- **Review replies** — every new review gets a reply in the reviewer's language, written in a
  natural owner's voice.
- **Profile kit** — description, services, keywords and FAQs built from the owner's own details.
- **Review QR** — a printed QR that opens the business's Google review box.
- **Growth tools** — rank checks, monthly reports, plan-expiry notifications.
- **10 languages** — English, Hindi, Gujarati, Marathi, Tamil, Telugu, Bengali, Kannada, Malayalam, Punjabi.

**Pricing:** one plan, ₹499 per business per 30 days, with a 4-day free trial the first time a
Google location is connected (`packages/shared/src/constants.ts`; the price is editable in the admin panel).

---

## 2. The big picture

```mermaid
flowchart LR
  subgraph Clients
    WEB["Website<br/>chamkomarketing.app<br/>(apps/web)"]
    DASH["Dashboard<br/>dash.chamkomarketing.app<br/>(apps/dash)"]
    ADMIN["Admin panel<br/>admin.chamkomarketing.app<br/>(apps/admin)"]
    APP["Mobile app<br/>Flutter (apps/mobile)"]
    QR["Customer scanning<br/>a review QR"]
  end

  subgraph Cloudflare
    API["API Worker<br/>chamko-api (Hono)<br/>/v1/*"]
    CRON["Cron trigger<br/>every 5 min"]
    Q[("Queue<br/>chamko-jobs")]
    AI["Workers AI<br/>Llama 3.3 70B · FLUX schnell"]
  end

  subgraph External
    DB[("MongoDB Atlas")]
    CLD["Cloudinary<br/>(images)"]
    GBP["Google Business<br/>Profile APIs"]
    PS["Google Pub/Sub<br/>(new-review events)"]
    RZP["Razorpay"]
    MAIL["Gmail SMTP"]
    PUSH["Browser push services<br/>(FCM / Mozilla / Apple)"]
    SHEET["Google Sheets<br/>(admin export)"]
    META["Facebook / Instagram"]
  end

  WEB -- "public endpoints" --> API
  DASH --> API
  ADMIN --> API
  APP --> API
  QR --> WEB --> API

  CRON --> API
  API -- "enqueue one job per business" --> Q --> API
  API --> AI
  API --> DB
  API --> CLD
  API <--> GBP
  PS -- "push" --> API
  API <--> RZP
  RZP -- "webhook" --> API
  API --> MAIL
  API --> PUSH
  API --> SHEET
  API --> META
```

There is **one backend**: the `chamko-api` Cloudflare Worker. Everything else is either a static
front-end or an outside service. The Worker answers HTTP requests, runs the cron trigger, and
consumes its own queue.

---

## 3. Monorepo layout

```mermaid
flowchart TB
  subgraph packages
    SHARED["packages/shared<br/>types · zod schemas · constants<br/>i18n JSON (10 languages)"]
    UI["packages/ui<br/>shadcn/ui components · theme<br/>API client · i18n provider · config"]
  end
  subgraph apps
    API["apps/api<br/>Hono on Workers"]
    WEB["apps/web<br/>Next.js static export"]
    DASH["apps/dash<br/>Next.js static export"]
    ADMIN["apps/admin<br/>Next.js static export"]
    MOB["apps/mobile<br/>Flutter"]
  end
  SHARED --> API
  SHARED --> UI
  UI --> WEB
  UI --> DASH
  UI --> ADMIN
  SHARED -. "i18n copied into assets" .-> MOB
```

| Path | Role | Hosted on |
|---|---|---|
| `apps/api` | REST API, cron, queue consumer | Cloudflare Workers (`chamko-api`) |
| `apps/web` | Marketing site, ~4,700 guides, free tools, review-QR redirect page | Cloudflare Pages (`chamko-web`) |
| `apps/dash` | Owner dashboard (same features as the app) | Cloudflare Pages (`chamko-dash`) |
| `apps/admin` | Internal admin panel | Cloudflare Pages (`chamko-admin`) |
| `apps/mobile` | Android / iOS app | Play Store / App Store |
| `packages/shared` | Single source of truth for types, limits and all UI strings | — |
| `packages/ui` | Shared React design system and API client | — |

Tooling: **pnpm workspaces + Turborepo**, TypeScript everywhere except the Flutter app.

---

## 4. Inside the API Worker

```mermaid
flowchart TB
  REQ["HTTPS request"] --> CORS["CORS + secure headers"]
  CORS --> RL["Rate limits<br/>AUTH_LIMITER 10/min · API_LIMITER 120/min"]
  RL --> LOG["Activity log middleware"]
  LOG --> ROUTER{"/v1 router"}

  ROUTER --> R1["/auth · /me"]
  ROUTER --> R2["/business · /posts · /media"]
  ROUTER --> R3["/reviews · /growth · /reports"]
  ROUTER --> R4["/google · /meta"]
  ROUTER --> R5["/billing · /push"]
  ROUTER --> R6["/admin"]
  ROUTER --> R7["/public · /public/google"]

  R1 & R2 & R3 & R4 & R5 & R6 & R7 --> SVC["services/*<br/>(business logic)"]
  SVC --> LIB["lib/*<br/>google · ai · studio · cloudinary<br/>razorpay · mail · webpush · sheets"]
  SVC --> DB[("MongoDB<br/>withDb(): one client per invocation")]
```

Code layout inside `apps/api/src`:

| Folder | Contains |
|---|---|
| `routes/` | One Hono router per area; validates input with zod and calls services |
| `services/` | Business logic: posts, reviews, billing, plan, push, rank, growth, settings… |
| `lib/` | Thin clients for outside systems, plus `ai/` (prompts + Workers AI) and `studio/` (image renderer) |
| `jobs/index.ts` | `schedule()` for the cron and `consume()` for the queue |
| `models.ts`, `lib/db.ts` | MongoDB document types and collections |

**Database rule:** each request, cron run or queue batch opens its own MongoDB client (`withDb`).
Workers cancel a request that touches another request's sockets, so clients are never shared.

### Main collections

| Collection | Holds |
|---|---|
| `users`, `admins` | Accounts, session version (sign out everywhere), admin access |
| `businesses` | Profile, Google connection (encrypted tokens), plan end date, autopilot settings, QR slug |
| `posts`, `media` | Daily posts (draft → scheduled → published) and uploaded photos |
| `reviews` | Google reviews and our replies |
| `payments`, `coupons`, `trials` | Razorpay orders and payments, discount codes, one-trial-per-location record |
| `push_subscriptions` | Browser push endpoints for plan reminders |
| `ai_cache`, `ai_usage`, `ai_meter`, `ai_accounts` | Cached AI results, daily limits, credit metering, extra AI accounts |
| `cloudinary_accounts` | Extra image-storage accounts (uploads move on as one fills up) |
| `settings`, `settings_history` | Admin-editable platform settings (deep-merged over defaults) and their history |
| `activity`, `snapshots`, `system`, `broadcasts` | Audit log, daily metric snapshots, system state, admin announcements |

---

## 5. Sign-in and connecting Google

New users sign up with Google in **one step**: sign-in and Business Profile access are granted together.

```mermaid
sequenceDiagram
  autonumber
  actor Owner
  participant D as Dashboard / App
  participant G as Google
  participant A as API Worker
  participant M as MongoDB

  Owner->>D: Continue with Google
  D->>G: OAuth (identity + business.manage scope)
  G-->>D: ID token + authorization code
  D->>A: POST /v1/auth/google
  A->>G: Verify ID token, exchange code for refresh token
  A->>M: Create / find user, store encrypted refresh token
  A-->>D: Session JWT (60 days, HS256)
  D->>A: GET /v1/google/locations
  A->>G: Account Management + Business Information APIs
  A-->>D: Owner's locations
  Owner->>D: Pick a location
  D->>A: Connect location
  A->>M: Save business, start the free trial (once per location)
  A->>G: Register for new-review Pub/Sub events
```

- Existing users can also sign in with an email OTP. New accounts must use Google.
- Bumping a user's `tokenVersion` invalidates every session on every device.
- Google tokens are encrypted at rest with `ENCRYPTION_KEY`.
- Admins sign in with Google **and** an authenticator code (TOTP), and every admin action is logged.

---

## 6. Automation: cron → queue → jobs

The cron fires **every 5 minutes**. It never does heavy work itself; it decides what is due and
enqueues **one message per business** on `chamko-jobs`. The queue consumer handles 5 messages at a
time and retries failures (up to 3 times, then a later sweep picks them up), so one broken
business never blocks the others.

```mermaid
flowchart TB
  CRON(("Cron<br/>*/5 min")) --> S["schedule()"]
  S --> P1{"From 20:00 IST"}
  P1 -- "tomorrow's post" --> J1["prepare"]
  S --> P2{"Post time reached"}
  P2 --> J2["publish"]
  S --> P3{"From 10:00 IST,<br/>no post today"}
  P3 --> J3["daily (with self-heal retry)"]
  S --> P4{"Reviews"}
  P4 -- "10 businesses per message" --> J4["reviewsBatch"]
  S --> P5{"Billing"}
  P5 --> J5["Razorpay reconciliation<br/>renewal e-mails"]
  S --> P6{"Hourly, 10:00–20:00 IST"}
  P6 --> J6["Plan-expiry push notifications"]
  S --> P7{"Other"}
  P7 --> J7["snapshot · report · gsync<br/>sheetsSync · email"]

  J1 & J2 & J3 & J4 & J7 --> Q[("chamko-jobs queue")]
  Q --> C["consume(): batch of 5,<br/>retry 3x, per-business isolation"]
```

Only businesses with **their own active plan** (paid or trial) get automation. Admins can pause
any automation globally from the settings panel, for example during an AI outage.

---

## 7. The daily post pipeline

```mermaid
sequenceDiagram
  autonumber
  participant Q as Queue job (prepare)
  participant AI as Workers AI (Llama 3.3)
  participant ST as Studio renderer
  participant IMG as Workers AI (FLUX) / owner photos
  participant CL as Cloudinary
  participant M as MongoDB
  participant G as Google Business Profile

  Q->>M: Load business, recent posts and headlines
  Q->>AI: postPrompt(theme of the day, focus service, keywords)
  AI-->>Q: JSON {content, headline, eyebrow, tagline, badge, cta, imagePrompt}
  Q->>IMG: Owner's photo, or generate one from imagePrompt
  Q->>ST: Compose design (layouts, palette from brand colour, fonts)
  ST-->>Q: Finished JPEG
  Q->>CL: Upload (first account under 85% of its credits)
  Q->>M: Save post as scheduled for 10:00 IST
  Note over Q,M: Owner and admins can edit or replace it before it goes out
  Q->>G: publish job at the scheduled time: create localPost
  Q->>M: Mark published, store Google post ID
```

- Themes, headline styles and the focus service rotate by day and business, so posts don't repeat.
- Prompt rules forbid phone numbers, URLs, hashtags, ALL CAPS and invented offers, all of which
  Google rejects.
- **Without Google API access**, posts are still created; the owner copies them and taps
  "I posted it".

---

## 8. Reviews: arrival and reply

```mermaid
flowchart LR
  NEW["New Google review"] --> PS["Pub/Sub push<br/>/v1/public/google"]
  NEW -. "safety net" .-> POLL["Cron sweep<br/>(reviewsBatch)"]
  PS --> EV["handleReviewEvent()"]
  POLL --> ALL["autoReplyAll()"]
  EV & ALL --> SAVE[("reviews collection")]
  SAVE --> AUTO{"Auto-reply on<br/>and not answered?"}
  AUTO -- yes --> GEN["reviewReplyPrompt()<br/>+ owner's last 6 replies"]
  GEN --> CLEAN["naturalReply()<br/>remove dashes, emojis, quotes"]
  CLEAN --> POST["Google: reply to review"]
  POST --> DONE[("reply saved")]
  POST -- failure --> RETRY["replyFailedAt → retried every 15 min"]
```

Replies follow the length and language of the review. They avoid template phrases and repeat
the owner's recent wording as little as possible. The shop name or a keyword appears only
occasionally, and never in replies to complaints. Owners can also edit or rewrite any reply from
the dashboard.

---

## 9. Review QR

```mermaid
sequenceDiagram
  actor C as Customer
  participant W as chamkomarketing.app/r/?s=SLUG
  participant A as API /v1/public/review/SLUG/go
  participant G as Google review box

  C->>W: Scan printed QR
  W->>A: Immediate redirect
  A->>A: Count scan (qrScans + 1)
  A-->>C: 302 → business's Google review link
  C->>G: Taps stars, writes own review, posts
```

- The QR URL never changes, so printed codes keep working forever.
- The API only redirects to Google domains. If the stored link isn't a Google link, it opens a
  Google search for the business instead, so the endpoint can't be misused as an open redirect.
- Chamko never writes review text for customers; that would break Google's review policy.

---

## 10. Billing

```mermaid
sequenceDiagram
  autonumber
  actor O as Owner
  participant D as Dashboard / App
  participant A as API
  participant R as Razorpay
  participant M as MongoDB

  O->>D: Renew plan for a business
  D->>A: POST /v1/billing/order (optional coupon)
  A->>M: Read price from settings, check coupon
  A->>R: Create order
  A-->>D: Order ID
  D->>R: Razorpay Checkout (UPI / card / netbanking)
  R-->>D: payment_id + signature
  D->>A: POST /v1/billing/verify
  A->>A: Verify HMAC signature
  A->>M: Record payment, extend the business's plan by 30 days
  R-->>A: Webhook (backup path)
  Note over A,R: Cron reconciles unconfirmed orders every 15 min
```

Each business has its own plan. We never see card or UPI details.

---

## 11. Plan-expiry push notifications

```mermaid
flowchart LR
  OPT["Owner taps 'Turn on alerts'<br/>(dashboard)"] --> SW["Service worker<br/>apps/dash/public/sw.js"]
  SW --> SUB["POST /v1/push/subscribe"] --> DB[("push_subscriptions")]
  CRON["Hourly sweep<br/>10:00–20:00 IST"] --> STAGE{"Stage?"}
  STAGE --> T1["trial1: trial ends within 24 h"]
  STAGE --> D3["d3: 3 days left"]
  STAGE --> D1["d1: ends tomorrow"]
  STAGE --> D0["d0: ended today"]
  STAGE --> A3["a3 / a7: 3 or 7 days after"]
  T1 & D3 & D1 & D0 & A3 --> SEND["sendWebPush()<br/>VAPID + aes128gcm (WebCrypto)"]
  SEND --> BROWSER["Notification → tap → /billing"]
```

- Each stage fires at most once per plan period.
- Text is in the owner's language.
- Subscriptions that the push service reports as gone are deleted automatically.

Works on Android Chrome and desktop browsers, and on iPhone when the dashboard is added to the
Home Screen.

---

## 12. The marketing website and SEO

```mermaid
flowchart LR
  MD["content/guides/*.md<br/>(~4,700 guides)"] --> LOADER["lib/content.ts<br/>frontmatter · quick answer · FAQ · TOC"]
  LOADER --> PAGES["/guides/[slug]/<br/>Article + FAQPage + Breadcrumb JSON-LD"]
  LOADER --> HUBS["/guides/topic/[topic]/ (paginated)<br/>/guides/language/[lang]/"]
  LOADER --> SM["sitemap.xml<br/>(hub lastmod = newest guide)"]
  LOADER --> LLMS["llms.txt · llms-full.txt (≤ 20 MiB)"]
  LOADER --> FEED["feed.xml"]
  TOOLS["/tools/* (16 free tools)"] --> SM
  PAGES & HUBS & SM & LLMS & FEED --> OUT["Static export → Cloudflare Pages"]
```

- **Guide format:** frontmatter (`title`, `description`, `category`, `date`, `keywords`, optional
  `lang`). The first blockquote becomes the "Quick answer", and `## Frequently asked questions`
  becomes FAQ structured data.
- **Related guides** are matched by weighted keywords, giving every page internal links.
- **Cloudflare Pages limits:** at most 20,000 files and 25 MiB per file. The deploy workflow
  removes the per-guide RSC `.txt` payloads and fails early if the file count is too high.
  `llms-full.txt` is capped at 20 MiB.
- `robots.txt` allows all crawlers, including AI assistants, and the `/r` QR page is `noindex`.

---

## 13. Front-end apps

| App | Main screens |
|---|---|
| Dashboard (`apps/dash`) | Home, Posts (all posts, published or not), Reviews, Photos, Profile, Growth, Billing (live plan, renew in the last 5 days), Settings, onboarding, printable poster and receipts |
| Admin (`apps/admin`) | Users, businesses, payments, coupons, posts, AI and Cloudinary credit accounts, platform settings (price, feature switches), broadcasts, health, activity log, team |
| Mobile (`apps/mobile`) | Same feature set as the dashboard (`features/auth, home, posts, reviews, photos, profile, billing, settings`) |

All three web apps are Next.js **static exports**, so there is no server rendering at runtime. They
call the API through the shared client in `packages/ui`. Strings come from `packages/shared/src/i18n`.
`en.json` is the source, and `pnpm i18n:translate` fills the other nine languages.

---

## 14. AI and image infrastructure

```mermaid
flowchart LR
  CALL["generateJson / generateImage"] --> FO["runWithFailover()"]
  FO --> ACC1["Main Cloudflare account"]
  FO -- "credit used up / error" --> ACC2["Extra accounts<br/>(ai_accounts, added in admin)"]
  CALL --> CACHE[("ai_cache")]
  CALL --> LIMIT[("ai_usage: per-business daily limit")]
  UP["uploadImage()"] --> CA1["Main Cloudinary"]
  UP -- "≥ 85% of monthly credits" --> CA2["Extra Cloudinary accounts"]
```

- Every prompt is a short role plus numbered hard rules, the facts it may use, and a **JSON
  schema**, so output is always parseable. Rule 1 everywhere is "never invent facts".
- Results are cached, and owner-triggered AI actions count against a daily per-business limit.
  Autopilot runs are metered but never blocked.

---

## 15. Build and deploy

```mermaid
flowchart LR
  DEV["git push → main"] --> GH["GitHub Actions"]
  GH --> CHK["checks.yml<br/>tsc for every app (PRs)"]
  GH --> DA["deploy-api.yml<br/>wrangler deploy"] --> W["Worker chamko-api"]
  GH --> DW["deploy-web.yml<br/>next build → trim → pages deploy"] --> PW["Pages chamko-web"]
  GH --> DD["deploy-dash.yml"] --> PD["Pages chamko-dash"]
  GH --> DAD["deploy-admin.yml"] --> PA["Pages chamko-admin"]
```

- Nothing is built on a laptop. Pushing to `main` deploys whatever changed.
- Each workflow runs only when its app or the shared packages change. A newer push cancels a web
  build that is still running.
- **Secrets** (`MONGODB_URI`, `JWT_SECRET`, `ENCRYPTION_KEY`, Razorpay, Cloudinary, Google, Meta,
  mail and `VAPID_PRIVATE_KEY`) live only in Cloudflare Worker secrets. They are never stored in the
  repo or in GitHub. Public values (URLs, model names, client IDs, the VAPID public key) are
  `vars` in `apps/api/wrangler.jsonc`.
- Local checks: `tsc --noEmit` per app and `wrangler deploy --dry-run` for the API.

---

## 16. Safety and policy guard-rails

| Area | Guard-rail |
|---|---|
| Google posts | No phone numbers, URLs, hashtags, ALL CAPS or invented offers; one post per business per day |
| Review replies | Natural owner voice, no keyword stuffing, no personal details, calm replies to complaints |
| Reviews | Chamko never writes reviews for customers; the QR goes straight to Google |
| Review requests | The WhatsApp message asks for an honest review and never offers anything in return |
| Google data | Used only to run the service (Google API Services User Data Policy, Limited Use); tokens encrypted |
| Accounts | Deleting an account removes businesses, posts, reviews, media, drafts and push subscriptions |
| API | Rate limits, secure headers, zod validation, redirects only to Google domains, admin TOTP + audit log |

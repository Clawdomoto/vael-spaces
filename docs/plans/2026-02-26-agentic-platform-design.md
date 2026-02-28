# Vael Spaces — Agentic Hospitality Marketing Platform

**Date:** 2026-02-26
**Status:** Approved
**Authors:** Moto (AI), Satoshi Clawdomoto

---

## Vision

Evolve Vael Spaces from an AI image generation tool into an autonomous marketing platform that progressively takes over a hotel's entire digital footprint. Hotels connect their channels, configure brand guardrails, and an AI marketing team plans, creates, schedules, publishes, and optimizes content across every surface — social media, Google Business Profile, OTA listings, and paid ads.

The image generation product is the foot in the door. The agentic platform is the long-term offering.

---

## Design Decisions

| Decision | Answer | Rationale |
|---|---|---|
| Architecture | B+C Hybrid — standalone platform with agent orchestration model designed in from day one, monolith initially, extract to services when scale demands | Clean security boundary, proper multi-tenancy, right abstractions without distributed systems overhead |
| Channel rollout | Graduated autonomy — social first, then GBP, OTA, ads, email | Mirrors how trust builds with conservative hoteliers. Each module is an upsell lever |
| Autonomy model | Per-channel tiers (Propose / Guardrailed / Autopilot) | Different risk profiles per channel. Social = low risk, ads = high risk |
| First module | Social content engine | Natural extension of current product. Fastest to ship, easiest to prove value |
| Pricing | Tiered platform plans (Starter → Growth → Pro → Enterprise) | Simpler packaging than module add-ons. Creates natural upgrade pressure |
| Operations | Managed onboarding + self-serve hybrid | White-glove setup builds trust. Self-serve adjustments scale. Vael ops monitors high-value accounts |

---

## 1. System Architecture

Three layers, two deployments.

```
┌─────────────────────────────────────────────────────────┐
│  SPACES PLATFORM (spaces.vaelcreative.com)              │
│  Next.js · Neon PostgreSQL · Vercel                     │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐             │
│  │ Customer  │  │ Agent    │  │ Admin     │             │
│  │ Dashboard │  │ Control  │  │ Panel     │             │
│  │           │  │ Plane    │  │ (Vael ops)│             │
│  └─────┬────┘  └────┬─────┘  └─────┬─────┘             │
│        │             │              │                    │
│  ┌─────┴─────────────┴──────────────┴─────┐             │
│  │         Agent Orchestration Layer       │             │
│  │                                         │             │
│  │  ┌──────────────────────────────────┐   │             │
│  │  │      Marketing Director Agent    │   │             │
│  │  │  (plans, schedules, coordinates) │   │             │
│  │  └──────┬───────┬───────┬───────┬───┘   │             │
│  │         │       │       │       │       │             │
│  │  ┌──────┴┐ ┌────┴──┐ ┌─┴────┐ ┌┴─────┐ │             │
│  │  │Social │ │ GBP   │ │ OTA  │ │ Ads  │ │             │
│  │  │Agent  │ │ Agent │ │Agent │ │Agent │ │             │
│  │  └───────┘ └───────┘ └──────┘ └──────┘ │             │
│  └─────────────────────────────────────────┘             │
│                                                         │
│  ┌─────────────────────────────────────────┐             │
│  │     Background Workers (Inngest)        │             │
│  │  generation · publishing · analytics    │             │
│  └─────────────────────────────────────────┘             │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  VAEL CREATIVE DASHBOARD (dashboard.vaelcreative.com)   │
│  Internal ops · CRM · Agency work                       │
│                                                         │
│  ◄──── API Bridge (sync brand DNA, assets, metrics) ──► │
└─────────────────────────────────────────────────────────┘
```

### Key Decisions

- **Spaces is a standalone Next.js app** with its own Neon database, auth, and Vercel project. The existing Vercel project (`vael-spaces`, ID: `prj_37nbIJMMGsnRLMxCYJTcHHLRcKU2`) and domain (`spaces.vaelcreative.com`) are already provisioned.
- **Agent orchestration lives inside Spaces** as a module, not a separate service. Each agent is a class implementing a shared interface. Extraction to separate workers happens when scale demands it — but the abstraction is there from day one.
- **Marketing Director** is the orchestrator — it owns the content calendar, delegates to channel agents, and aggregates performance data back into planning decisions.
- **Inngest** handles async execution (scheduled posts, analytics pulls, generation jobs, delayed metric collection).
- **API bridge** between Spaces and the Vael Creative dashboard is thin and read-heavy — syncs brand DNA, generated assets, and performance metrics. Vael ops onboards clients in the dashboard; agents consume that data in Spaces.

---

## 2. Multi-Tenant Data Model

The Spaces database is its own Neon instance — clean separation from the agency dashboard.

### Tenancy

```sql
organizations
  id, name, slug, stripe_customer_id, subscription_plan, subscription_status,
  created_at, updated_at

properties
  id, organization_id, name, type (hotel|resort|airbnb|venue),
  location, timezone, created_at

members
  id, organization_id, user_id, role (owner|admin|member),
  invited_by, joined_at

users
  id, email, name, password_hash, avatar_url, email_verified,
  created_at, last_login_at
```

### Brand & Content

```sql
brand_profiles
  id, property_id, voice_description, visual_style (JSONB),
  target_audience (JSONB), positioning, guardrails (JSONB),
  color_palette (JSONB), generated_by_ai (boolean),
  reviewed_by_human (boolean), created_at, updated_at

content_library
  id, property_id, type (image|video|copy), url, thumbnail_url,
  metadata (JSONB), tags (text[]), status (draft|approved|posted|archived),
  source (studio|upload|agent), created_at

content_performance
  id, content_id, channel, impressions, reach, engagement_rate,
  saves, shares, clicks, measured_at
```

### Agent Orchestration

```sql
agent_configs
  id, property_id, channel (social|gbp|ota|ads),
  autonomy_level (propose|guardrailed|autopilot),
  guardrails (JSONB), enabled (boolean),
  created_at, updated_at

content_calendar
  id, property_id, channel, content_id, caption,
  scheduled_at, status (planned|pending_approval|approved|published|failed|rejected),
  agent_rationale (text), rejection_feedback (text),
  published_at, published_id (platform post ID),
  created_by (agent|human), created_at

agent_decisions
  id, property_id, agent (director|social|gbp|ota|ads),
  action (plan|generate|publish|respond|optimize),
  rationale (text), input_data (JSONB), output_data (JSONB),
  outcome (auto_approved|human_approved|human_rejected|failed),
  performance_score (float), created_at

channel_connections
  id, property_id, provider (meta|google|tiktok|pinterest|booking|expedia),
  access_token_encrypted, refresh_token_encrypted,
  scopes (text[]), external_account_id, external_page_id,
  status (active|expired|revoked), connected_at, expires_at

agent_learnings
  id, property_id, channel, insight (text),
  evidence (JSONB), confidence (float),
  applied_count (int), created_at
```

### Key Decisions

- **Organization → Properties** hierarchy supports both single-property boutiques and multi-property groups from day one. Pricing attaches at the org level, agents operate at the property level.
- **`agent_configs`** is the control surface — one row per property per channel. The customer sets autonomy level and guardrails here. This is what makes "social = autopilot, ads = propose-only" work per-channel.
- **`content_calendar`** is the Marketing Director's workspace — it plans posts, the channel agents execute them. The approval flow depends on the autonomy level configured in `agent_configs`.
- **`agent_decisions`** is the full audit trail. Every action an agent takes is logged with its rationale. This builds trust with conservative hoteliers and feeds the learning loop.
- **`channel_connections`** stores encrypted OAuth tokens for each platform. Separate from user auth.
- **`agent_learnings`** stores per-property insights the Director extracts from performance data. These compound over time — the longer a property is on the platform, the smarter its agent becomes.

---

## 3. Agent Orchestration Model

### Marketing Director Agent

The central orchestrator. Runs a daily planning cycle per property.

```
Runs: Daily (Inngest cron, property-local-time, e.g. 6am)

Inputs:
  • Brand profile + guardrails
  • Content library (available assets)
  • Performance data (last 30 days by channel)
  • Calendar (what's already planned/posted)
  • Seasonal context (holidays, events, local happenings)
  • Agent learnings (accumulated per-property insights)

Outputs:
  • Content calendar entries for the next 7 days
  • Generation requests (if content gaps exist)
  • Channel-specific briefs dispatched to agents

Loop:
  Plan → Delegate → Monitor → Learn → Replan
```

### Agent Interface

Every channel agent implements the same contract. This is what makes the system extensible — add a new channel without modifying the orchestrator.

```typescript
interface ChannelAgent {
  channel: "social" | "gbp" | "ota" | "ads"

  // Director sends a brief, agent returns planned actions
  plan(brief: AgentBrief): Promise<PlannedAction[]>

  // Execute an approved action (post, update listing, etc.)
  execute(action: ApprovedAction): Promise<ExecutionResult>

  // Pull latest performance data from the channel
  measure(propertyId: string, dateRange: DateRange): Promise<ChannelMetrics>

  // Is the OAuth token valid, is the account connected?
  healthCheck(connection: ChannelConnection): Promise<HealthStatus>
}

interface AgentBrief {
  propertyId: string
  brandProfile: BrandProfile
  availableContent: ContentItem[]
  recentPerformance: ChannelMetrics
  existingCalendar: CalendarEntry[]
  learnings: AgentLearning[]
  dateRange: { start: Date; end: Date }
}

interface PlannedAction {
  type: "post" | "story" | "reel" | "update_listing" | "respond_review" | "create_ad"
  contentId: string
  caption: string
  scheduledAt: Date
  rationale: string
  confidence: number
}
```

### Autonomy Levels

The autonomy level on `agent_configs` controls the gate between `plan()` and `execute()`:

**PROPOSE (lowest trust)**
```
Director plans → Agent plans actions → Customer reviews
→ Customer approves/rejects each → Agent executes approved
```
Use case: New customers, paid ads, website changes.

**GUARDRAILED (medium trust)**
```
Director plans → Agent plans actions → Auto-check against guardrails
→ Pass = auto-execute → Fail = queue for human review
```
Use case: Social posts after 30 days of PROPOSE mode, GBP updates.

**AUTOPILOT (highest trust)**
```
Director plans → Agent plans + executes immediately
→ Customer notified post-publish → Can retroact (delete/edit)
```
Use case: Social posts for established customers, review responses.

All three levels log to `agent_decisions`. Emergency stop is always available — a single toggle that immediately pauses all agents and revokes publishing permissions.

### The Learning Loop

After every published action, the system closes the feedback loop:

```
Publish → Wait 48h → Pull metrics → Score outcome
→ Extract learning → Store in agent_learnings
→ Director incorporates into future planning

Metrics scored per channel:
  Social: engagement rate, reach, saves, shares
  GBP: views, clicks, direction requests, calls
  OTA: impression rank, click-through, booking conversion
  Ads: ROAS, CPA, CTR, frequency
```

The Director uses this to learn patterns like "Resort pool photos at 10am Tuesday get 3x the engagement of lobby photos at 6pm Friday for this property." Learning is **per-property** — what works for a ski lodge in Aspen is different from a beach resort in Maui.

### Social Agent (Module 1)

**Platforms:** Instagram (feed + stories + reels), Facebook (page posts). TikTok and Pinterest added later.

**Capabilities:**
- Select images from the content library matching the planned brief
- Generate captions using brand voice + guardrails
- Schedule posts via Meta Graph API
- Pull engagement metrics via Meta Insights API
- Respond to comments (GUARDRAILED or AUTOPILOT only, with brand voice constraints)

**Boundaries (v1):**
- Does NOT create new images (that's the content generation layer)
- Does NOT manage DMs (too high-risk)
- Does NOT run paid promotion (that's the Ads Agent)

### Execution Model (Inngest)

```
Cron: "0 6 * * *" (daily, property-local-time)
  → inngest.send("director/daily-plan", { propertyId })

  → Marketing Director runs plan()
  → Creates content_calendar entries
  → For each entry, dispatches to channel agent

  → Channel agent runs plan()
  → Returns PlannedActions

  → Autonomy gate:
      PROPOSE    → save as pending_approval, notify customer
      GUARDRAILED → run guardrail check → auto-approve or queue
      AUTOPILOT  → execute immediately

  → At scheduled publish time:
      inngest.send("agent/execute", { actionId })
      → Agent calls platform API
      → Logs to agent_decisions
      → Notifies customer (if configured)

  → 48h later:
      inngest.send("agent/measure", { actionId })
      → Pull metrics → Score → Extract learning → Store
```

---

## 4. Customer Experience

### Navigation

```
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────┐
│ Content  │ │ Calendar │ │ Channels │ │ Insights  │
│ Studio   │ │          │ │          │ │           │
└──────────┘ └──────────┘ └──────────┘ └───────────┘

┌──────────┐ ┌──────────┐ ┌──────────┐
│ Brand    │ │ Agent    │ │ Settings │
│ Profile  │ │ Activity │ │          │
└──────────┘ └──────────┘ └──────────┘
```

### Onboarding Flow

Vael ops kicks this off from the agency dashboard. The customer gets a welcome email and lands in a guided setup:

1. **Create account** — email + Google OAuth
2. **Add property** — name, type, location, upload 5-10 space photos
3. **Brand DNA wizard** — upload existing marketing materials or website URL. Gemini analyzes and extracts voice, color palette, target guest, positioning. Customer reviews and edits. Sets guardrails ("Never mention competitor names," "Always use luxury-casual tone," "No photos with alcohol near children").
4. **Connect channels** — Instagram/Facebook via Meta OAuth, Google Business Profile via Google OAuth. Future: TikTok, Pinterest, Booking.com, Expedia.
5. **Set autonomy levels** — defaults to PROPOSE for everything (safest). Vael ops can recommend upgrades after 30 days.

Vael ops monitors onboarding from the agency dashboard and can intervene at any step.

### Content Studio

The existing image generation product, repackaged as a tab within Spaces:

- Upload space photos → select talent → set vibe → generate lifestyle images
- Generated images land in the Content Library with status `draft`
- Images feed directly into the agentic pipeline — no longer a dead-end download

### Calendar View

The Marketing Director's work made visible:

```
┌──────────────────────────────────────────────────┐
│  February 2026                     [Week] [Month]│
│                                                  │
│  Mon 24    Tue 25    Wed 26    Thu 27    Fri 28   │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐ ┌──────┐│
│  │● IG  │  │○ IG  │  │● FB  │  │      │ │○ IG  ││
│  │Posted │  │Ready │  │Posted │  │      │ │Review││
│  │Pool   │  │Lobby │  │Spa   │  │      │ │Dining││
│  │eng 4.2│  │10am  │  │eng 3.8│ │      │ │10am  ││
│  └──────┘  └──────┘  └──────┘  └──────┘ └──────┘│
│            ┌──────┐                      ┌──────┐│
│            │● GBP │                      │◐ IG  ││
│            │Posted │                      │Plan'd││
│            │Photos │                      │Sunset││
│            └──────┘                      └──────┘│
│                                                  │
│  ● Published  ○ Pending approval  ◐ Planned      │
│                                                  │
│  [+ Add post]              [Let the agent plan ▶]│
└──────────────────────────────────────────────────┘
```

**Customer actions:**
- See everything planned, pending, and published
- Tap any pending post to approve, edit, or reject (with feedback the agent learns from)
- Manually add posts (the agent plans around them)
- Trigger on-demand planning cycle
- Drag posts to reschedule

**Post cards show:**
- Image thumbnail + caption preview
- Channel + scheduled time
- Agent's rationale ("Pool content performs best mid-week for your audience")
- Post-publish: engagement metrics inline

### Approval Queue

Dedicated inbox for PROPOSE and GUARDRAILED-flagged items:

```
┌──────────────────────────────────────────────────┐
│  Pending Approval (3)              [Approve All] │
│                                                  │
│  ┌────────────────────────────────────────────┐  │
│  │ IG Post · Thu Feb 27, 10:00 AM            │  │
│  │                                            │  │
│  │ [image]  "Unwind in our redesigned spa     │  │
│  │           suite, where every detail         │  │
│  │           invites calm."                    │  │
│  │                                            │  │
│  │ Agent: "Spa content has 2.1x engagement    │  │
│  │ rate vs. average. Thursday morning slots    │  │
│  │ outperform weekends by 40% for your        │  │
│  │ audience."                                  │  │
│  │                                            │  │
│  │ [Approve] [Edit & Approve] [Reject] [Skip] │  │
│  └────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────┘
```

Rejections include a feedback field. The agent stores this in `agent_decisions` and the Director factors it into future planning.

### Agent Activity Feed

Chronological log of everything the agent has done — the trust-building transparency layer:

```
┌──────────────────────────────────────────────────┐
│  Agent Activity                    [Filter]      │
│                                                  │
│  Today                                           │
│  09:12  Planned 5 posts for next 7 days          │
│  09:12  Requested 3 new images (pool, sunset,    │
│         dining) — content gap detected           │
│  09:13  Auto-published IG story (guardrailed)    │
│                                                  │
│  Yesterday                                       │
│  10:00  Published IG post — Spa suite            │
│         → 4.2% engagement (avg: 2.8%)            │
│  10:01  Updated GBP photos — added 4 new         │
│  14:30  Weekly performance digest generated      │
│  14:30  Learning: "Warm-toned evening shots      │
│         get 60% more saves than daylight"        │
│                                                  │
│  EMERGENCY STOP                      [STOP ALL]  │
└──────────────────────────────────────────────────┘
```

Emergency stop is always visible. One click pauses all agents, revokes publishing tokens, and notifies Vael ops.

### Insights Dashboard

Performance data pulled from connected channels, interpreted by the Director:

- Engagement trends by content type, time, and channel
- Top performing posts with analysis of why they worked
- Content gap recommendations ("No winter-themed content; ski season starts in 6 weeks")
- Audience growth over time
- For paid ads (future): ROAS, CPA, budget utilization

The Director provides **narrative insights**, not just metrics: "Your pool content consistently outperforms by 2x. Consider generating more poolside variations for next month."

### Notifications

Configurable per channel:
- **Email digest** (daily or weekly) — what was posted, what's pending, performance highlights
- **Push/in-app** — approval requests, emergency situations
- **Slack/webhook** (future) — for teams that live in Slack

---

## 5. Platform Tiers & Module Unlocking

### Tier Structure

| | Starter | Growth | Pro | Enterprise |
|---|---|---|---|---|
| **Price** | $149/mo/property | $399/mo/property | $799/mo/property | Custom ($2K+/property) |
| **Content Studio** | 80 images | 200 images | 500 images | Unlimited |
| **Social Agent** | — | 30 posts/mo | 90 posts/mo | Unlimited |
| **GBP Agent** | — | — | Included | Included |
| **OTA Agent** | — | — | Included | Included |
| **Ads Agent** | — | — | — | Included |
| **API Access** | — | — | — | Included |
| **White Label** | — | — | — | Included |
| **Channels** | — | 2 (IG+FB) | 5 | Unlimited |
| **Properties** | 1 | 3 | 10 | Unlimited |
| **Team members** | 1 | 3 | 10 | Unlimited |
| **Autonomy levels** | N/A (manual) | Propose + Guardrailed | All 3 | All 3 + custom |
| **Onboarding** | Self-serve + docs | Guided (1 call) + brand review | White-glove (full setup) | Dedicated CSM + quarterly review |
| **Image overage** | $3/img | $2/img | $1.50/img | Included |
| **Post overage** | — | $5/post | $3/post | Included |

### Multi-Property Discounts

| Properties | Discount |
|---|---|
| 1-2 | List price |
| 3-5 | 15% off |
| 6-10 | 25% off |
| 11-25 | 35% off |
| 25+ | Custom |

### Upgrade Triggers

The agent surfaces data-backed upgrade recommendations:

- **Starter → Growth:** "You generated 60 images this month but only posted 12 to Instagram. The Social Agent could have published 30 optimized posts for you." Shown after 2+ weeks of active Studio usage.
- **Growth → Pro:** "Your Google Business Profile hasn't been updated in 45 days. Properties with fresh GBP photos get 35% more direction requests." Shown when GBP connection is detected but the module isn't unlocked.
- **Pro → Enterprise:** "You're managing 8 properties. At Enterprise, you'd save $X/month with volume pricing and unlock paid ads management." Shown when property count exceeds 5.

### Free Trial

- 14 days, no credit card
- Unlocks Growth tier (Studio + Social Agent)
- Limited to 1 property, 20 images, 10 scheduled posts
- Agent operates in PROPOSE mode only (no auto-publishing during trial)
- Day 7: automated check-in email with engagement preview
- Day 12: "Your trial ends in 2 days — here's what the agent accomplished" summary
- Vael ops notified on trial signup for white-glove outreach to high-value leads

---

## 6. Implementation Roadmap

### Overview

```
Phase 1: Foundation     (Weeks 1-4)   → Starter tier live ($149/mo)
Phase 2: Social Agent   (Weeks 5-10)  → Growth tier live ($399/mo)
Phase 3: Expand Channels (Weeks 11-18) → Pro tier live ($799/mo)
Phase 4: Scale          (Weeks 19-26) → Enterprise tier live (custom)
```

### Phase 1: Foundation (Weeks 1-4)

Working customer-facing app with Studio repackaged. No agents yet.

**Week 1-2: Core Platform**
- Next.js app on existing `vael-spaces` Vercel project
- Neon database with multi-tenant schema (organizations, properties, members)
- Auth: NextAuth with email/password + Google OAuth
- Customer signup + invite flow (Vael ops invites from agency dashboard)
- Role model: owner, admin, member (per organization)
- Row-level isolation from day one — every query scoped to `organization_id`

**Week 3: Brand & Content**
- Brand DNA wizard (Gemini analysis)
- Guardrails editor (structured rules)
- Content library (images with status, tagging, search)
- API bridge to agency dashboard (sync brand DNA + generated assets)

**Week 4: Studio Integration**
- Port Studio generation engine into Spaces
- Upload space photos → generate lifestyle images → content library
- Stripe: Starter tier checkout, usage metering
- Onboarding flow polished

**Exit criteria:** Customer can sign up, build brand profile, generate images, manage content library. Starter billing works. Vael ops monitors from agency dashboard.

### Phase 2: Social Agent (Weeks 5-10)

First agentic module. Make-or-break phase.

**Week 5-6: Agent Orchestration Layer**
- Agent interface defined (plan, execute, measure, healthCheck)
- Marketing Director agent: daily planning cycle via Inngest cron
- `agent_configs`, `content_calendar`, `agent_decisions` tables
- Autonomy gate logic

**Week 7-8: Social Agent**
- Meta OAuth: connect Instagram + Facebook page
- Caption generation using brand voice + guardrails
- Post scheduling via Meta Graph API
- Autonomy gate: PROPOSE → approval queue, GUARDRAILED → auto-check, AUTOPILOT → direct
- Approval queue UI

**Week 9: Calendar & Activity**
- Calendar view (week + month, drag to reschedule)
- Agent activity feed
- Emergency stop
- Email digest notifications (daily/weekly)

**Week 10: Beta Launch**
- Onboard 10 existing Vael Creative agency clients at Growth tier
- All start in PROPOSE mode
- Vael ops reviews first 2 weeks before upgrading to GUARDRAILED
- Stripe: Growth tier, post overage metering

**Exit criteria:** 10 beta customers with connected Instagram. Agent plans and schedules daily. At least 3 customers upgraded to GUARDRAILED. Engagement metrics flowing back.

### Phase 3: Expand Channels (Weeks 11-18)

**Week 11-12: Learning Loop**
- 48h post-publish metric collection (Inngest delayed job)
- Performance scoring per content type, time slot, channel
- Director incorporates learnings into planning
- Insights dashboard: top performers, content gaps, narrative analysis

**Week 13-14: GBP Agent**
- Google Business Profile API: OAuth, photo upload, post creation, review pull
- Review response generation (brand voice, GUARDRAILED by default)
- Auto-update photos on new content generation
- Weekly GBP post scheduling

**Week 15-16: OTA Agent**
- Booking.com Partner API / Expedia Marketplace API
- Listing photo optimization
- Description rewriting using brand voice
- Review response generation

**Week 17-18: Pro Tier Launch**
- Stripe: Pro tier checkout, multi-property billing
- Multi-property dashboard (org-level view)
- Cross-property learning

**Exit criteria:** Pro tier live with GBP + OTA agents. Learning loop improving content selection. At least 30 paying customers across tiers.

### Phase 4: Scale (Weeks 19-26)

**Week 19-21: Ads Agent**
- Meta Ads API: campaign creation, audience targeting, budget management
- Creative selection from content library (agent picks best performers)
- Budget optimization loop: reallocate toward high-ROAS creatives
- Hard guardrails: daily spend caps, mandatory approval for budget changes >20%
- PROPOSE mode only for first 30 days (no exceptions)

**Week 22-23: API & White-Label**
- REST API for agencies and hotel management companies
- API key management, rate limiting, usage billing
- White-label: custom domain, logo, color scheme per organization
- Webhook system for external integrations

**Week 24-26: Enterprise Launch**
- Multi-property discount engine in Stripe
- Dedicated CSM tooling in agency dashboard
- Quarterly review report generation
- Video content support (short-form for Reels/TikTok)
- Public launch marketing push

**Exit criteria:** Enterprise tier with at least 2 signed contracts. Ads Agent in PROPOSE mode for 5+ customers. API serving external consumers. $40K+ MRR.

### Parallel Workstream: Security Remediation

The security fix on the agency dashboard runs independently — it does not block Spaces work since Spaces is a separate codebase. See `SECURITY_AUDIT.md` for the full remediation plan.

The photo pipeline continues feeding the content library throughout all phases (already operational).

---

## Appendix: Revenue Projections

| Milestone | Target |
|---|---|
| Month 3 (end Phase 2) | $10K MRR, 50+ trial signups, 10+ paying |
| Month 6 (end Phase 3) | $40K MRR, 100+ paying customers |
| Year 1 (end Phase 4 + growth) | $120K MRR / $1.44M ARR |
| Year 3 | $1M+ MRR / $12M+ ARR |

**PMF kill criteria (Day 90):** <10 paying customers, <30% trial conversion, <50% 60-day retention, or <20 images/month average usage → pivot to Airbnb/real estate vertical.

## Appendix: Competitive Moat

The defensibility compounds over time:

1. **Per-property learning** — the longer a hotel uses the platform, the smarter its agent becomes. Switching costs increase with every week of accumulated performance data.
2. **Content library lock-in** — months of generated, tagged, performance-scored content lives in Spaces. Migration is painful.
3. **Multi-channel orchestration** — competitors offer point solutions (just social scheduling, just GBP management). Spaces is the unified agent that coordinates across all channels with a single brand voice.
4. **Hospitality specialization** — generic tools (Buffer, Hootsuite, Canva) don't understand hotel seasonality, OTA dynamics, or luxury hospitality aesthetics. The agents are trained on this vertical.
5. **Image generation integration** — no competitor can generate the content AND distribute it AND optimize it in a single platform. The full vertical integration from pixel to post to performance is unique.

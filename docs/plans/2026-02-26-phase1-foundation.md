# Phase 1: Foundation — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Extend the existing Vael Spaces app with multi-tenant data model, brand profiles, content library, Stripe billing (Starter tier), and API bridge to the agency dashboard.

**Architecture:** Vael Spaces is an existing Next.js app deployed on Vercel (`spaces.vaelcreative.com`) with Google OAuth, a talent library, and Gemini-powered image generation. Phase 1 adds the multi-tenant foundation needed for the agentic platform: organizations/properties/members hierarchy, brand DNA with guardrails, a content library that feeds into future agents, and Stripe billing.

**Tech Stack:** Next.js (App Router), Neon PostgreSQL, NextAuth.js, Stripe, Vercel Blob, Google Gemini, Inngest, Tailwind CSS

**Repo:** `anonymousaccountforfun/vael-spaces` (private, GitHub). Clone and work from this repo.

**Design Doc:** `docs/plans/2026-02-26-agentic-platform-design.md` (in `Clawdomoto/vael-spaces`)

---

## Pre-Implementation: Clone and Audit

Before starting any tasks, the implementer must:

1. Clone the repo: `git clone git@github.com:anonymousaccountforfun/vael-spaces.git`
2. Run `npm install` and `npm run dev` to verify the app starts
3. Read the existing database schema (check for migrations dir, schema files, or Drizzle/Prisma config)
4. Run `find app/api -name "route.ts" | head -40` to inventory existing API routes
5. Read `package.json` for current dependencies
6. Read the NextAuth config to understand the current auth setup
7. Identify the current DB connection pattern (raw SQL via `neon()`, Drizzle, Prisma, etc.)

This audit determines how each task below adapts to the existing codebase. The tasks below assume the patterns discovered by our exploration (Neon serverless, NextAuth v4, Vercel Blob), but the implementer should verify.

---

## Task 1: Multi-Tenant Database Schema

**Files:**
- Create: `db/migrations/001-multi-tenant-schema.sql`
- Create: `lib/db/schema.ts` (if not exists — type definitions for all tables)
- Modify: `lib/db/index.ts` (add query helpers with tenant scoping)

**Step 1: Write the migration SQL**

```sql
-- Organizations (hotel group or single property)
CREATE TABLE IF NOT EXISTS organizations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  slug TEXT NOT NULL UNIQUE,
  stripe_customer_id TEXT,
  subscription_plan TEXT DEFAULT 'starter',
  subscription_status TEXT DEFAULT 'trialing',
  trial_ends_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Properties (individual hotels/venues within an org)
CREATE TABLE IF NOT EXISTS properties (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  type TEXT NOT NULL DEFAULT 'hotel',
  location TEXT,
  timezone TEXT DEFAULT 'America/New_York',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_properties_org ON properties(organization_id);

-- Members (users within an org)
CREATE TABLE IF NOT EXISTS members (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  role TEXT NOT NULL DEFAULT 'member' CHECK (role IN ('owner', 'admin', 'member')),
  invited_by UUID REFERENCES users(id),
  joined_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(organization_id, user_id)
);

CREATE INDEX idx_members_org ON members(organization_id);
CREATE INDEX idx_members_user ON members(user_id);

-- Brand profiles (per property)
CREATE TABLE IF NOT EXISTS brand_profiles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  property_id UUID NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
  voice_description TEXT,
  visual_style JSONB DEFAULT '{}',
  target_audience JSONB DEFAULT '{}',
  positioning TEXT,
  guardrails JSONB DEFAULT '[]',
  color_palette JSONB DEFAULT '[]',
  generated_by_ai BOOLEAN DEFAULT FALSE,
  reviewed_by_human BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(property_id)
);

-- Content library (generated assets)
CREATE TABLE IF NOT EXISTS content_library (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  property_id UUID NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
  type TEXT NOT NULL DEFAULT 'image' CHECK (type IN ('image', 'video', 'copy')),
  url TEXT NOT NULL,
  thumbnail_url TEXT,
  metadata JSONB DEFAULT '{}',
  tags TEXT[] DEFAULT '{}',
  status TEXT NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'approved', 'posted', 'archived')),
  source TEXT NOT NULL DEFAULT 'studio' CHECK (source IN ('studio', 'upload', 'agent')),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_content_library_property ON content_library(property_id);
CREATE INDEX idx_content_library_status ON content_library(property_id, status);

-- Content performance (metrics from channels, linked to content)
CREATE TABLE IF NOT EXISTS content_performance (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  content_id UUID NOT NULL REFERENCES content_library(id) ON DELETE CASCADE,
  channel TEXT NOT NULL,
  impressions INT DEFAULT 0,
  reach INT DEFAULT 0,
  engagement_rate FLOAT DEFAULT 0,
  saves INT DEFAULT 0,
  shares INT DEFAULT 0,
  clicks INT DEFAULT 0,
  measured_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_content_performance_content ON content_performance(content_id);

-- Usage tracking (for billing)
CREATE TABLE IF NOT EXISTS usage_records (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  property_id UUID REFERENCES properties(id),
  type TEXT NOT NULL CHECK (type IN ('image_generated', 'post_scheduled', 'api_call')),
  quantity INT NOT NULL DEFAULT 1,
  period_start DATE NOT NULL,
  period_end DATE NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_usage_org_period ON usage_records(organization_id, period_start, period_end);
```

**Step 2: Run the migration**

Connect to the Spaces Neon database and execute the SQL. The implementer should check how the existing app runs migrations (look for a `migrate` script in `package.json`, a `/api/migrate` route, or manual execution).

```bash
# If there's a migrate script:
npm run migrate

# If manual:
psql $DATABASE_URL -f db/migrations/001-multi-tenant-schema.sql
```

**Step 3: Write tenant-scoped query helpers**

Add to `lib/db/index.ts` (or create `lib/db/tenant.ts`):

```typescript
import { neon } from '@neondatabase/serverless';

const sql = neon(process.env.DATABASE_URL!);

export async function getOrganizationForUser(userId: string) {
  const rows = await sql`
    SELECT o.* FROM organizations o
    JOIN members m ON m.organization_id = o.id
    WHERE m.user_id = ${userId}
    LIMIT 1
  `;
  return rows[0] || null;
}

export async function getPropertiesForOrg(organizationId: string) {
  return sql`
    SELECT * FROM properties
    WHERE organization_id = ${organizationId}
    ORDER BY created_at
  `;
}

export async function getMemberRole(userId: string, organizationId: string) {
  const rows = await sql`
    SELECT role FROM members
    WHERE user_id = ${userId} AND organization_id = ${organizationId}
  `;
  return rows[0]?.role || null;
}

export async function requireOrgAccess(userId: string, organizationId: string) {
  const role = await getMemberRole(userId, organizationId);
  if (!role) {
    throw new Error('Unauthorized: no access to this organization');
  }
  return role;
}

export async function requirePropertyAccess(userId: string, propertyId: string) {
  const rows = await sql`
    SELECT m.role FROM members m
    JOIN properties p ON p.organization_id = m.organization_id
    WHERE m.user_id = ${userId} AND p.id = ${propertyId}
  `;
  if (!rows[0]) {
    throw new Error('Unauthorized: no access to this property');
  }
  return rows[0].role;
}
```

**Step 4: Write tests for tenant scoping**

Create `__tests__/db/tenant.test.ts`:

```typescript
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
// Import test helpers that seed the DB with test data
// Adapt to the existing test setup in the repo

describe('tenant scoping', () => {
  it('getOrganizationForUser returns null for unknown user', async () => {
    const org = await getOrganizationForUser('nonexistent-id');
    expect(org).toBeNull();
  });

  it('requirePropertyAccess throws for unauthorized user', async () => {
    await expect(
      requirePropertyAccess('wrong-user', 'some-property')
    ).rejects.toThrow('Unauthorized');
  });

  it('getMemberRole returns correct role for org member', async () => {
    // Seed org + member, then verify
    // Adapt to existing test patterns
  });
});
```

**Step 5: Run tests**

```bash
npm test -- __tests__/db/tenant.test.ts
```

Expected: All tests pass.

**Step 6: Commit**

```bash
git add db/migrations/001-multi-tenant-schema.sql lib/db/tenant.ts __tests__/db/tenant.test.ts
git commit -m "feat: add multi-tenant schema — organizations, properties, members, content library

Adds 6 tables: organizations, properties, members, brand_profiles,
content_library, content_performance, usage_records. All queries
scoped by organization_id. Tenant access helpers with tests."
```

---

## Task 2: Onboarding Flow — Organization & Property Setup

**Files:**
- Create: `app/(dashboard)/onboarding/page.tsx`
- Create: `app/(dashboard)/onboarding/steps/CreateOrgStep.tsx`
- Create: `app/(dashboard)/onboarding/steps/AddPropertyStep.tsx`
- Create: `app/(dashboard)/onboarding/steps/BrandDnaStep.tsx`
- Create: `app/(dashboard)/onboarding/steps/ConnectChannelsStep.tsx` (placeholder for Phase 2)
- Create: `app/api/onboarding/organization/route.ts`
- Create: `app/api/onboarding/property/route.ts`
- Modify: NextAuth callbacks to check onboarding status

**Note:** Check the existing app's route structure first. It may use `(auth)` and `(dashboard)` route groups, or a different layout pattern. Adapt file paths to match.

**Step 1: Write the onboarding check**

After a user logs in via Google OAuth, check if they have an organization. If not, redirect to `/onboarding`.

Modify the NextAuth `signIn` or `session` callback (or add middleware):

```typescript
// In middleware.ts or NextAuth callback
// If user has no organization, redirect to /onboarding
import { getOrganizationForUser } from '@/lib/db/tenant';

// In session callback or middleware:
const org = await getOrganizationForUser(session.user.id);
if (!org && !req.nextUrl.pathname.startsWith('/onboarding')) {
  return NextResponse.redirect(new URL('/onboarding', req.url));
}
```

**Step 2: Write the organization creation API**

```typescript
// app/api/onboarding/organization/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { getServerSession } from 'next-auth';
import { authOptions } from '@/lib/auth'; // adapt import path
import { neon } from '@neondatabase/serverless';

const sql = neon(process.env.DATABASE_URL!);

export async function POST(req: NextRequest) {
  const session = await getServerSession(authOptions);
  if (!session?.user?.id) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const { name } = await req.json();
  if (!name || typeof name !== 'string' || name.trim().length < 2) {
    return NextResponse.json({ error: 'Name is required (min 2 chars)' }, { status: 400 });
  }

  const slug = name.trim().toLowerCase().replace(/[^a-z0-9]+/g, '-').replace(/^-|-$/g, '');

  // Check slug uniqueness
  const existing = await sql`SELECT id FROM organizations WHERE slug = ${slug}`;
  if (existing.length > 0) {
    return NextResponse.json({ error: 'Organization name already taken' }, { status: 409 });
  }

  const [org] = await sql`
    INSERT INTO organizations (name, slug)
    VALUES (${name.trim()}, ${slug})
    RETURNING *
  `;

  // Add the current user as owner
  await sql`
    INSERT INTO members (organization_id, user_id, role)
    VALUES (${org.id}, ${session.user.id}, 'owner')
  `;

  return NextResponse.json(org, { status: 201 });
}
```

**Step 3: Write the property creation API**

```typescript
// app/api/onboarding/property/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { getServerSession } from 'next-auth';
import { authOptions } from '@/lib/auth';
import { neon } from '@neondatabase/serverless';
import { requireOrgAccess } from '@/lib/db/tenant';

const sql = neon(process.env.DATABASE_URL!);

export async function POST(req: NextRequest) {
  const session = await getServerSession(authOptions);
  if (!session?.user?.id) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const { organizationId, name, type, location, timezone } = await req.json();

  await requireOrgAccess(session.user.id, organizationId);

  if (!name || typeof name !== 'string') {
    return NextResponse.json({ error: 'Property name is required' }, { status: 400 });
  }

  const validTypes = ['hotel', 'resort', 'airbnb', 'venue', 'restaurant', 'spa'];
  const propertyType = validTypes.includes(type) ? type : 'hotel';

  const [property] = await sql`
    INSERT INTO properties (organization_id, name, type, location, timezone)
    VALUES (${organizationId}, ${name.trim()}, ${propertyType}, ${location || null}, ${timezone || 'America/New_York'})
    RETURNING *
  `;

  // Create empty brand profile for this property
  await sql`
    INSERT INTO brand_profiles (property_id)
    VALUES (${property.id})
  `;

  return NextResponse.json(property, { status: 201 });
}
```

**Step 4: Write the onboarding UI**

Create `app/(dashboard)/onboarding/page.tsx` — a multi-step wizard:

```tsx
'use client';

import { useState } from 'react';
import CreateOrgStep from './steps/CreateOrgStep';
import AddPropertyStep from './steps/AddPropertyStep';
import BrandDnaStep from './steps/BrandDnaStep';

export default function OnboardingPage() {
  const [step, setStep] = useState(1);
  const [orgId, setOrgId] = useState<string | null>(null);
  const [propertyId, setPropertyId] = useState<string | null>(null);

  return (
    <div className="min-h-screen bg-gray-50 flex items-center justify-center p-4">
      <div className="w-full max-w-2xl">
        {/* Progress indicator */}
        <div className="flex items-center justify-center gap-2 mb-8">
          {[1, 2, 3].map((s) => (
            <div
              key={s}
              className={`h-2 w-16 rounded-full transition-colors ${
                s <= step ? 'bg-black' : 'bg-gray-200'
              }`}
            />
          ))}
        </div>

        {step === 1 && (
          <CreateOrgStep
            onComplete={(id) => {
              setOrgId(id);
              setStep(2);
            }}
          />
        )}
        {step === 2 && orgId && (
          <AddPropertyStep
            organizationId={orgId}
            onComplete={(id) => {
              setPropertyId(id);
              setStep(3);
            }}
          />
        )}
        {step === 3 && propertyId && (
          <BrandDnaStep
            propertyId={propertyId}
            onComplete={() => {
              window.location.href = '/dashboard';
            }}
          />
        )}
      </div>
    </div>
  );
}
```

**Step 5: Write the CreateOrgStep component**

```tsx
// app/(dashboard)/onboarding/steps/CreateOrgStep.tsx
'use client';

import { useState } from 'react';

export default function CreateOrgStep({ onComplete }: { onComplete: (orgId: string) => void }) {
  const [name, setName] = useState('');
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState('');

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setLoading(true);
    setError('');

    const res = await fetch('/api/onboarding/organization', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ name }),
    });

    if (!res.ok) {
      const data = await res.json();
      setError(data.error || 'Something went wrong');
      setLoading(false);
      return;
    }

    const org = await res.json();
    onComplete(org.id);
  };

  return (
    <form onSubmit={handleSubmit} className="bg-white rounded-2xl shadow-sm border p-8">
      <h1 className="text-2xl font-semibold mb-2">Welcome to Spaces</h1>
      <p className="text-gray-500 mb-6">
        Let&apos;s set up your organization. This is your team&apos;s home on Spaces.
      </p>

      <label className="block text-sm font-medium text-gray-700 mb-2">
        Organization name
      </label>
      <input
        type="text"
        value={name}
        onChange={(e) => setName(e.target.value)}
        placeholder="e.g., Seaside Resort Group"
        className="w-full border rounded-lg px-4 py-3 text-lg focus:outline-none focus:ring-2 focus:ring-black"
        required
        minLength={2}
      />
      {error && <p className="text-red-500 text-sm mt-2">{error}</p>}

      <button
        type="submit"
        disabled={loading || name.trim().length < 2}
        className="mt-6 w-full bg-black text-white rounded-lg py-3 text-lg font-medium
                   hover:bg-gray-800 disabled:bg-gray-300 disabled:cursor-not-allowed transition-colors"
      >
        {loading ? 'Creating...' : 'Continue'}
      </button>
    </form>
  );
}
```

**Step 6: Write the AddPropertyStep component**

```tsx
// app/(dashboard)/onboarding/steps/AddPropertyStep.tsx
'use client';

import { useState } from 'react';

const PROPERTY_TYPES = [
  { value: 'hotel', label: 'Hotel' },
  { value: 'resort', label: 'Resort' },
  { value: 'airbnb', label: 'Vacation Rental' },
  { value: 'venue', label: 'Event Venue' },
  { value: 'restaurant', label: 'Restaurant' },
  { value: 'spa', label: 'Spa & Wellness' },
];

const TIMEZONES = [
  'America/New_York',
  'America/Chicago',
  'America/Denver',
  'America/Los_Angeles',
  'America/Anchorage',
  'Pacific/Honolulu',
];

export default function AddPropertyStep({
  organizationId,
  onComplete,
}: {
  organizationId: string;
  onComplete: (propertyId: string) => void;
}) {
  const [name, setName] = useState('');
  const [type, setType] = useState('hotel');
  const [location, setLocation] = useState('');
  const [timezone, setTimezone] = useState('America/New_York');
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState('');

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setLoading(true);
    setError('');

    const res = await fetch('/api/onboarding/property', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ organizationId, name, type, location, timezone }),
    });

    if (!res.ok) {
      const data = await res.json();
      setError(data.error || 'Something went wrong');
      setLoading(false);
      return;
    }

    const property = await res.json();
    onComplete(property.id);
  };

  return (
    <form onSubmit={handleSubmit} className="bg-white rounded-2xl shadow-sm border p-8">
      <h1 className="text-2xl font-semibold mb-2">Add your first property</h1>
      <p className="text-gray-500 mb-6">
        Tell us about the property you want to market.
      </p>

      <div className="space-y-4">
        <div>
          <label className="block text-sm font-medium text-gray-700 mb-1">Property name</label>
          <input
            type="text"
            value={name}
            onChange={(e) => setName(e.target.value)}
            placeholder="e.g., The Grand Seaside Hotel"
            className="w-full border rounded-lg px-4 py-3 focus:outline-none focus:ring-2 focus:ring-black"
            required
          />
        </div>

        <div>
          <label className="block text-sm font-medium text-gray-700 mb-1">Type</label>
          <div className="grid grid-cols-3 gap-2">
            {PROPERTY_TYPES.map((pt) => (
              <button
                key={pt.value}
                type="button"
                onClick={() => setType(pt.value)}
                className={`border rounded-lg px-3 py-2 text-sm transition-colors ${
                  type === pt.value
                    ? 'border-black bg-black text-white'
                    : 'border-gray-200 hover:border-gray-400'
                }`}
              >
                {pt.label}
              </button>
            ))}
          </div>
        </div>

        <div>
          <label className="block text-sm font-medium text-gray-700 mb-1">Location</label>
          <input
            type="text"
            value={location}
            onChange={(e) => setLocation(e.target.value)}
            placeholder="e.g., Miami Beach, FL"
            className="w-full border rounded-lg px-4 py-3 focus:outline-none focus:ring-2 focus:ring-black"
          />
        </div>

        <div>
          <label className="block text-sm font-medium text-gray-700 mb-1">Timezone</label>
          <select
            value={timezone}
            onChange={(e) => setTimezone(e.target.value)}
            className="w-full border rounded-lg px-4 py-3 focus:outline-none focus:ring-2 focus:ring-black"
          >
            {TIMEZONES.map((tz) => (
              <option key={tz} value={tz}>{tz.replace('_', ' ')}</option>
            ))}
          </select>
        </div>
      </div>

      {error && <p className="text-red-500 text-sm mt-2">{error}</p>}

      <button
        type="submit"
        disabled={loading || !name.trim()}
        className="mt-6 w-full bg-black text-white rounded-lg py-3 text-lg font-medium
                   hover:bg-gray-800 disabled:bg-gray-300 disabled:cursor-not-allowed transition-colors"
      >
        {loading ? 'Creating...' : 'Continue'}
      </button>
    </form>
  );
}
```

**Step 7: Run the app and verify**

```bash
npm run dev
# Navigate to /onboarding
# Create an org, add a property, verify DB records
```

**Step 8: Commit**

```bash
git add app/api/onboarding/ app/(dashboard)/onboarding/ lib/db/tenant.ts
git commit -m "feat: add onboarding flow — org creation, property setup

Multi-step wizard: create organization → add property → brand DNA.
Auto-redirects new users. Tenant-scoped access helpers."
```

---

## Task 3: Brand DNA Wizard with Gemini Analysis

**Files:**
- Create: `app/(dashboard)/onboarding/steps/BrandDnaStep.tsx`
- Create: `app/api/brand/analyze/route.ts`
- Create: `app/api/brand/[propertyId]/route.ts`
- Reuse: Studio engine's `analyzeBrand` function (if already in the codebase) or port from `vael-creative-repo/dashboard/lib/studio-engine.ts`

**Step 1: Write the brand analysis API**

```typescript
// app/api/brand/analyze/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { getServerSession } from 'next-auth';
import { authOptions } from '@/lib/auth';
import { GoogleGenAI } from '@google/genai';

const ai = new GoogleGenAI({ apiKey: process.env.GEMINI_API_KEY! });

export async function POST(req: NextRequest) {
  const session = await getServerSession(authOptions);
  if (!session?.user?.id) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const { websiteUrl, description } = await req.json();

  if (!websiteUrl && !description) {
    return NextResponse.json({ error: 'Provide a website URL or description' }, { status: 400 });
  }

  const prompt = `You are a luxury hospitality brand strategist. Analyze this brand and return a JSON object with:
{
  "voice_description": "2-3 sentence description of the brand voice and tone",
  "visual_style": {
    "aesthetic": "one of: modern-minimal, classic-luxury, bohemian-resort, rustic-elegant, tropical-vibrant, urban-chic",
    "lighting": "preferred lighting style",
    "mood": "overall visual mood"
  },
  "target_audience": {
    "demographics": "age range, income level",
    "psychographics": "lifestyle, values, travel preferences",
    "guest_type": "business, leisure, family, couples, wellness"
  },
  "positioning": "one sentence positioning statement",
  "color_palette": ["#hex1", "#hex2", "#hex3", "#hex4"],
  "suggested_guardrails": [
    "Never mention competitor names",
    "Always maintain luxury-casual tone",
    "Avoid stock photo aesthetic"
  ]
}

${websiteUrl ? `Brand website: ${websiteUrl}` : ''}
${description ? `Brand description: ${description}` : ''}

Return ONLY valid JSON, no markdown fences.`;

  try {
    const response = await ai.models.generateContent({
      model: 'gemini-3-flash-preview',
      contents: prompt,
      config: {
        tools: websiteUrl ? [{ googleSearch: {} }] : undefined,
      },
    });

    const text = response.text || '';
    const json = JSON.parse(text.replace(/```json\n?/g, '').replace(/```\n?/g, '').trim());
    return NextResponse.json(json);
  } catch (error) {
    console.error('Brand analysis error:', error);
    return NextResponse.json({ error: 'Failed to analyze brand' }, { status: 500 });
  }
}
```

**Step 2: Write the brand profile save/update API**

```typescript
// app/api/brand/[propertyId]/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { getServerSession } from 'next-auth';
import { authOptions } from '@/lib/auth';
import { neon } from '@neondatabase/serverless';
import { requirePropertyAccess } from '@/lib/db/tenant';

const sql = neon(process.env.DATABASE_URL!);

export async function GET(req: NextRequest, { params }: { params: { propertyId: string } }) {
  const session = await getServerSession(authOptions);
  if (!session?.user?.id) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });

  await requirePropertyAccess(session.user.id, params.propertyId);

  const rows = await sql`SELECT * FROM brand_profiles WHERE property_id = ${params.propertyId}`;
  return NextResponse.json(rows[0] || null);
}

export async function PUT(req: NextRequest, { params }: { params: { propertyId: string } }) {
  const session = await getServerSession(authOptions);
  if (!session?.user?.id) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });

  await requirePropertyAccess(session.user.id, params.propertyId);

  const body = await req.json();
  const { voice_description, visual_style, target_audience, positioning, guardrails, color_palette, reviewed_by_human } = body;

  const [profile] = await sql`
    UPDATE brand_profiles SET
      voice_description = COALESCE(${voice_description}, voice_description),
      visual_style = COALESCE(${JSON.stringify(visual_style)}, visual_style),
      target_audience = COALESCE(${JSON.stringify(target_audience)}, target_audience),
      positioning = COALESCE(${positioning}, positioning),
      guardrails = COALESCE(${JSON.stringify(guardrails)}, guardrails),
      color_palette = COALESCE(${JSON.stringify(color_palette)}, color_palette),
      reviewed_by_human = COALESCE(${reviewed_by_human}, reviewed_by_human),
      updated_at = NOW()
    WHERE property_id = ${params.propertyId}
    RETURNING *
  `;

  return NextResponse.json(profile);
}
```

**Step 3: Write the BrandDnaStep component**

```tsx
// app/(dashboard)/onboarding/steps/BrandDnaStep.tsx
'use client';

import { useState } from 'react';

interface BrandProfile {
  voice_description: string;
  visual_style: { aesthetic: string; lighting: string; mood: string };
  target_audience: { demographics: string; psychographics: string; guest_type: string };
  positioning: string;
  color_palette: string[];
  suggested_guardrails: string[];
}

export default function BrandDnaStep({
  propertyId,
  onComplete,
}: {
  propertyId: string;
  onComplete: () => void;
}) {
  const [websiteUrl, setWebsiteUrl] = useState('');
  const [description, setDescription] = useState('');
  const [analyzing, setAnalyzing] = useState(false);
  const [saving, setSaving] = useState(false);
  const [profile, setProfile] = useState<BrandProfile | null>(null);
  const [guardrails, setGuardrails] = useState<string[]>([]);
  const [newGuardrail, setNewGuardrail] = useState('');
  const [error, setError] = useState('');

  const handleAnalyze = async () => {
    setAnalyzing(true);
    setError('');

    const res = await fetch('/api/brand/analyze', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ websiteUrl: websiteUrl || undefined, description: description || undefined }),
    });

    if (!res.ok) {
      setError('Analysis failed. Try adding a description instead.');
      setAnalyzing(false);
      return;
    }

    const data = await res.json();
    setProfile(data);
    setGuardrails(data.suggested_guardrails || []);
    setAnalyzing(false);
  };

  const handleSave = async () => {
    if (!profile) return;
    setSaving(true);

    await fetch(`/api/brand/${propertyId}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        voice_description: profile.voice_description,
        visual_style: profile.visual_style,
        target_audience: profile.target_audience,
        positioning: profile.positioning,
        color_palette: profile.color_palette,
        guardrails,
        generated_by_ai: true,
        reviewed_by_human: true,
      }),
    });

    onComplete();
  };

  const addGuardrail = () => {
    if (newGuardrail.trim()) {
      setGuardrails([...guardrails, newGuardrail.trim()]);
      setNewGuardrail('');
    }
  };

  const removeGuardrail = (index: number) => {
    setGuardrails(guardrails.filter((_, i) => i !== index));
  };

  if (!profile) {
    return (
      <div className="bg-white rounded-2xl shadow-sm border p-8">
        <h1 className="text-2xl font-semibold mb-2">Define your brand</h1>
        <p className="text-gray-500 mb-6">
          Share your website or describe your brand, and we&apos;ll build your profile automatically.
        </p>

        <div className="space-y-4">
          <div>
            <label className="block text-sm font-medium text-gray-700 mb-1">Website URL</label>
            <input
              type="url"
              value={websiteUrl}
              onChange={(e) => setWebsiteUrl(e.target.value)}
              placeholder="https://yourbrand.com"
              className="w-full border rounded-lg px-4 py-3 focus:outline-none focus:ring-2 focus:ring-black"
            />
          </div>
          <div>
            <label className="block text-sm font-medium text-gray-700 mb-1">
              Or describe your brand
            </label>
            <textarea
              value={description}
              onChange={(e) => setDescription(e.target.value)}
              placeholder="A luxury boutique hotel on Miami Beach with Art Deco architecture, targeting affluent couples aged 30-50..."
              rows={4}
              className="w-full border rounded-lg px-4 py-3 focus:outline-none focus:ring-2 focus:ring-black resize-none"
            />
          </div>
        </div>

        {error && <p className="text-red-500 text-sm mt-2">{error}</p>}

        <button
          onClick={handleAnalyze}
          disabled={analyzing || (!websiteUrl && !description)}
          className="mt-6 w-full bg-black text-white rounded-lg py-3 text-lg font-medium
                     hover:bg-gray-800 disabled:bg-gray-300 disabled:cursor-not-allowed transition-colors"
        >
          {analyzing ? 'Analyzing your brand...' : 'Analyze'}
        </button>
      </div>
    );
  }

  return (
    <div className="bg-white rounded-2xl shadow-sm border p-8">
      <h1 className="text-2xl font-semibold mb-2">Review your brand profile</h1>
      <p className="text-gray-500 mb-6">
        We&apos;ve analyzed your brand. Edit anything that doesn&apos;t look right.
      </p>

      <div className="space-y-6">
        {/* Voice */}
        <div>
          <label className="block text-sm font-medium text-gray-700 mb-1">Brand voice</label>
          <textarea
            value={profile.voice_description}
            onChange={(e) => setProfile({ ...profile, voice_description: e.target.value })}
            rows={3}
            className="w-full border rounded-lg px-4 py-3 focus:outline-none focus:ring-2 focus:ring-black resize-none"
          />
        </div>

        {/* Positioning */}
        <div>
          <label className="block text-sm font-medium text-gray-700 mb-1">Positioning</label>
          <input
            type="text"
            value={profile.positioning}
            onChange={(e) => setProfile({ ...profile, positioning: e.target.value })}
            className="w-full border rounded-lg px-4 py-3 focus:outline-none focus:ring-2 focus:ring-black"
          />
        </div>

        {/* Color palette */}
        <div>
          <label className="block text-sm font-medium text-gray-700 mb-1">Color palette</label>
          <div className="flex gap-2">
            {profile.color_palette.map((color, i) => (
              <div
                key={i}
                className="w-10 h-10 rounded-lg border"
                style={{ backgroundColor: color }}
                title={color}
              />
            ))}
          </div>
        </div>

        {/* Guardrails */}
        <div>
          <label className="block text-sm font-medium text-gray-700 mb-1">
            Guardrails — rules the AI must follow
          </label>
          <div className="space-y-2 mb-3">
            {guardrails.map((g, i) => (
              <div key={i} className="flex items-center gap-2 bg-gray-50 rounded-lg px-3 py-2">
                <span className="flex-1 text-sm">{g}</span>
                <button
                  onClick={() => removeGuardrail(i)}
                  className="text-gray-400 hover:text-red-500 text-sm"
                >
                  Remove
                </button>
              </div>
            ))}
          </div>
          <div className="flex gap-2">
            <input
              type="text"
              value={newGuardrail}
              onChange={(e) => setNewGuardrail(e.target.value)}
              onKeyDown={(e) => e.key === 'Enter' && (e.preventDefault(), addGuardrail())}
              placeholder="Add a guardrail..."
              className="flex-1 border rounded-lg px-3 py-2 text-sm focus:outline-none focus:ring-2 focus:ring-black"
            />
            <button
              onClick={addGuardrail}
              className="border rounded-lg px-4 py-2 text-sm hover:bg-gray-50"
            >
              Add
            </button>
          </div>
        </div>
      </div>

      <button
        onClick={handleSave}
        disabled={saving}
        className="mt-6 w-full bg-black text-white rounded-lg py-3 text-lg font-medium
                   hover:bg-gray-800 disabled:bg-gray-300 disabled:cursor-not-allowed transition-colors"
      >
        {saving ? 'Saving...' : 'Complete setup'}
      </button>
    </div>
  );
}
```

**Step 4: Test the full onboarding flow manually**

```bash
npm run dev
# 1. Sign in with Google OAuth
# 2. Should redirect to /onboarding
# 3. Create an org → add a property → run brand analysis → save
# 4. Verify DB: organizations, properties, members, brand_profiles all populated
```

**Step 5: Commit**

```bash
git add app/api/brand/ app/(dashboard)/onboarding/
git commit -m "feat: add brand DNA wizard with Gemini analysis

Gemini-powered brand analysis from URL or description. Extracts voice,
visual style, target audience, positioning, color palette, and suggested
guardrails. Customer reviews and edits before saving."
```

---

## Task 4: Content Library

**Files:**
- Create: `app/(dashboard)/[propertyId]/library/page.tsx`
- Create: `app/(dashboard)/[propertyId]/library/ContentGrid.tsx`
- Create: `app/api/content/[propertyId]/route.ts`
- Create: `app/api/content/[propertyId]/[contentId]/route.ts`
- Create: `app/api/content/[propertyId]/upload/route.ts`

**Step 1: Write the content listing API**

```typescript
// app/api/content/[propertyId]/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { getServerSession } from 'next-auth';
import { authOptions } from '@/lib/auth';
import { neon } from '@neondatabase/serverless';
import { requirePropertyAccess } from '@/lib/db/tenant';

const sql = neon(process.env.DATABASE_URL!);

export async function GET(req: NextRequest, { params }: { params: { propertyId: string } }) {
  const session = await getServerSession(authOptions);
  if (!session?.user?.id) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });

  await requirePropertyAccess(session.user.id, params.propertyId);

  const { searchParams } = new URL(req.url);
  const status = searchParams.get('status');
  const limit = Math.min(parseInt(searchParams.get('limit') || '50'), 100);
  const offset = parseInt(searchParams.get('offset') || '0');

  let rows;
  if (status) {
    rows = await sql`
      SELECT * FROM content_library
      WHERE property_id = ${params.propertyId} AND status = ${status}
      ORDER BY created_at DESC
      LIMIT ${limit} OFFSET ${offset}
    `;
  } else {
    rows = await sql`
      SELECT * FROM content_library
      WHERE property_id = ${params.propertyId}
      ORDER BY created_at DESC
      LIMIT ${limit} OFFSET ${offset}
    `;
  }

  return NextResponse.json(rows);
}
```

**Step 2: Write the content upload API**

```typescript
// app/api/content/[propertyId]/upload/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { getServerSession } from 'next-auth';
import { authOptions } from '@/lib/auth';
import { put } from '@vercel/blob';
import { neon } from '@neondatabase/serverless';
import { requirePropertyAccess } from '@/lib/db/tenant';

const sql = neon(process.env.DATABASE_URL!);

export async function POST(req: NextRequest, { params }: { params: { propertyId: string } }) {
  const session = await getServerSession(authOptions);
  if (!session?.user?.id) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });

  await requirePropertyAccess(session.user.id, params.propertyId);

  const formData = await req.formData();
  const file = formData.get('file') as File | null;
  if (!file) return NextResponse.json({ error: 'No file provided' }, { status: 400 });

  const allowedTypes = ['image/jpeg', 'image/png', 'image/webp', 'image/heic'];
  if (!allowedTypes.includes(file.type)) {
    return NextResponse.json({ error: 'Invalid file type' }, { status: 400 });
  }

  // Upload to Vercel Blob
  const blob = await put(`content/${params.propertyId}/${crypto.randomUUID()}.${file.type.split('/')[1]}`, file, {
    access: 'public',
    contentType: file.type,
  });

  // Insert into content_library
  const [content] = await sql`
    INSERT INTO content_library (property_id, type, url, source, status)
    VALUES (${params.propertyId}, 'image', ${blob.url}, 'upload', 'draft')
    RETURNING *
  `;

  return NextResponse.json(content, { status: 201 });
}
```

**Step 3: Write the content status update API**

```typescript
// app/api/content/[propertyId]/[contentId]/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { getServerSession } from 'next-auth';
import { authOptions } from '@/lib/auth';
import { neon } from '@neondatabase/serverless';
import { requirePropertyAccess } from '@/lib/db/tenant';

const sql = neon(process.env.DATABASE_URL!);

export async function PATCH(
  req: NextRequest,
  { params }: { params: { propertyId: string; contentId: string } }
) {
  const session = await getServerSession(authOptions);
  if (!session?.user?.id) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });

  await requirePropertyAccess(session.user.id, params.propertyId);

  const { status, tags, metadata } = await req.json();

  const validStatuses = ['draft', 'approved', 'posted', 'archived'];
  if (status && !validStatuses.includes(status)) {
    return NextResponse.json({ error: 'Invalid status' }, { status: 400 });
  }

  const [updated] = await sql`
    UPDATE content_library SET
      status = COALESCE(${status}, status),
      tags = COALESCE(${tags ? JSON.stringify(tags) : null}::text[], tags),
      metadata = COALESCE(${metadata ? JSON.stringify(metadata) : null}::jsonb, metadata)
    WHERE id = ${params.contentId} AND property_id = ${params.propertyId}
    RETURNING *
  `;

  if (!updated) return NextResponse.json({ error: 'Not found' }, { status: 404 });
  return NextResponse.json(updated);
}

export async function DELETE(
  req: NextRequest,
  { params }: { params: { propertyId: string; contentId: string } }
) {
  const session = await getServerSession(authOptions);
  if (!session?.user?.id) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });

  await requirePropertyAccess(session.user.id, params.propertyId);

  await sql`
    DELETE FROM content_library
    WHERE id = ${params.contentId} AND property_id = ${params.propertyId}
  `;

  return NextResponse.json({ success: true });
}
```

**Step 4: Write the ContentGrid UI**

```tsx
// app/(dashboard)/[propertyId]/library/ContentGrid.tsx
'use client';

import { useState, useCallback } from 'react';
import Image from 'next/image';

interface ContentItem {
  id: string;
  url: string;
  status: string;
  source: string;
  tags: string[];
  created_at: string;
}

export default function ContentGrid({
  propertyId,
  initialContent,
}: {
  propertyId: string;
  initialContent: ContentItem[];
}) {
  const [content, setContent] = useState(initialContent);
  const [filter, setFilter] = useState<string | null>(null);
  const [uploading, setUploading] = useState(false);

  const filtered = filter ? content.filter((c) => c.status === filter) : content;

  const handleUpload = useCallback(async (files: FileList) => {
    setUploading(true);
    const uploaded: ContentItem[] = [];

    for (const file of Array.from(files)) {
      const formData = new FormData();
      formData.append('file', file);

      const res = await fetch(`/api/content/${propertyId}/upload`, {
        method: 'POST',
        body: formData,
      });

      if (res.ok) {
        const item = await res.json();
        uploaded.push(item);
      }
    }

    setContent([...uploaded, ...content]);
    setUploading(false);
  }, [propertyId, content]);

  const handleStatusChange = async (contentId: string, status: string) => {
    const res = await fetch(`/api/content/${propertyId}/${contentId}`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ status }),
    });

    if (res.ok) {
      setContent(content.map((c) => (c.id === contentId ? { ...c, status } : c)));
    }
  };

  return (
    <div>
      {/* Filter tabs */}
      <div className="flex gap-2 mb-6">
        {[null, 'draft', 'approved', 'posted', 'archived'].map((f) => (
          <button
            key={f ?? 'all'}
            onClick={() => setFilter(f)}
            className={`px-4 py-2 rounded-lg text-sm transition-colors ${
              filter === f ? 'bg-black text-white' : 'bg-gray-100 hover:bg-gray-200'
            }`}
          >
            {f ?? 'All'} ({f ? content.filter((c) => c.status === f).length : content.length})
          </button>
        ))}
      </div>

      {/* Upload zone */}
      <label className="block border-2 border-dashed rounded-xl p-8 text-center cursor-pointer hover:border-gray-400 transition-colors mb-6">
        <input
          type="file"
          multiple
          accept="image/jpeg,image/png,image/webp"
          className="hidden"
          onChange={(e) => e.target.files && handleUpload(e.target.files)}
        />
        <p className="text-gray-500">
          {uploading ? 'Uploading...' : 'Drop images here or click to upload'}
        </p>
      </label>

      {/* Grid */}
      <div className="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-4">
        {filtered.map((item) => (
          <div key={item.id} className="group relative aspect-square rounded-xl overflow-hidden border">
            <Image
              src={item.url}
              alt=""
              fill
              className="object-cover"
              sizes="(max-width: 768px) 50vw, 25vw"
            />
            <div className="absolute inset-0 bg-black/0 group-hover:bg-black/40 transition-colors flex items-end p-3">
              <div className="hidden group-hover:flex gap-1 w-full">
                {item.status === 'draft' && (
                  <button
                    onClick={() => handleStatusChange(item.id, 'approved')}
                    className="flex-1 bg-white text-black text-xs rounded px-2 py-1"
                  >
                    Approve
                  </button>
                )}
                <button
                  onClick={() => handleStatusChange(item.id, 'archived')}
                  className="bg-white/80 text-black text-xs rounded px-2 py-1"
                >
                  Archive
                </button>
              </div>
            </div>
            <span className={`absolute top-2 right-2 text-xs px-2 py-0.5 rounded-full ${
              item.status === 'approved' ? 'bg-green-100 text-green-700' :
              item.status === 'posted' ? 'bg-blue-100 text-blue-700' :
              item.status === 'archived' ? 'bg-gray-100 text-gray-500' :
              'bg-yellow-100 text-yellow-700'
            }`}>
              {item.status}
            </span>
          </div>
        ))}
      </div>
    </div>
  );
}
```

**Step 5: Wire up the library page**

```tsx
// app/(dashboard)/[propertyId]/library/page.tsx
import { getServerSession } from 'next-auth';
import { authOptions } from '@/lib/auth';
import { neon } from '@neondatabase/serverless';
import { requirePropertyAccess } from '@/lib/db/tenant';
import ContentGrid from './ContentGrid';
import { redirect } from 'next/navigation';

const sql = neon(process.env.DATABASE_URL!);

export default async function LibraryPage({ params }: { params: { propertyId: string } }) {
  const session = await getServerSession(authOptions);
  if (!session?.user?.id) redirect('/auth/signin');

  await requirePropertyAccess(session.user.id, params.propertyId);

  const content = await sql`
    SELECT * FROM content_library
    WHERE property_id = ${params.propertyId}
    ORDER BY created_at DESC
    LIMIT 100
  `;

  return (
    <div className="p-6">
      <h1 className="text-2xl font-semibold mb-6">Content Library</h1>
      <ContentGrid propertyId={params.propertyId} initialContent={content} />
    </div>
  );
}
```

**Step 6: Test upload and status flows**

```bash
npm run dev
# Navigate to /[propertyId]/library
# Upload images, verify they appear as 'draft'
# Approve an image, verify status changes
# Check Vercel Blob has the files
```

**Step 7: Commit**

```bash
git add app/api/content/ app/(dashboard)/[propertyId]/library/
git commit -m "feat: add content library — upload, filter, status management

Image upload to Vercel Blob, content grid with status filtering
(draft/approved/posted/archived), inline approve and archive actions.
All queries tenant-scoped via requirePropertyAccess."
```

---

## Task 5: Studio → Content Library Integration

**Files:**
- Modify: The existing Studio generation flow (find the generate route and post-generation handler)
- Goal: When Studio generates images, they automatically land in `content_library` with status `draft`

**Step 1: Identify the Studio generate flow**

Read the existing Studio generation route (likely at `app/api/studio/generate/route.ts` or similar). Find where generated images are saved (probably to Vercel Blob + a sessions table).

**Step 2: Add content_library insertion after generation**

After each image is generated and saved to Blob, also insert into `content_library`:

```typescript
// Add after the existing Blob upload in the generate route
await sql`
  INSERT INTO content_library (property_id, type, url, source, status, metadata)
  VALUES (
    ${propertyId},
    'image',
    ${blobUrl},
    'studio',
    'draft',
    ${JSON.stringify({
      session_id: sessionId,
      generation_model: 'gemini-3-pro-image-preview',
      mode: studioMode,
      prompt_used: promptUsed,
    })}
  )
`;
```

**Step 3: Verify the integration**

```bash
npm run dev
# Generate images in Studio
# Navigate to Content Library
# Verify the generated images appear with source='studio' and status='draft'
```

**Step 4: Commit**

```bash
git add app/api/studio/generate/route.ts  # adapt path
git commit -m "feat: bridge Studio → Content Library

Generated images automatically land in content_library as drafts.
Preserves generation metadata (session, model, mode, prompt)."
```

---

## Task 6: Stripe Billing — Starter Tier

**Files:**
- Create: `lib/stripe.ts`
- Create: `app/api/billing/checkout/route.ts`
- Create: `app/api/billing/portal/route.ts`
- Create: `app/api/webhooks/stripe/route.ts`
- Create: `lib/billing/usage.ts`
- Modify: `app/(dashboard)/settings/billing/page.tsx` (or create)

**Step 1: Install Stripe**

```bash
npm install stripe
```

**Step 2: Create Stripe products and prices**

This is a manual step in the Stripe Dashboard (or via CLI):

```bash
# Create the product
stripe products create --name="Vael Spaces Starter" --description="Content Studio — 80 images/month"

# Create the price (monthly recurring)
stripe prices create \
  --product=prod_XXXXX \
  --unit-amount=14900 \
  --currency=usd \
  --recurring-interval=month
```

Save the price ID to env vars: `STRIPE_STARTER_PRICE_ID`

Also set: `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`

**Step 3: Write the Stripe utility**

```typescript
// lib/stripe.ts
import Stripe from 'stripe';

export const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2024-12-18.acacia',
});

export const PLANS = {
  starter: {
    name: 'Starter',
    priceId: process.env.STRIPE_STARTER_PRICE_ID!,
    limits: { images: 80, posts: 0, properties: 1, members: 1 },
  },
  growth: {
    name: 'Growth',
    priceId: process.env.STRIPE_GROWTH_PRICE_ID || '',
    limits: { images: 200, posts: 30, properties: 3, members: 3 },
  },
  pro: {
    name: 'Pro',
    priceId: process.env.STRIPE_PRO_PRICE_ID || '',
    limits: { images: 500, posts: 90, properties: 10, members: 10 },
  },
} as const;

export type PlanId = keyof typeof PLANS;
```

**Step 4: Write the checkout route**

```typescript
// app/api/billing/checkout/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { getServerSession } from 'next-auth';
import { authOptions } from '@/lib/auth';
import { stripe, PLANS } from '@/lib/stripe';
import { neon } from '@neondatabase/serverless';
import { getOrganizationForUser } from '@/lib/db/tenant';

const sql = neon(process.env.DATABASE_URL!);

export async function POST(req: NextRequest) {
  const session = await getServerSession(authOptions);
  if (!session?.user?.id) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });

  const { plan } = await req.json();
  if (!plan || !PLANS[plan as keyof typeof PLANS]) {
    return NextResponse.json({ error: 'Invalid plan' }, { status: 400 });
  }

  const org = await getOrganizationForUser(session.user.id);
  if (!org) return NextResponse.json({ error: 'No organization' }, { status: 400 });

  // Get or create Stripe customer
  let customerId = org.stripe_customer_id;
  if (!customerId) {
    const customer = await stripe.customers.create({
      email: session.user.email!,
      metadata: { organization_id: org.id },
    });
    customerId = customer.id;
    await sql`
      UPDATE organizations SET stripe_customer_id = ${customerId} WHERE id = ${org.id}
    `;
  }

  const checkoutSession = await stripe.checkout.sessions.create({
    customer: customerId,
    mode: 'subscription',
    line_items: [{ price: PLANS[plan as keyof typeof PLANS].priceId, quantity: 1 }],
    success_url: `${process.env.NEXTAUTH_URL}/dashboard?billing=success`,
    cancel_url: `${process.env.NEXTAUTH_URL}/settings/billing`,
    subscription_data: {
      trial_period_days: 14,
      metadata: { organization_id: org.id },
    },
  });

  return NextResponse.json({ url: checkoutSession.url });
}
```

**Step 5: Write the webhook handler**

```typescript
// app/api/webhooks/stripe/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { stripe } from '@/lib/stripe';
import { neon } from '@neondatabase/serverless';
import { headers } from 'next/headers';

const sql = neon(process.env.DATABASE_URL!);

export async function POST(req: NextRequest) {
  const body = await req.text();
  const sig = (await headers()).get('stripe-signature')!;

  let event;
  try {
    event = stripe.webhooks.constructEvent(body, sig, process.env.STRIPE_WEBHOOK_SECRET!);
  } catch {
    return NextResponse.json({ error: 'Invalid signature' }, { status: 400 });
  }

  switch (event.type) {
    case 'checkout.session.completed': {
      const session = event.data.object;
      const orgId = session.subscription
        ? (await stripe.subscriptions.retrieve(session.subscription as string)).metadata.organization_id
        : session.metadata?.organization_id;

      if (orgId) {
        await sql`
          UPDATE organizations SET
            subscription_status = 'active',
            subscription_plan = 'starter',
            updated_at = NOW()
          WHERE id = ${orgId}
        `;
      }
      break;
    }

    case 'customer.subscription.updated': {
      const sub = event.data.object;
      const orgId = sub.metadata.organization_id;
      const status = sub.status;

      if (orgId) {
        await sql`
          UPDATE organizations SET
            subscription_status = ${status},
            updated_at = NOW()
          WHERE id = ${orgId}
        `;
      }
      break;
    }

    case 'customer.subscription.deleted': {
      const sub = event.data.object;
      const orgId = sub.metadata.organization_id;

      if (orgId) {
        await sql`
          UPDATE organizations SET
            subscription_status = 'canceled',
            updated_at = NOW()
          WHERE id = ${orgId}
        `;
      }
      break;
    }
  }

  return NextResponse.json({ received: true });
}
```

**Step 6: Write the usage tracking utility**

```typescript
// lib/billing/usage.ts
import { neon } from '@neondatabase/serverless';
import { PLANS, PlanId } from '@/lib/stripe';

const sql = neon(process.env.DATABASE_URL!);

export async function trackUsage(organizationId: string, type: string, propertyId?: string) {
  const now = new Date();
  const periodStart = new Date(now.getFullYear(), now.getMonth(), 1).toISOString().split('T')[0];
  const periodEnd = new Date(now.getFullYear(), now.getMonth() + 1, 0).toISOString().split('T')[0];

  await sql`
    INSERT INTO usage_records (organization_id, property_id, type, quantity, period_start, period_end)
    VALUES (${organizationId}, ${propertyId || null}, ${type}, 1, ${periodStart}, ${periodEnd})
  `;
}

export async function getUsage(organizationId: string, type: string): Promise<number> {
  const now = new Date();
  const periodStart = new Date(now.getFullYear(), now.getMonth(), 1).toISOString().split('T')[0];

  const rows = await sql`
    SELECT COALESCE(SUM(quantity), 0) as total
    FROM usage_records
    WHERE organization_id = ${organizationId}
      AND type = ${type}
      AND period_start = ${periodStart}
  `;

  return parseInt(rows[0].total);
}

export async function checkLimit(organizationId: string, type: string, plan: PlanId): Promise<boolean> {
  const usage = await getUsage(organizationId, type);
  const limitKey = type === 'image_generated' ? 'images' : type === 'post_scheduled' ? 'posts' : 'images';
  const limit = PLANS[plan].limits[limitKey];
  return usage < limit;
}
```

**Step 7: Wire usage tracking into Studio generation**

In the Studio generate route, after creating a content_library entry:

```typescript
import { trackUsage } from '@/lib/billing/usage';

// After generating each image:
await trackUsage(organizationId, 'image_generated', propertyId);
```

**Step 8: Test the billing flow**

```bash
# Set up Stripe CLI for local webhook testing:
stripe listen --forward-to localhost:3000/api/webhooks/stripe

npm run dev
# Navigate to billing settings
# Click subscribe → Stripe checkout → use test card 4242...
# Verify webhook fires → org subscription_status updates
# Generate images → verify usage_records increment
```

**Step 9: Commit**

```bash
git add lib/stripe.ts lib/billing/ app/api/billing/ app/api/webhooks/stripe/
git commit -m "feat: add Stripe billing — Starter tier, usage tracking, webhooks

Stripe checkout with 14-day trial, webhook handler for subscription
lifecycle events, usage metering per billing period, limit enforcement."
```

---

## Task 7: API Bridge to Agency Dashboard

**Files:**
- Create: `app/api/bridge/sync/route.ts`
- Create: `lib/bridge.ts`

**Step 1: Design the bridge**

The bridge is a simple API endpoint that the agency dashboard calls to sync data. It uses a shared API key for authentication (not user sessions — this is server-to-server).

```typescript
// lib/bridge.ts
export function validateBridgeKey(request: Request): boolean {
  const key = request.headers.get('x-bridge-api-key');
  return key === process.env.BRIDGE_API_KEY;
}
```

**Step 2: Write the sync endpoint**

```typescript
// app/api/bridge/sync/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { validateBridgeKey } from '@/lib/bridge';
import { neon } from '@neondatabase/serverless';

const sql = neon(process.env.DATABASE_URL!);

// Dashboard pushes brand DNA and client data to Spaces
export async function POST(req: NextRequest) {
  if (!validateBridgeKey(req)) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const { action, data } = await req.json();

  switch (action) {
    case 'sync_brand': {
      const { propertyId, brandProfile } = data;
      await sql`
        UPDATE brand_profiles SET
          voice_description = ${brandProfile.voice_description},
          visual_style = ${JSON.stringify(brandProfile.visual_style)},
          target_audience = ${JSON.stringify(brandProfile.target_audience)},
          positioning = ${brandProfile.positioning},
          guardrails = ${JSON.stringify(brandProfile.guardrails)},
          color_palette = ${JSON.stringify(brandProfile.color_palette)},
          updated_at = NOW()
        WHERE property_id = ${propertyId}
      `;
      return NextResponse.json({ success: true });
    }

    case 'sync_content': {
      const { propertyId, assets } = data;
      for (const asset of assets) {
        await sql`
          INSERT INTO content_library (property_id, type, url, source, status, metadata)
          VALUES (${propertyId}, 'image', ${asset.url}, 'agency', 'approved', ${JSON.stringify(asset.metadata || {})})
          ON CONFLICT DO NOTHING
        `;
      }
      return NextResponse.json({ success: true, synced: assets.length });
    }

    default:
      return NextResponse.json({ error: 'Unknown action' }, { status: 400 });
  }
}

// Spaces exposes data for the dashboard to pull
export async function GET(req: NextRequest) {
  if (!validateBridgeKey(req)) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const { searchParams } = new URL(req.url);
  const propertyId = searchParams.get('propertyId');
  const dataType = searchParams.get('type');

  if (!propertyId) return NextResponse.json({ error: 'propertyId required' }, { status: 400 });

  switch (dataType) {
    case 'usage': {
      const rows = await sql`
        SELECT type, SUM(quantity) as total
        FROM usage_records
        WHERE property_id = ${propertyId}
        GROUP BY type
      `;
      return NextResponse.json(rows);
    }

    case 'content_stats': {
      const rows = await sql`
        SELECT status, COUNT(*) as count
        FROM content_library
        WHERE property_id = ${propertyId}
        GROUP BY status
      `;
      return NextResponse.json(rows);
    }

    default:
      return NextResponse.json({ error: 'Unknown type' }, { status: 400 });
  }
}
```

**Step 3: Set the BRIDGE_API_KEY env var**

```bash
# Generate a secure key
openssl rand -base64 32
# Add to Vercel: BRIDGE_API_KEY=<generated-key>
# Add to the agency dashboard's env: SPACES_BRIDGE_API_KEY=<same-key>
# Add to the agency dashboard's env: SPACES_BRIDGE_URL=https://spaces.vaelcreative.com/api/bridge/sync
```

**Step 4: Commit**

```bash
git add lib/bridge.ts app/api/bridge/
git commit -m "feat: add API bridge for agency dashboard sync

Server-to-server API with shared key auth. Supports syncing brand
profiles and content from dashboard to Spaces, and pulling usage
stats and content metrics from Spaces to dashboard."
```

---

## Task 8: Dashboard Navigation & Property Switcher

**Files:**
- Create: `app/(dashboard)/dashboard/page.tsx`
- Create: `app/(dashboard)/components/Sidebar.tsx`
- Create: `app/(dashboard)/components/PropertySwitcher.tsx`
- Modify: `app/(dashboard)/layout.tsx`

This task depends heavily on the existing app's layout. The implementer should:

1. Read the existing layout structure
2. Add a sidebar (or modify the existing nav) with links to: Content Studio, Library, Brand Profile, Settings
3. Add a property switcher dropdown in the header for multi-property orgs
4. Add a dashboard home page that shows: usage stats, recent content, quick actions

**Step 1: Build the property switcher**

```tsx
// app/(dashboard)/components/PropertySwitcher.tsx
'use client';

import { useState } from 'react';
import { useRouter } from 'next/navigation';

interface Property {
  id: string;
  name: string;
  type: string;
  location: string;
}

export default function PropertySwitcher({
  properties,
  currentPropertyId,
}: {
  properties: Property[];
  currentPropertyId: string;
}) {
  const [open, setOpen] = useState(false);
  const router = useRouter();
  const current = properties.find((p) => p.id === currentPropertyId);

  return (
    <div className="relative">
      <button
        onClick={() => setOpen(!open)}
        className="flex items-center gap-2 px-3 py-2 rounded-lg hover:bg-gray-100 transition-colors"
      >
        <span className="font-medium">{current?.name || 'Select property'}</span>
        <svg className="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M19 9l-7 7-7-7" />
        </svg>
      </button>

      {open && (
        <div className="absolute top-full left-0 mt-1 w-64 bg-white border rounded-xl shadow-lg z-50">
          {properties.map((p) => (
            <button
              key={p.id}
              onClick={() => {
                router.push(`/${p.id}/library`);
                setOpen(false);
              }}
              className={`w-full text-left px-4 py-3 hover:bg-gray-50 first:rounded-t-xl last:rounded-b-xl ${
                p.id === currentPropertyId ? 'bg-gray-50' : ''
              }`}
            >
              <div className="font-medium">{p.name}</div>
              <div className="text-sm text-gray-500">{p.type} · {p.location}</div>
            </button>
          ))}
        </div>
      )}
    </div>
  );
}
```

**Step 2: Adapt to existing layout, test navigation**

```bash
npm run dev
# Verify: property switcher works, sidebar links route correctly,
# dashboard home shows usage stats
```

**Step 3: Commit**

```bash
git add app/(dashboard)/dashboard/ app/(dashboard)/components/
git commit -m "feat: add dashboard layout with property switcher and sidebar nav

Property switcher for multi-property orgs, sidebar with links to
Studio, Library, Brand, Settings. Dashboard home with usage stats."
```

---

## Task 9: Middleware — Auth & Tenant Security

**Files:**
- Create: `middleware.ts`

**Step 1: Write the middleware**

```typescript
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';
import { getToken } from 'next-auth/jwt';

// Routes that don't require authentication
const PUBLIC_ROUTES = [
  '/',                    // Landing page
  '/auth',                // Auth pages
  '/api/auth',            // NextAuth routes
  '/api/webhooks/stripe', // Stripe webhooks (signature-verified)
  '/api/bridge',          // Bridge API (key-verified)
  '/api/cron',            // Cron jobs (secret-verified)
  '/api/health',          // Health checks
];

export async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // Allow public routes
  if (PUBLIC_ROUTES.some((route) => pathname.startsWith(route))) {
    return NextResponse.next();
  }

  // Allow static assets
  if (pathname.startsWith('/_next') || pathname.startsWith('/favicon') || pathname.includes('.')) {
    return NextResponse.next();
  }

  // Check for valid session
  const token = await getToken({ req: request, secret: process.env.NEXTAUTH_SECRET });

  if (!token) {
    if (pathname.startsWith('/api/')) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
    }
    return NextResponse.redirect(new URL('/auth/signin', request.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico).*)'],
};
```

**Step 2: Test auth protection**

```bash
npm run dev
# Test: unauthenticated request to /api/content/xxx returns 401
# Test: unauthenticated visit to /dashboard redirects to /auth/signin
# Test: authenticated requests pass through
# Test: /api/webhooks/stripe is accessible without auth
# Test: landing page (/) is accessible without auth
```

**Step 3: Commit**

```bash
git add middleware.ts
git commit -m "feat: add auth middleware — deny-by-default for all routes

Blanket auth check for all routes except explicitly public ones
(landing, auth flows, webhooks, cron, health). API routes return
401, page routes redirect to signin."
```

---

## Phase 1 Exit Criteria Checklist

After completing all 9 tasks, verify:

- [ ] New user can sign up via Google OAuth
- [ ] New user is redirected to onboarding wizard
- [ ] Can create an organization and add a property
- [ ] Brand DNA wizard analyzes a website URL and generates profile
- [ ] Customer can edit brand voice, positioning, guardrails, and save
- [ ] Content Library shows images with status filtering
- [ ] Can upload images to Content Library
- [ ] Studio-generated images land in Content Library as drafts
- [ ] Stripe Starter tier checkout works with 14-day trial
- [ ] Usage tracking increments on image generation
- [ ] Subscription lifecycle webhooks update org status
- [ ] API bridge accepts server-to-server sync requests
- [ ] Middleware blocks unauthenticated access to all protected routes
- [ ] Multi-property navigation works (property switcher)
- [ ] All API routes validate tenant access via `requirePropertyAccess`

When all boxes are checked, Phase 1 is complete. Phase 2 (Social Agent) builds on this foundation.

# Color-Grade

Figma color grading agent with privacy-first training data pipeline.

## Architecture

**Raw feedback stays private. Only approved training profiles are published.**

```
Figma Plugin (credential-free)
    ↓ 
Supabase (raw user feedback, private)
    ↓ (consent-flagged records only)
Calibration Service (quality gate)
    ├─ Compare against frozen benchmark
    ├─ Run regression tests
    ├─ Generate training candidate
    ↓ (if passes)
GitHub Repo (approved profiles only, public)
    └─ /profiles/approved/{version}.json
    └─ /benchmarks/frozen-{date}.json (reference)
    ↓ (credential-free)
Figma Plugin (fetches approved profile)
```

## Data Privacy

- **Supabase (private):** Raw submissions, photos, session logs, user IDs
- **GitHub (public):** Only anonymized, approved training profiles
- **Consent:** Users opt-in to training via plugin UI
- **Rollback:** Previous approved versions always available in `/profiles/archive/`

## Implementation Status

### ✅ Implemented

- Supabase table `plugin_feedback` with consent flag
- RLS policies restrict raw data access
- `.env.example` for credential storage
- Sync script foundation
- Public approved-profile endpoint definition
- Calibration script template

### 🔄 Proposed (Next Phase)

- GitHub Actions workflow for scheduled calibration
- Benchmark comparison logic refinement
- Regression test suite expansion
- Approval gate automation
- Monitoring/alerting for calibration failures
- Automated rollback trigger

---

## Setup

### 1. Supabase (Private Storage)

Run in Supabase SQL editor:

```sql
create table public.plugin_feedback (
  id uuid default gen_random_uuid() primary key,
  created_at timestamptz default now(),
  user_id text,
  session_id text not null,
  event_type text not null,
  palette jsonb,
  rating integer,
  feedback text,
  design_context jsonb default '{}'::jsonb,
  consent_training boolean default false,
  metadata jsonb default '{}'::jsonb
);

alter table public.plugin_feedback enable row level security;

-- Allow plugin to insert (anon)
create policy "Allow plugin inserts"
on public.plugin_feedback for insert
with check (true);

-- Only service role can read raw data
create policy "Service role reads only"
on public.plugin_feedback for select
using (auth.role() = 'service_role');

-- Indexes for performance
create index idx_plugin_feedback_consent on public.plugin_feedback(consent_training);
create index idx_plugin_feedback_created_at on public.plugin_feedback(created_at);
```

### 2. Figma Plugin (Credential-Free)

```javascript
import { createClient } from "@supabase/supabase-js";

const SUPABASE_URL = "https://YOUR-PROJECT.supabase.co";
const SUPABASE_ANON_KEY = "YOUR_ANON_KEY";

const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY);

let sessionId = null;

function getSessionId() {
  if (!sessionId) {
    sessionId = `session-${Date.now()}-${Math.random().toString(16).slice(2)}`;
  }
  return sessionId;
}

async function saveFeedback({
  palette,
  rating,
  feedback,
  designContext,
  consentToTraining
}) {
  const { error } = await supabase.from("plugin_feedback").insert([
    {
      user_id: null, // Anonymous
      session_id: getSessionId(),
      event_type: "palette_saved",
      palette,
      rating,
      feedback,
      design_context: designContext,
      consent_training: consentToTraining, // ← User opts in
      metadata: { source: "figma-plugin", version: "1.0" }
    }
  ]);

  if (error) {
    console.error("Feedback save error:", error);
    return false;
  }

  return true;
}

// Load latest approved profile (public, credential-free)
async function fetchApprovedProfile() {
  try {
    const response = await fetch(
      "https://raw.githubusercontent.com/johnpauljay/Color-Grade/main/profiles/approved/latest.json"
    );
    if (!response.ok) throw new Error("Failed to fetch approved profile");
    return await response.json();
  } catch (error) {
    console.error("Profile fetch error:", error);
    return null;
  }
}

// Initialize plugin with approved profiles
async function initializePlugin() {
  const approved = await fetchApprovedProfile();
  if (approved) {
    console.log(`Loaded approved profile v${approved.version}`);
    return approved.profiles || [];
  }
  return [];
}
```

### 3. Environment Variables

```env name=.env.example
# Supabase (private, for calibration service only)
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_SERVICE_ROLE_KEY=your_service_role_key

# GitHub (private, for calibration service only)
GITHUB_OWNER=johnpauljay
GITHUB_REPO=Color-Grade
GITHUB_TOKEN=github_pat_xxx
GITHUB_BRANCH=main

# Calibration settings
BENCHMARK_VERSION=frozen-2026-10-11
REGRESSION_THRESHOLD=0.95
QUALITY_MIN_SAMPLES=50
```

### 4. Calibration Process (Scheduled Backend)

Place in `scripts/calibration.js`:

```javascript
require("dotenv").config();
const { createClient } = require("@supabase/supabase-js");
const axios = require("axios");

const supabase = createClient(
  process.env.SUPABASE_URL,
  process.env.SUPABASE_SERVICE_ROLE_KEY,
  { auth: { persistSession: false } }
);

async function calibrate() {
  console.log("📊 Starting calibration pipeline...\n");

  try {
    // Step 1: Fetch consent-flagged records only
    const { data: rawFeedback, error } = await supabase
      .from("plugin_feedback")
      .select("*")
      .eq("consent_training", true)
      .order("created_at", { ascending: false });

    if (error) throw error;

    if (rawFeedback.length < process.env.QUALITY_MIN_SAMPLES) {
      throw new Error(
        `Insufficient samples: ${rawFeedback.length} < ${process.env.QUALITY_MIN_SAMPLES}`
      );
    }

    console.log(`✅ Loaded ${rawFeedback.length} consent-flagged records\n`);

    // Step 2: Aggregate & anonymize into training candidate
    const candidate = aggregateCandidate(rawFeedback);
    console.log(
      `🔄 Generated candidate profile (${candidate.profiles.length} profiles)\n`
    );

    // Step 3: Load frozen benchmark
    const benchmark = await loadBenchmark(process.env.BENCHMARK_VERSION);
    console.log(`📌 Loaded frozen benchmark: ${benchmark.version}\n`);

    // Step 4: Regression test
    const regressionScore = compareAgainstBenchmark(candidate, benchmark);
    console.log(`📈 Regression score: ${regressionScore.toFixed(4)}\n`);

    if (regressionScore < process.env.REGRESSION_THRESHOLD) {
      throw new Error(
        `Regression detected: ${regressionScore} < ${process.env.REGRESSION_THRESHOLD}`
      );
    }

    // Step 5: Publish to GitHub
    const version = `approved-${new Date().toISOString().split("T")[0]}`;
    await publishApprovedProfile(candidate, version);

    console.log(`✅ Published ${version} to GitHub\n`);
    console.log(`📊 Metrics:`);
    console.log(`   - Samples: ${candidate.totalInputs}`);
    console.log(`   - Avg Rating: ${candidate.stats.avgRating}`);
    console.log(`   - Regression: ${regressionScore.toFixed(4)}`);

  } catch (error) {
    console.error("❌ Calibration failed:", error.message);
    process.exit(1);
  }
}

function aggregateCandidate(records) {
  // Anonymize & aggregate (strip user_id, session_id)
  const profiles = records.map((r) => ({
    palette: r.palette,
    rating: r.rating,
    context: r.design_context
  }));

  const ratings = profiles.map((p) => p.rating || 0).filter((r) => r > 0);
  const avgRating = (ratings.reduce((a, b) => a + b, 0) / ratings.length).toFixed(2);

  return {
    version: `candidate-${Date.now()}`,
    generatedAt: new Date().toISOString(),
    totalInputs: records.length,
    consentedSamples: records.filter((r) => r.consent_training).length,
    profiles,
    stats: {
      avgRating,
      totalRatings: ratings.length,
      paletteCount: profiles.length
    }
  };
}

async function loadBenchmark(version) {
  const url = `https://raw.githubusercontent.com/johnpauljay/Color-Grade/main/benchmarks/${version}.json`;
  try {
    const response = await axios.get(url);
    return response.data;
  } catch (error) {
    throw new Error(`Benchmark not found: ${version}`);
  }
}

function compareAgainstBenchmark(candidate, benchmark) {
  // Simple regression: compare avg rating drift
  const candidateRating = parseFloat(candidate.stats.avgRating);
  const benchmarkRating = parseFloat(benchmark.stats.avgRating);
  
  if (benchmarkRating === 0) return 1;
  
  const drift = Math.abs(candidateRating - benchmarkRating) / benchmarkRating;
  return Math.max(0, Math.min(1, 1 - drift * 0.3));
}

async function publishApprovedProfile(candidate, version) {
  const headers = {
    Authorization: `Bearer ${process.env.GITHUB_TOKEN}`,
    Accept: "application/vnd.github+json",
    "X-GitHub-Api-Version": "2022-11-28"
  };

  const content = Buffer.from(JSON.stringify(candidate, null, 2)).toString(
    "base64"
  );

  // Update latest.json
  const latestUrl = `https://api.github.com/repos/${process.env.GITHUB_OWNER}/${process.env.GITHUB_REPO}/contents/profiles/approved/latest.json`;

  let sha = null;
  try {
    const existing = await axios.get(latestUrl, { headers });
    sha = existing.data.sha;
  } catch (err) {
    if (err.response?.status !== 404) throw err;
  }

  await axios.put(
    latestUrl,
    {
      message: `chore: publish approved profile ${version}`,
      content,
      branch: process.env.GITHUB_BRANCH,
      ...(sha ? { sha } : {})
    },
    { headers }
  );

  // Archive versioned copy
  const archiveUrl = latestUrl.replace(
    "latest.json",
    `archive/${version}.json`
  );
  await axios.put(
    archiveUrl,
    {
      message: `chore: archive approved profile ${version}`,
      content,
      branch: process.env.GITHUB_BRANCH
    },
    { headers }
  );

  console.log(`📁 Archived to profiles/approved/archive/${version}.json`);
}

calibrate();
```

---

## Public API (Credential-Free)

### Fetch Approved Profile

**Endpoint:**

```
https://raw.githubusercontent.com/johnpauljay/Color-Grade/main/profiles/approved/latest.json
```

**Response Format:**

```json
{
  "version": "approved-2026-10-11",
  "generatedAt": "2026-10-11T14:00:00Z",
  "totalInputs": 250,
  "consentedSamples": 250,
  "profiles": [
    {
      "palette": ["#111827", "#f97316", "#facc15"],
      "rating": 5,
      "context": { "type": "landing_page" }
    },
    {
      "palette": ["#ffffff", "#000000", "#808080"],
      "rating": 4,
      "context": { "type": "dashboard" }
    }
  ],
  "stats": {
    "avgRating": "4.8",
    "totalRatings": 240,
    "paletteCount": 250
  }
}
```

### Figma Plugin Integration (Credential-Free)

```javascript
async function initializeWithApprovedProfile() {
  const response = await fetch(
    "https://raw.githubusercontent.com/johnpauljay/Color-Grade/main/profiles/approved/latest.json"
  );
  const approved = await response.json();

  console.log(`Using approved profile v${approved.version}`);
  return approved.profiles; // Array of { palette, rating, context }
}
```

---

## Rollback Procedure

### Check Available Versions

```bash
git log --oneline -- profiles/approved/archive/
```

### Revert to Previous Version

```bash
# Option 1: Promote archive version to latest
git show origin/main:profiles/approved/archive/approved-2026-10-10.json \
  > profiles/approved/latest.json
git add profiles/approved/latest.json
git commit -m "chore: rollback to approved-2026-10-10"
git push
```

Or via GitHub UI:
1. Navigate to `profiles/approved/archive/`
2. Copy contents of previous version
3. Edit `latest.json` with previous version content
4. Commit with message `chore: rollback to {version}`

---

## Directory Structure

```
Color-Grade/
├── README.md
├── .env.example
├── package.json
│
├── supabase/
│   └── schema.sql                      (private table + RLS)
│
├── scripts/
│   └── calibration.js                  (approval gate)
│
├── profiles/
│   └── approved/
│       ├── latest.json                 (current, public, credential-free)
│       └── archive/
│           ├── approved-2026-10-11.json
│           ├── approved-2026-10-10.json
│           └── approved-2026-10-09.json
│
└── benchmarks/
    └── frozen-2026-10-11.json          (reference standard)
```

---

## Workflow: Daily/Weekly Calibration

1. **Collect** (automated):
   - Supabase receives plugin feedback continuously
   - Only `consent_training = true` records are used

2. **Trigger** (scheduled):
   - GitHub Actions runs daily at 2 AM UTC (or manual trigger)
   - Fetches environment variables from GitHub Secrets

3. **Aggregate**:
   - Calibration script reads Supabase with service role
   - Anonymizes records (strips user_id, session_id)
   - Generates candidate profile JSON

4. **Test**:
   - Compares candidate against frozen benchmark
   - Calculates regression score (0-1)
   - Must pass threshold (default: 0.95)

5. **Publish** (if passes):
   - Writes to `profiles/approved/latest.json`
   - Archives versioned copy to `profiles/approved/archive/`
   - Public CDN serves immediately

6. **Report**:
   - Log regression score, sample count, avg rating
   - (Future) Send Slack/email notification
   - (Future) Create GitHub issue if failed

---

## Security & Privacy Summary

| Layer            | Storage       | Access              | Credentials         | Public? |
| ---------------- | ------------- | ------------------- | ------------------- | ------- |
| Raw Feedback     | Supabase      | Service role only   | GitHub Secrets      | ❌      |
| User ID / Photos | Supabase      | Never exported       | GitHub Secrets      | ❌      |
| Calibration Run  | Backend       | Scheduled job       | GitHub Secrets      | ❌      |
| Approved Profile | GitHub public | CDN (anyone)        | None (public JSON)  | ✅      |
| Figma Plugin     | Browser       | User device         | Supabase anon key   | ✅      |
| Benchmark        | GitHub public | Reference (checked) | None (public JSON)  | ✅      |

---

## Implementation Checklist

### Phase 1 (Now)
- [x] Define privacy-first architecture
- [x] Supabase schema with consent flag
- [x] Plugin code for credential-free fetch
- [x] Calibration script template
- [x] Public approved-profile endpoint format

### Phase 2 (Next)
- [ ] Create frozen benchmark: `benchmarks/frozen-2026-10-11.json`
- [ ] Set up GitHub Secrets: `SUPABASE_SERVICE_ROLE_KEY`, `GITHUB_TOKEN`
- [ ] Test calibration script locally: `node scripts/calibration.js`
- [ ] Add `package.json` dependencies
- [ ] Create GitHub Actions workflow: `.github/workflows/calibrate.yml`

### Phase 3 (Future)
- [ ] Implement detailed regression test suite
- [ ] Add monitoring/alerting (Slack integration)
- [ ] Automated rollback on regression detection
- [ ] Metrics dashboard
- [ ] Dataset versioning and retention policy

---

## Next: Send to Agents

**Figma Agent Prompt:**
```
The Figma plugin UI is built. Add the Supabase integration:
- Fetch SUPABASE_URL and SUPABASE_ANON_KEY from environment
- On "Save & Feedback" button, insert to plugin_feedback table
- Include consent checkbox: consent_training (boolean)
- Fetch approved profile from GitHub: https://raw.githubusercontent.com/johnpauljay/Color-Grade/main/profiles/approved/latest.json
- Keep all code credential-free (no GitHub token, no service keys)
- Test with sample palettes and ratings
```

**Supabase Agent Prompt:**
```
Set up the plugin_feedback table from supabase/schema.sql:
- Enable RLS with policies
- Allow anonymous inserts (anon key)
- Service role can read all rows
- Add consent_training boolean field for opt-in training data
- Create indexes on consent_training and created_at
```

**Backend/GitHub Actions Agent Prompt:**
```
Create two things:
1. scripts/calibration.js - reads consent-flagged rows from Supabase, compares against frozen benchmark, publishes to profiles/approved/latest.json
2. .github/workflows/calibrate.yml - runs calibration daily at 2 AM UTC, uses GitHub Secrets for SUPABASE_SERVICE_ROLE_KEY and GITHUB_TOKEN
Include regression testing and error handling.
```

# CLAUDE.md — Vpeace CRM / Sales Automation Web App

## Project Overview

Build a full sales automation web app for **Vpeace** (sole operator: Ryotaro Nemoto), a Japanese LLMO (LLM Optimization) web production service. The app covers the entire sales pipeline from automated lead discovery to contract and payment, all operated remotely via browser.

**Business context:**
- Service: AI-readable website + MCP server endpoint production for toC Japanese businesses
- Pricing: ¥148,000 initial fee + ¥29,800/month subscription (optional)
- Target clients: toC businesses (beauty clinics, restaurants, salons, etc.)
- Sole operator: no team, no auth needed — single-user personal tool

**Goal:** Replace the existing `cold_outreach.html` static file with a full Supabase-backed React web app deployed on Vercel.

---

## Tech Stack

| Layer | Choice | Notes |
|---|---|---|
| Frontend | React + Vite | |
| Styling | Tailwind CSS | Dark theme, same aesthetic as existing tool |
| State | Zustand | Global store for leads, settings, UI state |
| Backend | Supabase | PostgreSQL + Realtime |
| Hosting | Vercel | GitHub auto-deploy |
| Email send | Brevo REST API | Existing Brevo account |
| Lead discovery | Google Places API (New) | `places.googleapis.com/v1/places:searchText` |
| AI enrichment | Gemini 2.0 Flash API | Email extraction, IG search, form URL extraction |
| Payments | Stripe Payment Links | Pre-made static links, no Stripe API key needed |
| E-signature | CloudSign | Manual flow — store link in DB, open with button |

---

## Repository Setup

```
vpeace-crm/
├── CLAUDE.md                    ← this file
├── .env.local                   ← Supabase + API keys (never commit)
├── .env.example                 ← template with key names only
├── .gitignore
├── package.json
├── vite.config.js
├── tailwind.config.js
├── index.html
├── supabase/
│   └── migrations/
│       └── 20240101_initial.sql
└── src/
    ├── main.jsx
    ├── App.jsx
    ├── lib/
    │   ├── supabase.js
    │   ├── api/
    │   │   ├── brevo.js
    │   │   ├── gemini.js
    │   │   └── places.js
    │   └── utils.js
    ├── store/
    │   └── index.js             ← Zustand store
    ├── components/
    │   ├── Layout.jsx
    │   ├── Toast.jsx
    │   ├── tabs/
    │   │   ├── DiscoveryTab.jsx
    │   │   ├── PipelineTab.jsx
    │   │   ├── OutreachTab.jsx
    │   │   ├── DealTab.jsx
    │   │   ├── DeliveryTab.jsx
    │   │   └── SettingsTab.jsx
    │   └── shared/
    │       ├── Button.jsx
    │       ├── Input.jsx
    │       ├── Tag.jsx
    │       └── Modal.jsx
    └── styles/
        └── globals.css
```

---

## Environment Variables

### `.env.example`
```
VITE_SUPABASE_URL=
VITE_SUPABASE_ANON_KEY=
```

All other API keys (Brevo, Gemini, Google Places, Stripe links) are stored in the **Supabase `settings` table** and managed via the app's Settings tab — NOT in `.env`. This allows changing keys without redeployment.

---

## Database Schema

Run this as the initial Supabase migration: `supabase/migrations/20240101_initial.sql`

```sql
-- ============================================================
-- LEADS
-- ============================================================
CREATE TABLE leads (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name          TEXT,
  company       TEXT NOT NULL,
  address       TEXT,
  website       TEXT,
  phone         TEXT,
  email         TEXT,
  contact_form_url TEXT,
  ig_handle     TEXT,

  -- Pipeline stage
  stage         TEXT NOT NULL DEFAULT 'discovered',
  -- Allowed values:
  -- 'discovered'   → 発掘済み（未接触）
  -- 'contacted'    → 連絡済み
  -- 'replied'      → 返信あり
  -- 'proposed'     → 提案書送付済み
  -- 'contracted'   → 契約送付済み
  -- 'paid'         → 入金済み
  -- 'delivering'   → 制作中
  -- 'delivered'    → 納品済み
  -- 'lost'         → 失注

  source        TEXT DEFAULT 'google_maps',
  notes         TEXT,

  -- Email outreach tracking
  email_status  TEXT DEFAULT 'none',  -- none / sent / failed
  sent_at       TIMESTAMPTZ,
  fu1_at        TIMESTAMPTZ,
  fu2_at        TIMESTAMPTZ,
  fu1_due       TIMESTAMPTZ,
  fu2_due       TIMESTAMPTZ,

  -- Deal tracking
  proposal_sent_at    TIMESTAMPTZ,
  contract_url        TEXT,         -- CloudSign link (pasted manually)
  contract_sent_at    TIMESTAMPTZ,
  contract_signed_at  TIMESTAMPTZ,
  payment_initial_url TEXT,         -- Stripe one-time link (from settings, sent per lead)
  payment_sent_at     TIMESTAMPTZ,
  payment_received_at TIMESTAMPTZ,
  subscription_url    TEXT,         -- Stripe subscription link
  subscription_sent_at TIMESTAMPTZ,

  -- Delivery
  delivery_notes      TEXT,
  delivered_at        TIMESTAMPTZ,

  created_at    TIMESTAMPTZ DEFAULT NOW(),
  updated_at    TIMESTAMPTZ DEFAULT NOW()
);

-- Auto-update updated_at
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN NEW.updated_at = NOW(); RETURN NEW; END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER leads_updated_at
  BEFORE UPDATE ON leads
  FOR EACH ROW EXECUTE FUNCTION update_updated_at();

-- ============================================================
-- SETTINGS (key-value store for API keys and templates)
-- ============================================================
CREATE TABLE settings (
  key   TEXT PRIMARY KEY,
  value TEXT
);

-- Seed default keys (empty values)
INSERT INTO settings (key, value) VALUES
  ('brevo_api_key', ''),
  ('sender_email', 'vpeace.gmail@gmail.com'),
  ('sender_name', '根本 凌太朗 / Vpeace'),
  ('places_api_key', ''),
  ('gemini_api_key', ''),
  ('stripe_initial_url', ''),      -- pre-made Stripe payment link for ¥148,000
  ('stripe_subscription_url', ''), -- pre-made Stripe subscription link for ¥29,800/month
  ('proposal_url', ''),            -- URL of the Vpeace proposal page
  ('email_subject', ''),
  ('email_body', ''),
  ('contact_template', 'はじめまして。根本と申します。\n\n{{company}}様のウェブサイトを拝見し、ご連絡させていただきました。\n\nAI検索（ChatGPT・Geminiなど）で正しく認識されるサイトを制作しております。5分ほどお時間をいただけますでしょうか？\n\n根本 凌太朗 / Vpeace'),
  ('dm_template', 'はじめまして！根本と申します🙏\n\n{{company}}様のインスタを拝見し、DMさせていただきました。\n\nAI検索（ChatGPT・Gemini）に最適化したサイト制作をしております✨\nご興味があればぜひお話しさせてください😊'),
  ('fu1_enabled', 'false'),
  ('fu1_days', '3'),
  ('fu1_subject', ''),
  ('fu1_body', ''),
  ('fu2_enabled', 'false'),
  ('fu2_days', '7'),
  ('fu2_subject', ''),
  ('fu2_body', '');

-- ============================================================
-- DELIVERY CHECKLIST ITEMS (per lead)
-- ============================================================
CREATE TABLE delivery_items (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  lead_id    UUID NOT NULL REFERENCES leads(id) ON DELETE CASCADE,
  item       TEXT NOT NULL,
  done       BOOLEAN DEFAULT FALSE,
  done_at    TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Default checklist items to insert when a lead moves to 'delivering'
-- (insert these via app logic, not DB trigger):
-- 1. ビジネス情報ヒアリング完了
-- 2. llms.txt 作成
-- 3. JSON-LD 構造化データ作成
-- 4. セマンティックHTMLマークアップ
-- 5. MCP サーバーエンドポイント構築
-- 6. Supabase デプロイ完了
-- 7. Claude での AI 読み取りテスト
-- 8. ChatGPT での AI 読み取りテスト
-- 9. 動作検証レポート作成
-- 10. 納品・クライアントに共有完了
```

**Disable RLS** on all tables (single-user tool, no auth):
```sql
ALTER TABLE leads DISABLE ROW LEVEL SECURITY;
ALTER TABLE settings DISABLE ROW LEVEL SECURITY;
ALTER TABLE delivery_items DISABLE ROW LEVEL SECURITY;
```

---

## API Integration Code

### `src/lib/supabase.js`
```js
import { createClient } from '@supabase/supabase-js'

export const supabase = createClient(
  import.meta.env.VITE_SUPABASE_URL,
  import.meta.env.VITE_SUPABASE_ANON_KEY
)

// Load all settings as an object
export async function loadSettings() {
  const { data } = await supabase.from('settings').select('key, value')
  return Object.fromEntries((data || []).map(r => [r.key, r.value]))
}

// Save a single setting
export async function saveSetting(key, value) {
  await supabase.from('settings').upsert({ key, value })
}
```

### `src/lib/api/places.js`
```js
export async function searchPlaces(apiKey, query) {
  const r = await fetch('https://places.googleapis.com/v1/places:searchText', {
    method: 'POST',
    headers: {
      'X-Goog-Api-Key': apiKey,
      'X-Goog-FieldMask': 'places.id,places.displayName,places.formattedAddress,places.websiteUri,places.nationalPhoneNumber',
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ textQuery: query, languageCode: 'ja', maxResultCount: 20 })
  })
  if (!r.ok) throw new Error(`Places API error: ${r.status}`)
  const d = await r.json()
  return (d.places || []).map(p => ({
    _id: crypto.randomUUID(),
    company: p.displayName?.text || '',
    address: p.formattedAddress || '',
    website: p.websiteUri || '',
    phone: p.nationalPhoneNumber || '',
    email: '', contactFormUrl: '', igHandle: '',
    emailStatus: 'none', igStatus: 'none', selected: false
  }))
}
```

### `src/lib/api/gemini.js`
```js
const GEMINI_URL = key =>
  `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=${key}`

async function geminiSearch(apiKey, prompt) {
  const r = await fetch(GEMINI_URL(apiKey), {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      contents: [{ parts: [{ text: prompt }] }],
      tools: [{ googleSearch: {} }],
      generationConfig: { responseMimeType: 'text/plain' }
    })
  })
  if (!r.ok) throw new Error(`Gemini error: ${r.status}`)
  const d = await r.json()
  return (d.candidates?.[0]?.content?.parts || []).map(p => p.text || '').join('')
}

function extractJson(text) {
  const m = text.replace(/```json|```/g, '').trim().match(/\{[^}]+\}/)
  if (!m) return null
  try { return JSON.parse(m[0]) } catch { return null }
}

// Find Instagram handle
export async function findIG(apiKey, company, website) {
  const text = await geminiSearch(apiKey,
    `「${company}」という日本のビジネスの公式Instagramアカウントのハンドルを探してください。\nウェブサイト: ${website || '不明'}\nJSON形式のみで返答: {"handle":"アカウント名"} または {"handle":null}\n@マーク不要。説明文不要。`)
  return extractJson(text)?.handle || null
}

// Find email address from website
export async function findEmail(apiKey, company, website) {
  const text = await geminiSearch(apiKey,
    `「${company}」という日本のビジネスのメールアドレスを探してください。\nウェブサイト: ${website || '不明'}\nJSON形式のみで返答: {"email":"メールアドレス"} または {"email":null}\n説明文不要。`)
  const email = extractJson(text)?.email || null
  if (!email || !email.includes('@')) return null
  return email
}

// Find contact form URL
export async function findContactForm(apiKey, company, website) {
  const text = await geminiSearch(apiKey,
    `「${company}」という日本のビジネスのウェブサイトのお問い合わせフォームページのURLを探してください。\nウェブサイト: ${website || '不明'}\nJSON形式のみで返答: {"url":"フォームURL"} または {"url":null}\n説明文不要。`)
  const url = extractJson(text)?.url || null
  return url
}
```

### `src/lib/api/brevo.js`
```js
export async function brevoSend(apiKey, from, to, subject, htmlBody) {
  const r = await fetch('https://api.brevo.com/v3/smtp/email', {
    method: 'POST',
    headers: { 'api-key': apiKey, 'Content-Type': 'application/json' },
    body: JSON.stringify({
      sender: from,
      to: [to],
      subject,
      htmlContent: htmlBody
    })
  })
  if (!r.ok) {
    const e = await r.json().catch(() => ({}))
    throw new Error(e.message || `Brevo error: ${r.status}`)
  }
  return r.json()
}

// Template variable substitution: {{company}}, {{name}}
export function applyTemplate(template, vars) {
  return template
    .replace(/\{\{\s*company\s*\}\}/g, vars.company || '')
    .replace(/\{\{\s*name\s*\}\}/g, vars.name || vars.company || '')
    .replace(/\n/g, '<br>')
}
```

---

## UI Design

### Theme
Dark mode only. Base colors:
```
bg:       #0D1117   (page)
surface:  #111827   (cards)
border:   #1F2937
accent:   #4F46E5   (indigo — primary action)
success:  #059669
warning:  #D97706
danger:   #DC2626
text:     #F9FAFB
muted:    #6B7280
```

Use `DM Mono` for stats/codes/tags, `DM Sans` for body.

### Tab Structure
Six tabs in top navigation:

| Tab | Icon | Label |
|---|---|---|
| 0 | 🗺️ | 発掘 |
| 1 | 📊 | パイプライン |
| 2 | ✉️ | アウトリーチ |
| 3 | 💼 | 商談・契約 |
| 4 | ✅ | 納品 |
| 5 | ⚙️ | 設定 |

---

## Feature Specifications

### Tab 0 — 発掘 (Lead Discovery)

**Search flow:**
1. User enters query (e.g. `美容クリニック 渋谷`) and clicks 搜索
2. Call `searchPlaces(placesApiKey, query)` → show result cards
3. Each card shows: company name, address, website, phone
4. **Auto-enrichment buttons (per card):**
   - `📧 メール検索` → calls `findEmail(geminiKey, company, website)` → fills email field
   - `🌐 フォーム検索` → calls `findContactForm(...)` → fills contact form URL field  
   - `📸 IG検索` → calls `findIG(...)` → fills IG handle
5. **Batch buttons (top bar, applies to selected cards):**
   - `📧 選択分メール一括検索` → runs findEmail for all selected, 800ms delay between
   - `📸 選択分IG一括検索` → runs findIG for all selected
6. **Action buttons per card:**
   - `🌐 HPを開く` → copies contactTemplate to clipboard, opens website in new tab
   - `📸 DMを開く` → copies dmTemplate to clipboard, opens `https://ig.me/m/${igHandle}`
7. **Add to pipeline button (top):**
   - `➕ パイプラインに追加` → inserts selected leads to Supabase `leads` table (skip duplicates by website URL)
   - Shows count of added / skipped

**Card states:**
- Email: `未取得` (gray) / `取得済み ✓ email@...` (green) / `見つからず` (muted)
- Form: `未取得` / `あり` (show URL) / `なし`
- IG: `未取得` / `@handle` (pink) / `なし`

---

### Tab 1 — パイプライン (Kanban CRM)

**Kanban board** with 8 columns. Each column is scrollable vertically.

```
発掘済み → 連絡済み → 返信あり → 提案送付 → 契約送付 → 入金済み → 制作中 → 納品済み
```

Plus a hidden `失注` column (accessible via filter toggle).

**Column config:**
```js
const STAGES = [
  { id: 'discovered',  label: '発掘済み',  color: '#374151' },
  { id: 'contacted',   label: '連絡済み',  color: '#4F46E5' },
  { id: 'replied',     label: '返信あり',  color: '#0EA5E9' },
  { id: 'proposed',    label: '提案送付',  color: '#8B5CF6' },
  { id: 'contracted',  label: '契約送付',  color: '#D97706' },
  { id: 'paid',        label: '入金済み',  color: '#059669' },
  { id: 'delivering',  label: '制作中',    color: '#10B981' },
  { id: 'delivered',   label: '納品済み',  color: '#6EE7B7' },
]
```

**Lead card (in kanban):**
- Company name (bold)
- Website (truncated)
- Email (if set)
- Days in current stage (e.g. `3日`)
- Stage change buttons: `←` `→` (move left/right)
- Click card → opens **Lead Detail Modal**

**Lead Detail Modal:**
Full-screen modal with sections:
1. **基本情報** — edit company, website, email, phone, address, IG, notes
2. **アウトリーチ履歴** — sent_at, fu1_at, fu2_at timestamps
3. **商談** — buttons for each deal action (see Tab 3)
4. **メモ** — free text notes field (auto-saved)
5. **ステージ変更** — dropdown to jump to any stage
6. **削除** — delete lead (with confirm)

**Stats bar (above kanban):**
Row of stat cards: Total leads / 今月成約 / 月次収益（今月入金件数×¥148K） / 継続中（月額件数×¥29.8K）

---

### Tab 2 — アウトリーチ (Email Campaign)

Same functionality as existing `cold_outreach.html` campaign tab, but data from Supabase.

**Sections:**

**1. テンプレート設定**
- HP問い合わせフォーム用テンプレ (contactTemplate) — textarea
- Instagram DM用テンプレ (dmTemplate) — textarea
- メール件名 (email_subject) — input
- メール本文 (email_body) — textarea
- フォローアップ1: toggle + days + subject + body
- フォローアップ2: toggle + days + subject + body
- `保存` button → upserts all to settings table

**2. 送信対象リスト**
- Shows leads where stage = 'discovered' and email_status = 'none' and email is set
- Count badge
- `🚀 X件に一括送信` button

**3. 送信実行**
- Loop through targets
- Call `brevoSend(...)` with template applied
- Update lead: email_status='sent', sent_at=now, stage='contacted'
- Set fu1_due and fu2_due if enabled
- Progress bar during send (400ms delay between sends)

**4. フォローアップ自動チェック**
- On app load: query leads where (fu1_due <= now AND fu1_at IS NULL) OR (fu2_due <= now AND fu2_at IS NULL)
- Banner: `⏰ フォローアップ X件が送信待ち` with `今すぐ送信` button
- Sends FU emails, updates fu1_at/fu2_at

---

### Tab 3 — 商談・契約 (Deal Management)

Shows leads in stages: replied / proposed / contracted / paid

**Per-lead deal action panel:**

```
[提案書を送る]  →  Copies mailto link: gmail compose with proposal URL in body
                   Subject: "{{company}}様へのご提案"
                   Body: includes settings.proposal_url
                   After click: update lead stage → 'proposed', proposal_sent_at = now

[契約書を送る]  →  Opens modal to paste CloudSign link
                   Saves to lead.contract_url
                   Copies mailto with contract link
                   After send: update stage → 'contracted', contract_sent_at = now

[入金確認]      →  Opens payment link (settings.stripe_initial_url) in new tab
                   Button: `入金を確認した` → update stage → 'paid', payment_received_at = now

[月額スタート]  →  Opens settings.stripe_subscription_url in new tab
                   Button: `月額送付済み` → update subscription_sent_at = now
```

**Mailto helper:**
```js
function openGmailDraft(to, subject, body) {
  const url = `https://mail.google.com/mail/?view=cm&to=${encodeURIComponent(to)}&su=${encodeURIComponent(subject)}&body=${encodeURIComponent(body)}`
  window.open(url, '_blank')
}
```

**Pipeline view in this tab:**
Vertical list of leads (replied → contracted), sorted by most recent activity. Each row shows:
- Company name + stage badge
- Days since last action
- Action buttons appropriate to current stage

---

### Tab 4 — 納品 (Delivery)

Shows leads where stage = 'paid' or 'delivering' or 'delivered'.

**Per-lead delivery card:**
- Company name
- Progress bar: completed checklist items / total
- Checklist items (from `delivery_items` table):
  - Toggle done/not done
  - When all done → enable `納品完了` button
- `制作開始` button (when stage='paid') → sets stage='delivering', inserts default 10 checklist items
- `納品完了` button → sets stage='delivered', delivered_at=now, sends congrats toast

**Default checklist items (inserted on 制作開始):**
```js
const DEFAULT_CHECKLIST = [
  'ビジネス情報ヒアリング完了',
  'llms.txt 作成',
  'JSON-LD 構造化データ作成',
  'セマンティック HTML マークアップ',
  'MCP サーバーエンドポイント構築',
  'Supabase デプロイ完了',
  'Claude での AI 読み取りテスト',
  'ChatGPT での AI 読み取りテスト',
  '動作検証レポート作成',
  '納品・クライアントに URL 共有完了',
]
```

---

### Tab 5 — 設定 (Settings)

Form with all settings keys. Load from Supabase on mount, save on blur or save button.

**Sections:**
1. **Brevo** — brevo_api_key (password), sender_email, sender_name
2. **Google Places** — places_api_key (password)
3. **Gemini** — gemini_api_key (password)
4. **Stripe リンク** — stripe_initial_url, stripe_subscription_url (plain text URLs)
5. **提案書 URL** — proposal_url
6. **設定状態チェック** — green/red indicators for each required setting

---

## Key Implementation Notes

### No Auth
This is a single-user personal tool. No Supabase Auth, no login screen. Just use the anon key with RLS disabled.

### Data Loading Pattern
On app mount:
1. Load all settings from Supabase → store in Zustand
2. Load all leads from Supabase → store in Zustand
3. Check for pending follow-ups → show banner if any
4. Subscribe to leads table via Supabase Realtime for live updates

### Duplicate Prevention
When adding leads from discovery to pipeline, check by `website` URL. If already exists, skip and report count.

### Template Variables
Supported in all templates: `{{company}}`, `{{name}}` (same as company, or stripped company name)

### Error Handling
All API calls wrapped in try/catch. Errors shown as red toast notifications. Never crash the UI.

### LocalStorage Fallback
API keys typed in Settings tab are saved to Supabase. Do NOT fall back to localStorage for keys — always use Supabase as source of truth.

### Supabase Realtime
Subscribe to leads table changes so that if data is modified (e.g., Supabase dashboard), the UI updates live:
```js
supabase.channel('leads').on('postgres_changes', { event: '*', schema: 'public', table: 'leads' }, payload => {
  // update Zustand store
}).subscribe()
```

---

## Build Order (Recommended for Claude Code)

Build in this order to have a working app as early as possible:

1. **Project scaffold** — Vite + React + Tailwind + Supabase client setup, `.env.example`
2. **Supabase migration** — run `supabase/migrations/20240101_initial.sql`
3. **Settings tab** — settings load/save from Supabase (validates all API keys are wired correctly)
4. **発掘タブ** — Places search + Gemini email/IG/form extraction + add to pipeline
5. **パイプライン** — Kanban board + lead detail modal + stage transitions
6. **アウトリーチタブ** — template editor + Brevo batch send + FU auto-check on load
7. **商談タブ** — deal actions (propose, contract, payment confirmation)
8. **納品タブ** — delivery checklist
9. **Polish** — stats bar, toast notifications, loading states, mobile responsiveness

---

## Deployment

1. Push to GitHub
2. Connect repo to Vercel (auto-deploy on push to main)
3. Add env vars in Vercel dashboard: `VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY`
4. Set custom domain if desired

---

## Existing Reference Code

The existing `cold_outreach.html` (in this repo for reference) contains working implementations of:
- `searchPlaces()` — Google Places API call
- `findIG()` — Gemini Instagram search
- `brevoSend()` — Brevo email send
- `autoFU()` — Follow-up automation logic
- `parseCsv()` — CSV import
- UI components (Tag, Inp, TA, Btn, SC) — adapt these to React/Tailwind

Port these patterns into the new React component structure.

---

## Summary of What Gets Built

| Feature | Status |
|---|---|
| Google Maps lead discovery | Port from existing |
| **Email auto-extraction (Gemini)** | **NEW** |
| **Contact form URL extraction (Gemini)** | **NEW** |
| Instagram handle search (Gemini) | Port + improve |
| Brevo email batch send + FU automation | Port from existing |
| **Supabase database (persistent CRM)** | **NEW** |
| **Kanban pipeline (8 stages)** | **NEW** |
| **Lead detail modal** | **NEW** |
| **Proposal / contract / payment flow** | **NEW** |
| **Delivery checklist** | **NEW** |
| Settings management via Supabase | **NEW** |
| Vercel deployment | **NEW** |

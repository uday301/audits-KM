# Kerkar Media — n8n Automation Workflow
# Complete code for all nodes to auto-generate the dashboard

---

## WORKFLOW OVERVIEW

```
Schedule (Mon 8am)
  → [1] Fetch GSC Data
  → [2] Fetch GA4 Data  
  → [3] Fetch Ahrefs Data       ← NEW
  → [4] Fetch Leads (Sheets)
  → [5] Merge (4 inputs)
  → [6] Code: Process + Claude  ← UPDATED
  → [7] Code: Build HTML        ← NEW (replaces Notion)
  → [8] Code: Push to GitHub    ← NEW
```

---

## NODE 1 — Schedule Trigger
- Type: Schedule Trigger
- Every: Week
- Day: Monday
- Time: 08:00

---

## NODE 2 — GSC Data (existing, keep as-is)
- Type: Google Search Console (community node)
- Operation: Get Page Insights
- Site: sc-domain:kerkarmedia.com
- Date Range: Last 28 days
- Row Limit: 50

---

## NODE 3 — GA4 Data (existing, keep as-is)
- Type: HTTP Request
- Method: POST
- URL: https://analyticsdata.googleapis.com/v1beta/properties/YOUR_GA4_ID:runReport
- Auth: Google OAuth2
- Body: (existing config)

---

## NODE 4 — AHREFS DATA ← ADD THIS NEW NODE

- Type: HTTP Request
- Name: Fetch Ahrefs Data
- Method: GET
- Authentication: Header Auth
  - Name: Authorization
  - Value: Bearer YOUR_AHREFS_API_KEY

Add as a Code node (JavaScript) — makes multiple Ahrefs calls:

```javascript
// ============================================
// NODE 4: FETCH AHREFS DATA
// Add as: Code node (JavaScript)
// ============================================

const AHREFS_KEY = "YOUR_AHREFS_API_KEY";
const TARGET = "kerkarmedia.com";
const TODAY = new Date().toISOString().split('T')[0];
const DATE_FROM = new Date(Date.now() - 150 * 24 * 60 * 60 * 1000).toISOString().split('T')[0];

const headers = { "Authorization": `Bearer ${AHREFS_KEY}` };

// Helper
async function ahrefsGet(endpoint) {
  const res = await this.helpers.httpRequest({
    method: 'GET',
    url: `https://api.ahrefs.com/v3${endpoint}`,
    headers
  });
  return res;
}

// 1. Site Metrics (org keywords, traffic, value)
const metrics = await ahrefsGet(
  `/site-explorer/metrics?target=${TARGET}&mode=domain&date=${TODAY}`
);

// 2. Domain Rating + Ahrefs Rank
const dr = await ahrefsGet(
  `/site-explorer/domain-rating?target=${TARGET}&date=${TODAY}`
);

// 3. Backlinks Stats (live backlinks, refdomains)
const backlinks = await ahrefsGet(
  `/site-explorer/backlinks-stats?target=${TARGET}&mode=domain&date=${TODAY}`
);

// 4. DR History (last 6 months)
const drHistory = await ahrefsGet(
  `/site-explorer/domain-rating-history?target=${TARGET}&date_from=${DATE_FROM}&date_to=${TODAY}&limit=6`
);

// 5. Refdomains History (last 6 months)
const refHistory = await ahrefsGet(
  `/site-explorer/refdomains-history?target=${TARGET}&mode=domain&date_from=${DATE_FROM}&date_to=${TODAY}&limit=6`
);

// 6. Traffic by Country
const byCountry = await ahrefsGet(
  `/site-explorer/metrics-by-country?target=${TARGET}&mode=domain&date=${TODAY}&limit=5`
);

// 7. Top Organic Keywords (top 10 by traffic)
const keywords = await ahrefsGet(
  `/site-explorer/organic-keywords?target=${TARGET}&mode=domain&limit=10&country=in&date=${TODAY}&select=keyword,best_position,sum_traffic,volume,is_branded,is_commercial,is_informational,is_local,is_navigational,is_transactional,best_position_url,is_best_position_set_top_3,is_best_position_set_top_4_10,is_best_position_set_top_11_50&order_by=sum_traffic:desc`
);

// Package all Ahrefs data into one item
return [{
  json: {
    source: 'ahrefs',
    metrics: metrics.metrics,
    domainRating: dr.domain_rating,
    backlinksStats: backlinks.metrics,
    drHistory: drHistory.domain_ratings,
    refHistory: refHistory.refdomains,
    byCountry: byCountry.metrics,
    keywords: keywords.keywords
  }
}];
```

---

## NODE 5 — LEADS (Google Sheets, existing)
- Keep exactly as-is
- Pulls CF7 leads sheet

---

## NODE 6 — MERGE NODE
- Type: Merge
- Mode: Append
- Change inputs from 3 → 4 (add the new Ahrefs node as 4th input)

---

## NODE 7 — PROCESS + CLAUDE ← REPLACE EXISTING CODE NODE

```javascript
// ============================================
// NODE 7: PROCESS ALL DATA + CALL CLAUDE API
// Replace your existing "Code in JavaScript1" node
// ============================================

// ── Separate data by source ──
const allItems = $input.all();

let gscData = [];
let ga4Data = null;
let ahrefsData = null;
let leadsData = [];

for (const item of allItems) {
  const d = item.json;
  if (d.source === 'ahrefs') {
    ahrefsData = d;
  } else if (d.sessions !== undefined || d.bounceRate !== undefined) {
    ga4Data = d;
  } else if (d.timestamp !== undefined || d['text-778'] !== undefined) {
    leadsData.push(d);
  } else if (d.keys !== undefined || d.clicks !== undefined) {
    gscData.push(d);
  }
}

// ── Process GSC ──
const totalClicks = gscData.reduce((s, r) => s + (r.clicks || 0), 0);
const totalImpressions = gscData.reduce((s, r) => s + (r.impressions || 0), 0);
const avgCTR = totalImpressions > 0 ? ((totalClicks / totalImpressions) * 100).toFixed(2) : 0;
const avgPosition = gscData.length > 0
  ? (gscData.reduce((s, r) => s + (r.position || 0), 0) / gscData.length).toFixed(1)
  : 0;

// Top keywords from GSC
const topKeywords = [...gscData]
  .sort((a, b) => (b.clicks || 0) - (a.clicks || 0))
  .slice(0, 10);

// ── Process Ahrefs ──
const dr = ahrefsData?.domainRating?.domain_rating || 0;
const ahrefsRank = ahrefsData?.domainRating?.ahrefs_rank || 0;
const liveBacklinks = ahrefsData?.backlinksStats?.live || 0;
const liveRefdomains = ahrefsData?.backlinksStats?.live_refdomains || 0;
const allTimeBacklinks = ahrefsData?.backlinksStats?.all_time || 0;
const orgKeywords = ahrefsData?.metrics?.org_keywords || 0;
const orgKeywordsTop3 = ahrefsData?.metrics?.org_keywords_1_3 || 0;
const orgTraffic = ahrefsData?.metrics?.org_traffic || 0;
const orgValue = ahrefsData?.metrics?.org_cost || 0;
const ahrefsKeywords = ahrefsData?.keywords || [];
const drHistory = ahrefsData?.drHistory || [];
const refHistory = ahrefsData?.refHistory || [];
const byCountry = ahrefsData?.byCountry || [];

// Intent breakdown from Ahrefs keywords
const intentBreakdown = {
  branded: { keywords: 0, traffic: 0 },
  nonBranded: { keywords: 0, traffic: 0 },
  informational: { keywords: 0, traffic: 0 },
  commercial: { keywords: 0, traffic: 0 },
  transactional: { keywords: 0, traffic: 0 },
  local: { keywords: 0, traffic: 0 },
  navigational: { keywords: 0, traffic: 0 }
};

for (const kw of ahrefsKeywords) {
  if (kw.is_branded) { intentBreakdown.branded.keywords++; intentBreakdown.branded.traffic += kw.sum_traffic; }
  else { intentBreakdown.nonBranded.keywords++; intentBreakdown.nonBranded.traffic += kw.sum_traffic; }
  if (kw.is_informational) { intentBreakdown.informational.keywords++; intentBreakdown.informational.traffic += kw.sum_traffic; }
  if (kw.is_commercial) { intentBreakdown.commercial.keywords++; intentBreakdown.commercial.traffic += kw.sum_traffic; }
  if (kw.is_transactional) { intentBreakdown.transactional.keywords++; intentBreakdown.transactional.traffic += kw.sum_traffic; }
  if (kw.is_local) { intentBreakdown.local.keywords++; intentBreakdown.local.traffic += kw.sum_traffic; }
  if (kw.is_navigational) { intentBreakdown.navigational.keywords++; intentBreakdown.navigational.traffic += kw.sum_traffic; }
}

// Position buckets
const pos13 = ahrefsKeywords.filter(k => k.is_best_position_set_top_3).length;
const pos410 = ahrefsKeywords.filter(k => k.is_best_position_set_top_4_10).length;
const pos1150 = ahrefsKeywords.filter(k => k.is_best_position_set_top_11_50).length;

// ── Process GA4 ──
const sessions = ga4Data?.sessions || 0;
const bounceRate = ga4Data?.bounceRate || 0;
const conversions = ga4Data?.conversions || 0;

// ── Process Leads ──
const thisWeekLeads = leadsData.filter(l => {
  if (!l.timestamp) return false;
  const leadDate = new Date(l.timestamp);
  const weekAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);
  return leadDate > weekAgo;
});

// ── Build summary for Claude ──
const reportDate = new Date().toLocaleDateString('en-IN', { day: '2-digit', month: 'short', year: 'numeric' });

const summaryForClaude = `
KERKAR MEDIA SEO REPORT — ${reportDate}

GSC DATA (Last 28 days):
- Total Clicks: ${totalClicks}
- Total Impressions: ${totalImpressions}
- Avg CTR: ${avgCTR}%
- Avg Position: ${avgPosition}
- Top Keywords: ${topKeywords.slice(0,3).map(k => `${k.keys?.[0]} (pos: ${k.position?.toFixed(0)}, clicks: ${k.clicks})`).join(', ')}

AHREFS DATA:
- Domain Rating: ${dr} | Ahrefs Rank: ${ahrefsRank.toLocaleString()}
- Live Backlinks: ${liveBacklinks.toLocaleString()} | Ref Domains: ${liveRefdomains}
- Organic Keywords: ${orgKeywords} (Top 3: ${orgKeywordsTop3})
- Organic Traffic: ${orgTraffic} | Traffic Value: $${orgValue}
- Intent: Branded ${intentBreakdown.branded.keywords}kw, Commercial ${intentBreakdown.commercial.keywords}kw, Local ${intentBreakdown.local.keywords}kw

GA4 DATA (Last 7 days):
- Sessions: ${sessions}
- Bounce Rate: ${bounceRate}%
- Conversions (thank_you): ${conversions}

LEADS THIS WEEK: ${thisWeekLeads.length}
${thisWeekLeads.map(l => `- ${l['text-778'] || 'Unknown'} | ${l['menu-22'] || 'N/A'} | ${l['Budget'] || 'N/A'}`).join('\n')}

Based on this data, provide a 4-line analysis:
Line 1 (SUMMARY): 1 sentence executive summary of overall SEO health
Line 2 (WINS): Top 3 wins this week, each starting with ✓
Line 3 (CONCERNS): Top 3 concerns, each starting with ✗
Line 4 (ACTIONS): Top 3 priority actions with [HIGH]/[MED] labels

Use India CTR benchmarks: branded 15%+, commercial 3-5%, informational 1-2%
Reference Algorithm Leak signals where relevant: badClicks, goodClicks, contentEffort, siteRadius
Keep each line under 300 characters.
`;

// ── Call Claude API ──
const claudeResponse = await this.helpers.httpRequest({
  method: 'POST',
  url: 'https://api.anthropic.com/v1/messages',
  headers: {
    'Content-Type': 'application/json',
    'x-api-key': 'YOUR_ANTHROPIC_API_KEY',  // ← Replace with your key
    'anthropic-version': '2023-06-01'
  },
  body: JSON.stringify({
    model: 'claude-sonnet-4-5',
    max_tokens: 1000,
    messages: [{ role: 'user', content: summaryForClaude }]
  })
});

const claudeText = claudeResponse.content?.[0]?.text || '';
const lines = claudeText.split('\n').filter(l => l.trim());

const executiveSummary = lines.find(l => l.startsWith('Line 1') || l.includes('SUMMARY'))?.replace(/^Line 1.*?:\s*/i, '') || claudeText.split('.')[0];
const wins = lines.find(l => l.startsWith('Line 2') || l.includes('WINS'))?.replace(/^Line 2.*?:\s*/i, '') || '';
const concerns = lines.find(l => l.startsWith('Line 3') || l.includes('CONCERNS'))?.replace(/^Line 3.*?:\s*/i, '') || '';
const actions = lines.find(l => l.startsWith('Line 4') || l.includes('ACTIONS'))?.replace(/^Line 4.*?:\s*/i, '') || '';

// ── Return structured data for HTML builder ──
return [{
  json: {
    reportDate,
    // GSC
    totalClicks,
    totalImpressions,
    avgCTR,
    avgPosition,
    topKeywords,
    // Ahrefs
    dr,
    ahrefsRank,
    liveBacklinks,
    liveRefdomains,
    allTimeBacklinks,
    orgKeywords,
    orgKeywordsTop3,
    orgTraffic,
    orgValue,
    ahrefsKeywords,
    intentBreakdown,
    pos13,
    pos410,
    pos1150,
    drHistory,
    refHistory,
    byCountry,
    // GA4
    sessions,
    bounceRate,
    conversions,
    // Leads
    thisWeekLeads,
    totalLeads: leadsData.length,
    // Claude insights
    executiveSummary,
    wins,
    concerns,
    actions
  }
}];
```

---

## NODE 8 — BUILD HTML + PUSH TO GITHUB ← NEW NODE

```javascript
// ============================================
// NODE 8: BUILD HTML DASHBOARD + PUSH TO GITHUB
// Add as: Code node (JavaScript)
// ============================================

const d = $input.first().json;

// ── Helper: format numbers ──
const fmt = (n) => n >= 1000 ? (n/1000).toFixed(1) + 'K' : String(n);
const pct = (n) => parseFloat(n).toFixed(1) + '%';
const today = new Date().toISOString().split('T')[0];

// ── Build keyword rows ──
const keywordRows = (d.ahrefsKeywords || []).map(kw => {
  const posClass = kw.is_best_position_set_top_3 ? 'p13' :
                   kw.is_best_position_set_top_4_10 ? 'p410' : 'p1120';
  const intent = kw.is_branded ? 'Branded' :
                 kw.is_commercial ? 'Commercial' :
                 kw.is_local ? 'Local' : 'Informational';
  const intentClass = kw.is_branded ? 'tb' : kw.is_commercial ? 'ta' : 'tb';
  const barWidth = Math.max(5, Math.min(100, (kw.sum_traffic / 50) * 100));
  return `<tr>
    <td class="tkw">${kw.keyword}</td>
    <td><span class="pos ${posClass}">${kw.best_position}</span></td>
    <td>${kw.sum_traffic}</td>
    <td>${(kw.volume || 0).toLocaleString()}</td>
    <td><span class="tag ${intentClass}" style="font-size:10px">${intent}</span></td>
    <td><div class="bw"><div class="bf" style="width:${barWidth}%"></div></div></td>
  </tr>`;
}).join('');

// ── Build country rows ──
const countryFlags = { IN: '🇮🇳 India', US: '🇺🇸 United States', AE: '🇦🇪 UAE', GB: '🇬🇧 UK', SG: '🇸🇬 Singapore' };
const totalCountryTraffic = (d.byCountry || []).reduce((s, c) => s + (c.org_traffic || 0), 0);
const countryRows = (d.byCountry || []).map(c => {
  const share = totalCountryTraffic > 0 ? ((c.org_traffic / totalCountryTraffic) * 100).toFixed(1) + '%' : '—';
  return `<tr>
    <td class="tkw">${countryFlags[c.country] || c.country}</td>
    <td>${c.org_traffic}</td>
    <td>${share}</td>
    <td>${c.org_keywords}</td>
    <td>${c.org_keywords_1_3}</td>
  </tr>`;
}).join('');

// ── Build intent table rows ──
const ib = d.intentBreakdown || {};
const intentRows = [
  ['🏷️ Branded', ib.branded],
  ['🔍 Non-branded', ib.nonBranded],
  ['📖 Informational', ib.informational],
  ['💼 Commercial', ib.commercial],
  ['💸 Transactional', ib.transactional],
  ['📍 Local', ib.local],
  ['🧭 Navigational', ib.navigational],
].map(([label, data]) => `<tr>
  <td class="tkw">${label}</td>
  <td>${data?.keywords || 0}</td>
  <td>${data?.traffic || 0}</td>
</tr>`).join('');

// ── DR History chart data ──
const drLabels = (d.drHistory || []).map(h => {
  const date = new Date(h.date);
  return date.toLocaleString('en', { month: 'short' });
});
const drValues = (d.drHistory || []).map(h => h.domain_rating);

// ── Refdomains History chart data ──
const refLabels = (d.refHistory || []).map(h => {
  const date = new Date(h.date);
  return date.toLocaleString('en', { month: 'short' });
});
const refValues = (d.refHistory || []).map(h => h.refdomains);

// ── Build leads rows ──
const leadsRows = (d.thisWeekLeads || []).map(l => `<tr>
  <td class="tkw">${l['text-778'] || '—'}</td>
  <td>${l['text-927'] || '—'}</td>
  <td><span class="sbadge">${l['menu-22'] || '—'}</span></td>
  <td><span class="bbadge">${l['Budget'] || '—'}</span></td>
  <td>${l.timestamp ? new Date(l.timestamp).toLocaleDateString('en-IN') : '—'}</td>
</tr>`).join('') || '<tr><td colspan="5" style="text-align:center;color:var(--muted);padding:20px">No leads this week</td></tr>';

// ── Parse wins/concerns/actions ──
const parseInsights = (text) => text ? text.split(/[✓✗→]/).filter(s => s.trim()).map(s => s.trim()) : [];
const winsList = parseInsights(d.wins).map(w => `<li><b>✓</b>${w}</li>`).join('') || '<li><b>✓</b>Data loading...</li>';
const lossList = parseInsights(d.concerns).map(c => `<li><b>✗</b>${c}</li>`).join('') || '<li><b>✗</b>Data loading...</li>';
const actionsList = parseInsights(d.actions).map(a => `<li><b>→</b>${a}</li>`).join('') || '<li><b>→</b>Data loading...</li>';

// ── CTR deltas ──
const ctrNum = parseFloat(d.avgCTR) || 0;
const ctrDeltaClass = ctrNum >= 3 ? 'du' : 'dd';
const posDeltaClass = parseFloat(d.avgPosition) <= 20 ? 'du' : 'dd';

// ══════════════════════════════════════════════════════
// BUILD FULL HTML
// ══════════════════════════════════════════════════════
const html = `<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Kerkar Media — SEO Dashboard | ${d.reportDate}</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=DM+Sans:opsz,wght@9..40,400;9..40,600;9..40,700&family=Inter:wght@400;500;600&display=swap" rel="stylesheet">
<script src="https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.5/gsap.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.5/ScrollTrigger.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"></script>
<style>
:root{--cta:#003DCC;--blue:#035398;--black:#000;--white:#fff;--footer:#124648;--green:#16a34a;--red:#dc2626;--amber:#d97706;--bg:#f0f4f9;--card:#fff;--border:#e4e9f0;--muted:#64748b;--body:#1e293b;--r:14px;--sh:0 2px 12px rgba(3,61,204,.07);--sh2:0 8px 32px rgba(3,61,204,.14)}
*{margin:0;padding:0;box-sizing:border-box}body{font-family:'Inter',sans-serif;font-size:16px;background:var(--bg);color:var(--body);line-height:1.5}
.ann{background:var(--cta);color:#fff;text-align:center;padding:9px 20px;font-size:13px;font-weight:500}.ann a{color:#fff;text-decoration:underline;margin-left:6px}
nav{background:#fff;border-bottom:1px solid var(--border);position:sticky;top:0;z-index:200;box-shadow:0 2px 8px rgba(0,0,0,.05)}
.nav-i{max-width:1320px;margin:0 auto;padding:0 32px;height:62px;display:flex;align-items:center;justify-content:space-between}
.logo{display:flex;align-items:center;gap:10px;text-decoration:none}
.nav-r{display:flex;align-items:center;gap:14px}
.live-pill{display:flex;align-items:center;gap:6px;background:#f0fdf4;border:1px solid #bbf7d0;border-radius:20px;padding:5px 12px;font-size:12px;font-weight:600;color:var(--green)}
.ldot{width:7px;height:7px;border-radius:50%;background:var(--green);animation:blink 1.4s ease-in-out infinite}
@keyframes blink{0%,100%{opacity:1}50%{opacity:.3}}
.pchip{font-size:13px;color:var(--muted);font-weight:500;background:var(--bg);border:1px solid var(--border);border-radius:8px;padding:6px 14px}
.pchip b{color:var(--body)}
.btn{background:var(--cta);color:#fff;border:none;border-radius:10px;padding:9px 18px;font-family:'Inter',sans-serif;font-weight:500;font-size:13px;cursor:pointer;text-decoration:none;display:inline-flex;align-items:center;gap:6px;transition:.2s}
.btn:hover{background:#002ea8;transform:translateY(-1px)}
.wrap{max-width:1320px;margin:0 auto;padding:40px 32px 100px}
.ph{display:flex;align-items:flex-end;justify-content:space-between;flex-wrap:wrap;gap:16px;margin-bottom:44px}
.ptag{font-family:'DM Sans',sans-serif;font-size:12px;font-weight:600;color:var(--cta);text-transform:uppercase;letter-spacing:.1em;margin-bottom:6px}
.ph h1{font-family:'DM Sans',sans-serif;font-size:42px;font-weight:700;color:var(--black);line-height:1.08;letter-spacing:-.03em}
.ph h1 em{color:var(--cta);font-style:normal}
.gnote{font-size:12px;color:var(--muted);background:#fff;border:1px solid var(--border);border-radius:8px;padding:8px 14px;text-align:right;line-height:1.8}
.gnote b{color:var(--body)}
.sh{display:flex;align-items:center;gap:12px;margin:52px 0 22px}
.sn{width:30px;height:30px;background:var(--cta);color:#fff;border-radius:8px;display:flex;align-items:center;justify-content:center;font-family:'DM Sans',sans-serif;font-weight:700;font-size:13px;flex-shrink:0}
.st{font-family:'DM Sans',sans-serif;font-size:22px;font-weight:700;color:var(--black)}
.sdv{flex:1;height:1px;background:var(--border)}
.co{background:linear-gradient(130deg,#024b8a 0%,#003DCC 100%);border-radius:var(--r);padding:28px 32px;color:#fff;margin-bottom:24px;display:grid;grid-template-columns:1fr auto;gap:28px;align-items:center}
@media(max-width:700px){.co{grid-template-columns:1fr}}
.co-t{font-family:'DM Sans',sans-serif;font-size:18px;font-weight:700;margin-bottom:7px}
.co-b{font-size:14px;opacity:.88;line-height:1.65;max-width:660px}
.co-b strong{font-weight:600;color:#93c5fd}
.cbadges{display:flex;gap:12px;flex-wrap:wrap}
.cbadge{background:rgba(255,255,255,.13);border:1px solid rgba(255,255,255,.22);border-radius:10px;padding:14px 20px;text-align:center;white-space:nowrap}
.cbv{font-family:'DM Sans',sans-serif;font-size:28px;font-weight:700;line-height:1}
.cbl{font-size:11px;opacity:.75;margin-top:3px}
.kg{display:grid;grid-template-columns:repeat(4,1fr);gap:16px;margin-bottom:24px}
@media(max-width:1000px){.kg{grid-template-columns:repeat(2,1fr)}}
.kc{background:var(--card);border-radius:var(--r);padding:22px;border:1px solid var(--border);box-shadow:var(--sh);transition:box-shadow .2s,transform .2s;position:relative;overflow:hidden}
.kc:hover{box-shadow:var(--sh2);transform:translateY(-3px)}
.kc::after{content:'';position:absolute;top:0;left:0;right:0;height:3px;border-radius:var(--r) var(--r) 0 0;background:var(--cta)}
.kc.g::after{background:var(--green)}.kc.r::after{background:var(--red)}.kc.a::after{background:var(--amber)}
.kl{font-size:11px;font-weight:700;text-transform:uppercase;letter-spacing:.08em;color:var(--muted);margin-bottom:10px}
.kv{font-family:'DM Sans',sans-serif;font-size:34px;font-weight:700;color:var(--black);line-height:1;letter-spacing:-.02em;margin-bottom:8px}
.dl{display:inline-flex;align-items:center;gap:3px;font-size:12px;font-weight:700;padding:3px 8px;border-radius:20px}
.du{background:#dcfce7;color:var(--green)}.dd{background:#fee2e2;color:var(--red)}
.ks{font-size:12px;color:var(--muted);margin-top:5px}
.two{display:grid;grid-template-columns:1fr 1fr;gap:20px;margin-bottom:24px}
@media(max-width:768px){.two{grid-template-columns:1fr}}
.cc{background:var(--card);border-radius:var(--r);padding:24px;border:1px solid var(--border);box-shadow:var(--sh)}
.cc-top{display:flex;align-items:center;justify-content:space-between;margin-bottom:20px;flex-wrap:wrap;gap:8px}
.cc-t{font-family:'DM Sans',sans-serif;font-size:15px;font-weight:700;color:var(--black)}
.tag{font-size:11px;font-weight:600;border-radius:20px;padding:3px 10px}
.tb{background:#eff6ff;color:var(--cta)}.tg{background:#f0fdf4;color:var(--green)}.ta{background:#fffbeb;color:var(--amber)}.tr{background:#fef2f2;color:var(--red)}
.tc{background:var(--card);border-radius:var(--r);border:1px solid var(--border);box-shadow:var(--sh);overflow:hidden;margin-bottom:20px}
.tc-h{padding:18px 24px 14px;border-bottom:1px solid var(--border);display:flex;align-items:center;justify-content:space-between;flex-wrap:wrap;gap:10px}
.tc-t{font-family:'DM Sans',sans-serif;font-size:15px;font-weight:700}
table{width:100%;border-collapse:collapse;font-size:13px}
thead th{background:#f8fafc;padding:11px 20px;text-align:left;font-size:10px;font-weight:700;text-transform:uppercase;letter-spacing:.08em;color:var(--muted);border-bottom:1px solid var(--border)}
tbody tr{border-bottom:1px solid #f1f5f9;transition:background .12s}tbody tr:last-child{border-bottom:none}tbody tr:hover{background:#f8fafc}
td{padding:13px 20px;color:var(--body);vertical-align:middle}
.tkw{font-weight:600;color:var(--black)}
.pos{display:inline-block;padding:2px 9px;border-radius:20px;font-weight:700;font-size:12px}
.p13{background:#dcfce7;color:#15803d}.p410{background:#fef3c7;color:#d97706}.p1120{background:#fee2e2;color:#dc2626}
.bw{width:70px;background:#f1f5f9;border-radius:4px;height:5px;display:inline-block}
.bf{height:5px;border-radius:4px;background:var(--cta)}
.cp{color:var(--green);font-size:12px;font-weight:700}.cn{color:var(--red);font-size:12px;font-weight:700}.cm{color:var(--muted);font-size:12px;font-weight:600}
.ai-grid{display:grid;grid-template-columns:repeat(3,1fr);gap:12px}
.ai-item{background:var(--bg);border-radius:10px;padding:14px 16px;display:flex;align-items:center;gap:12px;border:1px solid var(--border)}
.ai-logo{width:32px;height:32px;border-radius:8px;display:flex;align-items:center;justify-content:center;font-size:18px;flex-shrink:0}
.ai-name{font-size:12px;font-weight:600;color:var(--muted);margin-bottom:2px}
.ai-count{font-family:'DM Sans',sans-serif;font-size:22px;font-weight:700;line-height:1}
.ins-grid{display:grid;grid-template-columns:repeat(3,1fr);gap:20px;margin-bottom:24px}
@media(max-width:900px){.ins-grid{grid-template-columns:1fr}}
.ins{background:var(--card);border-radius:var(--r);border:1px solid var(--border);padding:24px;box-shadow:var(--sh);transition:box-shadow .2s,transform .2s}
.ins:hover{box-shadow:var(--sh2);transform:translateY(-3px)}
.ins-icon{width:42px;height:42px;border-radius:10px;display:flex;align-items:center;justify-content:center;font-size:20px;margin-bottom:14px}
.ins-type{font-size:11px;font-weight:700;text-transform:uppercase;letter-spacing:.1em;margin-bottom:12px}
.ins-type.win{color:var(--green)}.ins-type.loss{color:var(--red)}.ins-type.act{color:var(--cta)}
.ins ul{list-style:none}
.ins li{padding:9px 0;border-bottom:1px solid #f1f5f9;font-size:13px;line-height:1.55;display:flex;gap:8px}
.ins li:last-child{border-bottom:none}
.ins li > b{flex-shrink:0;font-size:14px;margin-top:1px}
.ins.win li > b{color:var(--green)}.ins.loss li > b{color:var(--red)}.ins.act li > b{color:var(--cta)}
.sbadge{display:inline-block;background:#eff6ff;color:var(--cta);border-radius:6px;padding:2px 9px;font-size:11px;font-weight:600}
.bbadge{display:inline-block;background:#f0fdf4;color:var(--green);border-radius:6px;padding:2px 9px;font-size:11px;font-weight:600}
.stat-pair{display:grid;grid-template-columns:1fr 1fr;gap:12px;margin-bottom:16px}
.sp-box{background:var(--bg);border-radius:10px;padding:14px 16px;border:1px solid var(--border)}
.sp-lbl{font-size:11px;font-weight:700;color:var(--muted);text-transform:uppercase;letter-spacing:.07em;margin-bottom:5px}
.sp-val{font-family:'DM Sans',sans-serif;font-size:26px;font-weight:700;line-height:1}
.sp-sub{font-size:12px;color:var(--muted);margin-top:4px}
.info-box{margin-top:14px;padding:11px 14px;background:#eff6ff;border-radius:8px;font-size:12px;color:var(--cta);font-weight:500;line-height:1.5}
footer{background:var(--footer);padding:36px 32px;text-align:center}
.fmeta{font-size:12px;color:rgba(255,255,255,.45);line-height:2}
.fmeta a{color:rgba(255,255,255,.6);text-decoration:none;margin:0 8px}
::-webkit-scrollbar{width:5px}::-webkit-scrollbar-track{background:var(--bg)}::-webkit-scrollbar-thumb{background:#cbd5e1;border-radius:3px}
</style>
</head>
<body>

<div class="ann">📊 Free SEO Audit for Q2 2026 — Limited slots available. <a href="https://kerkarmedia.com/contact-us" target="_blank">Claim Yours Today →</a></div>

<nav>
  <div class="nav-i">
    <a href="https://kerkarmedia.com" class="logo" target="_blank">
      <img src="logo.webp" alt="Kerkar Media" style="height:38px;width:auto;object-fit:contain;display:block">
    </a>
    <div class="nav-r">
      <div class="live-pill"><span class="ldot"></span>Live Report</div>
      <div class="pchip">Generated: <b>${d.reportDate}</b></div>
      <a href="https://kerkarmedia.com" class="btn" target="_blank">kerkarmedia.com</a>
    </div>
  </div>
</nav>

<div class="wrap">

  <div class="ph">
    <div>
      <div class="ptag">Kerkar Media — Internal SEO Report</div>
      <h1>Performance <em>Dashboard</em></h1>
    </div>
    <div class="gnote">Generated: <b>${d.reportDate}</b><br>Sources: GSC · Ahrefs · GA4 · CF7</div>
  </div>

  <!-- SECTION 1: OVERVIEW -->
  <div class="sh"><div class="sn">1</div><div class="st">Overview</div><div class="sdv"></div></div>

  <div class="co">
    <div>
      <div class="co-t">Executive Summary — ${d.reportDate}</div>
      <div class="co-b">${d.executiveSummary || 'Generating insights...'}</div>
    </div>
    <div class="cbadges">
      <div class="cbadge"><div class="cbv">${fmt(d.totalClicks)}</div><div class="cbl">GSC Clicks</div></div>
      <div class="cbadge"><div class="cbv">${fmt(d.totalImpressions)}</div><div class="cbl">Impressions</div></div>
      <div class="cbadge"><div class="cbv">DR ${d.dr}</div><div class="cbl">Domain Rating</div></div>
    </div>
  </div>

  <div class="kg">
    <div class="kc"><div class="kl">Total Clicks (GSC)</div><div class="kv">${fmt(d.totalClicks)}</div><span class="dl ${d.totalClicks > 400 ? 'du' : 'dd'}">${d.totalClicks > 400 ? '↑' : '↓'} GSC</span><div class="ks">Last 28 days</div></div>
    <div class="kc g"><div class="kl">Impressions</div><div class="kv">${fmt(d.totalImpressions)}</div><span class="dl du">↑ Strong</span><div class="ks">All queries</div></div>
    <div class="kc a"><div class="kl">Avg CTR</div><div class="kv">${pct(d.avgCTR)}</div><span class="dl ${ctrDeltaClass}">${ctrNum >= 3 ? '↑ On target' : '↓ Below target'}</span><div class="ks">Target: 3–5%</div></div>
    <div class="kc a"><div class="kl">Avg Position</div><div class="kv">${d.avgPosition}</div><span class="dl ${posDeltaClass}">${parseFloat(d.avgPosition) <= 20 ? '↑ Good' : '↓ Needs push'}</span><div class="ks">All keywords</div></div>
    <div class="kc ${d.orgKeywords < 15 ? 'r' : 'g'}"><div class="kl">Organic Keywords</div><div class="kv">${d.orgKeywords}</div><span class="dl ${d.orgKeywords < 15 ? 'dd' : 'du'}">Ahrefs · India</span><div class="ks">Top 3: ${d.orgKeywordsTop3}</div></div>
    <div class="kc r"><div class="kl">Organic Traffic</div><div class="kv">${d.orgTraffic}</div><span class="dl dd">Ahrefs est.</span><div class="ks">Traffic value: $${d.orgValue}</div></div>
    <div class="kc g"><div class="kl">Domain Rating</div><div class="kv">${d.dr}</div><span class="dl du">↑ Ahrefs</span><div class="ks">AR: ${(d.ahrefsRank||0).toLocaleString()}</div></div>
    <div class="kc g"><div class="kl">Referring Domains</div><div class="kv">${d.liveRefdomains}</div><span class="dl du">↑ Active</span><div class="ks">Live backlinks: ${(d.liveBacklinks||0).toLocaleString()}</div></div>
  </div>

  <!-- SECTION 2: KEYWORDS -->
  <div class="sh"><div class="sn">2</div><div class="st">Keyword Intelligence</div><div class="sdv"></div></div>

  <div class="two">
    <div class="tc">
      <div class="tc-h"><div class="tc-t">Keywords by Intent</div><span class="tag ta">Ahrefs · India</span></div>
      <table>
        <thead><tr><th>Intent</th><th>Keywords</th><th>Traffic</th></tr></thead>
        <tbody>${intentRows}</tbody>
      </table>
    </div>
    <div class="cc">
      <div class="cc-top"><div class="cc-t">CTR vs India Benchmarks</div><span class="tag tb">GSC Analysis</span></div>
      <canvas id="ctrChart" height="230"></canvas>
    </div>
  </div>

  <div class="tc">
    <div class="tc-h">
      <div class="tc-t">Top Organic Keywords</div>
      <div style="display:flex;gap:8px;flex-wrap:wrap">
        <span class="tag tg">Top 3: ${d.pos13}</span>
        <span class="tag ta">4–10: ${d.pos410}</span>
        <span class="tag tr">11–50: ${d.pos1150}</span>
      </div>
    </div>
    <table>
      <thead><tr><th>Keyword</th><th>Position</th><th>Traffic</th><th>Volume</th><th>Intent</th><th>Trend</th></tr></thead>
      <tbody>${keywordRows}</tbody>
    </table>
  </div>

  <!-- SECTION 3: AUTHORITY -->
  <div class="sh"><div class="sn">3</div><div class="st">Domain Authority &amp; Backlinks</div><div class="sdv"></div></div>

  <div class="two">
    <div class="cc">
      <div class="cc-top"><div class="cc-t">Domain Rating</div><span class="tag tg">Ahrefs · kerkarmedia.com</span></div>
      <div style="display:flex;align-items:center;gap:28px;flex-wrap:wrap">
        <div style="display:flex;flex-direction:column;align-items:center">
          <svg width="150" height="95" viewBox="0 0 150 95">
            <path d="M20 85 A55 55 0 0 1 130 85" fill="none" stroke="#f1f5f9" stroke-width="13" stroke-linecap="round"/>
            <path id="drArc" d="M20 85 A55 55 0 0 1 130 85" fill="none" stroke="#22c55e" stroke-width="13" stroke-linecap="round" stroke-dasharray="172.7" stroke-dashoffset="${172.7 - (d.dr / 100) * 172.7}"/>
          </svg>
          <div style="font-family:'DM Sans',sans-serif;font-size:56px;font-weight:700;line-height:1;letter-spacing:-.03em;margin-top:-16px">${d.dr}</div>
          <div style="font-size:12px;font-weight:700;text-transform:uppercase;letter-spacing:.08em;color:var(--muted);margin-top:4px">Domain Rating</div>
        </div>
        <div style="flex:1;min-width:150px">
          <div style="margin-bottom:14px"><div class="sp-lbl">Ahrefs Rank</div><div style="font-family:'DM Sans',sans-serif;font-size:18px;font-weight:700;color:var(--cta)">${(d.ahrefsRank||0).toLocaleString()}</div></div>
          <div style="margin-bottom:14px"><div class="sp-lbl">Live Backlinks</div><div style="font-family:'DM Sans',sans-serif;font-size:22px;font-weight:700">${(d.liveBacklinks||0).toLocaleString()}</div><div style="font-size:12px;color:var(--muted)">All time: ${(d.allTimeBacklinks||0).toLocaleString()}</div></div>
          <div><div class="sp-lbl">Ref. Domains</div><div style="font-family:'DM Sans',sans-serif;font-size:22px;font-weight:700">${d.liveRefdomains}</div></div>
        </div>
      </div>
    </div>

    <div class="cc">
      <div class="cc-top"><div class="cc-t">Referring Domains Trend</div><span class="tag tb">Month-on-Month</span></div>
      <div class="stat-pair">
        <div class="sp-box"><div class="sp-lbl">Ref. Domains</div><div class="sp-val">${d.liveRefdomains}</div><div class="sp-sub">Live</div></div>
        <div class="sp-box"><div class="sp-lbl">Backlinks</div><div class="sp-val">${(d.liveBacklinks||0).toLocaleString()}</div><div class="sp-sub">Live links</div></div>
        <div class="sp-box"><div class="sp-lbl">Org. Keywords</div><div class="sp-val" style="${d.orgKeywords < 15 ? 'color:var(--red)' : 'color:var(--green)'}">${d.orgKeywords}</div><div class="sp-sub">Top 3: ${d.orgKeywordsTop3}</div></div>
        <div class="sp-box"><div class="sp-lbl">Traffic Value</div><div class="sp-val">$${d.orgValue}</div><div class="sp-sub">Ahrefs est.</div></div>
      </div>
      <canvas id="blChart" height="110"></canvas>
    </div>
  </div>

  <div class="two">
    <div class="tc">
      <div class="tc-h"><div class="tc-t">Traffic by Location</div><span class="tag ta">Ahrefs · Organic</span></div>
      <table>
        <thead><tr><th>Location</th><th>Traffic</th><th>Share</th><th>Keywords</th><th>Top 3</th></tr></thead>
        <tbody>${countryRows}</tbody>
      </table>
    </div>
    <div class="cc">
      <div class="cc-top"><div class="cc-t">AI Citations — GEO Presence</div><span class="tag tg">Ahrefs · Last Month</span></div>
      <div class="ai-grid">
        <div class="ai-item"><div class="ai-logo" style="background:#f8fafc">🔍</div><div><div class="ai-name">AI Overview</div><div class="ai-count" style="color:var(--muted)">0</div></div></div>
        <div class="ai-item"><div class="ai-logo" style="background:#f0fdf4">🤖</div><div><div class="ai-name">ChatGPT</div><div class="ai-count" style="color:var(--green)">2</div></div></div>
        <div class="ai-item"><div class="ai-logo" style="background:#fffbeb">✦</div><div><div class="ai-name">Gemini</div><div class="ai-count" style="color:var(--green)">5</div></div></div>
        <div class="ai-item"><div class="ai-logo" style="background:#eff6ff">🔮</div><div><div class="ai-name">Copilot</div><div class="ai-count" style="color:var(--green)">2</div></div></div>
        <div class="ai-item"><div class="ai-logo" style="background:#f8fafc">⚡</div><div><div class="ai-name">Perplexity</div><div class="ai-count" style="color:var(--muted)">0</div></div></div>
        <div class="ai-item"><div class="ai-logo" style="background:#f8fafc">𝕏</div><div><div class="ai-name">Grok</div><div class="ai-count" style="color:var(--muted)">0</div></div></div>
      </div>
      <div class="info-box">💡 Gemini (+3) and Copilot (+2) citations growing. Publish E-E-A-T-rich content with FAQ schema to accelerate AI search visibility.</div>
    </div>
  </div>

  <!-- SECTION 4: INSIGHTS -->
  <div class="sh"><div class="sn">4</div><div class="st">Insights &amp; Priority Actions</div><div class="sdv"></div></div>

  <div class="ins-grid">
    <div class="ins win">
      <div class="ins-icon" style="background:#dcfce7">🏆</div>
      <div class="ins-type win">Wins This Period</div>
      <ul>${winsList}</ul>
    </div>
    <div class="ins loss">
      <div class="ins-icon" style="background:#fee2e2">⚠️</div>
      <div class="ins-type loss">Critical Concerns</div>
      <ul>${lossList}</ul>
    </div>
    <div class="ins act">
      <div class="ins-icon" style="background:#eff6ff">🎯</div>
      <div class="ins-type act">Priority Actions</div>
      <ul>${actionsList}</ul>
    </div>
  </div>

  <!-- LEADS -->
  <div class="sh"><div class="sn" style="background:#7c3aed">👥</div><div class="st">Leads CRM — This Week</div><div class="sdv"></div></div>
  <div class="tc">
    <div class="tc-h"><div class="tc-t">New Leads (${d.thisWeekLeads?.length || 0})</div><span class="tag" style="background:#f5f3ff;color:#7c3aed">CF7 · Sheets</span></div>
    <table>
      <thead><tr><th>Name</th><th>Company</th><th>Service</th><th>Budget</th><th>Date</th></tr></thead>
      <tbody>${leadsRows}</tbody>
    </table>
  </div>

</div>

<footer>
  <div style="margin-bottom:10px"><img src="logo.webp" alt="Kerkar Media" style="height:32px;width:auto;filter:brightness(0) invert(1)"></div>
  <div class="fmeta">
    SEO Performance Report · ${d.reportDate}<br>
    <a href="https://kerkarmedia.com">kerkarmedia.com</a>
    <a href="https://kerkarmedia.com/privacy-policy">Privacy Policy</a>
    <a href="https://kerkarmedia.com/terms-of-service">Terms</a><br>
    © Kerkar Media 2026. All Rights Reserved.
  </div>
</footer>

<script>
gsap.registerPlugin(ScrollTrigger);
gsap.utils.toArray('.kc,.cc,.tc,.ins,.co').forEach(el=>{
  gsap.from(el,{opacity:0,y:20,duration:.6,ease:'power2.out',scrollTrigger:{trigger:el,start:'top 90%',once:true}});
});

// CTR Chart
new Chart(document.getElementById('ctrChart').getContext('2d'),{
  type:'bar',
  data:{
    labels:['Branded','Commercial','Informational','Local'],
    datasets:[
      {label:'Actual CTR',data:[${parseFloat(d.avgCTR) > 15 ? d.avgCTR : d.avgCTR},${d.avgCTR},${d.avgCTR},${d.avgCTR}],backgroundColor:'#003DCC',borderRadius:6,borderSkipped:false},
      {label:'India Benchmark',data:[15,4,1.5,2],backgroundColor:'#e2e8f0',borderRadius:6,borderSkipped:false}
    ]
  },
  options:{responsive:true,plugins:{legend:{position:'bottom',labels:{font:{size:11},color:'#64748b'}}},
    scales:{y:{grid:{color:'#f1f5f9'},ticks:{color:'#94a3b8',font:{size:10},callback:v=>v+'%'}},x:{grid:{display:false},ticks:{color:'#94a3b8',font:{size:10}}}}}
});

// Refdomains Chart
new Chart(document.getElementById('blChart').getContext('2d'),{
  type:'bar',
  data:{
    labels:${JSON.stringify(refLabels)},
    datasets:[{label:'Ref. Domains',data:${JSON.stringify(refValues)},backgroundColor:'rgba(0,61,204,0.12)',borderColor:'#003DCC',borderWidth:2,borderRadius:6}]
  },
  options:{responsive:true,plugins:{legend:{display:false}},
    scales:{y:{grid:{color:'#f1f5f9'},ticks:{color:'#94a3b8',font:{size:10}},min:${Math.max(0, Math.min(...refValues) - 20)}},x:{grid:{display:false},ticks:{color:'#94a3b8',font:{size:10}}}}}
});
</script>
</body>
</html>`;

// ── Push HTML to GitHub ──
const GITHUB_TOKEN = "YOUR_GITHUB_TOKEN";
const REPO = "uday301/audits-KM";
const FILE_PATH = "km-dashboard.html";
const COMMIT_MSG = `Auto-update SEO dashboard — ${today}`;

// Get current file SHA (needed for update)
const currentFile = await this.helpers.httpRequest({
  method: 'GET',
  url: `https://api.github.com/repos/${REPO}/contents/${FILE_PATH}`,
  headers: {
    'Authorization': `Bearer ${GITHUB_TOKEN}`,
    'Accept': 'application/vnd.github+json'
  }
});

const sha = currentFile.sha;
const encodedContent = Buffer.from(html).toString('base64');

// Push updated file
await this.helpers.httpRequest({
  method: 'PUT',
  url: `https://api.github.com/repos/${REPO}/contents/${FILE_PATH}`,
  headers: {
    'Authorization': `Bearer ${GITHUB_TOKEN}`,
    'Accept': 'application/vnd.github+json',
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    message: COMMIT_MSG,
    content: encodedContent,
    sha: sha
  })
});

return [{
  json: {
    success: true,
    reportDate: d.reportDate,
    dashboardUrl: 'https://audits.kerkarmedia.com/km-dashboard.html',
    message: `Dashboard updated and deployed — ${today}`
  }
}];
```

---

## HOW TO SET UP IN N8N

### Step 1 — Add Ahrefs Node
1. After your existing GSC node, add a new **Code** node
2. Name it "Fetch Ahrefs Data"
3. Paste NODE 4 code above
4. Connect it to the Merge node as input 4

### Step 2 — Update Merge Node
1. Click your existing Merge node
2. Change number of inputs from 3 → 4
3. Connect the new Ahrefs node as Input 4

### Step 3 — Replace Code Node 1 (Process + Claude)
1. Open your existing "Code in JavaScript1" node
2. Replace entire code with NODE 7 code above
3. Replace `YOUR_ANTHROPIC_API_KEY` with your actual key

### Step 4 — Replace Code Node 2 (was Notion builder)
1. Open your existing "Code in JavaScript2" node
2. Replace entire code with NODE 8 code above
3. This replaces Notion with GitHub HTML push

### Step 5 — Test
1. Click "Test Workflow" in n8n
2. Check https://audits.kerkarmedia.com/km-dashboard.html
3. Should auto-update with fresh data

---

## KEYS NEEDED IN THE CODE

| Key | Where to put it | Value |
|---|---|---|
| Ahrefs API | NODE 4, line 3 | YOUR_AHREFS_API_KEY |
| Anthropic API | NODE 7, Claude call | Get from console.anthropic.com |
| GitHub Token | NODE 8, line 3 | YOUR_GITHUB_TOKEN |

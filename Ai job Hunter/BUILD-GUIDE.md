# Job Hunter: build Versions 2 to 8 yourself

Work on your working Version 1. After every version: run the workflow, check the nodes are green, and download a copy of the workflow (workflow menu, three dots, Download). Name the copies v2, v3 and so on. Those files are your project history.

**Before you start**
- While testing, set `maxToScore` in Settings to 3. Every run uses your free Gemini quota.
- n8n renames references automatically, but if a node says it cannot find a node name, edit that expression.
- Remotive asks for at most 4 requests a day, so avoid running the whole workflow more than 4 times a day while testing.
- Tip: to insert a node on an existing line, hover the line and click the plus.
- Code nodes: set Mode to **Run Once for All Items** and Language to **JavaScript**.

---

## Sources: what is used and what is not

**Used (free public feeds, no key):** Remotive, Arbeitnow, Jobicy, Remote OK, We Work Remotely, Himalayas, Working Nomads.

**Not used, and why**
- FlexJobs: paid subscription.
- Upwork, Freelancer, Fiverr, Guru, PeoplePerHour, Toptal, Contra, Arc, Braintrust, Codeable, Lemon.io, Crossover, Andela, Hubstaff Talent, Outsourcely: freelance or application-based talent networks, not job feeds.
- Clickworker, Appen, DataAnnotation, Outlier, Toloka, UserTesting, Respondent: task and testing platforms, not jobs.
- Wellfound, Y Combinator, Startup.jobs, Crypto Jobs List, Web3 Career, PowerToFly and the small remote boards: no public feed that I could confirm. Scraping them is blocked or against their terms.
- To add one later, find its RSS or JSON feed first and send it to me.

---

## Version 2: Settings (configurable)

1. Double-click **My CV**. Click its name at the top and rename it **Settings**.
2. Keep the `cv` field. Click **Add Field** for each of these. For the numbers, change the type dropdown (it says String) to **Number**.

| Field | Type | Value |
|---|---|---|
| jobCategory | String | software-dev |
| jobLimit | Number | 20 |
| minimumScore | Number | 60 |
| maxToScore | Number | 10 |
| notifyEmail | String | your email address |
| model | String | gemini-flash-latest |
| keywords | String | developer,software,automation,ai,web,frontend,backend,full stack,engineer,technician,electrical,electronics,intern,junior,graduate,trainee |
| excludeTitleWords | String | senior,lead,principal,staff,manager,director,head of,vp,architect |
| targetRoles | String | junior or entry-level software, AI automation or web developer; graduate trainee; internship; electrical/electronics technician |
| candidateLocation | String | Benin City, Nigeria (open to Warri, Delta State, or remote) |
| searchTerm | String | junior developer |

3. Click **Execute step** and check the Output shows every field.
4. Rename **Get Jobs** to **Fetch Remotive**. In its URL box, click **Expression** and enter:
```
https://remotive.com/api/remote-jobs?category={{ $('Settings').first().json.jobCategory }}&limit={{ $('Settings').first().json.jobLimit }}
```
5. **Limit to 10**: set Max Items to Expression: `{{ $('Settings').first().json.maxToScore }}`
6. **Score with Gemini**: set the URL to Expression and enter the line below. This removes the key from the URL.
```
https://generativelanguage.googleapis.com/v1beta/models/{{ $('Settings').first().json.model }}:generateContent
```
   Then set Authentication to **Generic Credential Type**, Generic Auth Type to **Header Auth**, and create a credential: Name `x-goog-api-key`, Value your Gemini key. Name the credential "Gemini API Key".
7. Rename the **Score 60 or above** node to **Passes Threshold**. Set its right value to Expression: `{{ $('Settings').first().json.minimumScore }}`. If it complains about the type, open Options and turn on loose type validation.
8. **Send Email**: set To to Expression: `{{ $('Settings').first().json.notifyEmail }}`
9. Run the workflow. Download the copy as v2.

---

## Version 3: Seven job sources

1. Select **Fetch Remotive**, press Ctrl+C, then Ctrl+V six times. Rename the copies exactly as below. In each copy, switch the URL box back to **Fixed** (except Himalayas, which stays an **Expression**) and enter:

| Node name | URL |
|---|---|
| Fetch Arbeitnow | `https://www.arbeitnow.com/api/job-board-api` |
| Fetch Jobicy | `https://jobicy.com/api/v2/remote-jobs?count=30` |
| Fetch Remote OK | `https://remoteok.com/api` |
| Fetch We Work Remotely | `https://weworkremotely.com/remote-jobs.rss` |
| Fetch Himalayas | `https://himalayas.app/jobs/api/search?q={{ $('Settings').first().json.searchTerm }}&sort=recent` |
| Fetch Working Nomads | `https://www.workingnomads.com/api/exposed_jobs/` |

2. On **all seven** fetch nodes (including Fetch Remotive):
   - Open **Options**, click **Add Option**, choose **Response**, set Response Format to **Text** and Put Output in Field to `data`. Some feeds return a list, and n8n would split a list into many items and run every later node many times. Text keeps it to one item per source.
   - Open the **Settings** tab: turn on **Retry On Fail** (2 tries, 3000 ms) and set **On Error** to **Continue**.
3. On **Fetch Remote OK** and **Fetch We Work Remotely** also turn on **Send Headers** and add the header name `User-Agent` with the value `Mozilla/5.0 (compatible; JobHunterAgent/1.0)`.
4. Connect all seven in a row: Settings, Fetch Remotive, Fetch Arbeitnow, Fetch Jobicy, Fetch Remote OK, Fetch We Work Remotely, Fetch Himalayas, Fetch Working Nomads.
5. Delete the **Split Jobs** node. Add a Code node after **Fetch Working Nomads**, name it exactly **Normalize & Dedupe**, connect it to **Limit to 10**, and paste:
```javascript
const s = $('Settings').first().json;
const list = x => String(x || '').split(',').map(t => t.trim().toLowerCase()).filter(Boolean);
const keywords = list(s.keywords);
const blocked = list(s.excludeTitleWords);
const maxToScore = Number(s.maxToScore) || 10;
let seenUrls = new Set();
try { seenUrls = new Set($('Collect Seen URLs').first().json.seenUrls || []); } catch (e) {}

const strip = h => String(h || '').replace(/<!\[CDATA\[|\]\]>/g, '').replace(/<[^>]*>/g, ' ').replace(/&nbsp;|&amp;|&quot;|&#\d+;/g, ' ').replace(/\s+/g, ' ').trim();
const iso = v => {
  if (!v) return '';
  const d = typeof v === 'number' ? new Date(v > 1e12 ? v : v * 1000) : new Date(v);
  return isNaN(d) ? '' : d.toISOString();
};
const load = node => {
  const items = $(node).all().map(i => i.json);
  if (!items.length || items[0].error) throw new Error('source error');
  if (typeof items[0].data === 'string') {
    const t = items[0].data.trim();
    return t.startsWith('<') ? t : JSON.parse(t);
  }
  return items.length > 1 ? items : items[0];
};
const tag = (xml, name) => {
  const m = xml.match(new RegExp('<' + name + '[^>]*>([\\s\\S]*?)</' + name + '>'));
  return m ? m[1] : '';
};
const rssItems = xml => (typeof xml === 'string' ? (xml.match(/<item>[\s\S]*?<\/item>/g) || []) : []);

const sources = [
  { node: 'Fetch Remotive', label: 'Remotive', pick: r => r.jobs,
    map: j => ({ id: 'remotive-' + j.id, title: j.title, company: j.company_name, location: j.candidate_required_location || 'Remote', url: j.url, description: j.description, posted: iso(j.publication_date) }) },
  { node: 'Fetch Arbeitnow', label: 'Arbeitnow', pick: r => r.data,
    map: j => ({ id: 'arbeitnow-' + j.slug, title: j.title, company: j.company_name || j.companyName, location: (j.location || '') + (j.remote ? ' (Remote)' : ''), url: j.url || j.jobUrl, description: j.description, posted: iso(j.created_at) }) },
  { node: 'Fetch Jobicy', label: 'Jobicy', pick: r => r.jobs,
    map: j => ({ id: 'jobicy-' + j.id, title: j.jobTitle, company: j.companyName, location: j.jobGeo || 'Remote', url: j.url, description: j.jobDescription || j.jobExcerpt, posted: iso(j.pubDate) }) },
  { node: 'Fetch Remote OK', label: 'Remote OK', pick: r => (Array.isArray(r) ? r : []).filter(j => j && j.id && j.position),
    map: j => ({ id: 'remoteok-' + j.id, title: j.position, company: j.company, location: j.location || 'Remote', url: j.url || j.apply_url, description: j.description, posted: iso(j.date) }) },
  { node: 'Fetch We Work Remotely', label: 'We Work Remotely', pick: rssItems,
    map: x => {
      const full = strip(tag(x, 'title'));
      const i = full.indexOf(':');
      return { id: 'wwr-' + strip(tag(x, 'guid') || tag(x, 'link')), title: i > 0 ? full.slice(i + 1).trim() : full, company: i > 0 ? full.slice(0, i).trim() : '', location: strip(tag(x, 'region')) || 'Remote', url: strip(tag(x, 'link')), description: tag(x, 'description'), posted: iso(strip(tag(x, 'pubDate'))) };
    } },
  { node: 'Fetch Himalayas', label: 'Himalayas', pick: r => r.jobs,
    map: j => ({ id: 'himalayas-' + (j.guid || j.applicationLink || j.title), title: j.title, company: j.companyName || j.company, location: Array.isArray(j.locationRestrictions) && j.locationRestrictions.length ? j.locationRestrictions.join(', ') : 'Worldwide', url: j.applicationLink || j.url || j.guid, description: j.description || j.excerpt, posted: iso(j.pubDate) }) },
  { node: 'Fetch Working Nomads', label: 'Working Nomads', pick: r => (Array.isArray(r) ? r : []).filter(j => j && j.title && !j.expired),
    map: j => ({ id: 'workingnomads-' + j.url, title: j.title, company: j.company_name, location: j.location || 'Remote', url: j.url, description: j.description, posted: iso(j.pub_date) }) },
];

const stats = { sources: {}, fetched: 0, tooSenior: 0, offTopic: 0, duplicates: 0, alreadySeen: 0, queued: 0 };
const pool = [];
const keys = new Set();

for (const src of sources) {
  let raw = [];
  try {
    raw = src.pick(load(src.node)) || [];
    stats.sources[src.label] = raw.length;
  } catch (e) {
    stats.sources[src.label] = 'failed';
    continue;
  }
  stats.fetched += raw.length;
  for (const r of raw) {
    const j = src.map(r);
    j.source = src.label;
    j.title = strip(j.title);
    j.company = strip(j.company);
    j.description = strip(j.description).slice(0, 1800);
    if (!j.title || !j.url) continue;
    const title = j.title.toLowerCase();
    if (blocked.some(b => title.includes(b))) { stats.tooSenior++; continue; }
    const hits = keywords.filter(k => (title + ' ' + j.description.toLowerCase()).includes(k)).length;
    if (keywords.length && hits === 0) { stats.offTopic++; continue; }
    const key = (j.title + '|' + j.company).toLowerCase();
    if (keys.has(key) || keys.has(j.url)) { stats.duplicates++; continue; }
    keys.add(key); keys.add(j.url);
    if (seenUrls.has(j.url)) { stats.alreadySeen++; continue; }
    j.hits = hits;
    pool.push(j);
  }
}

pool.sort((a, b) => b.hits - a.hits || String(b.posted).localeCompare(String(a.posted)));
stats.queued = Math.min(pool.length, maxToScore);
return pool.map(j => ({ json: { ...j, company_name: j.company, _stats: stats } }));
```
6. Run it. Open Normalize & Dedupe and check the jobs show a `source` and the stats list each source with its count. A source shown as "failed" is skipped and the run continues. If one fails, open that fetch node and read its output.
7. Notes:
   - Press **D** on a selected fetch node to disable it while testing. It will then report 0 jobs.
   - Himalayas, Working Nomads and We Work Remotely field names were confirmed from descriptions, not from a live run, so check each source returns jobs on your first run. If one shows 0, send me a photo of its output and I will fix the mapping.
   - Remote OK and Remotive require you to link back and name them as the source. The email footer and your README do that.
   - We Work Remotely feeds are for personal job hunting only.

---

## Version 4: Skip jobs you already have

1. Hover the line between **Settings** and **Fetch Remotive** and click the plus. Add **Google Sheets**, operation **Get Row(s) in sheet**, using your credential, document and sheet. Name it **Read Past Jobs**. In its Settings tab turn on **Always Output Data**.
2. After it, insert a Code node named exactly **Collect Seen URLs**:
```javascript
const urls = $input.all().map(i => i.json.url).filter(Boolean);
return [{ json: { seenUrls: urls } }];
```
3. Run the workflow twice. On the second run, the Normalize stats should show `alreadySeen` above 0.
4. Note: until Version 5, only jobs above the threshold are saved, so low scorers can reappear. Version 5 fixes this.

---

## Version 5: Richer AI analysis

1. Add a Code node between **Limit to 10** and **Score with Gemini**, named exactly **Build Prompt**:
```javascript
const s = $('Settings').first().json;
const minScore = Number(s.minimumScore) || 60;
return $input.all().map(item => {
  const j = item.json;
  const system = 'You are a pragmatic technical recruiter judging job fit for a junior candidate. Be honest and calibrated, never inflate scores, and never invent facts about the candidate. Reply with JSON only.';
  const user = [
    'CANDIDATE CV:', s.cv, '',
    'CANDIDATE TARGETS:',
    'Roles: ' + s.targetRoles,
    'Location: ' + s.candidateLocation, '',
    'JOB:',
    'Title: ' + j.title,
    'Company: ' + j.company,
    'Location: ' + j.location,
    'Source: ' + j.source,
    'Description: ' + j.description, '',
    'SCORING RUBRIC (total 0-100):',
    '- Skills and tools overlap with the CV: 40 points',
    '- Seniority fit: junior, entry-level, graduate and internship roles score high; roles needing 3+ years of experience score at most 35 overall: 25 points',
    '- Location and work authorization: worldwide-remote or Africa-friendly is fine; if the job requires residency or work authorization in the US, UK or EU, cap the total score at 40: 20 points',
    '- Relevance to the candidate target roles: 15 points', '',
    'OUTPUT RULES (every list has at most 3 items, except skills which have at most 5):',
    '- matched_skills: skills from the CV that the job needs',
    '- missing_skills: job requirements that are not in the CV',
    '- strengths: why the candidate is a good fit',
    '- weaknesses: where the candidate is weaker for this job',
    '- resume_improvements: specific edits to the CV for this job',
    '- keywords: important keywords from the job post to add to the CV',
    '- interview_tips: what to prepare for the interview',
    '- reason: one sentence explaining the score',
    '- cover_letter: if score >= ' + minScore + ', write a tailored cover letter of about 120 words using only facts from the CV; otherwise an empty string',
  ].join('\n');
  const arr = { type: 'ARRAY', items: { type: 'STRING' } };
  const requestBody = {
    systemInstruction: { parts: [{ text: system }] },
    contents: [{ parts: [{ text: user }] }],
    generationConfig: {
      temperature: 0.2,
      responseMimeType: 'application/json',
      responseSchema: {
        type: 'OBJECT',
        properties: {
          score: { type: 'INTEGER' },
          fit: { type: 'STRING', enum: ['strong', 'moderate', 'weak'] },
          matched_skills: arr, missing_skills: arr, strengths: arr, weaknesses: arr,
          resume_improvements: arr, keywords: arr, interview_tips: arr,
          reason: { type: 'STRING' }, cover_letter: { type: 'STRING' },
        },
        required: ['score', 'fit', 'matched_skills', 'missing_skills', 'strengths', 'weaknesses', 'resume_improvements', 'keywords', 'interview_tips', 'reason', 'cover_letter'],
      },
    },
  };
  return { json: { ...j, requestBody } };
});
```
2. In **Score with Gemini**, set Specify Body to **Using JSON**, and set the JSON box to Expression: `{{ JSON.stringify($json.requestBody) }}` (replace the old body).
3. Replace the code in **Parse Scores** with:
```javascript
const jobs = $('Build Prompt').all();
const today = new Date().toISOString().slice(0, 10);
const join = v => (Array.isArray(v) ? v.join('; ') : '');
const out = [];

$input.all().forEach((item, i) => {
  const job = jobs[i].json;
  const base = { date: today, source: job.source, title: job.title, company: job.company, location: job.location, url: job.url };
  let r = null;
  try {
    r = JSON.parse(item.json.candidates[0].content.parts[0].text);
  } catch (e) {}
  if (!r) {
    out.push({ json: { ...base, score: 0, fit: 'error', matched_skills: '', missing_skills: '', strengths: '', weaknesses: '', resume_improvements: '', keywords: '', interview_tips: '', reason: 'Scoring failed', cover_letter: '', application_status: 'New', status: 'failed' } });
    return;
  }
  const score = Math.max(0, Math.min(100, Math.round(Number(r.score) || 0)));
  out.push({ json: { ...base, score, fit: r.fit || '', matched_skills: join(r.matched_skills), missing_skills: join(r.missing_skills), strengths: join(r.strengths), weaknesses: join(r.weaknesses), resume_improvements: join(r.resume_improvements), keywords: join(r.keywords), interview_tips: join(r.interview_tips), reason: r.reason || '', cover_letter: r.cover_letter || '', application_status: 'New', status: 'scored' } });
});
return out;
```
4. Put these headers in row 1 of your Google Sheet (a new tab is cleanest):
```
date, source, title, company, location, url, score, fit, matched_skills, missing_skills, strengths, weaknesses, resume_improvements, keywords, interview_tips, reason, cover_letter, application_status, status
```
5. Change the filter in front of **Save to Google Sheets** so it keeps every successfully scored job: condition `{{ $json.status }}` (String) is equal to `scored`. Rename it **Scored OK**. Now all scored jobs are saved and Version 4 skips them tomorrow.
6. Run it. Check the Sheet has the new columns.

---

## Version 6: Daily report email

1. Delete the old **Build Email** node. Add a Code node named exactly **Build Digest**, connect it **from Parse Scores** (not from the filter), and paste:
```javascript
const s = $('Settings').first().json;
const minScore = Number(s.minimumScore) || 60;
const stats = $('Normalize & Dedupe').first().json._stats || {};
const rows = $input.all().map(i => i.json);
const scored = rows.filter(r => r.status === 'scored');
const failed = rows.length - scored.length;
const matches = scored.filter(r => r.score >= minScore).sort((a, b) => b.score - a.score);
const avg = scored.length ? Math.round(scored.reduce((t, r) => t + r.score, 0) / scored.length) : 0;
const best = scored.length ? Math.max(...scored.map(r => r.score)) : 0;
const topCompanies = [...new Set(matches.map(m => m.company))].slice(0, 3).join(', ') || 'none';
const esc = x => String(x == null ? '' : x).replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;');
const date = new Date().toISOString().slice(0, 10);
const src = Object.entries(stats.sources || {}).map(([k, v]) => k + ': ' + v).join(' | ');

let html = '<h2>Daily Job Report - ' + date + '</h2>';
html += '<p><b>Jobs fetched:</b> ' + (stats.fetched || 0) + '<br><b>Scored by AI:</b> ' + scored.length + '<br><b>Qualified (' + minScore + '+):</b> ' + matches.length + '<br><b>Highest match:</b> ' + best + '%<br><b>Average score:</b> ' + avg + '%<br><b>Top companies:</b> ' + esc(topCompanies) + '<br><b>Sources:</b> ' + esc(src) + (failed ? '<br><b>Warning:</b> ' + failed + ' job(s) failed to score' : '') + '</p>';

if (!matches.length) {
  html += '<p>No jobs reached the ' + minScore + ' point threshold today.</p>';
} else {
  html += '<table border="1" cellpadding="6" cellspacing="0" style="border-collapse:collapse"><tr><th>Score</th><th>Role</th><th>Company</th><th>Source</th></tr>';
  for (const m of matches) {
    html += '<tr><td>' + m.score + '</td><td><a href="' + esc(m.url) + '">' + esc(m.title) + '</a></td><td>' + esc(m.company) + '</td><td>' + esc(m.source) + '</td></tr>';
  }
  html += '</table>';
  for (const m of matches.slice(0, 3)) {
    html += '<hr><h3>' + esc(m.title) + ' at ' + esc(m.company) + ' (' + m.score + '/100)</h3><p>' + esc(m.reason) + '</p>';
    html += '<p><b>Strengths:</b> ' + esc(m.strengths) + '<br><b>Weaknesses:</b> ' + esc(m.weaknesses) + '<br><b>CV improvements:</b> ' + esc(m.resume_improvements) + '<br><b>Keywords:</b> ' + esc(m.keywords) + '<br><b>Interview tips:</b> ' + esc(m.interview_tips) + '</p>';
    if (m.cover_letter) html += '<p><b>Draft cover letter:</b><br>' + esc(m.cover_letter).replace(/\n/g, '<br>') + '</p>';
  }
}
html += '<hr><p style="color:#777;font-size:12px">Listings come from the public feeds of Remotive, Remote OK, We Work Remotely, Himalayas, Jobicy, Working Nomads and Arbeitnow. Follow each link to apply on the source site.</p>';
return [{ json: { html, subject: 'Daily Job Report: ' + matches.length + ' match' + (matches.length === 1 ? '' : 'es') + ' - ' + date } }];
```
2. Connect **Build Digest** to **Send Email**. In Send Email set Subject to Expression `{{ $json.subject }}` and Message to Expression `{{ $json.html }}`, with Email Type HTML.
3. Run it and check the email: counts, highest match, average, top companies, and the top three analyses.

---

## Version 7: Error handling

1. **Score with Gemini**, Settings tab: **Retry On Fail** on (3 tries, 5000 ms), **On Error** set to **Continue**. Failed jobs are counted and shown as a warning in the report.
2. Create a second workflow named **Job Hunter Error Alert**: add an **Error Trigger** node, then a **Gmail** node (Send) with To your email, Subject Expression `Job Hunter failed: {{ $json.workflow.name }}`, and Message Expression `Node: {{ $json.execution.lastNodeExecuted }}<br>Error: {{ $json.execution.error.message }}`. Save it.
3. Open your main workflow, three dots, **Settings**, and choose **Job Hunter Error Alert** as the Error workflow.
4. Note: error workflows fire only for scheduled or published runs, not manual tests.

---

## Version 8: Dashboard (free)

1. In the Sheet, select the `application_status` column (column R), then Insert, **Dropdown**, with the options New, Applied, Interview, Offer, Rejected.
2. Add a tab named **Dashboard** with these formulas, assuming your data tab is called Sheet1:

| Label | Formula |
|---|---|
| Jobs found | `=COUNTA(Sheet1!F2:F)` |
| Qualified | `=COUNTIF(Sheet1!G2:G,">=60")` |
| Average score | `=ROUND(AVERAGE(Sheet1!G2:G),1)` |
| Highest match | `=MAX(Sheet1!G2:G)` |
| Applied | `=COUNTIF(Sheet1!R2:R,"Applied")` |
| Interviews | `=COUNTIF(Sheet1!R2:R,"Interview")` |
| Offers | `=COUNTIF(Sheet1!R2:R,"Offer")` |
| Success rate | `=IFERROR(Offers/Applied,0)` |

   (Replace the names in the last formula with the cell addresses of Offers and Applied.)
3. Optional: open lookerstudio.google.com (free), Create, Report, connect your Google Sheet, and add scorecards plus a bar chart of jobs by source and a line chart of score by date. Take a screenshot.

---

## Publishing checklist

Before you download any workflow copy:
- In **Settings**, replace the CV with `PASTE YOUR CV TEXT HERE` and the email with a placeholder. Credentials are not exported, but the CV is.
- Cover personal details in screenshots.
- Credit your sources in the README: list Remotive, Remote OK, We Work Remotely, Himalayas, Jobicy, Working Nomads and Arbeitnow with links to each site.

On GitHub (`exports` folder): `v1-basic.json`, `v2-configurable.json` and so on, plus screenshots of the canvas, the email, the Sheet and the dashboard. Update the README to list the versions and what each one added.

**LinkedIn post (edit it so every line is true for you):**
> Project: AI Job Hunter Agent (n8n + Gemini). Built in 8 versions: configurable settings, seven job feeds, duplicate detection against Google Sheets, structured AI analysis (strengths, gaps, CV tips, interview prep), a daily report email, retries and error alerts, and a free dashboard. Stack: n8n, Gemini API, JavaScript, Google Sheets, Gmail. Repo and screenshots on my GitHub.

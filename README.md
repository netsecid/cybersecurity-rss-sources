# Cybersecurity RSS Sources

A curated personal collection of cybersecurity RSS feeds covering news, threat intelligence, vulnerabilities, malware research, vendor blogs, and government advisories.

## Repository Structure

```
cybersecurity-rss-sources/
├── README.md
├── sources/                  # Markdown tables — human-readable, easy to review
│   ├── news.md
│   ├── threat-intel.md
│   ├── vulnerabilities.md
│   ├── malware.md
│   ├── vendor-blogs.md
│   └── government.md
└── feeds/                    # JSON format — for automation, tools, RSS platforms
    ├── news.json
    ├── threat-intel.json
    ├── vulnerabilities.json
    ├── malware.json
    ├── vendor-blogs.json
    ├── government.json
    └── all.json              # Combined master feed list
```

## Categories

| Category | Description | Sources |
|----------|-------------|---------|
| [News](sources/news.md) | General cybersecurity news and reporting | General infosec news outlets |
| [Threat Intelligence](sources/threat-intel.md) | IOCs, threat actor tracking, campaign analysis | TI platforms and research teams |
| [Vulnerabilities](sources/vulnerabilities.md) | CVEs, advisories, exploit disclosures | NVD, vendor advisories, exploit DBs |
| [Malware & Research](sources/malware.md) | Malware analysis, reverse engineering, research blogs | AV/EDR vendor research labs |
| [Vendor Blogs](sources/vendor-blogs.md) | Security blogs from major vendors and companies | Big tech and security vendors |
| [Government & CERT](sources/government.md) | Official advisories and alerts from government bodies | CISA, CERT/CC, national CERTs |

## How to Use

### RSS Reader / Platform
Import individual feed URLs directly into any RSS reader (Feedly, Inoreader, Miniflux, tt-rss, etc.).

### JSON (Automation / Tooling)
Each `feeds/*.json` file follows this schema:

```json
{
  "category": "Category Name",
  "description": "Category description",
  "updated": "YYYY-MM-DD",
  "feeds": [
    {
      "name": "Source Name",
      "url": "https://example.com/feed.xml",
      "description": "What this feed covers",
      "tags": ["tag1", "tag2"],
      "language": "en",
      "active": true
    }
  ]
}
```

Use `feeds/all.json` for a single file that aggregates all categories.

### OPML
To generate an OPML file for bulk import into RSS readers, you can use the JSON feeds with a simple script:

```python
import json, xml.etree.ElementTree as ET

with open("feeds/all.json") as f:
    data = json.load(f)

opml = ET.Element("opml", version="2.0")
body = ET.SubElement(opml, "body")
for category in data["categories"]:
    outline = ET.SubElement(body, "outline", text=category["category"], title=category["category"])
    for feed in category["feeds"]:
        ET.SubElement(outline, "outline", type="rss", text=feed["name"],
                      title=feed["name"], xmlUrl=feed["url"])

ET.ElementTree(opml).write("cybersecurity.opml", xml_declaration=True, encoding="utf-8")
```

## Operationalizing It

Different setups depending on how deep you want to go. Pick the one that fits your team's workflow.

---

### Level 1 — RSS Reader (Simplest)

Just import the feeds into a hosted RSS reader. Good for personal use or small teams who want a single place to browse.

**Recommended tools:** [Feedly](https://feedly.com), [Inoreader](https://inoreader.com), [Miniflux](https://miniflux.app) (self-hosted), [tt-rss](https://tt-rss.org) (self-hosted)

- Use the OPML script (see above) to bulk-import all feeds at once
- Create folders/boards per category (News, TI, Vulns, etc.)
- Use the reader's built-in keyword filtering to surface only what matters
- Good enough for staying aware — not great for team-wide sharing

---

### Level 2 — Slack RSS App (Team-Friendly)

Push specific feeds directly into Slack channels. No code, no infra. The Slack RSS app polls feeds and posts new items automatically.

**Setup:**
1. In Slack, go to **Apps** → search **RSS** → install the RSS app
2. In a channel (e.g. `#sec-advisories`), type `/feed subscribe <rss-url>`
3. Repeat for each feed you want in that channel

**Suggested channel layout:**

| Slack Channel | Feeds to subscribe |
|---------------|--------------------|
| `#sec-news` | Krebs, Bleeping Computer, Dark Reading |
| `#sec-vulns` | NVD, Exploit-DB, ZDI, CISA KEV |
| `#sec-threat-intel` | Talos, Unit 42, SANS ISC, Mandiant |
| `#sec-advisories` | CISA Alerts, NCSC UK, CERT/CC |

**Tip:** Don't subscribe everything — only the highest-signal feeds per channel or it becomes noise nobody reads.

---

### Level 3 — n8n (Keyword-Filtered Delivery)

Fetch all feeds on a schedule, filter by keywords, and only post matching items to Slack. Eliminates noise without manual curation.

**How it works:**
1. **Schedule trigger** — runs every hour (or however often you want)
2. **RSS Read nodes** — one per feed, or loop over `feeds/all.json`
3. **Filter node** — match item title/description against your keyword list
4. **Slack node** — post matches to the right channel based on tags

**Example keyword groups:**

```
ransomware, lockbit, clop, akira
zero-day, 0day, actively exploited, in the wild
critical, RCE, unauthenticated, pre-auth
APT, nation-state, espionage, supply chain
phishing, BEC, credential, MFA bypass
```

**n8n setup tips:**
- Self-host n8n (Docker) or use n8n Cloud
- Store your keyword lists in a JSON/env config so you can update without touching the workflow
- Use the `tags` field from `feeds/all.json` to route items to the right Slack channel automatically
- Add a deduplication step (store seen item GUIDs in a simple DB or Redis) to avoid reposting

---

### Level 4 — Lambda + LLM Summary → Slack (What I Do)

Fetch everything daily, summarize with an LLM, and post a single digest to Slack. Instead of a firehose of individual articles, the team gets one clean briefing per day.

**Architecture:**

```
EventBridge (cron: 0 7 * * *) → Lambda → fetch all feeds → chunk by category
                                        → LLM API (summarize + highlight critical)
                                        → format as Slack Block Kit message
                                        → post to #sec-daily-briefing
```

**Lambda function outline (Python):**

```python
import json, feedparser, boto3
from datetime import datetime, timedelta, timezone
from anthropic import Anthropic

FEEDS = json.load(open("feeds/all.json"))
client = Anthropic()

def fetch_recent_items(feed_url, hours=24):
    d = feedparser.parse(feed_url)
    cutoff = datetime.now(timezone.utc) - timedelta(hours=hours)
    return [
        {"title": e.title, "link": e.link, "summary": getattr(e, "summary", "")}
        for e in d.entries
        if hasattr(e, "published_parsed")
        and datetime(*e.published_parsed[:6], tzinfo=timezone.utc) > cutoff
    ]

def summarize(items, category):
    if not items:
        return None
    content = "\n".join(f"- {i['title']}: {i['link']}" for i in items)
    resp = client.messages.create(
        model="claude-opus-4-6",
        max_tokens=512,
        messages=[{
            "role": "user",
            "content": (
                f"You are a cybersecurity analyst. Summarize these {category} headlines "
                f"from the last 24 hours. Flag anything critical or urgent. Be concise.\n\n"
                f"{content}"
            )
        }]
    )
    return resp.content[0].text

def lambda_handler(event, context):
    blocks = [{"type": "header", "text": {"type": "plain_text",
               "text": f"Security Briefing — {datetime.now().strftime('%b %d, %Y')}"}}]

    for category in FEEDS["categories"]:
        items = []
        for feed in category["feeds"]:
            if feed["active"]:
                items += fetch_recent_items(feed["url"])

        summary = summarize(items, category["category"])
        if summary:
            blocks += [
                {"type": "section", "text": {"type": "mrkdwn",
                 "text": f"*{category['category']}*\n{summary}"}},
                {"type": "divider"}
            ]

    boto3.client("lambda").invoke(   # or call Slack webhook directly
        FunctionName="slack-poster",
        Payload=json.dumps({"channel": "#sec-daily-briefing", "blocks": blocks})
    )
```

**What makes this work well:**
- Run once a day (morning) so the team starts the day informed, not interrupted
- Prompt the LLM to flag `CRITICAL` items explicitly — those get a separate ping
- Keep the Lambda under 5 min by capping items per feed (last 24h only)
- Store the raw items in S3 if you ever want to go back and read the originals
- Layer in your own keyword boosting in the prompt ("pay extra attention to: ransomware, RCE, our tech stack keywords")

---

## Tags Reference

| Tag | Meaning |
|-----|---------|
| `news` | General security news |
| `threat-intel` | Threat intelligence and IOCs |
| `cve` | CVE and vulnerability tracking |
| `exploit` | Exploit code and PoC releases |
| `malware` | Malware analysis and samples |
| `apt` | APT / nation-state threat actors |
| `ransomware` | Ransomware tracking |
| `research` | Academic or deep-dive research |
| `advisory` | Official security advisories |
| `vendor` | Security vendor blogs |
| `government` | Government / CERT sources |
| `podcast` | Audio / podcast RSS |

## Contributing

1. Add the source to the appropriate `sources/*.md` table
2. Add the same entry to the corresponding `feeds/*.json`
3. Update `feeds/all.json` with the same entry under the correct category
4. Keep `active: false` for feeds that are no longer maintained

---

*Personal collection — last reviewed: 2026-04-03*

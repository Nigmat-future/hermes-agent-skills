---
name: advisor-insight
description: "导师锐评 · PI Profile Analyzer: objectively evaluate academic advisors by mining publications, grants, co-author networks, and career trajectory. Truthful, comprehensive, bias-free."
version: 1.0.0
author: Nigmat Rahim
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [导师锐评, advisor-evaluation, pi-profile, academic-assessment, phd-advising]
    related_skills: [arxiv, huggingface-hub, youtube-content]
---

# Advisor Insight · 导师锐评

## Overview

Choosing a PhD advisor is one of the most consequential decisions in an academic career. 
This skill systematically evaluates a potential advisor's **real research profile** — 
stripping away institutional branding, inflated titles, and team-based prestige to reveal 
the individual's actual independent contributions.

**Core principle:** Data over reputation. Evidence over rhetoric. Comprehensive over selective.

**Target audience:** Students choosing PhD/postdoc advisors. The skill helps answer:
- Is this person truly an independent PI, or a platform-dependent contributor?
- Do they have a real, coherent research program, or do they chase trends?
- What is their actual publication independence ratio vs. team co-authorship?
- How competitive would I be joining their group vs. their track record?

---

## Phase 0: Disambiguation — Get the Right Person

**CRITICAL: Never assume a name+institution pair is unique. Chinese names in particular are extremely ambiguous.**

### Step 0.1 — Ask clarifying questions first

Before any search, ask the user these questions:

1. **Full Chinese name + any English name(s) / aliases** (e.g., "Zhang Weilong", "Weilong Zhang", "王磊", "Wang Lei")
2. **Current institution + department** (e.g., "北京大学第三医院血液科")
3. **Title / rank** (if known: "主治医师", "教授", "研究员", "副教授")
4. **ORCID if available** — this is the gold standard for disambiguation
5. **Any known collaborators or students** — helps confirm identity
6. **Research area keywords** — e.g., "mantle cell lymphoma", "single-cell"
7. **Approximate career stage** — how many years since PhD?

If the user provides only a name, **do not proceed** until at least institution + department are clarified.

### Step 0.2 — Cross-reference multiple identifiers

After getting the user's answers, verify the person exists across multiple sources:

- **PubMed**: Search `"Full Name" AND "Institution"` with known variations
- **ORCID**: If provided, this is definitive
- **Google Scholar**: Check profile has the claimed institution
- **Institutional website**: Faculty/staff directory
- **ResearchGate / Loop**: Additional profile verification

**Cross-check rule:** If two sources disagree on the person's affiliation or identity, STOP and ask the user for clarification before proceeding.

---

## Phase 1: Comprehensive Publication Audit

### 1.1 — PubMed Harvest

Query PubMed via NCBI E-utilities (free, no API key needed):

```bash
# Get PMID list
curl -s "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi?db=pubmed&term=%22Zhang+Weilong%22+AND+%22Peking+University+Third+Hospital%22&retmax=200&retmode=json"

# Get paper details (use comma-separated PMIDs)
curl -s "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esummary.fcgi?db=pubmed&id=PMID1,PMID2,...&retmode=json"
```

**Collect for each paper:**
- Journal, year, title
- Author list with position
- Grant/funding acknowledgments
- PMID for cross-referencing

### 1.2 — Role Classification

Classify EVERY paper into one of:

| Role | Definition | Weight |
|------|-----------|--------|
| **First Author** | Position 1 in author list | High independence |
| **Co-first Author** | Marked with equal-contribution symbol | Moderate independence |
| **Last/Corresponding Author** | Last position + corresponding marker | PI-level (high) |
| **Co-corresponding** | Marked as corresponding but not last | Moderate PI |
| **Middle Author** (2 to N-1) | Team contributor | Low independence |
| **Distant Author** (N-2+ or 9+) | Minimal contribution likely | Minimal independence |
| **Large Consortium** (>20 authors) | Contribution unclear, likely minor | Context-dependent |

**Calculate Independence Ratio:**
```
Independence Score = (FirstAuthor + CoFirst*0.7 + LastCorr*1.0 + CoCorr*0.5) / TotalPapers
```

**Thresholds:**
- `>0.30`: Credible independent researcher
- `0.15-0.30`: Developing independence
- `<0.15`: Primarily team-dependent; likely not an independent PI

### 1.3 — Journal Quality Distribution

Map journals to tiers:

| Tier | IF Range | Examples |
|------|----------|---------|
| **T1 Elite** | IF > 15 | Nature, Science, Cell, NEJM, Lancet, Nat Biotech, Nat Med, Cancer Cell |
| **T2 Top** | IF 8-15 | Nat Commun, Sci Adv, PNAS, Genome Biol, Leukemia, J Clin Oncol |
| **T3 Good** | IF 4-8 | Brief Bioinform, Bioinformatics, eLife, PLoS Comput Biol |
| **T4 Mid** | IF 2-4 | Int J Mol Sci, J Transl Med, Ann Hematol, Blood Res |
| **T5 Low** | IF < 2 or on Beall's/predatory lists | J Cancer, Oncol Lett, Cancer Manag Res, Int J Clin Exp Pathol |

**IMPORTANT:** For multi-author papers in T1/T2, note each paper's author position. A distant author (position 20+/30) on a CNS paper does NOT indicate PI-caliber work — it indicates sample contribution.

### 1.4 — Citation Analysis

If Google Scholar is accessible:
- **Total citations**
- **h-index** (and whether self-citations inflate it)
- **i10-index**
- **Citation trajectory** — accelerating, plateauing, or declining?

If Google Scholar is not directly accessible, infer from:
- Average IF of journals where they publish
- Pattern of first/corresponding author papers (these get cited more)
- Co-author network strength

---

## Phase 2: Grant/Funding Analysis

### 2.1 — Extract from PubMed Acknowledgments

Parse `<Grant>` elements from PubMed XML for each paper:

```python
# Parse grant IDs and agencies from PubMed XML
for grant in article.findall('.//Grant'):
    grant_id = grant.find('GrantID')
    agency = grant.find('Agency')
    # Match grant IDs to known NSFC types:
    # 8xxxxxx = Youth Project (~21万 CNY, entry-level)
    # 3xxxxxx = General/General Program (~55万, independent PI)
    # 2xxxxxx = Excellent Young Scientists Fund (~200万, elite)
    # 7xxxxxx = Key Program (~300万, senior PI)
```

### 2.2 — Grant Cross-Reference

**Determine who OWNS each grant:**
- Check other papers that acknowledge the same grant ID
- Who is the corresponding/PI author on those papers?
- A grant acknowledged on someone else's papers → not their grant
- A grant appearing only on papers where they are the last author → likely theirs

**Key indicators of PI status:**
- Owns a NSFC General Program (面上项目, 3xxxxxx) — **minimum threshold for independent PI**
- Owns an Excellent Young Scientists Fund (优青, 2xxxxxx) — competitive
- Owns a Key Program (重点项目, 7xxxxxx) — established senior PI

**Red flags:**
- Only Youth Project (青年项目) + institutional internal grants — may be a "permanent attending" (万年主治)
- All grants are acknowledged on papers where another person is last author — person is a team member, not PI
- No grants at all found — low research funding activity

---

## Phase 3: Co-author Network & Academic Food Chain

### 3.1 — Identify the Power Structure

Count co-occurrence frequency for all co-authors:

```python
from collections import Counter
coauthors = Counter()
for paper in papers:
    for author in paper.authors:
        if author.name != target_name:
            coauthors[author.name] += 1
```

**Top co-authors reveal the power structure:**
- **The Boss** — appears in 70%+ of papers, typically last author on most papers. This is the actual PI.
- **Peers** — appear frequently, similar author positions. Same career stage.
- **Junior/Students** — appear in fewer papers, earlier positions (1st-3rd author). Their trainees.
- **External Collaborators** — appear on specific sub-topics only.

### 3.2 — Map the Food Chain

For each paper, record the target's position relative to the last author:

```
Journal: Leukemia (2024)
Last author: Jing H (THE BOSS)
Positions: Yang P[5], Zhang W[7], Wu C[12], He X[14]
→ Zhang is junior to Yang, senior to Wu and He
→ Pattern: consistent mid-pack position
```

**Pattern recognition:**
- Always 5-8 of 10-15 in Boss's papers → senior team member, not PI
- Sometimes last author → developing independence
- Always last author on their own small papers → this IS their independent work
- Never last author → zero PI experience

### 3.3 — External Collaboration Network

Identify non-local co-authors:
- Are there collaborations with other institutions/departments?
- Are they lead or contributor in these collaborations?
- Do external collaborations center on the Boss's reputation or the target's?

---

## Phase 4: Career Trajectory Reconstruction

### 4.1 — Timeline from Publications

Reconstruct career from publication history:

| Period | Publication Pattern | Career Inference |
|--------|-------------------|-----------------|
| No publications | Pre-PhD / early career | ... |
| Middle-author, low-tier | PhD / early postdoc | Learning phase |
| First middle-author, then last-author | Postdoc → early PI | Developing independence |
| Consistent last-author + first-author | Established PI | Independent |
| Flat middle-author throughout | "Permanent team member" | Not progressing to independence |

### 4.2 — Title/Position Evidence

Cross-reference with:
- Hospital/university website staff page
- News articles mentioning their team
- Conference speaker listings
- Editorial board memberships
- PubMed affiliation field over time

### 4.3 — Independence Trajectory

**Key question:** Is the person moving toward or away from independence?

- Authorship position improving over time? → Positive trajectory
- Publishing without the Boss? → Real independence
- Still publishing primarily with same Boss after 8+ years? → Career plateau
- Grants shifting from Youth to General? → Promotion trajectory

---

## Phase 5: Synthesis & Scoring

### 5.1 — Standard Scoring Framework

Score each dimension 0-10 with clear evidence:

| Dimension | Description | Weight |
|-----------|-------------|--------|
| **Research Independence** | First/corresponding author ratio, own grants, own students | 25% |
| **Research Depth** | Coherent research program vs. topical scattering, depth vs. breadth | 20% |
| **Funding Competitiveness** | Grant portfolio quality (NSFC type, total amount) | 15% |
| **Publication Quality** | Journal tier distribution, not just count | 15% |
| **Career Trajectory** | Trending up, plateaued, or declining | 10% |
| **Mentorship Capacity** | Evidence of training students/postdocs (co-authored with juniors) | 10% |
| **Collaboration Breadth** | External partners, interdisciplinary work | 5% |

### 5.2 — Scoring Guidelines

| Score | Meaning |
|-------|---------|
| 9-10 | Elite. Exceptional track record, clear PI identity |
| 7-8 | Strong. Good independence, well-funded, coherent program |
| 5-6 | Adequate. Developing, moderate independence, some grants |
| 3-4 | Weak. Heavy team-dependence, limited funding, no clear direction |
| 1-2 | Concerning. Near-zero independence, likely misleading claims |
| 0 | Unable to evaluate (too early career or insufficient data) |

### 5.3 — Final Report Structure

```
# Advisor Insight Report: [Name]

## 1. Identity Confirmation
   - Cross-referenced identifiers used
   - Confidence level: HIGH / MEDIUM / LOW

## 2. Publication Profile
   - Total papers: N
   - First author: N (X%) | Last/corr author: N (X%) | Middle author: N
   - Independence Ratio: X.XX
   - Journal tier distribution (T1-T5 counts)
   - Citation metrics (if available)

## 3. Funding Analysis
   - Owned grants (with types and amounts)
   - Shared grants (not theirs but acknowledged)
   - Estimated total research budget

## 4. Power Structure
   - Identified boss/mentor: [Name]
   - Position in group: [Role]
   - Evidence of own group: Yes/No/Developing

## 5. Career Status
   - Estimated/confirmed title: [Title]
   - Independence trajectory: [Improving/Stable/Plateaued/Declining]
   - Years since first publication: [N]

## 6. Scoring
   - Research Independence: X/10
   - Research Depth: X/10
   - Funding Competitiveness: X/10
   - Publication Quality: X/10
   - Career Trajectory: X/10
   - Mentorship Capacity: X/10
   - Collaboration Breadth: X/10
   - **OVERALL: X.X/10**

## 7. Bottom Line
   - One-paragraph summary of who this person really is as a researcher
   - What students should know before joining their group
   - What to ask in an interview
```

---

## Truth & Bias Guardrails

### What this skill DOES:
- Uses **only publicly verifiable data** (PubMed, grants databases, institutional sites)
- States **confidence levels** (don't claim what you can't verify)
- **Distinguishes correlation from causation** in authorship position
- **Reports conflicting evidence** when sources disagree
- **Prefaces every judgment** with the data supporting it

### What this skill DOES NOT do:
- **No personality judgments** — never comment on teaching ability, mentoring style, or personal character without direct evidence
- **No rumor or hearsay** — only published, citable evidence
- **No institutional prestige bias** — a person at a famous university with weak metrics gets the same analysis
- **No field bias** — adjust expectations for field norms (e.g., computer science has different authorship norms than clinical medicine)
- **No "potential" claims** — evaluate what EXISTS, not what might exist
- **Never assumes malice** — low productivity doesn't mean lazy; high co-authorship doesn't mean freeloading
- **Never compares to absolute standards** — compare to field-appropriate and career-stage-appropriate benchmarks

### Disambiguation Guard (Always On):
If after Phase 0 you discover **MULTIPLE people share this name** at different institutions, and you cannot confirm with certainty which one is the target, STOP and report the ambiguity. Present the evidence for each candidate and ask the user to confirm.

### Disclosure:
All data is from publicly available sources. This analysis is an evidence-based assessment of research metrics, not a personal evaluation. The subject has not been contacted or notified.

---

## Tool Integration

### Required tools:
- `terminal` — for PubMed E-utilities API calls (curl), data parsing (Python)
- `browser_navigate` / `browser_snapshot` — for institutional websites, Google Scholar (if accessible)
- `read_file` / `search_files` — for saving and analyzing intermediate data
- `delegate_task` — for parallel sub-analyses (grants, co-author network, publications)

### Python parsing scripts (inline):

```python
# Parse PubMed ESummary JSON
import json
data = json.loads(pubmed_response)
for pmid, info in data['result'].items():
    if pmid == 'uids': continue
    title = info.get('title', '')
    authors = [a['name'] for a in info.get('authors', [])]
    journal = info.get('source', '')
    pubdate = info.get('pubdate', '')
    # Find target position
    target_pos = next((i+1 for i, a in enumerate(authors) 
                       if target_name.lower() in a.lower()), None)
```

### Suggestion for delegate_task:

```python
delegate_task(
    goal="Harvest all PubMed publications and analyze author positions for [Person Name]",
    context=f"""
    Target: {name}, Institution: {institution}
    PubMed query: {pubmed_query}
    Extract: total papers, first/last/middle author counts, co-author frequencies, grant IDs
    Return: structured JSON with all findings
    """,
    toolsets=['terminal', 'file']
)
```

---

## Common Pitfalls

1. **Same name, different person** — ALWAYS disambiguate first. Chinese names (张伟, 王磊, 刘洋) routinely have 50+ published people with the same name.
2. **"Team" vs. "Independent PI"** — institutional news articles routinely call any group member a "team lead." Verify from author positions, not press releases.
3. **ORCID is empty** — ORCIDs with no content are a yellow flag, not a red flag. Many established PIs don't maintain them.
4. **Google Scholar unreliable** — May include non-peer-reviewed preprints, auto-merged profiles, incorrect citation counts. Use PubMed as ground truth.
5. **Grant attribution** — A grant ID appearing in the acknowledgments doesn't mean the person holds the grant. Cross-reference against other papers using the same grant.
6. **Early-career PIs** — A new PI (<3 years) will naturally have low independence metrics. Adjust scoring: weight trajectory over current metrics.
7. **Field-specific norms** — Clinical researchers publish differently (more middle-author, more team papers) than basic scientists. Adjust expectations.
8. **Large consortium papers** — Many-author papers (50+ authors) should be de-weighted in independence analysis. They reflect consortium access, not individual contribution.

## Example Applications

Real evaluation sessions exist in the session archive demonstrating the full investigative workflow — from disambiguation through synthesis — applied to Chinese clinical researchers at various career stages.

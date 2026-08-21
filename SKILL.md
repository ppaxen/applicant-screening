---
name: applicant-screening
description: Screens startup/program applicants stored in a Notion database and produces a first-pass review database for a selection committee. Use this whenever the user mentions screening applicants, 지원자 서류 심사, 1차 평가, accelerator/program applications sitting in Notion, classifying startups by industry or business model, or reducing repetitive work in reviewing many applications (e.g. "노션에 있는 지원자들 분류해줘", "지원서 90개 정리해줘", "1차 심사 자료 만들어줘"). Trigger even if the user doesn't say "skill" or "screening" explicitly — any request to read a Notion applicant DB and organize/classify entries for a review process qualifies.
---

# Applicant Screening

## What this skill does, and does not do

This skill turns a pile of raw applications into an organized, comparable first-pass
document for a human committee. It does the repetitive reading and summarizing so
people don't have to re-read the same IR deck five times to extract the same facts.

**It classifies and summarizes. It does not decide.** Never write a final Pass /
Neutral / Non-Pass (P / N / NP) verdict into any property meant for the committee's
judgment — leave those blank. The reasoning behind this split: when a group makes a
decision together, forcing early convergence on one "answer" throws away the
disagreement that's actually useful to discuss. This skill's job is to make sure
everyone is arguing from the same well-organized facts, not to pre-argue the case
for them.

## Step 1: Identify source, destination, and where the real content lives

Ask the user (if not already clear from context):
1. The URL or name of the source Notion database containing applicants.
2. Where the new results database should live (which Notion page as parent). If
   they don't have a preference, propose creating it as a sibling page next to the
   source database.

Use `fetch` on the source database URL to get its schema and data source ID
(`collection://...`). Check what the substantive business description actually
lives in before assuming it's readable:

- If it's a rich_text/text property with real content, you can read it straight
  from `query_data_sources`.
- If the real description is inside the page body, `fetch` each applicant page
  individually.
- **If it's a file attachment (IR deck, application form as PDF/PPTX/etc.)**,
  check whether it's actually reachable:
  - Native Notion attachments (`attachment:...` references) generally cannot be
    downloaded through the connected Notion tools — there's no general
    file-content-extraction path for binary files uploaded directly to Notion.
  - Attachments hosted on an external service (e.g. a form tool's storage URL)
    may resolve to a real URL, but check the actual file format before trusting
    it — a `.pages`/`.docx`/`.zip`-like binary won't yield useful text through a
    generic web fetch, even if the URL itself loads fine.
  - If neither path gives real text, tell the user directly rather than guessing
    or producing a summary from a one-line title alone. Ask them to export/convert
    the source files to PDF and place them in a local folder — PDFs can be read
    directly. Agree on a filename convention that lets you match each file back
    to its Notion row unambiguously — the most reliable key is a unique property
    already in the database (e.g. an applicant number/ID), used as a filename
    prefix like `07_팀명.pdf`. Confirm the convention with the user before they
    start exporting so you don't end up with unmatchable files.

## Step 2: Use the existing taxonomy if there is one

Before inventing categories, check whether the source database (or the
organization's past practice) already has an industry/category taxonomy in use —
e.g. existing select/multi-select properties with real options already populated.
If so, classify against that existing taxonomy rather than inventing a new one:
consistency with past cohorts/batches matters more than a locally "cleaner"
scheme, and reinventing categories each run makes results incomparable over time.

Only draft a new taxonomy from scratch if nothing existing fits reasonably:

1. Query all applicant rows via `query_data_sources` (SQL mode) to get an overview
   of the full batch first.
2. Skim the business descriptions across the whole batch and draft a working set of
   industry categories (산업분야) and business model types (BM유형) that fits what's
   actually in this batch — don't reuse a generic canned list from a different
   domain. Keep the taxonomy small enough to be useful for grouping (roughly 5-12
   industry categories, 3-6 BM types is typical, but let the actual data decide).
3. Classify every applicant against that same working taxonomy. It's fine to add a
   category mid-way if a genuinely new one appears, but don't fragment a single
   concept into near-duplicate labels.

There is usually no existing taxonomy for BM유형 (business model type) even when an
industry taxonomy already exists — that one you'll typically still need to draft
per Step 2's method above.

## Step 3: Create the results database

Create it under the destination page with `create_database`, using this schema
(adjust Korean labels if the user prefers different wording, but keep the split
between AI-filled fields and committee-filled fields):

```
CREATE TABLE (
  "지원자/팀명" TITLE,
  "산업분야" SELECT,
  "BM유형" SELECT,
  "핵심요약" RICH_TEXT COMMENT 'AI가 작성. 아이템, 문제, 솔루션, 팀을 3-5문장으로 요약',
  "원본 링크" URL,
  "①시장성_근거요약" RICH_TEXT COMMENT '아래 기준 참고',
  "①시장성 판정" SELECT('NP':red, 'N':yellow, 'P':green) COMMENT '위원이 직접 입력 - AI는 채우지 않음',
  "②솔루션타당성_근거요약" RICH_TEXT,
  "②솔루션타당성 판정" SELECT('NP':red, 'N':yellow, 'P':green) COMMENT '위원이 직접 입력',
  "③팀전문성_근거요약" RICH_TEXT,
  "③팀전문성 판정" SELECT('NP':red, 'N':yellow, 'P':green) COMMENT '위원이 직접 입력',
  "④검증성장_근거요약" RICH_TEXT,
  "④검증성장 판정" SELECT('NP':red, 'N':yellow, 'P':green) COMMENT '위원이 직접 입력',
  "후속 검증 항목 (AI 초안)" RICH_TEXT COMMENT 'AI가 초안 작성, 위원이 검토/수정'
)
```

Leave the four "판정" (verdict) properties and the human review process entirely to
the committee — don't set a value for them when creating pages.

## Step 4: Write the evidence summary for each of the four criteria

This is the core of the skill. For each applicant, write a short, evidence-grounded
summary per criterion — not a verdict, a summary of what's actually observable in
the application. Use these specific lenses (from the committee's own rubric):

**① 시장성 및 문제의 중요도 (market & problem significance)**
Check whether the problem/market claims in the IR deck or application check out
factually where verifiable (numbers, cited sources, claims that seem inflated or
unsupported), then summarize how severe the problem actually seems and how large
the addressable market actually looks — not just what the applicant claims about
severity/size, but your own read given the evidence presented.

**② 솔루션 및 사업모델의 타당성 (solution & business model validity)**
Assess whether the proposed solution logically follows from the problem described
in ① — does it actually address the stated problem, or is there a gap between the
problem framing and what's being built/sold? Note any logical disconnect explicitly.

**③ 팀의 전문성 및 실행력 (team expertise & execution capacity)**
If the industry plausibly requires deep technical expertise (deep tech, biotech,
hardware, advanced ML, etc.), check whether the team actually includes people with
that specific expertise — don't just note "strong team," identify whether the
specific expertise required is present or missing. Also convert team stability into
something checkable where the application gives enough signal: e.g., how many
members are full-time vs. part-time, any indication of recent team turnover or
key members who joined very recently / are moonlighting elsewhere. State these as
concrete observations ("3 of 4 listed as full-time, 1 as part-time advisor") rather
than vague impressions, and say when the application simply doesn't give enough
information to tell.

**④ 검증 수준과 성장 가능성 (validation level & growth potential)**
Look for concrete evidence of progress — PoC results, pilot customers, revenue,
user numbers, LOIs — rather than aspirational statements. Separately note growth
signals: how fast metrics (users, revenue, waitlist, etc.) appear to be moving where
the application provides numbers over time.

For every criterion, if the application doesn't contain enough information to say
anything concrete, say so plainly in the summary rather than padding it with
generic language — that gap itself is useful signal, and gaps are exactly what
should also surface in the "후속 검증 항목 (AI 초안)" field so the committee can
raise them in the follow-up interview/coffee chat stage.

## Step 5: Populate the results database

If working from local PDFs, read each one with the file-reading tool (which
supports PDFs directly) and match it to its Notion row using the filename
convention agreed in Step 1 — don't guess a match from team name similarity alone
if the numbering/ID prefix is available; it's the unambiguous key.

Use `create_pages` with the destination data source as parent, batching multiple
applicants per call (up to 100 pages per call) rather than one call per applicant.
Set the "원본 링크" property to the source page's URL so committee members can jump
back to the full original application.

Applicant PDFs contain personal/business-sensitive information — keep them in a
local folder outside of anything that gets pushed to a public repo (e.g. add the
folder to `.gitignore` if this skill lives in a git repo). The skill itself
contains no applicant data and is safe to share.

## Step 6: Report back

Tell the user how many applicants were processed, link to the new database, and
flag any applicants that were skipped or had unusually thin information (these are
worth a manual look before the committee meeting).

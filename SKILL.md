---
name: skeptical-ga4-admin-api-automation
description: "Script GA4 channel groupings, audiences, annotations, and enhanced measurement via the Admin API. Use when rolling consistent channel groupings, audiences or annotations across many GA4 properties from the terminal, or when the GA4 UI is too slow or inconsistent for fleet changes."
when-to-use: "Rolling consistent channel groupings, audiences or annotations across many GA4 properties from the terminal (service account plus Python requests), or the GA4 UI is too slow or inconsistent for fleet changes."
---

# GA4 Admin API automation

## What this does (plain English)

You can configure Google Analytics 4 from a script through the GA4 Admin API, but the API has many undocumented quirks that cost hours: some resources only exist in v1alpha, field names must use a specific prefix, one filter type works in the API but shows up blank in the UI, and there is a hard limit on custom channel groupings. This skill gives your AI assistant the quirks verified live while managing a fleet of properties: custom channel groupings, audiences, annotations and enhanced measurement toggles, plus idempotent deploy habits (skip what exists, back off on errors, verify after writing). It is written for people who manage more than a couple of GA4 properties, such as agencies.

## Responsible use

Only change GA4 accounts and properties you own or are authorized to administer. These changes apply to live properties, and most affect future data only. Test on one property first, re-audit the current state before any fleet write, and warn report consumers before flipping anything that changes event volume.

## Install what you need

You need a Google Cloud service account with the GA4 Admin API enabled, granted Editor on each GA4 account, and Python with the `requests` library (as the recipe describes).

```
python3 -m pip install requests
```

## Worked example

Roll out one custom channel grouping to one property, using only the rules from the recipe below.

1. Find the property. List properties for an account (replace `<id>` with the GA4 account ID; the path `accounts/<id>/properties` does not exist):

```
GET https://analyticsadmin.googleapis.com/v1beta/properties?filter=parent:accounts/<id>
```

2. Use the v1alpha `properties.customChannelGroups` resource (v1beta returns an HTML 404).
3. Write fields as `eachScopeSource`, `eachScopeMedium`, `eachScopeCampaignName` or `eachScopeDefaultChannelGroup`. `session*` names are rejected.
4. Shape the expression as CNF (top-level `andGroup`, then `orGroups`, then filters). If your spec is an OR of AND groups, distribute it yourself first. Stay at 50 rules or fewer.
5. For lists of values, use `FULL_REGEXP` with anchored alternation, never `inListFilter` (it shows blank in the GA4 admin UI).
6. Order rules by priority: campaign-name-specific rules first, token-based rules next, and the catch-all last: `eachScopeSource FULL_REGEXP .*`.
7. Check you are within the limit of 2 custom channel groupings per property (the 3rd create returns 429).
8. Read back and verify. Remember there is no retroactive regrouping: only future sessions use the new grouping.

---

The full production recipe follows.


## When to Use
- Rolling consistent channel groupings/audiences/annotations across many GA4
  properties from the terminal (service account + Python requests)
- The GA4 UI is too slow or inconsistent for fleet changes

## Setup
- GCP service account with the GA4 Admin API enabled; grant Editor on each GA4
  account (property access alone is not enough).
- List properties: `GET https://analyticsadmin.googleapis.com/v1beta/properties?filter=parent:accounts/<id>`
  (the path accounts/<id>/properties does NOT exist - HTML 404).

## Custom channel groupings (the quirks are the skill)
- Resource: v1alpha properties.customChannelGroups - v1beta does NOT exist
  (HTML 404).
- Field names must be eachScope*: eachScopeSource, eachScopeMedium,
  eachScopeCampaignName, eachScopeDefaultChannelGroup. session* names are
  rejected with unsupported-channel-grouping-field.
- Expression shape is CNF: top-level andGroup -> orGroups -> filters. If your
  spec is DNF (OR of AND groups), distribute it yourself before sending.
  Max 50 rules per grouping.
- String filters are case-insensitive natively; strip any (?i) prefixes; match
  types: EXACT / PARTIAL_REGEXP / FULL_REGEXP (RE2).
- NEVER use inListFilter: it works at the API and reporting level but the GA4
  admin UI renders those conditions BLANK (looks like data loss to every human
  reviewer). Use FULL_REGEXP with anchored alternation instead - full-regex is
  whole-string, so A|B behaves like an exact-match list.
- HARD LIMIT: 2 custom channel groupings per property. The 3rd create returns
  429 limit-exceeded.
- groupingRule order = match priority; first match wins; channels are mutually
  exclusive. Campaign-name-specific rules must precede token-based rules.
  Put the catch-all last: `eachScopeSource FULL_REGEXP .*`
- NO retroactive regrouping: only future sessions use the new grouping.
- Validate rules against LIVE data first: pull source_medium x campaign
  breakdowns (Data API or your reporting tool), 2-3 properties is enough to
  catch naming variance before fleet deploy.
- One grouping can be set primary (updates the property's default reporting
  grouping); the system Default grouping stays untouched either way.

## Audiences (v1alpha properties.audiences)
- filterClauses are IMMUTABLE: to fix one, archive (:archive) and rebuild.
  Plain DELETE leaves them around.
- A simple filter still requires scope (AUDIENCE_FILTER_SCOPE_ACROSS_ALL_SESSIONS)
  and the same CNF quirk: andGroup elements must be orGroup wrappers.
- Dimension field names DIFFER from the reporting API: pageLocation works;
  pagePath / page_location / city are rejected; country, pageTitle,
  deviceCategory, sessionSource are fine.
- Description limit ~100 chars (a 123-char description was rejected).
- Make deploys idempotent: skip audiences that already exist by displayName.
  Match the property's existing naming conventions ("Prefix - Name" vs "Name")
  or skip-by-name won't dedupe and near-duplicates pile up.
- Publishing audiences to Google Ads is manual (Admin > Audiences > Publish)
  and requires a Google Ads link.

## Annotations (v1alpha properties.reportingDataAnnotations)
- color is an ENUM and UPPERCASE only: RED, BROWN, BLUE, GREEN, CYAN, PURPLE
  (ORANGE is reserved for system annotations; lowercase rejected).
- Description limit 150 chars. Dedupe by title + start date; plain DELETE works.
- Use for algo updates, incidents, launches, config changes - future-you
  reading a traffic graph will care.

## Enhanced measurement toggles (per stream)
- Resource: `v1alpha properties/{p}/dataStreams/{ds}/enhancedMeasurementSettings`
  (GET/PATCH with `?updateMask=<field>`). v1beta/v1 404 on this subresource, and
  the dataStream object itself does NOT carry EM settings.
- Fields: streamEnabled, pageChangesEnabled, formInteractionsEnabled,
  scrollsEnabled, outboundClicksEnabled, siteSearchEnabled,
  videoEngagementEnabled, fileDownloadsEnabled, searchQueryParameter,
  uriQueryParameter.
- FALSE FIELDS ARE OMITTED from GET responses (proto3 defaults): absent = off,
  not unknown.
- Common production config when GTM owns tagging: formInteractionsEnabled=false
  (EM auto form_submit double-counts and fires on Next clicks/failed submits)
  and pageChangesEnabled=false (the googtag config already sends page_view;
  EM history-change page views are redundant on SPA-less sites).
- Detection WITHOUT the API: the compiled `gtag/js?id=G-XXXX` bundle is compiled
  PER STREAM and instantiates {"function":"__ccd_em_form" etc. only for enabled
  modules - grep the instantiated pattern, not bare module names (definitions
  are always bundled). Edge cache lags ~15-30 min after a toggle; the API GET
  is immediately ground truth.
- Toggles affect FUTURE events only; warn report consumers before flipping
  anything that changes event volume (a form_submit key event can roughly halve).

## Fleet deploy hygiene
- Make every operation idempotent: PATCH-if-exists for groupings,
  skip-by-displayName for audiences, dedupe annotations by title+date.
- Backoff on 429/5xx (20s+); throttle between properties; save an incremental
  report; finish with a read-back verify pass per property.
- Re-audit current state before any fleet write; don't guess from last month's
  inventory.

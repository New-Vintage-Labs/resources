# Holiday Campaign Planner — live data upgrade

- **Version:** 0.5.1
- **Upgrades:** planners built from [holiday_planner_SPEC.md](holiday_planner_SPEC.md) (v0.5)
- **Add-on:** [vip_gifters_ADD-ON.md](vip_gifters_ADD-ON.md) (v0.1.1), after this upgrade

*For the winery: when you're ready to make your planner's numbers live, turn on the New Vintage connector in Claude, then upload this file in the chat where you built your planner and say "Connect my planner to New Vintage."*

You are upgrading a Holiday Campaign Planner that was built from the build spec ([holiday_planner_SPEC.md](holiday_planner_SPEC.md)). You'll turn it into a live planner whose numbers come from the winery's Commerce7, email and SMS data through the New Vintage connector. The page, its layout and the team's activities stay as they are. Work through the **Steps** in order. Each step ends on a **Done when** line; finish it before starting the next.

All data access is read-only:
- Run SELECT queries through `safe_tenant_sql`.
- Leave every write tool uncalled.
- Keep customer names, emails and phones out of the chat and out of the page. Every query below returns aggregates only.
- **Call the connector one query at a time**, from chat, from subagents and from the page. Parallel calls make it stop responding. When a response carries a `conversation_id`, send it with the next call.

**When the user moves on without confirming.** If the user skips a confirmation, don't ask again. Use what you proposed, mark each unconfirmed item **"Assumed — not confirmed,"** and list those items in the U7 handover. The one exception is the tenant: without it, stop.

---

## Steps

### U1. Check the connection

1. Run `discover_tenant_data_sources`. If it reaches several tenants, ask which winery this is, and record its exact `tenantId`.
2. Record Commerce7's last sync, and which email platforms (Klaviyo, Mailchimp) and SMS platforms (RedChirp) are synced.
3. Run **Q2**, then **Q3** for last season, to see which platforms sent campaigns and from what date each platform's recipient history is complete. Measure only complete campaigns. An SMS platform with linked texts but no clicks has a click tracking gap: read it as missing data, never as 0% conversion.
4. Show the user this as a four-line summary.

If the connector is missing, or the account lacks Pro access, tell the user to turn on the New Vintage connector in Claude's settings, then stop.

**Done when:** the user has seen the summary and you know the one tenant.

### U2. Start from the latest plan

Start from the latest whole saved plan when the artifact exposes it. If the user has edited the page since your last change and you cannot read that saved plan, ask them to use More → Download CSV and attach the file. The CSV contains activity rows only: merge them into the last whole plan rather than treating the CSV as a whole-plan replacement. Preserve the existing activity ids and fields absent from the CSV; match rows by their existing title, lane and start week, and confirm ambiguous matches, added rows or removed rows before changing them.

Confirm any page edits not carried by the CSV: winery, revenue goal, actual revenue/booked-through week, last-year figures, lane defaults, overlap discount, key dates and season dates. Keep those values from the latest saved plan or the user's confirmed corrections. If any are unknown, stop before replacing the saved plan; do not silently reset them to an example or older chat copy. Otherwise use the latest whole plan you already have.

**Done when:** you have the current plan, including every typed number.

### U3. Measure last season

With last season's dates:
1. Run **Q1** for last year's total and weekly figures.
2. Run **Q5** on last season and group its campaigns into offers (see **Grouping sends into offers**).
3. Run **Q6** with the offers built from Q3's complete campaigns, for the Email and SMS lane defaults: conversion = matched orders ÷ person-offers, with two decimals; average order = matched revenue ÷ matched orders.
4. Run **Q7** on last season for the Club shipment default: bill rate = billed ÷ scheduled shipments; average order = billed revenue ÷ billed shipments.

Show one table comparing each measured value with what the user typed. They choose, per row, the measured value or their own. Explain one difference plainly: typed rates usually come from Klaviyo or RedChirp reports, which credit people who never clicked and count every send, so they run several times higher than matched rates. The live page counts each order once, so it drops the overlap discount (see **U6**).

**Done when:** every row has the user's choice.

### U4. Agree how corporate gifting and private client orders are marked

Wineries mark these orders in different ways, so use only the rules the user confirms. The same procedure runs later from the page's **Setup** button (see **When the user pastes a Setup prompt**).

1. **Corporate gifting default.** Orders made with Commerce7's Corporate Orders tool carry `purchase_type = 'Corporate Order'`. (The tool isn't available on C7 Lite.) Run the **Program template** on last season with that one rule. If it finds orders, propose it as the first corporate gifting rule.
2. **Channels vary, so don't assume one.** Corporate gift orders can arrive on Web (the buyer orders online), on Inbound (the team keys orders in by hand, common for white-glove corporate service), through a gifting partner that creates batch orders (on Web or Inbound, often with a partner tag, code or customer), or occasionally on Club. Ask whether a partner sends corporate orders, and match rules on every channel unless the split shows a rule catching club or tasting-room orders it shouldn't.
3. **Ask about the rest, one question per program,** allowing several answers (`multiSelect`). Header "Corporate." Question: "Besides the Corporate Orders tool, how does your team mark **corporate gifting** orders in Commerce7?" (drop "Besides the Corporate Orders tool" when step 1 found nothing). Then header "Private client," with the same question for **private client** orders.

   | Option | Description |
   | --- | --- |
   | A tag | "A customer tag or an order tag" |
   | A sales attribute code | "A code your team sets on the order" |
   | A promotion or coupon | "A promotion or coupon used only for these orders" |
   | Nothing else | "We don't mark them any other way" |

   For a tag, ask for its name, check whether it is a customer tag or an order tag, and look up its id. For a code or promotion, ask for its name, then look up its id.
4. **Show the split and confirm.** Run the **Program template** with the chosen rules on last season. Show the resulting split (club, ecommerce, corporate gifting, private client, tasting room, other) and ask the user to confirm. When a rule finds nothing last season, run it on the last 90 days too, since imported history often lacks tags, codes and order types. Warn if a rule catches more than 30% of Other revenue, or catches club or tasting-room orders.
5. **Save** the rules as `live.programRules`, the confirmed split by program and week as last year's program figures, and each program's setup state in `programSetup`: `'rules'` with confirmed rules, `'pending'` when the user said it isn't marked. Rules are checked in order; the first match wins, before the channel defaults. Rule types: `purchase_type`, `customer_tag` (id), `order_tag` (id), `promotion` (promotion_id), `sales_attribute_code` (lowercased), `coupon_text`, `min_total_cents` (against `sub_total`), and the optional `channel`.

   ```js
   { program: 'corp', rule: 'purchase_type', value: 'Corporate Order' }
   ```

**Done when:** each program has confirmed rules, or the user has confirmed it isn't marked, and the user has confirmed the split.

### U5. Link campaigns and audiences

1. Group this season's **Q5** campaigns into offers. For each Email and SMS activity whose send date has passed, propose linking all of one offer's campaigns: same lane, sent within 7 days of the activity's start week (using Q5's real send date), and a similar name or audience.
2. Run **Q4** once for the winery's segments, clubs and customer tags. For each activity's audience, propose a match, then size every proposed audience in one **Q9** call. Never preselect a tag Q4 flags as a likely exclusion or test list. When the winery sends to lists defined in Klaviyo or Mailchimp that nothing in Q4 matches, suggest connecting Claude's Klaviyo or Mailchimp connector for the exact list; that is outside this upgrade, so keep the audience as a label meanwhile.
3. Show one table, and ask for corrections in one reply. An activity may stay unlinked, and an audience may stay a label with a typed list size.

**Done when:** every proposed link and audience has been confirmed, changed or declined.

### U6. Build the sync into the page

Update the page so it syncs **when it opens and when the user presses Refresh**. Grant the page New Vintage's `safe_tenant_sql` only, through your artifact tool's connector capability (for example `{ mcp: { servers: [{ server: 'New Vintage', tools: ['safe_tenant_sql'] }] } }`).

| Page value | Source | When |
| --- | --- | --- |
| Actual revenue's weekly figures and the booked-through week | **Q1**, this season up to today | Every sync |
| Program actuals (all six), so Actual revenue | **Program template** with `live.programRules` | Every sync |
| Email and SMS activity actuals | **Q8** per-campaign variant, summed over each activity's linked campaigns; actual orders from the same rows | Every sync |
| Club activity actuals | **Q7**, this season: billed revenue and billed shipments in each week, credited to the Club activity that bills a shipment that week | Every sync |
| Linked campaign picker | **Q5**, this season | Every sync |
| Audience list sizes | **Q9**, all picked audiences in one call | When picked, and on Refresh when over 24 hours old |
| Audience picker | **Q4** | When the picker opens, if over 24 hours old |
| Last year, baseline and lane defaults | U3 and U4's accepted values | Stored; refreshed only on request |

Page changes:
- **Programs:** add **Tasting room** (baseline only) next to club, ecommerce, corporate gifting, private client and other, so the six programs add up to actual revenue. Actual revenue becomes the sum of program actuals.
- **Baseline:** planned revenue gains last year's weekly Tasting room and Other figures, plus a growth % in Assumptions (default 0%). The goal includes tasting room and phone sales, so without the baseline the plan would always read short.
- **Overlap discount:** remove it and its slider. Matched orders count each order once, so a discount would take overlap off twice.
- **Synced and override fields:** every synced value (program actuals, Email, SMS and Club activity actuals and orders, list sizes, last year) becomes a read-only synced value with its source line ("From Commerce7 · synced 4 min ago") and an **Override** input beside it. With an override, show both values ("Synced $612,400 · Override $640,000 (+$27,600)"); the override drives every total, a sync never changes it, and **Clear override** removes it. Events, Direct outreach and the goal stay plain typed fields. Remove any "Synced" or "Typed" badges inside table rows.
- **List size** for SMS shows promotional consent beside any consent ("312 with promotional consent · 2,932 with any consent"); the synced value is promotional consent.
- **Setup:** a corporate gifting or private client legend row in `programSetup` `'pending'` shows a **Setup** button. It opens a dialog titled "Set up {program}" with a copy-ready prompt and a **Copy** button (clipboard API; when blocked, select the text and say "Press ⌘C to copy"), a **Keep entering by hand** button (sets `'manual'` and hides Setup) and **Close**. The prompt:

  > Set up how my holiday planner recognizes {program} orders. Ask me how my team marks these orders in Commerce7 (tags, codes, promotions, the Corporate Orders tool, or a gifting partner) and which channels they arrive on. Then run the Program template to show me last season's split and confirm it with me. Once I confirm, verify the queries, lock the rules, recalculate this season and last season by program, and update the page so the Setup button for {program} goes away.
- **Season bar:** add a **Refresh** button and a "Synced 4 min ago" stamp at the far right, before More. More gains **Sync details** (where the numbers come from, how orders are matched within 7 days, how programs are assigned, what stays typed) and loses Connect live data.
- **When a sync fails:** keep the last synced numbers with their time, plus "Couldn't refresh. Showing data from Tue 9:14 AM." and a **Retry** button.
- **Saved data:** save the sync results with the plan, as aggregates only.
- **Data model:** keep `schemaVersion` 5, set `edition: 'new-vintage'` and raise `planRevision` by 1. Add `live: { tenantId, syncedAt, programRules, sections, audienceSizes }`, `programSetup: { corp, pc }`, `programActuals` (six programs), `actualByWeek`, and `baseline: { growthPct, byWeek: { tasting, other } }`; drop `overlapDiscountPct`. Each activity gains `linkedCampaigns: [{ platform, campaignId }]`. Each synced value is stored as `{ synced, syncedAt, override }`; carry every number the team typed before the upgrade over as an override, so nothing they entered is lost.
- **Klaviyo's own connector:** use it only for what New Vintage doesn't have yet, such as campaigns not yet sent. Call only its read tools (`get_campaigns`, `get_campaign`, `get_campaign_report`).

**Done when:** the page completes a sync on open, every synced value shows its source line with a working override, and the Setup button shows exactly for programs in `'pending'`.

### U7. Check and hand over

1. Run **Q8**'s bucket output for the season so far. Confirm that `email` + `sms` + `club` + `not_from_activity` equals Q1's actual revenue to the dollar, and that the Program template's six programs add up to the same figure. Fix any gap before going on.
2. Type an override on one program, refresh, and confirm it survives and drives the totals; then clear it.
3. Tell the user, in six lines or fewer:
   - what now syncs
   - what stays typed
   - how to override a number, and that both values stay visible
   - how to link a campaign once it's sent
   - that a program showing **Setup** isn't recognized yet, and the button gives them a prompt to paste here
   - that teammates need their own New Vintage connection to refresh, and every item marked "Assumed — not confirmed," if any

**Done when:** the reconciliation matches, the override check passes, and the user has the notes.

---

## Later requests

### When the user pastes a Setup prompt

1. Run **U4** for that program only.
2. Show the split and ask the user to confirm it. If the winery truly doesn't mark these orders, offer **Keep entering by hand**, which saves `programSetup` as `'manual'`.
3. **Verify the queries:** run the Program template with the new rules for this season up to today and for last season. Both must return, and the six programs must add up to Q1's revenue to the dollar.
4. **Lock the decision:** save the rules in `live.programRules` with `confirmedAt`, set `programSetup` to `'rules'`, and re-run the split for this season's program actuals and last season's program figures. Orders the program now claims leave ecommerce, Other or tasting room, so nothing counts twice.
5. Raise `planRevision` by 1, so the page loads the update and the Setup button for that program disappears.
6. Confirm in one sentence, with the program's actual revenue so far this season.

**Done when:** the program has locked rules (or is set to manual), and the Setup button is gone for it.

---

## Rules

- **Revenue** is `orders.sub_total` (product revenue, in cents), net of refunds. Refund rows (`purchase_type = 'Refund'`) are negative, so sums are already net. Order counts and matched orders count only orders with `sub_total > 0`.
- **Weeks:** the season is 13 weeks from the first Monday on or after Oct 1. The week index is (local order date − season start) ÷ 7. Use `America/Los_Angeles` unless the winery gives another timezone.
- **Programs:** every order lands in exactly one program: the program rules first (corporate gifting, then private client), then Club → club, Web → ecommerce, POS → tasting room, everything else (Inbound included) → other.
- **Matched orders:** a click on a linked email or SMS by a customer, then that customer's **Web or Inbound** paid order within **7 days** of the click. The **most recent click before the order**, across email and SMS together, gets the credit (**Q8**), so no order counts twice. RedChirp stores only a recipient's last click, so for a recipient who clicked more than once, their delivered send time counts as a click too.
- **Lane defaults** are measured per person per offer: matched orders ÷ the distinct people who received each offer, with segment splits, reminders and resends inside the offer. The Club lane default (bill rate and average order) applies only to activities that bill a shipment.
- **Not from an activity** = actual revenue − the sum of activity actuals, never below $0.
- **Provider-reported revenue** (RedChirp's `sms_conversions` and `sms_attribution_summaries`, Klaviyo and Mailchimp in `esp_campaign_reported_revenue`) counts non-clickers, so it runs higher. Show it only as a comparison in chat, never in totals.

## Grouping sends into offers

One offer is one activity. Group a season's Q5 campaigns like this:
1. **Leave out** tests (Q5 `looks_test`) and card-decline texts (Q5 `looks_decline`). Keep everything else, including shipping notices and will-call reminders, and list sends under 25 recipients and unnamed campaigns ("Message 1") for the user to confirm.
2. **Same lane, same send day, one audience between them:** segment splits ("Black Friday – Members" and "Black Friday – Non-Members"), and the same send split across Klaviyo and Mailchimp during a platform switch, are one offer.
3. **Follow-ups within 7 days** of the offer's first send fold into it: "[Follow-up]" campaigns, resends, reminders, "Last chance," "Final hours" and corrections of the same offer.
4. **Group by lane, send day and audience coverage; use names only as a hint.** Klaviyo clone names ("(clone)") and stale subject lines break name matching.
5. **Keep apart:** segments that belong to different programs (a corporate segment of a broader send), and the same offer by email and by SMS (two lanes, two activities).
6. **List size and conversion** count people, not sends: an offer's list size is the distinct people across its campaigns, and its conversion is matched orders ÷ those people. Reminders raise the conversion an offer earns; they don't add list size.

## Data facts

- **Email:** `esp_campaigns` (`id` uuid, `esp_provider` 'klaviyo' | 'mailchimp', `sent_at` = the send time) and `esp_campaign_recipients` (`campaign_id` uuid, `email`, `sent_at`, `first_clicked_at`, `last_clicked_at`). Join to customers through `customer_emails` on lowercased email.
- **SMS:** `sms_campaigns` (`id` text, `sms_provider`, `external_campaign_id`) and `sms_campaign_recipients` (`campaign_id` text, `part_id`, `phone`, `sent_at`, `delivery_state`). Only RedChirp has recipient rows; Mailchimp SMS campaigns have none, so the planner tracks RedChirp texts only. `sms_campaigns.sent_at` is when the text was **scheduled**: the real send is the earliest `sent_at` among `delivery_state = 'sent'` recipients, up to 11 days later. About half of RedChirp recipient rows are `filtered` (never sent).
- **Sent counts:** Klaviyo, the reporting API's count in `esp_campaign_reported_revenue.metadata->>'recipients'` when present, else recipient rows with `sent_at`; Mailchimp, `esp_campaigns.metadata #>> '{report_totals,emails_sent}'`; RedChirp, recipients with `delivery_state IN ('sent','failed')` on linked parts.
- **Complete recipient history:** Klaviyo, rows with `sent_at` ≥ 50% of rows; Mailchimp, rows ≥ 90% of `emails_sent`; RedChirp, sent-or-failed rows ≥ 90% of the latest `sms_attribution_summaries.sent_count`. Older Klaviyo campaigns, and some sent during the busiest weeks, hold only the people who engaged, so they can't be measured. Measure only complete campaigns.
- **SMS phones:** match them through `sms_contacts` **plus** normalized `customer_phones` (97% match, vs 54% with `sms_contacts` alone). Drop a phone that maps to more than 3 customers.
- **Clicks:** only the first and last click per recipient are stored (only the last for RedChirp).
- **Email subscription:** `customers.email_marketing_status` ('Subscribed', 'Unsubscribed' or NULL). `customer_emails.status` is deliverability only ('Ok', 'Bounced').
- **SMS consent:** `sms_consent_current` per contact and category (`promotional`, `conversational`, `informational`), state `subscribed` | `unsubscribed` | `unknown`. Promotional consent is often far smaller than any consent.
- **Club shipments:** `club_member_shipments` (one row per member per package: `club_package_id`, `process_date`, `order_id`). Scheduled = rows; billed = rows with an order.
- **Tags:** `tags.object_type` is capitalized ('Customer', 'Order', 'ClubMembership', 'Reservation').
- **Imported history:** history imported from another platform (for example WineDirect) may lack coupons, promotions, `Corporate Order`, reservations and item types. Rules and baselines must tolerate empty results.
- **Connector limits:**
  - Bound every email recipient query by `campaign_id IN (date-bounded campaigns)`, and every SMS recipient query by `part_id IN (SELECT id FROM sms_campaign_parts WHERE campaign_id IN (...))`; filtering SMS recipients by `campaign_id` times out on large wineries.
  - Compare bare timestamp columns to converted literals.
  - Rejected: `unnest()`, `jsonb_each` and set-returning functions in SELECT, CTE column lists (`x(a, b)`), `MATERIALIZED`, `INTERSECT`, `GROUPING SETS` and `EXPLAIN`.
  - After a timeout, rewrite or narrow the query instead of retrying it.
  - Join a lookup table (tags, promotions) only when a rule uses it.

## Queries

The page substitutes these parameters before each call; keep `'__TENANT_ID__'` literally:
- `:season_start` and `:season_end`: season end is exclusive, `season_start` + 91 days
- `:sms_lookback_start`: `:season_start` − 14 days
- `:order_end`: `least(:season_end, today + 1 day)`

Call `safe_tenant_sql` with `sqlText`, a one-sentence `context`, `tenantId`, and `statementTimeoutMs` 60000 for Q6, Q8 and the Program template (15000 for the rest).

### Q1. Actual revenue by week

```sql
WITH o AS (
  SELECT ((o.order_paid_date AT TIME ZONE 'America/Los_Angeles')::date - DATE ':season_start') / 7 AS week_idx,
         o.channel, o.sub_total
  FROM orders o
  WHERE o.tenant_id = '__TENANT_ID__'
    AND o.order_paid_date >= timestamptz ':season_start 00:00:00 America/Los_Angeles'
    AND o.order_paid_date <  timestamptz ':order_end 00:00:00 America/Los_Angeles'
)
SELECT week_idx,
       count(*) FILTER (WHERE sub_total > 0) AS orders,
       round(sum(sub_total)/100.0, 2) AS revenue_usd,
       round(sum(sub_total) FILTER (WHERE channel = 'Web')/100.0, 2) AS web_usd,
       round(sum(sub_total) FILTER (WHERE channel = 'Inbound')/100.0, 2) AS inbound_usd,
       round(sum(sub_total) FILTER (WHERE channel = 'Club')/100.0, 2) AS club_usd,
       round(sum(sub_total) FILTER (WHERE channel = 'POS')/100.0, 2) AS pos_usd
FROM o GROUP BY week_idx ORDER BY week_idx
```

### Q2. Campaign coverage by season

`:coverage_start` is Oct 1 three seasons back; `:coverage_end` is Jan 1 after the most recent season. SMS seasons here use the scheduling date; Q3 corrects it.

```sql
WITH camp AS (
  SELECT 'email:' || c.esp_provider AS platform, c.sent_at FROM esp_campaigns c
  WHERE c.tenant_id = '__TENANT_ID__' AND c.status = 'sent'
    AND c.sent_at >= timestamptz ':coverage_start 00:00:00 America/Los_Angeles'
    AND c.sent_at <  timestamptz ':coverage_end 00:00:00 America/Los_Angeles'
  UNION ALL
  SELECT 'sms:' || s.sms_provider, s.sent_at FROM sms_campaigns s
  WHERE s.tenant_id = '__TENANT_ID__'
    AND s.sent_at >= timestamptz ':coverage_start 00:00:00 America/Los_Angeles'
    AND s.sent_at <  timestamptz ':coverage_end 00:00:00 America/Los_Angeles'
)
SELECT platform, extract(year FROM sent_at AT TIME ZONE 'America/Los_Angeles')::int AS season,
       count(*) AS campaigns,
       (min(sent_at) AT TIME ZONE 'America/Los_Angeles')::date AS first_sent,
       (max(sent_at) AT TIME ZONE 'America/Los_Angeles')::date AS last_sent
FROM camp WHERE extract(month FROM sent_at AT TIME ZONE 'America/Los_Angeles') >= 10
GROUP BY 1, 2 ORDER BY 1, 2
```

### Q3. Recipient history completeness, one season per call

`:season_year` is the season's year. Run once per season, one at a time. `complete_tail_from` is the date from which every campaign on that platform is complete.

```sql
WITH esp_c AS (
  SELECT c.id, c.esp_provider, c.sent_at,
         nullif(c.metadata #>> '{report_totals,emails_sent}', '')::numeric AS reported_sent
  FROM esp_campaigns c
  WHERE c.tenant_id = '__TENANT_ID__' AND c.status = 'sent'
    AND c.sent_at >= timestamptz ':season_start 00:00:00 America/Los_Angeles'
    AND c.sent_at <  timestamptz ':season_end 00:00:00 America/Los_Angeles'
),
esp_r AS (
  SELECT r.campaign_id, count(*) AS rows_n, count(r.sent_at) AS rows_with_sent_at,
         count(r.first_clicked_at) AS clickers
  FROM esp_campaign_recipients r
  WHERE r.tenant_id = '__TENANT_ID__' AND r.campaign_id IN (SELECT id FROM esp_c)
  GROUP BY r.campaign_id
),
esp_cc AS (
  SELECT 'email:' || c.esp_provider AS platform, c.sent_at,
         coalesce(r.rows_n, 0) AS rows_n, coalesce(r.clickers, 0) AS clickers,
         CAST(NULL AS boolean) AS has_link,
         CASE WHEN c.esp_provider = 'mailchimp' THEN coalesce(r.rows_n, 0) ELSE coalesce(r.rows_with_sent_at, 0) END AS sent_denom,
         CASE
           WHEN c.esp_provider = 'mailchimp'
             THEN coalesce(c.reported_sent > 0 AND coalesce(r.rows_n, 0) >= 0.9 * c.reported_sent, false)
           ELSE coalesce(r.rows_n, 0) > 0 AND r.rows_with_sent_at >= 0.5 * r.rows_n
         END AS is_complete
  FROM esp_c c LEFT JOIN esp_r r ON r.campaign_id = c.id
),
sms_c AS (
  SELECT sc.id, sc.sms_provider, sc.external_campaign_id, sc.sent_at,
         coalesce(position('http' IN sc.message_body) > 0, false) AS has_link
  FROM sms_campaigns sc
  WHERE sc.tenant_id = '__TENANT_ID__'
    AND sc.sent_at >= timestamptz ':sms_lookback_start 00:00:00 America/Los_Angeles'
    AND sc.sent_at <  timestamptz ':season_end 00:00:00 America/Los_Angeles'
),
sms_p AS (
  SELECT sp.id, sp.campaign_id
  FROM sms_campaign_parts sp
  WHERE sp.tenant_id = '__TENANT_ID__' AND sp.campaign_id IN (SELECT id FROM sms_c)
),
sms_r AS (
  SELECT p.campaign_id,
         sum(x.rows_sent) AS rows_sent, sum(x.rows_sent_or_failed) AS rows_sent_or_failed,
         sum(x.clickers) AS clickers, min(x.real_sent_at) AS real_sent_at
  FROM (
    SELECT r.part_id,
           count(*) FILTER (WHERE r.delivery_state = 'sent') AS rows_sent,
           count(*) FILTER (WHERE r.delivery_state IN ('sent', 'failed')) AS rows_sent_or_failed,
           count(r.first_clicked_at) AS clickers,
           min(r.sent_at) FILTER (WHERE r.delivery_state = 'sent') AS real_sent_at
    FROM sms_campaign_recipients r
    WHERE r.tenant_id = '__TENANT_ID__' AND r.part_id IN (SELECT id FROM sms_p)
    GROUP BY r.part_id
  ) x
  JOIN sms_p p ON p.id = x.part_id
  GROUP BY p.campaign_id
),
sms_sum AS (
  SELECT DISTINCT ON (a.sms_provider, a.analytics_group_id) a.sms_provider, a.analytics_group_id, a.sent_count
  FROM sms_attribution_summaries a
  WHERE a.tenant_id = '__TENANT_ID__' AND a.analytics_group_id IN (SELECT external_campaign_id FROM sms_c)
  ORDER BY a.sms_provider, a.analytics_group_id, a.report_end_date DESC
),
sms_cc AS (
  SELECT 'sms:' || c.sms_provider AS platform, coalesce(r.real_sent_at, c.sent_at) AS sent_at,
         coalesce(r.rows_sent_or_failed, 0) AS rows_n, coalesce(r.clickers, 0) AS clickers, c.has_link,
         coalesce(r.rows_sent_or_failed, 0) AS sent_denom,
         coalesce(sm.sent_count > 0 AND coalesce(r.rows_sent_or_failed, 0) >= 0.9 * sm.sent_count, false) AS is_complete
  FROM sms_c c
  LEFT JOIN sms_r r ON r.campaign_id = c.id
  LEFT JOIN sms_sum sm ON sm.analytics_group_id = c.external_campaign_id AND sm.sms_provider = c.sms_provider
  WHERE NOT (coalesce(sm.sent_count, -1) = 0 AND coalesce(r.rows_sent, 0) = 0)
    AND coalesce(r.real_sent_at, c.sent_at) >= timestamptz ':season_start 00:00:00 America/Los_Angeles'
    AND coalesce(r.real_sent_at, c.sent_at) <  timestamptz ':season_end 00:00:00 America/Los_Angeles'
),
all_cc AS (
  SELECT platform, sent_at, rows_n, clickers, has_link, sent_denom, is_complete FROM esp_cc
  UNION ALL
  SELECT platform, sent_at, rows_n, clickers, has_link, sent_denom, is_complete FROM sms_cc
),
flagged AS (
  SELECT platform, sent_at, rows_n, clickers, has_link, sent_denom, is_complete,
         max(CASE WHEN NOT is_complete THEN sent_at END) OVER (PARTITION BY platform) AS last_incomplete_ts
  FROM all_cc
)
SELECT platform, :season_year AS season,
       count(*) AS campaigns_sent,
       count(*) FILTER (WHERE rows_n > 0) AS with_recipient_rows,
       count(*) FILTER (WHERE clickers > 0) AS with_clicks,
       count(*) FILTER (WHERE is_complete) AS complete,
       count(*) FILTER (WHERE is_complete AND clickers > 0) AS complete_with_clicks,
       count(*) FILTER (WHERE rows_n > 0 AND NOT is_complete) AS partial_rows,
       count(*) FILTER (WHERE has_link) AS sms_with_link,
       count(*) FILTER (WHERE has_link AND is_complete AND clickers = 0) AS sms_link_no_clicks,
       round(100.0 * count(*) FILTER (WHERE is_complete) / count(*), 0) AS pct_complete,
       (min(sent_at) FILTER (WHERE is_complete) AT TIME ZONE 'America/Los_Angeles')::date AS first_complete,
       (max(sent_at) FILTER (WHERE is_complete) AT TIME ZONE 'America/Los_Angeles')::date AS last_complete,
       (max(sent_at) FILTER (WHERE NOT is_complete) AT TIME ZONE 'America/Los_Angeles')::date AS last_incomplete,
       (min(sent_at) FILTER (WHERE is_complete AND (last_incomplete_ts IS NULL OR sent_at > last_incomplete_ts))
          AT TIME ZONE 'America/Los_Angeles')::date AS complete_tail_from,
       sum(clickers) AS clickers_all,
       sum(sent_denom) FILTER (WHERE is_complete) AS complete_sent,
       sum(clickers) FILTER (WHERE is_complete) AS complete_clickers
FROM flagged
GROUP BY platform
ORDER BY platform
```

### Q4. Audience picker: segments, clubs and customer tags

New Vintage segments run in the last 45 days, clubs with Active or On Hold members (plus "All active club members," id `ALL`), and customer tags with 25 or more members. `hint` flags likely exclusion lists (`likely_exclusion`) and tests (`test`): keep them in the picker, never preselect them. In the picker, show all segments, then clubs by size (those under 10 members behind "Show all"), then the 25 largest unflagged tags, with the rest searchable and flagged tags last.

```sql
WITH seg_latest AS (
  SELECT DISTINCT ON (h.segment_id) h.segment_id, h.id AS run_id, h.run_date, cardinality(h.customer_ids) AS array_size
  FROM segment_run_history h
  WHERE h.tenant_id = '__TENANT_ID__' AND h.run_date >= now() - interval '45 days'
  ORDER BY h.segment_id, h.run_date DESC, h.id DESC
), seg_rows AS (
  SELECT s.run_id, count(DISTINCT s.customer_id) AS members
  FROM segments s
  WHERE s.tenant_id = '__TENANT_ID__' AND s.run_id IN (SELECT run_id FROM seg_latest)
  GROUP BY s.run_id
), seg AS (
  SELECT 'segment'::text AS kind, l.segment_id::text AS id, d.title AS name,
         coalesce(r.members, l.array_size) AS members, l.run_date AS as_of, d.description AS detail, NULL::text AS hint
  FROM seg_latest l JOIN segment_descriptions d ON d.id = l.segment_id
  LEFT JOIN seg_rows r ON r.run_id = l.run_id
), club AS (
  SELECT 'club'::text, c.id, c.title, count(DISTINCT m.customer_id), max(m.signup_date),
         concat_ws(' · ', c.type, 'web ' || c.web_status, 'admin ' || c.admin_status,
                   count(DISTINCT m.customer_id) FILTER (WHERE m.status = 'On Hold') || ' on hold'),
         CASE WHEN c.title ~* '(^|[^a-z])test' THEN 'test' END
  FROM clubs c JOIN club_memberships m ON m.tenant_id = c.tenant_id AND m.club_id = c.id
  WHERE c.tenant_id = '__TENANT_ID__' AND m.status IN ('Active', 'On Hold')
  GROUP BY c.id, c.title, c.type, c.web_status, c.admin_status
), club_all AS (
  SELECT 'club'::text, 'ALL', 'All active club members', count(DISTINCT m.customer_id), max(m.signup_date),
         'Any club, Active or On Hold', NULL::text
  FROM club_memberships m WHERE m.tenant_id = '__TENANT_ID__' AND m.status IN ('Active', 'On Hold')
), tag_size AS (
  SELECT ct.tag_id, count(*) AS members FROM customer_tags ct
  WHERE ct.tenant_id = '__TENANT_ID__' GROUP BY ct.tag_id HAVING count(*) >= 25
), tag AS (
  SELECT DISTINCT ON (lower(t.title), ts.members)
         'tag'::text, t.id, t.title, ts.members, NULL::timestamptz, t.type,
         CASE WHEN t.title ~* '(unsub|bounce|exclu|suppress|do not|dnc|opt.?out|denied|inactive|fatigue|deceased|blacklist|block|^x[ _-])' THEN 'likely_exclusion'
              WHEN t.title ~* '(^|[^a-z])test' THEN 'test'
              WHEN t.title ~* '^nvl ' THEN 'nvl_segment_tag' END
  FROM tag_size ts JOIN tags t ON t.tenant_id = '__TENANT_ID__' AND t.id = ts.tag_id
  WHERE t.object_type IN ('Customer', 'customer')
  ORDER BY lower(t.title), ts.members, t.updated_at DESC
)
SELECT kind, id, name, members, as_of, detail, hint
FROM (SELECT * FROM seg UNION ALL SELECT * FROM club_all UNION ALL SELECT * FROM club UNION ALL SELECT * FROM tag) a
WHERE members > 0
ORDER BY CASE kind WHEN 'segment' THEN 1 WHEN 'club' THEN 2 ELSE 3 END, (hint IS NOT NULL), members DESC
```

### Q5. Sent campaigns (linked campaign picker, drafting and grouping)

Output: `id, lane, platform, name, sent_date (the real send date), sent_count, complete, clickers, looks_test, looks_decline`. SMS covers RedChirp only, because only RedChirp has recipient rows to match.

```sql
WITH esp_c AS (
  SELECT c.id, c.esp_provider, c.name, c.sent_at,
         nullif(c.metadata #>> '{report_totals,emails_sent}', '')::numeric AS mc_sent
  FROM esp_campaigns c
  WHERE c.tenant_id = '__TENANT_ID__' AND c.status = 'sent'
    AND c.sent_at >= timestamptz ':season_start 00:00:00 America/Los_Angeles'
    AND c.sent_at <  timestamptz ':season_end 00:00:00 America/Los_Angeles'
), esp_r AS (
  SELECT r.campaign_id, count(*) AS rows_n, count(r.sent_at) AS rows_sent_at, count(r.first_clicked_at) AS clickers
  FROM esp_campaign_recipients r
  WHERE r.tenant_id = '__TENANT_ID__' AND r.campaign_id IN (SELECT id FROM esp_c)
  GROUP BY r.campaign_id
), esp_rr AS (
  SELECT rr.campaign_id, nullif(rr.metadata ->> 'recipients', '')::numeric AS api_sent
  FROM esp_campaign_reported_revenue rr
  WHERE rr.tenant_id = '__TENANT_ID__' AND rr.campaign_id IN (SELECT id FROM esp_c)
), sms_c AS (
  SELECT sc.id, sc.name, sc.external_campaign_id, sc.sent_at, coalesce(sc.message_body, '') AS body
  FROM sms_campaigns sc
  WHERE sc.tenant_id = '__TENANT_ID__' AND sc.sms_provider = 'redchirp'
    AND sc.sent_at >= timestamptz ':sms_lookback_start 00:00:00 America/Los_Angeles'
    AND sc.sent_at <  timestamptz ':season_end 00:00:00 America/Los_Angeles'
), sms_p AS (
  SELECT sp.id, sp.campaign_id FROM sms_campaign_parts sp
  WHERE sp.tenant_id = '__TENANT_ID__' AND sp.campaign_id IN (SELECT id FROM sms_c)
), sms_r AS (
  SELECT p.campaign_id, sum(x.n_sent) AS n_sent, sum(x.n_sf) AS n_sf, sum(x.clickers) AS clickers,
         min(x.real_sent_at) AS real_sent_at
  FROM (
    SELECT r.part_id,
           count(*) FILTER (WHERE r.delivery_state = 'sent') AS n_sent,
           count(*) FILTER (WHERE r.delivery_state IN ('sent', 'failed')) AS n_sf,
           count(r.first_clicked_at) AS clickers,
           min(r.sent_at) FILTER (WHERE r.delivery_state = 'sent') AS real_sent_at
    FROM sms_campaign_recipients r
    WHERE r.tenant_id = '__TENANT_ID__' AND r.part_id IN (SELECT id FROM sms_p)
    GROUP BY r.part_id
  ) x JOIN sms_p p ON p.id = x.part_id
  GROUP BY p.campaign_id
), sms_sum AS (
  SELECT DISTINCT ON (a.analytics_group_id) a.analytics_group_id, a.sent_count
  FROM sms_attribution_summaries a
  WHERE a.tenant_id = '__TENANT_ID__' AND a.sms_provider = 'redchirp'
    AND a.analytics_group_id IN (SELECT external_campaign_id FROM sms_c)
  ORDER BY a.analytics_group_id, a.report_end_date DESC
)
SELECT c.id::text AS id, 'email' AS lane, c.esp_provider AS platform, c.name,
       (c.sent_at AT TIME ZONE 'America/Los_Angeles')::date AS sent_date,
       CASE WHEN c.esp_provider = 'mailchimp' THEN c.mc_sent ELSE coalesce(rr.api_sent, r.rows_sent_at) END AS sent_count,
       CASE WHEN c.esp_provider = 'mailchimp' THEN coalesce(c.mc_sent > 0 AND coalesce(r.rows_n, 0) >= 0.9 * c.mc_sent, false)
            ELSE coalesce(r.rows_n, 0) > 0 AND r.rows_sent_at >= 0.5 * r.rows_n END AS complete,
       coalesce(r.clickers, 0) AS clickers,
       coalesce(c.name, '') ~* '(^|[^a-z])test' AS looks_test,
       false AS looks_decline
FROM esp_c c LEFT JOIN esp_r r ON r.campaign_id = c.id LEFT JOIN esp_rr rr ON rr.campaign_id = c.id
UNION ALL
SELECT c.id, 'sms', 'redchirp', c.name,
       (coalesce(r.real_sent_at, c.sent_at) AT TIME ZONE 'America/Los_Angeles')::date,
       coalesce(r.n_sf, 0),
       coalesce(sm.sent_count > 0 AND coalesce(r.n_sf, 0) >= 0.9 * sm.sent_count, false),
       coalesce(r.clickers, 0),
       coalesce(c.name, '') ~* '(^|[^a-z])test',
       (coalesce(c.name, '') || ' ' || c.body) ~* '(declin|card (has )?expired|card (was )?failed|update (your )?(card|payment)|payment (failed|issue))'
FROM sms_c c LEFT JOIN sms_r r ON r.campaign_id = c.id LEFT JOIN sms_sum sm ON sm.analytics_group_id = c.external_campaign_id
WHERE NOT (coalesce(sm.sent_count, -1) = 0 AND coalesce(r.n_sent, 0) = 0)
  AND coalesce(r.real_sent_at, c.sent_at) >= timestamptz ':season_start 00:00:00 America/Los_Angeles'
  AND coalesce(r.real_sent_at, c.sent_at) <  timestamptz ':season_end 00:00:00 America/Los_Angeles'
ORDER BY sent_date, lane
```

### Q6. Lane defaults: matched orders per person per offer

Run with the season's complete Email and SMS campaigns grouped into offers. `:offer_rows` is a `VALUES` list with one row per complete campaign, `('lane', 'platform', 'campaign_id', 'offer_id')`: lane `'email'` or `'sms'`; platform `'klaviyo'`, `'mailchimp'` or `'redchirp'`; the campaign's Q5 id; and any text shared by the campaigns of one offer. `:match_end` is `:season_end` + 7 days, so orders after the season's last sends can still match. Output, one row per lane: `lane, offers, person_offers, sends, matched_orders, matched_revenue_usd`. Conversion = `matched_orders ÷ person_offers`; average order = `matched_revenue_usd ÷ matched_orders`.

```sql
WITH offers AS (
  SELECT column1 AS lane, column2 AS platform, column3 AS campaign_id, column4 AS offer_id
  FROM (VALUES :offer_rows) v
), ecamp AS (
  SELECT c.id AS campaign_id, o.platform, o.offer_id, c.sent_at AS camp_sent_at
  FROM esp_campaigns c JOIN offers o ON o.lane = 'email' AND o.campaign_id = c.id::text
  WHERE c.tenant_id = '__TENANT_ID__'
), er AS (            -- email sends
  SELECT e.offer_id, lower(r.email) AS ident, r.first_clicked_at, r.last_clicked_at
  FROM esp_campaign_recipients r JOIN ecamp e ON e.campaign_id = r.campaign_id
  WHERE r.tenant_id = '__TENANT_ID__' AND r.campaign_id IN (SELECT campaign_id FROM ecamp)
    AND (e.platform = 'mailchimp' OR r.sent_at IS NOT NULL OR r.first_clicked_at IS NOT NULL)
), sp AS (            -- linked RedChirp parts of the listed SMS campaigns
  SELECT p.id AS part_id, o.offer_id
  FROM sms_campaign_parts p JOIN offers o ON o.lane = 'sms' AND o.campaign_id = p.campaign_id
  WHERE p.tenant_id = '__TENANT_ID__' AND p.campaign_id IN (SELECT campaign_id FROM offers WHERE lane = 'sms')
), sr AS (            -- SMS sends
  SELECT s.offer_id, r.phone AS ident, r.first_clicked_at, r.last_clicked_at, r.num_clicks, r.sent_at, r.delivery_state
  FROM sms_campaign_recipients r JOIN sp s ON s.part_id = r.part_id
  WHERE r.tenant_id = '__TENANT_ID__' AND r.part_id IN (SELECT part_id FROM sp)
    AND r.delivery_state IN ('sent','failed')
), reach AS (
  SELECT 'email' AS lane, count(DISTINCT offer_id) AS offers, count(DISTINCT (offer_id, ident)) AS person_offers, count(*) AS sends FROM er
  UNION ALL
  SELECT 'sms', count(DISTINCT offer_id), count(DISTINCT (offer_id, ident)), count(*) FROM sr
), clk AS (           -- click times per recipient
  SELECT 'email' AS lane, ident, first_clicked_at AS click_ts FROM er WHERE first_clicked_at IS NOT NULL
  UNION SELECT 'email', ident, last_clicked_at FROM er WHERE last_clicked_at IS NOT NULL
  UNION SELECT 'sms', ident, first_clicked_at FROM sr WHERE first_clicked_at IS NOT NULL
  UNION SELECT 'sms', ident, last_clicked_at FROM sr WHERE last_clicked_at IS NOT NULL
  UNION SELECT 'sms', ident, sent_at FROM sr WHERE num_clicks > 1 AND delivery_state = 'sent' AND sent_at IS NOT NULL
), ordw AS (          -- candidate orders
  SELECT o.id, o.customer_id, o.order_paid_date, o.sub_total FROM orders o
  WHERE o.tenant_id = '__TENANT_ID__'
    AND o.order_paid_date >= timestamptz ':season_start 00:00:00 America/Los_Angeles'
    AND o.order_paid_date <  timestamptz ':match_end 00:00:00 America/Los_Angeles'
    AND o.channel IN ('Web','Inbound') AND o.purchase_type IS DISTINCT FROM 'Refund' AND o.sub_total > 0
), em AS (            -- email identity map, restricted to ordering customers (fast on large tenants)
  SELECT DISTINCT lower(ce.email) AS ident, ce.customer_id FROM customer_emails ce
  WHERE ce.tenant_id = '__TENANT_ID__' AND ce.customer_id IN (SELECT customer_id FROM ordw)
    AND lower(ce.email) IN (SELECT ident FROM clk WHERE lane = 'email')
), ph_all AS (        -- phone identity map for clicker phones (RedChirp contacts + normalized customer phones)
  SELECT s.phone AS ident, s.customer_id FROM sms_contacts s
  WHERE s.tenant_id = '__TENANT_ID__' AND s.sms_provider = 'redchirp' AND s.customer_id IS NOT NULL
    AND s.phone IN (SELECT ident FROM clk WHERE lane = 'sms')
  UNION
  SELECT y.ident, y.customer_id FROM (
    SELECT CASE WHEN length(regexp_replace(x.phone, '[^0-9]', '', 'g')) = 10 THEN '+1' || regexp_replace(x.phone, '[^0-9]', '', 'g')
                ELSE '+' || regexp_replace(x.phone, '[^0-9]', '', 'g') END AS ident, x.customer_id
    FROM customer_phones x WHERE x.tenant_id = '__TENANT_ID__') y
  WHERE y.ident IN (SELECT ident FROM clk WHERE lane = 'sms')
), ph AS (            -- drop shared phone keys (> 3 customers), keep ordering customers
  SELECT ident, customer_id FROM ph_all
  WHERE customer_id IN (SELECT customer_id FROM ordw)
    AND ident IN (SELECT ident FROM ph_all GROUP BY ident HAVING count(DISTINCT customer_id) <= 3)
), clicks AS (
  SELECT k.lane, k.click_ts, m.customer_id FROM clk k JOIN em m ON m.ident = k.ident WHERE k.lane = 'email'
  UNION
  SELECT k.lane, k.click_ts, m.customer_id FROM clk k JOIN ph m ON m.ident = k.ident WHERE k.lane = 'sms'
), win AS (           -- most recent click within 7 days before the order wins; one credit per order
  SELECT DISTINCT ON (o.id) o.id, o.sub_total, c.lane
  FROM ordw o JOIN clicks c ON c.customer_id = o.customer_id
   AND c.click_ts <= o.order_paid_date AND o.order_paid_date < c.click_ts + interval '7 days'
  ORDER BY o.id, c.click_ts DESC, c.lane
), m AS (
  SELECT lane, count(*) AS matched_orders, round(sum(sub_total) / 100.0, 2) AS matched_revenue_usd FROM win GROUP BY lane
)
SELECT r.lane, r.offers, r.person_offers, r.sends,
       coalesce(m.matched_orders, 0) AS matched_orders, coalesce(m.matched_revenue_usd, 0) AS matched_revenue_usd
FROM reach r LEFT JOIN m ON m.lane = r.lane
ORDER BY r.lane
-- per-offer variant (replay list sizes): replace the final SELECT with
-- SELECT offer_id, count(DISTINCT ident) AS people FROM (SELECT offer_id, ident FROM er UNION ALL SELECT offer_id, ident FROM sr) x GROUP BY offer_id
```

### Q7. Club shipments by week

Each package's shipments count in the week it first processed. Output: `week_idx, packages, scheduled, billed, billed_usd`.

```sql
WITH s AS (
  SELECT s.club_package_id, s.order_id, s.process_date
  FROM club_member_shipments s
  WHERE s.tenant_id = '__TENANT_ID__'
    AND s.process_date >= timestamptz ':season_start 00:00:00 America/Los_Angeles'
    AND s.process_date <  timestamptz ':order_end 00:00:00 America/Los_Angeles'
), pk AS (
  SELECT club_package_id, min(process_date) AS first_process FROM s GROUP BY club_package_id
), o AS (
  SELECT o.id, o.sub_total FROM orders o
  WHERE o.tenant_id = '__TENANT_ID__' AND o.sub_total > 0
    AND o.id IN (SELECT order_id FROM s WHERE order_id IS NOT NULL)
)
SELECT ((pk.first_process AT TIME ZONE 'America/Los_Angeles')::date - DATE ':season_start') / 7 AS week_idx,
       count(DISTINCT s.club_package_id) AS packages,
       count(*) AS scheduled,
       count(o.id) AS billed,
       round(coalesce(sum(o.sub_total), 0) / 100.0, 2) AS billed_usd
FROM s JOIN pk ON pk.club_package_id = s.club_package_id
LEFT JOIN o ON o.id = s.order_id
GROUP BY 1 ORDER BY 1
```

### Q8. Matched orders across email and SMS, and reconciliation

Output: `bucket (club | email | not_from_activity | sms), orders, revenue_usd, reconciled_total_usd, actual_usd`. The buckets add up to Q1's revenue. The per-campaign variant, which gives Email and SMS activity actuals, is in the last comment line.

```sql
WITH ecamp AS (
  SELECT c.id FROM esp_campaigns c
  WHERE c.tenant_id = '__TENANT_ID__' AND c.status = 'sent'
    AND c.sent_at >= timestamptz ':season_start 00:00:00 America/Los_Angeles'
    AND c.sent_at <  timestamptz ':season_end 00:00:00 America/Los_Angeles'
), sp AS (
  SELECT p.id AS part_id, p.campaign_id FROM sms_campaign_parts p
  WHERE p.tenant_id = '__TENANT_ID__'
    AND p.campaign_id IN (SELECT c.id FROM sms_campaigns c
                          WHERE c.tenant_id = '__TENANT_ID__' AND c.sms_provider = 'redchirp'
                            AND c.sent_at >= timestamptz ':sms_lookback_start 00:00:00 America/Los_Angeles'
                            AND c.sent_at <  timestamptz ':season_end 00:00:00 America/Los_Angeles')
), er AS (
  SELECT r.campaign_id::text AS campaign_id, lower(r.email) AS ident, r.first_clicked_at, r.last_clicked_at
  FROM esp_campaign_recipients r
  WHERE r.tenant_id = '__TENANT_ID__' AND r.campaign_id IN (SELECT id FROM ecamp) AND r.first_clicked_at IS NOT NULL
), sr AS (
  SELECT s.campaign_id, r.phone AS ident, r.first_clicked_at, r.last_clicked_at, r.num_clicks, r.sent_at, r.delivery_state
  FROM sms_campaign_recipients r JOIN sp s ON s.part_id = r.part_id
  WHERE r.tenant_id = '__TENANT_ID__' AND r.part_id IN (SELECT part_id FROM sp)
    AND r.first_clicked_at IS NOT NULL AND r.delivery_state IN ('sent', 'failed')
), clk AS (
  SELECT 'email' AS lane, campaign_id, ident, first_clicked_at AS click_ts FROM er
  UNION SELECT 'email', campaign_id, ident, last_clicked_at FROM er WHERE last_clicked_at IS NOT NULL
  UNION SELECT 'sms', campaign_id, ident, first_clicked_at FROM sr
  UNION SELECT 'sms', campaign_id, ident, last_clicked_at FROM sr WHERE last_clicked_at IS NOT NULL
  UNION SELECT 'sms', campaign_id, ident, sent_at FROM sr WHERE num_clicks > 1 AND delivery_state = 'sent' AND sent_at IS NOT NULL
), ord AS (
  SELECT o.id, o.customer_id, o.order_paid_date, o.sub_total, o.channel, o.purchase_type FROM orders o
  WHERE o.tenant_id = '__TENANT_ID__'
    AND o.order_paid_date >= timestamptz ':season_start 00:00:00 America/Los_Angeles'
    AND o.order_paid_date <  timestamptz ':order_end 00:00:00 America/Los_Angeles'
), ordw AS (
  SELECT id, customer_id, order_paid_date, sub_total FROM ord
  WHERE channel IN ('Web', 'Inbound') AND purchase_type IS DISTINCT FROM 'Refund' AND sub_total > 0
), em AS (
  SELECT DISTINCT lower(ce.email) AS ident, ce.customer_id FROM customer_emails ce
  WHERE ce.tenant_id = '__TENANT_ID__' AND ce.customer_id IN (SELECT customer_id FROM ordw)
    AND lower(ce.email) IN (SELECT ident FROM clk WHERE lane = 'email')
), ph_all AS (
  SELECT s.phone AS ident, s.customer_id FROM sms_contacts s
  WHERE s.tenant_id = '__TENANT_ID__' AND s.sms_provider = 'redchirp' AND s.customer_id IS NOT NULL
    AND s.phone IN (SELECT ident FROM clk WHERE lane = 'sms')
  UNION
  SELECT y.ident, y.customer_id FROM (
    SELECT CASE WHEN length(regexp_replace(x.phone, '[^0-9]', '', 'g')) = 10 THEN '+1' || regexp_replace(x.phone, '[^0-9]', '', 'g')
                ELSE '+' || regexp_replace(x.phone, '[^0-9]', '', 'g') END AS ident, x.customer_id
    FROM customer_phones x WHERE x.tenant_id = '__TENANT_ID__') y
  WHERE y.ident IN (SELECT ident FROM clk WHERE lane = 'sms')
), ph AS (
  SELECT ident, customer_id FROM ph_all
  WHERE customer_id IN (SELECT customer_id FROM ordw)
    AND ident IN (SELECT ident FROM ph_all GROUP BY ident HAVING count(DISTINCT customer_id) <= 3)
), clicks AS (
  SELECT k.lane, k.campaign_id, k.click_ts, m.customer_id FROM clk k JOIN em m ON m.ident = k.ident WHERE k.lane = 'email'
  UNION
  SELECT k.lane, k.campaign_id, k.click_ts, m.customer_id FROM clk k JOIN ph m ON m.ident = k.ident WHERE k.lane = 'sms'
), win AS (
  SELECT DISTINCT ON (o.id) o.id AS order_id, o.sub_total, c.lane, c.campaign_id
  FROM ordw o JOIN clicks c ON c.customer_id = o.customer_id
   AND c.click_ts <= o.order_paid_date AND o.order_paid_date < c.click_ts + interval '7 days'
  ORDER BY o.id, c.click_ts DESC, c.lane, c.campaign_id
), bucketed AS (
  SELECT ord.sub_total,
         CASE WHEN win.lane IS NOT NULL THEN win.lane
              WHEN ord.channel = 'Club' THEN 'club'
              ELSE 'not_from_activity' END AS bucket
  FROM ord LEFT JOIN win ON win.order_id = ord.id
)
SELECT bucket, count(*) FILTER (WHERE sub_total > 0) AS orders, round(sum(sub_total) / 100.0, 2) AS revenue_usd,
       round(sum(sum(sub_total)) OVER () / 100.0, 2) AS reconciled_total_usd,
       (SELECT round(sum(sub_total) / 100.0, 2) FROM ord) AS actual_usd
FROM bucketed GROUP BY bucket ORDER BY bucket
-- per-campaign variant: replace the final SELECT with
-- SELECT lane, campaign_id, count(*) AS orders, round(sum(sub_total) / 100.0, 2) AS revenue_usd FROM win GROUP BY lane, campaign_id
```

### Q9. Audience sizes

One call sizes every picked audience: add one `aud` branch per picked audience, with ids inlined as literals, and drop the branches nobody picked. `combined|ALL_PICKED` sizes the picked audiences together, without double counting. Cache the result for 24 hours.

```sql
WITH aud AS (
  SELECT 'segment|5' AS k, s.customer_id FROM segments s
  WHERE s.tenant_id = '__TENANT_ID__'
    AND s.run_id = (SELECT h.id FROM segment_run_history h
                    WHERE h.tenant_id = '__TENANT_ID__' AND h.segment_id = 5
                    ORDER BY h.run_date DESC, h.id DESC LIMIT 1)
  UNION
  SELECT 'club|' || m.club_id, m.customer_id FROM club_memberships m
  WHERE m.tenant_id = '__TENANT_ID__' AND m.status IN ('Active', 'On Hold') AND m.club_id IN (':club_id')
  UNION
  SELECT 'club|ALL', m.customer_id FROM club_memberships m
  WHERE m.tenant_id = '__TENANT_ID__' AND m.status IN ('Active', 'On Hold')
  UNION
  SELECT 'tag|' || ct.tag_id, ct.customer_id FROM customer_tags ct
  WHERE ct.tenant_id = '__TENANT_ID__' AND ct.tag_id IN (':tag_id')
), sms AS (
  SELECT sc.customer_id,
         bool_or(scc.consent_category IN ('promotional', 'marketing') AND scc.state = 'subscribed') AS promo,
         bool_or(scc.state = 'subscribed') AS anysub,
         bool_or(scc.consent_category IN ('promotional', 'marketing') AND scc.state IN ('unsubscribed', 'suppressed')) AS blocked
  FROM sms_contacts sc
  JOIN sms_consent_current scc ON scc.tenant_id = sc.tenant_id AND scc.sms_contact_id = sc.id
  WHERE sc.tenant_id = '__TENANT_ID__' AND scc.tenant_id = '__TENANT_ID__' AND sc.customer_id IS NOT NULL
  GROUP BY sc.customer_id
), flags AS (
  SELECT cu.customer_id,
         EXISTS (SELECT 1 FROM customers c
                 WHERE c.tenant_id = '__TENANT_ID__' AND c.id = cu.customer_id
                   AND c.email_marketing_status IN ('Subscribed', 'subscribed'))
         AND EXISTS (SELECT 1 FROM customer_emails ce
                     WHERE ce.tenant_id = '__TENANT_ID__' AND ce.customer_id = cu.customer_id
                       AND ce.status IS DISTINCT FROM 'Bounced') AS e,
         EXISTS (SELECT 1 FROM customer_phones cp
                 WHERE cp.tenant_id = '__TENANT_ID__' AND cp.customer_id = cu.customer_id) AS p,
         coalesce(x.promo, false) AS s,
         coalesce(x.anysub AND NOT x.blocked, false) AS sa
  FROM (SELECT DISTINCT customer_id FROM aud WHERE customer_id IS NOT NULL) cu
  LEFT JOIN sms x ON x.customer_id = cu.customer_id
), g AS (
  SELECT a.k, count(*) AS members, count(*) FILTER (WHERE f.e) AS with_email, count(*) FILTER (WHERE f.s) AS with_sms_promo,
         count(*) FILTER (WHERE f.sa) AS with_sms_any_consent, count(*) FILTER (WHERE f.p) AS with_phone
  FROM aud a JOIN flags f ON f.customer_id = a.customer_id
  GROUP BY a.k
  UNION ALL
  SELECT 'combined|ALL_PICKED', count(*), count(*) FILTER (WHERE f.e), count(*) FILTER (WHERE f.s),
         count(*) FILTER (WHERE f.sa), count(*) FILTER (WHERE f.p)
  FROM flags f
), sms_flag AS (
  SELECT EXISTS (SELECT 1 FROM sms_consent_current z
                 WHERE z.tenant_id = '__TENANT_ID__' AND z.consent_category IN ('promotional', 'marketing')) AS has_consent
)
SELECT split_part(g.k, '|', 1) AS kind, split_part(g.k, '|', 2) AS id, g.members, g.with_email,
       CASE WHEN sf.has_consent THEN g.with_sms_promo ELSE g.with_phone END AS with_sms,
       CASE WHEN sf.has_consent THEN 'promotional_consent' ELSE 'phone_on_file' END AS sms_basis,
       g.with_sms_any_consent, g.with_phone
FROM g CROSS JOIN sms_flag sf ORDER BY 1, 2
```

### Program template (program revenue by week, with `live.programRules`)

Fill each `IN (...)` from the confirmed rules. For every rule type the plan doesn't use, remove its CTE, its `LEFT JOIN` and its condition, and replace an unused `:…_inline_predicate` with `FALSE`. `:corp_channel_predicate` and `:pc_channel_predicate` are `TRUE` unless a `channel` rule narrows the program (then `ord.channel IN ('Web','Inbound')`, for example). Keep the template to the lookups the rules need, because an empty lookup still scans the whole tag table and times out. With only the `Corporate Order` rule, the template reduces to a single query on `orders`. The six programs always add up to Q1's revenue.

```sql
WITH ord AS (
  SELECT o.id, o.customer_id, o.channel, o.purchase_type, o.sales_attribute_code, o.sub_total,
         coalesce(o.coupons::text, '') AS coupons_txt,
         ((o.order_paid_date AT TIME ZONE 'America/Los_Angeles')::date - DATE ':season_start') / 7 AS week_idx
  FROM orders o
  WHERE o.tenant_id = '__TENANT_ID__'
    AND o.order_paid_date >= timestamptz ':season_start 00:00:00 America/Los_Angeles'
    AND o.order_paid_date <  timestamptz ':order_end 00:00:00 America/Los_Angeles'
), corp_cust AS (
  SELECT DISTINCT ct.customer_id FROM customer_tags ct WHERE ct.tenant_id = '__TENANT_ID__' AND ct.tag_id IN (:corp_customer_tag_ids)
), corp_ord AS (
  SELECT ot.order_id FROM order_tags ot WHERE ot.tenant_id = '__TENANT_ID__' AND ot.tag_id IN (:corp_order_tag_ids)
  UNION SELECT p.order_id FROM order_promotions p WHERE p.tenant_id = '__TENANT_ID__' AND p.promotion_id IN (:corp_promotion_ids)
), pc_cust AS (
  SELECT DISTINCT ct.customer_id FROM customer_tags ct WHERE ct.tenant_id = '__TENANT_ID__' AND ct.tag_id IN (:pc_customer_tag_ids)
), pc_ord AS (
  SELECT ot.order_id FROM order_tags ot WHERE ot.tenant_id = '__TENANT_ID__' AND ot.tag_id IN (:pc_order_tag_ids)
  UNION SELECT p.order_id FROM order_promotions p WHERE p.tenant_id = '__TENANT_ID__' AND p.promotion_id IN (:pc_promotion_ids)
), prog AS (
  SELECT ord.week_idx, ord.sub_total,
    CASE
      WHEN (corp_cust.customer_id IS NOT NULL OR corp_ord.order_id IS NOT NULL
            OR ord.purchase_type IN (:corp_purchase_types) OR lower(ord.sales_attribute_code) IN (:corp_sales_codes)
            OR :corp_inline_predicate) AND :corp_channel_predicate THEN 'corp'
      WHEN (pc_cust.customer_id IS NOT NULL OR pc_ord.order_id IS NOT NULL
            OR ord.purchase_type IN (:pc_purchase_types) OR lower(ord.sales_attribute_code) IN (:pc_sales_codes)
            OR :pc_inline_predicate) AND :pc_channel_predicate THEN 'pc'
      WHEN ord.channel = 'Club' THEN 'club'
      WHEN ord.channel = 'Web'  THEN 'ecom'
      WHEN ord.channel = 'POS'  THEN 'tasting'
      ELSE 'other'
    END AS program
  FROM ord
  LEFT JOIN corp_cust ON corp_cust.customer_id = ord.customer_id
  LEFT JOIN corp_ord  ON corp_ord.order_id = ord.id
  LEFT JOIN pc_cust   ON pc_cust.customer_id = ord.customer_id
  LEFT JOIN pc_ord    ON pc_ord.order_id = ord.id
)
SELECT week_idx, program,
       count(*) FILTER (WHERE sub_total > 0) AS orders,
       round(sum(sub_total) / 100.0, 2) AS revenue_usd
FROM prog GROUP BY 1, 2 ORDER BY 1, 2
```

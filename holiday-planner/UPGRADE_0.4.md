# Holiday Campaign Planner — live data upgrade v0.4

*For the winery: when you're ready to make your planner's numbers live, turn on the New Vintage connector in Claude, then upload this file in the chat where you built your planner and say "Connect my planner to New Vintage."*

You are upgrading a Holiday Campaign Planner that was built from the build spec. You'll turn it into a live planner whose numbers come from the winery's Commerce7, email and SMS data through the New Vintage connector. The page, its layout and the team's activities stay as they are. Work through the **Steps** in order. Each step ends on a **Done when** line; finish it before starting the next.

All data access is read-only:
- Run SELECT queries through `safe_tenant_sql`.
- Leave every write tool uncalled.
- Keep customer names, emails and phones out of the chat and out of the page. Every query below returns aggregates only.

---

## Steps

### U1. Check the connection

1. Run `discover_tenant_data_sources`. If it reaches several tenants, ask which winery this is.
2. Record:
   - Commerce7's last sync
   - which email platforms are synced (Klaviyo, Mailchimp or both) and whether RedChirp is
   - for each, the date range of campaigns with recipient click history
3. Show the user this as a four-line summary.

If the connector is missing, or the account lacks Pro access, tell the user to turn on the New Vintage connector in Claude's settings, then stop.

**Done when:** the user has seen the summary and you know the one tenant.

### U2. Start from the latest plan

If the user has edited the page since your last change in this chat, ask them to use More → Download CSV and attach the file, and treat it as the current plan. Otherwise use the plan you last built.

**Done when:** you have the current plan, including every typed number.

### U3. Measure last season

With last season's dates:
1. Run **Q1** for last year's total and weekly figures.
2. Run the **Program template** with no rules (Web → ecommerce, Club → club, the rest unmapped) for the program figures.
3. Run **Q9** (per-campaign variant, grouped by lane) and **Q-Sent** for the Email and SMS lane defaults:
   - conversion = matched orders ÷ sent
   - average order = matched revenue ÷ matched orders

Show one table comparing each value with what the user typed. They choose, per row, to use the measured value or keep their own.

**Done when:** every row has the user's choice.

### U4. Link campaigns and audiences

1. For each Email and SMS activity whose send date has passed, propose a linked campaign from **Q5**: same lane, sent within 7 days of the activity's start week, and a similar name or audience.
2. For each activity's audience, propose a New Vintage segment, Commerce7 club or Commerce7 tag, with its size from **Q10**.
3. Show one table, and ask for corrections in one reply. An activity may stay unlinked, and an audience may stay a label with a typed list size.

**Done when:** every proposed link and audience has been confirmed, changed or declined.

### U5. Agree how corporate gifting and private client orders are marked

Wineries mark these orders in different ways, so use only the rules the user confirms.

1. **Corporate gifting default.** Orders made with Commerce7's Corporate Orders tool carry `purchase_type = 'Corporate Order'` and arrive on the Inbound channel. (The tool isn't available on C7 Lite.) Run the **Program template** on last season with that one rule. If it finds orders, propose it as the first corporate gifting rule.
2. **Ask about the rest.** Corporate gifts taken by phone or on the website usually carry no type, and private client orders never do. Ask one question per program: "How does your team mark **Corporate gifting** orders in Commerce7?" Then the same for **Private client**. Options:
   - a customer tag
   - an order tag
   - a sales attribute code
   - a promotion or coupon
   - only the Corporate Orders tool (corporate gifting only)
   - "We don't mark them"

   For a tag, code or promotion, ask for its name, then look up its id by that name.
3. **Show the split and confirm.** Run the **Program template** with the chosen rules on last season. Show the resulting split (corporate gifting, private client, ecommerce, club, unmapped) and ask the user to confirm. Warn if a rule catches more than 30% of unmapped revenue or overlaps Club.
4. **Save the rules** as `live.programRules`. Rules are checked in order; the first match wins, before the Web and Club defaults.

   ```js
   { program: 'corp', rule: 'purchase_type', value: 'Corporate Order' }
   ```

   Rule types:
   - `purchase_type`
   - `customer_tag` (id)
   - `order_tag` (id)
   - `promotion` (promotion_id)
   - `sales_attribute_code` (lowercased)
   - `coupon_text`
   - `min_total_cents` (against `sub_total`)

   A program the user doesn't mark keeps its typed figures.

**Done when:** each program has confirmed rules, or the user has confirmed it isn't marked, and the split has been shown.

### U6. Build the sync into the page

Update the page so it syncs **when it opens and when the user presses Refresh**:

| Page value | Source | Notes |
| --- | --- | --- |
| Actual revenue and booked-through week | **Q1**, this season up to today | |
| Program actuals in the legend | **Program template** with `live.programRules` | Unmapped revenue shows as Other DTC |
| Email and SMS activity actuals | **Q9** per-campaign variant, summed over each activity's linked campaigns | Actual orders come from the same query |
| Club activity actuals | Club-channel orders in the activity's weeks, credited to the Club activity that bills a shipment that week | |
| Audience list sizes | **Q10** | |
| Last year and lane defaults | U3's accepted values | Stored; refreshed only on request |

Page behavior:
- **Stays typed:** events and Direct outreach, and anything the user chose to keep typed.
- **Synced values** show a small source line ("From Commerce7 · synced 4 min ago") and a **Type my own** link.
- **Typed overrides:** a typed value always wins. It shows "Typed by you" and a **Use synced value** link.
- **Season bar:** add a **Refresh** button and a "Synced 4 min ago" stamp.
- **When a sync fails:** keep the last synced numbers with their time, plus "Couldn't refresh. Showing data from Tue 9:14 AM." and a **Retry** button.
- **Saved data:** save the sync results with the plan, as aggregates only.
- **Data model:** add a `live` object to the plan:

  ```
  live: { tenantId, syncedAt, programRules, laneDefaultSource, lastYearSource }
  ```

  - Each activity gains `linkedCampaigns: [{platform, campaignId}]`.
  - Each synced field is stored as `{ value, source: 'synced' | 'typed', syncedAt }`.
  - Raise `planRevision` by 1.
- **Klaviyo's own connector:** use it only for what New Vintage doesn't have yet, such as campaigns not yet sent. Call only its read tools (`get_campaigns`, `get_campaign`, `get_campaign_report`).

**Done when:** the page completes a sync on open, and every synced value shows its source line with a working override.

### U7. Check and hand over

1. Run **Q9**'s bucket output for the season so far. Confirm that email + SMS + club + Other DTC equals Q1's actual revenue to the dollar, and fix any gap before going on.
2. Tell the user, in five lines or fewer:
   - what now syncs
   - what stays typed
   - how to override a number
   - how to link a campaign once it's sent
   - that teammates need their own New Vintage connection to refresh

**Done when:** the reconciliation matches, and the user has the notes.

---

## Rules

- **Revenue** is `orders.sub_total` (product revenue, in cents), net of refunds. Refund rows (`purchase_type = 'Refund'`) are negative, so sums are already net.
- **Weeks:** the season is 13 weeks from the first Monday on or after Oct 1. The week index is (local order date − season start) ÷ 7. Use `America/Los_Angeles` unless the winery gives another timezone.
- **Matched orders:** a click on a linked email or SMS by a customer, then that customer's **Web or Inbound** paid order within **5 days** of the click. The **most recent click before the order**, across email and SMS together, gets the credit (**Q9**), so no order counts twice. Inbound matters: many wineries take email- and SMS-driven orders by phone.
- **Other DTC** = actual revenue − matched email and SMS revenue − Club revenue.
- **Provider-reported revenue** (RedChirp's `sms_conversions` and `sms_attribution_summaries`, Klaviyo reports) is send-based and counts non-clickers, so it runs higher. Show it only as a comparison, never in totals.

## Data facts

- **Email recipients:** `esp_campaign_recipients` (`campaign_id`, `email`, `first_clicked_at`, `last_clicked_at`). Join to customers through `customer_emails` on lowercased email.
- **Recipient dates:** take them from `esp_campaigns.sent_at`. Mailchimp recipient rows can carry an empty `sent_at`.
- **Sent counts:** for Mailchimp, `esp_campaigns.metadata #>> '{report_totals,emails_sent}'`; for Klaviyo, recipient rows with `sent_at`; for RedChirp, the latest `sms_attribution_summaries.sent_count` per campaign (**Q-Sent**).
- **SMS phones:** match them through `sms_contacts` **plus** normalized `customer_phones`; `sms_contacts` alone misses many customers.
- **Clicks:** only the first and last click per recipient are stored, so "most recent click" uses those two.
- **Imported history:** history imported from another platform may lack coupons, promotions, `Corporate Order`, reservations and item types. Rules and baselines must tolerate empty results.
- **Connector limits:**
  - Bound every recipient query by `campaign_id IN (date-bounded campaigns)`.
  - Compare bare timestamp columns to converted literals.
  - `unnest()`, `jsonb_each` and set-returning functions in SELECT are rejected; use `#>>`, `= ANY()` or `::text ~*`.
  - Give attribution queries `statementTimeoutMs` 45000–60000. After a timeout, narrow the window instead of retrying.
  - Join a lookup table (tags, promotions) only when a rule uses it.

## Queries

The page substitutes these parameters before each call; keep `'__TENANT_ID__'` literally:
- `:season_start` and `:season_end`: season end is exclusive, `season_start` + 91 days
- `:order_end`: `least(:season_end, today + 1 day)`

Call `safe_tenant_sql` with `sqlText`, a one-sentence `context` and, if needed, `tenantId`.

### Q1. Actual revenue by week

```sql
WITH o AS (
  SELECT ((o.order_paid_date AT TIME ZONE 'America/Los_Angeles')::date - DATE ':season_start') / 7 AS week_idx,
         o.channel, o.purchase_type, o.sub_total
  FROM orders o
  WHERE o.tenant_id = '__TENANT_ID__'
    AND o.order_paid_date >= timestamptz ':season_start 00:00:00 America/Los_Angeles'
    AND o.order_paid_date <  timestamptz ':order_end 00:00:00 America/Los_Angeles'
)
SELECT week_idx,
       count(*) FILTER (WHERE purchase_type IS DISTINCT FROM 'Refund') AS orders,
       round(sum(sub_total)/100.0, 2) AS revenue_usd,
       round(sum(sub_total) FILTER (WHERE channel = 'Web')/100.0, 2) AS web_usd,
       round(sum(sub_total) FILTER (WHERE channel = 'Inbound')/100.0, 2) AS inbound_usd,
       round(sum(sub_total) FILTER (WHERE channel = 'Club')/100.0, 2) AS club_usd,
       round(sum(sub_total) FILTER (WHERE channel = 'POS')/100.0, 2) AS pos_usd
FROM o GROUP BY week_idx ORDER BY week_idx
```

### Q5. Sent campaigns (linked campaign picker)

Output: `id, platform (klaviyo | mailchimp | redchirp), name, sent_date, sent_count`.

```sql
WITH ec AS (
  SELECT c.id::text AS id, c.esp_provider AS platform, c.name, c.sent_at,
         (c.metadata #>> '{report_totals,emails_sent}')::int AS mc_sent
  FROM esp_campaigns c
  WHERE c.tenant_id = '__TENANT_ID__' AND c.status = 'sent'
    AND c.sent_at >= timestamptz ':season_start 00:00:00 America/Los_Angeles'
    AND c.sent_at <  timestamptz ':season_end 00:00:00 America/Los_Angeles'
), kr AS (
  SELECT r.campaign_id::text AS id, count(*) FILTER (WHERE r.sent_at IS NOT NULL) AS sent
  FROM esp_campaign_recipients r
  WHERE r.tenant_id = '__TENANT_ID__'
    AND r.campaign_id IN (SELECT c.id FROM esp_campaigns c WHERE c.tenant_id = '__TENANT_ID__' AND c.esp_provider = 'klaviyo'
                          AND c.sent_at >= timestamptz ':season_start 00:00:00 America/Los_Angeles'
                          AND c.sent_at <  timestamptz ':season_end 00:00:00 America/Los_Angeles')
  GROUP BY r.campaign_id
), sc AS (
  SELECT c.id, c.sms_provider AS platform, c.name, c.sent_at, c.external_campaign_id
  FROM sms_campaigns c
  WHERE c.tenant_id = '__TENANT_ID__'
    AND c.sent_at >= timestamptz ':season_start 00:00:00 America/Los_Angeles'
    AND c.sent_at <  timestamptz ':season_end 00:00:00 America/Los_Angeles'
), ss AS (
  SELECT DISTINCT ON (s.analytics_group_id) s.analytics_group_id, s.sent_count
  FROM sms_attribution_summaries s
  WHERE s.tenant_id = '__TENANT_ID__' AND s.analytics_group_id IN (SELECT external_campaign_id FROM sc)
  ORDER BY s.analytics_group_id, s.report_end_date DESC
)
SELECT ec.id, ec.platform, ec.name, (ec.sent_at AT TIME ZONE 'America/Los_Angeles')::date AS sent_date,
       CASE WHEN ec.platform = 'mailchimp' THEN ec.mc_sent ELSE kr.sent END AS sent_count
FROM ec LEFT JOIN kr ON kr.id = ec.id
UNION ALL
SELECT sc.id, sc.platform, sc.name, (sc.sent_at AT TIME ZONE 'America/Los_Angeles')::date, ss.sent_count
FROM sc LEFT JOIN ss ON ss.analytics_group_id = sc.external_campaign_id
ORDER BY sent_date, platform
```

### Q9. Matched orders across email and SMS, and reconciliation

Output: `bucket (email | sms | club | other_dtc), orders, revenue_usd, reconciled_total_usd, actual_usd`. The per-campaign variant is in the last comment line.

```sql
WITH ecamp AS (
  SELECT c.id FROM esp_campaigns c
  WHERE c.tenant_id = '__TENANT_ID__' AND c.status = 'sent'
    AND c.sent_at >= timestamptz ':season_start 00:00:00 America/Los_Angeles'
    AND c.sent_at <  timestamptz ':season_end 00:00:00 America/Los_Angeles'
), scamp AS (
  SELECT c.id FROM sms_campaigns c
  WHERE c.tenant_id = '__TENANT_ID__' AND c.sms_provider = 'redchirp'
    AND c.sent_at >= timestamptz ':season_start 00:00:00 America/Los_Angeles'
    AND c.sent_at <  timestamptz ':season_end 00:00:00 America/Los_Angeles'
), erc AS (
  SELECT r.campaign_id::text AS campaign_id, lower(r.email) AS email, r.first_clicked_at, r.last_clicked_at
  FROM esp_campaign_recipients r
  WHERE r.tenant_id = '__TENANT_ID__' AND r.campaign_id IN (SELECT id FROM ecamp) AND r.first_clicked_at IS NOT NULL
), src AS (
  SELECT r.campaign_id, r.phone, r.first_clicked_at, r.last_clicked_at FROM sms_campaign_recipients r
  WHERE r.tenant_id = '__TENANT_ID__' AND r.campaign_id IN (SELECT id FROM scamp) AND r.first_clicked_at IS NOT NULL
), em AS (
  SELECT DISTINCT lower(ce.email) AS email, ce.customer_id FROM customer_emails ce WHERE ce.tenant_id = '__TENANT_ID__'
), ph AS (
  SELECT s.phone, s.customer_id FROM sms_contacts s
  WHERE s.tenant_id = '__TENANT_ID__' AND s.sms_provider = 'redchirp' AND s.customer_id IS NOT NULL
  UNION
  SELECT CASE WHEN length(regexp_replace(x.phone, '[^0-9]', '', 'g')) = 10 THEN '+1' || regexp_replace(x.phone, '[^0-9]', '', 'g')
              ELSE '+' || regexp_replace(x.phone, '[^0-9]', '', 'g') END, x.customer_id
  FROM customer_phones x WHERE x.tenant_id = '__TENANT_ID__'
), eclk AS (
  SELECT campaign_id, email, first_clicked_at AS click_ts FROM erc
  UNION SELECT campaign_id, email, last_clicked_at FROM erc WHERE last_clicked_at IS NOT NULL
), sclk AS (
  SELECT campaign_id, phone, first_clicked_at AS click_ts FROM src
  UNION SELECT campaign_id, phone, last_clicked_at FROM src WHERE last_clicked_at IS NOT NULL
), clicks AS (
  SELECT 'email' AS lane, eclk.campaign_id, eclk.click_ts, em.customer_id FROM eclk JOIN em ON em.email = eclk.email
  UNION
  SELECT 'sms', sclk.campaign_id, sclk.click_ts, ph.customer_id FROM sclk JOIN ph ON ph.phone = sclk.phone
), ord AS (
  SELECT o.id, o.customer_id, o.order_paid_date, o.sub_total, o.channel, o.purchase_type FROM orders o
  WHERE o.tenant_id = '__TENANT_ID__'
    AND o.order_paid_date >= timestamptz ':season_start 00:00:00 America/Los_Angeles'
    AND o.order_paid_date <  timestamptz ':order_end 00:00:00 America/Los_Angeles'
), cand AS (
  SELECT ord.id AS order_id, ord.sub_total, c.lane, c.campaign_id,
         row_number() OVER (PARTITION BY ord.id ORDER BY c.click_ts DESC, c.lane, c.campaign_id) AS rn
  FROM ord JOIN clicks c ON c.customer_id = ord.customer_id
   AND c.click_ts <= ord.order_paid_date AND ord.order_paid_date < c.click_ts + interval '5 days'
  WHERE ord.channel IN ('Web','Inbound') AND ord.purchase_type IS DISTINCT FROM 'Refund'
), win AS (SELECT order_id, lane, campaign_id FROM cand WHERE rn = 1),
bucketed AS (
  SELECT ord.sub_total,
         CASE WHEN win.lane IS NOT NULL THEN win.lane
              WHEN ord.channel = 'Club' THEN 'club'
              ELSE 'other_dtc' END AS bucket
  FROM ord LEFT JOIN win ON win.order_id = ord.id
)
SELECT bucket, count(*) AS orders, round(sum(sub_total)/100.0,2) AS revenue_usd,
       round(sum(sum(sub_total)) OVER ()/100.0,2) AS reconciled_total_usd,
       (SELECT round(sum(sub_total)/100.0,2) FROM ord) AS actual_usd
FROM bucketed GROUP BY bucket ORDER BY bucket
-- per-campaign variant: SELECT lane, campaign_id, count(*), round(sum(sub_total)/100.0,2) FROM cand WHERE rn = 1 GROUP BY 1,2
```

### Q-Sent. Sent counts for lane defaults

```sql
WITH ecamp AS (
  SELECT c.id, c.esp_provider, (c.metadata #>> '{report_totals,emails_sent}')::int AS mc_sent FROM esp_campaigns c
  WHERE c.tenant_id = '__TENANT_ID__' AND c.status = 'sent'
    AND c.sent_at >= timestamptz ':season_start 00:00:00 America/Los_Angeles'
    AND c.sent_at <  timestamptz ':season_end 00:00:00 America/Los_Angeles'
), kl AS (
  SELECT r.campaign_id, count(*) FILTER (WHERE r.sent_at IS NOT NULL) AS n FROM esp_campaign_recipients r
  WHERE r.tenant_id = '__TENANT_ID__' AND r.campaign_id IN (SELECT id FROM ecamp WHERE esp_provider = 'klaviyo') GROUP BY r.campaign_id
), scamp AS (
  SELECT c.external_campaign_id FROM sms_campaigns c
  WHERE c.tenant_id = '__TENANT_ID__' AND c.sms_provider = 'redchirp'
    AND c.sent_at >= timestamptz ':season_start 00:00:00 America/Los_Angeles'
    AND c.sent_at <  timestamptz ':season_end 00:00:00 America/Los_Angeles'
), ss AS (
  SELECT DISTINCT ON (x.analytics_group_id) x.sent_count FROM sms_attribution_summaries x
  WHERE x.tenant_id = '__TENANT_ID__' AND x.sms_provider = 'redchirp' AND x.analytics_group_id IN (SELECT external_campaign_id FROM scamp)
  ORDER BY x.analytics_group_id, x.report_end_date DESC
)
SELECT 'email' AS lane, (SELECT sum(mc_sent) FROM ecamp WHERE esp_provider = 'mailchimp') + (SELECT coalesce(sum(n),0) FROM kl) AS sent
UNION ALL SELECT 'sms', (SELECT sum(sent_count) FROM ss)
```

### Q10. Audience sizes

```sql
WITH seg AS (
  SELECT h.customer_ids FROM segment_run_history h
  WHERE h.tenant_id = '__TENANT_ID__' AND h.segment_id = :segment_id
  ORDER BY h.run_date DESC LIMIT 1
), aud AS (
  SELECT 'segment' AS kind, ':segment_id' AS id, c.id AS customer_id
  FROM customers c, seg WHERE c.tenant_id = '__TENANT_ID__' AND c.id = ANY(seg.customer_ids)
  UNION
  SELECT 'club', m.club_id, m.customer_id FROM club_memberships m
  WHERE m.tenant_id = '__TENANT_ID__' AND m.club_id = ':club_id' AND m.status = 'Active'
  UNION
  SELECT 'tag', ct.tag_id, ct.customer_id FROM customer_tags ct
  WHERE ct.tenant_id = '__TENANT_ID__' AND ct.tag_id = ':tag_id'
), em AS (
  SELECT DISTINCT ce.customer_id FROM customer_emails ce
  WHERE ce.tenant_id = '__TENANT_ID__' AND ce.customer_id IN (SELECT customer_id FROM aud)
    AND coalesce(ce.status,'subscribed') NOT IN ('unsubscribed','bounced')
), ph AS (
  SELECT DISTINCT cp.customer_id FROM customer_phones cp
  WHERE cp.tenant_id = '__TENANT_ID__' AND cp.customer_id IN (SELECT customer_id FROM aud)
)
SELECT aud.kind, aud.id, count(DISTINCT aud.customer_id) AS members,
       count(DISTINCT em.customer_id) AS with_email, count(DISTINCT ph.customer_id) AS with_phone
FROM aud LEFT JOIN em ON em.customer_id = aud.customer_id LEFT JOIN ph ON ph.customer_id = aud.customer_id
GROUP BY aud.kind, aud.id
```

### Program template (program revenue by week, with `live.programRules`)

Fill each `IN (...)` from the confirmed rules. For every rule type the plan doesn't use, remove its CTE, its `LEFT JOIN` and its condition. Keep the template to the lookups the rules need, because an empty lookup still scans the whole tag table and times out. With only the `Corporate Order` rule, the template reduces to a single query on `orders`.

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
  SELECT ord.week_idx, ord.sub_total, ord.purchase_type,
    CASE
      WHEN ord.channel IN (:corp_channels) AND (corp_cust.customer_id IS NOT NULL OR corp_ord.order_id IS NOT NULL
           OR ord.purchase_type IN (:corp_purchase_types) OR lower(ord.sales_attribute_code) IN (:corp_sales_codes)
           OR :corp_inline_predicate) THEN 'corp'          -- e.g. ord.coupons_txt ILIKE '%CORP%' / ord.sub_total >= 250000 / FALSE
      WHEN ord.channel IN (:pc_channels) AND (pc_cust.customer_id IS NOT NULL OR pc_ord.order_id IS NOT NULL
           OR ord.purchase_type IN (:pc_purchase_types) OR lower(ord.sales_attribute_code) IN (:pc_sales_codes)
           OR :pc_inline_predicate) THEN 'pc'
      WHEN ord.channel = 'Club' THEN 'club'
      WHEN ord.channel = 'Web'  THEN 'ecom'
      ELSE 'unmapped'
    END AS program
  FROM ord
  LEFT JOIN corp_cust ON corp_cust.customer_id = ord.customer_id
  LEFT JOIN corp_ord  ON corp_ord.order_id = ord.id
  LEFT JOIN pc_cust   ON pc_cust.customer_id = ord.customer_id
  LEFT JOIN pc_ord    ON pc_ord.order_id = ord.id
)
SELECT week_idx, program,
       count(*) FILTER (WHERE purchase_type IS DISTINCT FROM 'Refund') AS orders,
       round(sum(sub_total)/100.0, 2) AS revenue_usd
FROM prog GROUP BY 1,2 ORDER BY 1,2
```

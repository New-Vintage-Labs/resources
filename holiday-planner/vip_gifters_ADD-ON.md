# Holiday Campaign Planner — VIP gifters module

- **Version:** 0.1.1
- **Requires:** a connected planner, built from [holiday_planner_SPEC.md](holiday_planner_SPEC.md) (v0.5) and upgraded with [holiday_planner_UPGRADE.md](holiday_planner_UPGRADE.md) (v0.5.1), or built directly from [holiday_planner_SPEC_NVPRO.md](holiday_planner_SPEC_NVPRO.md) (v0.5)

*For the winery: use the same chat and artifact as your existing New Vintage planner, with the New Vintage connector on. If you started with the manual holiday_planner_SPEC.md, first run holiday_planner_UPGRADE.md on that same artifact. A planner built directly from holiday_planner_SPEC_NVPRO.md is also supported. Then upload this file and say "Add VIP gifters to my planner." Do not rebuild an existing planner.*

You are adding a **VIP gifters** tab to a winery's live Holiday Campaign Planner. It answers one question: **which of our biggest gift-givers have ordered this season, and whose revenue is at risk?** A handful of corporate and individual gifters often carry a large share of holiday revenue, and one or two of them skipping a season can break the plan. The tab finds those gifters in Commerce7 through the New Vintage connector, compares each one's ordering this season with their own history, and ties the corporate ones to the planner's corporate gifting actual. The plan, its programs and every number the team typed stay as they are.

Work through the **Steps** in order. Each step ends on a **Done when** line; finish it before starting the next. The sections after the Steps define the page, the rules and the data. The page follows them exactly, and every label, total and explanation on it uses the words in **Terms**. Where this file names a holiday_planner_SPEC_NVPRO.md section, that section applies unchanged.

All data access is read-only:
- Run SELECT queries through `safe_tenant_sql`, from chat and from the page, with the planner's `live.tenantId`.
- Leave every write tool uncalled. When the winery needs something created in Commerce7 (a tag, a code), tell the user how to do it there.
- **Call the connector one query at a time**, as holiday_planner_SPEC_NVPRO.md → **Calling the connector** says.
- **Personal data:** this module is the one place the planner shows customers by name. Show and save a gifter's Commerce7 customer id and name only. Gift recipients are counted, never named. Emails, phones and addresses stay out of the chat, the page and the saved plan.

**When the user moves on without confirming,** follow holiday_planner_SPEC_NVPRO.md's rule: use what you proposed, mark it **"Assumed — not confirmed,"** and list it in the V6 handover.

---

## Steps

### V1. Check the planner and the connection

1. Find the existing planner in this chat. It must be a New Vintage edition live artifact (`edition: 'new-vintage'`, `schemaVersion` 5 or later) with a `live.tenantId`. If it is still the manual planner, stop and ask the user to run holiday_planner_UPGRADE.md on the same artifact first. If no planner exists, offer holiday_planner_SPEC.md followed by holiday_planner_UPGRADE.md, or a direct build from holiday_planner_SPEC_NVPRO.md, then stop. Never rebuild or reset an existing planner to satisfy this check.
2. Run `discover_tenant_data_sources` with that `tenantId`, and record Commerce7's last sync.
3. Read `programSetup.corp` and `live.programRules`:
   - **`'rules'`:** corporate gifters come from those rules.
   - **`'pending'` or `'manual'`:** the planner can't recognize corporate gifting orders yet. Offer to run holiday_planner_SPEC_NVPRO.md → **When the user pastes a Setup prompt** for corporate gifting now. If the user declines, continue with individual gifters only and set `settings.types` to `['individual']`.
4. Show the user a three-line summary: the winery, Commerce7's last sync, and how corporate gifting orders are recognized ("orders with sales attribute code 'corp'," or "not set up yet").

**Done when:** you know the one tenant, the planner exists, and corporate gifting is either set up or the user chose individuals only.

### V2. Ask the setup questions

Run holiday_planner_SPEC_NVPRO.md's **Q4** for the winery's customer tags, then ask every question below in **one message**, numbered, each with its default, so the user can reply "defaults are fine." Save the answers in `modules.vip.settings`.

| # | Question | Default |
| --- | --- | --- |
| 1 | Track corporate gifters, individual gifters, or both? (Skip when V1 found no corporate rules; only individuals are tracked.) | Both |
| 2 | How much gift revenue in one season makes someone a VIP? | $2,500 (I'll show how many gifters that keeps) |
| 3 | How many past seasons should I read? (1–3) | 3 |
| 4 | Do you keep a customer tag for VIP gifters? I'll include everyone with it, whatever they spent. (List the Q4 tags whose names contain "gift," "vip" or "corp.") | None |
| 5 | How many days after a gifter's usual order date should I mark them At risk? | 7 |

**Done when:** every row has an answer or an accepted default.

### V3. Read gift history

1. Run **Q-Gifters** in **full** mode (every season in **Seasons read**).
2. Show the user:
   - one line: "{N} VIP gifters: {c} corporate, {i} individual · ${last season total} in gift revenue last season · {r} found only by the {tag} tag"
   - the top 10 by last season's gift revenue: name, type, last season, this season, status
   - status counts as of today, using **Rules**
   - one data line: "{p}% of individual gift orders had a ship-to address, so recipient counts cover those."
3. When the count is under 5 or over 100, propose a minimum that keeps 15–60 gifters (rounded to $500) and ask once.

**Done when:** Q-Gifters returned `resultMode: complete`, and the user has seen the summary and confirmed the minimum (or it's marked assumed).

### V4. Build the tab

Start from the latest whole plan. Apply holiday_planner_UPGRADE.md → **U2. Start from the latest plan**'s whole-plan/CSV merge checks for both upgraded and direct-build planners; the CSV alone is not the plan. Then follow holiday_planner_SPEC_NVPRO.md → **When the user asks for changes in chat**. Build the tab to **Page**, **Rules** and **Look**, make every change in **Planner changes**, and save the V3 results in `modules.vip` (see **Saving**). Update the same artifact in place and raise `planRevision` by 1.

**Done when:** the VIP gifters tab has every part and state in **Page**, every item in **Planner changes** is in place, the tab opens with the winery's gifters and the planner's synced stamp, and holiday_planner_SPEC_NVPRO.md's step 7 checks still match.

### V5. Check the numbers

Run checks 1–12 on a fresh copy of **Example gifters**, with today set to **Nov 16, 2026**, so no check inherits another's edits and the winery's own data stays untouched. Use the page's own code where you can run it; otherwise trace the calculation to the code. Confirm all of the following:

1. **Headline:** "$52,900 of last season's gifting hasn't come back yet." with "4 gifters had ordered by this point last season and haven't ordered this season. 3 more usually order later."
2. **Totals:** At risk $52,900 · 4 gifters; Expected later $33,800 · 3 gifters; Ordered this season $77,400 · 4 gifters · 1 new; Last season $151,400 · 10 gifters.
3. **Groups,** in this order: At risk (Calder & Finch LLP, Summit Ridge Partners, Lumen Advisory, Marisol Vega); Expected later (Northgate Realty, Bayview Orthopedics, Priya Natarajan); Ordered this season (Harborline Capital, Meridian Health Partners, Oakmont Builders, Tom Ellery). The Ordered group line reads "$77,400 so far, against $64,700 from the same gifters last season."
4. **Usual dates:** Calder & Finch "By Nov 3" and "13 days past usual"; Summit Ridge "By Oct 27", "20 days past usual"; Northgate "By Nov 24", "Due in 8 days"; Bayview "By Dec 1", "Due in 15 days".
5. **Ordered rows:** Harborline "+7% vs last season"; Meridian "−13% vs last season"; Oakmont "New" with "New gifter this season" and "—" for Last season and Usually orders.
6. **Reconciliation:** "Corporate gifting · $84,600 actual" and "Your 8 corporate VIPs account for $72,250 of it. The other $12,350 came from smaller corporate orders."
7. **Filters:** Corporate shows 8 rows and the totals for corporate only (At risk $49,000 · 3 gifters). Status At risk shows 4 rows. "Over $10,000" leaves 6 gifters and keeps Oakmont (best season $12,300).
8. **A new order:** adding a gift order for Calder & Finch (Nov 16, 2026, $25,000, corporate) and raising the corporate gifting actual by $25,000 moves Calder & Finch to Ordered with "−7% vs last season". At risk becomes $26,000 · 3 gifters, Ordered $102,400 · 5 gifters, the headline "$26,000 of last season's gifting hasn't come back yet.", and the reconciliation "$97,250 of $109,600."
9. **Drawer:** clicking Calder & Finch opens the drawer with the season bars ($22,100 · $24,300 · $26,900 · Not yet), both holiday 2025 gift orders, and the usual-date sentence. Typing an owner, a last-contacted date and a note survives a reload and a Refresh.
10. **Outreach:** **Add to an outreach activity** on Calder & Finch creates one Direct outreach activity, "At-risk VIP gifters," in the week of Nov 16, program corporate gifting, audience label "At-risk VIP gifters (1)", list size override 1. Pressing it on Summit Ridge updates the same activity to "(2)" and list size 2.
11. **Layout:** at 1440px all eight columns show. At 720px (the artifact pane) the table drops Last gift order and Recipients into the drawer, and the page never scrolls sideways. At 390px each gifter is a two-line card.
12. **Privacy:** the saved plan's `modules.vip` holds ids, names, numbers, dates and the team's typed follow-up only; searching it for "@", for phone patterns and for "address" finds nothing.

Then run these checks on the winery's own data:
- Σ this-season gift revenue of corporate VIPs ≤ the Program template's synced corporate gifting actual for the same days (Q-Gifters counts paid, non-refund orders, so the program figure may be lower only by refunds; report any gap over 2%).
- Every gifter's season figures match a second Q-Gifters run in **season** mode for this season.
- Every V3 top-10 gifter appears on the page with the same figures.

Tell the user, in one sentence each, how much gift revenue is at risk and which gifter is the largest at risk.

**Done when:** all twelve example checks and the three live checks match, and any mismatch is fixed in the code.

### V6. Hand over

Tell the user, in six lines or fewer:
- what At risk, Expected later, Ordered and New mean, in one line
- that the tab refreshes with the planner's Refresh, and the corporate gifting actual on the Plan tab moves with it
- how to use the drawer's follow-up fields and **Add to an outreach activity**
- that "Re-read gift history" in this chat changes the minimum, seasons or tag
- that the tab shows customer names, so share the page only with people who should see them
- every item marked "Assumed — not confirmed," if any

**Done when:** the user has the tab and those notes.

---

## Later requests

These run only when the user asks, after the tab is built.

- **"Refresh VIP gifters"** (chat sync, or a page that can't reach the connector): run **Q-Gifters** in **season** mode, merge the complete result into the latest saved roster using **Saving → Loading** below, replace the season-0 figures and `modules.vip.syncedAt`, keep `planRevision` as it is, and say how at-risk revenue moved, in one sentence. This refresh changes the VIP module only; it does not claim that the main program actuals or other sections refreshed. Use the full planner Refresh to update both the program actuals and VIP figures.
- **"Re-read gift history"** (new minimum, seasons, tag or types): re-run V2's changed questions and V3, replace `modules.vip.gifters` figures (keeping each gifter's `followUp`, `hidden` and `outreach`), and raise `planRevision` by 1.
- **"Draft outreach for at-risk gifters":** propose one Direct outreach activity per owner (or one for all when no owners are typed) covering the at-risk gifters, with talking points from each gifter's history. Add them on the user's yes, and raise `planRevision` by 1.
- **Any other edit to the tab:** follow holiday_planner_SPEC_NVPRO.md → **When the user asks for changes in chat**, keeping `modules.vip` figures and follow-up as they are.

---

## Terms

- **Season**, **plan's season**, **last season**, **today** and **replay**: as holiday_planner_SPEC_NVPRO.md → **Terms** defines them.
- **Seasons read**: the plan's season (season 0) and up to three seasons before it (1 = last season).
- **Gift order**: a paid, non-refund, non-Club order with `sub_total > 0` and a customer that is either a **corporate gift order** (it matches the planner's corporate gifting rules) or an **individual gift order** (it has a gift message or ships to an address other than the billing address).
- **Gift revenue**: Σ `sub_total` of a gifter's gift orders of their type in a season.
- **Corporate gifter**: a customer with a corporate gift order in any season read.
- **Individual gifter**: a customer who isn't a corporate gifter, with at least 2 individual gift orders in one season read.
- **VIP gifter**: a corporate or individual gifter whose gift revenue reached the minimum in at least one season read, or who carries the VIP tag. Shown as "VIP" only in the tab name; rows say "gifter."
- **Usual date**: the latest day-of-season a gifter placed their first gift order, across the past seasons they ordered in, moved onto the plan's season ("By Nov 3").
- **Grace days**: days after the usual date before a gifter turns At risk (default 7).
- **Status**: **Ordered**, **New**, **Expected later**, **At risk** or **Lapsed** (see **Rules**).
- **At-risk revenue**: Σ last season's gift revenue of At risk gifters.
- **Recipients**: distinct ship-to addresses across a gifter's gift orders in a season. Counted, never listed.
- **Follow-up**: the team's typed owner, last-contacted date and note for a gifter.

---

## Page

The **VIP gifters** tab sits after Plan (and after Gift packs, when present), before How to use. It uses the planner's season bar, Refresh and synced stamp unchanged. Top to bottom: headline, totals card, toolbar, roster, source line.

### Headline

- **At-risk revenue above $0:** an alert icon and "${at-risk revenue} of last season's gifting hasn't come back yet." in `status-warning` (the Verdict style), then a `body-sm` line: "{n} gifters had ordered by this point last season and haven't ordered this season. {m} more usually order later." Drop the second sentence when m is 0.
- **None at risk:** a check icon and "Every gifter due by now has ordered." in `status-success`, then "{m} more usually order later." or "That's everyone from last season."
- **Before the season:** "{n} VIP gifters gave ${last season} last season. The first usually orders by {earliest usual date}." in `text-primary`.

### Totals card

One card, five cells left to right; under 1100px the four totals wrap to two rows and the reconciliation drops below them.

| Cell | Eyebrow | Figure | Caption |
| --- | --- | --- | --- |
| 1 | AT RISK (`status-warning`) | At-risk revenue | "{n} gifters · last season's value" |
| 2 | EXPECTED LATER | Σ last season of Expected later gifters | "{n} gifters · not due yet" |
| 3 | ORDERED THIS SEASON (`status-success`) | Σ this season of Ordered and New gifters | "{n} gifters · {k} new" |
| 4 | LAST SEASON | Σ last season of every VIP gifter | "{n} gifters · holiday {year}" |
| 5 | Reconciliation | see below | |

**Reconciliation** (shown when corporate gifters are tracked): the corporate gifting program color dot and "Corporate gifting · ${actual} actual" (the program's effective actual from the planner), then "Your {n} corporate VIPs account for ${x} of it. The other ${actual − x} came from smaller corporate orders." When x exceeds the actual: "Your corporate VIPs' orders add up to ${x − actual} more than the corporate gifting actual. Check its override, or press Refresh." A **See it on the plan** link opens the Plan tab with the program filter set to corporate gifting.

Totals follow the toolbar's filters.

### Toolbar

- **Type:** a segmented control, "All {n} | Corporate {c} | Individual {i}". Hidden when only one type is tracked.
- **Status:** a checkbox menu (At risk, Expected later, Ordered, New, Lapsed), all on except Lapsed.
- **Minimum:** a menu of "Over ${settings minimum}" (default) and the steps $5,000, $10,000 and $25,000 above it, filtering on a gifter's best season.
- At the right, in `text-tertiary`: "Corporate = your corporate gifting rules · Individual = 2+ gift orders in a season."
- Filters are viewer preferences, saved under `holiday-planner.vip-view` (with try/catch), never in the plan.

### Roster

A card holding a table with a sticky head. Columns: Gifter (name, with "Ordered {k} of the last {s} seasons" or "First gift order this season" under it), Type (a chip), Last season, This season, Usually orders, Last gift order, Recipients (last season's, or this season's for New), Status (icon and word, with the reason under it). Figures are right-aligned with tabular figures; "—" marks none.

Rows sit in status groups, each opened by a full-width group row:

| Group | Group row | Reason under the status | Order within |
| --- | --- | --- | --- |
| At risk | `status-warning-bg`: "At risk · {n}" and "Past their usual order date with no gift order yet · ${sum} last season" | "{d} days past usual" | Last season, high to low |
| Expected later | `surface-subtle`: "Expected later · {n}" and "Not ordered yet, but their usual date is still ahead · ${sum} last season" | "Due in {d} days", or "Due now" inside the grace days | Usual date, soonest first |
| Ordered this season | `status-success-bg`: "Ordered this season · {n}" and "${this season} so far, against ${their last season} from the same gifters last season" | "+7% vs last season" (Ordered) or "New gifter this season" (New) | This season, high to low |
| Lapsed | collapsed by default: "Lapsed · {n} · ordered before last season, not since" | "Last ordered holiday {year}" | Best season, high to low |

Clicking a row opens the **Gifter drawer**. An empty group is omitted.

### Gifter drawer

A drawer over the right of the tab (a bottom sheet on a phone), like the planner's activity drawer. Esc closes it.

- **Head:** the name (Verdict style), the type chip, the status pill ("At risk · 13 days past usual"), and **Open in Commerce7** (see **Data facts**).
- **Gift revenue by season:** one bar per season read, oldest first, labeled with its year and amount; the plan's season reads "Not yet" over an empty track until it has revenue. Bars fill with the corporate gifting color for corporate gifters and `text-secondary` for individuals.
- **Last season's gift orders:** one line per order, "Nov 4 · Corporate order · 128 recipients" with the amount right-aligned, up to 10, then "+{n} more." Then the usual-date sentence: "Usually orders by {date}: their latest first gift order in the last {s} seasons, moved to this year's calendar. They turn At risk {g} days after that."
- **Follow-up · typed by your team:** Owner (text), Last contacted (date), Note (multi-line). Edits apply as the user types. A sync never changes them.
- **Actions:** **Add to an outreach activity** (primary; see **Rules**), and **Hide from this list** (two-step, like Delete), which moves the gifter to a "Hidden · {n}" line at the bottom of the roster with **Show**.

### Source line

Under the roster in `text-tertiary`: "From Commerce7 through New Vintage · gift orders from holiday {oldest}–{plan's season} · Corporate uses your corporate gifting rules; Individual counts orders with a gift message or a different ship-to address. Recipients are counted, never listed." At the right, two links: **How status works** (a popover with the Status rules in plain words) and **Change who counts as a VIP** (a dialog: "To change the minimum, seasons or tag, say 'Re-read gift history' in the chat where you built this planner." with **Got it**).

### States

| State | Treatment |
| --- | --- |
| Example view | The planner's example banner; the tab shows **Example gifters** with sync off |
| Corporate not set up | Type shows Individual only, and a one-line note above the totals: "Corporate gifting isn't set up yet, so corporate gifters aren't tracked. Press Setup on the Plan tab." |
| Before the season | The before-the-season headline; every returning gifter is Expected later |
| No gifters | "No gifters reached ${minimum} in the seasons read. Say 'Re-read gift history' in the chat to lower it." |
| Filters hide everything | "No gifters match these filters · Show all" |
| Replay | Today is the as-of date everywhere (holiday_planner_SPEC_NVPRO.md → **Replay**) |
| Sync states | The planner's **Sync states**, with VIP gifters as its own section |

---

## Planner changes

1. **Tab:** add **VIP gifters** after Plan (after Gift packs when present), before How to use.
2. **Program legend:** on the corporate gifting row, a small **VIP gifters** link after the name, opening the tab.
3. **What syncs:** add a row, "VIP gifters' this-season figures · **Q-Gifters**, season mode · every sync, after the Program template." Last season and earlier change only on Re-read.
4. **Sync details dialog:** add the VIP gifters row, and under **How programs are assigned** add: "VIP gifters: corporate gifters are customers with corporate gifting orders; individual gifters placed 2 or more gift orders in a season. Gift recipients are counted, never listed."
5. **More menu:** add **Download VIP list** (CSV, through the download capability): Name, Commerce7 id, Type, each season's gift revenue, Usually orders, Last gift order, Recipients, Status, Owner, Last contacted, Note. Hidden gifters are left out.
6. **How to use:** add a section, "VIP gifters": "The VIP gifters tab lists your biggest corporate and individual gift-givers and whether each has ordered yet this season. A gifter turns At risk a week after the date they usually order by. Open a gifter to note who owns the relationship and when you last reached out, or add them to an outreach activity so the calls show on the calendar."
7. **Activity fields:** add `vipGifterIds: [customerId]`, empty by default, on Direct outreach activities.
8. **Example view:** load **Example gifters** into the tab.

---

## Rules

- **Seasons read** use holiday_planner_SPEC_NVPRO.md's season (13 weeks from the first Monday on or after Oct 1). Season 0 runs to `:order_end`; earlier seasons are complete.
- **Day of season** = (local paid date − season start) in whole days. **Usual date** = the plan's season start + the largest first-gift-order day of season across seasons 1–3 the gifter ordered in. A New gifter has none.
- **Status,** checked in order:
  1. **Ordered:** gift revenue in season 0 > 0, and gift orders in at least one earlier season.
  2. **New:** gift revenue in season 0 > 0, and none earlier.
  3. **Lapsed:** no gift order in season 0 or season 1.
  4. **At risk:** today > usual date + grace days.
  5. **Expected later:** otherwise. "Due now" when today is past the usual date but inside the grace days.
- **Vs last season** = season 0 ÷ season 1 − 1, as a whole % with + or −. With no season 1, "Back after a season off."
- **Corporate VIPs' season** = Σ season-0 gift revenue of corporate VIP gifters, hidden ones included (they still count toward the program).
- **Totals and groups** leave out hidden gifters.
- **Minimum** applies to a gifter's best season in seasons 0–3. Tagged gifters always qualify.
- **Add to an outreach activity:** find the Direct outreach activity this module created (`modules.vip.outreachActivityId`) whose weeks include today. With none, create one: title "At-risk VIP gifters," lane Direct outreach, start week = today's season week, 1 week, program corporate gifting (or other when only individuals are tracked). Add the gifter's id to `vipGifterIds`, set the audience to the label "At-risk VIP gifters ({n})" and the list size override to n. The plan's totals follow from the activity as usual.
- **Formatting:** holiday_planner_SPEC_NVPRO.md → **Rules**, Formatting. Dates in the table read "Nov 3" for the plan's season and "Nov 4, 2025" for other years.

---

## Saving

- **Where:** inside the planner's plan object, under `modules.vip`, so the planner's own `schemaVersion` stays as it is and modules can be added in any order. An older plan without `modules` gains `modules: {}`.
- **Shape:**

  `modules.vip = { v: 1, source: 'tenant' | 'example', tenantId, syncedAt, settings: { types, minimumUsd, seasonsBack, tagId, graceDays }, seasons: [{ index, label, start, end }], gifters: [{ id, name, type: 'corp' | 'individual', tagged, bySeason: { [index]: { revenue, orders, firstDay, lastDate, recipients } }, followUp: { owner, contactedOn, note }, hidden, outreach }], outreachActivityId }`

  `id` is the Commerce7 customer id; `name` is "{first name} {last name}." Dates are ISO; `firstDay` is a day of season; money is whole dollars.
- **Each sync** replaces `bySeason[0]` for every gifter, adds gifters who newly qualify, and sets `syncedAt`. **Re-read** replaces every `bySeason` and `seasons`. `followUp`, `hidden` and `outreach` change only when the team changes them.
- **Loading:** extend holiday_planner_SPEC_NVPRO.md → **Saving** without replacing its plan-revision rule. When `planRevision` is unchanged and the built-in `modules.vip.syncedAt` is newer than the saved module timestamp, merge the VIP result independently of `live.syncedAt`, including when the main planner timestamp is unchanged. Match gifters by customer id; replace only `bySeason[0]` with the new current-season figures (zero figures for an existing gifter absent from a complete season result), add newly qualifying gifters, and update the module timestamp. Preserve every existing gifter's past-season history, `followUp`, `hidden` and `outreach`, as well as module settings, outreach activity links, all plan edits and every override. Save the merged whole plan. Never advance `live.syncedAt` to imply sections that were not refreshed; ordinary planner-sync timestamp merging still follows the base spec.
- **When saving isn't available:** holiday_planner_SPEC_NVPRO.md → **Saving**.

---

## Data

### Data facts

| Topic | Fact |
| --- | --- |
| Names | `customers.first_name`, `last_name`. There is no company field on customers, and `orders.bill_to->>'company'` was empty on every holiday 2025 gift order at a large tenant, so a corporate gifter shows as the person who buys. |
| Customer link | `orders.customer_id` → `customers.id`. About 0.5% of orders have no customer; they can't be gifters. |
| Corporate rules | Build `:corp_predicate` from `live.programRules` exactly as holiday_planner_SPEC_NVPRO.md's Program template builds its corporate gifting branch, so the tab and the program count the same orders. |
| Corporate Orders tool | Many tenants never use `purchase_type = 'Corporate Order'` (a large tenant had none in holiday 2025). The confirmed rules decide. |
| Gift signal | Imported history often lacks gift messages; the address test still works. |
| Recipients | Count distinct `lower(btrim(ship_to->>'address')) || '|' || lower(btrim(ship_to->>'city'))` over orders with a ship-to address. Pickup and carry-out orders add none. |
| Commerce7 link | UNVERIFIED: the admin URL pattern for a customer profile. Confirm it before building; until then, omit the link. |
| Limits | holiday_planner_SPEC_NVPRO.md → **Data facts**, Connector limits. Give Q-Gifters `statementTimeoutMs` 60000. |

### Queries

Fill these before each call; keep `'__TENANT_ID__'` literally.
- `:s0_start` … `:s3_start` and `:s1_end` … `:s3_end`: each season read's first Monday and that day + 91; `:s0_end` is `:order_end`
- `:corp_predicate`: a boolean SQL expression on `o`, or `false` when corporate gifters aren't tracked
- `:minimum_cents`, `:tag_id` (or `NULL`)
- `:known_corp_ids`: `ARRAY['id1','id2']::text[]` holding the saved corporate gifters in season mode, so a corporate gifter with no corporate order yet this season keeps its type; `ARRAY[]::text[]` in full mode
- **Mode:** **full** reads every season; **season** reads season 0 only: drop the earlier seasons' CASE branches and use `:s0_start` as the WHERE lower bound

**Q-Gifters.** DRAFT, not yet verified on a tenant.

```sql
WITH o AS (
  SELECT o.customer_id, o.id, o.sub_total, o.order_paid_date, o.ship_to,
         CASE
           WHEN o.order_paid_date >= timestamptz ':s0_start 00:00:00 America/Los_Angeles' AND o.order_paid_date < timestamptz ':s0_end 00:00:00 America/Los_Angeles' THEN 0
           WHEN o.order_paid_date >= timestamptz ':s1_start 00:00:00 America/Los_Angeles' AND o.order_paid_date < timestamptz ':s1_end 00:00:00 America/Los_Angeles' THEN 1
           WHEN o.order_paid_date >= timestamptz ':s2_start 00:00:00 America/Los_Angeles' AND o.order_paid_date < timestamptz ':s2_end 00:00:00 America/Los_Angeles' THEN 2
           WHEN o.order_paid_date >= timestamptz ':s3_start 00:00:00 America/Los_Angeles' AND o.order_paid_date < timestamptz ':s3_end 00:00:00 America/Los_Angeles' THEN 3
         END AS season,
         (:corp_predicate) AS is_corp,
         ((o.gift_message IS NOT NULL AND btrim(o.gift_message) <> '')
          OR (coalesce(o.ship_to->>'address','') <> '' AND coalesce(o.bill_to->>'address','') <> ''
              AND lower(btrim(o.ship_to->>'address')) <> lower(btrim(o.bill_to->>'address')))) AS is_gift
  FROM orders o
  WHERE o.tenant_id = '__TENANT_ID__' AND o.customer_id IS NOT NULL AND o.sub_total > 0
    AND o.purchase_type IS DISTINCT FROM 'Refund' AND o.channel IS DISTINCT FROM 'Club'
    AND o.order_paid_date >= timestamptz ':s3_start 00:00:00 America/Los_Angeles'
    AND o.order_paid_date <  timestamptz ':s0_end 00:00:00 America/Los_Angeles'
), corp AS (
  SELECT DISTINCT customer_id FROM o WHERE season IS NOT NULL AND is_corp
), g AS (
  SELECT o.*, (c.customer_id IS NOT NULL OR o.customer_id = ANY(:known_corp_ids)) AS corp_gifter
  FROM o LEFT JOIN corp c ON c.customer_id = o.customer_id
  WHERE o.season IS NOT NULL
), gg AS (
  SELECT * FROM g WHERE g.is_corp OR (NOT g.corp_gifter AND g.is_gift)
), s AS (
  SELECT gg.customer_id, bool_or(gg.corp_gifter) AS corp_gifter, gg.season,
         sum(gg.sub_total) AS revenue_cents, count(*) AS orders,
         min(gg.order_paid_date) AS first_paid, max(gg.order_paid_date) AS last_paid,
         count(DISTINCT lower(btrim(gg.ship_to->>'address')) || '|' || lower(btrim(coalesce(gg.ship_to->>'city',''))))
           FILTER (WHERE coalesce(gg.ship_to->>'address','') <> '') AS recipients
  FROM gg GROUP BY gg.customer_id, gg.season
), tagged AS (
  SELECT t.object_id AS customer_id FROM object_tags t
  WHERE t.tenant_id = '__TENANT_ID__' AND t.tag_id = :tag_id
), q AS (
  SELECT s.customer_id FROM s
  GROUP BY s.customer_id
  HAVING (bool_or(s.corp_gifter) OR max(s.orders) FILTER (WHERE NOT s.corp_gifter) >= 2)
     AND (max(s.revenue_cents) >= :minimum_cents OR s.customer_id IN (SELECT customer_id FROM tagged))
  ORDER BY max(s.revenue_cents) DESC LIMIT 150
)
SELECT s.customer_id, btrim(coalesce(cu.first_name,'') || ' ' || coalesce(cu.last_name,'')) AS name,
       s.corp_gifter, (s.customer_id IN (SELECT customer_id FROM tagged)) AS tagged, s.season,
       round(s.revenue_cents / 100.0, 0) AS revenue_usd, s.orders,
       (s.first_paid AT TIME ZONE 'America/Los_Angeles')::date AS first_date,
       (s.last_paid AT TIME ZONE 'America/Los_Angeles')::date AS last_date, s.recipients
FROM s JOIN q ON q.customer_id = s.customer_id
JOIN customers cu ON cu.tenant_id = '__TENANT_ID__' AND cu.id = s.customer_id
ORDER BY s.customer_id, s.season
```

Output: one row per gifter and season with gift orders. The page computes day of season from `first_date`. **Before relying on it,** confirm with `describe_tenant_sql_sources`: the customer-tag table and its columns (written here as `object_tags`), and the `ship_to` key for city. In **season** mode, the qualify step sees season 0 only, so merge its gifters into the saved roster rather than replacing it.

**Last season's gift orders (drawer).** DRAFT: the same `o` and `g` blocks for season 1 and one `customer_id`, returning paid date, amount and recipients per order, at most 11 rows. The page runs it when a drawer opens, one call at a time.

---

## Look

The tab uses holiday_planner_SPEC_NVPRO.md → **Look** unchanged, in both looks. It adds:

- **Status colors:** At risk uses `status-warning`; Ordered and New use `status-success`; Expected later uses `text-secondary`; Lapsed uses `text-tertiary`. Each always comes with its icon (alert circle, check circle, clock, minus circle) and its word. Group rows use the matching `-bg` token, and `surface-subtle` for Expected later and Lapsed.
- **Type chips:** Corporate takes the planner's chip style in the corporate gifting color. Individual takes the neutral badge (`surface-panel` ground, a 3px `text-tertiary` left border).
- **Totals card:** the planner's card, cells split by 1px `border-subtle` lines, eyebrows in `label-caps`, figures in 28px `numeric`.
- **Buttons:** primary for **Add to an outreach activity**; text links for **See it on the plan**, **How status works** and **Change who counts as a VIP**.
- **Access:** rows are buttons that open the drawer; the headline and totals sit in an `aria-live="polite"` region; the follow-up fields have visible labels.

---

## Example gifters

Shown in the example view and used by V5. Season starts: 2023 Oct 2, 2024 Oct 7, 2025 Oct 6, 2026 Oct 5. Minimum $2,500, 3 seasons, grace 7 days, both types, no tag. The planner example's corporate gifting actual is $84,600.

Each season cell is "gift revenue (first gift order date)". Last gift order equals the first gift order date except where noted.

| Gifter | Type | 2023 | 2024 | 2025 | 2026 | Recipients 2025 |
| --- | --- | --- | --- | --- | --- | --- |
| Calder & Finch LLP | Corporate | $22,100 (Oct 30) | $24,300 (Nov 1) | $26,900 (Nov 4; last Dec 1) | — | 140 |
| Summit Ridge Partners | Corporate | — | $13,000 (Oct 25) | $14,200 (Oct 28) | — | 61 |
| Lumen Advisory | Corporate | $6,800 (Nov 2) | $7,400 (Nov 5) | $7,900 (Nov 7) | — | 32 |
| Marisol Vega | Individual | $3,100 (Oct 30) | — | $3,900 (Nov 3) | — | 15 |
| Northgate Realty | Corporate | $15,900 (Nov 20) | $16,700 (Nov 25) | $17,800 (Nov 25) | — | 88 |
| Bayview Orthopedics | Corporate | — | $8,900 (Dec 1) | $9,600 (Dec 2) | — | 40 |
| Priya Natarajan | Individual | $5,600 (Dec 4) | $6,100 (Dec 9) | $6,400 (Dec 8) | — | 24 |
| Harborline Capital | Corporate | $33,000 (Oct 17) | $35,500 (Oct 21) | $38,400 (Oct 21) | $41,200 (Oct 22) | 212 |
| Meridian Health Partners | Corporate | $19,900 (Nov 6) | $20,800 (Nov 8) | $21,500 (Nov 11) | $18,750 (Nov 9) | 96 |
| Oakmont Builders | Corporate | — | — | — | $12,300 (Nov 2) | 55 (2026) |
| Tom Ellery | Individual | $4,200 (Nov 10) | $4,500 (Nov 13) | $4,800 (Nov 15) | $5,150 (Nov 12) | 18 |

Calder & Finch's holiday 2025 gift orders: Nov 4, $24,960, 128 recipients; Dec 1, $1,940, 12 recipients. Each other gifter has one gift order per season.

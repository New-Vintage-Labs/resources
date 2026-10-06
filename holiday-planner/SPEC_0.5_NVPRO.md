# Holiday Campaign Planner, New Vintage edition — build spec v0.5

*For the winery: in Claude, turn on the New Vintage connector (a New Vintage Pro account), then upload this file to a new chat and say "Build my holiday planner." Have your holiday calendar handy, or let Claude draft one from last season's campaigns.*

You are building a one-page holiday campaign planner for a winery's DTC team, covering **October–December only**, as a **live artifact**: a Claude artifact that reads the winery's Commerce7, email and SMS data through the New Vintage connector and keeps its plan in persistent storage. It answers one question: **does our October–December plan reach our revenue goal, and how is it tracking?** The team writes the plan; the results fill in from real orders (see **What syncs**). It is a starting point the team makes their own, on the page itself or by asking you in chat.

Work through the **Steps** in order. Each step ends on a **Done when** line; finish it before starting the next. The sections after the Steps (**Terms**, **Page**, **Sync**, **Replay**, **Example view**, **Rules**, **Queries** and **Look**) define the page, the sync, the rules and the data. The page follows them exactly, and every label, total and explanation on it uses the words in **Terms**.

All data access is read-only:
- Run SELECT queries through `safe_tenant_sql`, from chat and from the page.
- Leave every write tool uncalled.
- Keep customer names, emails and phones out of the chat and out of the page. Every query in **Queries** returns aggregates only.
- **Call the connector one query at a time**, from chat, from subagents and from the page. Parallel calls make it stop responding. When a response carries a `conversation_id`, send it with the next call.

**When the user moves on without confirming.** If the user skips a confirmation (answers something else, or says "keep going"), don't ask again. Use what you proposed, mark each unconfirmed item **"Assumed — not confirmed"** in the plan, and list those items in the step 8 handover so the user can correct them. The one exception is the tenant: without it, stop.

---

## Steps

### 1. Check the connection

1. Run `discover_tenant_data_sources`. If it reaches several tenants, ask which winery this is, and record its exact `tenantId`. Pass that `tenantId` on every later call, from chat and from the page.
2. Record Commerce7's last sync, and which email platforms (Klaviyo, Mailchimp) and SMS platforms (RedChirp) are synced.
3. Run **Q2** for which platforms sent campaigns in each of the last three seasons. Then, one season at a time, run **Q3** for the two most recent seasons to see how complete each platform's recipient history is.
4. Run **Q1** for the two most recent seasons.
5. Classify each of the two seasons:
   - **Replayable:** Q1 finds orders and Q2 finds campaigns.
   - **Measurable:** Q3 finds complete campaigns with clicks. A platform that is complete only from a date (`complete_tail_from`) is measurable from that date. Measure from complete campaigns only.
   - **Click tracking gap:** an SMS platform with texts that carried links (`sms_with_link` > 0) but no clicks. Read it as missing data, never as 0% conversion.
6. Show the user a five-line summary: the winery, Commerce7's last sync, each email and SMS platform with the dates its history is complete, each of the two seasons' DTC revenue, and which seasons can be replayed and measured.

If the connector is missing, or the account lacks Pro access, tell the user to turn on the New Vintage connector in Claude's settings, then stop.

**Done when:** the user has seen the summary and you know the one `tenantId`.

### 2. Ask the setup questions

Use the words below exactly, so a first-time user never meets a term without its meaning. Write every season as its year ("holiday 2025"), filling each `{…}` with its value. A question tool allows at most four options per question; every question below fits.

**If you have a multiple-choice question tool** (AskUserQuestion or ask_user_input), use it:

1. **First call, one question.** Header "Start." Question: "How would you like to start?"

   | Option | Description |
   | --- | --- |
   | Plan this season (recommended) | "Build your October–December {season} plan. Actual revenue and each email and SMS result fill in from your systems as the season runs." |
   | Replay holiday {most recent season} | "Rebuild holiday {most recent season} from its real campaigns and see the planner as of a day you pick, with real results up to that day." |
   | See the example first | "A finished example winery with typed figures, to explore before building yours." |

   Offer Replay only when the most recent season is replayable. If they pick the example, go to step 6 and build the page with no plan, opening on the example view (see **Example view**).
2. **Second call, two questions.** Ask the **a** that matches their start.

   **a. This season: header "Plan."** Question: "Where should your plan come from?"

   | Option | Description |
   | --- | --- |
   | My calendar | "Paste or upload it next, and I'll map each item onto the planner" |
   | Draft from last season | "I'll move holiday {last season}'s emails, texts and club shipments to this year's weeks for you to edit" |
   | Start empty | "You'll add activities on the page" |

   **a. Replay: header "As of."** Question: "Which day should the replay show? The planner acts as if it's that day, with real results up to it."

   | Option | Description |
   | --- | --- |
   | {week 9 Monday} (recommended) | "After Black Friday and Cyber Monday, with December still ahead" |
   | {week 6 Monday} | "Early in the season, before the holiday rush" |
   | Season end | "The whole season, finished" |

   Label each with the replayed year's date: for holiday 2025, Dec 1 and Nov 10. Season end is the day after week 13.

   **b. Header "Cutoffs."** Question: "Use the usual holiday deadlines? They're marked on the calendar so you can plan around them."

   | Option | Description |
   | --- | --- |
   | Use the usual dates (recommended) | "Corporate gift orders by Nov 6, ground shipping Dec 15, 2-day shipping Dec 21" |
   | I'll give mine | "I'll ask for your three dates" |

3. **Then send one plain message** asking for:
   - confirmation of the winery name, as the tenant shows it
   - the revenue goal ("your October–December DTC revenue target, including tasting room and phone sales"), offering the **Goal default** as a number
   - their calendar, pasted or uploaded, if they chose My calendar
   - results they already have for events and Direct outreach, which stay typed
   - any custom cutoffs

**Otherwise** (no question tool), ask all of the above in one numbered message, using the same questions, descriptions and defaults, so the user can reply "defaults are fine."

**Goal default:** this season, last season's actual revenue + 5%, rounded to the nearest $10,000. In a replay, the actual revenue of the season before the replayed one + 5%, rounded the same way; with no such season, the replayed season's actual revenue, rounded. With no last season at all, ask for the goal with no default. The goal covers all DTC revenue, so the plan includes the tasting room and Other baselines (see **Baseline**).

**Done when:** either the user chose the example, or every question above has an answer or an accepted default and you have their calendar if they chose My calendar.

### 3. Measure last season

With last season's dates:
1. Run **Q1** for last year's total and weekly figures.
2. Run **Q5** on last season and **group its campaigns into offers** (see **Grouping sends into offers**). Leave out tests and card-decline texts. Keep the groups; step 5 reuses them when drafting from last season.
3. Run **Q6** with the offers built from Q3's complete campaigns, for the Email and SMS lane defaults:
   - conversion = matched orders ÷ person-offers (distinct people who received each offer, summed over offers), with two decimals
   - average order = matched revenue ÷ matched orders
4. Run **Q7** on last season for the Club shipment default:
   - conversion (bill rate) = billed shipments ÷ scheduled shipments
   - average order = billed revenue ÷ billed shipments

Show one table with a row per lane: the measured value (labeled "holiday {last season} actual") beside the New Vintage median from **Lane defaults**. Direct outreach and Events & tasting room show the median only, as does any lane with no complete campaigns last season. The user chooses, per row, the measured value or the median. Wineries vary widely (up to 30× between wineries), so recommend the measured value whenever there is one.

**Done when:** every lane has a lane default the user chose, labeled with its source.

### 4. Agree how corporate gifting and private client orders are marked

Wineries mark these orders in different ways, so use only the rules the user confirms. The same procedure runs later from the page's **Setup** button (see **When the user pastes a Setup prompt**).

1. **Corporate gifting default.** Orders made with Commerce7's Corporate Orders tool carry `purchase_type = 'Corporate Order'`. (The tool isn't available on C7 Lite.) Run the **Program template** on last season with that one rule. If it finds orders, propose it as the first corporate gifting rule.
2. **Channels vary, so don't assume one.** Corporate gift orders can arrive on any channel, depending on how the winery runs its business:
   - **Web:** the buyer orders on the website, sometimes through a corporate landing page.
   - **Inbound:** the team keys the order in by hand after an email or call, which is common for white-glove corporate service.
   - **A partner:** a third-party gifting partner creates batch orders, which may arrive on Web or Inbound with a partner tag, code or customer.
   - **Club:** occasionally the Corporate Orders tool books on the Club channel.

   Ask whether a partner sends corporate orders, and match rules on every channel unless the split shows a rule catching club or tasting-room orders it shouldn't.
3. **Ask about the rest, one question per program,** allowing several answers (`multiSelect`). Header "Corporate." Question: "Besides the Corporate Orders tool, how does your team mark **corporate gifting** orders in Commerce7?" (drop "Besides the Corporate Orders tool" when step 4.1 found nothing). Then header "Private client," with the same question for **private client** orders.

   | Option | Description |
   | --- | --- |
   | A tag | "A customer tag or an order tag" |
   | A sales attribute code | "A code your team sets on the order" |
   | A promotion or coupon | "A promotion or coupon used only for these orders" |
   | Nothing else | "We don't mark them any other way" |

   For a tag, ask for its name, then check whether it is a customer tag or an order tag and look up its id. For a code or promotion, ask for its name, then look up its id.
4. **Show the split and confirm.** Run the **Program template** with the chosen rules on last season. Show the resulting split (club, ecommerce, corporate gifting, private client, tasting room, other) and ask the user to confirm. When a rule finds nothing last season, run it on the last 90 days of orders too, since history imported from another platform often lacks tags, codes and order types. Warn if a rule catches more than 30% of Other revenue, or catches club or tasting-room orders.
5. **Save the rules** as `live.programRules`, and the confirmed split, by program and by week, as last year's program figures. Rules are checked in order; the first match wins, before the channel defaults. Save each program's setup state in `programSetup`: `'rules'` when it has confirmed rules, `'pending'` when the user said it isn't marked.

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
   - `channel` (optional; narrows the program's other rules to the listed channels)
6. **Set the baseline** from the confirmed split: last year's weekly Tasting room and Other figures (see **Baseline**).

**Done when:** each program has confirmed rules, or the user has confirmed it isn't marked, the user has confirmed the split, and the baseline is set.

### 5. Map the plan, audiences and links

1. **Build the activities**, each with every field in **Activity fields**, from the plan source:

   | Plan source | Activities |
   | --- | --- |
   | My calendar | Each calendar item in October–December. Infer lane, start week and number of weeks from dates and channel words (table below). Keep the user's titles. |
   | Draft from last season | Step 3's offers from last season, each one activity in the same season week this year, titled with the offer's main campaign name (without date stamps or segment suffixes), linked to nothing yet. Program is ecommerce unless the name says corporate or private client. From last season's **Q7**, add one Club activity that bills a shipment for each week a shipment was processed, with its scheduled count as list size. |
   | Replay | The same as Draft from last season, run on the replayed season itself. Each Email and SMS activity is linked to all of its offer's campaigns. |
   | Start empty | None. The page opens on the Empty plan state. |

   | Words | Lane |
   | --- | --- |
   | "eblast", "newsletter" | Email |
   | "text" | SMS |
   | "calls", "mailer", "catalogue" | Direct outreach |
   | "pickup party", "tasting", "open house" | Events & tasting room |

2. **Propose audiences.** Run **Q4** once for the winery's segments, clubs and customer tags. For each activity's audience, propose a match, then size every proposed audience in one **Q9** call. Never preselect a tag Q4 flags as a likely exclusion or test list. When the winery sends to lists defined in Klaviyo or Mailchimp that no segment, club or tag matches, suggest connecting Claude's Klaviyo or Mailchimp connector to bring in the exact list. That is outside this build, so keep the audience as a label with its sent count meanwhile.
3. **Propose links** (this season only). For each Email and SMS activity whose send date has passed, propose linked campaigns from this season's **Q5**: same lane, sent within 7 days of the activity's start week (using Q5's real send date), and a similar name or audience. Group this season's campaigns into offers the same way, so one activity links all of its offer's campaigns.
4. **Show the user** one table covering activity, lane, week, audience (with size), program and linked campaigns. Under it, list anything you couldn't place, guessed, or left out: tests, card-decline texts, items outside October–December, sends under 25 recipients and unnamed campaigns ("Message 1"). Ask them to confirm or correct it in one reply.

Leave list size, conversion and average order blank where no source gives them. Those activities show as incomplete, which becomes the team's to-do list. An activity may stay unlinked, and an audience may stay a label with a typed list size.

**Done when:** every activity is one the user has confirmed (or is marked "Assumed — not confirmed"); every audience maps to a real list or the user chose a label; and every Email or SMS activity whose send date has passed is linked or marked unlinked.

### 6. Build the live artifact

1. **Choose the sync path** (see **Sync**). Read your artifact tool's documentation for a way a published page calls a connector tool. Where there is one, use **page sync** and grant the page New Vintage's `safe_tenant_sql` only. Otherwise use **chat sync**.
2. **Grant downloads.** Download CSV and Download the template need your artifact tool's file-download capability (for example `downloads`: `downloads.save({ filename, data })`, which asks the viewer to confirm). A page can't start a download any other way. Where the capability is unavailable, hide both and say so in Sync details.
3. **Build the page** to **Page**, **Sync**, **Replay**, **Example view**, **Rules** and **Look**, and publish it as a live artifact. Load everything Steps 1–5 confirmed as the starting data (tenant, sync path, mode and as-of date, goal, cutoffs, lane defaults, last year, baseline, program rules and setup states, and activities), with `planRevision: 1`. Include Example winery behind **More → Show the example**. Build both looks (New Vintage and Commerce7, see **Look**), with New Vintage as the default.
4. **Run the first sync**, from the page (page sync) or from chat (chat sync).

**Done when:** the page has the season bar with Refresh and the synced stamp, the goal header, the calendar with all three views, both filters and the stepper, the activity drawer, all four dialogs (Last year, Import activities, Sync details, Program setup), the How to use tab, both looks and every row of **States**; it opens on the Plan tab in the Quarter view with the user's data loaded; and every **What syncs** section shows Synced (an empty result counts) with `live.syncedAt` set. On the example start, the page opens on the example view instead, and the sync criteria wait until there is a plan.

### 7. Check the numbers

Run checks 1–14 on Example winery (More → Show the example) and checks 15–21 on the winery's own plan; on the example start, run 1–14 only. Compute checks 1–8, 15, 16 and 21 with the page's own code where you can run it, otherwise by tracing the calculation. For checks 9–14 and 17–20, which need a rendered page, trace each to the code that implements it. Confirm all of the following:

1. Planned revenue **$967,698**, including the $180,000 baseline. The verdict reads **"The plan is $332,302 short of the goal."**
2. By program (planned): club $577,785 · ecommerce $78,233 · corporate gifting $29,580 · private client $73,300 · tasting room $140,000 · other $68,800. These add up to $967,698. Setting the baseline growth to 10% raises planned revenue to **$985,698**; set it back to 0% for the remaining checks.
3. **"15 of 16 activities have revenue inputs."** (Member holiday add-on has no list size, and as a non-shipment Club activity it has no lane default.)
4. Against last year ($1,180,000): goal **+10%**, plan **−18%**.
5. With program actuals typed as overrides (club $601,000 · ecommerce $95,000 · tasting room $78,000 · other $26,000, so actual revenue $800,000) and booked through the week of Nov 9:
   - pacing reads **"$30,080 ahead of plan"**
   - last year at this point reads **$833,000**
6. The Fall shipment bills chip shows **$545K**.
7. **Activity actuals:** Fall release email (e1) with $12,600 actual and Fall shipment bills (c3) with $551,000 actual, on top of check 5, gives:
   - the Actual revenue line reads "Activity actuals $563,600 · Not from an activity $236,400"
   - e1 reads **"+2% vs plan"** and c3 reads **"+1% vs plan"**
   - program actuals and pacing are unchanged
8. **Import:** importing the **Import check** CSV with **Add to my plan**:
   - the preview reads **"2 activities ready, 1 needs a lane"**
   - after import, planned revenue is **$1,030,833**, the verdict reads **"$269,167 short,"** and corporate gifting is **$88,740**
9. **Views:** the calendar opens in the Quarter view with all 13 weeks. Month shows the weeks whose Monday falls in the chosen month, and Week shows one week as a list per lane. In Month and Week, the stepper moves one month or one week and stops at the season's ends; Quarter has no stepper, and there are no separate month buttons. The week headers stay visible while the page scrolls.
   **Key dates line up in every view:** Ground cutoff (Dec 15, 2026) sits one-seventh of the way into the week of Dec 14 in Quarter and in Month (December), and is listed under the week of Dec 14 in Week. No key date line or label is drawn outside the weeks a view shows, and the Month columns fill the frame.
10. **Filters:** deselecting SMS in the channel filter hides the SMS lane and leaves every total unchanged. Selecting only Corporate gifting in the program filter shows only corporate gifting activities, in every lane.
11. **Click to add:** in the Quarter view, clicking the empty Email cell in the week of Oct 26 opens a new activity with lane Email and start week Oct 26 filled in, with the title focused. Closing it without a title removes it.
12. **Narrow screens:** at 390px wide, and at 720px (the artifact pane beside a chat), only the calendar frame scrolls sideways; the page itself stays within the frame, and the lane labels stay in view.
13. **Logo and look:** the logo pill sits at the left end of the season bar, shows the New Vintage lockup at 26px tall in light and dark, and links exactly to the URL in **Logo**. More → Look → Commerce7 survives a reload, and checks 1–8 give the same numbers in both looks.
14. **Sync details and How to use:** More → Sync details opens the dialog in **Sync details dialog**, and the How to use tab has all eight sections.
15. **Reconciliation:** run **Q8**'s bucket output for the plan's season up to today (all $0 before the season). Its `email`, `sms`, `club` and `not_from_activity` buckets add up to Q1's revenue to the dollar, and the page's synced Actual revenue equals that same figure. So does the sum of the **Program template**'s six programs.
16. **Roll-ups:** each linked Email or SMS activity's synced actual equals the sum of its campaigns' rows in Q8's per-campaign variant, and the program actuals add up to actual revenue.
17. **Overrides:** typing an override shows both values ("Synced $612,400 · Override $640,000"), the override drives every total, it survives Refresh, and **Clear override** restores the synced figure.
18. **Setup:** a program in `programSetup` `'pending'` shows **Setup** in the program legend, and the button opens the Program setup dialog with the copy-ready prompt. A program with confirmed rules shows no Setup button.
19. **Failed sync** (page sync only): with the next call forced to reject (a test flag in the code, off by default), the page keeps the last numbers with their time, shows "Couldn't refresh," and offers Retry.
20. **Replay** (replay pages only): every order-based query stops before the as-of date, the Today line and the status line use the as-of date, and pacing counts only weeks before it.
21. **Privacy:** the saved plan holds aggregates only; searching it for "@" and for phone numbers finds nothing.

Then tell the user, one sentence each: their planned revenue and verdict, and their actual revenue and pacing (or, before the season, when pacing starts).

**Done when:** every check that applies matches exactly or is traced to its code, and any mismatch has been fixed in the code.

### 8. Hand over

Tell the user, in seven lines or fewer:
- that the **How to use** tab explains the page
- what syncs by itself, and what they type (the plan, the revenue goal, cutoffs, events and Direct outreach)
- how to override a synced number, and how to link a campaign once it's sent
- that a program showing **Setup** isn't recognized in Commerce7 yet, and the button gives them a prompt to paste here
- how the numbers refresh: on open and with Refresh (page sync), or by saying "Refresh my planner" in this chat (chat sync)
- that teammates need their own New Vintage connection to refresh the numbers
- every item marked "Assumed — not confirmed," if any

On the example start, say one line instead: the example is theirs to explore, and saying "Build my plan" here sets up their own (step 2, without the Start question).

**Done when:** the user has the page and those notes.

---

## Later requests

These run only when the user asks, after the page is built.

### When the user asks for changes in chat

Treat any later request ("add a Black Friday SMS," "change the club list size to 2,300," "add a wholesale program," "make the chips bigger") as an edit to this page:

1. Start from the most recent plan you know. If the user has edited on the page since your last change, ask them to download the CSV and attach it, and use it as the current plan.
2. Make the change, keeping `live` and every synced value as they are.
3. Raise `planRevision` by 1, so the page loads your update over its saved copy (see **Saving**).
4. Confirm the change in one sentence, including any total it moved.

**Done when:** the updated page loads the change, and the user has the one-sentence confirmation.

### When the user pastes a Setup prompt

The Program setup dialog gives the user a prompt to paste into this chat (see **Program setup dialog**). When it arrives:

1. Run step 4 for that program only: the Corporate Orders default (corporate gifting only), the channel question, the "how does your team mark" question, the lookups, and the Program template on last season and on the last 90 days.
2. Show the split and ask the user to confirm it. If the winery truly doesn't mark these orders, offer **Keep entering by hand**, which saves `programSetup` as `'manual'` and hides the Setup button without adding rules.
3. **Verify the queries:** run the Program template with the new rules for this season up to today and for last season. Both must return, and the six programs must add up to Q1's revenue to the dollar.
4. **Lock the decision:** save the rules in `live.programRules` with `confirmedAt`, set `programSetup` to `'rules'`, and re-run the split for this season's program actuals and last season's program figures, so the comparison with last year stays like for like. The rule order means orders the program now claims leave ecommerce, Other or tasting room, so nothing counts twice. Lane defaults stay as they are.
5. Raise `planRevision` by 1. The page loads the update and no longer shows Setup for that program.
6. Confirm in one sentence, with the program's actual revenue so far this season and the program it came from.

**Done when:** the program has locked rules (or is set to manual), the page has loaded them, and the Setup button is gone for that program.

### When the user asks to refresh from chat

For chat sync, or when the page can't reach the connector ("refresh my planner"): run every query in **What syncs**, write the results into the page with a newer `live.syncedAt`, keep `planRevision` as it is (see **Saving**, Loading), and tell the user the new actual revenue and pacing in one sentence.

**Done when:** the updated page shows the new synced stamp.

### When the user asks to plan this season after a replay

Build a second live artifact for this season: run step 2 (without the Start question), step 3, step 4's split on the new last season, and steps 5–8, reusing the tenant and the program rules. The replay page stays as it is.

**Done when:** the user has both pages.

---

## Terms

- **Season**: October–December of one year: 13 weeks from the first Monday on or after Oct 1.
- **Season week**: one of the 13 weeks, named by its Monday ("Week of Oct 19"). Week 1 is the first.
- **Plan's season**: the season the page plans: this year's, or the replayed one.
- **Most recent season**: the latest finished season (holiday 2025, for a page built in 2026).
- **Last season**: the season before the plan's season. For a 2026 plan it is holiday 2025; in a replay of 2025 it is holiday 2024.
- **Replay**: a planner for a past season that acts as if today were the **as-of date** inside it (see **Replay**).
- **Today**: the current date, or the as-of date in a replay. Orders count up to the start of today.
- **Plan**: the calendar of activities as the team wrote it.
- **Activity**: one item on the plan, such as an email, SMS, club shipment or event, running for one or more weeks. An email or SMS activity is one **offer**, with its segment splits, reminders and resends inside it.
- **Lane**: a calendar row grouping activities by channel.
- **Direct outreach**: the lane for direct mail and personal phone calls to top customers.
- **Program**: the revenue bucket revenue rolls up to: club, ecommerce, corporate gifting, private client, tasting room or other. Activities belong to club, ecommerce, corporate gifting, private client or other (events usually go in other). Tasting room holds only its baseline.
- **Baseline**: the revenue the plan expects without any activity: last year's weekly Tasting room and Other figures, plus a growth %. It counts toward planned revenue.
- **Program rules**: the confirmed ways the winery marks corporate gifting and private client orders in Commerce7 (step 4).
- **Audience**: the people an activity targets: a New Vintage segment, Commerce7 club or Commerce7 tag, or a typed label.
- **Linked campaign**: a real email or SMS campaign that delivered an activity. An activity can link several.
- **Revenue goal**: the season's DTC dollar target, tasting room and phone included.
- **Assumptions**: the numbers the plan rests on: lane defaults and the baseline growth %.
- **Lane default**: the conversion % and average order an activity inherits from its lane until it has its own, measured from last season or the New Vintage median. The Club lane default applies only to activities that bill a club shipment.
- **Projected revenue**: one activity's expected revenue.
- **Planned revenue**: the sum of projected revenue across complete activities, plus the baseline.
- **Verdict**: the headline sentence saying whether planned revenue reaches the revenue goal, and by how much.
- **Matched orders**: Web or Inbound orders a customer placed within 7 days of clicking a linked email or SMS (see **Data rules**).
- **Activity actuals**: the revenue an activity actually brought in, and its orders: matched orders for Email and SMS, billed shipments for a club shipment, and typed figures for events and Direct outreach.
- **Actual revenue**: all DTC revenue from Commerce7 paid orders so far this season, net of refunds. It is the sum of the six program actuals.
- **Not from an activity**: actual revenue minus the sum of activity actuals, such as tasting-room sales and web orders with no click before them.
- **Booked through**: the last season week that actual revenue covers.
- **Planned to date**: the part of planned revenue scheduled up to the booked-through week.
- **Pacing**: actual revenue minus planned to date: ahead or behind plan.
- **Last year**: last season's actual October–December revenue, with program and weekly figures.
- **Synced value**: a value the page fills from the New Vintage connector. It is read-only and shows its source and time.
- **Override**: a value the team types next to a synced value. When there is one, it drives every total, and both values stay visible.
- **Typed**: a value with no synced source (events, Direct outreach, the goal), entered by the team.
- **Calculated**: a value the page works out from synced, override and typed inputs. It's labeled "Calculated."
- **Look**: the page's visual style, New Vintage (default) or Commerce7, chosen in More. It changes appearance only.
- **Incomplete**: an activity on a revenue lane that is missing list size, conversion or average order. The page labels it "needs numbers," and it reads as a to-do for the team.

---

## Page

### Season bar

The top strip, left to right:
- **Logo:** the New Vintage logo pill, at the left end (see **Logo**).
- **Heading:** "{Winery} · Holiday plan, October–December 2026" (using the season's year), with the status line under it:

| When | Status line |
| --- | --- |
| Before the season | "13 weeks · Oct 5 – Jan 3 · Starts in 6 days" |
| During the season | "13 weeks · Oct 5 – Jan 3 · Week 3 of 13" |
| After the season | "13 weeks · Oct 5 – Jan 3 · Season ended" |
| Replay | "Replay of holiday 2025 as of Dec 1, 2025 · Week 9 of 13" |

- **At the far right:** the **Refresh** button with its stamp, "Synced 4 min ago" (see **Sync states**), then the **More** menu.
- **More menu:**
  - **Sync details**: opens the Sync details dialog
  - **Download CSV**
  - **Import activities**
  - **Edit last year**
  - **Look**: two radio items, **New Vintage** (default) and **Commerce7** (see **Look**)
  - **Show the example**: opens the example view (see **Example view**)
- **Tabs,** under the bar: **Plan** and **How to use**. On first open, a one-line hint under the tabs reads "New here? Start with How to use." It closes when clicked away.

On a phone the logo, heading and More share the first row, and Refresh drops to a second row.

### Goal header (Plan tab)

| Element | Behavior |
| --- | --- |
| Verdict | The headline, 20px or larger, in its status color with an icon: "The plan is $332,302 short of the goal." or "The plan covers the goal with $40,000 to spare." |
| Numbers | Revenue goal (editable) · Planned revenue (Calculated) · Actual revenue (Calculated: the sum of program actuals). Under Actual revenue, its source in one line: "Sum of your programs, from Commerce7 paid orders · synced 4 min ago" (or "· includes 1 override"). Under that, one line splits it: "Activity actuals $563,600 · Not from an activity $236,400." Then a "Booked through" label above a dropdown of weeks ("Week of Nov 2"), synced with actual revenue. |
| Goal bar | One horizontal bar of planned revenue, split into program segments. Unlabeled marks sit on the bar: a solid tick for the revenue goal, a dashed tick for last year, and a small triangle for actual revenue. A key under the bar names each with its amount: "Goal $1,300,000 · Last year $1,180,000 · Actual $800,000." |
| Program legend | A table, one row per program: color and name, Planned, Synced actual, Override, and Share, every figure right-aligned. Tasting room's Planned cell reads "Baseline" in its tooltip. A corporate gifting or private client row in `programSetup` `'pending'` shows a **Setup** button after its name (see **Program setup dialog**). Programs with no planned or actual revenue collapse into one line, "Not used: Private client." |
| Last-year line | "Last year: $1,180,000 · Goal +10% · Plan −18%." When last year is empty, this shows an **Add last year** link. |
| Pacing line | "$30,080 ahead of plan through the week of Nov 9 · Last year at this point: $833,000." Before any actual revenue exists, it reads "Pacing starts when the season's first orders sync." |
| Needs numbers | The line "15 of 16 activities have revenue inputs," beside a button reading "Show what needs numbers (1)." The button highlights incomplete activities and dims the rest. Pressed again, it clears the highlight. |
| Assumptions | A disclosure, collapsed by default. Its header is one full-width `<button aria-expanded aria-controls>`: a chevron icon (pointing right when collapsed, down when open, rotating in 160ms), the title "Assumptions," and a one-line summary of the current values, "Email 0.06% · SMS 0.24% · Baseline +0% · Show," whose last word reads "Hide" when open. It takes the hover background and a pointer cursor, so it reads as clickable and never as an empty card. Open, it holds the Lane defaults table (each lane's source: "holiday 2025 actual" or "New Vintage median") and the **Baseline** row: "Tasting room and Other, from last year's weekly figures," with a growth % input (default 0%). |

When the page scrolls past the header, it collapses into one sticky strip: verdict, planned revenue, revenue goal and pacing. Totals sit in an `aria-live="polite"` region. Currency inputs accept "$1,300,000" or "1300000" and reformat on blur as whole US dollars. When activity actuals add up to more than actual revenue, a note under Actual revenue reads "Activity actuals add up to $X more than actual revenue. Check the overrides and typed figures."

### Synced and override fields

Every synced value uses one component: total program actuals, Email, SMS and Club activity actuals and orders, list sizes, and last year's figures.

- **Synced value:** read-only, with a source line in `text-tertiary` ("From Commerce7 · synced 4 min ago"). Before its first sync it reads "Not synced yet."
- **Override:** an input beside it, placeholder "Override," labeled for screen readers "Override {field}."
- **With an override:** the field shows both values on one line, "Synced $612,400 · Override $640,000 (+$27,600)," and the override drives every total. A **Clear override** link removes it. A sync never changes an override.
- **Typed-only fields** (events, Direct outreach, the goal) are plain inputs with no synced value.
- **List size** has no actual: sent counts aren't tracked as an actual. Its synced value is the audience size (see **List size**).

There is no "Synced" or "Not marked" badge inside table rows. Where a row needs a status, it shows a small icon with a tooltip that is also its accessible name.

### Program setup dialog

Opens from a program's **Setup** button. Title: "Set up {program}."

- **Intro:** "Commerce7 doesn't mark {program} orders in a way this planner recognizes yet, so its actual revenue is entered by hand. Paste the prompt below into the chat where you built this page, and Claude will help you set it up."
- **Prompt,** in a read-only text box with a **Copy** button (use the clipboard API; when it's blocked, select the text and say "Press ⌘C to copy"):

  > Set up how my holiday planner recognizes {program} orders. Ask me how my team marks these orders in Commerce7 (tags, codes, promotions, the Corporate Orders tool, or a gifting partner) and which channels they arrive on. Then run the Program template to show me last season's split and confirm it with me. Once I confirm, verify the queries, lock the rules, recalculate this season and last season by program, and update the page so the Setup button for {program} goes away.

- **Buttons:** **Keep entering by hand**, which sets `programSetup` to `'manual'` and hides the button; and **Close**.

The button shows only while `programSetup` for that program is `'pending'`.

### Last year dialog

This opens from **Add last year**, **Edit last year**, or the last-year line. It has three sections, each a synced value with an override:
- **Total**
- **By program**, all six programs
- **By week** (13 rows)

Every figure is right-aligned. When the program or weekly figures don't add up to the total, the dialog shows the difference: "Weekly figures add up to $1,150,000, $30,000 less than the total." Last year syncs at setup, and again when the user presses **Re-measure** in the dialog (page sync). Under chat sync the button reads "Re-measure in chat" and opens the same note as Refresh. Re-measuring also refreshes the baseline.

### Calendar (Plan tab)

- **Toolbar,** above the grid, wrapping on narrow screens:
  - **View:** a segmented control, "Quarter | Month | Week," default Quarter.
  - **Stepper,** in Month and Week views only, beside the view control: previous and next arrows around the current label ("‹ December ›" in Month, "‹ Week of Dec 7 ›" in Week), stepping one month or one week and stopping at the season's ends. Quarter shows no stepper, because all 13 weeks are already on the grid. There are no separate month buttons.
  - **Layout:** one row at full width: the view control and stepper on the left; Channels, Programs, Density and **Add activity** on the right. On narrow screens it wraps to two rows, with the view control and stepper on the first.
  - **Channels:** a menu of checkboxes, one per lane, all on by default. Turning a lane off hides it from view only; totals never change. While any lane is off, the toolbar reads "Showing 5 of 7 lanes · Show all."
  - **Programs:** a menu of checkboxes, one per activity program (club, ecommerce, corporate gifting, private client, other), all on by default. Turning one off hides its activities in every lane; Key dates and Campaign theme activities with no program stay visible. While any is off, the toolbar reads "Showing 3 of 5 programs · Show all."
  - **Density:** a toggle button reading **Compact** while chips are full size and **Expand** once compacted (`aria-pressed`). Compact chips show one line.
  - **Add activity:** a button that opens a new activity drawer.
  - The view, channels and programs are viewer preferences, saved under `holiday-planner.view` (with try/catch), never in the plan.
- **Quarter view:** all 13 weeks, labeled by Monday date and grouped by month. Each week column takes an equal share of the frame, never narrower than 96px; when the frame is narrower than that allows (the artifact pane beside a chat, a phone), the calendar scrolls sideways inside its frame, with the lane labels pinned at the left. On open, it scrolls so the current week (or the as-of week) sits second from the left, with one week before it in view.
- **Month view:** the weeks whose Monday falls in the month (four or five). Their columns split the frame's full width after the lane labels equally, never narrower than 160px each, so no empty space trails the last week. Activities that start before or run past the month show a small arrow at that edge.
- **Week view:** one season week as a list, one section per lane: each activity as a full-width row with title, audience and size, program, offer, projected and actual revenue, and its link status, then that week's key dates with their dates. Each lane ends with **+ Add to {lane}**.
- **Sticky header:** the month and week header row stays in view while the page scrolls, below the collapsed goal strip.
- **Today line:** a 2px solid line in `text-primary` marks today while the season runs.
- **Lanes,** top to bottom:

| Lane | Revenue | Actuals |
| --- | --- | --- |
| Key dates | none (labels) | none |
| Campaign theme | none | none |
| Email | yes | synced from matched orders |
| SMS | yes | synced from matched orders |
| Direct outreach | yes | typed |
| Club | yes | synced from billed shipments |
| Events & tasting room | yes | typed |

- **Grid:** no vertical lines between weeks; alternate weeks take a faint band (`surface-subtle` at 50%) instead. Lanes are separated by 1px `border-subtle` horizontal lines, including the line under Key dates. Dotted lines are reserved for key dates.
- **Key dates:**
  - **Which dates:** Halloween (Oct 31), Thanksgiving (fourth Thursday of November), Black Friday, Cyber Monday, Hanukkah (its first night, from the **Hanukkah** table), Christmas, and the three cutoff dates.
  - **How they show:** each is drawn down through every lane as a **2px dotted line in `accent-editorial-500`** (copper), behind the chips.
  - **Position:** in Quarter and Month, a key date's line and label sit at its day inside its week: x = the left edge of the week's column + (days since that week's Monday ÷ 7) × the column width. Key dates, chips, the Today line, the week bands and the header all share one column geometry, recalculated whenever the view changes or the frame resizes, so a date never drifts from its week. A view draws only the key dates that fall in the weeks it shows: Month (December) draws Ground cutoff, 2-day cutoff and Christmas inside the Dec 14 and Dec 21 columns, and nothing past the last column. In Week, key dates are listed with their dates, not drawn.
  - **Labels:** in the Key dates lane, each date is a small copper pill showing its name only ("Thanksgiving"). The date is in its tooltip and accessible name. Labels that would overlap drop to a second or third row, so every label is readable (Hanukkah and Christmas share Dec 25 in some years).
  - **Editing:** users can edit, add and remove them.
- **Chips:**
  - Each activity is a `<button>` spanning its weeks. It shows the title (up to two lines, with long words hyphenated), the audience size, and projected revenue in compact form ("$595K"). Once the activity has actuals, the chip shows actual revenue instead ("$601K actual").
  - A tooltip and the `aria-label` add the full title, audience, dates, linked campaigns, and plan and actual figures.
  - Program shows as the program color **plus** a short text label.
  - Incomplete chips show "needs numbers."
  - Email and SMS chips whose send date has passed with no linked campaign show "Link campaign."
  - Campaign theme chips are neutral outlines with no program fill.
- **Click to add** (Quarter and Month): every empty lane-and-week cell is a target. On hover it shows a faint "+ Add," and clicking it opens a new activity drawer with that lane and start week filled in and the title focused. Closing a new activity that still has no title and no numbers removes it. A click in an empty cell of the Key dates lane adds a key date on that week's Monday instead.
- **Stacking:** activities that overlap within a lane stack in sub-rows, each fully visible. Sort a lane's activities by start week, then longest first, and place each in the first free sub-row. Empty space in any sub-row counts as that lane's cell.
- **Capacity:** the calendar holds 60+ activities.

### Activity drawer

Clicking a chip slides a drawer in over the right side of the calendar, so the grid keeps its width. On a phone it opens as a bottom sheet. Edits apply as the user types, and totals recompute immediately. Labels, inputs and numbers align to the top of each row, and figures are right-aligned.

| Group | Fields |
| --- | --- |
| Basics | Title; lane (dropdown); start week (dropdown); number of weeks |
| Who and why | Audience (a picker of segments, clubs and tags from **Q4** with their sizes, or a typed label); program (dropdown: club, ecommerce, corporate gifting, private client, other, or none); offer |
| Linked campaigns | Email and SMS only: a picker of this season's sent campaigns in the lane (**Q5**), those sent within 7 days of the start week first. An activity can link several. |
| Plan | List size (a synced value with an override, see **List size**); conversion % and average order, each showing "Using Email default: 0.06% (New Vintage median)" until the activity has its own, with a "Reset to default" link |
| Club lane only | "Bills a club shipment" switch. Off, the activity has no lane default and needs its own conversion and average order. |
| Result | Projected revenue (Calculated) |
| Actuals | Actual revenue and actual orders: synced values with overrides for Email, SMS and Club, typed for the rest. Direct outreach and events read "Calls, mail and events aren't tracked. Type the results here." Once there is actual revenue, the group shows "+2% vs plan." With orders and list size, it also shows the actual conversion % and average order. A line reads "Counts toward {program}." |
| Actions | Duplicate; Delete (two-step: the first click changes the label to "Click again to delete," which resets after 3 seconds) |

Key dates and Campaign theme activities show only Basics and Who and why. People move activities with the lane and start-week dropdowns. With a chip focused, Alt+←/→ moves it one week and Alt+↑/↓ one lane. Esc closes the drawer and returns focus to the chip.

### Import activities

A dialog whose first line says what it does: "Add activities from a spreadsheet. Each row becomes one activity on the calendar. Your goal, key dates and last year stay as they are."

- A **Download the template** link gives a CSV with the columns in **Activity fields**, using readable headers: Title, Lane, Start week, Weeks, Audience, Program, Offer, List size, Conversion %, Average order, Bills club shipment, Actual revenue, Actual orders. It includes two filled example rows.
- **Start week** accepts a date ("Oct 19") or a week number. Lane and Program accept their names as shown on the page.
- The user picks a CSV and chooses **Add to my plan** or **Replace my plan**.
- A preview follows, for example "18 activities ready, 2 need a lane," listing each problem row by row number and title before anything changes. A note says rows that need fixing are skipped.
- The button names what it will do ("Import 18 activities") and adds only the ready rows. Imported audiences arrive as labels, and imported actuals and list sizes as overrides.

### Sync details dialog

This opens from More → Sync details, and from the synced stamp. It shows where the numbers come from:

- **Intro:** "Actual revenue, last year, audiences and each email, SMS and club result fill in from {Winery}'s Commerce7, Klaviyo, Mailchimp and RedChirp data through New Vintage." Name only the platforms the tenant syncs.
- **What syncs:** one row per item in **What syncs**, with its last sync time and status ("Synced 4 min ago," "Couldn't refresh").
- **How orders are matched:** "An email or SMS gets credit for a web or phone order its clicker placed within 7 days. When someone clicked several, the most recent click gets it."
- **How programs are assigned,** in plain words: "Corporate gifting: orders with sales attribute code 'corp'. Club: Club channel. Ecommerce: Web. Tasting room: POS. Other: phone and everything else." A program without rules reads "Not set up yet. Enter its actual by hand, or press Setup."
- **What stays typed:** events, Direct outreach, and every override (with a count: "3 overrides").
- **Note:** "Teammates need their own New Vintage connection to refresh. Without one, they see the last sync."
- **Buttons:**
  - **About New Vintage** links to the Logo URL, with `utm_content=sync-details`
  - **Close**

### How to use tab

A short orientation page, readable in about two minutes. It has eight sections, each two to four sentences, written for someone new to the page:

1. **What this page answers:** does the plan reach the goal, and how is it tracking?
2. **Reading the verdict, goal bar and programs,** including the baseline: tasting room and other sales the plan expects without any campaign.
3. **Finding your way around the calendar:** Quarter, Month and Week views, the arrows that step through months or weeks, and the channel and program filters, which change only what you see.
4. **Adding and editing activities:** click an empty spot in a lane to add one right where it belongs, click a chip to edit it, and use the dropdowns to move it. Link each email and SMS to its campaigns once it's sent.
5. **Bringing in your own calendar (import):**
   1. More → Import activities → Download the template.
   2. Open it in Excel or Google Sheets. Put one activity on each row: its title, lane, start week and so on. Keep the column headings.
   3. Save or download it as CSV.
   4. Back on this page, choose Import activities, pick the file, then choose Add to my plan or Replace my plan.
   5. Check the preview, which lists any rows that need fixing, then press Import.

   Your goal, key dates and last year stay as they are.
6. **Synced values, overrides and your weekly habit:** what fills in by itself, what you enter, and what the page works out. Type an override next to any synced number; both stay visible and yours drives the totals. A program showing **Setup** isn't recognized in Commerce7 yet. Each week:
   - press Refresh
   - link the week's sent emails and texts to their activities
   - type actuals for events and Direct outreach
   - check pacing
7. **Make it yours by asking:** you can ask the assistant in the chat where you built this page. For example:
   - "Add a Black Friday SMS to SMS subscribers in the week of Nov 23."
   - "Change the club shipment list size to 2,300."
   - "Add a wholesale program."

   It updates the page for you. If you've edited here since, download the CSV and attach it to your request, so it starts from your latest plan. More → Sync details explains where the numbers come from.
8. **Saving and sharing:**
   - changes save by themselves
   - teammates need their own New Vintage connection to refresh
   - Download CSV opens in Excel or Google Sheets

The tab ends with a **Go to my plan** button.

### Logo

The New Vintage lockup (mark and wordmark), in a white pill at the left end of the season bar:

- **Pill:** `#ffffff` background in both themes and both looks, a 1px `border-default` edge, the control radius, 6px × 14px padding, at least 40px tall. On hover, the edge moves to `border-strong` and the card shadow appears.
- **Image:** an `<img>` 26px tall with auto width (about 78px) and an empty `alt`. Embed the source exactly as given below.
- **Link:** `https://newvintage.ai/?utm_source=holiday-planner&utm_medium=referral&utm_campaign=c7-webinar-2026&utm_content=season-bar-logo&utm_term=new-vintage`, opening in a new tab with `rel="noopener"`. The accessible name is "New Vintage."

The image `src` (the SVG from newvintage.ai):

```text
data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='130 130 1717 575' fill='none'%3E%3Cg clip-path='url(%23a)'%3E%3Crect width='550' height='550' x='140' y='140' fill='%23fff' rx='275'/%3E%3Cpath fill='url(%23b)' d='M140  415a275  275  0  1  1  550  0H140'/%3E%3Cpath fill='%230b6623' d='M-27  420.9  404.2  526.2c8  1.9  3.3-7.1  11.3-5.2  7.7  1.9  16.8-1  23.3-5.5  6.5-4.5  22.7 .6  24.2-7.7s-.2-16.8-4.7-23.3c-4.5-6.5-11.6-10.6-19.5-11.5l-489.8-58zM809.7  426.4l-471.9  341.9c-12.9  9.4-28.6  12.8-44  8.8-15.4-4-29.2-15.2-38.1-30.3-8.8-15.1-11.8-32.7-7.7-48  4-15.4  14.7-27.4  29.2-34l27.9-12.8  501.6-230.7  27.9-12.8z'/%3E%3Cpath fill='%230b6623' d='m217.8  623.4-32.7  11c-15.1  5.1-31.2  3.7-44.7-4.6-13.6-8.3-23.6-22.8-27.6-39.7-4.1-16.8-1.8-34.4  6.5-47.9  8.3-13.6  21.9-22.1  37.7-24.5l34.1-5.1  613.9-92.4  34.1-5.1z'/%3E%3Cpath fill='%230b6623' d='M17.8  418  1255.6  471.5l68.8  3c8 .3  15.6-2.5  21.4-8  5.7-5.5  9.1-13.2  9.2-21.3 .2-8.1-2.8-16-8.3-21.7-5.5-5.7-13.1-8.9-21-8.9l-68.8 0-1239 .4-68.8 0z'/%3E%3C/g%3E%3Cpath fill='%23222' d='M1154  200.1h37.2l31.1  118.8h.6l29.9-118.8h35.4l28.6  118.8h.6l32.3-118.8h35.7l-49.9  159.1h-36l-29.5-118.2h-.6l-29.2  118.2h-36.9zM1111.3  265.6a53.5  53.5  0  0  0-3.7-16c-1.8-5.1-4.5-9.5-8-13.2-3.3-3.9-7.4-7-12.3-9.2-4.7-2.5-10.1-3.7-16-3.7q-9.2  0-16.9  3.4a37.5  37.5  0  0  0-12.9  8.9c-3.5  3.7-6.4  8.1-8.6  13.2-2  5.1-3.2  10.7-3.4  16.6zm-81.8  23.1q0  9.2  2.5  17.8c1.8  5.7  4.5  10.8  8  15.1  3.5  4.3  7.9  7.8  13.2  10.5q8  3.7  19.1  3.7c10.2  0  18.5-2.2  24.6-6.5q9.5-6.8  14.2-20h33.2q-2.8  12.9-9.5  23.1c-4.5  6.8-9.9  12.5-16.3  17.2q-9.5  6.8-21.5  10.2c-7.8  2.5-16  3.7-24.6  3.7-12.5  0-23.6-2.1-33.2-6.2-9.6-4.1-17.8-9.8-24.6-17.2-6.6-7.4-11.6-16.2-15.1-26.5q-4.9-15.4-4.9-33.8  0-16.9  5.2-32c3.7-10.3  8.8-19.2  15.4-26.8  6.8-7.8  14.9-13.9  24.3-18.5  9.4-4.5  20.1-6.8  32-6.8q18.8  0  33.5  8  15.1  7.7  24.9  20.6c6.6  8.6  11.3  18.6  14.2  29.8q4.6  16.6  2.5  34.5zM830  200.1h33.2v23.4l.6 .6q8-13.2  20.9-20.6  12.9-7.7  28.6-7.7  26.2  0  41.2  13.5t15.1  40.6v109.2h-35.1V259.1q-.6-18.8-8-27.1-7.4-8.6-23.1-8.6-8.9  0-16  3.4-7.1  3.1-12  8.9-4.9  5.5-7.7  13.2t-2.8  16.3v93.8H830zM1801.2  537.6a53.2  53.2  0  0  0-3.7-16c-1.8-5.1-4.5-9.5-8-13.2q-4.9-5.8-12.3-9.2c-4.7-2.5-10.1-3.7-16-3.7-6.2  0-11.8  1.1-16.9  3.4a37.5  37.5  0  0  0-12.9  8.9c-3.5  3.7-6.4  8.1-8.6  13.2-2  5.1-3.2  10.7-3.4  16.6zm-81.8  23.1q0  9.2  2.5  17.8c1.9  5.7  4.5  10.8  8  15.1q5.2  6.5  13.2  10.5  8  3.7  19.1  3.7c10.3  0  18.5-2.2  24.6-6.5q9.5-6.8  14.2-20h33.2q-2.8  12.9-9.5  23.1c-4.5  6.8-9.9  12.5-16.3  17.2q-9.5  6.8-21.5  10.2c-7.8  2.5-16  3.7-24.6  3.7-12.5  0-23.6-2.1-33.2-6.2s-17.9-9.8-24.6-17.2c-6.6-7.4-11.6-16.2-15.1-26.5q-4.9-15.4-4.9-33.8c0-11.3  1.7-21.9  5.2-32q5.5-15.4  15.4-26.8  10.2-11.7  24.3-18.5  14.2-6.8  32-6.8  18.8  0  33.5  8c10.1  5.1  18.4  12  24.9  20.6  6.6  8.6  11.3  18.6  14.2  29.8q4.6  16.6  2.5  34.5zM1661.3  622.8q0  36-20.3  53.5c-13.3  11.9-32.6  17.8-57.9  17.8q-12  0-24.3-2.5c-8-1.6-15.4-4.4-22.2-8.3-6.6-3.9-12.1-9-16.6-15.4-4.5-6.4-7.2-14.2-8-23.4h35.1c1  4.9  2.8  8.9  5.2  12  2.5  3.1  5.3  5.4  8.6  7.1  3.5  1.8  7.3  3  11.4  3.4  4.1 .6  8.4 .9  12.9 .9q21.2  0  31.1-10.5  9.8-10.5  9.8-30.2v-24.3h-.6q-7.4  13.2-20.3  20.6-12.6  7.4-27.4  7.4c-12.7  0-23.6-2.2-32.6-6.5-8.8-4.5-16.2-10.6-22.2-18.2-5.7-7.8-9.9-16.7-12.6-26.8-2.7-10.1-4-20.8-4-32.3q0-16  4.9-30.5c3.3-9.6  8-18.1  14.2-25.2q9.2-11.1  22.5-17.5c9-4.3  19.2-6.5  30.5-6.5q15.1  0  27.7  6.5c8.4  4.1  14.9  10.7  19.4  19.7h.6v-21.8h35.1zm-77.8-19.4c7.8  0  14.4-1.5  19.7-4.6q8.3-4.9  13.2-12.6c3.5-5.3  6-11.3  7.4-17.8q2.5-10.2  2.5-20.3  0-10.2-2.5-19.7c-1.6-6.4-4.2-12-7.7-16.9q-4.9-7.4-13.2-11.7c-5.3-2.9-11.8-4.3-19.4-4.3-7.8  0-14.4  1.6-19.7  4.9s-9.6  7.6-12.9  12.9q-4.9  7.7-7.1  17.8-2.2  9.8-2.2  19.7c0  6.6 .8  13  2.5  19.4q2.5  9.2  7.4  16.6c3.5  4.9  7.8  8.9  12.9  12q8  4.6  19.1  4.6M1479.5  595.8c0  4.3 .5  7.4  1.5  9.2  1.2  1.8  3.5  2.8  6.8  2.8h3.7c1.4  0  3.1-.2  4.9-.6v24.3c-1.2 .4-2.9 .8-4.9  1.2a49  49  0  0  1-5.8  1.5c-2 .4-4.1 .7-6.2 .9s-3.8 .3-5.2 .3c-7.2  0-13.1-1.4-17.8-4.3q-7.1-4.3-9.2-15.1c-7  6.8-15.6  11.7-25.8  14.8-10.1  3.1-19.8  4.6-29.2  4.6-7.2  0-14.1-1-20.6-3.1-6.6-1.8-12.4-4.6-17.5-8.3q-7.4-5.8-12-14.5c-2.9-5.9-4.3-12.8-4.3-20.6  0-9.8  1.8-17.8  5.2-24  3.7-6.2  8.4-11  14.2-14.5  6-3.5  12.5-5.9  19.7-7.4a208  208  0  0  1  22.2-3.7c6.3-1.2  12.4-2.1  18.1-2.5  5.7-.6  10.8-1.5  15.1-2.8  4.5-1.2  8-3.1  10.5-5.5  2.7-2.7  4-6.6  4-11.7q0-6.8-3.4-11.1c-2-2.9-4.7-5-8-6.5-3.1-1.6-6.6-2.7-10.5-3.1-3.9-.6-7.6-.9-11.1-.9-9.8  0-17.9  2.1-24.3  6.2q-9.5  6.2-10.8  19.1h-35.1q.9-15.4  7.4-25.5c4.3-6.8  9.8-12.2  16.3-16.3q10.2-6.2  22.8-8.6c8.4-1.6  17-2.5  25.9-2.5  7.8  0  15.5 .8  23.1  2.5s14.4  4.3  20.3  8c6.2  3.7  11.1  8.5  14.8  14.5  3.7  5.7  5.5  12.8  5.5  21.2zm-35.1-44.3c-5.3  3.5-11.9  5.6-19.7  6.5-7.8 .6-15.6  1.6-23.4  3.1a64  64  0  0  0-10.8  2.8c-3.5  1-6.6  2.6-9.2  4.6-2.7  1.8-4.8  4.4-6.5  7.7q-2.2  4.6-2.2  11.4  0  5.8  3.4  9.8a27  27  0  0  0  8  6.5c3.3  1.4  6.8  2.5  10.5  3.1  3.9 .6  7.4 .9  10.5 .9  3.9  0  8.1-.5  12.6-1.5a40.1  40.1  0  0  0  12.6-5.2c4.1-2.5  7.5-5.5  10.2-9.2  2.7-3.9  4-8.6  4-14.2zM1230.9  472.1h26.5v-47.7h35.1v47.7h31.7v26.2h-31.7v84.9c0  3.7 .1  6.9 .3  9.5 .4  2.7  1.1  4.9  2.1  6.8  1.2  1.8  3  3.3  5.2  4.3  2.3 .8  5.3  1.2  9.2  1.2h7.4q3.7-.3  7.4-1.2v27.1c-3.9 .4-7.7 .8-11.4  1.2-3.7 .4-7.5 .6-11.4 .6-9.2  0-16.7-.8-22.5-2.5q-8.3-2.8-13.2-7.7c-3.1-3.5-5.2-7.8-6.5-12.9-1-5.1-1.6-11-1.9-17.5v-93.8h-26.5zM1073.3  472.1h33.2v23.4l.6 .6c5.3-8.8  12.3-15.7  20.9-20.6  8.6-5.1  18.1-7.7  28.6-7.7  17.4  0  31.2  4.5  41.2  13.5  10.1  9  15.1  22.6  15.1  40.6v109.2h-35.1V531.2c-.4-12.5-3.1-21.5-8-27.1-4.9-5.7-12.6-8.6-23.1-8.6-6  0-11.3  1.1-16  3.4q-7.1  3.1-12  8.9c-3.3  3.7-5.8  8.1-7.7  13.2q-2.8  7.7-2.8  16.3v93.8h-35.1zM1003.1  411.5h35.1v33.2h-35.1zm0  60.6h35.1v159.1h-35.1zM830  472.1h38.2l40.3  122.2h.6l38.8-122.2h36.3l-56.9  159.1h-39.4z'/%3E%3Cdefs%3E%3CradialGradient id='b' cx='0' cy='0' r='1' gradientTransform='matrix(0 -252.5 245.861 0 415 392.5)' gradientUnits='userSpaceOnUse'%3E%3Cstop stop-color='%23ffc720'/%3E%3Cstop offset='1' stop-color='%23ffbf00' stop-opacity='.96'/%3E%3C/radialGradient%3E%3CclipPath id='a'%3E%3Crect width='550' height='550' x='140' y='140' fill='%23fff' rx='275'/%3E%3C/clipPath%3E%3C/defs%3E%3C/svg%3E
```

### Activity fields

`id, title, lane, startWeek (0–12), weeks, audience, program, offer, listSize, conversionPct, aov, billsClubShipment, actualRevenue, actualOrders, linkedCampaigns, assumed`

- `audience` is `{ kind: 'segment' | 'club' | 'tag' | 'label', id, name }`.
- `linkedCampaigns` is `[{ platform, campaignId }]`, for Email and SMS.
- `assumed` is true when the user moved on without confirming the activity.
- Each synced value (`listSize`, `actualRevenue`, `actualOrders`, and the plan's program actuals, `actualByWeek`, `actualThroughWeek`, `lastYear` and baseline figures) is stored as `{ synced, syncedAt, override }`, where `synced` and `override` are numbers or null. The effective value is `override ?? synced`.

Lane ids: `key` Key dates · `camp` Campaign theme · `email` Email · `sms` SMS · `outreach` Direct outreach · `club` Club · `event` Events & tasting room. Program ids: `club`, `ecom` ecommerce, `corp` corporate gifting, `pc` private client, `tasting` tasting room (baseline only), `other` other.

### Saving

- **What's saved:** the whole plan is one object:

  `{ schemaVersion: 5, edition: 'new-vintage', planRevision, mode: 'season' | 'replay', asOf, winery, goal, programActuals, actualByWeek, actualThroughWeek, seasonStart, laneDefaults, baseline: { growthPct, byWeek: { tasting, other } }, keyDates, lastYear: { total, byProgram, byWeek, byProgramWeek }, programSetup: { corp, pc }, activities, live }`

  `programActuals` and `lastYear.byProgram` carry all six programs. `programSetup` values are `'rules' | 'pending' | 'manual'`.

  `live` is `{ tenantId, syncPath: 'page' | 'chat', syncedAt, programRules, sections, audienceSizes }`, where `sections` holds each **What syncs** item's last result, time and status, as aggregates only, and `audienceSizes` caches Q9 results with their time. Selection, filters, the view, the open drawer and the example view stay out of it. The look and the view are viewer preferences, saved separately (see **Calendar** and **Commerce7 look**).
- **Where:** the platform's persistent artifact storage, shared by everyone who opens the page where the platform offers that; otherwise browser `localStorage` under the key `holiday-planner`. Wrap every read and write in try/catch.
- **Loading:**
  - When the page's built-in plan has a higher `planRevision` than the saved copy, it loads the built-in plan and keeps the saved copy one step back. A banner reads "Updated from chat. **Undo**."
  - When only the built-in plan's `live.syncedAt` is newer (a chat refresh), it copies the synced values and `live.sections` into the saved copy, leaving every override, typed value and other field as saved.
  - Otherwise it loads the saved copy.
- **Older saves:** upgrade a schemaVersion 4 plan: turn each `{ value, source, syncedAt }` into `{ synced: value if source was 'synced', syncedAt, override: value if source was 'typed' }`; drop `overlapDiscountPct`; drop `programActuals.none` and `lastYear.byProgram.none` (the next sync and Re-measure fill tasting room and other); set `programSetup` from `live.programRules` (`'rules'` where a program has rules, else `'pending'`); set `baseline.growthPct` to 0 with empty weeks until the next Re-measure.
- **When saving isn't available:** show one line: "Changes won't be saved in this browser. Use Download CSV to keep a copy."
- **Download CSV** exports activities in the template's columns, plus projected revenue, through the download capability (see step 6). It opens in Excel or Google Sheets and imports back unchanged.

### States

| State | Treatment |
| --- | --- |
| Example view | See **Example view** |
| Empty plan | A **Next steps** checklist in place of bars and chips, ticking itself off: add your first activity (click a spot on the calendar, import a spreadsheet, or ask in chat); fill in the numbers; link sent campaigns. |
| Half-filled | Totals cover complete activities; the Needs numbers button shows the count |
| Short of goal / covers goal | The verdict's status color, icon and words change |
| No last year | The last-year line becomes the Add last year link, the goal bar drops its last-year tick, pacing drops "last year at this point," and the baseline is $0 until last year exists |
| Before the season | Actual revenue reads $0 and the pacing line waits (see Pacing line) |
| Filters on | The toolbar's "Showing … · Show all" lines |
| Setup needed | The Setup button on that program's legend row |
| Assumed items | A one-line note above the calendar, "2 items are assumed — not confirmed," that highlights them when clicked |
| Replay | See **Replay** |
| Updated from chat | The Undo banner (see Saving) |
| Sync states | See **Sync states** |

---

## Sync

### What syncs

| Page value | Source | When |
| --- | --- | --- |
| Actual revenue's weekly figures and the booked-through week | **Q1**, the plan's season up to today | Every sync |
| Program actuals (all six), so Actual revenue | **Program template** with `live.programRules` | Every sync |
| Email and SMS activity actuals | **Q8** per-campaign variant, summed over each activity's linked campaigns; actual orders from the same rows | Every sync |
| Club activity actuals | **Q7**, the plan's season: billed revenue and billed shipments in each week, credited to the Club activity that bills a shipment that week | Every sync |
| Linked campaign picker | **Q5**, the plan's season | Every sync |
| Audience list sizes | **Q9**, all picked audiences in one call (see **List size**) | At setup, when an audience is picked, and on Refresh when the cached size is over 24 hours old |
| Audience picker | **Q4** | At setup, and when the picker opens if the cached list is over 24 hours old |
| Last year and baseline | Steps 3 and 4 | At setup, and on Re-measure |
| Lane defaults | Step 3 | At setup; to re-measure, the user asks in chat |

Everything else stays typed: events, Direct outreach, the revenue goal, cutoffs and key dates, and every override.

**List size** = Q9's `with_email` for Email, and `members` for Direct outreach, Club and events. For SMS, the activity shows Q9's `with_sms` with `sms_basis`, and beside it `with_sms_any_consent`: "312 with promotional consent · 2,932 with any consent." The synced value is promotional consent; the team overrides it to use any consent. Where `sms_basis` is `phone_on_file`, label it "phone on file, consent unknown." For a Club shipment already processed, list size is Q7's scheduled count. A label audience keeps an override, or its linked campaigns' sent count. In a replay, list size is the offer's person count from Q6's per-offer variant.

### Sync paths

- **Page sync:** the page calls `safe_tenant_sql` itself, with the viewer's own New Vintage connection, when it opens and when the user presses Refresh. For example, on a platform with an `mcp` capability, declare `{ mcp: { servers: [{ server: 'New Vintage', tools: ['safe_tenant_sql'] }] } }` and call `callTool('New Vintage', 'safe_tenant_sql', input)`. Follow your artifact tool's own documentation for the exact calls.
- **Chat sync:** you run the queries in chat and write the results into the page (see **When the user asks to refresh from chat**). The Refresh button then reads "Refresh in chat" and opens a note: "Say 'Refresh my planner' in the chat where you built this page."

### Calling the connector

- **Input:** `sqlText` (the query with its parameters filled in, keeping `'__TENANT_ID__'` literally), `tenantId` from `live.tenantId`, a one-sentence `context` naming the page's goal and holding no data ("Refresh the holiday planner's actual revenue by week."), and `statementTimeoutMs` 60000 for Q6, Q8 and the Program template, 15000 for the rest.
- **Order:** run the queries one at a time, never in parallel. When a response carries a `conversation_id`, send it with the next call.
- **Response:** a JSON payload `{ ok, resultMode, rows }`. Use it when `ok` is true and `resultMode` is `"complete"`; treat anything else as a failed section. Numbers arrive as strings ("518823.30"), so convert each one.

### Sync states

Each **What syncs** item is a section that succeeds or fails on its own, keeping its last good result.

| State | Treatment |
| --- | --- |
| Syncing | Numbers keep their last values, and the stamp reads "Syncing…" |
| Synced | "Synced 4 min ago," from the oldest section's time |
| A section failed | That section keeps its last numbers with their time: "Couldn't refresh. Showing data from Tue 9:14 AM." with a **Retry** button. Retry a failure the platform marks retryable once, after a short wait. |
| Every section failed the same way | One message for the page in place of the per-section ones |
| Connection needed | When the viewer has no New Vintage connection, or it lapsed: "Connect New Vintage in Claude's settings to refresh. Showing data from Tue 9:14 AM." |
| Not allowed | When the viewer declined the page's access: "This page can't reach New Vintage for you. Showing data from Tue 9:14 AM." |

---

## Replay

A replay page plans a past season as if today were its as-of date:
- **Today** is the as-of date everywhere: the status line ("Replay of holiday 2025 as of Dec 1, 2025 · Week 9 of 13"), the Today line, the booked-through default, the Quarter view's opening scroll and every order-based query (`:order_end`).
- **Plan:** built from the replayed season's own offers, each Email and SMS activity linked to its campaigns (step 5).
- **Last season**, its lane defaults and its baseline are the season before the replayed one; the **Goal default** says how the goal is set.
- **List size** is each offer's person count.

## Example view

More → Show the example opens Example winery in place of the plan:
- A banner reads "Example winery, example figures. Sync is off." with a **Back to my plan** button. With no plan yet, the banner reads instead "To build yours, say 'Build my plan' in the chat."
- Every value is typed (as overrides, with no synced values), sync is off, and the saved plan stays untouched. No program shows Setup.
- The example always uses the 2026 season (first Monday Oct 5) and the New Vintage median **Lane defaults**.

---

## Rules

- **Season:** 13 weeks from the first Monday on or after Oct 1. `startWeek` 0 is that Monday. Assign each week to the month of its Monday.
- **Effective conversion** = the activity's own conversion %, else its lane default. **Effective average order** works the same way. A Club-lane activity that doesn't bill a shipment has no lane default.
- **Complete** = list size > 0, effective conversion > 0 and effective average order > 0. Incomplete activities count $0 of planned revenue.
- **Projected revenue** = list size × effective conversion ÷ 100 × effective average order.
- **Baseline** for each season week and each of Tasting room and Other = last year's figure for that program and week × (1 + growth % ÷ 100). With no weekly program figures, the program's last-year total spread evenly over 13 weeks.
- **Planned revenue** = Σ projected revenue of complete activities + Σ baseline.
- **Program planned** = Σ projected revenue of the program's complete activities, plus its baseline (Tasting room and Other).
- **Gap to goal** = revenue goal − planned revenue. It drives the verdict: positive is short, zero or negative covers the goal.
- **Effective value** of any synced field = its override when there is one, else its synced value.
- **Activity actuals:**
  - **Vs plan** = actual revenue ÷ projected revenue − 1, shown as a whole-number % with + or −.
  - **Actual conversion** = actual orders ÷ list size × 100.
  - **Actual average order** = actual revenue ÷ actual orders.
- **Program actual** = the program's effective value. In the example view, with no override, Σ activity actual revenue of the program's activities.
- **Actual revenue** = Σ program actuals.
- **Not from an activity** = actual revenue − Σ activity actual revenue, never below $0.
- **Planned to date** = each activity's projected revenue spread evenly across its weeks, plus the baseline's weekly figures, summed over weeks 0 through the booked-through week.
- **Actual to date** = for each program, its synced weekly figures summed over weeks 0 through the booked-through week (a week in progress stays out); for a program with an override, the override in full. In the example view, actual revenue.
- **Pacing** = actual to date − planned to date. Positive reads "ahead of plan"; negative reads "behind plan."
- **Booked-through default** = the last season week that ended before today, clamped to the season. Before the season starts, there is none, and pacing waits.
- **Last-year total:** the Total field; otherwise the sum of the weekly figures; otherwise the sum of the program figures. Program and weekly figures stay as detail and never override the total.
- **Last year at this point** = the sum of last year's weekly figures for weeks 0 through the booked-through week.
- **% vs last year** = (value − last-year total) ÷ last-year total × 100, rounded to a whole number and shown with + or −.
- **Program figures** (planned and actual) are rounded to whole dollars with largest-remainder rounding, so they always add up to the rounded total.
- **Validation:**
  - weeks is a whole number from 1 to 13 − start week; values over that are clamped, with the note "Shortened to fit the season"
  - conversion is 0–100, with two decimals, kept exactly as entered or loaded (never rounded to one decimal)
  - list size, average order, actuals and overrides are 0 or more; growth % is −50 to 100
- **Formatting:**
  - whole US dollars with comma grouping
  - compact on chips and bars ($595K, $1.3M)
  - conversion with up to two decimals, trailing zeros dropped ("0.31%," "1.8%")
  - weeks written "Week of Oct 19"
  - every figure in a table or legend right-aligned, with tabular figures

### Lane defaults

The New Vintage medians, used where the user didn't choose a measured value. Email and SMS are click-matched, per person per offer, measured across New Vintage wineries' holiday seasons; they vary widely between wineries, so a measured value always beats them.

| Lane | Conversion % | Average order |
| --- | --- | --- |
| Email | 0.06 | $417 |
| SMS | 0.24 | $344 |
| Direct outreach | 20 | $800 |
| Club (shipments only) | 85 | $300 |
| Events & tasting room | 20 | $150 |

Changing a lane default re-projects every activity that inherits it.

### Grouping sends into offers

One offer is one activity. Group a season's Q5 campaigns like this:
1. **Leave out** tests (Q5 `looks_test`) and card-decline texts (Q5 `looks_decline`). Keep everything else, including shipping notices and will-call reminders, and list sends under 25 recipients and unnamed campaigns ("Message 1") for the user to confirm.
2. **Same lane, same send day, one audience between them:** segment splits ("Black Friday – Members" and "Black Friday – Non-Members"), and the same send split across Klaviyo and Mailchimp during a platform switch, are one offer.
3. **Follow-ups within 7 days** of the offer's first send fold into it: "[Follow-up]" campaigns, resends, reminders, "Last chance," "Final hours" and corrections of the same offer.
4. **Group by lane, send day and audience coverage; use names only as a hint.** Klaviyo clone names ("(clone)") and stale subject lines break name matching.
5. **Keep apart:** segments that belong to different programs (a corporate segment of a broader send), and the same offer by email and by SMS (two lanes, two activities).
6. **List size and conversion** count people, not sends: an offer's list size is the distinct people across its campaigns, and its conversion is matched orders ÷ those people. Reminders raise the conversion an offer earns; they don't add list size.

### Hanukkah

The first night (the evening it begins), by season year:

| Year | First night |
| --- | --- |
| 2023 | Dec 7 |
| 2024 | Dec 25 |
| 2025 | Dec 14 |
| 2026 | Dec 4 |
| 2027 | Dec 24 |
| 2028 | Dec 12 |
| 2029 | Dec 1 |
| 2030 | Dec 20 |
| 2031 | Dec 9 |
| 2032 | Nov 27 |
| 2033 | Dec 16 |
| 2034 | Dec 6 |
| 2035 | Dec 25 |

### Data rules

- **Revenue** is `orders.sub_total` (product revenue, in cents), net of refunds. Refund rows (`purchase_type = 'Refund'`) are negative, so sums are already net. Order counts and matched orders count only orders with `sub_total > 0`, so $0 will-call, pickup-to-ship and comp orders never count.
- **Weeks:** the week index is (local order date − season start) ÷ 7. Use `America/Los_Angeles` unless the winery gives another timezone.
- **Programs:** every order lands in exactly one program. The program rules come first, in order (corporate gifting, then private client), then the channel defaults: Club → club, Web → ecommerce, POS → tasting room, everything else (Inbound included) → other. So the six programs always add up to actual revenue.
- **Matched orders:** a click on a linked email or SMS by a customer, then that customer's **Web or Inbound** paid order within **7 days** of the click. The most recent click before the order, across email and SMS together, gets the credit (**Q8**), so no order counts twice. Inbound matters: many wineries take email- and SMS-driven orders by phone.
- **SMS clicks:** RedChirp stores only a recipient's last click, so for a recipient who clicked more than once, their delivered send time counts as a click too.
- **Totals use matched orders only.** Provider-reported revenue (RedChirp's `sms_conversions` and `sms_attribution_summaries`, Klaviyo and Mailchimp reports in `esp_campaign_reported_revenue`) counts people who never clicked, so it serves only as a cross-check in chat.
- **Klaviyo's own connector:** use it only for what New Vintage doesn't have yet, such as campaigns not yet sent, and call only its read tools (`get_campaigns`, `get_campaign`, `get_campaign_report`).

### Data facts

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

---

## Queries

Fill these parameters before each call:
- `:season_start` and `:season_end`: the season's first Monday, and 91 days later (exclusive)
- `:sms_lookback_start`: `:season_start` − 14 days, because texts are scheduled ahead of their send
- `:order_end`: `least(:season_end, today + 1 day)`; in a replay, the as-of date

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


---

## Look

The page uses the **New Vintage Labs Design System** by default, with a **Commerce7** look the user can switch to. Radii, the popover shadow and heading weight are tokens too (named below), so the Commerce7 look only swaps token values.

- **Build:** one self-contained HTML file using plain HTML, CSS and JavaScript. Load Newsreader and Be Vietnam Pro from Google Fonts, and draw icons as inline SVG in the Lucide style (24px grid, 2px stroke, round caps). Use only standard browser APIs plus the artifact platform's connector, download and storage calls.
- **Tokens:** define them as CSS custom properties on `:root`. Apply dark values under `@media (prefers-color-scheme: dark)`, and give `body` an explicit background.

| Token | Light | Dark | Use |
| --- | --- | --- | --- |
| `surface-base` | #f7f5f1 | #171614 | Page background |
| `surface-subtle` | #f2efea | #1f1d1b | Wells, alternate-week bands |
| `surface-panel` | #ece8e1 | #262320 | Grouped blocks, the collapsed header strip |
| `surface-raised` | #ffffff | #34302d | Cards, drawer, dialogs, menus |
| `text-primary` | #1a1917 | #f5f3ef | Body and headings; the Today line |
| `text-secondary` | #4f4a43 | #d0c8bd | Captions, table cells |
| `text-tertiary` | #6b655d | #b4aba0 | Eyebrows, metadata, source lines |
| `text-link` | #1a6b30 | #8cd0a0 | Links (underline on hover) |
| `border-subtle` | #ebe7df | #3b3632 | Rows and cells inside a card; lane lines |
| `border-default` | #ddd8ce | #4c4742 | Card, tile and table outlines |
| `border-strong` | #8d877f | #8a8279 | Every control edge: inputs, selects, outline buttons |
| `action-primary-bg` / `-fg` / `-hover` | #0b6623 / #ffffff / #155826 | #57bd76 / #06210d / #7ecb93 | Primary buttons |
| `interactive-hover-bg` | rgba(26,107,48,.08) | rgba(140,208,160,.10) | Hover |
| `interactive-selected-bg` | rgba(26,107,48,.12) | rgba(140,208,160,.14) | Selected tab, pressed toggle, selected view, selected chip |
| `focus-ring-color` | #1a6b30 | #8cd0a0 | 2px focus ring at 2px offset on every control |
| `status-success` / `-bg` | #127a45 / #edf8f1 | #47d78b / rgba(71,215,139,.16) | Verdict when the plan covers the goal; "ahead of plan" |
| `status-warning` / `-bg` | #9c6110 / #fff7e7 | #f0c36a / rgba(240,195,106,.16) | Verdict when short; "behind plan"; the over-total note |
| `status-error` / `-bg` | #b73847 / #fdf0f2 | #f28c95 / rgba(242,140,149,.12) | Invalid field values only |
| `accent-editorial-500` | #996345 | #bc825c | Key date lines and labels |
| Program: club | #7a4d8c | #c9a3d9 | Chips, goal bar, legend |
| Program: ecommerce | #305fc9 | #8cb5ff | Chips, goal bar, legend |
| Program: corporate gifting | #1d7373 | #7fcfcf | Chips, goal bar, legend |
| Program: private client | #2f3642 | #e3dccf | Chips, goal bar, legend |
| Program: tasting room | #8a6a12 | #e3c46a | Goal bar, legend |
| Program: other | #a03d5c | #f29ab3 | Chips, goal bar, legend |

- **Color meaning:**
  - Green is for actions and success.
  - Copper marks key dates.
  - The six program colors appear only on program data.
  - Status colors appear only with an icon and a word.
  - Needs-numbers markers use the neutral badge (`surface-panel` ground, `text-secondary` text).
- **Type:**

| Style | Font | Use |
| --- | --- | --- |
| `display-lg` | Newsreader 32px / 1.25, weight 300, −0.02em | Page heading |
| Verdict | Newsreader 24px, weight 300, in its status color | Headings stay at weight 300 |
| `body-md` | Be Vietnam Pro 16px / 1.5 | Paragraphs (How to use) |
| `body-sm` | Be Vietnam Pro 14px / 1.5 | UI default: labels, table cells, buttons |
| `numeric` | Be Vietnam Pro 500, tabular figures | Every figure; headline numbers at 28px |
| `label-caps` | Be Vietnam Pro 12px, 600, +0.12em, uppercase | Eyebrows above numbers ("PLANNED REVENUE") |

  Heading weight and tracking are tokens: `--display-weight` (300) and `--display-tracking` (−0.02em). Write everything else in sentence case.
- **Spacing:** 4px grid: 8px inside a control, 12px between sibling controls, 24px card padding and page inset, 32px and up between sections. Max width 1400px.
- **Alignment:** in every table and form row, labels, inputs and figures align to the top of the row; figures are right-aligned.
- **Radii:** `--radius-chip` 6px (chips' right corners, checkboxes, tooltips); `--radius-control` 10px (buttons, selects); `--radius-input` 10px (inputs); `--radius-card` 16px (cards, the drawer, dialogs); full only on pills and badges. Chips have square left corners (`border-radius: 0 var(--radius-chip) var(--radius-chip) 0`), so their left edges line up cleanly across rows.
- **Elevation:** cards take a `border-default` hairline plus `0 6px 18px rgba(26,25,23,.07)`, which menus and toasts reuse as `--shadow-pop`. The drawer and dialogs take `0 24px 64px rgba(26,25,23,.12)` above a `rgba(26,25,23,.44)` scrim.
- **Motion:** 120ms for color, 160ms for movement (the drawer slide, the month scroll), 200ms at most, easing `cubic-bezier(0.2,0,0.2,1)`. All durations are 0ms under `prefers-reduced-motion`.
- **Chips:** the program color at 12% opacity over `surface-raised`, a 3px left border in the full program color, and `text-primary` text.
- **Targets:** every clickable element is at least 24px tall, and at least 44px on a phone.
- **Screens:**
  - **Desktop, beside a chat:** most people view the page in the artifact pane next to the chat, about 600–900px wide. Everything outside the calendar fits that width without sideways scrolling; the goal header's numbers wrap to two rows below 760px.
  - **Desktop, full width:** legible at 1440px, where the Quarter view shows all 13 weeks without scrolling.
  - **Phone (390px):** the verdict comes first, with 16px side gutters and 88px lane labels. Modules stack, and only the calendar scrolls sideways (inside its own frame).
- **Voice:** address the reader as "you," plain and winery-literate, the way a sharp DTC director talks to their team. When something fails, say how to recover. Write "New Vintage" in full.

### Commerce7 look

A second look the user picks under More → Look. It swaps token values and a few component styles; numbers, words and color meanings stay the same.

- **Control:** in the More menu, a "Look" group with two `menuitemradio` items, "New Vintage" and "Commerce7," the chosen one checked. New Vintage is the default.
- **Behavior:** Commerce7 sets `data-look="commerce7"` on `<html>`; New Vintage removes it. The choice saves under `holiday-planner.look` (with try/catch), and a script in `<head>` applies it before first paint. Load Inter only when Commerce7 is first chosen.
- **Tokens:** apply these under `:root[data-look="commerce7"]`, with the dark values under the same dark-mode pattern:

| Token | Light | Dark |
| --- | --- | --- |
| `surface-base` / `-subtle` / `-panel` / `-raised` | #ffffff / #f6f7f9 / #eff1f4 / #ffffff | #161c27 / #121721 / #292f3d / #1d232f |
| `text-primary` / `-secondary` / `-tertiary` / `-link` | #161c27 / #4d5361 / #5f6675 / #0363a6 | #eff1f4 / #cdd0d6 / #9da3ae / #42acf0 |
| `border-subtle` / `-default` / `-strong` | #dddfe4 / #cdd0d6 / #7d8492 | #343946 / #4d5361 / #7a8190 |
| `action-primary-bg` / `-fg` / `-hover` | #0363a6 / #ffffff / #054483 | same, plus a #2490d6 edge |
| `interactive-hover-bg` / `-selected-bg` | #ebf6fe / #ddeffd | rgba(255,255,255,.05) / rgba(255,255,255,.08) |
| `focus-ring-color` | #0363a6 | #42acf0 |
| `status-success` / `-bg` | #1b7864 / #e4f2ef | #3fbf9f / rgba(35,156,130,.16) |
| `status-warning` / `-bg` | #9a5300 / #ffeedb | #f79448 / rgba(247,148,72,.14) |
| `status-error` / `-bg` | #b13434 / #fceff0 | #ec7a7a / rgba(223,95,95,.14) |
| `accent-editorial-500` (key dates) | #0e7490 | #4fc3dc |
| Program: club / ecommerce / corporate gifting / private client | #b0246b / #1030ae / #5c5f00 / #4d5361 | #f07ab5 / #8fa1ff / #cfc86a / #cdd0d6 |
| Program: tasting room / other | #7a3fb0 / #2f7a6b | #c39af0 / #6fd1bd |
| Fonts | Inter 400/500/600 for body and headings | same |
| Radii: chip / control / input / card | 4px / 4px / 3px / 6px | same |
| Shadows: card / popover / dialog and drawer | `0 1px 4px rgba(0,0,0,.1)` / `2px 4px 6px rgba(0,0,0,.15)` / `0 8px 24px rgba(0,0,0,.2)` | opacities .6 / .5 / .6 |

- **Component styles:**
  - **Headings:** Inter weight 500 with no tracking. Page title 30px, verdict 26px, card, drawer and dialog titles 20px. On a phone: title 18px, verdict 20px. Body text is 14.5px.
  - **Buttons:**
    - Primary: `action-primary-bg`, 36px tall, 15px text.
    - Secondary: a blue outline (#2490d6) with `text-link` text on white.
    - Cancel in dialogs and Delete in the drawer: text buttons (Delete in `status-error`).
  - **Inputs:** 40px tall, 3px radius.
  - **Tabs:** folder tabs. Inactive tabs sit on `surface-subtle` with a `border-default` edge; the active tab joins the page (`surface-base`, no bottom edge).
  - **Tables and calendar headers:** month and week headers, and table heads, are uppercase 12–13px at weight 600. Table rows stripe odd rows with `surface-subtle` and hover with `interactive-hover-bg`.
  - **Info banners and the hint bar:** a `#ebf6fe` ground (dark: 12% blue) with a 4px left rule in `text-link`.
  - **Chips:** a 4px left border, square left corners.
  - **Logo:** the New Vintage pill is the only logo.

---

## Example winery

Shown by the example view (see **Example view**). Checks 1–14 run on it.

- **Plan settings:** season 2026 · goal $1,300,000 · baseline growth 0% · the New Vintage median lane defaults · no program needs Setup.
- **Last year:** total $1,180,000.
  - By program: club $640,000 · ecommerce $230,000 · corporate gifting $70,000 · private client $60,000 · tasting room $140,000 · other $40,000.
  - By week (weeks 1–13): 30,000 · 610,000 · 70,000 · 45,000 · 38,000 · 40,000 · 60,000 · 95,000 · 60,000 · 45,000 · 40,000 · 30,000 · 17,000.
  - Tasting room by week (the baseline): 9,000 · 10,000 · 10,000 · 10,000 · 11,000 · 11,000 · 12,000 · 14,000 · 13,000 · 12,000 · 11,000 · 10,000 · 7,000.
  - Other by week (the baseline): 2,000 · 3,000 · 3,000 · 3,000 · 3,000 · 3,000 · 3,000 · 4,000 · 4,000 · 4,000 · 3,000 · 3,000 · 2,000.

```csv
id,lane,startWeek,weeks,title,program,audience,offer,listSize,conversionPct,aov,billsClubShipment,actualRevenue,actualOrders
t1,camp,0,2,Fall release,ecom,All contacts,"Story first, new vintages",,,,false,,
t2,camp,2,2,Cellar sale,ecom,All contacts,"Mystery cases, library bottles",,,,false,,
t4,camp,6,4,Holiday gifting & entertaining,ecom,All contacts,Gift packs from $95,,,,false,,
t5,camp,10,2,Last call & gift cards,ecom,All contacts,Free 2-day over $250,,,,false,,
e1,email,1,1,Fall release,ecom,Past purchasers,Free shipping on 6+,8400,0.35,420,false,,
e2,email,2,1,Corporate gift program,corp,"Corporate buyers, 3 yrs","Volume pricing on 12+, order by Nov 6",340,6,1450,false,,
e3,email,2,1,Cellar sale opens,ecom,All contacts,Mystery case $199,26500,0.15,199,false,,
e5,email,4,1,Thanksgiving wines,ecom,All contacts,Pairing three-pack,26500,0.12,165,false,,
e7,email,6,1,Holiday gift guide,ecom,All contacts,Shipping included over $250,26500,0.2,260,false,,
e8,email,7,1,Black Friday,ecom,"All contacts, excl. Top 500",20% off 6+ bottles,26000,0.3,230,false,,
e9,email,8,1,Cyber Monday,ecom,"Past purchasers, excl. club",30% off + $5 shipping,6300,0.4,210,false,,
e11,email,9,1,Last chance for gift packs,ecom,"All contacts, excl. this season's buyers",Shipping included over $250,23900,0.16,250,false,,
s1,sms,7,1,"Black Friday, 9am",ecom,SMS subscribers,20% off 6+ bottles,3800,0.45,220,false,,
s2,sms,10,1,48 hours to ground cutoff,ecom,SMS subscribers,Free 2-day over $250,3800,0.3,210,false,,
m1,outreach,3,1,Holiday catalogue in home,pc,Top 500 lifetime,"Reserve allocation, gift packing",500,14,640,false,,
m2,outreach,5,2,"Personal calls, top 100",pc,Top 100 lifetime,"Hand-picked gift list, we ship",100,30,950,false,,
c3,club,1,1,Fall shipment bills,club,Club members,Shipment,2100,88,295,true,,
c5,club,5,1,Member holiday add-on,club,Club members,30% off add-ons,,11,265,false,,
v1,event,1,1,Fall pickup party,club,"Club members, local",Free to members,900,25,145,false,,
v3,event,8,1,Holiday open house & gift wrap,other,Local customers,Free tasting with purchase,1200,15,160,false,,
```

### Import check

Check 8's import test, in the template's format:

```csv
Title,Lane,Start week,Weeks,Audience,Program,Offer,List size,Conversion %,Average order,Bills club shipment,Actual revenue,Actual orders
Gift card reminder,Email,Dec 14,1,All contacts,Ecommerce,Digital gift card,26500,0.12,125,No,,
Corporate follow-up calls,Direct outreach,Oct 26,2,"Corporate buyers, 3 yrs",Corporate gifting,Volume pricing on 12+,340,12,1450,No,,
Holiday card,,Dec 14,1,All contacts,,Season's greetings,,,,No,,
```

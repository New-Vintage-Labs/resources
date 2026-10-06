# Holiday Campaign Planner — build spec v0.5

*For the winery: upload this file to a new chat in Claude or ChatGPT and say "Build my holiday planner." Have your holiday calendar handy (a spreadsheet, doc or typed list). When you're ready to make the numbers live, the companion file UPGRADE_0.5.md does that.*

You are building a one-page holiday campaign planner for a winery's DTC team, covering **October–December only**. It answers one question: **does our October–December plan reach our revenue goal, and how is it tracking?** The team types the plan and its actuals; the page calculates everything else and labels which is which. It is a starting point the team makes their own, on the page itself or by asking you in chat.

Work through the **Steps** in order. Each step ends on a **Done when** line; finish it before starting the next. The **Reference** sections define the page and the rules. The page follows them exactly, and every label, total and explanation on it uses the words in **Terms**.

**When the user moves on without confirming.** If the user skips a confirmation (answers something else, or says "keep going"), don't ask again. Use what you proposed, mark each unconfirmed item **"Assumed — not confirmed"** in the plan, and list those items in the step 5 handover so the user can correct them.

---

## Steps

### 1. Ask the setup questions

Use the words below exactly, so a first-time user never meets a term without its meaning. A question tool allows at most four options per question; every question below fits.

**If you have a multiple-choice question tool** (AskUserQuestion or ask_user_input in Claude, request_user_input in ChatGPT), use it:

1. **First call, one question.** Header "Start." Question: "How would you like to start?"

   | Option | Description |
   | --- | --- |
   | See the example first (recommended) | "Opens a finished example winery plan you can explore, then replace with your own." |
   | Set up my own plan | "I'll ask a few questions, then build your plan from your calendar." |

   If they pick the example, skip to step 3 with Example winery.
2. **Second call, three questions:**

   **a. Header "Last year."** Question: "Do you have last year's October–December revenue? It lets the planner compare this plan with last season."

   | Option | Description |
   | --- | --- |
   | Season total only | "One number, from your sales report" |
   | Total by program | "Also split into club, ecommerce, corporate gifting, private client and other" |
   | Total by program and week | "Also turns on 'last year at this point' in the weekly pacing line" |
   | Not now | "You can add it later from the page" |

   **b. Header "Cutoffs."** Question: "Use the usual holiday deadlines? They're marked on the calendar so you can plan around them."

   | Option | Description |
   | --- | --- |
   | Use the usual dates (recommended) | "Corporate gift orders by Nov 6, ground shipping Dec 15, 2-day shipping Dec 21" |
   | I'll give mine | "I'll ask for your three dates" |

   **c. Header "Assumptions."** Question: "Use our recommended planning assumptions? The planner estimates revenue from typical conversion rates and discounts overlapping offers. You can change any of it later on the page."

   | Option | Description |
   | --- | --- |
   | Use recommended | "Typical conversion and order size for each channel, and a 15% overlap discount" |
   | Review them now | "I'll walk you through each one" |

3. **Only if they chose "Review them now,"** make a third call with two questions:

   **a. Header "Overlap."** Question: "Many customers get more than one offer, so adding every offer up overstates revenue. How much should the plan discount offer revenue? Club shipments and events are never discounted."

   | Option | Description |
   | --- | --- |
   | 15% (recommended) | "Typical for a winery sending 2–3 promotions a month" |
   | 10% | "Few overlapping sends" |
   | 20% | "A heavy promotion calendar" |
   | None | "Count every offer in full" |

   **b. Header "Rates."** Question: "Use typical conversion rates and order sizes per channel?" followed by each lane's values from **Lane defaults** ("Email 0.27% at $417, SMS 0.74% at $344, Direct outreach 20% at $800, club shipments 85% at $300, events 20% at $150").

   | Option | Description |
   | --- | --- |
   | Use these | "You can change any of them later" |
   | I'll give mine | "I'll ask for yours, channel by channel" |

4. **Then send one plain message** asking for the typed values:
   - winery name
   - revenue goal ("your October–December DTC revenue target")
   - actual revenue booked so far, and the last week it covers
   - last year's figures, at the detail they chose
   - any custom cutoffs or rates
   - their calendar, pasted or uploaded, with any results they already have per activity

**Otherwise** (no question tool), ask all of the above in one numbered message, using the same questions, descriptions and defaults, so the user can reply "use the example" or "defaults are fine."

**Done when:** either the user chose the example, or every question above has an answer or an accepted default and you have their calendar.

### 2. Map the calendar into activities

Turn each calendar item that falls in October–December into an activity with every field in **Activity fields**. Infer lane, start week and number of weeks from dates and channel words:

| Words | Lane |
| --- | --- |
| "eblast", "newsletter" | Email |
| "text" | SMS |
| "calls", "mailer", "catalogue" | Direct outreach |
| "pickup party", "tasting", "open house" | Events & tasting room |

Keep the user's titles. Where the calendar gives results for an activity that has already run, record them as that activity's actuals. Events that belong to no other program go in **other**.

Then show the user:
1. a compact table of what you mapped
2. a short list of anything you couldn't place, guessed, or left out because it falls outside October–December

Ask them to confirm or correct it in one reply. Leave list size, conversion and average order blank where the calendar doesn't give them. Those activities show as incomplete, which becomes the team's to-do list.

**Done when:** every calendar item is either an activity the user has confirmed (or marked "Assumed — not confirmed"), or on the list the user has answered.

### 3. Build the live artifact

Build the page to **Page**, **Rules** and **Look**, and publish it where connectors can reach it later:

- **In Claude:** a live artifact, one that can use connectors and persistent storage. Grant it the file-download capability (for example `downloads`: `downloads.save({ filename, data })`, which asks the viewer to confirm), because a page can't start a download any other way; Download CSV and Download the template use it.
- **In ChatGPT:** a Site where Sites is available. Business and Enterprise workspaces can connect live data to it later. Otherwise, use a canvas page. Downloads use a standard download link there.

Either way, include Example winery behind **More → Reset to example**, and load the user's confirmed plan (or the example) as the starting data, with `planRevision: 1`. Build both looks (New Vintage and Commerce7, see **Look**), with New Vintage as the default.

**Done when:** the page has the season bar, goal header, the calendar with all three views, both filters and the stepper, the activity drawer, all four dialogs (Last year, Start my own, Import activities, Connect live data), the How to use tab, both looks and every row of **States**, and it opens on the Plan tab in the Quarter view with the user's data loaded.

### 4. Check the numbers

Run the Rules on Example winery, using code if you can run it and otherwise tracing the calculation. Confirm the page's code produces all of the following:

1. Planned revenue **$1,012,823**. The verdict reads **"The plan is $287,177 short of the goal."**
2. By program (planned): club $659,745 · ecommerce $253,059 · private client $62,305 · corporate gifting $37,714. These add up to $1,012,823.
3. **"15 of 16 activities have revenue inputs."**
4. Against last year ($1,180,000): goal **+10%**, plan **−14%**.
5. With total actual revenue $800,000 booked through the week of Nov 9:
   - pacing reads **"$23,189 ahead of plan"**
   - last year at this point reads **$833,000**
6. The Fall shipment bills chip shows **$595K**.
7. **Activity actuals:** Fall release email (e1) with $21,000 actual, and Fall shipment bills (c3) with $601,000 actual, plus the $800,000 total from check 5, gives:
   - club actual $601,000 · ecommerce actual $21,000 · Not from an activity $178,000
   - e1 reads **"+2% vs plan"** and c3 reads **"+1% vs plan"**
   - pacing is unchanged
8. **Import:** importing the **Import check** CSV with **Add to my plan**:
   - the preview reads **"2 activities ready, 1 needs a lane"**
   - after import, planned revenue is **$1,074,372**, the verdict reads **"$225,628 short,"** and corporate gifting is **$88,001**
9. **Views:** the calendar opens in the Quarter view with all 13 weeks. Month shows the weeks whose Monday falls in the chosen month, and Week shows one week as a list per lane. In Month and Week, the stepper moves one month or one week and stops at the season's ends; Quarter has no stepper, and there are no separate month buttons. The week headers stay visible while the page scrolls.
   **Key dates line up in every view:** Ground cutoff (Dec 15, 2026) sits one-seventh of the way into the week of Dec 14 in Quarter and in Month (December), and is listed under the week of Dec 14 in Week. No key date line or label is drawn outside the weeks a view shows, and the Month columns fill the frame.
10. **Filters:** deselecting SMS in the channel filter hides the SMS lane and leaves every total unchanged. Selecting only Corporate gifting in the program filter shows only corporate gifting activities, in every lane.
11. **Click to add:** in the Quarter view, clicking the empty Email cell in the week of Oct 26 opens a new activity with lane Email and start week Oct 26 filled in, with the title focused. Closing it without a title removes it.
12. **Narrow screens:** at 390px wide, and at 720px (the artifact pane beside a chat), the page itself never scrolls sideways; only the calendar frame does, and the lane labels stay in view.
13. **Logo and look:** the logo pill sits at the left end of the season bar, shows the New Vintage lockup at 26px tall in light and dark, and links exactly to the URL in **Logo**. More → Look → Commerce7 survives a reload, and checks 1–8 give the same numbers in both looks.
14. **Connect live data and How to use:** More → Connect live data opens the dialog in **Connect live data dialog**, and the How to use tab has all eight sections.

Then tell the user their own planned revenue and verdict in one sentence each.

**Done when:** all fourteen checks match exactly, and any mismatch has been fixed in the code.

### 5. Hand over

Tell the user, in six lines or fewer:
- that the **How to use** tab explains the page
- how to enter actuals, for each activity or as a season total
- that they can ask you in this chat to add or change anything
- how to download the plan as CSV
- that every typed number is theirs to keep current, and More → Connect live data explains how to make the numbers live
- every item marked "Assumed — not confirmed," if any

**Done when:** the user has the page and those notes.

---

## Later requests

These run only when the user asks, after the page is built.

### When the user asks for changes in chat

Treat any later request ("add a Black Friday SMS," "change the club list size to 2,300," "add a wholesale program," "make the chips bigger") as an edit to this page:

1. Start from the most recent plan you know. If the user has edited on the page since your last change, ask them to download the CSV and attach it, and use it as the current plan.
2. Make the change.
3. Raise `planRevision` by 1, so the page loads your update over its saved copy (see **Saving**).
4. Confirm the change in one sentence, including any total it moved.

**Done when:** the updated page loads the change, and the user has the one-sentence confirmation.

### When the user asks to connect live data

When the user asks to make the numbers live ("connect my planner to New Vintage") or to pull real numbers from Commerce7, Klaviyo, Mailchimp or RedChirp, follow **UPGRADE_0.5.md**. If it isn't in this chat, ask them to upload it from the same place they got this file. Until then, keep every number typed.

**Done when:** you have shown UPGRADE_0.5.md's step U1 summary, or the user has been asked to upload the file.

---

## Terms

- **Season**: October–December of one year: 13 weeks from the first Monday on or after Oct 1.
- **Season week**: one of the 13 weeks, named by its Monday ("Week of Oct 19"). Week 1 is the first.
- **Plan**: the calendar of activities as the team wrote it.
- **Activity**: one item on the plan, such as an email, SMS, club shipment or event, running for one or more weeks.
- **Lane**: a calendar row grouping activities by channel.
- **Direct outreach**: the lane for direct mail and personal phone calls to top customers.
- **Program**: the revenue bucket an activity rolls up to: club, ecommerce, corporate gifting, private client or other (events usually go in other).
- **Audience**: the people an activity targets, written as a label ("Club members").
- **Revenue goal**: the season's DTC dollar target.
- **Assumptions**: the numbers the plan rests on: lane defaults and the overlap discount.
- **Lane default**: the conversion % and average order an activity inherits from its lane until it has its own. The Club lane default applies only to activities that bill a club shipment.
- **Overlap discount**: one season-wide percentage taken off offer revenue, because some customers get several offers. Club shipments and events are exempt.
- **Projected revenue**: one activity's expected revenue, after the overlap discount where it applies.
- **Planned revenue**: the sum of projected revenue across complete activities.
- **Verdict**: the headline sentence saying whether planned revenue reaches the revenue goal, and by how much.
- **Activity actuals**: the revenue an activity actually brought in (and optionally its orders), typed by the team once it has run. They roll up to the activity's program.
- **Actual revenue**: all DTC revenue booked so far this season. The team types the total from their sales report; left blank, it equals the sum of activity actuals.
- **Not from an activity**: actual revenue tied to no activity: actual revenue minus the sum of activity actuals, such as tasting-room sales and unprompted web orders.
- **Booked through**: the last season week that actual revenue covers.
- **Planned to date**: the part of planned revenue scheduled up to the booked-through week.
- **Pacing**: actual revenue minus planned to date: ahead or behind plan.
- **Last year**: last season's actual October–December revenue, typed as a total, with optional program and weekly figures.
- **Look**: the page's visual style, New Vintage (default) or Commerce7, chosen in More. It changes appearance only.
- **Calculated**: a value the page works out from typed inputs. It's labeled "Calculated," while typed values show as editable fields.
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

- **At the far right:** the **More** menu:
  - **Connect live data**: opens the Connect live data dialog
  - **Download CSV**
  - **Import activities**
  - **Edit last year**
  - **Look**: two radio items, **New Vintage** (default) and **Commerce7** (see **Look**)
  - **Start my own**
  - **Reset to example** (two-step: the first click changes the label to "Click again to reset," which resets after 3 seconds)
- **Tabs,** under the bar: **Plan** and **How to use**. On first open, a one-line hint under the tabs reads "New here? Start with How to use." It closes when clicked away.

### Goal header (Plan tab)

| Element | Behavior |
| --- | --- |
| Verdict | The headline, 20px or larger, in its status color with an icon: "The plan is $287,177 short of the goal." or "The plan covers the goal with $40,000 to spare." |
| Numbers | Revenue goal (editable) · Planned revenue (Calculated) · Actual revenue. Actual revenue is editable and labeled "Total from your sales report. You update this weekly." When blank, it shows the sum of activity actuals, labeled Calculated. Under it, a "Booked through" label sits above a dropdown of weeks ("Week of Nov 2"). |
| Goal bar | One horizontal bar of planned revenue, split into program segments. Unlabeled marks sit on the bar: a solid tick for the revenue goal, a dashed tick for last year (when entered), and a small triangle for actual revenue. A key under the bar names each with its amount: "Goal $1,300,000 · Last year $1,180,000 · Actual $800,000." |
| Program legend | A table, one row per program: color and name, Planned, Actual (from activity actuals) and Share, every figure right-aligned. A row, **Not from an activity**, appears once actual revenue exceeds the sum of activity actuals. Programs with no planned or actual revenue collapse into one line, "Not used: Other." |
| Last-year line | "Last year: $1,180,000 · Goal +10% · Plan −14%." When last year is empty, this shows an **Add last year** link. |
| Pacing line | "$23,189 ahead of plan through the week of Nov 9 · Last year at this point: $833,000." The last-year part appears only with weekly figures. Before any actual revenue exists, it reads "Pacing starts when you enter actual revenue." |
| Needs numbers | The line "15 of 16 activities have revenue inputs," beside a button reading "Show what needs numbers (1)." The button highlights incomplete activities and dims the rest. Pressed again, it clears the highlight. |
| Assumptions | A disclosure, collapsed by default. Its header is one full-width `<button aria-expanded aria-controls>`: a chevron icon (pointing right when collapsed, down when open, rotating in 160ms), the title "Assumptions," and a one-line summary of the current values, "Email 0.27% · SMS 0.74% · Overlap 15% · Show," whose last word reads "Hide" when open. It takes the hover background and a pointer cursor, so it reads as clickable and never as an empty card. Open, it holds the "Overlap discount" slider (0–40%, step 5, with the line "Takes 15% off offers because some customers get several. Club shipments and events aren't reduced.") and the Lane defaults table. |

When the page scrolls past the header, it collapses into one sticky strip: verdict, planned revenue, revenue goal and pacing. Totals sit in an `aria-live="polite"` region. Currency inputs accept "$1,300,000" or "1300000" and reformat on blur as whole US dollars. When activity actuals add up to more than the typed total, a note under Actual revenue reads "Activity actuals add up to $X more than your total. Check the total or the activity figures."

### Last year dialog

This opens from **Add last year**, **Edit last year**, or the last-year line. It has three sections, every figure right-aligned:
- **Total** (required)
- **By program** (optional)
- **By week** (optional, 13 rows)

A note explains that weekly figures turn on "last year at this point." When the program or weekly figures don't add up to the total, the dialog shows the difference: "Weekly figures add up to $1,150,000, $30,000 less than the total."

### Calendar (Plan tab)

- **Toolbar,** above the grid, wrapping on narrow screens:
  - **View:** a segmented control, "Quarter | Month | Week," default Quarter.
  - **Stepper,** in Month and Week views only, beside the view control: previous and next arrows around the current label ("‹ December ›" in Month, "‹ Week of Dec 7 ›" in Week), stepping one month or one week and stopping at the season's ends. Quarter shows no stepper, because all 13 weeks are already on the grid. There are no separate month buttons.
  - **Layout:** one row at full width: the view control and stepper on the left; Channels, Programs, Density and **Add activity** on the right. On narrow screens it wraps to two rows, with the view control and stepper on the first.
  - **Channels:** a menu of checkboxes, one per lane, all on by default. Turning a lane off hides it from view only; totals never change. While any lane is off, the toolbar reads "Showing 5 of 7 lanes · Show all."
  - **Programs:** a menu of checkboxes, one per program, all on by default. Turning one off hides its activities in every lane; Key dates and Campaign theme activities with no program stay visible. While any is off, the toolbar reads "Showing 3 of 5 programs · Show all."
  - **Density:** a toggle button reading **Compact** while chips are full size and **Expand** once compacted (`aria-pressed`). Compact chips show one line.
  - **Add activity:** a button that opens a new activity drawer.
  - The view, channels and programs are viewer preferences, saved under `holiday-planner.view` (with try/catch), never in the plan.
- **Quarter view:** all 13 weeks, labeled by Monday date and grouped by month. Each week column takes an equal share of the frame, never narrower than 96px; when the frame is narrower than that allows (the artifact pane beside a chat, a phone), the calendar scrolls sideways inside its frame, with the lane labels pinned at the left. On open, it scrolls so the current week sits second from the left, with one week before it in view.
- **Month view:** the weeks whose Monday falls in the month (four or five). Their columns split the frame's full width after the lane labels equally, never narrower than 160px each, so no empty space trails the last week. Activities that start before or run past the month show a small arrow at that edge.
- **Week view:** one season week as a list, one section per lane: each activity as a full-width row with title, audience, program, offer, and projected and actual revenue, then that week's key dates with their dates. Each lane ends with **+ Add to {lane}**.
- **Sticky header:** the month and week header row stays in view while the page scrolls, below the collapsed goal strip.
- **Today line:** a 2px solid line in `text-primary` marks today's date while the season runs.
- **Lanes,** top to bottom:

| Lane | Revenue |
| --- | --- |
| Key dates | none (labels) |
| Campaign theme | none |
| Email | yes |
| SMS | yes |
| Direct outreach | yes |
| Club | yes |
| Events & tasting room | yes, always fixed |

- **Grid:** no vertical lines between weeks; alternate weeks take a faint band (`surface-subtle` at 50%) instead. Lanes are separated by 1px `border-subtle` horizontal lines, including the line under Key dates. Dotted lines are reserved for key dates.
- **Key dates:**
  - **Which dates:** Halloween (Oct 31), Thanksgiving (fourth Thursday of November), Black Friday, Cyber Monday, Hanukkah (its first night, from the **Hanukkah** table), Christmas, and the three cutoff dates.
  - **How they show:** each is drawn down through every lane as a **2px dotted line in `accent-editorial-500`** (copper), behind the chips.
  - **Position:** in Quarter and Month, a key date's line and label sit at its day inside its week: x = the left edge of the week's column + (days since that week's Monday ÷ 7) × the column width. Key dates, chips, the Today line, the week bands and the header all share one column geometry, recalculated whenever the view changes or the frame resizes, so a date never drifts from its week. A view draws only the key dates that fall in the weeks it shows: Month (December) draws Ground cutoff, 2-day cutoff and Christmas inside the Dec 14 and Dec 21 columns, and nothing past the last column. In Week, key dates are listed with their dates, not drawn.
  - **Labels:** in the Key dates lane, each date is a small copper pill showing its name only ("Thanksgiving"). The date is in its tooltip and accessible name. Labels that would overlap drop to a second or third row, so every label is readable (Hanukkah and Christmas share Dec 25 in some years).
  - **Editing:** users can edit, add and remove them.
- **Chips:**
  - Each activity is a `<button>` spanning its weeks. It shows the title (up to two lines, with long words hyphenated) and projected revenue in compact form ("$595K"). Once the activity has actuals, the chip shows actual revenue instead ("$601K actual").
  - A tooltip and the `aria-label` add the full title, audience, dates, and plan and actual figures.
  - Program shows as the program color **plus** a short text label.
  - Incomplete chips show "needs numbers."
  - Campaign theme chips are neutral outlines with no program fill.
- **Click to add** (Quarter and Month): every empty lane-and-week cell is a target. On hover it shows a faint "+ Add," and clicking it opens a new activity drawer with that lane and start week filled in and the title focused. Closing a new activity that still has no title and no numbers removes it. A click in an empty cell of the Key dates lane adds a key date on that week's Monday instead.
- **Stacking:** activities that overlap within a lane stack in sub-rows, each fully visible. Sort a lane's activities by start week, then longest first, and place each in the first free sub-row. Empty space in any sub-row counts as that lane's cell.
- **Capacity:** the calendar holds 60+ activities.

### Activity drawer

Clicking a chip slides a drawer in over the right side of the calendar, so the grid keeps its width. On a phone it opens as a bottom sheet. Edits apply as the user types, and totals recompute immediately. Labels, inputs and numbers align to the top of each row, and figures are right-aligned.

| Group | Fields |
| --- | --- |
| Basics | Title; lane (dropdown); start week (dropdown); number of weeks |
| Who and why | Audience (text); program (dropdown: club, ecommerce, corporate gifting, private client, other, or none); offer |
| Plan | List size; conversion % and average order, each showing "Using Email default: 0.27%" until overridden, with a "Reset to default" link |
| Club lane only | "Bills a club shipment" switch. Off, the activity has no lane default and needs its own conversion and average order. |
| Result | Projected revenue (Calculated), with one sentence explaining the overlap discount or why it's exempt |
| Actuals | Actual revenue; actual orders (optional). Once actual revenue is entered, the group shows "+2% vs plan." With orders and list size, it also shows the actual conversion % and average order. A line reads "Counts toward {program}." |
| Actions | Duplicate; Delete (two-step, like Reset) |

Key dates and Campaign theme activities show only Basics and Who and why. People move activities with the lane and start-week dropdowns. With a chip focused, Alt+←/→ moves it one week and Alt+↑/↓ one lane. Esc closes the drawer and returns focus to the chip.

### Start my own

A setup sheet with three groups:

| Group | Fields |
| --- | --- |
| Required | Winery name, revenue goal |
| Recommended | Last year, cutoff dates |
| Can wait | Actual revenue, lane defaults, overlap discount |

- **At the top:** "This clears the example's activities and last-year figures. It keeps key dates and lane defaults."
- **Submit button:** it names what's missing ("Add a revenue goal to continue") until the Required fields are filled, then reads **Start my plan**.
- **After submitting:** the empty plan shows a **Next steps** checklist that ticks itself off as the user goes:
  1. Add your first activity: click a spot on the calendar, import a spreadsheet, or ask in chat.
  2. Fill in the numbers.
  3. Add last year.
  4. Enter actuals each week.

### Import activities

A dialog whose first line says what it does: "Add activities from a spreadsheet. Each row becomes one activity on the calendar. Your goal, key dates and last year stay as they are."

- A **Download the template** link gives a CSV with the columns in **Activity fields**, using readable headers: Title, Lane, Start week, Weeks, Audience, Program, Offer, List size, Conversion %, Average order, Bills club shipment, Actual revenue, Actual orders. It includes two filled example rows.
- **Start week** accepts a date ("Oct 19") or a week number. Lane and Program accept their names as shown on the page.
- The user picks a CSV and chooses **Add to my plan** or **Replace my plan**.
- A preview follows, for example "18 activities ready, 2 need a lane," listing each problem row by row number and title before anything changes. A note says rows that need fixing are skipped.
- The button names what it will do ("Import 18 activities") and adds only the ready rows.

### Connect live data dialog

This opens from More → Connect live data. It explains the next step without doing it:

- **Intro:** "Right now you type every number. With New Vintage connected, actual revenue, last year, lane defaults and each email and SMS result fill in from Commerce7, Klaviyo, Mailchimp and RedChirp, and update when you open this page."
- **Steps:**
  1. Turn on the New Vintage connector in Claude's settings (a New Vintage Pro account).
  2. In the chat where you built this planner, say "Connect my planner to New Vintage." You may be asked to upload one more file, from wherever you got this planner.
  3. Answer a few questions, such as how your winery marks corporate gifting orders. Your activities stay as they are.
- **Note:** "You can still type over anything that syncs, and events stay typed."
- **Buttons:**
  - **About New Vintage** links to the Logo URL, with `utm_content=connect-live-data`
  - **Got it** closes the dialog

### How to use tab

A short orientation page, readable in about two minutes. It has eight sections, each two to four sentences, written for someone new to the page:

1. **What this page answers:** does the plan reach the goal, and how is it tracking?
2. **Reading the verdict and goal bar.**
3. **Finding your way around and editing:** switch between Quarter, Month and Week views, step through months or weeks with the arrows, and narrow what you see with the channel and program filters (totals don't change). Click an empty spot in a lane to add an activity right where it belongs, click a chip to edit it, and use the dropdowns to move it.
4. **Bringing in your own calendar (import):**
   1. More → Import activities → Download the template.
   2. Open it in Excel or Google Sheets. Put one activity on each row: its title, lane, start week and so on. Keep the column headings.
   3. Save or download it as CSV.
   4. Back on this page, choose Import activities, pick the file, then choose Add to my plan or Replace my plan.
   5. Check the preview, which lists any rows that need fixing, then press Import.

   Your goal, key dates and last year stay as they are.
5. **Typed vs Calculated, and your weekly habit:** what you enter and what the page works out. Each week:
   - update actual revenue and the booked-through week
   - add actuals to activities that have finished
   - check pacing
6. **Make it yours by asking:** you can ask the assistant in the chat where you built this page. For example:
   - "Add a Black Friday SMS to SMS subscribers in the week of Nov 23."
   - "Change the club shipment list size to 2,300."
   - "Add a wholesale program."

   It updates the page for you. If you've edited here since, download the CSV and attach it to your request, so it starts from your latest plan.
7. **Making the numbers live:** when you're ready, turn on the New Vintage connector in Claude and say "Connect my planner to New Vintage" in this chat. Actual revenue, last year and each email and SMS result then fill in from your systems, and you can still type over any number. More → Connect live data has the steps.
8. **Saving and sharing:**
   - changes save by themselves
   - Download CSV opens in Excel or Google Sheets

The tab ends with a **Go to my plan** button.

### Logo

The New Vintage lockup (mark and wordmark), in a white pill at the left end of the season bar:

- **Pill:** `#ffffff` background in both themes and both looks, a 1px `border-default` edge, the control radius, 6px × 14px padding, at least 40px tall. On hover, the edge moves to `border-strong` and the card shadow appears.
- **Image:** an `<img>` 26px tall with auto width (about 78px) and an empty `alt`. Embed the source exactly as given below.
- **Link:** `https://newvintage.ai/?utm_source=holiday-planner&utm_medium=referral&utm_campaign=c7-webinar-2026&utm_content=season-bar-logo&utm_term=generic-ai`, opening in a new tab with `rel="noopener"`. The accessible name is "New Vintage."

The image `src` (the SVG from newvintage.ai):

```text
data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='130 130 1717 575' fill='none'%3E%3Cg clip-path='url(%23a)'%3E%3Crect width='550' height='550' x='140' y='140' fill='%23fff' rx='275'/%3E%3Cpath fill='url(%23b)' d='M140  415a275  275  0  1  1  550  0H140'/%3E%3Cpath fill='%230b6623' d='M-27  420.9  404.2  526.2c8  1.9  3.3-7.1  11.3-5.2  7.7  1.9  16.8-1  23.3-5.5  6.5-4.5  22.7 .6  24.2-7.7s-.2-16.8-4.7-23.3c-4.5-6.5-11.6-10.6-19.5-11.5l-489.8-58zM809.7  426.4l-471.9  341.9c-12.9  9.4-28.6  12.8-44  8.8-15.4-4-29.2-15.2-38.1-30.3-8.8-15.1-11.8-32.7-7.7-48  4-15.4  14.7-27.4  29.2-34l27.9-12.8  501.6-230.7  27.9-12.8z'/%3E%3Cpath fill='%230b6623' d='m217.8  623.4-32.7  11c-15.1  5.1-31.2  3.7-44.7-4.6-13.6-8.3-23.6-22.8-27.6-39.7-4.1-16.8-1.8-34.4  6.5-47.9  8.3-13.6  21.9-22.1  37.7-24.5l34.1-5.1  613.9-92.4  34.1-5.1z'/%3E%3Cpath fill='%230b6623' d='M17.8  418  1255.6  471.5l68.8  3c8 .3  15.6-2.5  21.4-8  5.7-5.5  9.1-13.2  9.2-21.3 .2-8.1-2.8-16-8.3-21.7-5.5-5.7-13.1-8.9-21-8.9l-68.8 0-1239 .4-68.8 0z'/%3E%3C/g%3E%3Cpath fill='%23222' d='M1154  200.1h37.2l31.1  118.8h.6l29.9-118.8h35.4l28.6  118.8h.6l32.3-118.8h35.7l-49.9  159.1h-36l-29.5-118.2h-.6l-29.2  118.2h-36.9zM1111.3  265.6a53.5  53.5  0  0  0-3.7-16c-1.8-5.1-4.5-9.5-8-13.2-3.3-3.9-7.4-7-12.3-9.2-4.7-2.5-10.1-3.7-16-3.7q-9.2  0-16.9  3.4a37.5  37.5  0  0  0-12.9  8.9c-3.5  3.7-6.4  8.1-8.6  13.2-2  5.1-3.2  10.7-3.4  16.6zm-81.8  23.1q0  9.2  2.5  17.8c1.8  5.7  4.5  10.8  8  15.1  3.5  4.3  7.9  7.8  13.2  10.5q8  3.7  19.1  3.7c10.2  0  18.5-2.2  24.6-6.5q9.5-6.8  14.2-20h33.2q-2.8  12.9-9.5  23.1c-4.5  6.8-9.9  12.5-16.3  17.2q-9.5  6.8-21.5  10.2c-7.8  2.5-16  3.7-24.6  3.7-12.5  0-23.6-2.1-33.2-6.2-9.6-4.1-17.8-9.8-24.6-17.2-6.6-7.4-11.6-16.2-15.1-26.5q-4.9-15.4-4.9-33.8  0-16.9  5.2-32c3.7-10.3  8.8-19.2  15.4-26.8  6.8-7.8  14.9-13.9  24.3-18.5  9.4-4.5  20.1-6.8  32-6.8q18.8  0  33.5  8  15.1  7.7  24.9  20.6c6.6  8.6  11.3  18.6  14.2  29.8q4.6  16.6  2.5  34.5zM830  200.1h33.2v23.4l.6 .6q8-13.2  20.9-20.6  12.9-7.7  28.6-7.7  26.2  0  41.2  13.5t15.1  40.6v109.2h-35.1V259.1q-.6-18.8-8-27.1-7.4-8.6-23.1-8.6-8.9  0-16  3.4-7.1  3.1-12  8.9-4.9  5.5-7.7  13.2t-2.8  16.3v93.8H830zM1801.2  537.6a53.2  53.2  0  0  0-3.7-16c-1.8-5.1-4.5-9.5-8-13.2q-4.9-5.8-12.3-9.2c-4.7-2.5-10.1-3.7-16-3.7-6.2  0-11.8  1.1-16.9  3.4a37.5  37.5  0  0  0-12.9  8.9c-3.5  3.7-6.4  8.1-8.6  13.2-2  5.1-3.2  10.7-3.4  16.6zm-81.8  23.1q0  9.2  2.5  17.8c1.9  5.7  4.5  10.8  8  15.1q5.2  6.5  13.2  10.5  8  3.7  19.1  3.7c10.3  0  18.5-2.2  24.6-6.5q9.5-6.8  14.2-20h33.2q-2.8  12.9-9.5  23.1c-4.5  6.8-9.9  12.5-16.3  17.2q-9.5  6.8-21.5  10.2c-7.8  2.5-16  3.7-24.6  3.7-12.5  0-23.6-2.1-33.2-6.2s-17.9-9.8-24.6-17.2c-6.6-7.4-11.6-16.2-15.1-26.5q-4.9-15.4-4.9-33.8c0-11.3  1.7-21.9  5.2-32q5.5-15.4  15.4-26.8  10.2-11.7  24.3-18.5  14.2-6.8  32-6.8  18.8  0  33.5  8c10.1  5.1  18.4  12  24.9  20.6  6.6  8.6  11.3  18.6  14.2  29.8q4.6  16.6  2.5  34.5zM1661.3  622.8q0  36-20.3  53.5c-13.3  11.9-32.6  17.8-57.9  17.8q-12  0-24.3-2.5c-8-1.6-15.4-4.4-22.2-8.3-6.6-3.9-12.1-9-16.6-15.4-4.5-6.4-7.2-14.2-8-23.4h35.1c1  4.9  2.8  8.9  5.2  12  2.5  3.1  5.3  5.4  8.6  7.1  3.5  1.8  7.3  3  11.4  3.4  4.1 .6  8.4 .9  12.9 .9q21.2  0  31.1-10.5  9.8-10.5  9.8-30.2v-24.3h-.6q-7.4  13.2-20.3  20.6-12.6  7.4-27.4  7.4c-12.7  0-23.6-2.2-32.6-6.5-8.8-4.5-16.2-10.6-22.2-18.2-5.7-7.8-9.9-16.7-12.6-26.8-2.7-10.1-4-20.8-4-32.3q0-16  4.9-30.5c3.3-9.6  8-18.1  14.2-25.2q9.2-11.1  22.5-17.5c9-4.3  19.2-6.5  30.5-6.5q15.1  0  27.7  6.5c8.4  4.1  14.9  10.7  19.4  19.7h.6v-21.8h35.1zm-77.8-19.4c7.8  0  14.4-1.5  19.7-4.6q8.3-4.9  13.2-12.6c3.5-5.3  6-11.3  7.4-17.8q2.5-10.2  2.5-20.3  0-10.2-2.5-19.7c-1.6-6.4-4.2-12-7.7-16.9q-4.9-7.4-13.2-11.7c-5.3-2.9-11.8-4.3-19.4-4.3-7.8  0-14.4  1.6-19.7  4.9s-9.6  7.6-12.9  12.9q-4.9  7.7-7.1  17.8-2.2  9.8-2.2  19.7c0  6.6 .8  13  2.5  19.4q2.5  9.2  7.4  16.6c3.5  4.9  7.8  8.9  12.9  12q8  4.6  19.1  4.6M1479.5  595.8c0  4.3 .5  7.4  1.5  9.2  1.2  1.8  3.5  2.8  6.8  2.8h3.7c1.4  0  3.1-.2  4.9-.6v24.3c-1.2 .4-2.9 .8-4.9  1.2a49  49  0  0  1-5.8  1.5c-2 .4-4.1 .7-6.2 .9s-3.8 .3-5.2 .3c-7.2  0-13.1-1.4-17.8-4.3q-7.1-4.3-9.2-15.1c-7  6.8-15.6  11.7-25.8  14.8-10.1  3.1-19.8  4.6-29.2  4.6-7.2  0-14.1-1-20.6-3.1-6.6-1.8-12.4-4.6-17.5-8.3q-7.4-5.8-12-14.5c-2.9-5.9-4.3-12.8-4.3-20.6  0-9.8  1.8-17.8  5.2-24  3.7-6.2  8.4-11  14.2-14.5  6-3.5  12.5-5.9  19.7-7.4a208  208  0  0  1  22.2-3.7c6.3-1.2  12.4-2.1  18.1-2.5  5.7-.6  10.8-1.5  15.1-2.8  4.5-1.2  8-3.1  10.5-5.5  2.7-2.7  4-6.6  4-11.7q0-6.8-3.4-11.1c-2-2.9-4.7-5-8-6.5-3.1-1.6-6.6-2.7-10.5-3.1-3.9-.6-7.6-.9-11.1-.9-9.8  0-17.9  2.1-24.3  6.2q-9.5  6.2-10.8  19.1h-35.1q.9-15.4  7.4-25.5c4.3-6.8  9.8-12.2  16.3-16.3q10.2-6.2  22.8-8.6c8.4-1.6  17-2.5  25.9-2.5  7.8  0  15.5 .8  23.1  2.5s14.4  4.3  20.3  8c6.2  3.7  11.1  8.5  14.8  14.5  3.7  5.7  5.5  12.8  5.5  21.2zm-35.1-44.3c-5.3  3.5-11.9  5.6-19.7  6.5-7.8 .6-15.6  1.6-23.4  3.1a64  64  0  0  0-10.8  2.8c-3.5  1-6.6  2.6-9.2  4.6-2.7  1.8-4.8  4.4-6.5  7.7q-2.2  4.6-2.2  11.4  0  5.8  3.4  9.8a27  27  0  0  0  8  6.5c3.3  1.4  6.8  2.5  10.5  3.1  3.9 .6  7.4 .9  10.5 .9  3.9  0  8.1-.5  12.6-1.5a40.1  40.1  0  0  0  12.6-5.2c4.1-2.5  7.5-5.5  10.2-9.2  2.7-3.9  4-8.6  4-14.2zM1230.9  472.1h26.5v-47.7h35.1v47.7h31.7v26.2h-31.7v84.9c0  3.7 .1  6.9 .3  9.5 .4  2.7  1.1  4.9  2.1  6.8  1.2  1.8  3  3.3  5.2  4.3  2.3 .8  5.3  1.2  9.2  1.2h7.4q3.7-.3  7.4-1.2v27.1c-3.9 .4-7.7 .8-11.4  1.2-3.7 .4-7.5 .6-11.4 .6-9.2  0-16.7-.8-22.5-2.5q-8.3-2.8-13.2-7.7c-3.1-3.5-5.2-7.8-6.5-12.9-1-5.1-1.6-11-1.9-17.5v-93.8h-26.5zM1073.3  472.1h33.2v23.4l.6 .6c5.3-8.8  12.3-15.7  20.9-20.6  8.6-5.1  18.1-7.7  28.6-7.7  17.4  0  31.2  4.5  41.2  13.5  10.1  9  15.1  22.6  15.1  40.6v109.2h-35.1V531.2c-.4-12.5-3.1-21.5-8-27.1-4.9-5.7-12.6-8.6-23.1-8.6-6  0-11.3  1.1-16  3.4q-7.1  3.1-12  8.9c-3.3  3.7-5.8  8.1-7.7  13.2q-2.8  7.7-2.8  16.3v93.8h-35.1zM1003.1  411.5h35.1v33.2h-35.1zm0  60.6h35.1v159.1h-35.1zM830  472.1h38.2l40.3  122.2h.6l38.8-122.2h36.3l-56.9  159.1h-39.4z'/%3E%3Cdefs%3E%3CradialGradient id='b' cx='0' cy='0' r='1' gradientTransform='matrix(0 -252.5 245.861 0 415 392.5)' gradientUnits='userSpaceOnUse'%3E%3Cstop stop-color='%23ffc720'/%3E%3Cstop offset='1' stop-color='%23ffbf00' stop-opacity='.96'/%3E%3C/radialGradient%3E%3CclipPath id='a'%3E%3Crect width='550' height='550' x='140' y='140' fill='%23fff' rx='275'/%3E%3C/clipPath%3E%3C/defs%3E%3C/svg%3E
```

### Activity fields

`id, title, lane, startWeek (0–12), weeks, audience, program, offer, listSize, conversionPct, aov, billsClubShipment, actualRevenue, actualOrders, assumed`

`assumed` is true when the user moved on without confirming the activity.

Lane ids: `key` Key dates · `camp` Campaign theme · `email` Email · `sms` SMS · `outreach` Direct outreach · `club` Club · `event` Events & tasting room. Program ids: `club`, `ecom` ecommerce, `corp` corporate gifting, `pc` private client, `other` other.

### Saving

- **What's saved:** the whole plan is one object:

  `{ schemaVersion: 5, planRevision, winery, goal, actual, actualThroughWeek, overlapDiscountPct, seasonStart, laneDefaults, keyDates, lastYear: { total, byProgram, byWeek }, activities }`

  Selection, filters, the view and the open drawer stay out of it. The look and the view are viewer preferences, saved separately (see **Calendar** and **Commerce7 look**).
- **Where:** the platform's persistent artifact storage (`window.storage` in Claude) when available; otherwise browser `localStorage` under the key `holiday-planner`. Wrap every read and write in try/catch.
- **Loading:** when the page's built-in plan has a higher `planRevision` than the saved copy, it loads the built-in plan and keeps the saved copy one step back. A banner reads "Updated from chat. **Undo**." Otherwise it loads the saved copy.
- **When saving isn't available:** show one line: "Changes won't be saved in this browser. Use Download CSV to keep a copy."
- **Older saves:** upgrade a saved plan from schemaVersion 1–4 by filling in the missing fields: `lastYear` empty, `actualThroughWeek` at the Booked-through default, activity actuals empty, `planRevision` 1. Rename `haircutPct` to `overlapDiscountPct`. A schemaVersion 4 plan needs nothing else.
- **Download CSV** exports activities in the template's columns, plus projected revenue, through the platform's download capability (see step 3). It opens in Excel or Google Sheets and imports back unchanged.

### States

| State | Treatment |
| --- | --- |
| Example loaded | A banner reads "Example winery, example figures," with a **Start my own** button |
| Empty plan | The Next steps checklist, in place of bars and chips |
| Half-filled | Totals cover complete activities; the Needs numbers button shows the count |
| Short of goal / covers goal | The verdict's status color, icon and words change |
| No last year | The last-year line becomes the Add last year link, and the goal bar drops its last-year tick |
| Filters on | The toolbar's "Showing … · Show all" lines |
| Assumed items | A one-line note above the calendar, "2 items are assumed — not confirmed," that highlights them when clicked |
| Updated from chat | The Undo banner (see Saving) |

---

## Rules

- **Season:** 13 weeks from the first Monday on or after Oct 1. `startWeek` 0 is that Monday. Assign each week to the month of its Monday.
- **Effective conversion** = the activity's own conversion %, else its lane default. **Effective average order** works the same way. A Club-lane activity that doesn't bill a shipment has no lane default.
- **Complete** = list size > 0, effective conversion > 0 and effective average order > 0. Incomplete activities count $0 of planned revenue.
- **Gross** = list size × effective conversion ÷ 100 × effective average order.
- **Fixed** = an Events & tasting room activity, or a Club activity with "Bills a club shipment" on.
- **Projected revenue** = gross for fixed activities; gross × (1 − overlap discount ÷ 100) for the rest.
- **Planned revenue** = Σ projected revenue of complete activities.
- **Gap to goal** = revenue goal − planned revenue. It drives the verdict: positive is short, zero or negative covers the goal.
- **Activity actuals:**
  - **Vs plan** = actual revenue ÷ projected revenue − 1, shown as a whole-number % with + or −.
  - **Actual conversion** = actual orders ÷ list size × 100.
  - **Actual average order** = actual revenue ÷ actual orders.
- **Program actual** = Σ activity actual revenue of the program's activities.
- **Actual revenue** = the typed total when filled; otherwise Σ activity actual revenue.
- **Not from an activity** = actual revenue − Σ activity actual revenue, never below $0. It shows only when positive.
- **Planned to date** = each activity's projected revenue spread evenly across its weeks, summed over weeks 0 through the booked-through week.
- **Pacing** = actual revenue − planned to date. Positive reads "ahead of plan"; negative reads "behind plan."
- **Booked-through default** = the last season week that ended before today, clamped to the season. Before the season starts, there is none, and pacing waits.
- **Last-year total:** the Total field when filled; otherwise the sum of the weekly figures; otherwise the sum of the program figures. Program and weekly figures stay as detail and never override a typed total.
- **Last year at this point** = the sum of last year's weekly figures for weeks 0 through the booked-through week.
- **% vs last year** = (value − last-year total) ÷ last-year total × 100, rounded to a whole number and shown with + or −.
- **Program figures** (planned and actual) are rounded to whole dollars with largest-remainder rounding, so they always add up to the rounded total.
- **Validation:**
  - weeks is a whole number from 1 to 13 − start week; values over that are clamped, with the note "Shortened to fit the season"
  - conversion is 0–100, with two decimals, kept exactly as entered (never rounded to one decimal)
  - list size, average order and actuals are 0 or more
- **Formatting:**
  - whole US dollars with comma grouping
  - compact on chips and bars ($595K, $1.3M)
  - conversion with up to two decimals, trailing zeros dropped ("0.25%," "1.8%")
  - weeks written "Week of Oct 19"
  - every figure in a table or legend right-aligned, with tabular figures

### Lane defaults

Typical values per send, as Klaviyo, Mailchimp and RedChirp report them, measured across New Vintage wineries' holiday seasons. Your own platform's campaign reports are the best source for your numbers.

| Lane | Conversion % | Average order |
| --- | --- | --- |
| Email | 0.27 | $417 |
| SMS | 0.74 | $344 |
| Direct outreach | 20 | $800 |
| Club (shipments only) | 85 | $300 |
| Events & tasting room | 20 | $150 |

Changing a lane default re-projects every activity that inherits it.

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

---

## Look

The page uses the **New Vintage Labs Design System** by default, with a **Commerce7** look the user can switch to. Everything needed is below, so the build needs nothing else. Radii, the popover shadow and heading weight are tokens too (named below), so the Commerce7 look only swaps token values.

- **Build:** one self-contained HTML file using plain HTML, CSS and JavaScript. Load Newsreader and Be Vietnam Pro from Google Fonts, and draw icons as inline SVG in the Lucide style (24px grid, 2px stroke, round caps). Use only standard browser APIs plus the platform's storage and download calls, so the page renders the same in Claude and ChatGPT.
- **Tokens:** define them as CSS custom properties on `:root`. Apply dark values under `@media (prefers-color-scheme: dark)`, and give `body` an explicit background.

| Token | Light | Dark | Use |
| --- | --- | --- | --- |
| `surface-base` | #f7f5f1 | #171614 | Page background |
| `surface-subtle` | #f2efea | #1f1d1b | Wells, alternate-week bands |
| `surface-panel` | #ece8e1 | #262320 | Grouped blocks, the collapsed header strip |
| `surface-raised` | #ffffff | #34302d | Cards, drawer, dialogs, menus |
| `text-primary` | #1a1917 | #f5f3ef | Body and headings; the Today line |
| `text-secondary` | #4f4a43 | #d0c8bd | Captions, table cells |
| `text-tertiary` | #6b655d | #b4aba0 | Eyebrows, metadata |
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
| Program: other | #a03d5c | #f29ab3 | Chips, goal bar, legend |

- **Color meaning:**
  - Green is for actions and success.
  - Copper marks key dates.
  - The program colors appear only on program data.
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
| Program: club / ecommerce / corporate gifting / private client / other | #b0246b / #1030ae / #5c5f00 / #4d5361 / #2f7a6b | #f07ab5 / #8fa1ff / #cfc86a / #cdd0d6 / #6fd1bd |
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

- **Plan settings:** goal $1,300,000 · actual blank · overlap discount 15% · lane defaults as above.
- **Last year:** total $1,180,000.
  - By program: club $640,000 · ecommerce $380,000 · corporate gifting $70,000 · private client $90,000.
  - By week (weeks 1–13): 30,000 · 610,000 · 70,000 · 45,000 · 38,000 · 40,000 · 60,000 · 95,000 · 60,000 · 45,000 · 40,000 · 30,000 · 17,000.

```csv
id,lane,startWeek,weeks,title,program,audience,offer,listSize,conversionPct,aov,billsClubShipment,actualRevenue,actualOrders
t1,camp,0,2,Fall release,ecom,All contacts,"Story first, new vintages",,,,false,,
t2,camp,2,2,Cellar sale,ecom,All contacts,"Mystery cases, library bottles",,,,false,,
t4,camp,6,4,Holiday gifting & entertaining,ecom,All contacts,Gift packs from $95,,,,false,,
t5,camp,10,2,Last call & gift cards,ecom,All contacts,Free 2-day over $250,,,,false,,
e1,email,1,1,Fall release,ecom,Past purchasers,Free shipping on 6+,8400,1.2,240,false,,
e2,email,2,1,Corporate gift program,corp,"Corporate buyers, 3 yrs","Volume pricing on 12+, order by Nov 6",340,9,1450,false,,
e3,email,2,1,Cellar sale opens,ecom,All contacts,Mystery case $199,26500,0.5,199,false,,
e5,email,4,1,Thanksgiving wines,ecom,All contacts,Pairing three-pack,26500,0.5,165,false,,
e7,email,6,1,Holiday gift guide,ecom,All contacts,Shipping included over $250,26500,0.9,260,false,,
e8,email,7,1,Black Friday,ecom,"All contacts, excl. Top 500",20% off 6+ bottles,26000,1.4,230,false,,
e9,email,8,1,Cyber Monday,ecom,"Past purchasers, excl. club",30% off + $5 shipping,6300,1.3,210,false,,
e11,email,9,1,Last chance for gift packs,ecom,"All contacts, excl. this season's buyers",Shipping included over $250,23900,0.5,250,false,,
s1,sms,7,1,"Black Friday, 9am",ecom,SMS subscribers,20% off 6+ bottles,3800,2.4,225,false,,
s2,sms,10,1,48 hours to ground cutoff,ecom,SMS subscribers,Free 2-day over $250,3800,1.5,210,false,,
m1,outreach,3,1,Holiday catalogue in home,pc,Top 500 lifetime,"Reserve allocation, gift packing",500,14,640,false,,
m2,outreach,5,2,"Personal calls, top 100",pc,Top 100 lifetime,"Hand-picked gift list, we ship",100,30,950,false,,
c3,club,1,1,Fall shipment bills,club,Club members,Shipment,2100,96,295,true,,
c5,club,5,1,Member holiday add-on,club,Club members,30% off add-ons,,11,265,false,,
v1,event,1,1,Fall pickup party,club,"Club members, local",Free to members,900,25,145,false,,
v3,event,8,1,Holiday pickup & gift wrap,club,"Club members, local",Free to members,900,20,180,false,,
```

### Import check

The Step 4 import test, in the template's format:

```csv
Title,Lane,Start week,Weeks,Audience,Program,Offer,List size,Conversion %,Average order,Bills club shipment,Actual revenue,Actual orders
Gift card reminder,Email,Dec 14,1,All contacts,Ecommerce,Digital gift card,26500,0.4,125,No,,
Corporate follow-up calls,Direct outreach,Oct 26,2,"Corporate buyers, 3 yrs",Corporate gifting,Volume pricing on 12+,340,12,1450,No,,
Holiday card,,Dec 14,1,All contacts,,Season's greetings,,,,No,,
```

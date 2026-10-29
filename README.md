# Pipeline Velocity Bottleneck Detection

Automated weekly pipeline health reporting for B2B SaaS sales teams. Detects deal stalls, identifies bottleneck stages, and quantifies revenue at risk, delivered to Slack every Friday morning.

**Current version:** v2.1. See [CHANGELOG](#changelog). Earlier versions are in [`archive/`](archive/).

## Problem

Sales leadership lacks systematic visibility into pipeline velocity. Manual reporting is time-consuming (45+ min/week), bottleneck detection is anecdotal ("Discovery feels slow"), and revenue risk isn't quantified. This delays intervention and blocks data-driven decision-making.

## Solution

An n8n automation that runs weekly (Friday, 8 AM) to:
- Load pipeline rules (stages, SLAs, activity threshold, velocity window, currency, loss probabilities) from one settings node
- Fetch HubSpot owners and all deals
- Classify each open deal's risk (HIGH/MEDIUM/OK) by **days in stage vs. SLA** combined with **days since last activity**
- Quantify revenue at risk using tiered loss probabilities (HIGH 40%, MEDIUM 15%)
- Detect bottleneck stages from **historical stage velocity**: the typical (median) time deals spent in a stage before moving on
- Run a data check that warns when many open deals have no activity logged
- Track week-over-week trends in Google Sheets
- Post a compact Slack digest that shows the most important numbers at first glance and the top 3 deals to act on

## Architecture

### Workflow Components

1. **Every Friday 8 AM** (Schedule Trigger): runs weekly; timezone comes from the workflow settings
2. **Pipeline Settings** (Edit Fields): central config for stage IDs and names, SLA days, thresholds, velocity window, currency and loss probabilities
3. **Get HubSpot Owners** (HTTP Request): maps owner IDs to names; continues with IDs if the request fails
4. **Get All HubSpot Deals** (HubSpot): fetches all deals with a defined property list, including last activity and per-stage history
5. **Calculate Days & Risk** (Code): maps stages, calculates days in stage and days since last activity, classifies risk with a reason, computes revenue at risk, extracts per-stage history, flags data issues
6. **Create Risk Summary** (Code): counts at-risk and quiet deals, splits revenue at risk into HIGH/MEDIUM, ranks the top 3 deals, runs the data check
7. **Read Trend History** (Google Sheets): reads previous weekly snapshots
8. **Calculate Trend** (Code): week-over-week comparison; reports a baseline week on the first run
9. **Calculate Stage Stats** (Code): typical time per stage (median, last 90 days), slowest/fastest stage vs. SLA, bottlenecks, aging warning
10. **Merge Results**: combines trend and stage data
11. **Post Slack Digest** (Slack): posts the compact weekly brief (no n8n footer)
12. **Save Weekly Snapshot** (Google Sheets): writes one row per week, matched on `Week Of`

### Data Flow

```
[Every Friday 8 AM]
    ↓
[Pipeline Settings] → stages, SLAs, thresholds, velocity window, loss probabilities
    ↓
[Get HubSpot Owners] → owner ID → name map
    ↓
[Get All HubSpot Deals]
    ↓
[Calculate Days & Risk] → deals + risk tier + reason + revenue at risk + stage history
    ↓                        ↓
[Create Risk Summary]    [Calculate Stage Stats] → velocity, bottlenecks, aging
    ↓                        │
[Read Trend History]         │
    ↓                        │
[Calculate Trend] → WoW      │
    ↓                        ↓
[Merge Results] ─→ [Post Slack Digest]
                └→ [Save Weekly Snapshot] → Google Sheets
```

## Configuration

All business rules live in the **Pipeline Settings** node. Edit its JSON to match your pipeline; the code nodes don't need changes.

### SLA Thresholds (Days in Stage)

| Stage | HubSpot stage ID (default pipeline) | SLA | Rationale |
|---|---|---|---|
| Qualified Lead | `appointmentscheduled` | 5 | Entry point; fast movement expected |
| Discovery | `qualifiedtobuy` | 5 | Fact-finding; should be quick |
| Proposal/Demo | `presentationscheduled` | 21 | Complex selling cycle; more time needed |
| Negotiation | `decisionmakerboughtin` | 14 | Final stage; delays indicate stalled deals |

Find your own stage IDs in HubSpot under Settings → Objects → Deals → Pipelines. Open deals in stages that aren't listed are flagged as `stage_not_in_pipeline_settings` and excluded from the risk calculation.

### Risk Classification

Risk combines how long a deal has been in its stage with whether anyone is still working it. Activity comes from HubSpot's *Last activity date* (`notes_last_updated`); a deal counts as **quiet** after `activityThresholdDays` (default 7) without activity.

| Days in stage | Quiet | Recently active |
|---|---|---|
| Past SLA | **HIGH** (40% loss probability) | **MEDIUM** (15%) |
| Past SLA × 0.7 | **MEDIUM** (15%) | OK (0%) |
| Within SLA × 0.7 | OK (0%) | OK (0%) |

The 0.7 share, the activity threshold and both probabilities are set in Pipeline Settings (`mediumThresholdShare`, `activityThresholdDays`, `lossProbabilityHigh`, `lossProbabilityMedium`).

**Note:** Loss probabilities are estimates. Validate them against your historical close rates and adjust as needed.

### Stage Velocity and Bottlenecks

A stage is a **bottleneck** when the typical time deals spend in it is above its SLA.

- **Typical time** = the median of HubSpot's `hs_v2_cumulative_time_in_<stageId>` for deals that left the stage (`hs_v2_date_exited_<stageId>`) within the last `velocityLookbackDays` (default 90).
- A stage needs at least `velocityMinSample` (default 3) such deals before a typical time is shown; otherwise the digest shows "n/a".
- Slowest and fastest stage are ranked **relative to each stage's SLA**, since stages have different expected lengths.
- **Aging** (the average days open deals have already spent in a stage) is still calculated and available in the workflow data (`agingStages`), but isn't shown in the Slack digest. At-risk deals already cover that signal.

Why velocity instead of open-deal age: averaging the age of open deals tells you how old today's pipeline is, not how fast deals actually move. HubSpot only fills the cumulative-time properties once a deal has left a stage, so velocity has to come from deal history.

### Data Check

Deals without any logged activity count as **quiet** on purpose: a stage move alone doesn't count as activity, so missing call/email logging or a stage moved without a real action stays visible instead of being hidden.

To make the cause visible, the digest adds one warning line when:
- at least `dataQualityWarningShare` (default 50%) of open deals have **no activity ever logged**, or
- deals can't be classified (stage not in Pipeline Settings, or missing stage entry date).

Example: `⚠️ Data check: 16 of 16 open deals have no activity logged. Check CRM logging.` The line disappears when the data is healthy.

## Setup

### Prerequisites

- n8n instance (self-hosted or cloud)
- HubSpot account (free tier works) with a private app
- Google account with a service account for Sheets
- Slack workspace

### Steps

1. **Import the workflow**
   - In n8n: ⋯ → Import from File → `hubspot-Pipeline-Velocity-Bottleneck-Detection-v2.1.json`
   - The JSON contains no credentials; you'll connect your own in the next steps

2. **Connect HubSpot**
   - Create a private app with scopes `crm.objects.deals.read` and `crm.objects.owners.read`
   - Add the token as a *HubSpot App Token* credential in **Get HubSpot Owners** and **Get All HubSpot Deals**

3. **Request the stage history properties**
   - In **Get All HubSpot Deals**, make sure `hs_v2_cumulative_time_in_<stageId>` and `hs_v2_date_exited_<stageId>` are requested for every stage in Pipeline Settings (the defaults cover the four stages above)
   - HubSpot can adjust stage IDs to fit property naming rules. If a property name differs from the stage ID, add `"propertyKey": "<name used in the property>"` to that stage in Pipeline Settings

4. **Set up Google Sheets**
   - Create a sheet tab named `Weekly_Trend_Tracker_Simple` with these headers in row 1: `Week Of`, `At-Risk Count`, `Previous Week Count`, `Trend %`, `Revenue at Risk`, `Slowest Stage`, `Bottleneck Count`, `Notes`
   - Format the `Week Of` column as **Plain text** (otherwise date matching fails and rows get duplicated)
   - Share the sheet with your service account as Editor
   - Paste the sheet URL into **Read Trend History** and **Save Weekly Snapshot**

5. **Connect Slack**
   - Connect your Slack account (OAuth2) in **Post Slack Digest**
   - Replace `YOUR_SLACK_CHANNEL_ID` with your channel's ID and invite the app to that channel

6. **Edit Pipeline Settings**
   - Replace the stage IDs, SLA days and currency with your own
   - Adjust `activityThresholdDays`, `velocityLookbackDays`, `velocityMinSample` and `dataQualityWarningShare` if needed

7. **Check the schedule**
   - Default: every Friday, 08:00
   - Set the timezone under workflow Settings → Timezone

8. **Test**
   - Run the workflow manually; the first run creates the baseline row
   - Verify the Slack message and the new row in Google Sheets
   - Activate the workflow

**Upgrading from v1.x:** v2.0 counts fewer deals as at risk (overdue deals that are still active drop from HIGH to MEDIUM) and measures "slowest stage" differently. Start a fresh trend sheet or clear old rows so the first week-over-week comparison isn't skewed.

## Example Output

```
📊 *Pipeline Brief · 29 Oct 2026*
*€16,250 at risk* · 3 of 4 open deals · 🔴 +1 vs last week
🔴 1 HIGH · 🟡 2 MEDIUM

⚠️ Bottleneck: Discovery (typically 7d vs 5d SLA)

*Act on these first:*
1. 🔴 *Acme Legal* · €20k · Discovery 10d (SLA 5) · quiet 12d · Lena Vogt
2. 🟡 *Gamma Tax* · €40k · Proposal/Demo 16d (SLA 21) · quiet 10d · Lena Vogt
3. 🟡 *Beta Consulting* · €15k · Discovery 8d (SLA 5) · active 2d ago · Jonas Ritter
```

When the data check or history is missing, the middle lines change, for example:

```
⚠️ Data check: 16 of 16 open deals have no activity logged. Check CRM logging.

ℹ️ Not enough history for bottlenecks yet
```

*Example data is illustrative (fictional test deals).*

## Troubleshooting

### Briefing not posting to Slack
- Check that the Slack credential in **Post Slack Digest** is still connected
- Confirm the channel ID is correct and the Slack app is invited to the channel
- Check the execution log for errors in **Post Slack Digest**

### Owner IDs shown instead of names
- Add the scope `crm.objects.owners.read` to your HubSpot private app
- Check the output of **Get HubSpot Owners**; the workflow continues with IDs if this request fails

### Deals missing from the briefing
- Check that **Get All HubSpot Deals** returns all expected deals
- Confirm the stage IDs in **Pipeline Settings** match HubSpot's internal IDs (e.g. `qualifiedtobuy`, not "Discovery")
- Look for `data_issue` flags in the output of **Calculate Days & Risk**

### Typical stage time shows "n/a"
- The stage has fewer than `velocityMinSample` deals that left it in the last `velocityLookbackDays`; lower the sample or widen the window
- Check that `hs_v2_cumulative_time_in_<stageId>` and `hs_v2_date_exited_<stageId>` are requested in **Get All HubSpot Deals** and return values
- If HubSpot uses a different property name for a stage, add `propertyKey` to that stage in **Pipeline Settings**

### Too many deals marked as quiet
- Check whether your team logs activities in HubSpot; deals with no logged activity count as quiet (the data check line tells you when this affects most deals)
- Moving a deal to another stage does **not** update HubSpot's *Last activity date*; only logged calls, emails, meetings, notes, tasks and chats do
- Raise `activityThresholdDays` if your sales cycle has longer natural gaps

### Trend not calculating or duplicate rows in Google Sheets
- Verify the Google credential and that the service account has Editor access
- Check that the tab name and headers match exactly
- Make sure the `Week Of` column is formatted as plain text

### Unexpected risk classifications
- Review the SLA days and thresholds in **Pipeline Settings**
- Days in stage are calculated from HubSpot's `hs_v2_date_entered_current_stage`; days since activity from `notes_last_updated`
- The top-deal lines in Slack show the reason for each classification

## Design Note: Revenue-at-Risk Calculation

The workflow uses tiered loss probabilities (HIGH 40%, MEDIUM 15%). Since v2.0, HIGH means a deal is both overdue and quiet, which is a stronger loss signal than deal age alone. The percentages are still assumptions: **validate them against your historical data** and adjust them in Pipeline Settings.

## Changelog

### v2.1: Compact Slack digest and data check
- Slack digest rebuilt for first-glance reading: revenue at risk, trend, tier counts, bottleneck and the top 3 deals, one line each
- Stage breakdown, slowest/fastest stage and aging lines removed from Slack (still available in the workflow data)
- New data check: warns when at least half of open deals have no activity logged, or deals can't be classified
- Deals without logged activity still count as quiet, so missing logging stays visible
- Removed the "Automated with this n8n workflow" footer
- New Pipeline Setting: `dataQualityWarningShare`
- v2.0 moved to `archive/`

### v2.0: Activity signal and historical stage velocity
- Risk tiers now combine days in stage vs. SLA with days since last activity: HIGH = past SLA and quiet; MEDIUM = past SLA but active, or close to SLA and quiet
- Top risk deals in Slack show the reason for their tier; the digest also counts quiet deals
- Bottlenecks now use historical stage velocity (median cumulative time in stage for deals that left it in the last 90 days) instead of the average age of open deals
- Slowest/fastest stage are ranked relative to each stage's SLA
- Open-deal age is kept as a separate "aging now" warning
- New Pipeline Settings: `activityThresholdDays`, `velocityLookbackDays`, `velocityMinSample`, optional per-stage `propertyKey`
- v1.3 moved to `archive/`

### v1.3: Template-ready release
- Added HubSpot owner lookup (names instead of IDs in Slack)
- Added Pipeline Settings node: stages, SLAs, currency and loss probabilities in one place
- Added data quality flags for unmapped stages and missing stage entry dates
- Added Overview & Setup sticky note; renamed nodes to descriptive names
- Removed credential references, instance IDs and company-specific comments from the JSON

## License

This workflow is provided as-is for operational use and case study documentation.

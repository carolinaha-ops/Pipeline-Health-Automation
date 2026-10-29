# Pipeline Velocity Bottleneck Detection

Automated weekly pipeline health reporting for B2B SaaS sales teams. Detects deal stalls, identifies bottleneck stages, and quantifies revenue at risk, delivered to Slack every Friday morning.

**Current version:** v2.0. See [CHANGELOG](#changelog). Earlier versions are in [`archive/`](archive/).

## Problem

Sales leadership lacks systematic visibility into pipeline velocity. Manual reporting is time-consuming (45+ min/week), bottleneck detection is anecdotal ("Discovery feels slow"), and revenue risk isn't quantified. This delays intervention and blocks data-driven decision-making.

## Solution

An n8n automation that runs weekly (Friday, 8 AM) to:
- Load pipeline rules (stages, SLAs, activity threshold, velocity window, currency, loss probabilities) from one settings node
- Fetch HubSpot owners and all deals
- Classify each open deal's risk (HIGH/MEDIUM/OK) by **days in stage vs. SLA** combined with **days since last activity**
- Quantify revenue at risk using tiered loss probabilities (HIGH 40%, MEDIUM 15%)
- Detect bottleneck stages from **historical stage velocity**: the typical (median) time deals spent in a stage before moving on
- Flag stages whose open deals are already aging past the SLA
- Track week-over-week trends in Google Sheets
- Post a formatted Slack digest with owner names and the reason each top deal is at risk

## Architecture

### Workflow Components

1. **Every Friday 8 AM** (Schedule Trigger): runs weekly; timezone comes from the workflow settings
2. **Pipeline Settings** (Edit Fields): central config for stage IDs and names, SLA days, thresholds, velocity window, currency and loss probabilities
3. **Get HubSpot Owners** (HTTP Request): maps owner IDs to names; continues with IDs if the request fails
4. **Get All HubSpot Deals** (HubSpot): fetches all deals with a defined property list, including last activity and per-stage history
5. **Calculate Days & Risk** (Code): maps stages, calculates days in stage and days since last activity, classifies risk with a reason, computes revenue at risk, extracts per-stage history, flags data issues
6. **Create Risk Summary** (Code): counts at-risk and quiet deals, splits revenue at risk into HIGH/MEDIUM, ranks the top 3 deals
7. **Read Trend History** (Google Sheets): reads previous weekly snapshots
8. **Calculate Trend** (Code): week-over-week comparison; reports a baseline week on the first run
9. **Calculate Stage Stats** (Code): typical time per stage (median, last 90 days), slowest/fastest stage vs. SLA, bottlenecks, aging warning
10. **Merge Results**: combines trend and stage data
11. **Post Slack Digest** (Slack): posts the weekly brief
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
- **Aging** is a separate, current signal: the average days open deals have already spent in a stage. If it's above the SLA, the stage is flagged as "aging now" even if its historical velocity is fine.

Why two signals: averaging the age of open deals tells you how old today's pipeline is, not how fast deals actually move. HubSpot only fills the cumulative-time properties once a deal has left a stage, so velocity has to come from deal history.

## Setup

### Prerequisites

- n8n instance (self-hosted or cloud)
- HubSpot account (free tier works) with a private app
- Google account with a service account for Sheets
- Slack workspace

### Steps

1. **Import the workflow**
   - In n8n: ⋯ → Import from File → `hubspot-Pipeline-Velocity-Bottleneck-Detection-v2.0.json`
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
   - Adjust `activityThresholdDays`, `velocityLookbackDays` and `velocityMinSample` if needed

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
📊 *Pipeline Velocity Brief*

⚠️ *Revenue at Risk:* €16,250 across 3 of 4 open deals
🔴 HIGH (past SLA, no recent activity): €8,000 | 1 deals
🟡 MEDIUM (past SLA but active, or close to SLA and quiet): €8,250 | 2 deals
💤 Quiet deals (no activity in 7+ days): 2
📈 *vs Last Week:* 🔴 +1 deals (was 2)

*By Stage:*
• Qualified Lead: 0 at risk of 0 open | typical time n/a (n=0) vs SLA 5d | open avg 0d
• Discovery: 2 at risk of 2 open | typically 7d (n=3) vs SLA 5d | open avg 9d
• Proposal/Demo: 1 at risk of 1 open | typically 12d (n=3) vs SLA 21d | open avg 16d
• Negotiation: 0 at risk of 1 open | typical time n/a (n=2) vs SLA 14d | open avg 3d

*Pipeline Health:*
• Slowest vs SLA: Discovery (typically 7 days, SLA 5)
• Fastest vs SLA: Proposal/Demo (typically 12 days, SLA 21)
_Typical = median for deals that left the stage in the last 90 days_
⚠️ 1 bottleneck (typical time above SLA):
• Discovery: typically 7 days (SLA 5, +2) | based on 3 deals
⏳ Aging now (open deals avg above SLA): Discovery (9d)

*Highest Risk Deals (Priority Order):*
1️⃣ *Acme Legal* [HIGH]
• Discovery: 10 days (SLA 5) | past SLA (+5 days), no activity for 12 days
• Amount: €20,000 | Owner: Lena Vogt
2️⃣ *Gamma Tax* [MEDIUM]
• Proposal/Demo: 16 days (SLA 21) | close to SLA, no activity for 10 days
• Amount: €40,000 | Owner: Lena Vogt
3️⃣ *Beta Consulting* [MEDIUM]
• Discovery: 8 days (SLA 5) | past SLA (+3 days), active 2 days ago
• Amount: €15,000 | Owner: Jonas Ritter
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
- Check whether your team logs activities in HubSpot; deals with no logged activity count as quiet
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

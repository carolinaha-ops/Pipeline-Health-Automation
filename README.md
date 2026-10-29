# Pipeline Velocity Bottleneck Detection

Automated weekly pipeline health reporting for B2B SaaS sales teams. Detects deal stalls, identifies bottleneck stages, and quantifies revenue at risk, delivered to Slack every Friday morning.

**Current version:** v1.3 (template-ready). See [CHANGELOG](#changelog).

## Problem

Sales leadership lacks systematic visibility into pipeline velocity. Manual reporting is time-consuming (45+ min/week), bottleneck detection is anecdotal ("Discovery feels slow"), and revenue risk isn't quantified. This delays intervention and blocks data-driven decision-making.

## Solution

An n8n automation that runs weekly (Friday, 8 AM) to:
- Load pipeline rules (stages, SLAs, currency, loss probabilities) from one settings node
- Fetch HubSpot owners and all deals
- Calculate days in current stage and classify risk (HIGH/MEDIUM/OK) against each stage's SLA
- Quantify revenue at risk using tiered loss probabilities (HIGH 40%, MEDIUM 15%)
- Detect bottleneck stages where the average time in stage exceeds the SLA
- Track week-over-week trends in Google Sheets
- Post a formatted Slack digest with owner names for leadership review

## Architecture

### Workflow Components

1. **Every Friday 8 AM** (Schedule Trigger): runs weekly; timezone comes from the workflow settings
2. **Pipeline Settings** (Edit Fields): central config for stage IDs and names, SLA days, currency, MEDIUM threshold and loss probabilities
3. **Get HubSpot Owners** (HTTP Request): maps owner IDs to names; continues with IDs if the request fails
4. **Get All HubSpot Deals** (HubSpot): fetches all deals with a defined property list
5. **Calculate Days & Risk** (Code): maps stages, calculates days in stage, classifies risk, computes revenue at risk, flags data issues
6. **Create Risk Summary** (Code): counts at-risk deals, splits revenue at risk into HIGH/MEDIUM, ranks the top 3 deals
7. **Read Trend History** (Google Sheets): reads previous weekly snapshots
8. **Calculate Trend** (Code): week-over-week comparison; reports a baseline week on the first run
9. **Calculate Stage Stats** (Code): average days per stage, slowest/fastest stage, bottlenecks
10. **Merge Results**: combines trend and stage data
11. **Post Slack Digest** (Slack): posts the weekly brief
12. **Save Weekly Snapshot** (Google Sheets): writes one row per week, matched on `Week Of`

### Data Flow

```
[Every Friday 8 AM]
    ↓
[Pipeline Settings] → stages, SLAs, currency, loss probabilities
    ↓
[Get HubSpot Owners] → owner ID → name map
    ↓
[Get All HubSpot Deals]
    ↓
[Calculate Days & Risk] → normalized deals + risk tier + revenue at risk
    ↓                        ↓
[Create Risk Summary]    [Calculate Stage Stats] → bottlenecks
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

- **HIGH:** days in stage > SLA (40% loss probability)
- **MEDIUM:** days in stage > SLA × 0.7 (15% loss probability)
- **OK:** days in stage ≤ SLA × 0.7 (0%)

The 0.7 threshold and both probabilities are set in Pipeline Settings (`mediumThresholdShare`, `lossProbabilityHigh`, `lossProbabilityMedium`).

**Note:** Loss probabilities are estimates based on deal age vs. SLA. Validate them against your historical close rates and adjust as needed.

## Setup

### Prerequisites

- n8n instance (self-hosted or cloud)
- HubSpot account (free tier works) with a private app
- Google account with a service account for Sheets
- Slack workspace

### Steps

1. **Import the workflow**
   - In n8n: ⋯ → Import from File → `hubspot-Pipeline-Velocity-Bottleneck-Detection-v1.3.json`
   - The JSON contains no credentials; you'll connect your own in the next steps

2. **Connect HubSpot**
   - Create a private app with scopes `crm.objects.deals.read` and `crm.objects.owners.read`
   - Add the token as a *HubSpot App Token* credential in **Get HubSpot Owners** and **Get All HubSpot Deals**

3. **Set up Google Sheets**
   - Create a sheet tab named `Weekly_Trend_Tracker_Simple` with these headers in row 1: `Week Of`, `At-Risk Count`, `Previous Week Count`, `Trend %`, `Revenue at Risk`, `Slowest Stage`, `Bottleneck Count`, `Notes`
   - Format the `Week Of` column as **Plain text** (otherwise date matching fails and rows get duplicated)
   - Share the sheet with your service account as Editor
   - Paste the sheet URL into **Read Trend History** and **Save Weekly Snapshot**

4. **Connect Slack**
   - Connect your Slack account (OAuth2) in **Post Slack Digest**
   - Replace `YOUR_SLACK_CHANNEL_ID` with your channel's ID and invite the app to that channel

5. **Edit Pipeline Settings**
   - Replace the stage IDs, SLA days and currency with your own

6. **Check the schedule**
   - Default: every Friday, 08:00
   - Set the timezone under workflow Settings → Timezone

7. **Test**
   - Run the workflow manually; the first run creates the baseline row
   - Verify the Slack message and the new row in Google Sheets
   - Activate the workflow

## Example Output

```
📊 *Pipeline Velocity Brief*

⚠️ *Revenue at Risk:* €46,500 across 9 deals
🔴 HIGH (likely loss): €38,000 | 5 deals
🟡 MEDIUM (possible loss): €8,500 | 4 deals
📈 *vs Last Week:* 🟢 -2 deals (was 11)

*By Stage:*
• Qualified Lead: 1 at risk of 6 open (HIGH: €0)
• Discovery: 4 at risk of 7 open (HIGH: €20,000)
• Proposal/Demo: 3 at risk of 5 open (HIGH: €12,000)
• Negotiation: 1 at risk of 3 open (HIGH: €6,000)

*Pipeline Health:*
• Slowest: Proposal/Demo (avg 16 days)
• Fastest: Qualified Lead (avg 2 days)
⚠️ 1 bottleneck (stage avg above SLA):
• Discovery: 7 days avg (SLA: 5, +2 days over)

*Highest Risk Deals (Priority Order):*
1️⃣ *BigCorp Expansion* [HIGH]
• Stage: Discovery (9 days, SLA 5) | Amount: €50,000 | Owner: Mike Jensen
2️⃣ *TechStart Migration* [HIGH]
• Stage: Proposal/Demo (25 days, SLA 21) | Amount: €30,000 | Owner: Sarah Klein
3️⃣ *Enterprise Deal* [HIGH]
• Stage: Negotiation (16 days, SLA 14) | Amount: €15,000 | Owner: Carlos Ruiz
```

*Example data is illustrative.*

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

### Trend not calculating or duplicate rows in Google Sheets
- Verify the Google credential and that the service account has Editor access
- Check that the tab name and headers match exactly
- Make sure the `Week Of` column is formatted as plain text

### Unexpected risk classifications
- Review the SLA days and thresholds in **Pipeline Settings**
- Days in stage are calculated from HubSpot's `hs_v2_date_entered_current_stage`: `Math.floor((today - stageEntryDate) / (1000 * 60 * 60 * 24))`

## Design Note: Revenue-at-Risk Calculation

The workflow uses tiered loss probabilities (HIGH 40%, MEDIUM 15%) based on deal age vs. SLA. This assumes these percentages correlate with actual close rates. **Important:** validate them against your historical data and adjust them in Pipeline Settings.

## Changelog

### v1.3: Template-ready release
- Added HubSpot owner lookup (names instead of IDs in Slack)
- Added Pipeline Settings node: stages, SLAs, currency and loss probabilities in one place
- Added data quality flags for unmapped stages and missing stage entry dates
- Added Overview & Setup sticky note; renamed nodes to descriptive names
- Removed credential references, instance IDs and company-specific comments from the JSON

## License

This workflow is provided as-is for operational use and case study documentation.

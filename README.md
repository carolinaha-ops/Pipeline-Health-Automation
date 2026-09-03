# Pipeline Velocity Bottleneck Detection

Automated weekly pipeline health reporting for B2B SaaS sales teams. Detects deal stalls, identifies bottleneck stages, and quantifies revenue at risk—all delivered automatically every Friday morning.

## Problem

Sales leadership lacks systematic visibility into pipeline velocity. Manual reporting is time-consuming (45+ min/week), bottleneck detection is anecdotal ("Discovery feels slow"), and revenue risk is quantified. This delays intervention and blocks data-driven decision-making.

## Solution

An n8n automation that runs weekly (Friday, 8 AM) to:
- Fetch all open deals from HubSpot
- Calculate days-in-stage and classify risk (HIGH/MEDIUM/OK) based on SLA thresholds
- Detect bottleneck stages where SLA is consistently breached
- Quantify revenue at risk using tiered loss probability (HIGH 40%, MEDIUM 15%)
- Post a formatted Slack summary for leadership review
- Track week-over-week trends automatically

## Architecture

### Workflow Components

1. **Schedule Trigger** — Runs every Friday, 8 AM (Berlin time)
2. **Get Deals** — HubSpot API query of all open deals
3. **Calculate Days & Risk** — JavaScript: normalizes stage names, calculates days-in-stage, classifies risk, computes revenue at risk
4. **Aggregate Summary** — Counts at-risk deals by risk level, extracts top 3 high-risk deals
5. **Calculate Trends** — Week-over-week comparison using Google Sheets baseline
6. **Calculate Stage Stats** — Identifies slowest/fastest stages, detects bottlenecks
7. **Merge Results** — Combines trend and stage data
8. **Post Slack Digest** — Formatted message to #all-ai-automation
9. **Update Config** — Auto-updates Google Sheets with current week's at-risk count

### Data Flow

```
HubSpot API
    ↓
[Fetch Deals]
    ↓
[Calculate Risk] → normalized deals + risk classification
    ↓
[Aggregate] → summary data
    ↓
[Trend] → WoW comparison
    ↓
[Stage Stats] → bottleneck detection
    ↓
[Merge & Post] → Slack briefing
    ↓
[Update Config] → Google Sheets baseline
```

## Configuration

### SLA Thresholds (Days in Stage)

| Stage | SLA | Rationale |
|---|---|---|
| Qualified Lead | 5 | Entry point; fast movement expected |
| Discovery | 5 | Fact-finding; should be quick |
| Proposal/Demo | 21 | Complex selling cycle; more time needed |
| Negotiation | 14 | Final stage; delays indicate stalled deals |

Adjust thresholds in the "Calculate Days & Risk" node to match your actual sales cycle.

### Risk Classification

- **HIGH:** Days in stage > SLA threshold (40% loss probability)
- **MEDIUM:** Days in stage > (SLA × 0.7) (15% loss probability)
- **OK:** Days in stage ≤ (SLA × 0.7) (0% loss probability)

**Note:** Loss probabilities are estimates based on deal age vs. SLA. Validate against your historical close rates and adjust percentages as needed.

## Setup

### Prerequisites

- n8n instance (self-hosted or cloud)
- HubSpot account with API access
- Google Sheets account
- Slack workspace with bot access

### Steps

1. **Create n8n Workflow**
   - Import the workflow JSON from this repository
   - Update credentials for HubSpot, Google Sheets, Slack

2. **Configure HubSpot Connection**
   - Generate a private app token with "CRM → Deals (read)" scope
   - Enter token in n8n HubSpot node

3. **Set Up Google Sheets**
   - Create a sheet named "Weekly_Trend_Tracker_Simple"
   - Column A: Week date
   - Column B: At-risk deal count
   - Share with n8n service account (if using API)

4. **Configure Slack Webhook**
   - Create an incoming webhook in your Slack workspace
   - Point to channel #all-ai-automation (or your preferred channel)
   - Add webhook URL to n8n Slack node

5. **Schedule Trigger**
   - Set to "Every Friday, 08:00 AM" (or your preferred time)
   - Timezone: Europe/Berlin (adjust as needed)

6. **Test**
   - Manually trigger the workflow
   - Verify Slack message posts correctly
   - Confirm Google Sheets baseline updates

## Example Output

```
📊 *Pipeline Velocity Brief*

⚠️ *Deals at Risk:* 12 deals | $145,000
📈 *vs Last Week:* 🟢 -2 deals (was 14)

*By Stage:*
• Qualified Lead: 1
• Discovery: 5
• Proposal/Demo: 4
• Negotiation: 2

*Pipeline Health:*
• Slowest: Proposal/Demo (avg 18 days)
• Fastest: Qualified Lead (avg 2 days)

1 Bottlenecks
⚠️ Proposal/Demo: 18 days avg (SLA: 21, -3 days OK)

*Highest Risk Deals:*
1️⃣ *BigCorp Expansion*
• Stage: Discovery | Amount: $75K | Owner: @mike

2️⃣ *TechStart Migration*
• Stage: Proposal/Demo | Amount: $45K | Owner: @sarah

3️⃣ *Enterprise Deal*
• Stage: Discovery | Amount: $25K | Owner: @carlos
```

## Troubleshooting

### Briefing not posting to Slack
- Verify Slack webhook URL is current (regenerate if needed)
- Check n8n logs for API errors in the "Post Slack Digest" node
- Ensure channel exists and bot has permission to post

### Missing or stale data in briefing
- Verify HubSpot API credentials are valid
- Check that "Get many deals" node is fetching all expected deals
- Confirm stage names in n8n match HubSpot's internal IDs (e.g., "qualifiedtobuy" not "Qualified Lead")

### Google Sheets baseline not updating
- Verify Google Sheets credentials in n8n
- Check that sheet tab name and range match configuration
- Confirm bot has "Editor" access to the sheet

### Incorrect risk classifications
- Review SLA thresholds in "Calculate Days & Risk" node
- Verify date calculations: `Math.floor((now - createdate) / (1000 * 60 * 60 * 24))`
- Test with a specific deal by manually running the workflow

## Design Note: Revenue-at-Risk Calculation

The workflow uses tiered loss probabilities (HIGH 40%, MEDIUM 15%) based on deal age vs. SLA. This assumes these percentages correlate with actual close rates. **Important:** Validate these percentages against your historical data and adjust as needed.

## License

This workflow is provided as-is for operational use and case study documentation.

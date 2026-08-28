# PRD: CRM Silent Deal Slack Alerting Service

## Problem Statement

Sales reps at Vantara log calls and update deal stages in a Notion CRM database, but nothing happens after that. Deals go cold and no one notices until pipeline cleanup at the quarterly review — by which point the opportunity is lost. Reps have no signal telling them when a contact has gone silent, and there is no lightweight mechanism to surface this information inside the tools they already use day-to-day.

## Solution

A standalone alerting service that runs every morning at 9am GMT+7, queries a Notion CRM database for deals where the contact has gone silent beyond a stage-specific threshold, and sends a single plain-text Slack DM to each owning sales rep summarising all their breached deals in a digest list. The service matches deal owner emails from Notion to Slack users by email, requires no new dashboard, and delivers the alert inside the tool reps already live in.

## User Stories

1. As a sales rep, I want to receive a single daily Slack DM listing all deals that need my follow-up, so that I am not spammed with multiple individual notifications.
2. As a sales rep, I want each item in the digest to show the deal name, contact name, how many days they've been silent, and the current deal stage, so that I can immediately understand the situation without logging into Notion.
3. As a sales rep, I want to receive the alert at the start of my working day (9am GMT+7), so that I can act on it at the beginning of my day rather than being interrupted mid-flow.
4. As a sales rep working an Active deal, I want to be alerted if a contact has been silent for 2 or more days, so that I can respond quickly on my hottest opportunities.
5. As a sales rep working a Prospect deal, I want to be alerted if a contact has been silent for 3 or more days, so that early-stage leads don't slip through the cracks.
6. As a sales rep working a Cold deal, I want to be alerted if a contact has been silent for 5 or more days, so that I can re-engage before the contact loses interest entirely.
7. As a sales rep working a Warm deal, I want to be alerted if a contact has been silent for 7 or more days, so that a promising deal is not lost to inaction.
8. As a sales rep, I want to only receive alerts relevant to deals I own, so that I am not overwhelmed by noise from other reps' pipelines.
9. As a sales rep, I want the alert message to be plain text, so that it is fast to read and available on mobile without rendering issues.
10. As a sales rep, I want to receive one alert per breached deal per day, so that I am not spammed repeatedly about the same deal.
11. As a system administrator, I want the service credentials (Notion API key, Slack bot token) stored in environment variables and excluded from version control, so that secrets are never committed to the repository.
12. As a system administrator, I want a `.env.example` file committed to the repository, so that any developer can onboard the service without needing to ask for the list of required variables.
13. As a system administrator, I want the service to run on a daily cron schedule at 2am UTC (9am GMT+7), so that reps receive alerts at the start of their working day.
14. As a system administrator, I want the service to skip deals in unknown or unmapped stages gracefully, so that a missing stage mapping does not crash the service or suppress other alerts.
15. As a system administrator, I want the service to match Notion deal owners to Slack users by email address, so that no manual mapping table needs to be maintained.

## Implementation Decisions

### Architecture
- The alerting service is a **standalone Node.js script**, entirely separate from the existing course platform repository. It has no dependency on or coupling to the existing codebase.
- The service is invoked by an external cron scheduler (e.g., a VPS cron, GitHub Actions scheduled workflow, or Railway) on the schedule `0 2 * * *` (2am UTC = 9am GMT+7).

### Silence Thresholds by Deal Stage
| Stage | Silence Threshold |
|-------|-----------|
| Active | 2 days |
| Prospect | 3 days |
| Cold | 5 days |
| Warm | 7 days |

Deals in stages not present in the above map are skipped without error.

### Notion Integration Module
- Uses the official `@notionhq/client` npm package.
- Queries the designated Notion CRM database, filtering for open deals.
- Reads the following properties per deal: deal name, deal stage, owner email, and last contact date.
- Owner email is read directly from a property on the Notion database row — no secondary API call needed.

### Silence Detection Module
- Pure function: accepts a deal and a threshold map, returns whether the deal is breached.
- Computes `daysSinceContact = today - lastContactDate`.
- Returns `true` if `daysSinceContact >= threshold` for the deal's stage.

### Slack Notification Module
- Uses the official `@slack/web-api` npm package.
- Resolves rep Slack user ID via `users.lookupByEmail` using the Notion owner email.
- Sends a single plain-text digest DM per rep via `chat.postMessage`, grouping all breached deals into one message.
- Message format:
  ```
  Hi [First Name]! Here are the deals that need your follow-up today:

  • [Deal Name] with [Contact Name] — silent for [X] days ([Stage])
  • [Deal Name] with [Contact Name] — silent for [X] days ([Stage])
  ```
- If a rep has no breached deals, no message is sent.

### Secrets Management
- `NOTION_API_KEY`, `NOTION_DATABASE_ID`, and `SLACK_BOT_TOKEN` are stored in a `.env` file.
- `.env` is listed in `.gitignore`.
- `.env.example` is committed with empty values as a template.

## Testing Decisions

- **Good tests** verify external behavior, not implementation details. Tests should assert on outputs (which deals are flagged, what message is sent) given controlled inputs, not on internal function calls.
- **Silence Detection Module** should be unit tested in isolation. It is a pure function with no external dependencies — given a deal object and a threshold map, assert the correct boolean output. Cover: exact threshold boundary (N-1 days = no alert, N days = alert), unknown stage (should return false/skip), missing last contact date (should return false/skip).
- **Notion and Slack modules** should be integration tested against sandbox/test credentials rather than mocked, to avoid mock/prod divergence.

## Out of Scope

- Manager-level visibility or coaching reports.
- Deal prioritization scoring or ranking logic.
- Interactive Slack messages (buttons, dropdowns, Block Kit).
- Per-rep timezone scheduling (all alerts run at a single time, 9am GMT+7).
- Native Notion task creation or sequence enrollment.
- Real-time or event-driven alerting via Notion webhooks.
- Any changes to the existing course platform repository.

## Further Notes

- The stage threshold map should be easy to extend as Vantara adds new deal stages to their Notion pipeline.
- The "Active" threshold of 2 days is the tightest because the cost of missing a hot deal is highest; this may need tuning once reps begin using the system.
- Delivery channel is Slack for now. Notion native tasks or email digest can be considered in a future iteration once signal quality is validated.

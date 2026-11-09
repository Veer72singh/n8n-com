# Ripple_Impact — Telegram News Pipeline

A robust, enterprise-grade automated news intelligence and distribution pipeline built for **n8n** (Enterprise v2.25.7) with PostgreSQL, Groq Cloud LLM (`llama-3.3-70b-versatile`), and Telegram.

The pipeline monitors 17+ premier global institutions, central banks, and media feeds, analyzes each item for tangible human impact, filters out noise, formats structured intelligence briefs, and publishes them to the [@ripple_impact](https://t.me/ripple_impact) channel with optional human-in-the-loop admin verification.

---

## Overview

Mainstream financial and global news media often buries critical decisions in bureaucratic jargon or prioritizes sensationalism over practical consequences. **Ripple_Impact** surfaces the quiet macroeconomic shifts, central bank policy adjustments, and global regulatory changes that directly impact ordinary people's mortgages, grocery bills, jobs, utilities, and retirement savings.

- **Telegram Channel**: [@ripple_impact](https://t.me/ripple_impact)
- **Voice & Tone**: Investigative, quietly urgent, creative, and strictly grounded in verified facts. It reveals what mainstream media glosses over — explaining the world clearly like a trusted insider.
- **Strict Guardrails**: Zero hallucination tolerance (strictly derived from input facts), zero ungrounded financial advice, and automated exclusion of sports, entertainment, gaming, and superficial gossip.
- **Autonomous Distribution**: 100% hands-free autonomous pipeline. Stories meeting relevance criteria (`relevant == true` and `impact_score >= 3`) are automatically formatted, rate-limited, published to [@ripple_impact](https://t.me/ripple_impact), and deduplicated without requiring human review.

---

## Architecture Diagram

The end-to-end pipeline operates across three decoupled n8n workflows (`workflow.json`, `approval-handler.json`, `error-handler.json`) connected through PostgreSQL:

```text
+---------------------------------------------------------------------------------------------------+
|                                  MAIN PIPELINE (workflow.json)                                    |
|                                                                                                   |
|  [ Schedule Trigger ] (Every 30 Mins)                                                             |
|           |                                                                                       |
|           v                                                                                       |
|  +---------------------------------------------------------------------------------------------+  |
|  | [1. Multi-Source Ingestion Engine (17 Nodes in Parallel with 'Continue On Fail')           |  |
|  |   - Institutional RSS (Fed, ECB, BIS Speeches, IMF, World Bank, WHO)                       |  |
|  |   - News Feeds (BBC World/Biz/Health, Al Jazeera, Google News Economy)                      |  |
|  |   - Macro APIs (GDELT 2.0, The Guardian, Currents, Finnhub, FRED, BLS, Frankfurter, WB)   |  |
|  +---------------------------------------------------------------------------------------------+  |
|           |                                                                                       |
|           v                                                                                       |
|  [ 2. Merge & Normalize Items ]                                                                   |
|           |                                                                                       |
|           v                                                                                       |
|  [ 3. PostgreSQL Dedup Engine ] --------(Already Exists)---------> [ Drop / Log Duplicate ]       |
|    Query: ripple_dedup_store                                                                      |
|           | (New Items Only)                                                                      |
|           v                                                                                       |
|  [ 4. Groq LLM Analysis (llama-3.3-70b-versatile) ]                                               |
|    Schema: { relevant, category, impact_score, headline, summary, why_it_matters, who_is_affected }|
|           |                                                                                       |
|           v                                                                                       |
|  [ 5. Relevance & Impact Gate ] --------(relevant: false)--------> [ Drop / No-Op Gate ]          |
|    Condition: relevant == true AND impact_score >= 2                                             |
|           | (Passed Gate)                                                                         |
|           v                                                                                       |
|  [ 6. Routing Switch (APPROVAL_REQUIRED) ]                                                        |
|         /                                   \                                                     |
|   (true)                                   (false)                                                |
|       /                                       \                                                   |
|      v                                         v                                                  |
|  [ Insert ripple_approval_queue ]        [ 7. HTML Format & Compliance Decorator ]                |
|  [ Send Telegram Admin Inline Buttons ]           |                                               |
|    (Admin: 5587670796)                            v                                               |
|                                          [ 8. Publish to Telegram Channel: @ripple_impact ]       |
|                                                   |                                               |
|                                                   v                                               |
|                                          [ 9. Log Execution to ripple_post_log ]                  |
+---------------------------------------------------|-----------------------------------------------+
                                                    |
       +--------------------------------------------+
       |
       v
+---------------------------------------------------------------------------------------------------+
|                             APPROVAL HANDLER (approval-handler.json)                              |
|                                                                                                   |
|  [ Telegram Webhook: /webhook/telegram-approval ] (Listens for Inline Button Clicks)              |
|           |                                                                                       |
|           v                                                                                       |
|  [ Extract Callback Data (action: approve/reject, queue_id) ]                                     |
|           |                                                                                       |
|     +-----+----------------------------------+                                                    |
|     | (action == 'approve')                  | (action == 'reject')                               |
|     v                                        v                                                    |
|  [ Fetch Item from ripple_approval_queue ]  [ UPDATE ripple_approval_queue SET status='rejected' ]|
|  [ Format Telegram HTML Brief ]             [ Update Admin Telegram Message to '❌ Rejected' ]   |
|  [ Publish to Channel: @ripple_impact ]                                                           |
|  [ Log to ripple_post_log ]                                                                       |
|  [ UPDATE queue SET status='approved' ]                                                           |
|  [ Update Admin Telegram Message to '✅ Approved & Posted' ]                                      |
+---------------------------------------------------------------------------------------------------+

+---------------------------------------------------------------------------------------------------+
|                              ERROR HANDLER (error-handler.json)                                   |
|                                                                                                   |
|  [ Error Trigger ] (Fires on any unhandled node failure across workflows)                         |
|           |                                                                                       |
|           +---> [ Insert Failure Context into ripple_error_log ]                                  |
|           |                                                                                       |
|           +---> [ Send Urgent Telegram Alert to Admin (@veer72singh, 5587670796) ]                |
+---------------------------------------------------------------------------------------------------+
```

---

## Prerequisites

Before deploying the pipeline, ensure you have:

1. **Docker & n8n**: Self-hosted n8n instance (Enterprise v2.25.7+) running on Node.js 24 with standard webhook reverse proxy (Nginx, Traefik, or Cloudflare Tunnel) with valid SSL.
2. **PostgreSQL Database**: PostgreSQL 14+ database accessible from the n8n container.
3. **Groq Cloud Account**: Free API account at [console.groq.com](https://console.groq.com) for high-speed inference of `llama-3.3-70b-versatile`.
4. **Telegram Bot**: Created via [@BotFather](https://t.me/BotFather) with its auth token.
5. **Free API Keys**:
   - The Guardian Developer Key
   - St. Louis FRED Key
   - Bureau of Labor Statistics (BLS) Key (optional for v2, recommended)
   - Currents API Key
   - Finnhub API Key

---

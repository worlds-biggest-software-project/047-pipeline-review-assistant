# Pipeline Review Assistant

An AI-native revenue forecasting and pipeline analysis tool that surfaces deal risk, guides rep selling activities, and auto-generates deal-level coaching recommendations—purpose-built for RevOps teams and sales managers seeking forecast accuracy without Clari's complexity.

## Problem Statement

Revenue forecasting platforms have a complexity-cost trade-off:
- **Clari** ($60–$100/user/month) offers 85–96% forecast accuracy but requires significant configuration and a dedicated RevOps resource
- **Lightweight alternatives** (Forecastio, Revenue Grid) offer better ease-of-use but lack deal-level intelligence and coaching
- **Forecast miss rates remain stubborn**: 87% of enterprises missed revenue targets in 2025 despite record AI investment (Clari Labs 2026)
- **Reps lack deal guidance**: most tools surface risk but offer no actionable next-step recommendations

## Key Differentiators

### Forecast Accuracy Without Complexity
- AI-adjusted confidence ranges that correct for manager optimism bias
- Bottom-up pipeline roll-up with manager override workflow—transparency from deal to forecast
- Achieves Clari-level accuracy (85–96%) with lighter configuration burden
- 15–20% fewer false positives than rule-based scoring

### Deal-Level Coaching Automation
- Auto-generated next-step recommendations for each at-risk deal based on deal characteristics
- Suggested activities (outreach sequence, stakeholder meeting, proposal revision) surfaced to reps in real-time
- Reduces time-to-coaching from manager review cycles to minutes

### Pipeline Health Insights
- Stalled deal detection: identify opportunities without activity in 14+ days
- Qualification gap analysis: highlight MEDDIC/SPICED signal gaps per deal
- Historical pattern matching: compare current pipeline to similar historical cohorts for risk scoring

### Revenue Forecasting Beyond Pipeline
- Predictive analytics: activity metrics, stakeholder engagement, email response velocity feed forecast models
- Multi-scenario forecasting: "best case," "committed," "pipeline" scenarios with independent roll-ups
- Quarterly trend analysis: week-over-week pipeline movement, velocity metrics, and hidden funnel discovery

## Market Context

- **Market size**: AI sales pipeline management growing within $91B+ CRM market; revenue intelligence subset $3–5B in 2025, growing 20% CAGR
- **Forecast accuracy benchmarks**: Manual judgment ~66%, AI-assisted ~85–96% for mature deployments
- **Key competitors**: Clari ($60–$100/user/month), Gong Forecast ($1,200–$1,600/seat/year + platform fee), Forecastio ($49–$149/user/month)
- **Positioning**: Clari alternative for mid-market RevOps teams seeking forecast accuracy with lighter configuration

## Recommended MVP Scope

- Deal scoring combining CRM activity, engagement signals, and historical patterns
- Real-time risk alerts: stalled deals, qualification gaps, single-threading
- Rep coaching recommendations: suggested activities and next steps per deal
- Pipeline forecasting: bottom-up roll-up with manager confidence adjustments
- Multi-scenario forecasting: committed, likely, pipeline views with independent analytics
- CRM bi-directional sync (Salesforce, HubSpot native)
- RevOps dashboard: pipeline health, forecast variance tracking, performance trends
- Manager coaching dashboard with guided selling workflows per rep/deal

## Resources

**Research Materials**: [research.md](research.md) | **Feature Analysis**: [features.md](features.md)

## Target Users

- RevOps managers and sales operations leaders needing accurate quarterly forecasts
- Sales managers seeking to reduce forecast miss rates and coach reps on stalled deals
- VP Sales/CRO at mid-market companies building pipeline governance and deal reviews
- Enterprise procurement/finance teams responsible for revenue visibility

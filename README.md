Health Services Client — Customer Service & SLA Optimisation Analytics
Tool Stack: Power BI | DAX | SQL | Snowflake | Airbyte | Amazon S3  
Sector: Healthcare Operations  
Role: Lead Analytics Consultant — Amdari Limited  
Date: June 2026
---
Business Problem
A multi-location healthcare customer service operation was experiencing a growing operational crisis. SLA commitments were consistently missed, backlogs were accumulating, data was fragmented across CRM systems, Excel trackers and email chains, and escalation management was entirely reactive.
The client had no single source of truth and no visibility into agent workload distribution or capacity utilisation.
---
Objectives
Centralise all ticket, client, SLA and escalation data into a single analytical layer
Monitor SLA timelines and surface at-risk tickets
Deliver interactive Power BI dashboards with full operational visibility
Build an agent workload heatmap and capacity utilisation reporting
Establish standardised KPIs and historical performance tracking
Forecast ticket demand and SLA breach risk using time-series modelling
---
Data Architecture
Pipeline: Source Systems → Amazon S3 → Airbyte → Snowflake (REPORTING schema) → Power BI (Import mode)
9 Tables loaded and verified:
Table	Description	Rows
TICKETS	Core operational fact table	3,500
CLIENTS	Client organisations	80
AGENTS	Operational agents	120
TICKET_AUDIT_LOGS	Full lifecycle tracking	17,751
ESCALATION_LOG	Escalation events	464
SLA_DEFINITIONS	Contractual targets	12
TICKET_CATEGORIES	Service categories	5
PRIORITY_LEVELS	P1 Critical to P4 Low	4
TEAMS	Hub teams	4
Key design decision: SLA_DEFINITIONS uses a composite key (ContractTier × PriorityID) and cannot be joined via a single physical relationship. All SLA lookups are handled via `LOOKUPVALUE()`.
Data type issue resolved: SLABREACHED and PRIORITYID arrived from Snowflake as TEXT despite appearing numeric. `VALUE()` wrapper applied to all 41 DAX measures to prevent type mismatch errors. This was the single most impactful technical fix in the project.
---
Analytical Framework
Rather than disconnected dashboards, the project follows a five-stage narrative framework:
Page	Question	Focus
1 — Executive Summary	What is happening?	KPI overview, SLA compliance, backlog
2 — Operational Drilldown	Where is it happening?	Category and region breakdown
3 — SLA Risk Monitoring	What risk does it create?	Active breaches, financial exposure
4 — Workload & Capacity	Why is it continuing?	Agent heatmap, utilisation bands
5 — Demand & Forecasting	What happens next?	Time-series forecast, backlog trajectory
---
Key Findings
Metric	Value
Total Tickets	3,500
SLA Compliance Rate	78.5%
SLA Breach Rate	21.5%
Total SLA Breaches	754
Open Backlog	199 tickets
Credit-Exposed Tickets	279
Avg First Response Time	5.1 hours
Avg Resolution Time	40.9 hours
Total Escalations	464
SLA-Risk Escalations	420 of 464
Overloaded Agents	17
Underutilised Agents	35
Avg Monthly Demand	130 tickets
Backlog Growth Rate	70.1%
Core finding: Demand was stable throughout the reporting period. The performance crisis was not caused by rising demand — it was caused by inefficient workload allocation and process gaps. The solution did not require additional headcount; it required smarter deployment of the 114 agents already in place.
---
DAX Measures (41 Total)
Selected Examples
SLA Compliance %
```dax
SLA Compliance % =
DIVIDE(
    COUNTROWS(FILTER(TICKETS, VALUE(TICKETS[SLABREACHED]) = 0)),
    COUNTROWS(TICKETS)
)
```
Active Credit Risk Tickets
```dax
Active Credit Risk =
COUNTROWS(FILTER(TICKETS,
    VALUE(TICKETS[SLABREACHED]) = 1
    && TICKETS[STATUS] IN {"Open", "In Progress", "Escalated", "Pending Client"}
    && RELATED(CLIENTS[SLACREDITCLAUSE]) = 1))
```
Critical Risk Tickets (P1 + Breached + Active)
```dax
Critical Risk Tickets =
COUNTROWS(FILTER(TICKETS,
    VALUE(TICKETS[PRIORITYID]) = 1
    && VALUE(TICKETS[SLABREACHED]) = 1
    && TICKETS[STATUS] IN {"Open", "In Progress", "Escalated", "Pending Client"}))
```
Workload Band (Calculated Column)
```dax
Workload Band =
IF(AGENTS[TicketCount] < 21,
    "Underutilised",
    IF(AGENTS[TicketCount] >= 47,
        "Overloaded",
        "Optimal"
    )
)
```
> Thresholds derived statistically: Low = population mean ÷ agent count = 21. High = Mean (30.7) + StdDev (15.9) = 47.
Calendar Table
```dax
Calendar =
ADDCOLUMNS(
    CALENDAR(
        DATEVALUE(MINX(TICKETS, TICKETS[CREATEDAT])),
        MAXX(FILTER(TICKETS, NOT ISBLANK(TICKETS[RESOLVEDAT])), TICKETS[RESOLVEDAT])
    ),
    "Year", YEAR([Date]),
    "Month Number", MONTH([Date]),
    "Month Name", FORMAT([Date], "MMMM"),
    "Quarter", "Q" & QUARTER([Date]),
    "Year Month", FORMAT([Date], "YYYY-MM"),
    "Day Name", FORMAT([Date], "dddd"),
    "Day Number", WEEKDAY([Date], 2),
    "Is Weekend", IF(WEEKDAY([Date], 2) >= 6, TRUE(), FALSE())
)
```
---
Workforce Modelling
Workload thresholds were derived from the actual ticket distribution rather than assumed benchmarks:
Underutilised: < 21 tickets (below mean ÷ agents)
Optimal: 21–46 tickets
Overloaded: ≥ 47 tickets (mean + 1 standard deviation)
Result: 17 agents overloaded, 35 underutilised — confirming a distribution problem, not a capacity problem. Rebalancing the 52 agents in non-optimal bands represented the primary improvement opportunity.
---
Demand Forecasting
Power BI's built-in time-series forecasting was applied to 27 months of ticket volume data.
Hindcast validation (withheld final 3 months):
Month	Actual	Forecast	Result
Oct 2024	127	143	Just outside CI ⚠
Nov 2024	117	126	Within CI ✅
Dec 2024	121	148	Below CI (seasonal trough) ❌
The model overestimates Q4 due to holiday-period demand reduction. With only 27 months of data covering two Q4 cycles, seasonal sensitivity is limited. The forecast provides reliable directional guidance for operational planning.
---
Recommendations
Rebalance agent allocation using statistically-derived workload bands — target the 52 agents currently outside the optimal range before considering new hires
Implement real-time SLA alerting to shift escalation management from reactive to proactive
Review SLA window definitions for Document Requests and Complaints categories, where SLA misalignment was driving 24%+ of breaches
Establish monthly backlog review cadence — at 70.1% growth rate, the backlog will become unmanageable without active intervention
Migrate to DirectQuery for Phase 2 to enable live Snowflake reporting rather than scheduled Import refreshes
---
Technical Notes
Import mode used for build phase due to Snowflake credential constraints; DirectQuery recommended for production
USERELATIONSHIP() used to activate the inactive Calendar → RESOLVEDAT relationship for resolution analysis
All 279 credit-exposed tickets identified via composite filter: SLABREACHED=1 + client SLA credit clause flag
41 DAX measures grouped into: Operational KPIs, SLA KPIs, Financial & Risk, Forecasting, and Workforce categories
---
Client name and identifying information withheld in line with professional confidentiality obligations. All figures are from the analytical engagement and are accurate.

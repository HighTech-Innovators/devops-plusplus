\# Sustainability-by-Design Principles (P1–P10)



This overview links to each principle’s folder and summarizes the intent in one line.



\## Principles



\- \*\*\[P1 — Eliminate Idle Compute](./P1-eliminate-idle-compute/)\*\*  

&nbsp; Remove non-value background activity and unused capacity so systems become physically quiet when demand is low.



\- \*\*\[P2 — Ensure CPU Usage Has a Clear Purpose](./P2-ensure-cpu-usage-has-a-clear-purpose/)\*\*  

&nbsp; Require every sustained CPU cost to be justified by a business outcome, risk reduction, or operational necessity.



\- \*\*\[P3 — Right-Size Memory With Runtime Feedback](./P3-right-size-memory-with-runtime-feedback/)\*\*  

&nbsp; Tune memory requests/limits using real runtime signals (RSS, allocation rate, GC, page faults) instead of guesswork.



\- \*\*\[P4 — Prioritize IO Visibility Over Compute Assumptions](./P4-prioritize-io-visibility-over-compute-assumptions/)\*\*  

&nbsp; Make call counts, payload sizes, dependency tails, retries, and queues observable so decisions are based on I/O reality.



\- \*\*\[P5 — Validate Performance Early, Not Post-Deployment](./P5-validate-performance-early-not-post-deployment/)\*\*  

&nbsp; Gate merges/releases with repeatable performance + energy regression tests in CI/CD, calibrated by production feedback.



\- \*\*\[P6 — Prune Data at the Source](./P6-prune-data-at-the-source/)\*\*  

&nbsp; Prevent data exhaust (logs/events/metrics/payload bloat) from multiplying downstream CPU, memory, network, and storage cost.



\- \*\*\[P7 — Choose Compute Location Intelligently](./P7-choose-compute-location-intelligently/)\*\*  

&nbsp; Place workloads across cloud/on-prem/edge/home/travel/near-renewables based on latency, data locality, risk, and energy.



\- \*\*\[P8 — Embrace Lightweight Execution](./P8-embrace-lightweight-execution/)\*\*  

&nbsp; Reduce per-unit overhead (allocations, wakeups, contention, transformations) and prevent nested “time/energy hit” patterns.



\- \*\*\[P9 — Tie Performance Thresholds to Perception, Not Tradition](./P9-tie-performance-thresholds-to-perception-not-tradition/)\*\*  

&nbsp; Set latency/freshness/reliability budgets based on what users notice (p95/p99 experience), not inherited numbers.



\- \*\*\[P10 — Measure Efficiency in User Terms](./P10-measure-efficiency-in-user-terms/)\*\*  

&nbsp; Track efficiency as cost/energy per unit of user value (CPU-per-checkout, calls-per-workflow, joules-per-result), not raw utilization.




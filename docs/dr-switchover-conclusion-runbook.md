# Production DR Switchover Conclusion Runbook

Use this runbook to formally conclude a production disaster recovery (DR)
switchover, whether the event was a planned DR test/drill or an unplanned
failover triggered by an incident. "Concluding" a switchover means the team
has explicitly decided the current state (running on DR, or failed back to
primary) is the intended steady state, validated it, communicated it, and
closed out all temporary/emergency changes made during the cutover.

Do not treat a switchover as "done" just because traffic is flowing. It is
only concluded once health, data integrity, monitoring, and ownership are all
confirmed and documented.

## Goals

- Confirm the current site (DR or primary) is healthy and stable enough to be
  the long-term state, or trigger a controlled fail-back.
- Verify no data loss, data divergence, or replication drift remains unresolved.
- Remove or reconcile temporary/emergency changes made during the switchover.
- Restore normal monitoring, alerting, and on-call posture.
- Communicate closure clearly to stakeholders.
- Capture a record for audit and post-incident review.

## Prerequisites

Before starting closure activities, confirm:

- The switchover trigger (incident, DR drill, planned maintenance) is
  identified and referenced (ticket/incident ID).
- The incident commander or DR lead has declared the acute phase over.
- You have access to monitoring dashboards, replication status tools, and
  DNS/traffic routing controls for both regions/sites.
- You know which direction the switchover is concluding in:
  - **Stay on DR** — DR site becomes the new primary (temporarily or
    permanently).
  - **Fail back** — traffic and writes return to the original primary site.

## Step 1: Validate current site health

1. Confirm application-level health checks are green on the currently active
   site.

   ```bash
   curl -sf https://<active-site-health-endpoint>/healthz
   ```

2. Confirm key SLOs/metrics are within normal bounds (latency, error rate,
   throughput, queue depth) for a sustained period, not just a momentary
   check.
3. Confirm all critical downstream dependencies (databases, caches, queues,
   third-party integrations) report healthy from the active site.
4. Confirm background jobs, cron schedules, and async workers are running
   only on the active site (not duplicated on both sites).

## Step 2: Validate data consistency and replication

1. Check replication/sync status between the two sites.

   ```bash
   # Example — replace with your actual replication status command
   pg_stat_replication  # or your DB/replication tool's equivalent
   ```

2. Confirm there is no unresolved write conflict, split-brain, or
   replication lag beyond the acceptable threshold.
3. If the switchover involved a period of dual writes or manual data
   reconciliation, confirm reconciliation is complete and reviewed by the
   data/DB owner.
4. Record the final replication lag and reconciliation status in the
   incident/drill ticket.

## Step 3: Decide and execute the conclusion direction

### Option A — Conclude by staying on DR

1. Get explicit sign-off from the service owner and DR lead that the DR site
   is approved as the new (temporary or permanent) primary.
2. Re-point DNS/load balancers/traffic manager so the DR site is the
   documented primary, not just the active fallback target.
3. Re-establish replication **from** the new primary (DR site) **to** the
   original site, so it becomes the new standby.
4. Update runbooks, architecture diagrams, and on-call documentation to
   reflect the new primary/standby assignment.

### Option B — Conclude by failing back to the original primary

1. Confirm the original primary site's infrastructure is fully repaired,
   patched, and validated before failing back.
2. Re-sync the original primary from the current active (DR) site and
   confirm replication is caught up and stable.
3. Schedule a low-traffic or maintenance window for the fail-back cutover if
   the application cannot tolerate a live cutover.
4. Re-point DNS/load balancers/traffic manager back to the original primary.
5. Confirm the DR site returns cleanly to standby mode with replication
   flowing from primary to DR again.

## Step 4: Verify traffic and routing

1. Confirm DNS TTLs have expired and traffic is fully routed to the intended
   site.

   ```bash
   dig +short <service-hostname>
   ```

2. Confirm no clients, jobs, or internal services are still pinned to the
   old active site (check hardcoded IPs, stale connection pools, cached
   endpoints).
3. Monitor traffic split during the transition window to confirm it reaches
   100% on the intended site before declaring closure.

## Step 5: Restore normal operating posture

1. Disable any temporary DR automation, feature flags, or emergency
   overrides that were enabled to force or support the switchover.
2. Re-enable standard deployment pipelines if they were paused during the
   event.
3. Confirm alerting/monitoring is pointed at the correct active site and
   that alert routing/on-call assignments reflect the current topology.
4. Confirm backups are running against the correct primary.
5. Remove any temporary access grants or elevated permissions issued during
   the incident.

## Step 6: Communicate closure

Notify stakeholders (incident channel, status page, affected teams) that the
switchover is concluded. Include:

- Final state: which site is primary and which is standby.
- Confirmation that data is consistent and no further action is needed from
  consumers of the service.
- Any residual risks or follow-up work still open.
- Timestamp of formal closure.

## Step 7: Post-incident/drill review

1. Schedule a retro/post-incident review if not already scheduled.
2. Capture:
   - Total time to detect, switch over, and conclude.
   - What worked well and what didn't during the switchover.
   - Any manual steps that should be automated.
   - Gaps in monitoring, documentation, or runbooks discovered during the
     event.
3. File follow-up tickets for identified gaps and link them from the
   incident/drill record.

## Final checklist

- [ ] Active site health checks and SLOs are stable over a sustained window.
- [ ] Replication/data consistency between sites is verified with no
      unresolved conflicts.
- [ ] Conclusion direction (stay on DR vs. fail back) was explicitly
      approved by the service/DR owner.
- [ ] DNS/traffic routing fully reflects the concluded state.
- [ ] Standby site is confirmed replicating correctly from the new primary.
- [ ] Temporary DR automation, flags, and emergency access are reverted.
- [ ] Monitoring, alerting, and on-call routing reflect the current topology.
- [ ] Backups are running against the correct primary.
- [ ] Stakeholders were notified of closure.
- [ ] Post-incident/drill review is scheduled or completed, with follow-up
      tickets filed.

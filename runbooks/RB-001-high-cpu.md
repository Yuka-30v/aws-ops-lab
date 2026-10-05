# RB-001: High CPU on EC2 instance

## Alert
- Alarm: lab-linux-01-high-cpu
- Condition: CPUUtilization > 70% (average over 5 minutes)
- Notification: SNS topic lab-alerts (email)

## What it means
The instance is using most of its CPU. Users may see slow responses or timeouts.

## Impact check (first 5 minutes)
1. Is the service still responding? `curl -I http://<public-ip>` (expect 200 OK)
2. How long has CPU been high? Check the alarm graph in CloudWatch.

## Investigation
1. SSH to the instance and confirm the host with `hostname`.
2. `top` : which process is using the CPU? (press q to quit)
3. `uptime` : load average trend
4. `sudo journalctl --since "30 min ago" --no-pager | tail -50` : recent errors
5. Was there a recent deployment or change? Check the CI/CD history.

## Common causes and actions
| Cause | Action |
|---|---|
| Known runaway process | Restart the service: `sudo systemctl restart <service>` |
| Unknown process | Do NOT kill. Collect evidence and escalate. |
| Traffic spike | Record the time and escalate for a scaling decision. |

## Escalate to Level 2 when
- CPU stays above 70% for more than 15 minutes after the first actions
- The service is down or customers are affected
- An unknown or suspicious process is found
- The fix needs a change outside this runbook

## When escalating, include
Instance ID, alarm time, `top` output, recent log lines, actions already taken.

## After resolution
Confirm the alarm returned to OK. Update the ticket with the root cause and actions.
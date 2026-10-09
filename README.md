# AWS Ops Lab

Hands-on lab covering first-line cloud operations on AWS (eu-west-1): Linux and Windows Server
administration on EC2, security group troubleshooting, CloudWatch alerting with an escalation
runbook, and a GitHub Actions CI/CD pipeline to Amazon S3.

**Skills shown:** EC2 (Linux and Windows) · security groups · IAM least privilege · CloudWatch and SNS · runbooks and escalation · Git and GitHub Actions · S3 static hosting · structured troubleshooting

## What I built

- **Linux:** Amazon Linux 2023 on EC2 with nginx; patched with dnf; managed with systemctl and journalctl
- **Windows:** Windows Server 2025 on EC2 via RDP; Event Viewer, Services, local users and groups, PowerShell
- **Monitoring:** CloudWatch alarm (CPU > 70% over 5 minutes) with SNS email alerts and a runbook ([RB-001](runbooks/RB-001-high-cpu.md))
- **CI/CD:** GitHub Actions pipeline that deploys to an S3 static website on every push, with a smoke test

## Architecture

```mermaid
flowchart LR
  Dev[My laptop] -- git push --> GH[GitHub repo]
  GH --> GA[GitHub Actions]
  GA -- s3 sync --> S3[S3 static website]
  User[Browser] --> S3
  Dev -- SSH 22, my IP only --> L[EC2 Amazon Linux 2023 + nginx]
  Dev -- RDP 3389, my IP only --> W[EC2 Windows Server 2025]
  L -- CPU metrics --> CW[CloudWatch alarm]
  CW --> SNS[SNS email]
```

## Troubleshooting cases

| # | Symptom | How I diagnosed it | Root cause | Fix |
|---|---|---|---|---|
| 1 | Browser timed out on port 80 | `curl localhost` on the server returned 200, so the problem was outside the server | No inbound rule for HTTP | Added HTTP rule limited to my IP |
| 2 | Browser showed "connection refused" | Refused (not timeout) meant traffic reached the server; `systemctl status` showed nginx inactive | nginx stopped | Restarted nginx |
| 3 | nginx failed to restart | `nginx -t` pointed to the invalid lines; `journalctl` confirmed | A piped command split over two lines made `tee` write my next commands into nginx.conf | Restored the config from a backup taken before editing |
| 4 | Logs did not match my actions | Event Viewer showed USB and laptop power events | I was looking at my own laptop, not the server (Windows key shortcuts go to the local PC when RDP is not full screen) | Always confirm the host with `hostname` first |
| 5 | SSH timed out after restarting the instance | `Test-NetConnection -Port 22` failed; ping failure was expected (ICMP not allowed) | When adding the HTTP rule I had overwritten the SSH rule; the open session kept working because security groups keep established connections | Re-added the SSH rule; now test with a fresh connection after any firewall change |
| 6 | Pipeline job stuck in "Queued" | Checked githubstatus.com before changing anything | GitHub Actions outage (runner assignment delays) | Waited, no config changes; run #1 completed after the incident |
| 7 | Pipeline failed (red x) | Credentials step passed, so the keys were valid; the upload step logged `NoSuchBucket` | Deliberate test: wrong bucket name in deploy.yml | Corrected the bucket name and pushed again |

## Key evidence

<p>
  <img src="screenshots/08-alarm-full-cycle.png" width="650" alt="CloudWatch alarm full cycle">
  <br>
  <em>CloudWatch alarm cycle: OK, then In alarm when CPU hit 100%, then back to OK after I stopped the load.</em>
</p>

<p>
  <img src="screenshots/10-pipeline-history.png" width="650" alt="GitHub Actions run history">
  <br>
  <em>Pipeline history: #1 slowed by a GitHub outage (10m 40s), #2 auto-deployed version 2, #3 deliberate failure, #4 fixed.</em>
</p>

## More screenshots

<details>
<summary><strong>Linux: security group and service troubleshooting (cases 1 and 2)</strong></summary>
<br>
<p>
  <img src="screenshots/01-ec2-running.png" width="650" alt="EC2 instance running">
  <br><em>Amazon Linux instance running in eu-west-1a, all status checks passed.</em>
</p>
<p>
  <img src="screenshots/02-sg-timeout.png" width="450" alt="Connection timed out">
  <br><em>Case 1: timeout, because the security group had no rule for port 80.</em>
</p>
<p>
  <img src="screenshots/03-nginx-ok.png" width="450" alt="nginx welcome page">
  <br><em>After adding the HTTP rule (my IP only), nginx is reachable.</em>
</p>
<p>
  <img src="screenshots/04-service-refused.png" width="450" alt="Connection refused">
  <br><em>Case 2: connection refused, because nginx was stopped. Traffic reached the server, but nothing was listening.</em>
</p>
</details>

<details>
<summary><strong>Windows Server 2025</strong></summary>
<br>
<p>
  <img src="screenshots/05-windows-server.png" width="650" alt="Windows Server 2025 desktop over RDP">
  <br><em>Windows Server 2025 over RDP (access limited to my IP), used for Event Viewer, Services, local users and network checks.</em>
</p>
</details>

<details>
<summary><strong>CloudWatch alarm test</strong></summary>
<br>
<p>
  <img src="screenshots/06-cpu-load-top.png" width="550" alt="top showing two yes processes at 100% CPU">
  <br><em>Two <code>yes</code> processes pushed both vCPUs of the t3.micro to 100%.</em>
</p>
<p>
  <img src="screenshots/07-alarm-in-alarm.png" width="650" alt="Alarm in alarm state">
  <br><em>CPU crossed the 70% threshold and the alarm moved to In alarm, sending an SNS email.</em>
</p>
</details>

<details>
<summary><strong>CI/CD pipeline</strong></summary>
<br>
<p>
  <img src="screenshots/09-site-v2.png" width="450" alt="Site showing version 2">
  <br><em>Changing one line and pushing updated the live S3 site to version 2 automatically.</em>
</p>
<p>
  <img src="screenshots/11-pipeline-failure.png" width="650" alt="Pipeline failure log">
  <br><em>Case 7: the credentials step passed and the upload step failed with NoSuchBucket, which narrowed the cause to the bucket name.</em>
</p>
</details>

## Security choices

- Root user protected with MFA; day-to-day work as an IAM user with MFA
- SSH and RDP open to my IP address only
- Deploy user limited to one S3 bucket; access keys stored in GitHub Secrets, never in code

## What I would do next

- Replace long-lived access keys with OIDC (short-lived credentials)
- Serve the site through CloudFront with a private bucket
- Define the infrastructure as code (CloudFormation or Terraform)

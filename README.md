\# AWS Ops Lab



Hands-on lab covering first-line cloud operations on AWS (eu-west-1): Linux and Windows Server

administration on EC2, security group troubleshooting, CloudWatch alerting with an escalation

runbook, and a GitHub Actions CI/CD pipeline to Amazon S3.



\## Architecture



```mermaid

flowchart LR

&#x20; Dev\[My laptop] -- git push --> GH\[GitHub repo]

&#x20; GH --> GA\[GitHub Actions]

&#x20; GA -- s3 sync --> S3\[S3 static website]

&#x20; User\[Browser] --> S3

&#x20; Dev -- SSH 22, my IP only --> L\[EC2 Amazon Linux 2023 + nginx]

&#x20; Dev -- RDP 3389, my IP only --> W\[EC2 Windows Server 2025]

&#x20; L -- CPU metrics --> CW\[CloudWatch alarm]

&#x20; CW --> SNS\[SNS email]

```



\## What I built



\- \*\*Linux:\*\* Amazon Linux 2023 on EC2 with nginx; patched with dnf; managed with systemctl and journalctl

\- \*\*Windows:\*\* Windows Server 2025 on EC2 via RDP; Event Viewer, Services, local users and groups, PowerShell

\- \*\*Monitoring:\*\* CloudWatch alarm (CPU > 70% over 5 minutes) with SNS email alerts and a runbook (\[RB-001](runbooks/RB-001-high-cpu.md))

\- \*\*CI/CD:\*\* GitHub Actions pipeline that deploys to an S3 static website on every push, with a smoke test



!\[EC2 instance running](screenshots/01-ec2-running.png)

!\[Windows Server 2025](screenshots/05-windows-server.png)



\## Troubleshooting cases



| # | Symptom | How I diagnosed it | Root cause | Fix |

|---|---|---|---|---|

| 1 | Browser timed out on port 80 | `curl localhost` on the server returned 200, so the problem was outside the server | No inbound rule for HTTP | Added HTTP rule limited to my IP |

| 2 | Browser showed "connection refused" | Refused (not timeout) meant traffic reached the server; `systemctl status` showed nginx inactive | nginx stopped | Restarted nginx |

| 3 | nginx failed to restart | `nginx -t` pointed to the invalid lines; `journalctl` confirmed | A piped command split over two lines made `tee` write my next commands into nginx.conf | Restored the config from a backup taken before editing |

| 4 | Logs did not match my actions | Event Viewer showed USB and laptop power events | I was looking at my own laptop, not the server (Windows key shortcuts go to the local PC when RDP is not full screen) | Always confirm the host with `hostname` first |

| 5 | SSH timed out after restarting the instance | `Test-NetConnection -Port 22` failed; ping failure was expected (ICMP not allowed) | When adding the HTTP rule I had overwritten the SSH rule; the open session kept working because security groups keep established connections | Re-added the SSH rule; now test with a fresh connection after any firewall change |

| 6 | Pipeline job stuck in "Queued" | Checked githubstatus.com before changing anything | GitHub Actions outage (runner assignment delays) | Waited, no config changes; run #1 completed after the incident |

| 7 | Pipeline failed (red x) | Credentials step passed, so the keys were valid; the upload step logged `NoSuchBucket` | Deliberate test: wrong bucket name in deploy.yml | Corrected the bucket name and pushed again |



!\[Timeout](screenshots/02-sg-timeout.png)

!\[nginx working](screenshots/03-nginx-ok.png)

!\[Connection refused](screenshots/04-service-refused.png)



\## Monitoring: alarm fired and recovered



Two `yes` processes pushed both vCPUs to 100%. The alarm moved from OK to In alarm, sent an

email through SNS, and returned to OK after I stopped the load.



!\[CPU load in top](screenshots/06-cpu-load-top.png)

!\[Alarm in alarm](screenshots/07-alarm-in-alarm.png)

!\[Full alarm cycle](screenshots/08-alarm-full-cycle.png)



\## CI/CD: push to deploy



!\[Site version 2](screenshots/09-site-v2.png)

!\[Pipeline history](screenshots/10-pipeline-history.png)

!\[Pipeline failure log](screenshots/11-pipeline-failure.png)



\## Security choices



\- Root user protected with MFA; day-to-day work as an IAM user with MFA

\- SSH and RDP open to my IP address only

\- Deploy user limited to one S3 bucket; access keys stored in GitHub Secrets, never in code



\## What I would do next



\- Replace long-lived access keys with OIDC (short-lived credentials)

\- Serve the site through CloudFront with a private bucket

\- Define the infrastructure as code (CloudFormation or Terraform)


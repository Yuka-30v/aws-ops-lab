\# AWS Ops Lab



Hands-on lab covering first-line cloud operations tasks on AWS (eu-west-1):

Linux and Windows Server EC2 administration, security group troubleshooting,

CloudWatch alerting with a runbook, and a GitHub Actions CI/CD pipeline to S3.



\## Architecture



```mermaid

flowchart LR

&#x20; Dev\[My laptop] -- git push --> GH\[GitHub repo]

&#x20; GH --> GA\[GitHub Actions]

&#x20; GA -- s3 sync --> S3\[S3 static website]

&#x20; User\[Browser] --> S3

&#x20; Dev -- SSH 22 My IP --> L\[EC2 Amazon Linux 2023 + nginx]

&#x20; Dev -- RDP 3389 My IP --> W\[EC2 Windows Server 2025]

&#x20; L -- metrics --> CW\[CloudWatch alarm]

&#x20; CW --> SNS\[SNS email]

```



\## What I built

\- Amazon Linux 2023 EC2 with nginx; SSH restricted to my IP

\- Windows Server 2025 EC2; RDP restricted to my IP; Event Viewer, Services, local users

\- CloudWatch alarm (CPU > 70%) with SNS email alerts and a runbook (runbooks/RB-001-high-cpu.md)

\- GitHub Actions pipeline deploying to S3 with a least-privilege IAM user and a smoke test



\## Troubleshooting cases

| # | Symptom | Root cause | Fix |

|---|---|---|---|

| 1 | Browser timed out on port 80 | No inbound rule for HTTP | Added HTTP rule (My IP) |

| 2 | Connection refused | nginx stopped | Restarted the service |

| 3 | nginx failed to restart | Commands accidentally written into nginx.conf by tee | nginx -t, restored from backup |

| 4 | Logs did not match my actions | Checked Event Viewer on my laptop, not the server | Confirm the host with hostname first |

| 5 | SSH timed out after restart | SSH rule overwritten when adding HTTP | Re-added SSH rule; test with a fresh connection |

| 6 | Pipeline failed (AccessDenied) | Bucket name outside the IAM policy scope | Corrected the bucket name |



\## Security choices

\- Root user protected with MFA; daily work as an IAM user with MFA

\- SSH and RDP limited to my IP only

\- Deploy user limited to one bucket; keys stored in GitHub Secrets



\## What I would do next

\- Replace access keys with OIDC

\- Serve the site through CloudFront with a private bucket

\- Define the infrastructure as code (CloudFormation or Terraform)


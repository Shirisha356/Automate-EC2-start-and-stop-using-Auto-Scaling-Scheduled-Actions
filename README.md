# EC2 Scheduled Auto Scaling (IST)

Automatically run an EC2 instance only when you need it. This project uses an **EC2 Launch Template**, an **Auto Scaling Group (ASG)**, and **Scheduled Actions** to:

- **Scale down to 0** at **12:00 AM IST** every day
- **Scale up to 1** at **07:00 PM IST** every day

The instance therefore runs from 7:00 PM to midnight (5 hours/day), which can significantly cut compute costs for dev, test, or training workloads.

---

## Table of Contents

- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Configuration Summary](#configuration-summary)
- [Setup via AWS Console](#setup-via-aws-console)
- [Setup via AWS CLI](#setup-via-aws-cli)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)
- [Cleanup](#cleanup)
- [Important Notes](#important-notes)

---

## Architecture

```
            ┌──────────────────────────────┐
            │      Scheduled Actions       │
            │  12:00 AM IST → desired = 0  │
            │  07:00 PM IST → desired = 1  │
            └──────────────┬───────────────┘
                           │
                           ▼
            ┌──────────────────────────────┐
            │     Auto Scaling Group       │
            │   min=0  desired=1  max=1    │
            └──────────────┬───────────────┘
                           │ uses
                           ▼
            ┌──────────────────────────────┐
            │       Launch Template        │
            │  AMI, type, key, SG, etc.    │
            └──────────────┬───────────────┘
                           │ launches
                           ▼
                    ┌─────────────┐
                    │ EC2 Instance│
                    └─────────────┘
```

## Prerequisites

- An AWS account with permissions for EC2 and Auto Scaling
- A VPC with at least one subnet
- (Optional) An EC2 key pair for SSH access
- (Optional) [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) configured with `aws configure`

## Configuration Summary

| Resource | Setting | Value |
|---|---|---|
| Launch Template | Name | `my-lt-scheduled` |
| Launch Template | AMI | Amazon Linux 2023 |
| Launch Template | Instance type | `t3.micro` (or `t2.micro`) |
| Auto Scaling Group | Name | `my-asg-scheduled` |
| Auto Scaling Group | Min / Desired / Max | `0` / `1` / `1` |
| Scheduled Action 1 | Name | `scale-down-midnight` |
| Scheduled Action 1 | Cron | `0 0 * * *` → desired `0` |
| Scheduled Action 2 | Name | `scale-up-7pm` |
| Scheduled Action 2 | Cron | `0 19 * * *` → desired `1` |
| Time zone | | `Asia/Kolkata` (IST, UTC+5:30) |

> **Cron format:** `minute hour day-of-month month day-of-week`

---

## Setup via AWS Console

### 1. Create the Launch Template

1. Open **EC2 → Launch Templates → Create launch template**.
2. Enter name `my-lt-scheduled` and a description.
3. Choose the AMI (Amazon Linux 2023), instance type (`t3.micro`), key pair, and security group.
4. Click **Create launch template**.

### 2. Create the Auto Scaling Group

1. Open **EC2 → Auto Scaling Groups → Create Auto Scaling group**.
2. Name it `my-asg-scheduled` and select the launch template.
3. Select your VPC and one or more subnets.
4. Set **Desired = 1**, **Min = 0**, **Max = 1**. Min must be `0`, otherwise the group cannot scale to zero.
5. Choose no scaling policies, then click **Create Auto Scaling group**.

### 3. Create the Scheduled Actions

Open the ASG → **Automatic scaling** tab → **Scheduled actions → Create scheduled action**.

**Scale down at midnight**

| Field | Value |
|---|---|
| Name | `scale-down-midnight` |
| Desired / Min / Max | `0` / `0` / `1` |
| Recurrence | Cron: `0 0 * * *` |
| Time zone | `Asia/Kolkata` |

**Scale up at 7 PM**

| Field | Value |
|---|---|
| Name | `scale-up-7pm` |
| Desired / Min / Max | `1` / `1` / `1` |
| Recurrence | Cron: `0 19 * * *` |
| Time zone | `Asia/Kolkata` |

---

## Setup via AWS CLI

Replace the placeholder values (`ami-xxxx`, `subnet-xxxx`, `sg-xxxx`, `my-key`) with your own.

```bash
REGION=ap-south-1

# 1. Launch template
aws ec2 create-launch-template \
  --region $REGION \
  --launch-template-name my-lt-scheduled \
  --version-description "v1" \
  --launch-template-data '{
    "ImageId": "ami-xxxx",
    "InstanceType": "t3.micro",
    "KeyName": "my-key",
    "SecurityGroupIds": ["sg-xxxx"]
  }'

# 2. Auto Scaling group
aws autoscaling create-auto-scaling-group \
  --region $REGION \
  --auto-scaling-group-name my-asg-scheduled \
  --launch-template "LaunchTemplateName=my-lt-scheduled,Version=\$Latest" \
  --min-size 0 --max-size 1 --desired-capacity 1 \
  --vpc-zone-identifier "subnet-xxxx"

# 3a. Scale down to 0 at 12:00 AM IST
aws autoscaling put-scheduled-update-group-action \
  --region $REGION \
  --auto-scaling-group-name my-asg-scheduled \
  --scheduled-action-name scale-down-midnight \
  --recurrence "0 0 * * *" \
  --time-zone "Asia/Kolkata" \
  --min-size 0 --max-size 1 --desired-capacity 0

# 3b. Scale up to 1 at 7:00 PM IST
aws autoscaling put-scheduled-update-group-action \
  --region $REGION \
  --auto-scaling-group-name my-asg-scheduled \
  --scheduled-action-name scale-up-7pm \
  --recurrence "0 19 * * *" \
  --time-zone "Asia/Kolkata" \
  --min-size 1 --max-size 1 --desired-capacity 1
```

---

## Verification

### Check the configuration

```bash
aws autoscaling describe-scheduled-actions \
  --auto-scaling-group-name my-asg-scheduled --region ap-south-1
```

Confirm each action shows `"TimeZone": "Asia/Kolkata"` and the correct `Recurrence`.

### Test without waiting until midnight

1. Edit `scale-down-midnight` and set the cron to 3–5 minutes from now (in IST). For example, at 3:42 PM IST use `45 15 * * *`.
2. Wait, then confirm:
   - **ASG → Activity → Activity history** shows *Terminating EC2 instance*
   - **ASG → Details** shows desired capacity `0`
   - **EC2 → Instances** shows the instance as `shutting-down` / `terminated`
3. Repeat for `scale-up-7pm` and confirm *Launching a new EC2 instance* appears and a new instance is `running`.
4. **Restore the real schedules** (`0 0 * * *` and `0 19 * * *`, `Asia/Kolkata`).

### Review activity history

```bash
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name my-asg-scheduled --region ap-south-1 \
  --query "Activities[].[StartTime,Description,Cause]" --output table
```

Timestamps are returned in UTC. IST = UTC + 5:30, so midnight IST appears as `18:30 UTC` and 7 PM IST as `13:30 UTC`.

---

## Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| Instance doesn't scale to 0 | ASG minimum is not 0 | Set ASG **Min = 0** |
| Actions fire at the wrong time | Time zone missing or set to UTC | Set time zone to `Asia/Kolkata` on both actions |
| Instance doesn't launch at 7 PM | Quota, invalid AMI, or bad subnet | Check **Activity history** for the error message |
| Scheduled action never runs | Start time in the past or End time reached | Leave Start/End time blank |

## Cleanup

Delete in this order to avoid ongoing charges:

```bash
aws autoscaling delete-auto-scaling-group \
  --auto-scaling-group-name my-asg-scheduled --force-delete --region ap-south-1

aws ec2 delete-launch-template \
  --launch-template-name my-lt-scheduled --region ap-south-1
```

## Important Notes

- **Scale-in terminates the instance.** Data stored only on the root EBS volume is lost. Persist anything important to S3, EFS, or an extra EBS volume that is not deleted on termination, or rebuild the instance state with a **User Data** script.
- A new instance gets a **new public IP** each time. Use an Elastic Load Balancer or a DNS-based approach if you need a stable endpoint.
- Scheduled actions use the time zone you set. Without one, they default to **UTC**.



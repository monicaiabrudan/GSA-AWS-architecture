# GSA 2027 AWS Learning Plan

25 November 2026 · Monica Abrudan

## Goal and how to use this plan

By the end, you can build a small working copy of the GSA 2027 cloud classroom in your own AWS account: an Nginx gateway at a public address that sends each `/studentX` path to a private student desktop, in two regions. Plan on about 4–6 weeks at 3–4 hours a week, in four stages.

Each stage builds on the last. Do the stages in order, and tick each step when done. The build steps refer to the **GSA 2027 on AWS** guide (ask Monica to share it with you), which has the exact commands.

## Before you start: your practice account

A new AWS account gets up to $200 in credits: $100 at sign-up and up to $100 more for trying core services. The free plan lasts 6 months or until credits run out ([AWS Free Tier](https://aws.amazon.com/free/), [announcement](https://aws.amazon.com/about-aws/whats-new/2025/07/aws-free-tier-credits-month-free-plan)). Practice for this plan should cost about $20–40 in total if you tear down after each session.

- [ ] Create the account at [aws.amazon.com/free](https://aws.amazon.com/free/) with your own email. Free Tier credits are for new customers only.
- [ ] Turn on MFA for the root user ([guide](https://docs.aws.amazon.com/IAM/latest/UserGuide/enable-mfa-for-root.html)).
- [ ] Create a budget alert at $20 so AWS emails you before anything surprising happens ([create a budget](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-create.html)).
- [ ] Set the console region to **Asia Pacific (Singapore)** and keep it there until Stage 3.

Three habits that keep practice cheap:

1. Stop or terminate instances at the end of every session.
2. Delete NAT gateways after each session. They charge by the hour even when idle.
3. Check **Billing → Free credits** weekly.

### Which machines to use for practice

Use only Free Tier eligible machines. Never launch `t3.large` or `t3.xlarge` in your practice account; those are for the real course account only. For accounts created after 15 July 2025, the eligible types are ([EC2 Free Tier usage](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-free-tier-usage.html), [AWS free compute](https://aws.amazon.com/free/compute/)):

| Practice role | Instance type | Size | Why |
| --- | --- | --- | --- |
| First test machines, Nginx gateway | `t3.micro` | 2 vCPU, 1 GB | Enough for Nginx and simple tests |
| Student desktop (Xfce + noVNC) | `t3.small` | 2 vCPU, 2 GB | Smallest size that runs a light desktop smoothly |
| Trying a real course tool (optional) | `m7i-flex.large` | 2 vCPU, 8 GB | Closest free size to the course VM; launch for a short test only |

In the launch wizard, tick **Free tier eligible** in the instance-type list. To check from the command line in your region:

```
aws ec2 describe-instance-types --filters Name=free-tier-eligible,Values=true --query "InstanceTypes[].InstanceType"
```

Free machines still use credits for other things. The NAT gateway is not free (about $1.40 a day), and neither are Elastic IPs or data transfer, so delete them after each session.

## Stage 1 — Foundations (week 1, ~8 hours, free)

Learn the vocabulary: regions, EC2, VPC, security groups and billing. None of this uses your account.

| Step | Resource | Time | Focus on |
| --- | --- | --- | --- |
| 1 | [AWS Cloud Practitioner Essentials](https://aws.amazon.com/training/digital) on AWS Skill Builder (search the course name after signing in) | ~6 h | Modules on compute, networking, security and pricing |
| 2 | [Free Builder Labs: Introduction to AWS Cloud](https://aws.amazon.com/blogs/training-and-certification/begin-your-aws-journey-with-new-free-aws-builder-labs-learning-plan-on-aws-skill-builder) | ~2 h | The EC2 and VPC labs; they run in a temporary sandbox account |

You're ready for Stage 2 when you can explain, in a sentence each: region vs availability zone, public vs private subnet, security group, AMI.

## Stage 2 — Core building blocks in your own account (weeks 2–3, ~8 hours, ~$5)

Practise each piece the GSA setup uses, one tutorial at a time, in Singapore.

1. **Launch and connect to a Linux machine.** [Tutorial 1: Launch my first EC2 instance](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/tutorial-launch-my-first-ec2-instance.html), then [Tutorial 2: Launch a test instance and connect](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/tutorial-launch-a-test-ec2-instance.html). Choose Ubuntu, as the course VM does, and the t3.micro type.
2. **Use the command line.** Open [AWS CloudShell](https://docs.aws.amazon.com/cloudshell/latest/userguide/welcome.html) in the console and run `aws ec2 describe-instances`. Optional: [install the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html) on your laptop.
3. **Firewall rules.** Read [security groups](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html); add a rule for port 6080 to your test instance and remove it again.
4. **Private network with NAT.** Follow [Example: VPC with servers in private subnets and NAT](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-example-private-subnets-nat.html). Use the "VPC and more" wizard with 1 availability zone and 1 NAT gateway. This is the heart of the GSA design.
5. **Reach a private machine.** Launch an instance in the private subnet with no public IP, then connect with [Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html). Check `curl https://github.com` works (outbound via NAT).
6. **Start-up scripts.** Re-launch it with a [user data](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html) script that installs a package on first boot.
7. **Images and templates.** [Create an AMI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/creating-an-ami-ebs.html) from that instance, then launch 2 copies from a [launch template](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-launch-templates.html).
8. **Tear down.** Terminate instances, delete the NAT gateway, release its Elastic IP, and delete the VPC.

You're ready for Stage 3 when you can rebuild steps 4–7 in under an hour without the tutorials.

## Stage 3 — Build a mini GSA classroom (weeks 4–5, 4 sessions, ~$10–15, mostly NAT)

Build the real design at small scale: 1 t3.micro gateway and 2 t3.small student desktops in Singapore, then 1 in Mumbai. Follow the matching steps in the GSA 2027 on AWS guide. Tear down at the end of each session, except the AMI.

| Session | Build | Guide steps | Tutorials |
| --- | --- | --- | --- |
| A (~2 h) | Singapore VPC with public + private subnets and NAT, using the 2026 IP ranges; gateway and student security groups | 3, 4 | [VPC with private subnets and NAT](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-example-private-subnets-nat.html) |
| B (~3 h) | Student desktop image: Ubuntu + Xfce + TigerVNC + websockify on port 6080, then save as an AMI. If the 2026 GCP image is available, also try importing it | 5 | [noVNC](https://github.com/novnc/noVNC), [VM Import requirements](https://docs.aws.amazon.com/vm-import/latest/userguide/prerequisites.html), [create an AMI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/creating-an-ami-ebs.html) |
| C (~3 h) | Gateway with an Elastic IP and Nginx; map `/student1` and `/student2` to the 2 private desktops; open both in a browser | 6, 7, 8 | [Elastic IPs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html), [Nginx WebSocket proxying](https://nginx.org/en/docs/http/websocket.html) |
| D (~2 h) | Copy the AMI to Mumbai, build a Mumbai VPC (10.190.0.0/16), peer it with Singapore, add `/student3` | 3, 5 | [Copy an AMI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/CopyingAMIs.html), [create a peering connection](https://docs.aws.amazon.com/vpc/latest/peering/create-vpc-peering-connection.html) |

Practice tips:

- You don't need the `gsa-course.com` domain to practise. Use `http://<gateway IP>/student1/vnc.html?path=student1/websockify` and skip HTTPS until the real build.
- For HTTPS later, point a test domain at the gateway and follow [Certbot's Nginx instructions](https://certbot.eff.org/instructions).
- Write every command you run into a text file. That file becomes the runbook for the real course account.

Done when a colleague on another network opens all three URLs and runs a short practical in each.

## Stage 4 — Get ready for course scale (week 6, ~4 hours)

Learn the tools that turn 3 desktops into 300, then move to the Sanger course account.

1. **Quotas.** Read [requesting a quota increase](https://docs.aws.amazon.com/servicequotas/latest/userguide/request-quota-increase.html). In the course account, request vCPU quota early (about 1,200 for 300 students).
2. **Scripted launches.** Launch 5 tagged t3.small desktops with one `run-instances` command, then generate the Nginx map and URL list with the script in guide step 8.
3. **Schedules.** Stop and start desktops by tag; optionally automate with [EventBridge Scheduler](https://docs.aws.amazon.com/scheduler/latest/UserGuide/what-is-scheduler.html).
4. **Costs.** Read your practice bill in [Cost Explorer](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-what-is.html), grouped by service, and re-estimate course week in the [AWS Pricing Calculator](https://calculator.aws/).
5. **Hand-over.** Move the runbook to the course account (through Sanger IT) and rebuild there.

Skills checklist before the real build:

- [ ] Build a VPC with public and private subnets and a NAT gateway
- [ ] Write security group rules so only the gateway reaches port 6080
- [ ] Build, update and copy an AMI between regions
- [ ] Configure Nginx to proxy noVNC over WebSockets
- [ ] Launch, tag, stop, start and terminate many instances from the CLI
- [ ] Peer two VPCs in different regions
- [ ] Read a bill and find what costs most
- [ ] Tear everything down to zero ongoing cost

## When you get stuck

- **Can't reach a desktop:** check, in order, the security group, the route table (private subnet → NAT), and whether VNC and websockify are running on the student VM.
- **Unexpected charges:** look for running NAT gateways, unattached Elastic IPs and stopped instances' disks in Cost Explorer.
- **Credits for the real course:** Sanger can apply to [AWS Cloud Credit for Research](https://aws.amazon.com/government-education/research-and-technical-computing/cloud-credit-for-research/), which accepts workshop and course applications. Reviews can take months, so apply early.

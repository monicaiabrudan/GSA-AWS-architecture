Genome Sequence Analysis · Asia · 2027

# Rebuilding the GSA cloud classroom on AWS

In 2026, participants opened `gsa-course.com/student[X]` in a browser and got their own Linux desktop with the course tools. This guide rebuilds the same system on AWS, piece by piece, with the commands to run.

**1 gateway**Nginx reverse proxy, the only public machine

**2 regions**Singapore + India, student VMs on private subnets

**e2-standard-4**per student: 4 vCPU, 16 GB

**\~£1,800**2026 GCP spend (£1,586 in course month)

## The 2026 setup, translated

Source: the “Google Cloud Platform System Setup” figure in the 2026 course evaluation. Each GCP component has an AWS counterpart. Three differences matter, marked in orange.

| Component | GCP 2026 | AWS 2027 |
| --- | --- | --- |
| Project / network | One project, `default-vpc` (global) | An AWS account with one VPC per regionVPCs are regional in AWS, not global |
| Singapore subnet | asia-southeast1, 10.148.0.0/20 | `ap-southeast-1`, private subnet 10.148.0.0/20 |
| Delhi subnet | asia-south2 (Delhi), 10.190.0.0/20 | `ap-south-1` (Mumbai), private subnet 10.190.0.0/20AWS has no Delhi region; Mumbai or Hyderabad |
| Gateway VM | VM with public IP, Nginx reverse proxy, `gateway-startup.sh` | EC2 instance in a public subnet with an Elastic IP, same Nginx config |
| Firewall | `allow-http-server` rule (80/443) | Security groups: gateway open on 80/443; students accept 6080 only from the gateway |
| Student VMs | e2-standard-4, internal IP only | EC2 `t3.xlarge` (4 vCPU, 16 GB), no public IP |
| Desktop startup | Startup script: GUI desktop, VNC, Websockify | User data script (or baked into the image) |
| Course image | Custom image | AMI, copied to both regions |
| Outbound internet | Cloud NAT per region, outbound only | NAT Gateway per region, outbound onlyBilled per hour and per GB; the largest fixed cost |
| Domain | `gsa-course.com` → gateway | Same domain, DNS A record → gateway Elastic IP |

Because students reach their VM by path, not by IP, stopping and starting VMs overnight doesn’t change any URL. AWS keeps an instance’s private IP across stop/start, so the Nginx map stays valid all week.

## The AWS architecture

Students reach one public gateway over HTTPS. The gateway proxies each `/studentX` path to that student's private desktop in Singapore, or across VPC peering to Mumbai. Student desktops have no public IP and reach the internet only outbound, through a NAT gateway in each region.

![GSA 2027 AWS architecture: student browsers connect over HTTPS to an Nginx gateway in a public subnet in Singapore, which proxies each /studentX path to private student VMs in Singapore and, via VPC peering, in Mumbai; each region has a NAT gateway for outbound internet only](gsa-2027-aws-architecture.png)

Solid grey: student traffic (inbound via the gateway). Dashed orange: outbound only, via NAT.

Simpler alternative: one gateway per region, no peering

Put a second Nginx gateway in Mumbai and give it a subdomain (for example `in.gsa-course.com/student301`). India, Pakistan, Bangladesh and Sri Lanka classrooms then connect to a nearby machine instead of routing via Singapore, and you skip VPC peering. The cost is one more small instance and two URL patterns instead of one.

## The build, step by step

Run commands in [AWS CloudShell](https://console.aws.amazon.com/cloudshell) (a terminal inside the AWS console, already logged in), or on a laptop after `aws configure`. Replace anything in `<angle brackets>`. If you have the 2026 `gateway-startup.sh` and student startup script, most of their contents carry over unchanged.

1. ### Account, billing and money guardrails December
   - Ask Sanger IT whether Sanger has an AWS Organization to join, so billing works like the 2026 GCP re-charge.
   - Turn on MFA for the root user; work day to day as an IAM Identity Center user.
   - Apply for AWS Cloud Credit for Research now (it accepts workshop and course applications; reviews can take 90–120 days), alongside the AWS sponsorship conversation already noted in the budget.

   ```
   aws budgets create-budget --account-id <ACCOUNT_ID> \
     --budget '{"BudgetName":"GSA2027","BudgetLimit":{"Amount":"3000","Unit":"USD"},"TimeUnit":"MONTHLY","BudgetType":"COST"}' \
     --notifications-with-subscribers '[{"Notification":{"NotificationType":"ACTUAL","ComparisonOperator":"GREATER_THAN","Threshold":50},"Subscribers":[{"SubscriptionType":"EMAIL","Address":"<your-email>"}]},{"Notification":{"NotificationType":"FORECASTED","ComparisonOperator":"GREATER_THAN","Threshold":100},"Subscribers":[{"SubscriptionType":"EMAIL","Address":"<your-email>"}]}]'
   ```
2. ### Request vCPU quota increases in both regions December

   New accounts are capped far below course scale. The quota counts vCPUs across all running standard instances: 300 students × 4 vCPU = 1,200, plus the gateway. Request per region for the students you’ll place there.

   ```
   aws service-quotas request-service-quota-increase --region ap-southeast-1 \
     --service-code ec2 --quota-code L-1216C47A --desired-value 1000
   aws service-quotas request-service-quota-increase --region ap-south-1 \
     --service-code ec2 --quota-code L-1216C47A --desired-value 600
   ```

   **Most likely blocker.** AWS can query a big jump on a new account. Explain the training use and dates in the support case, and step up gradually if asked (for example 100 for the February facilitator training, then the full number for May).
3. ### Build the networks January

   The VPC console’s **Create VPC → “VPC and more”** wizard builds this in one form: pick 1 public + 1 private subnet and “NAT gateways: In 1 AZ”. Use the 2026 ranges so nothing overlaps:
   - Singapore: VPC `10.148.0.0/16`, public subnet `10.148.100.0/24` (gateway), private subnet `10.148.0.0/20` (students)
   - Mumbai: VPC `10.190.0.0/16`, private subnet `10.190.0.0/20` (+ the small public subnet the NAT gateway needs)

   Then link the two VPCs (skip if you use one gateway per region):

   ```
   # request from Singapore, accept in Mumbai
   PCX=$(aws ec2 create-vpc-peering-connection --region ap-southeast-1 \
     --vpc-id <vpc-sg> --peer-vpc-id <vpc-in> --peer-region ap-south-1 \
     --query VpcPeeringConnection.VpcPeeringConnectionId --output text)
   aws ec2 accept-vpc-peering-connection --region ap-south-1 --vpc-peering-connection-id $PCX

   # routes both ways
   aws ec2 create-route --region ap-southeast-1 --route-table-id <rtb-sg-public> \
     --destination-cidr-block 10.190.0.0/16 --vpc-peering-connection-id $PCX
   aws ec2 create-route --region ap-south-1 --route-table-id <rtb-in-private> \
     --destination-cidr-block 10.148.0.0/16 --vpc-peering-connection-id $PCX
   ```
4. ### Security groups (the firewall) January

   Same rules as 2026: only the gateway is reachable from the internet; student desktops accept connections only from the gateway.

   ```
   # Gateway (Singapore): web open to all, SSH from Sanger only
   GW_SG=$(aws ec2 create-security-group --region ap-southeast-1 --vpc-id <vpc-sg> \
     --group-name gsa-gateway --description "GSA gateway" --query GroupId --output text)
   aws ec2 authorize-security-group-ingress --region ap-southeast-1 --group-id $GW_SG --protocol tcp --port 80  --cidr 0.0.0.0/0
   aws ec2 authorize-security-group-ingress --region ap-southeast-1 --group-id $GW_SG --protocol tcp --port 443 --cidr 0.0.0.0/0
   aws ec2 authorize-security-group-ingress --region ap-southeast-1 --group-id $GW_SG --protocol tcp --port 22  --cidr <SANGER-IP-RANGE>

   # Students: noVNC port only from the gateway's subnet (repeat in Mumbai with its VPC id)
   ST_SG=$(aws ec2 create-security-group --region ap-southeast-1 --vpc-id <vpc-sg> \
     --group-name gsa-student --description "GSA student VMs" --query GroupId --output text)
   aws ec2 authorize-security-group-ingress --region ap-southeast-1 --group-id $ST_SG \
     --protocol tcp --port 6080 --cidr 10.148.100.0/24
   ```
5. ### Build the student image (AMI) January

   Route A: export the 2026 GCP image (most faithful)

   If the 2026 project still exists, export the student image as VMDK and import it. AWS VM Import accepts VMDK, VHD, VHDX, OVA and raw.

   ```
   # On GCP
   gcloud compute images export --image=<gsa2026-student-image> \
     --export-format=vmdk --destination-uri=gs://<bucket>/gsa-student.vmdk

   # On AWS (needs the one-off "vmimport" service role from the VM Import docs)
   aws s3 mb s3://gsa2027-images --region ap-southeast-1
   aws s3 cp gsa-student.vmdk s3://gsa2027-images/
   aws ec2 import-image --region ap-southeast-1 --description "GSA student" \
     --disk-containers "Format=vmdk,UserBucket={S3Bucket=gsa2027-images,S3Key=gsa-student.vmdk}"
   aws ec2 describe-import-image-tasks --region ap-southeast-1   # wait for "completed"
   ```

   Route B: build fresh on EC2 (reproducible for future years)

   Launch one Ubuntu LTS `t3.xlarge` with a 60 GB disk in the private subnet, install the course tools and data, then the desktop stack the 2026 startup script ran:

   ```
   sudo apt update && sudo apt install -y xfce4 xfce4-terminal tigervnc-standalone-server novnc websockify firefox
   vncpasswd                                   # participant password
   vncserver :1 -localhost yes -geometry 1600x900
   websockify --web /usr/share/novnc 6080 localhost:5901
   # wrap both in systemd services (or user data) so they start on every boot
   ```

   Freeze it, then copy it to Mumbai:

   ```
   AMI=$(aws ec2 create-image --region ap-southeast-1 --instance-id <i-xxxx> --name gsa2027-student-v1 --query ImageId --output text)
   aws ec2 copy-image --region ap-south-1 --source-region ap-southeast-1 --source-image-id $AMI --name gsa2027-student-v1
   ```
6. ### Launch the gateway with Nginx and HTTPS January

   A small instance in the Singapore public subnet with an Elastic IP. Point `gsa-course.com` (A record, at whoever hosts the domain’s DNS) at that IP, then install Nginx and a free Let’s Encrypt certificate.

   ```
   aws ec2 run-instances --region ap-southeast-1 --image-id <ubuntu-ami> --instance-type m7i.large \
     --subnet-id <public-subnet> --security-group-ids $GW_SG --key-name <your-key> \
     --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=gsa-gateway}]'
   EIP=$(aws ec2 allocate-address --region ap-southeast-1 --query AllocationId --output text)
   aws ec2 associate-address --region ap-southeast-1 --instance-id <gateway-id> --allocation-id $EIP

   # on the gateway
   sudo apt install -y nginx certbot python3-certbot-nginx
   sudo certbot --nginx -d gsa-course.com
   ```

   One `location` block per student, carrying the websocket upgrade that noVNC needs. This is what the 2026 proxy mapping did:

   ```
   location /student1/ {
       proxy_pass http://10.148.0.11:6080/;
       proxy_http_version 1.1;
       proxy_set_header Upgrade $http_upgrade;
       proxy_set_header Connection "upgrade";
       proxy_read_timeout 3600s;     # keep idle desktops connected
   }
   ```

   Students then open `https://gsa-course.com/student1/vnc.html?path=student1/websockify&autoconnect=1&reconnect=1`, which also turns on the auto-reconnect that the 2026 guide asked people to set by hand.
7. ### Launch student VMs per classroom Feb (training), May (course)

   ```
   aws ec2 create-launch-template --region ap-southeast-1 --launch-template-name gsa-student --launch-template-data '{
     "ImageId":"<ami-sg>","InstanceType":"t3.xlarge",
     "NetworkInterfaces":[{"DeviceIndex":0,"SubnetId":"<private-subnet-sg>","Groups":["'$ST_SG'"],"AssociatePublicIpAddress":false}],
     "CreditSpecification":{"CpuCredits":"standard"}}'

   # e.g. 25 VMs for one classroom, tagged so they can be managed together
   aws ec2 run-instances --region ap-southeast-1 --launch-template LaunchTemplateName=gsa-student --count 25 \
     --tag-specifications 'ResourceType=instance,Tags=[{Key=Course,Value=GSA2027},{Key=Classroom,Value=VinUni}]'
   ```

   Repeat in `ap-south-1` with the Mumbai AMI and subnet for the South Asia classrooms. “standard” CPU credits stop t3 burst charges surprising you; if heavy steps feel slow, switch to “unlimited” or to `m7i.xlarge`.
8. ### Generate the Nginx map and URL lists After each launch

   Replaces hand-editing the proxy map. Numbers students in launch order, writes the Nginx config, and writes a CSV per classroom for local trainers.

   ```
   : > students.conf; : > urls.csv
   { for R in ap-southeast-1 ap-south-1; do
       aws ec2 describe-instances --region $R \
         --filters Name=tag:Course,Values=GSA2027 Name=instance-state-name,Values=running,stopped \
         --query 'Reservations[].Instances[].[Tags[?Key==`Classroom`]|[0].Value,PrivateIpAddress]' --output text | sort
     done; } | {
     n=0
     while read room ip; do
       n=$((n+1))
       printf 'location /student%s/ { proxy_pass http://%s:6080/; proxy_http_version 1.1; proxy_set_header Upgrade $http_upgrade; proxy_set_header Connection "upgrade"; proxy_read_timeout 3600s; }\n' $n $ip >> students.conf
       echo "$room,student$n,https://gsa-course.com/student$n/vnc.html?path=student$n/websockify&autoconnect=1&reconnect=1" >> urls.csv
     done; }
   # copy students.conf to the gateway, include it inside the server{} block, then: sudo nginx -t && sudo systemctl reload nginx
   ```

   **Numbering stays stable** only if you run this once after all VMs exist. Re-running after adding VMs can shift numbers; tag each VM with a fixed `Student` number instead if classrooms are added late.
9. ### Start and stop around teaching hours Course week

   Stopping students’ VMs overnight halves compute cost; disks and private IPs are kept, so work and URLs survive. Leave the gateway and NAT running.

   ```
   IDS=$(aws ec2 describe-instances --region ap-southeast-1 --filters Name=tag:Classroom,Values=VinUni \
     --query 'Reservations[].Instances[].InstanceId' --output text)
   aws ec2 start-instances --region ap-southeast-1 --instance-ids $IDS   # morning
   aws ec2 stop-instances  --region ap-southeast-1 --instance-ids $IDS   # evening
   ```

   To automate, create an EventBridge Scheduler schedule per classroom time zone.
10. ### Tear down and check the bill After the course

    NAT gateways and disks keep charging while idle. Terminate student VMs and the gateway, delete both NAT gateways, release the Elastic IP, and keep only the AMIs and scripts for next year.

    ```
    for R in ap-southeast-1 ap-south-1; do
      IDS=$(aws ec2 describe-instances --region $R --filters Name=tag:Course,Values=GSA2027 \
        --query 'Reservations[].Instances[].InstanceId' --output text)
      [ -n "$IDS" ] && aws ec2 terminate-instances --region $R --instance-ids $IDS
      for N in $(aws ec2 describe-nat-gateways --region $R --filter Name=state,Values=available \
        --query 'NatGateways[].NatGatewayId' --output text); do
        aws ec2 delete-nat-gateway --region $R --nat-gateway-id $N; done
    done
    ```

    Then open Cost Explorer, group by service, and compare against the budget.

## What it will cost

Set the number of participants who will use cloud desktops (some hosts, such as icddr,b, have their own HPC). Prices are on-demand USD for Singapore and Mumbai.

Students on Singapore VMs Students on Mumbai VMs Student instance type Running hours per student VM Student disk size (GB) Days the setup exists (launch → teardown) USD → GBP rate

Estimated course-week total

–

–

Assumes gp3 disks ≈ $0.096/GB-month; one m7i.large gateway ≈ $0.126/h running 24/7; one NAT gateway per region ≈ $0.059/h plus $0.059/GB with \~1 GB per student; \~2 GB of desktop traffic per student out to the internet at ≈ $0.12/GB, and the same again across the peering link for Mumbai students at ≈ $0.09/GB. Instance prices are on-demand; disk, NAT and transfer rates are approximate. Confirm in the [AWS Pricing Calculator](https://calculator.aws/) before finalising the budget.

**Budget check.** The budget holds £2,000 for course-week cloud, close to 2026’s £1,586 course-month bill, when 4 of the 6 classrooms (145 participants in all) used GCP. At 2027 scale the same design comes to about £3,200 with the defaults above (300 students on cloud). Options: confirm the real cloud head-count with each host, use `t3.large` if the practicals fit in 8 GB, or close the gap with AWS credits.

## Timeline to the May course

**Dec 2026**Account and billing via Sanger IT · budget alerts · credit application · quota requests in both regions

**Jan 2027**VPCs, peering, security groups · build and test the student AMI · gateway with HTTPS on gsa-course.com

**Feb–Mar**Facilitator pre-course training on a small fleet · collect cloud head-counts per classroom

**Apr**Dress rehearsal: launch one full classroom and connect from the host site over their own network

**May**Launch all classrooms \~3 days before · send URL lists to local trainers · daily start/stop

**Jun**Terminate VMs, delete NAT gateways, release the IP · keep AMIs and scripts · reconcile the bill

## Sources

- Internal: *Genome Sequence Analysis Programme – Course evaluation* (figure “Google Cloud Platform System Setup”); 2026 computing-environment guides; GSA 2027 budget workbook; 2025-26 events budget (GCP re-charges).
- EC2 prices: DoIT [t3.xlarge Singapore](https://www.doit.com/compute/spot/ap-southeast-1/t3.xlarge) · [t3.xlarge Mumbai](https://www.doit.com/compute/spot/ap-south-1/t3.xlarge) · [m7i.xlarge Singapore](https://www.doit.com/compute/spot/ap-southeast-1/m7i.xlarge)
- [AWS VM Import/Export: supported formats](https://docs.aws.amazon.com/vm-import/latest/userguide/prerequisites.html)
- [AWS NAT gateway pricing](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-pricing.html) · [EBS pricing](https://aws.amazon.com/ebs/pricing) · [Public IPv4 charge](https://aws.amazon.com/blogs/aws/new-aws-public-ipv4-address-charge-public-ip-insights)
- [AWS Cloud Credit for Research](https://aws.amazon.com/government-education/research-and-technical-computing/cloud-credit-for-research/)
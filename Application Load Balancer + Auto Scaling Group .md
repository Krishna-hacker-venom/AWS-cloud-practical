# AWS Practical: Application Load Balancer + Auto Scaling Group

> **Topic:** ALB (Application Load Balancer) + EC2 Launch Template + Auto Scaling Group with Dynamic Scaling Policy  
> **Services Used:** EC2, ALB, Target Groups, AMI, Launch Templates, Auto Scaling

---

## Architecture Overview

```
Internet
   |
   v
Application Load Balancer (internet-facing)
   |
   v
Target Group
   |
   v
Auto Scaling Group (min/desired/max EC2 instances)
   |
   v
EC2 Instances (Amazon Linux + Apache httpd)
```

---

## Part 1 — Create Application Load Balancer

### Step 1: Create Security Group for ALB

1. Go to **EC2 → Security Groups → Create Security Group**
2. Configure:
   - **Name:** `alb-sg` (or any preferred name)
   - **Description:** Security group for ALB
   - **Inbound Rules:** Allow `HTTP (port 80)` from `0.0.0.0/0`
3. Click **Create Security Group**

---

### Step 2: Create Target Group

1. Go to **EC2 → Target Groups → Create Target Group**
2. Configure:
   - **Target type:** Instances
   - **Target group name:** your choice
   - **Protocol:** HTTP | **Port:** 80
   - **VPC:** Default VPC
3. Skip registering instances for now (no instances yet)
4. Click **Create Target Group**

---

### Step 3: Create Application Load Balancer

1. Go to **EC2 → Load Balancers → Create Load Balancer**
2. Select **Application Load Balancer**
3. Configure:
   - **Name:** any name (e.g., `my-app-alb`)
   - **Scheme:** Internet-facing
   - **IP address type:** IPv4
   - **Listeners:** HTTP on port 80
   - **Availability Zones:** Select at least 2 AZs (required for ALB)
   - **Security Group:** Select the one created above
4. **Listener & Routing:** Forward to the Target Group created above
5. Everything else — default
6. Click **Create Load Balancer**

> **Note:** ALB requires at least 2 subnets in different Availability Zones.

---

## Part 2 — Launch 2 EC2 Instances Manually & Install Apache

### Step 1: Launch EC2 Instances

1. Go to **EC2 → Launch Instances**
2. Configure:
   - **AMI:** Amazon Linux 2
   - **Instance Type:** t2.micro
   - **Number of instances:** 2
   - **Subnet:** No specific preference (use default)
   - **Security Group:** Allow **All Traffic** (for lab purposes)
3. Launch instances

---

### Step 2: Install & Configure Apache (httpd) on Each Instance

Connect to each instance via EC2 Instance Connect or SSH, then run:

```bash
#!/bin/bash
sudo su
yum install httpd -y
systemctl start httpd
systemctl enable httpd
echo "Welcome" > /var/www/html/index.html
```

#### Verify the Web Page

```bash
# Check the file was created
cat /var/www/html/index.html

# Check Apache status
systemctl status httpd
```

Then open the instance's **Public IP** in a browser — you should see `Welcome`.

---

## Part 3 — Create AMI (Amazon Machine Image)

> An AMI saves the current state of the instance so it can be reused in the Launch Template.

### Steps:

1. Go to **EC2 → Instances**
2. Select your configured instance
3. Click **Actions → Image and Templates → Create Image**
4. Configure:
   - **Image name:** any (e.g., `apache-web-server-ami`)
   - **Description:** Apache pre-installed image
   - **Reboot instance:** Enabled (recommended for consistent AMI)
   - **Delete on termination:** Enabled (for root volume — default)
   - Everything else — default
5. Click **Create Image**
6. Go to **EC2 → AMIs** to verify the image status (`pending` → `available`)

---

## Part 4 — Create Launch Template

> A Launch Template stores EC2 configuration so Auto Scaling can launch identical instances automatically.

### Steps:

1. Go to **EC2 → Launch Templates → Create Launch Template**
2. Configure:
   - **Launch template name:** any (e.g., `web-server-template`)
   - **Template version description:** v1
   - **AMI:** Select the AMI you created above
   - **Instance type:** t2.micro
   - **Key pair:** Select an existing key pair
   - **Security Group:** Select existing security group (All Traffic)
3. **User Data** (under Advanced Details — optional but recommended):

```bash
#!/bin/bash
sudo su
yum install httpd -y
systemctl start httpd
systemctl enable httpd
echo "Welcome from Auto Scaling" > /var/www/html/index.html
```

4. Click **Create Launch Template**

---

## Part 5 — Create Auto Scaling Group (ASG)

### Step 1: Basic Configuration

1. Go to **EC2 → Auto Scaling Groups → Create Auto Scaling Group**
2. Configure:
   - **Name:** any (e.g., `web-asg`)
   - **Launch Template:** Select the one created above
   - **Version:** 1 (default)

---

### Step 2: Network Configuration

1. **VPC:** Select the same VPC used previously (default)
2. **Availability Zones / Subnets:** Select the same subnets/AZs used for the ALB

---

### Step 3: Load Balancer Integration

1. Select **Attach to an existing load balancer**
2. Choose **Target Group** created in Part 1
3. **Health checks:**
   - Enable **ELB health checks** (in addition to EC2 health checks)

---

### Step 4: Group Size (Capacity)

| Setting | Value |
|---|---|
| Minimum capacity | 0 |
| Desired capacity | 2 |
| Maximum capacity | 7 |

> Adjust these values based on your lab requirements.

---

### Step 5: Finish & Create

- Click **Next** through remaining options
- Click **Create Auto Scaling Group**

---

## Part 6 — Create Dynamic Scaling Policy

> Dynamic Scaling automatically adds/removes instances based on real-time metrics.

### Steps:

1. Go to **EC2 → Auto Scaling Groups**
2. Select your ASG → **Automatic Scaling** tab
3. Click **Create Dynamic Scaling Policy**
4. Configure:

| Setting | Value |
|---|---|
| Policy type | Target Tracking Scaling |
| Metric type | Average CPU Utilization |
| Target value | 50 (adjust as needed) |
| Warm-up period | 120 seconds (adjustable) |

5. Click **Create**

---

## Part 7 — Verify Auto Scaling is Working

### Step 1: Check Instances Launched by ASG

After creating the ASG, it will automatically launch EC2 instances based on the **desired capacity**.

Go to **EC2 → Instances** — you should see new instances launched by the ASG.

---

### Step 2: Stress Test (Trigger Scale-Out)

Connect to one of the ASG instances and run a CPU stress test:

```bash
# Connect to instance
sudo su

# Run CPU stress to trigger scaling
yes > /dev/null &
```

> `yes > /dev/null` runs an infinite loop that maxes out a CPU core, pushing CPU utilization above the target value and triggering the scaling policy to add instances.

#### Monitor Scaling Activity

- Go to **EC2 → Auto Scaling Groups → Activity** tab
- Watch new instances being launched automatically

#### Stop the Stress Test

```bash
# Find and kill the yes process
top
# Press 'k', enter the PID of yes process
```

---

## Cleanup (To Avoid Charges)

1. Delete **Auto Scaling Group** (this terminates ASG-managed instances)
2. Terminate any manually launched EC2 instances
3. Delete **Load Balancer**
4. Delete **Target Group**
5. Deregister / Delete **AMI**
6. Delete associated **Snapshots** (from AMI)
7. Delete **Launch Template**
8. Delete **Security Groups** created for this lab

---

## Key Concepts Summary

| Concept | Description |
|---|---|
| **ALB** | Layer 7 load balancer; routes HTTP/HTTPS traffic across instances |
| **Target Group** | Logical grouping of EC2 instances that receive traffic from ALB |
| **AMI** | Snapshot of an EC2 instance; used as the base for new instances |
| **Launch Template** | Reusable EC2 config (AMI, type, SG, user data) for ASG to use |
| **Auto Scaling Group** | Automatically manages a fleet of EC2 instances |
| **Dynamic Scaling Policy** | Scales in/out based on CloudWatch metrics (e.g., CPU) |
| **User Data** | Bash script that runs automatically when an instance launches |
| **Warm-up Period** | Time (seconds) before a new instance is counted in scaling metrics |

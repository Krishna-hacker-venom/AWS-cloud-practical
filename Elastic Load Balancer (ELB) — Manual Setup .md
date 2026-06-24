# AWS Practical: Elastic Load Balancer (ELB) — Manual Setup

> **Topic:** Application Load Balancer (ALB) with 2 EC2 instances — manual configuration and failover testing  
> **Services Used:** EC2, Security Groups, ALB, Target Groups  
> **Cost:** Free tier eligible (t2.micro instances)

---

## Architecture Overview

```
Browser
   |
   v
ALB DNS Name (internet-facing)
   |
   +--> EC2 Instance 1 (AZ-1, Subnet-1) → "Welcome from Instance 1"
   +--> EC2 Instance 2 (AZ-2, Subnet-2) → "Welcome from Instance 2"
```

> ALB distributes incoming traffic across both instances. If one goes down, all traffic routes to the healthy one automatically.

---

## Part 1 — Create Security Group

1. Go to **EC2 → Security Groups → Create Security Group**
2. Configure:

**Inbound Rules:**

| Type | Protocol | Port | Source |
|---|---|---|---|
| HTTP | TCP | 80 | Anywhere (`0.0.0.0/0`) |
| SSH | TCP | 22 | My IP |

**Outbound Rules:**

| Type | Protocol | Port | Destination |
|---|---|---|---|
| All traffic | All | All | `0.0.0.0/0` |

3. Click **Create Security Group**

---

## Part 2 — Launch EC2 Instance 1

1. Go to **EC2 → Launch Instances**
2. Configure:

| Setting | Value |
|---|---|
| AMI | Amazon Linux 2 |
| Instance type | t2.micro |
| **AZ** | Change to AZ-1 (e.g., `ap-south-1a`) |
| **Subnet** | Change to a subnet in AZ-1 |
| Auto-assign Public IP | Enable |
| Security Group | Select existing SG created above |

3. **User Data** (under Advanced Details):

```bash
#!/bin/bash
sudo su
yum install httpd -y
systemctl start httpd
systemctl enable httpd
echo "Welcome from Instance 1" > /var/www/html/index.html
```

4. Click **Launch Instance**

---

## Part 3 — Launch EC2 Instance 2

> Same configuration as Instance 1 — **only change the subnet/AZ**.

1. Go to **EC2 → Launch Instances**
2. Configure:

| Setting | Value |
|---|---|
| AMI | Amazon Linux 2 |
| Instance type | t2.micro |
| **AZ** | Change to AZ-2 (e.g., `ap-south-1b`) |
| **Subnet** | Change to a subnet in AZ-2 |
| Auto-assign Public IP | Enable |
| Security Group | Same existing SG |

3. **User Data:**

```bash
#!/bin/bash
sudo su
yum install httpd -y
systemctl start httpd
systemctl enable httpd
echo "Welcome from Instance 2" > /var/www/html/index.html
```

4. Click **Launch Instance**

> Both instances are now in **different AZs** — this is required for ALB to distribute traffic across them.

---

## Part 4 — Create Target Group

> A Target Group tells the ALB which instances to send traffic to, and monitors their health.

1. Go to **EC2 → Target Groups → Create Target Group**
2. Configure:

| Setting | Value |
|---|---|
| Target type | Instances |
| Protocol | HTTP |
| Port | 80 |
| VPC | Default VPC |

3. **Health Check Settings:**

| Setting | Value |
|---|---|
| Healthy threshold | 2 |
| Unhealthy threshold | 2 |
| Timeout | 5 seconds |
| Interval | 10 seconds |

4. Click **Next**
5. Select **both EC2 instances** → Click **Include as pending below**
6. Click **Create Target Group**

---

## Part 5 — Create Application Load Balancer

1. Go to **EC2 → Load Balancers → Create Load Balancer**
2. Select **Application Load Balancer**
3. Configure:

| Setting | Value |
|---|---|
| Name | any (e.g., `my-alb`) |
| Scheme | Internet-facing |
| IP address type | IPv4 |
| VPC | Default VPC |
| Availability Zones | Select both AZs used by your instances |
| Security Group | Select the SG created in Part 1 |

4. **Listeners & Routing:**
   - Protocol: HTTP | Port: 80
   - Default action: **Forward to** → select the Target Group created above

5. Click **Create Load Balancer**

---

## Part 6 — Test Load Balancing

### Step 1: Get ALB DNS Name

1. Go to **EC2 → Load Balancers**
2. Select your ALB → copy the **DNS name** (e.g., `my-alb-123456.ap-south-1.elb.amazonaws.com`)

### Step 2: Test in Browser

1. Paste the DNS name into a new browser tab
2. You should see: `Welcome from Instance 1` or `Welcome from Instance 2`
3. Keep **refreshing** — the response alternates between instances, proving load balancing is working

---

## Part 7 — Test Failover (High Availability)

> Stop Apache on one instance and verify all traffic routes to the healthy instance.

### Step 1: Connect to Instance 1

Use EC2 Instance Connect or SSH:

```bash
sudo su
```

### Step 2: Stop the Web Server

```bash
systemctl stop httpd
```

### Step 3: Verify Failover in Browser

1. Go back to the browser tab with the ALB DNS name
2. Refresh multiple times
3. Now **only** `Welcome from Instance 2` should appear
4. The ALB detected Instance 1 as unhealthy (failed health check) and stopped routing traffic to it

### Step 4: Restore Instance 1

```bash
systemctl start httpd
```

- Wait ~10–20 seconds for health checks to pass
- Refresh browser — traffic resumes distributing to both instances

---

## Health Check Flow

```
ALB sends GET / to each instance every 10 seconds
       |
       +--> Response 200 OK → Healthy → receives traffic
       +--> No response / timeout → Unhealthy → removed from rotation
```

---

## Key Concepts Summary

| Concept | Description |
|---|---|
| **ALB** | Layer 7 load balancer — routes HTTP/HTTPS traffic |
| **Target Group** | Group of instances that receive traffic from the ALB |
| **Health Check** | ALB periodically pings each instance to verify it's alive |
| **Healthy Threshold** | Consecutive successful checks before marking instance healthy |
| **Unhealthy Threshold** | Consecutive failed checks before removing instance from rotation |
| **Internet-facing** | ALB is accessible from the public internet |
| **Failover** | ALB automatically stops sending traffic to unhealthy instances |
| **DNS Name** | ALB provides a DNS name instead of a static IP |

---

## Cleanup (To Avoid Charges)

1. Delete **Load Balancer**
2. Delete **Target Group**
3. Terminate both **EC2 Instances**
4. Delete the **Security Group** created for this lab

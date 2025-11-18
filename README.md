Project Overview

This project provisions a production-ready infrastructure that includes:

A Custom VPC with public & private subnets
EC2 Auto Scaling Group hosting the web application
Application Load Balancer routing incoming traffic
Amazon RDS (MySQL/PostgreSQL) in a private subnet
CloudWatch for monitoring and alarms
SNS for real-time notifications
Optional Route 53 for custom domain routing

Architecture Diagram
                           ┌──────────────────────────────┐
                           │        Internet Users         │
                           └───────────────┬──────────────┘
                                           │
                                   ┌───────▼────────┐
                                   │ Application     │
                                   │ Load Balancer   │ (Public Subnets)
                                   └───────┬────────┘
                                           │
                     ┌─────────────────────┴─────────────────────┐
                     │                                           │
        ┌────────────▼────────────┐               ┌──────────────▼────────────┐
        │   EC2 Instance (AZ1)    │               │   EC2 Instance (AZ2)       │
        │   Auto Scaling Group    │               │   Auto Scaling Group        │
        └────────────┬────────────┘               └──────────────┬─────────────┘
                     │                                           │
                     └─────────────────────┬─────────────────────┘
                                           │
                              ┌────────────▼─────────────┐
                              │      Amazon RDS DB        │ (Private Subnets)
                              │  (MySQL/PostgreSQL)      │
                              └───────────────────────────┘

CloudWatch → Metrics, Logs, Alarms → SNS → Email/SMS Notifications

 Features
✔ Custom Virtual Private Cloud (VPC)
Public subnets for ALB
Private subnets for EC2 and RDS
Internet Gateway + NAT Gateway
Strict network isolation and routing

✔ Auto Scaling Web Tier
EC2 instances scale automatically
Launch Template with bootstrap script
High availability across multiple AZs

✔ Application Load Balancer
Handles routing, health checks, failover
Supports HTTP/HTTPS
Sticky sessions (optional)

✔ Amazon RDS Connectivity
Supports MySQL / PostgreSQL
Runs in private subnets (not publicly accessible)
Uses Security Groups for EC2-to-DB access
Multi-AZ failover supported

✔ CloudWatch Monitoring
Metrics for EC2, ALB, and RDS
Dashboards and logs
Alarms for CPU, latency, DB connections

✔ SNS Notifications
Email/SMS alerts for issues
Triggered by CloudWatch

Alerts for scaling events, high usage, DB failures

Project Structure
├── README.md
├── /scripts
│   └── user-data.sh               # Web server installation + DB connection setup
├── /infrastructure
│   ├── vpc.yaml                   
│   ├── alb.yaml                   
│   ├── ec2-asg.yaml               
│   ├── rds.yaml                   # RDS database configuration
│   ├── monitoring.yaml            
│   └── iam.yaml                   
└── /website
    └── index.php (or HTML)        # Sample app with DB connectivity

RDS Database Setup
1. Create an RDS Instance
Recommended engine:
MySQL 8.x
PostgreSQL 15+

Use:
Multi-AZ enabled (recommended)
Private subnets only
Security Group allowing access ONLY from EC2 instances

2. Database Security Group
Inbound: Allow TCP 3306 (MySQL) or 5432 (PostgreSQL)
Source: EC2 security group
(Not from internet!)

EC2 to RDS Connectivity
Example PHP test script:
<?php
$servername = "your-rds-endpoint.amazonaws.com";
$username = "admin";
$password = "YourDBPassword";
$dbname = "mydb";

$conn = new mysqli($servername, $username, $password, $dbname);

if ($conn->connect_error) {
    die("Connection failed: " . $conn->connect_error);
}
echo "Database Connected Successfully!";
?>

Example user-data script (install Apache + PHP + database client):
#!/bin/bash
yum update -y
yum install -y httpd php php-mysqli
systemctl enable httpd
systemctl start httpd

cat <<EOF > /var/www/html/dbtest.php
<?php
\$conn = new mysqli("${db_endpoint}", "admin", "password", "mydb");
if (\$conn->connect_error) { die("DB Connection Failed: " . \$conn->connect_error); }
echo "DB Connected Successfully!";
?>
EOF

Deployment Steps
1. Create the Custom VPC
Create subnets (public + private)
Add IGW & NAT gateway
Configure routing tables

2. Create Security Groups
ALB SG → inbound from internet
EC2 SG → inbound only from ALB SG
RDS SG → inbound only from EC2 SG

3. Deploy RDS in Private Subnets
Choose DB engine
Create admin credentials
Store credentials in SSM Parameter Store (recommended)
Ensure connectivity test succeeds

4. Create EC2 Launch Template
Includes:
AMI
Key pair
Security group
User-data script (installs web server + DB client)

5. Create Auto Scaling Group
Uses private subnets
Desired capacity: 2
Load balancing enabled

6. Deploy Application Load Balancer
Public subnets
Listener on port 80/443
Target group for EC2 instances

7. Configure CloudWatch Alarms
Examples:
CPUUtilization > 80%
RDS Free Storage < threshold
ALB 5xx errors high
Auto Scaling activities

8. Configure SNS
Create SNS topic
Add email/SMS subscriber
Add CloudWatch → SNS action

Testing Checklist
Website:
Access ALB DNS name
Ensure load balancer distributes traffic

Database:

Visit /dbtest.php
Should show: DB Connected Successfully!

Monitoring:
Check CloudWatch metrics & alarms
Trigger alarms to confirm SNS works

Auto Scaling:
Increase traffic to trigger scale-out
Stop an instance → ALB replaces it

Security Considerations
Keep RDS in private subnets only
Use SSM Parameter Store or Secrets Manager for DB credentials
Restrict SSH access (use Session Manager)
Enable ALB access logs
Enable RDS encryption at rest

Alerts & Notifications
SNS alerts include:
High CPU on EC2
RDS storage threshold
RDS failover events
Application errors (5xx)
Auto Scaling activities

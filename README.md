  Highly Available and Scalable Web Application Infrastructure on AWS
An end-to-end AWS CloudOps project demonstrating multi-Availability Zone hosting, load balancing, automatic recovery, CPU-based scaling, infrastructure monitoring, and HTTPS access through a custom domain.
Visit the live website
> **Project status:** Core infrastructure implemented and successfully tested. Custom-domain HTTPS/TLS deployment completed. The live website is hosted at **https://cloudproject.projectstackhub.com**.
Project Overview
This project deploys a lightweight HTML website using Apache HTTP Server on Amazon EC2 behind an Application Load Balancer (ALB). An EC2 Auto Scaling Group manages instances across two AWS Availability Zones, allowing the application to recover from instance failures and adjust capacity in response to CPU utilization.
The infrastructure is built inside a custom Amazon VPC with public and private subnets. Amazon CloudWatch provides visibility into infrastructure metrics and scaling alarms. A custom domain and an AWS Certificate Manager (ACM) TLS certificate provide browser-to-ALB HTTPS access.
The goal is to demonstrate practical cloud engineering skills: networking, Linux administration, application deployment, high availability, scaling, monitoring, security, and systematic troubleshooting.
Live Application
Website: https://cloudproject.projectstackhub.com
Application: Static HTML page served by Apache (`httpd`)
Displayed heading: `Welcome to My CloudOps Project`
AWS Region: `us-east-1` (US East — N. Virginia)
> The project uses live AWS resources. Availability may depend on whether the EC2 Auto Scaling capacity is running; resources may be scaled down between demonstrations to control costs.
AWS Services and Technologies
Service / Technology	Role in the project
Amazon VPC	Isolated networking with public and private subnets
Internet Gateway and route tables	Internet connectivity for the public subnets
Amazon EC2	Amazon Linux 2023 web servers hosting Apache and the HTML page
EC2 Launch Template	Reproducible EC2 configuration and bootstrap User Data
EC2 Auto Scaling	Instance lifecycle, desired-capacity maintenance, CPU-based scale-out and scale-in
Application Load Balancer	Incoming web traffic routing across healthy targets
ALB Target Group	HTTP health checks and backend EC2 registration
Amazon CloudWatch	EC2/Auto Scaling metrics, target-tracking alarms, and scaling observability
AWS Certificate Manager	TLS certificate for custom-domain HTTPS access
DNS	Routes the custom subdomain to the AWS load balancer
IAM and Security Groups	AWS access management and network-level traffic restrictions
Linux, Apache, Bash	Web hosting and automated instance bootstrap
Architecture
```text
                   Internet users
                         |
                         | HTTPS (TCP 443)
                         v
          cloudproject.projectstackhub.com
                         |
                 Custom-domain DNS
                         |
                         v
             Application Load Balancer
           (public subnets in 2 AZs)
                 /             \
                /               \
       Target Group: HTTP (80), / health check
                \               /
                 \             /
                EC2 Auto Scaling Group
                  (Launch Template v3)
                 /               \
                v                 v
      Availability Zone 1a   Availability Zone 1b
        EC2 + Apache          EC2 + Apache
         Public subnet         Public subnet
                 \               /
                  \_____________/
                       |
                 Amazon CloudWatch
            (metrics, alarms, scaling)

       Custom VPC also includes 2 private subnets
       reserved for future architecture enhancements.
```
Network and request flow
A visitor opens `https://cloudproject.projectstackhub.com`.
DNS resolves the custom hostname to the Application Load Balancer.
The ALB accepts the HTTPS connection using an ACM-managed TLS certificate.
The ALB forwards the request to a healthy EC2 target through the HTTP target group on port 80.
Apache serves the HTML webpage, and the response returns to the visitor through the ALB.
Auto Scaling maintains the required number of instances, replaces failed instances, and adjusts capacity in response to the configured CPU target-tracking policy.
Networking note: The tested Apache instances were deployed in public subnets across `us-east-1a` and `us-east-1b`. The private subnets exist in the VPC but were not used to host these web instances. This distinction is important when interpreting the architecture.
Implementation Highlights
1. VPC and networking
Created a custom VPC (`10.0.0.0/16`) spanning two Availability Zones.
Configured two public and two private subnets, an Internet Gateway, and public route-table associations.
Used dedicated security groups for the load balancer and EC2 web servers.
2. Web server bootstrap
Deployed Amazon Linux 2023 EC2 instances with Apache HTTP Server.
Hosted a custom static HTML page in `/var/www/html/index.html`.
Automated Apache installation and startup using EC2 Launch Template User Data.
Corrected the User Data bootstrap configuration and created Launch Template version 3 for subsequent instances.
3. Load balancing and high availability
Created an internet-facing ALB and an HTTP target group using a `/` health-check path.
Associated the target group with the Auto Scaling Group.
Confirmed two healthy backend targets in two different Availability Zones.
4. Automatic scaling
Configured a target-tracking policy using 50% average CPU utilization.
During testing, allowed capacity to scale between 2 and 3 instances.
Verified both automatic scale-out and scale-in using Auto Scaling Activity History.
5. Monitoring
Enabled Auto Scaling Group metrics collection in CloudWatch.
Inspected the automatically managed target-tracking AlarmHigh and AlarmLow alarms.
Used CloudWatch alarm information and Auto Scaling Activity History to understand scaling decisions.
6. HTTPS and custom domain
Connected the application to the custom hostname `cloudproject.projectstackhub.com`.
Completed custom-domain HTTPS/TLS configuration using an ACM certificate and the existing ALB.
Kept the existing EC2/ALB/Auto Scaling application architecture rather than rebuilding it for the domain.
Testing and Verified Results
Test	What was verified	Result
Apache startup	Website responded locally with HTTP `200 OK`	Passed
ALB accessibility	Webpage loaded successfully through the ALB DNS name	Passed
Multi-AZ deployment	ASG instances operated in `us-east-1a` and `us-east-1b`	Passed
Target health	Two EC2 targets were reported healthy by the ALB target group	Passed
Instance recovery	An ASG-managed EC2 instance was intentionally terminated and replaced	Passed
Website continuity	Website remained accessible during the tested replacement scenario	Passed
Recovery completion	Two healthy ALB targets were restored after replacement	Passed
Automatic scale-out	CPU target-tracking activity increased desired capacity from 2 to 3	Passed
Automatic scale-in	Subsequent low-load activity reduced desired capacity from 3 to 2	Passed
Monitoring	CloudWatch ASG metrics and target-tracking alarms were inspected	Passed
Network controls	EC2 HTTP ingress restricted to the ALB security group	Reviewed
Custom-domain HTTPS	Application accessible at the custom HTTPS URL	Completed
> Tests record outcomes observed during the project. A successfully tested failover does not by itself guarantee uninterrupted service under every possible failure scenario.
Troubleshooting and Lessons Learned
EC2 User Data bootstrap failure
Symptom: Newly launched Auto Scaling instances failed during `cloud-final.service`, and the ALB could not use those instances as healthy web targets.
Diagnosis: Inspected `/var/log/cloud-init-output.log`, `/var/log/cloud-init.log`, and the generated User Data script. The script had an empty line before its `#!/bin/bash` shebang, leading to an `Exec format error`.
Resolution: Removed the leading blank line, successfully executed the corrected script (`exit code 0`), and confirmed the Apache endpoint returned HTTP `200`. Created Launch Template version 3 with the corrected User Data and verified fresh ASG instances became healthy ALB targets.
Lesson: Linux User Data shell scripts must start with the interpreter directive at the beginning of the file; launch health and application health must be checked separately.
Unexpected Auto Scaling scale-to-zero
Symptom: Instances were terminated unexpectedly during an instance-refresh test.
Diagnosis: Auto Scaling Activity History showed a low-CPU target-tracking alarm reducing desired capacity from `1` to `0`, which was allowed by the then-current minimum-capacity setting.
Resolution: Adjusted minimum and desired capacity for controlled testing, preventing the CPU scaling policy from reducing the group below the intended test capacity.
Lesson: Always inspect Auto Scaling policies, activity history, and minimum/maximum capacity together when diagnosing instance termination.
SSH and CloudWatch troubleshooting
Used SSH and Cloud Init logs to diagnose instance bootstrap issues instead of repeatedly changing infrastructure settings.
Distinguished EC2/ASG health from ALB target health.
Inspected CloudWatch alarm states and Auto Scaling Activity History to verify that CPU-driven scale-out and scale-in actually occurred.
Security Design
ALB security group: Accepts public web traffic; HTTPS is served on port 443.
EC2 web security group: Accepts HTTP port 80 from the ALB security group, rather than from unrestricted internet sources.
SSH: Port 22 access was limited to an explicitly allowed administrator IP (`/32`) during testing.
TLS: ACM-managed certificate secures the browser-to-ALB HTTPS connection.
IAM: AWS account permissions and resource access are managed separately from web application traffic controls.
Production considerations
This is a tested cloud engineering portfolio deployment, not a claim of full enterprise production hardening. Current design considerations include:
The ALB-to-EC2 backend connection uses HTTP, so end-to-end TLS is not configured.
The tested EC2 web instances reside in public subnets. A future production enhancement would place application instances in private subnets and plan secure outbound access.
Further production improvements could include AWS WAF, centralized application logging, alert notifications, deployment pipelines, infrastructure as code, and stronger operational runbooks.
Domain and HTTPS availability depend on correct DNS, certificate validity, ALB configuration, and sufficient running backend capacity.
Portfolio Evidence
Screenshots captured during implementation and testing:
Evidence	Filename
Website served through an Auto Scaling instance behind the ALB	`Project1_ASG_ALB_Website_Success.png`
Two healthy targets across Availability Zones	`Project1_MultiAZ_Two_Healthy_Targets.png`
Two healthy targets restored after an instance replacement	`Project1_AutoRecovery_Two_Healthy_Targets.png`
CPU-triggered scale-out and scale-in Activity History	`Project1_CPU_AutoScaling_ScaleOut_ScaleIn.png`
These screenshots document the observed AWS Console and browser results. No credentials, private keys, or sensitive account details should be uploaded.
Cost Awareness and Resource Lifecycle
EC2, public IPv4 addresses, EBS storage, and Application Load Balancer usage can generate charges or consume AWS promotional credits.
The ALB can continue accruing charges even when the Auto Scaling Group has zero instances.
During development, the ASG was scaled down to desired/minimum/maximum capacity `0/0/0` when compute instances were not needed; this shuts down ASG-managed instances rather than stopping the group itself.
Before demonstrating the live application, the ASG must have healthy running instances. A live URL should not be interpreted as a guarantee that the application is permanently running at full capacity.
Budget alerts notify about spending thresholds but do not automatically stop AWS charges.
Skills Demonstrated
AWS networking · EC2 · Linux administration · Apache HTTP Server · Application Load Balancer · Target Groups and health checks · EC2 Auto Scaling · Multi-AZ architecture · CloudWatch alarms and metrics · CPU target tracking · AWS Certificate Manager · DNS · HTTPS/TLS · IAM and Security Groups · Troubleshooting · Cost awareness
Project Outcome
Implemented and tested a highly available, CPU-scalable Apache web application on AWS, including:
Two-AZ deployment and ALB health-based routing.
Automatic instance replacement after a controlled failure.
Verified CPU-based 2 → 3 → 2 scaling activity.
CloudWatch monitoring and documented troubleshooting.
Custom-domain HTTPS access at https://cloudproject.projectstackhub.com.
The project demonstrates practical experience operating and validating a cloud-based web application, while explicitly documenting architectural trade-offs and opportunities for further hardening.

CloudBridge Technologies — AWS Deployment
A responsive static website for CloudBridge Technologies, built with HTML, CSS, and JavaScript and deployed on AWS using CentOS EC2 instances, an Auto Scaling Group (ASG), an Application Load Balancer (ALB), a custom VPC, Route 53, AWS Certificate Manager (ACM), and CloudFront.
Project Overview
The website presents CloudBridge Technologies' cloud engineering, consulting, DevOps, and training services. The frontend is made up of:
- index.html — page structure and content
- style.css — layout, responsive styling, and visual effects
- script.js — navigation menu, animated statistics, scroll-reveal effects, and other client-side interactions
- profile.jpeg, flyer1.jpeg, flyer2.jpeg — site images
- assets/favicon.png — browser favicon
The site uses the Poppins font, Font Awesome icons, and custom CSS-drawn AWS-style icons. Those external font/icon resources require internet access in the visitor's browser unless you choose to self-host them.
AWS Architecture
Services used
Service	Purpose
Amazon VPC	Isolated network for the application
Four subnets across two Availability Zones	Network segmentation and multi-AZ design
Amazon EC2 (CentOS)	Runs the web server and serves the website files
Auto Scaling Group (ASG)	Maintains the desired number of EC2 instances and can replace unhealthy instances
Application Load Balancer (ALB)	Receives web requests and forwards them to registered targets
ALB Target Group	Health-checks and routes requests to the web-server instances
Amazon Route 53	Hosts DNS records for the custom domain
Go54	Domain registrar; the domain's DNS delegation should point to Route 53 name servers
AWS Certificate Manager (ACM)	Provides TLS certificates for HTTPS
Amazon CloudFront	CDN and public entry point for cached delivery, with the ALB configured as the origin


Network layout
The VPC contains four subnets across two Availability Zones. A typical layout is:
- Availability Zone A: public subnet A and private subnet A
- Availability Zone B: public subnet B and private subnet B
Public subnets normally route outbound internet traffic through an Internet Gateway. Private subnets normally use a NAT Gateway for outbound internet access when required, or VPC endpoints for supported AWS services.
The current deployment was described as having two running instances, with one instance in public subnet A and one in private subnet B. This is documented as the reported arrangement, but it is not the recommended final production layout.
Security recommendation: place both application EC2 instances in private subnets and place the internet-facing ALB in the two public subnets. Configure the ASG to use the private subnets across both Availability Zones. Allow inbound HTTP/HTTPS to the ALB from the intended sources, and allow the EC2 security group to receive the application traffic only from the ALB security group. Avoid exposing SSH (port 22) to the internet; use Systems Manager Session Manager or a restricted administrative path where possible.

Request flow
Visitor
  |
  | HTTPS — custom domain
  v
Route 53 DNS
  |
  v
CloudFront distribution
  |
  | HTTPS to origin
  v
Application Load Balancer (public subnets in AZ A and AZ B)
  |
  v
ALB Target Group
  |
  +-------------------------------+
  |                               |
  v                               v
EC2 CentOS web server         EC2 CentOS web server
Private subnet A              Private subnet B
Availability Zone A           Availability Zone B
  |                               |
  +---------------+---------------+
                  |
           Auto Scaling Group
      maintains desired capacity and
       replaces unhealthy instances
The diagram shows the recommended production arrangement. If an instance is currently in public subnet A, verify whether it is intentionally public and plan a controlled move into a private subnet before treating this diagram as the exact current state.
DNS and HTTPS
1. Keep the domain registered with Go54.
2. Create a public hosted zone for the domain in Route 53.
3. Update the domain's nameservers at Go54 to the four Route 53 nameservers shown in the hosted zone.
4. Request or import the required ACM certificate(s).
5. For a CloudFront viewer certificate, the ACM certificate must be in US East (N. Virginia), us-east-1.
6. If the ALB origin also uses HTTPS, request or use a separate ACM certificate in the same AWS Region as the ALB and configure an HTTPS listener on the ALB.
7. Configure CloudFront to use the ALB DNS name as its origin, set the appropriate viewer protocol policy (typically redirect HTTP to HTTPS), and attach the alternate domain name(s) and viewer certificate.
8. Create the Route 53 alias record(s) pointing the custom domain to the CloudFront distribution. Configure both the apex/root domain and www if both should work.
9. Wait for DNS propagation and certificate validation. Confirm that the CloudFront distribution is deployed before testing.
Important: Route 53 does not register the domain in this setup; Go54 remains the registrar. Route 53 provides DNS hosting after the nameserver delegation is updated.
Deployment prerequisites
- An AWS account with permission to manage VPC, EC2, Auto Scaling, Elastic Load Balancing, Route 53, ACM, and CloudFront.
- A CentOS-compatible EC2 AMI and a working SSH/Systems Manager administrative path.
- A configured VPC with four subnets across two Availability Zones.
- An internet-facing ALB in the public subnets.
- A target group configured for the web server's listening port and a valid health-check path.
- An ASG attached to the target group and configured across the intended private subnets.
- Security groups that allow only the required traffic.
- A domain registered at Go54 and a Route 53 public hosted zone.
- Valid ACM certificates in the correct Regions.
Deploying the website files on CentOS
The archive contains the frontend files, not an EC2 image or infrastructure-as-code deployment. Copy the site files to each instance (or use a repeatable launch template/user-data deployment process), then configure a web server such as Apache HTTP Server or Nginx to serve them.
Example using Apache on a CentOS-compatible system:
# Install Apache (package availability can differ by CentOS version)
sudo dnf install -y httpd

# Start Apache and enable it at boot
sudo systemctl enable --now httpd

# Create the web root
sudo mkdir -p /var/www/html

# Copy index.html, style.css, script.js, profile.jpeg, flyer1.jpeg,
# flyer2.jpeg, and the assets/ directory into /var/www/html/.
# Run these commands from the directory containing the site files:
sudo cp index.html style.css script.js profile.jpeg flyer1.jpeg flyer2.jpeg /var/www/html/
sudo cp -r assets /var/www/html/

# Ensure Apache can read the files
sudo chown -R root:root /var/www/html
sudo find /var/www/html -type d -exec chmod 755 {} \;
sudo find /var/www/html -type f -exec chmod 644 {} \;

# Check the service
sudo systemctl status httpd
On systems where dnf is not available, use the package manager supported by the installed CentOS release. Confirm that the web server listens on the port configured in the target group (commonly HTTP port 80).
Health checks
- Set the target group's health-check protocol and port to match the web server.
- Use / as the health-check path if the home page returns a successful status.
- Verify that the target group reports healthy targets before testing through the ALB or CloudFront.
- Confirm that the instance security group allows traffic from the ALB security group on the target port.
Auto Scaling notes
- Associate the target group with the ASG.
- Configure the ASG to launch instances in the intended subnets across both Availability Zones.
- Use a launch template to define the AMI, instance type, IAM instance profile, security groups, storage, and user data.
- Choose a desired/minimum/maximum capacity appropriate for the workload and budget.
- Keep website deployment repeatable. A manually copied file on one instance may disappear when ASG replaces that instance; use user data, a deployment pipeline, or an artifact source so every new instance receives the same release.
CloudFront considerations
- Use the ALB as the origin when CloudFront is intended to sit in front of the application.
- Use HTTPS between CloudFront and the ALB where possible.
- Restrict direct access to the ALB origin if the architecture requires traffic to come through CloudFront. Do this carefully so health checks and operational access continue to work.
- Choose cache policies appropriate for the assets. HTML may need shorter caching or invalidations after releases; versioned CSS, JavaScript, and image assets can usually be cached longer.
- After deploying an update, create a CloudFront invalidation for changed paths if the cache policy would otherwise keep serving older content.
Application notes and limitations
- The contact form currently contains a placeholder endpoint check in script.js. It is not connected to a live form-processing backend until a real endpoint is configured.
- The frontend includes external Google Fonts and Font Awesome CDN links.
- The animated counters in the hero section are presentation values from the HTML; they should be updated to verified business metrics before public use.
- The included files do not contain AWS credentials, VPC configuration, launch templates, ASG definitions, ALB listeners, target groups, Route 53 records, ACM certificates, or CloudFront configuration. Those resources must be configured separately in AWS.
Validation checklist
- [ ] All site files and the assets/ directory are present on every serving instance.
- [ ] Apache/Nginx starts automatically after reboot.
- [ ] The target group reports both intended EC2 targets as healthy.
- [ ] The ASG spans the intended subnets and Availability Zones.
- [ ] EC2 instances are not unnecessarily exposed to the public internet.
- [ ] ALB listener and target-group ports match the web server configuration.
- [ ] Go54 nameservers point to the Route 53 hosted zone's nameservers.
- [ ] ACM certificates are issued and in the correct Regions.
- [ ] CloudFront uses the intended ALB origin and HTTPS settings.
- [ ] Route 53 alias records point to CloudFront.
- [ ] The custom domain loads over HTTPS with no certificate warnings.
- [ ] Mobile navigation, page sections, images, and JavaScript interactions work.
- [ ] The contact form has a real backend or is clearly marked as not yet available.
- [ ] CloudWatch alarms/logging and AWS billing alerts are configured.
Repository structure
cloudbridge/
├── index.html
├── style.css
├── script.js
├── profile.jpeg
├── flyer1.jpeg
├── flyer2.jpeg
├── assets/
│   └── favicon.png
└── README.md
Cost and security reminder
AWS services such as NAT Gateways, ALB, EC2, CloudFront, Route 53 hosted zones and DNS queries, and data transfer may incur charges. Review pricing and remove resources that are no longer needed. Follow least-privilege IAM practices, keep the operating system patched, protect administrative access, and monitor resource health and costs.

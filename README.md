# Using Auto Scaling in AWS (Linux)

## 1. Project Overview

This project demonstrates how to deploy a scalable Linux web application on Amazon Web Services (AWS) using Amazon EC2, an Application Load Balancer, and EC2 Auto Scaling.

I used the AWS Command Line Interface (AWS CLI) to launch a web server and create a custom Amazon Machine Image (AMI). I then used the AWS Management Console to configure an Application Load Balancer, launch template, Auto Scaling group, and target tracking scaling policy.

Finally, I tested the application under increased CPU load and observed AWS launch an additional EC2 instance automatically.

## 2. Objectives

- Launch an EC2 instance using the AWS CLI.
- Create a custom AMI using the AWS CLI.
- Create an EC2 launch template.
- Configure an EC2 Auto Scaling group.
- Configure target tracking based on average CPU utilization.
- Distribute web traffic through an Application Load Balancer across multiple Availability Zones.
- Verify automatic scale-out under increased workload.

## 3. AWS Services and Tools

| Service or Tool | Purpose |
|---|---|
| Amazon EC2 | Runs the Linux web servers |
| AWS CLI | Launches the initial EC2 instance and creates the AMI |
| Amazon Machine Image (AMI) | Provides a reusable image for new web servers |
| Application Load Balancer | Distributes incoming HTTP requests |
| Target Groups | Register instances and perform health checks |
| EC2 Launch Templates | Define the configuration for newly launched instances |
| EC2 Auto Scaling | Maintains capacity and adjusts the number of instances |
| Amazon VPC | Provides network isolation and subnet configuration |
| CloudWatch metrics | Supply CPU utilization metrics used by target tracking |

## 4. Architecture

The solution uses an internet-facing Application Load Balancer in public subnets and EC2 web servers in private subnets across two Availability Zones.

**Traffic flow:**

Internet → Application Load Balancer → Target Group → EC2 Web Servers

**Scaling flow:**

Increased application load → Higher CPU utilization → Target tracking policy responds → Additional EC2 instance launches

### Network design

- **Public subnets:** Host the internet-facing load balancer.
- **Private subnets:** Host the Auto Scaling group's EC2 instances.
- **Target group:** Routes requests to registered web servers and checks their health.
- **Auto Scaling group:** Maintains the desired capacity and can add instances when required.

## 5. Implementation

### Step 1: Configure the AWS CLI

Connected to the preconfigured Command Host instance using EC2 Instance Connect.

Verified the AWS Region and configured the AWS CLI for the lab environment.

### Step 2: Launch the initial web server

Used the AWS CLI to launch an EC2 instance with the supplied AMI, `t3.micro` instance type, HTTP access security group, subnet, and user-data script.

The user-data script installed and configured the web application.

Waited for the instance to enter the running state and retrieved its public DNS name.

### Step 3: Verify the web application

Opened the instance's public DNS address in a browser and accessed `/index.php`.

Confirmed that the web application loaded successfully before creating the image.

### Step 4: Create a custom AMI

Created an image named `WebServerAMI` from the working web server.

This image provides the base configuration for instances launched by the Auto Scaling group.

### Step 5: Create the Application Load Balancer

Created an internet-facing Application Load Balancer named `WebServerELB`.

Configured it to use both public subnets in the Lab VPC and the existing `HTTPAccess` security group.

Configured an HTTP listener on port 80.

### Step 6: Create the target group

Created a target group named `webserver-app` with the following configuration:

- Target type: Instances
- Protocol: HTTP
- Port: 80
- Health check path: `/index.php`

Configured the load balancer to forward requests to this target group.

### Step 7: Create the launch template

Created `web-app-launch-template` using the custom `WebServerAMI`.

Configured the `t3.micro` instance type, excluded the key pair from the template, and selected the `HTTPAccess` security group.

### Step 8: Create the Auto Scaling group

Created `Web App Auto Scaling Group` in the Lab VPC across two private subnets:

- Private Subnet 1: `10.0.2.0/24`
- Private Subnet 2: `10.0.4.0/24`

Attached the `webserver-app` target group and enabled Elastic Load Balancing health checks.

Configured the group with a desired capacity of two instances, a minimum of two, and a maximum of four.

Added the instance tag `Name=WebApp`.

### Step 9: Configure the scaling policy

Configured a target tracking scaling policy with these settings:

| Setting | Value |
|---|---|
| Metric | Average CPU utilization |
| Target value | 50% |
| Minimum capacity | 2 instances |
| Desired capacity | 2 instances |
| Maximum capacity | 4 instances |

This policy allows Auto Scaling to adjust capacity as workload changes, aiming to keep average CPU utilization close to the configured target.

### Step 10: Verify the deployment

Verified that the Auto Scaling group launched two instances.

Waited for the instances to pass their status checks and for the target group to report healthy targets.

Opened the application using the load balancer's DNS name and confirmed that it loaded successfully.

### Step 11: Test automatic scaling

Selected **Start Stress** in the web application to generate CPU load.

Monitored the Auto Scaling group's activity history in the AWS Management Console.

AWS began launching an additional EC2 instance in response to the increased workload. I verified that the new instance became healthy.

## 6. Test Results

| Test | Result |
|---|---|
| Initial web server deployment | Successful |
| Custom AMI creation | Successful |
| Application access through the load balancer | Successful |
| Initial Auto Scaling capacity | Two instances |
| Target group health checks | Healthy instances verified |
| Stress test | Executed successfully |
| Automatic scale-out | Additional instance launched |
| New instance health | Healthy status verified |

## 7. Screenshots and Evidence

Screenshots will document the following stages of the lab:

1. Initial EC2 web server running.
2. Custom AMI named `WebServerAMI`.
3. Application Load Balancer configuration.
4. Target group showing healthy instances.
5. Launch template configuration.
6. Auto Scaling group capacity settings.
7. Target tracking policy with a 50% CPU utilization target.
8. EC2 instances before and after scale-out.
9. Web application loading through the load balancer.
10. Auto Scaling activity history showing the additional instance launch.

## 8. Skills Demonstrated

- Linux web server deployment
- AWS CLI usage
- EC2 provisioning and AMI creation
- Application Load Balancer configuration
- Target group health checks
- VPC and subnet configuration
- Launch template creation
- EC2 Auto Scaling
- Target tracking scaling policies
- Application testing and infrastructure monitoring
- Troubleshooting and validation of cloud deployments

## 9. Key Takeaways

This lab provided hands-on experience deploying a web application designed to handle changing workloads.

I learned how to create a reusable EC2 image, distribute traffic through an Application Load Balancer, deploy instances across private subnets in multiple Availability Zones, and use a target tracking policy to respond to increased CPU demand.

These skills form a foundation for building scalable and highly available AWS infrastructure.

**Environment note:** This project was completed in a training lab. Resource settings and availability may differ in production environments.

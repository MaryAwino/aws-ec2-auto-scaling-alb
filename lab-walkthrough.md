# AWS EC2 Auto Scaling and Application Load Balancer — Lab Walkthrough

## 1. Project Overview

This project documents a hands-on AWS lab demonstrating how to deploy a Linux web application on Amazon EC2, distribute incoming requests through an Application Load Balancer, and automatically adjust the number of web server instances using an EC2 Auto Scaling group.

The lab combined AWS CLI commands with configuration and verification through the AWS Management Console.

**AWS Region:** `us-west-2`

**Operating system:** Amazon Linux

**Instance type:** `t3.micro`

## 2. Objectives

The objectives of this lab were to:

- Configure and use the AWS CLI.
- Launch and test a Linux web server on EC2.
- Create a reusable Amazon Machine Image (AMI).
- Configure an Application Load Balancer.
- Create a target group and health check.
- Create an EC2 launch template.
- Deploy an Auto Scaling group across private subnets.
- Configure target tracking based on CPU utilization.
- Test the application through the load balancer.
- Observe a scale-out event during a load test.

## 3. AWS Services Used

| Service | Purpose |
|---|---|
| Amazon EC2 | Hosts the Linux web servers |
| Amazon Machine Image (AMI) | Provides a reusable image for launching instances |
| Application Load Balancer | Distributes incoming HTTP requests |
| Target Group | Registers web servers and monitors their health |
| Launch Template | Defines the configuration for new EC2 instances |
| EC2 Auto Scaling | Maintains capacity and adjusts the number of instances |
| Amazon VPC | Provides network isolation and subnets |
| Security Groups | Control permitted network traffic |
| AWS CLI | Provides command-line access to AWS services |

## 4. Architecture

The lab used an internet-facing Application Load Balancer in public subnets and EC2 web servers in private subnets.

Request flow:

1. A user accesses the application using the load balancer's DNS name.
2. The Application Load Balancer receives the HTTP request on port 80.
3. The load balancer forwards the request to a healthy registered target.
4. The EC2 web server processes the request and returns the web page.
5. Auto Scaling can launch additional instances when the scaling policy and its evaluation conditions call for increased capacity.

### Network layout

- **Public Subnet 1:** Hosts one load balancer node.
- **Public Subnet 2:** Hosts another load balancer node.
- **Private Subnet 1:** `10.0.2.0/24`
- **Private Subnet 2:** `10.0.4.0/24`

The private subnets were selected for the Auto Scaling group's EC2 instances.

The load balancer was configured to accept HTTP traffic on port 80. The security groups controlled the allowed traffic between the load balancer and web servers.

## 5. Configure the AWS CLI

The AWS CLI was used to interact with AWS resources.

Run the configuration command in the provided lab environment:

```bash
aws configure
```

Enter the lab-provided credentials when prompted and configure the region as `us-west-2`.

Verify the configured CLI settings:

```bash
aws configure list
```

The configured region was:

```text
us-west-2
```

Security reminder: Never upload AWS secret keys, access keys, session tokens, or credentials to GitHub.

## 6. Launch the Initial EC2 Web Server

An initial EC2 instance was launched to host the web application before the custom AMI and Auto Scaling configuration were created.

The instance used the `t3.micro` instance type.

The launch configuration included:

- A Linux AMI supplied by the lab.
- The `vockey` key pair.
- The `HTTPACCESS` security group.
- A subnet in the lab VPC.
- A user-data script to install and start the web server software.

The initial instance ID recorded during the lab was:

`i-0f7f39073e53eae2a`

The instance was launched using the AWS CLI.

### Example CLI command

The following is an example command structure. Replace the placeholder with the actual initial AMI ID if you need to run it again.

```bash
aws ec2 run-instances \
  --image-id YOUR_INITIAL_AMI_ID \
  --instance-type t3.micro \
  --key-name vockey \
  --security-group-ids sg-0719f226d722e7b05 \
  --subnet-id subnet-007f2b612f5b8eb3c \
  --count 1 \
  --region us-west-2
```

### Verify the instance

```bash
aws ec2 describe-instances \
  --instance-ids i-0f7f39073e53eae2a \
  --region us-west-2
```

The instance was allowed to initialize before the application was tested.

## 7. Verify the Web Application

After the EC2 instance was running, its public DNS name was retrieved.

The web application was accessed using the `/index.php` path.

Example URL format:

```text
http://PUBLIC_DNS_NAME/index.php
```

Replace `PUBLIC_DNS_NAME` with the public DNS name returned by AWS.

The application loaded successfully in the browser, confirming that the web server was responding.

## 8. Create a Custom AMI

A custom AMI named `WebServerAMI` was created from the configured web server.

This image provided a reusable template for launching additional instances with the application environment already prepared.

The AMI ID recorded during the lab was:

`ami-0e4ea7066ac706098`

### Example CLI command

```bash
aws ec2 create-image \
  --instance-id i-0f7f39073e53eae2a \
  --name WebServerAMI \
  --description "Web server image for the Auto Scaling lab" \
  --region us-west-2
```

The AMI creation process may take time. The image should be available before it is selected in the launch template.

## 9. Create the Application Load Balancer

An internet-facing Application Load Balancer named `WebServerELB` was created.

Configuration included:

- **Scheme:** Internet-facing
- **IP address type:** IPv4
- **Listener:** HTTP on port 80
- **Network:** Lab VPC
- **Subnets:** Public Subnet 1 and Public Subnet 2
- **Security group:** `HTTPAccess`

The load balancer provided a single DNS endpoint through which users could access the application.

Using two public subnets across Availability Zones supported the load balancer's multi-zone configuration.

## 10. Create the Target Group

A target group named `webserver-app` was created to receive traffic from the load balancer.

Configuration:

| Setting | Value |
|---|---|
| Target type | Instances |
| Protocol | HTTP |
| Port | 80 |
| Health check path | `/index.php` |

The target group's health checks were used to determine whether registered EC2 instances could receive traffic.

The load balancer listener was configured to forward incoming HTTP requests to this target group.

## 11. Create the Launch Template

A launch template named `web-app-launch-template` was created.

Description:

`A web server for the load test app`

The template defined the settings used when Auto Scaling launched additional EC2 instances.

Configuration:

| Setting | Value |
|---|---|
| AMI | `WebServerAMI` |
| Instance type | `t3.micro` |
| Key pair | Not included in the launch template |
| Security group | `HTTPAccess` |

The launch template allowed Auto Scaling to launch instances consistently using the selected image and configuration.

## 12. Create the Auto Scaling Group

An Auto Scaling group named `Web App Auto Scaling Group` was created using the launch template.

### Network configuration

The Auto Scaling group used the lab VPC and the following private subnets:

- Private Subnet 1: `10.0.2.0/24`
- Private Subnet 2: `10.0.4.0/24`

This placed the web servers in private subnets while the internet-facing load balancer handled incoming public traffic.

### Capacity configuration

| Setting | Value |
|---|---|
| Desired capacity | 2 |
| Minimum capacity | 2 |
| Maximum capacity | 4 |
| Health check type | Elastic Load Balancing |
| Target group | `webserver-app` |
| Instance tag | `Name=WebApp` |

The group initially launched two instances. Its minimum capacity was two and it could scale out to a maximum of four instances.

The Auto Scaling group was associated with the `webserver-app` target group so that instances could be registered with the load balancer.

## 13. Configure Target Tracking Scaling

A target tracking scaling policy was configured for average CPU utilization.

**Target CPU utilization:** 50%

The policy allowed Auto Scaling to adjust the group's capacity in response to changes in the average CPU utilization of the group.

Target tracking does not mean that every brief CPU increase immediately launches an instance. CloudWatch metrics, evaluation periods, scaling behavior, and cooldown or warm-up behavior affect when scaling actions occur.

## 14. Verify the Initial Deployment

After the Auto Scaling group was created, its EC2 instances were checked.

The initial instances reached a healthy state, and the target group reported healthy targets.

The application was then tested through the Application Load Balancer's DNS name.

This confirmed that the load balancer could forward requests to the web servers.

## 15. Perform the Load Test

The application included a **Start Stress** control for initiating a load test.

The load test was started, and the Auto Scaling group's activity history was monitored.

During testing:

1. The load test generated additional application load.
2. Auto Scaling evaluated the group's metrics against its scaling policy.
3. AWS recorded a scale-out action in the activity history.
4. An additional EC2 instance launched.
5. The new instance became healthy.

This demonstrated the scale-out behavior of the configured Auto Scaling group.

The actual timing and amount of scaling depended on the metrics and policy evaluation.

## 16. AWS CLI Verification Commands

The following commands can be used to inspect resources if the training environment is still available. They are verification examples and should not be treated as a transcript of commands unless they were actually executed.

### Inspect the Auto Scaling group

```bash
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names "Web App Auto Scaling Group" \
  --region us-west-2
```

### Inspect scaling activity history

```bash
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name "Web App Auto Scaling Group" \
  --region us-west-2
```

### Inspect target group configuration

```bash
aws elbv2 describe-target-groups \
  --names webserver-app \
  --region us-west-2
```

### Check target health

Use the target group ARN returned by the preceding command:

```bash
aws elbv2 describe-target-health \
  --target-group-arn YOUR_TARGET_GROUP_ARN \
  --region us-west-2
```

### Inspect the AMI

```bash
aws ec2 describe-images \
  --image-ids ami-0e4ea7066ac706098 \
  --region us-west-2
```

## 17. Results

The lab was completed successfully.

| Test | Result |
|---|---|
| Initial EC2 web server launched | Successful |
| Web application loaded at `/index.php` | Successful |
| Custom AMI created | Successful |
| Application Load Balancer configured | Successful |
| Target group configured | Successful |
| Launch template created | Successful |
| Auto Scaling group configured | Successful |
| Initial instances became healthy | Successful |
| Application tested through ALB | Successful |
| Scale-out action observed | Successful |
| Additional instance became healthy | Successful |

These results describe the observations made during the lab. They do not guarantee that the resources remain active in AWS.

## 18. Evidence and Screenshots

Screenshots should be stored in the `screenshots/` directory.

Recommended evidence:

1. Auto Scaling group configuration.
2. Target tracking scaling policy.
3. EC2 instances after scale-out.
4. Healthy target group.
5. Application loaded through the load balancer.
6. Auto Scaling activity history.

Example Markdown:

```markdown
![Auto Scaling group](screenshots/autoscaling-group.png)

![Target tracking policy](screenshots/target-tracking-policy.png)

![EC2 instances after scale-out](screenshots/ec2-scale-out.png)

![Healthy targets](screenshots/healthy-targets.png)

![Load balancer application test](screenshots/load-balancer-test.png)

![Scaling activity history](screenshots/scaling-activity.png)
```

Ensure that the filenames match the screenshots actually uploaded to GitHub.

## 19. Key Learnings

This lab provided practical experience with:

- Launching and inspecting EC2 instances using the AWS CLI.
- Creating a reusable AMI.
- Configuring an internet-facing Application Load Balancer.
- Using target groups and health checks.
- Defining reusable EC2 launch settings.
- Deploying an Auto Scaling group across private subnets.
- Configuring target tracking scaling.
- Verifying application availability through a load balancer.
- Interpreting Auto Scaling activity during a load test.
- Documenting cloud infrastructure and testing evidence in GitHub.

## 20. Security and Cost Considerations

- Use security groups that allow only the required traffic.
- Prefer private subnets for application instances where the architecture permits.
- Restrict administrative access and avoid exposing SSH unnecessarily.
- Do not publish credentials or sensitive account information.
- Stop load tests when they are no longer required.
- Follow the lab's cleanup instructions to remove resources that are no longer needed.
- Check for related resources such as load balancers, target groups, AMIs, and snapshots before completing cleanup.

## 21. Conclusion

This lab demonstrated how an EC2-based web application can be placed behind an Application Load Balancer and managed by an Auto Scaling group.

The application was successfully tested through the load balancer, and a scale-out event was observed during the load test. The lab reinforced key AWS concepts related to compute, networking, health checks, scaling, high availability, and infrastructure verification.

The documentation forms part of my ongoing cloud engineering portfolio and records the configuration, testing process, and lessons learned.

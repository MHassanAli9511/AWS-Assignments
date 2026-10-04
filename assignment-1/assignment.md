# Assignment 1 - VPC & Networking

## Overview

In this assignment, I created a custom AWS VPC environment containing both a public and private subnet.

I configured the networking components required for public and private connectivity, including an Internet Gateway, NAT Gateway, route tables, security groups and EC2 instances.

I then tested connectivity between the instances and configured CloudWatch monitoring and alarms.

<!-- IMAGE: Final architecture diagram if you have one -->

---

## 1. Creating the VPC

I first created a custom VPC using the following CIDR block:

- VPC: `10.0.0.0/16`

<!-- IMAGE: VPC configuration -->

I then created two subnets inside the VPC:

- Public subnet: `10.0.0.0/24`
- Private subnet: `10.0.1.0/24`

<!-- IMAGE: Public and private subnet configuration -->

---

## 2. Internet Gateway

I created an Internet Gateway and attached it to my VPC.

The Internet Gateway allows resources in the VPC to communicate with the internet when the correct routing and security rules are also configured.

<!-- IMAGE: Internet Gateway -->

<!-- IMAGE: Internet Gateway attached to VPC -->

---

## 3. NAT Gateway

I created an Elastic IP address and then created a NAT Gateway inside the public subnet.

I associated the Elastic IP address with the NAT Gateway.

The NAT Gateway was required so that the EC2 instance in the private subnet could make outbound connections to the internet without being directly exposed to incoming internet traffic.

<!-- IMAGE: Elastic IP -->

<!-- IMAGE: NAT Gateway -->

---

## 4. Public Route Table

I configured a route table for the public subnet.

The route table contained:

- The local route for traffic inside the VPC
- `0.0.0.0/0` pointing to the Internet Gateway

I then associated this route table with the public subnet.

<!-- IMAGE: Public route table -->

<!-- IMAGE: Public subnet association -->

---

## 5. Private Route Table

I created a separate route table for the private subnet.

The private route table contained:

- The local VPC route
- `0.0.0.0/0` pointing to the NAT Gateway

I then associated this route table with the private subnet.

<!-- IMAGE: Private route table -->

<!-- IMAGE: Private subnet association -->

---

## 6. EC2 Instances

I created two EC2 instances:

- A public EC2 instance in the public subnet
- A private EC2 instance in the private subnet

I also created an SSH key for accessing the instances.

<!-- IMAGE: Public EC2 networking configuration -->

<!-- IMAGE: Private EC2 networking configuration -->

---

## 7. Security Groups

For the public EC2 instance, I configured security group rules for SSH and HTTP access.

<!-- IMAGE: Public EC2 security group -->

For the private EC2 instance, I configured SSH access so that traffic was allowed from the security group associated with the public EC2 instance.

Using the security group as the source means I do not have to update the private security group rule if the public EC2 instance's private IP address changes.

<!-- IMAGE: Private EC2 security group -->

---

## 8. Troubleshooting EC2 Instance Connect

### Problem

When I first tried to connect to the public EC2 instance using browser-based EC2 Instance Connect, the connection failed.

### Cause

My security group only allowed SSH traffic on port 22 from my own IP address.

EC2 Instance Connect uses AWS service IP ranges, so the connection was not being allowed by the existing security group rule.

### Solution

I navigated to the EC2 security group inbound rules and added another SSH rule using the AWS-managed prefix list:

`com.amazonaws.eu-north-1.ec2-instance-connect`

I kept my existing SSH rule for my own IP address.

After adding the prefix list, EC2 Instance Connect worked successfully.

### What I Learned

Having a public IP address and a route through an Internet Gateway is not enough on its own.

The security group must also allow traffic from the actual source of the connection.

<!-- IMAGE: EC2 Instance Connect security group rule -->

<!-- IMAGE: Successful EC2 Instance Connect session -->

---

## 9. Using the Public Instance as a Bastion Host

I connected to the public EC2 instance using EC2 Instance Connect.

I then used the public instance as a bastion host to SSH into the private EC2 instance.

This allowed me to access the private instance without giving the private instance direct public access.

<!-- IMAGE: Public EC2 terminal -->

<!-- IMAGE: SSH connection from public EC2 to private EC2 -->

---

## 10. Testing Internet Connectivity from the Private Instance

After connecting to the private EC2 instance, I tested its internet connectivity using:

```bash
curl -I https://www.google.com
```
The request returned:

```text
HTTP/2 200
```
This showed that the request successfully reached the internet and received a response.
It also provided evidence that the NAT Gateway configuration was working.
<!-- IMAGE: curl command from private EC2 -->

<!-- IMAGE: HTTP/2 200 response -->

---

## 11. CloudWatch Detailed Monitoring

I then configured monitoring for both EC2 instances.

I first viewed the standard monitoring information available for the public EC2 instance.

I then enabled **Detailed Monitoring**.

I repeated the same process for the private EC2 instance.

<!-- IMAGE: EC2 Monitoring tab -->

<!-- IMAGE: Enabling detailed monitoring -->

---

## 12. CloudWatch CPU Alarms

I learned that a CloudWatch alarm needs three main things:

- **Metric** – what is being monitored
- **Threshold** – the value that causes the alarm condition
- **Evaluation period** – how long the condition must persist

For this assignment, I monitored EC2 **CPU utilisation**.

<!-- IMAGE: Selecting CPUUtilization metric -->

I configured the threshold so that CPU utilisation above `80%` would be considered an alarm condition.

<!-- IMAGE: 80% CPU threshold -->

I also configured the alarm to use `2 out of 2` datapoints.

This means the alarm would only trigger if two consecutive datapoints crossed the threshold.

For example, if one datapoint recorded `85%` CPU usage but the next recorded `40%`, the alarm would not trigger.

<!-- IMAGE: Datapoints to alarm configuration -->

---

## 13. Alarm Notifications

I configured an action so that an email notification would be sent if the alarm entered the **ALARM** state.

I gave the alarm a name and description and completed the configuration.

I then created the same type of monitoring for the private EC2 instance.

<!-- IMAGE: SNS/email notification configuration -->

<!-- IMAGE: Public EC2 CloudWatch alarm -->

<!-- IMAGE: Public and private EC2 alarms -->

---

## CloudWatch Detailed Monitoring vs CloudWatch Agent

I also learned that EC2 Detailed Monitoring and the CloudWatch Agent are different.

**Detailed Monitoring** provides more frequent EC2 monitoring data.

The **CloudWatch Agent** can collect additional performance metrics and logs from inside the operating system, such as:

- Memory usage
- Disk space usage
- Operating system logs

> **Note:** For this assignment, I enabled EC2 Detailed Monitoring. I did not configure the CloudWatch Agent.

<!-- IMAGE: CloudWatch Agent comparison -->

---

## What I Learned

Through this assignment, I learned how the main AWS networking components work together.

I developed a better understanding of:

- VPCs and CIDR ranges
- Public and private subnets
- Internet Gateways
- NAT Gateways
- Elastic IP addresses
- Public and private route tables
- Security groups
- EC2 connectivity
- Bastion hosts
- EC2 Instance Connect
- CloudWatch Detailed Monitoring
- CloudWatch alarms and datapoints

I also learned that troubleshooting AWS networking requires checking several separate components, including routing, security groups, and the source of the connection.

---

## Challenges and How I Solved Them

The main problem I encountered was being unable to connect to my public EC2 instance using browser-based **EC2 Instance Connect**.

The instance had a public IP address and the correct Internet Gateway route, so I investigated the security group configuration.

I found that SSH was only allowed from my own IP address, while EC2 Instance Connect uses AWS-managed service IP ranges.

I solved the issue by adding the AWS-managed EC2 Instance Connect prefix list to the security group's SSH rules.

After making this change, I was able to connect successfully and continue using the public EC2 instance as a **bastion host**.

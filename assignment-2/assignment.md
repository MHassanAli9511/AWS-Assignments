# Assignment 2 - Application Load Balancer


## Overview

In this assignment, I extended the VPC environment created in Assignment 1 and built a highly available web application architecture across multiple Availability Zones.

I created additional public and private subnets, launched two EC2 web servers, configured an Application Load Balancer, created a target group, and then implemented an Auto Scaling Group.

I also configured dynamic scaling based on CPU utilisation and tested the Auto Scaling Group by terminating an instance and confirming that AWS automatically launched a replacement.

<!-- IMAGE: Final architecture diagram -->

---

## 1. Extending the Existing VPC

I reused the VPC environment from Assignment 1 instead of creating a new network from scratch.

To improve availability, I created an additional public subnet in a different Availability Zone.

<!-- IMAGE: Public subnet B  second Availability Zone -->

Because the original public subnet already had a configured public route table, I did not need to create another route table.

Instead, I associated the new public subnet with the existing public route table.

<!-- IMAGE: Public subnet route table association -->

I followed the same approach for the private subnet by creating another private subnet in a different Availability Zone.


This gave me public and private subnets across multiple Availability Zones.

---

## 2. Launching the Web Servers

I launched two EC2 instances to act as backend web servers.

Each instance was placed in a private subnet.

I configured the instances with a simple Apache web server using EC2 user data.

For example:

```bash
#!/bin/bash
yum update -y
yum install -y httpd

systemctl start httpd
systemctl enable httpd

echo "Hello from EC2 A" > /var/www/html/index.html
```

I configured the second EC2 instance in the other private subnet with a similar web page so I could tell which backend server handled each request.


## 3. Application Load Balancer Security Group

I created a dedicated security group for the Application Load Balancer.

The ALB security group allowed HTTP traffic on port `80` from IPv4 addresses.

<!-- IMAGE: ALB security group -->

I then updated the EC2 security group so that HTTP traffic was allowed from the ALB security group.

This means the backend EC2 instances do not need to accept HTTP traffic directly from the public internet.

Instead, traffic reaches them through the Application Load Balancer.

<!-- IMAGE: EC2 security group allowing ALB security group -->

---

## 4. Creating the Application Load Balancer

I created an Application Load Balancer and configured it to use the public subnets in different Availability Zones.

I also associated the ALB security group with the load balancer.

Using subnets in multiple Availability Zones improves availability because the load balancer can distribute traffic across more than one zone.

---

## 5. Creating the Target Group

I created a target group for the EC2 web servers.

The target group configuration used:

- Target type: Instances
- Protocol: HTTP
- Port: 80
- VPC: My custom VPC
- Health check protocol: HTTP
- Health check path: `/`


I registered both EC2 A and EC2 B as targets.

<!-- IMAGE: EC2 A and B registered as targets -->

After creating the target group, AWS showed both targets as healthy.

<!-- IMAGE: Healthy target group -->

---

## 6. Attaching the Target Group to the Load Balancer

I configured the Application Load Balancer listener to forward HTTP traffic to the target group.

<!-- IMAGE: ALB listener forwarding to target group -->

This created the following traffic flow:

    Internet
       |
       v
    Application Load Balancer
       |
       v
    Target Group
      /   \
     v     v
    EC2 A  EC2 B

---

## 7. Testing the Load Balancer

I tested the Application Load Balancer using its DNS name.

The web page loaded successfully.

When I refreshed the page, the response changed between EC2 A and EC2 B.

This demonstrated that the Application Load Balancer was distributing requests across both backend instances.

<!-- IMAGE/VIDEO: ALB DNS test showing EC2 A and EC2 B -->

---

## 8. Creating a Launch Template

After completing the load-balancing configuration, I moved on to Auto Scaling.

I first created a launch template.

The launch template defines the configuration AWS should use when the Auto Scaling Group needs to create new EC2 instances.

<!-- IMAGE: Launch template configuration -->

---

## 9. Creating the Auto Scaling Group

I created an Auto Scaling Group using the launch template.

<!-- IMAGE: ASG using launch template -->

I selected the existing VPC and configured the Auto Scaling Group to use the two private subnets in different Availability Zones.

<!-- IMAGE: ASG subnet and Availability Zone configuration -->

This allows the Auto Scaling Group to distribute instances across multiple Availability Zones.

---

## 10. Configuring Auto Scaling Capacity

I configured the Auto Scaling Group with:

- Minimum capacity: 2
- Desired capacity: 2
- Maximum capacity: 4

<!-- IMAGE: Minimum, desired and maximum capacity -->

The minimum capacity ensures that AWS maintains at least two running instances.

The desired capacity represents the number of instances that the Auto Scaling Group attempts to maintain under normal conditions.

The maximum capacity prevents the group from scaling beyond four instances.

---

## 11. Connecting the Auto Scaling Group to the Load Balancer

I attached the Auto Scaling Group to the existing Application Load Balancer target group.

<!-- IMAGE: ASG attached to target group -->

This means that new EC2 instances launched by the Auto Scaling Group can automatically become part of the load-balanced application.

---

## 12. Dynamic Scaling Policy

I created a dynamic target-tracking scaling policy.

The policy monitored average CPU utilisation.

I configured the target value as:

`50%`

<!-- IMAGE: Target tracking scaling policy -->

A scaling policy tells the Auto Scaling Group when it should increase or decrease the number of EC2 instances.

For example:

- If average CPU utilisation becomes too high, AWS can add instances.
- If CPU utilisation decreases, AWS can reduce the number of instances.

The scaling policy allows the Auto Scaling Group to move between the configured minimum and maximum capacity.

Without a dynamic scaling policy, the Auto Scaling Group can still replace unhealthy instances, but it would not automatically scale based on CPU demand.

---

## 13. Verifying the Auto Scaling Group

After configuring the Auto Scaling Group, I checked the EC2 instances and target group.

The environment contained multiple running EC2 instances, and the target group reported them as healthy.

<!-- IMAGE: EC2 instances created by ASG -->

<!-- IMAGE: Healthy targets -->

---

## 14. Testing Automatic Instance Replacement

To test the Auto Scaling Group, I manually terminated one of the managed EC2 instances.

Because the Auto Scaling Group was configured to maintain its required capacity, AWS automatically launched another EC2 instance to replace the terminated instance.

<!-- IMAGE: Replacement EC2 instance launching -->

This demonstrated that Auto Scaling provides self-healing behaviour as well as scaling.

---

## What I Learned

Through this assignment, I developed a better understanding of:

- Using multiple Availability Zones for higher availability
- Reusing existing VPC route tables
- Hosting web servers on EC2
- Application Load Balancers
- Security groups between an ALB and backend instances
- Target groups
- Health checks
- Load balancing between EC2 instances
- Launch templates
- Auto Scaling Groups
- Minimum, desired and maximum capacity
- Target-tracking scaling policies
- CPU-based dynamic scaling
- Automatic replacement of unhealthy or terminated instances

I also learned how Application Load Balancers and Auto Scaling Groups work together.

The load balancer distributes incoming traffic between healthy instances, while the Auto Scaling Group maintains the required number of instances and can add or remove capacity as demand changes.

---

## Testing and Validation

I tested the architecture in several ways.

First, I accessed the Application Load Balancer using its DNS name and confirmed that traffic was distributed between EC2 A and EC2 B.

I then confirmed that the registered targets were healthy.

Finally, I manually terminated an Auto Scaling Group instance and confirmed that AWS automatically launched a replacement instance to maintain the configured capacity.

These tests confirmed that both the load-balancing and Auto Scaling configurations were functioning successfully.

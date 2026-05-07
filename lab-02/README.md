# Lab-002: A Single EC2 Instance in a Private Subnet with a Bastion Host

| Item | Details |
|------|---------|
| **Lab ID** | Lab-002 |
| **Title** | A Single EC2 Instance in a Private Subnet with a Bastion Host |
| **Difficulty** | Beginner |
| **Author** | Manoj M |
| **Creation Date** | May 7, 2026 |
| **Primary Goal** | Access an EC2 instance in a private subnet through a bastion host in a public subnet |

---

## Overview

This lab demonstrates how to create a custom VPC with both public and private subnets, configure internet access for the public subnet, and use a bastion host to connect securely to an EC2 instance running inside the private subnet.

The public EC2 instance acts as the **bastion host** or **jump host**. It has a public IPv4 address and can be reached from your local system using SSH. The private EC2 instance does not have direct internet access and is reached only by first connecting to the bastion host.

---

## Services Used

- Amazon VPC
- Subnets
- Internet Gateway
- Route Tables
- Amazon EC2
- Security Groups
- SSH Agent Forwarding

---

## Before You Begin

Use the same AWS Region for the full lab. The screenshots use **US East (N. Virginia) / us-east-1**.

You also need:

- An AWS account with permission to create VPC and EC2 resources
- A Linux terminal, Git Bash, or WSL for SSH commands
- One EC2 key pair named `snakepass.pem`

> Important: Use the same key pair, `snakepass.pem`, for both EC2-A and EC2-B. If the private instance uses a different key pair, SSH from the bastion host will fail.

---

## Architecture Diagram

![Arcitecture](<images/image.png>)

---

## Learning Outcomes

After completing this lab, you will be able to:

- Create a custom VPC with an IPv4 CIDR block
- Create and attach an Internet Gateway
- Create public and private subnets
- Enable auto-assign public IPv4 for a public subnet
- Create a custom route table for internet access
- Associate a route table with a public subnet
- Launch EC2 instances in different subnets
- Use a bastion host to reach a private EC2 instance
- Connect using SSH agent forwarding

---

## Key Concepts

- **VPC**: A logically isolated network in AWS where resources are launched.
- **Public Subnet**: A subnet with a route to an Internet Gateway.
- **Private Subnet**: A subnet without a direct route to the internet.
- **Internet Gateway**: Allows resources in a public subnet to communicate with the internet.
- **Route Table**: Controls where network traffic is directed.
- **Bastion Host**: A public EC2 instance used as a secure entry point to private instances.
- **SSH Agent Forwarding**: Lets you SSH from the bastion host to the private instance without copying the private key to the bastion host.

---

## Network Plan

| Resource | Name | CIDR / Setting |
|----------|------|----------------|
| VPC | `Lab-02` | `192.168.0.0/16` |
| Internet Gateway | `lab-02` | Attached to `Lab-02` VPC |
| Public Subnet | `Public_Subnet-01` | `192.168.100.0/24` |
| Private Subnet | `private` | `192.168.200.0/24` |
| Public Route | `0.0.0.0/0` | Internet Gateway |
| Bastion Host | EC2 A | Public subnet |
| Private Instance | EC2 B | Private subnet |
| Key Pair | `snakepass.pem` | Used by both EC2 instances |
| Bastion Security Group | `lab-002-bastion-sg` | SSH from My IP |
| Private Security Group | `lab-002-private-sg` | SSH from bastion security group |

---

## How Bastion Host Access Works in This Lab

The EC2 instance in the private subnet does not receive a public IPv4 address, so you cannot SSH into it directly from your local machine.

Instead, the connection flow is:

1. Your local system connects to the bastion host using the bastion host public IPv4 address.
2. The bastion host connects to the private EC2 instance using the private instance private IPv4 address.
3. SSH agent forwarding allows the private key authentication to work without placing the `.pem` key file on the bastion host.

This is a safer pattern because the private instance remains isolated from direct internet access.

---

## Step-by-Step Procedure

### Step 1: Open the VPC Dashboard

- Sign in to the AWS Management Console.
- Search for **VPC** and open the VPC service.
- From **Your VPCs**, click **Create VPC**.

This starts the process of creating a custom network for the lab.

![VPC Dashboard](<images/Vpc creation.png>)

---

### Step 2: Create the VPC

- Select **VPC only**.
- Enter the name tag as `Lab-02`.
- Set the IPv4 CIDR block to `192.168.0.0/16`.
- Keep the tenancy as **Default**.
- Click **Create VPC**.

The VPC CIDR block provides the private IP address range from which the subnets will be created.

![Create VPC](<images/Vpc created.png>)

---

### Step 3: Confirm the VPC Was Created

- Verify that the new VPC is listed.
- Confirm the VPC state is **Available**.
- Confirm the IPv4 CIDR block is `192.168.0.0/16`.

The VPC creation is successful when the VPC is available and can be selected later while creating subnets.

---

### Step 4: Create an Internet Gateway

- In the VPC console, open **Internet gateways**.
- Click **Create internet gateway**.
- Enter the name tag as `lab-02`.
- Click **Create internet gateway**.

The Internet Gateway will allow the public subnet to communicate with the internet after it is attached and routed.

![Internet Gateway List](<images/internet gateway create.png>)

![Create Internet Gateway](<images/internet gateway created.png>)

---

### Step 5: Attach the Internet Gateway to the VPC

- After the Internet Gateway is created, open **Actions**.
- Click **Attach to VPC**.

![Attach to VPC Action](<images/Attach to Vpc.png>)

- Select the `Lab-02` VPC.
- Click **Attach internet gateway**.

![Select VPC for Internet Gateway](<images/Attached to vpc.png>)

---

### Step 6: Verify Internet Gateway Attachment

- Open the Internet Gateways list.
- Confirm the `lab-02` Internet Gateway state is **Attached**.
- Confirm it is attached to the `Lab-02` VPC.

![Internet Gateway Attached](<images/vpc attach check.png>)

---

### Step 7: Create the Public Subnet

- Open **Subnets** in the VPC console.
- Click **Create subnet**.
- Select the `Lab-02` VPC.

![Select VPC for Subnet](<images/vpc selection.png>)

- Enter subnet name as `Public_Subnet-01`.
- Select an Availability Zone.
- Set the IPv4 subnet CIDR block to `192.168.100.0/24`.
- Click **Create subnet**.

![Create Public Subnet](<images/subnet creation.png>)

---

### Step 8: Confirm the Public Subnet

- Verify that `Public_Subnet-01` was created.
- Confirm its state is **Available**.
- Confirm the IPv4 CIDR is `192.168.100.0/24`.

![Public Subnet Created](<images/public subnet check.png>)

---

### Step 9: Enable Auto-Assign Public IPv4 on the Public Subnet

- Select `Public_Subnet-01`.
- Open **Actions**.
- Click **Edit subnet settings**.

![Edit Subnet Settings](<images/edit subnet.png>)

- Enable **Auto-assign public IPv4 address**.
- Click **Save**.

This setting ensures that new EC2 instances launched in the public subnet can automatically receive a public IPv4 address.

![Enable Auto Assign Public IPv4](<images/auto assign ip to subnet.png>)

---

### Step 10: Create a Route Table for the Public Subnet

- Open **Route tables**.
- Click **Create route table**.

![Route Tables Page](<images/create rounte table.png>)

- Enter the name as `public`.
- Select the `Lab-02` VPC.
- Click **Create route table**.

![Create Public Route Table](<images/route_table created.png>)

---

### Step 11: Add a Default Route to the Internet Gateway

- Open the newly created `public` route table.
- Go to the **Routes** tab.
- Click **Edit routes**.

![Edit Routes](<images/edit_routes.png>)

- Click **Add route**.
- Set destination to `0.0.0.0/0`.
- Set target to **Internet Gateway**.
- Select the `lab-02` Internet Gateway.
- Click **Save changes**.

This route sends internet-bound traffic from the public subnet to the Internet Gateway.

![Add Internet Gateway Route](<images/adding routes.png>)

---

### Step 12: Associate the Route Table with the Public Subnet

- Select the `public` route table.
- Open the **Subnet associations** tab.
- Click **Edit subnet associations**.

![Subnet Associations Tab](<images/subnet  association.png>)

- Select `Public_Subnet-01`.
- Click **Save associations**.

![Save Subnet Association](<images/subnet association created.png>)

The public subnet is now connected to the route table that has the route to the Internet Gateway.

---

### Step 13: Create the Private Subnet

- Open **Subnets**.
- Click **Create subnet**.
- Select the `Lab-02` VPC.
- Enter subnet name as `private`.
- Select an Availability Zone.
- Set the IPv4 subnet CIDR block to `192.168.200.0/24`.
- Click **Create subnet**.

The private subnet does not need auto-assign public IPv4 and should not be associated with the public route table.

![Create Private Subnet](<images/private subnet.png>)

---

### Step 14: Launch the Bastion Host in the Public Subnet

Launch the first EC2 instance in the public subnet. This instance is EC2-A, the bastion host.

Use these settings:

| Setting | Value |
|---------|-------|
| Instance name | `Lab-002-Bastion` |
| Operating system / AMI | Amazon Linux OS / Amazon Linux 2023 |
| Instance type | `t3.micro` or free-tier eligible type |
| VPC | `Lab-02` |
| Subnet | `Public_Subnet-01` |
| Auto-assign public IP | Enabled or Use subnet setting |
| Key pair | `snakepass.pem` |
| Security group name | `lab-002-bastion-sg` |

Create the bastion host security group with this inbound rule:

| Type | Port | Source |
|------|------|--------|
| SSH | `22` | My IP |

Keep the default outbound rule as **All traffic**.

After launching, confirm these details on EC2-A:

- **Instance state:** `Running`
- **Public IPv4 address:** Available
- **Private IPv4 address:** From the `192.168.100.0/24` range
- **Key pair name:** `snakepass.pem`
- **Security group:** `lab-002-bastion-sg`

Copy the **public IPv4 address** of EC2-A. You will use it to connect from your local terminal.

---

### Step 15: Launch the Private EC2 Instance

Launch the second EC2 instance in the private subnet. This instance is EC2-B, the private instance.

Use these settings:

| Setting | Value |
|---------|-------|
| Instance name | `Lab-002-Private` |
| Operating system / AMI | Amazon Linux OS / Amazon Linux 2023 |
| Instance type | `t3.micro` or free-tier eligible type |
| VPC | `Lab-02` |
| Subnet | `private` |
| Auto-assign public IP | Disabled |
| Key pair | `snakepass.pem` |
| Security group name | `lab-002-private-sg` |

Create the private instance security group with this inbound rule:

| Type | Port | Source |
|------|------|--------|
| SSH | `22` | `lab-002-bastion-sg` |

When adding the SSH rule, choose **Custom** as the source and select or paste the bastion host security group ID. This allows SSH only from EC2-A and is more secure than allowing the full public subnet CIDR.

Keep the default outbound rule as **All traffic**.

After launching, confirm these details on EC2-B:

- **Instance state:** `Running`
- **Public IPv4 address:** Not assigned
- **Private IPv4 address:** From the `192.168.200.0/24` range
- **Key pair name:** `snakepass.pem`
- **Security group:** `lab-002-private-sg`

Copy the **private IPv4 address** of EC2-B. You will use it from inside EC2-A.

---

### Step 16: Start the SSH Agent and Add the PEM Key

Before connecting through the bastion host, set secure permissions on the PEM key, start the SSH agent on your Linux terminal, and add the key.

Set secure permissions:

```bash
chmod 400 snakepass.pem
```

Run:

```bash
eval "$(ssh-agent -s)"
```

Then add the key:

```bash
ssh-add snakepass.pem
```

Expected result:

```text
Identity added: snakepass.pem (snakepass.pem)
```

> Important: Use the same key pair, `snakepass.pem`, for both EC2 instances. Do not copy the private key file to the bastion host. SSH agent forwarding keeps the key on your local machine.

---

### Step 17: Connect to the Bastion Host

From your local Linux terminal, connect to the bastion host using SSH agent forwarding.

Replace `<public-IP-of-bastion-host>` with the public IPv4 address of your EC2-A instance.

```bash
ssh -A -i snakepass.pem ec2-user@<public-IP-of-bastion-host>
```

Example:

```bash
ssh -A -i snakepass.pem ec2-user@3.238.66.207
```

The `-A` option is mandatory because it enables SSH agent forwarding.

---

### Step 18: Verify SSH Agent Forwarding on the Bastion Host

After logging in to EC2-A, run:

```bash
ssh-add -L
```

If you see a long SSH public key, agent forwarding is working.

If you see this message, agent forwarding failed:

```text
The agent has no identities
```

If this happens, exit from EC2-A and reconnect using the `-A` option:

```bash
ssh -A -i snakepass.pem ec2-user@<public-IP-of-bastion-host>
```

---

### Step 19: Connect from the Bastion Host to the Private Instance

After logging in to the bastion host, connect to the private EC2 instance using its private IPv4 address.

Replace `<private-IP-of-private-instance>` with the private IPv4 address of your EC2-B instance.

```bash
ssh ec2-user@<private-IP-of-private-instance>
```

Example:

```bash
ssh ec2-user@192.168.200.232
```

If the security groups and SSH agent forwarding are configured correctly, you will log in to the private EC2 instance without storing the `.pem` file on the bastion host.

![SSH Agent Forwarding and Private Instance Login](<images/ssh-agent-forwarding.png>)

> Very important: EC2-B must use the same key pair as EC2-A: `snakepass.pem`. In the AWS console, check **EC2 > instance-B > Key pair name**. If EC2-B uses a different key pair, SSH authentication will fail.

---

## Validation Checklist

Use this checklist to confirm the lab was completed successfully:

- Custom VPC `Lab-02` created with CIDR `192.168.0.0/16`
- Internet Gateway `lab-02` created
- Internet Gateway attached to the `Lab-02` VPC
- Public subnet created with CIDR `192.168.100.0/24`
- Auto-assign public IPv4 enabled for the public subnet
- Public route table created
- Route `0.0.0.0/0` points to the Internet Gateway
- Public route table associated with the public subnet
- Private subnet created with CIDR `192.168.200.0/24`
- Bastion host launched in the public subnet
- Private EC2 instance launched in the private subnet
- Both EC2 instances use the same key pair, `snakepass.pem`
- Bastion security group allows SSH from My IP
- Private instance security group allows SSH from the bastion security group
- Bastion host has a public IPv4 address
- Private instance has only a private IPv4 address
- SSH agent forwarding is verified using `ssh-add -L`
- SSH to bastion host works
- SSH from bastion host to private instance works

---

## Commands Used in This Lab

```bash
chmod 400 snakepass.pem
eval "$(ssh-agent -s)"
ssh-add snakepass.pem
ssh -A -i snakepass.pem ec2-user@<public-IP-of-bastion-host>
ssh-add -L
ssh ec2-user@<private-IP-of-private-instance>
```

---

## Expected Result

At the end of this lab:

- A custom VPC is created with public and private subnets.
- The public subnet can access the internet through an Internet Gateway.
- The bastion host is reachable from your local system using SSH.
- The private EC2 instance is not directly reachable from the internet.
- You can SSH into the private EC2 instance only through the bastion host.

---

## Common Mistakes

- Forgetting to attach the Internet Gateway to the VPC
- Creating the Internet Gateway but not adding a route to it
- Not associating the public route table with the public subnet
- Forgetting to enable auto-assign public IPv4 for the public subnet
- Launching the bastion host in the private subnet by mistake
- Launching the private instance with a public IPv4 address
- Copying the `.pem` key to the bastion host instead of using SSH agent forwarding
- Using different key pairs for EC2-A and EC2-B
- Not allowing SSH from the bastion host to the private instance
- Using the public IP instead of the private IP when connecting from the bastion host to the private instance

---

## Troubleshooting

| Problem | Likely Cause | Fix |
|---------|--------------|-----|
| Cannot SSH to EC2-A | EC2-A has no public IP or SSH is not allowed from your IP | Confirm EC2-A is in `Public_Subnet-01`, has a public IPv4 address, and `lab-002-bastion-sg` allows SSH from My IP |
| `WARNING: UNPROTECTED PRIVATE KEY FILE` | PEM file permissions are too open | Run `chmod 400 snakepass.pem` |
| `ssh-add -L` shows `The agent has no identities` | Agent forwarding was not enabled | Exit EC2-A and reconnect using `ssh -A -i snakepass.pem ec2-user@<public-IP-of-bastion-host>` |
| Cannot SSH from EC2-A to EC2-B | EC2-B security group does not allow SSH from EC2-A | In `lab-002-private-sg`, allow SSH from `lab-002-bastion-sg` |
| Permission denied while connecting to EC2-B | EC2-B uses a different key pair | Confirm EC2-B key pair name is `snakepass.pem` |

---

## Cleanup

To avoid unnecessary AWS charges after completing the lab, delete the resources in this order:

1. Terminate the private EC2 instance.
2. Terminate the bastion host EC2 instance.
3. Wait until both instances are fully terminated.
4. Delete the private subnet.
5. Delete the public subnet.
6. Delete the custom `public` route table.
7. Detach the Internet Gateway from the VPC.
8. Delete the Internet Gateway.
9. Delete the `Lab-02` VPC.

---

## Conclusion

In this lab, a custom VPC was created with a public subnet and a private subnet. An Internet Gateway and public route table were configured so the bastion host could be accessed from the internet. A second EC2 instance was launched in the private subnet and accessed securely through the bastion host using SSH agent forwarding.

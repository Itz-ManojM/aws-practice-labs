# Lab-003: Enable Internet Access for a Private EC2 Instance Using a NAT Gateway

| Item | Details |
|------|---------|
| **Lab ID** | Lab-003 |
| **Title** | Enable Internet Access for a Private EC2 Instance Using a NAT Gateway |
| **Difficulty** | Intermediate Beginner |
| **Author** | Manoj M |
| **Creation Date** | May 7, 2026 |
| **Primary Goal** | Allow an EC2 instance in a private subnet to access the internet through a NAT Gateway |

---

## Overview

This lab is a direct extension of **Lab-002**. You keep the same VPC, public subnet, private subnet, bastion host, and private EC2 instance, then add a **NAT Gateway** so the private EC2 instance can make **outbound internet requests** without becoming publicly accessible.

This means:

- The private instance still does **not** get a public IP address
- You still access the private instance through the **bastion host**
- The private instance can now reach the internet for updates, package downloads, and outbound requests such as `curl google.com`

---

## Architecture Diagram

The diagram below shows the Lab-003 network layout. It reuses the **Lab-002** design with a bastion host in the public subnet and adds a **NAT Gateway** for outbound traffic from the private subnet.

- **EC2 Instance A** is the bastion host in the public subnet.
- **EC2 Instance B** is the private instance in the private subnet.
- The public subnet route table sends `0.0.0.0/0` traffic to the Internet Gateway.
- The private subnet route table sends `0.0.0.0/0` traffic to the NAT Gateway.
- The private instance remains inaccessible directly from the internet.

![Architecture Diagram](<images/Architecture.png>)

---

## Starting Point

Complete **Lab-002: A Single EC2 Instance in a Private Subnet with a Bastion Host** before starting this lab.

Before continuing, confirm these resources are already available:

- VPC `Lab-02` with CIDR `192.168.0.0/16`
- Public subnet `Public_Subnet-01` with CIDR `192.168.100.0/24`
- Private subnet `private` with CIDR `192.168.200.0/24`
- Internet Gateway attached to the VPC
- Public route table with `0.0.0.0/0` pointing to the Internet Gateway
- EC2-A bastion host in the public subnet
- EC2-B private instance in the private subnet
- Key pair `snakepass.pem` used by both EC2 instances
- SSH access from your local machine to EC2-B through EC2-A

> If you do not already have the Lab-002 environment, complete Lab-002 first, then return to this lab.

---

## Services Used

- Amazon VPC
- NAT Gateway
- Elastic IP
- Route Tables
- Amazon EC2
- Internet Gateway
- SSH Agent Forwarding

---

## Learning Outcomes

After completing this lab, you will be able to:

- Create a NAT Gateway in a public subnet
- Understand why a NAT Gateway needs internet connectivity
- Create or update a route table for private subnet outbound traffic
- Add a default route from a private subnet to the NAT Gateway
- Associate the private route table with the private subnet
- Validate outbound internet access from a private EC2 instance

---

## Key Concepts

- **Private Subnet**: A subnet where instances do not have direct public internet access.
- **NAT Gateway**: A managed AWS service that allows instances in a private subnet to access the internet for outbound traffic.
- **Elastic IP**: A static public IPv4 address associated with the NAT Gateway.
- **Route Table**: Defines where subnet traffic is sent.
- **Outbound Internet Access**: Allows the private instance to reach the internet without accepting inbound internet connections.

---

## How NAT Gateway Works in This Lab

The NAT Gateway is created inside the **public subnet** from Lab-002.

It works like this:

1. The private EC2 instance sends internet-bound traffic to its route table.
2. The private route table forwards `0.0.0.0/0` traffic to the NAT Gateway.
3. The NAT Gateway sends that traffic out through the public subnet and Internet Gateway.
4. The response comes back to the NAT Gateway.
5. The NAT Gateway forwards the response back to the private EC2 instance.

This allows the private instance to:

- Download packages
- Access public websites
- Reach external services

It still does **not** allow inbound SSH or browser access directly from the internet.

---

## Network Flow

| Source | Destination | Path |
|--------|-------------|------|
| Your local machine | Bastion Host | Public IP over SSH |
| Bastion Host | Private EC2 Instance | Private IP over SSH |
| Private EC2 Instance | Internet | Private route table -> NAT Gateway -> Internet Gateway |

---

## Lab-003 Resource Plan

| Resource | Name / Value |
|----------|--------------|
| NAT Gateway | `my-natgw-03` |
| NAT Gateway subnet | Public subnet |
| NAT Gateway connectivity type | Public |
| Elastic IP | Automatically allocated |
| Private route table | `private` |
| Private default route | `0.0.0.0/0 -> NAT Gateway` |
| Test command | `curl google.com` from EC2-B |

---

## Important Notes

- Create the NAT Gateway in the **public subnet**, not the private subnet.
- The public subnet must already route `0.0.0.0/0` traffic to the Internet Gateway.
- A public NAT Gateway requires an Elastic IP.
- The private EC2 instance should still have **no public IPv4 address**.
- NAT Gateway is used for outbound traffic only. It does not make the private instance publicly reachable.
- NAT Gateway can create AWS charges while it is running. Delete it after the lab if you no longer need it.

---

## Step-by-Step Procedure

### Step 1: Confirm the Lab-002 Environment

Before creating the NAT Gateway, confirm that the Lab-002 environment is working.

Check the following:

- EC2-A bastion host is running in the public subnet.
- EC2-A has a public IPv4 address.
- EC2-B private instance is running in the private subnet.
- EC2-B has no public IPv4 address.
- You can SSH from your local machine to EC2-A.
- You can SSH from EC2-A to EC2-B using EC2-B's private IPv4 address.

> For this lab, you are not creating the full network from scratch again. You are extending the Lab-002 environment by adding a NAT Gateway and a private route table.

---

### Step 2: Open the NAT Gateway Dashboard

- Sign in to the AWS Management Console.
- Open the **VPC** service.
- In the left navigation panel, click **NAT gateways**.
- Click **Create NAT gateway**.

This opens the NAT Gateway creation page.

![NAT Gateway Dashboard](<images/Nat dash.png>)

---

### Step 3: Create the NAT Gateway

Create the NAT Gateway inside the **public subnet** from Lab-002.

Use these settings:

| Setting | Value |
|---------|-------|
| Name | `my-natgw-03` |
| Availability mode | `Regional` |
| VPC | Same VPC created in Lab-002 |
| Connectivity type | `Public` |
| Elastic IP allocation | `Automatic` |
| Subnet | Public subnet from Lab-002 |

Important points:

- The NAT Gateway must be created in the **public subnet**.
- The public subnet must already have a route to the **Internet Gateway**.
- A public NAT Gateway requires an **Elastic IP**.

Then click **Create NAT gateway**.

![Create NAT Gateway](<images/nat_creation.png>)

---

### Step 4: Wait Until the NAT Gateway Becomes Available

After creation, wait for the NAT Gateway state to change to **Available**.

Do not continue until the NAT Gateway is fully ready. If you add a route before the NAT Gateway is ready, the route may not become active immediately.

Also confirm that the NAT Gateway has:

- Connectivity type: `Public`
- Subnet: the public subnet
- Elastic IP: allocated

---

### Step 5: Create the Private Route Table

Create a separate route table for the private subnet. Do **not** edit the public route table created in Lab-002.

Use these values:

| Setting | Value |
|---------|-------|
| Route table name | `private` |
| VPC | `Lab-02` VPC |

After creating the route table, open it and go to the **Routes** tab.

> Note: If a `private` route table already exists for this lab, open that route table instead of creating a duplicate.

---

### Step 6: Add a Default Route to the NAT Gateway

In the `private` route table:

- Go to the **Routes** tab.
- Click **Edit routes**.
- Click **Add route**.

Add this route:

| Destination | Target |
|-------------|--------|
| `0.0.0.0/0` | NAT Gateway |

- Select the NAT Gateway created in Step 3.
- Click **Save changes**.

This route allows instances in the private subnet to send outbound traffic through the NAT Gateway.

![Add NAT Gateway Route](<images/route creation.png>)

---

### Step 7: Verify the Private Route Table Route

After saving, verify that the private route table shows:

- `0.0.0.0/0` pointing to the NAT Gateway
- Route status as **Active**
- The local VPC route still present

This confirms that the private route is configured correctly.

![Private Route Table Active Route](<images/ensure activity status.png>)

---

### Step 8: Associate the Route Table with the Private Subnet

Now associate the `private` route table with the **private subnet** created in Lab-002.

In the `private` route table:

- Go to **Subnet associations**.
- Click **Edit subnet associations**.
- Select the private subnet.
- Click **Save associations**.

This step is required. The private EC2 instance only uses the NAT Gateway if its subnet is associated with the route table that points to the NAT Gateway.

After saving, confirm that the private subnet appears under **Explicit subnet associations** for the `private` route table.

> Important: Do not associate this private route table with the public subnet. The public subnet should continue using the public route table with the Internet Gateway route.

---

### Step 9: Connect to the Private EC2 Instance Through the Bastion Host

Run each command in the correct place:

| Command | Where to run it |
|---------|-----------------|
| `chmod 400 snakepass.pem` | Local Linux terminal |
| `eval "$(ssh-agent -s)"` | Local Linux terminal |
| `ssh-add snakepass.pem` | Local Linux terminal |
| `ssh -A -i snakepass.pem ec2-user@<public-IP-of-bastion-host>` | Local Linux terminal |
| `ssh ec2-user@<private-IP-of-private-instance>` | EC2-A bastion host |
| `curl google.com` | EC2-B private instance |

From your local Linux terminal:

```bash
chmod 400 snakepass.pem
eval "$(ssh-agent -s)"
ssh-add snakepass.pem
ssh -A -i snakepass.pem ec2-user@<public-IP-of-bastion-host>
ssh ec2-user@<private-IP-of-private-instance>
```

Replace:

- `<public-IP-of-bastion-host>` with the public IPv4 address of EC2-A
- `<private-IP-of-private-instance>` with the private IPv4 address of EC2-B

Make sure you are logged in to **EC2-B** before running the internet test.

You can confirm you are on EC2-B by checking the terminal prompt or by running:

```bash
hostname -I
```

The output should show a private IP from the `192.168.200.0/24` range.

---

### Step 10: Test Internet Access from the Private EC2 Instance

Run this command from **EC2-B**, not from the bastion host:

```bash
curl google.com
```

If the NAT Gateway and private route table are configured correctly, the command should return an HTTP response from Google.

In this lab, the output may show a `301 Moved` response. That is normal and still proves that the private instance successfully reached the public internet.

This confirms the following:

- The private EC2 instance can access the internet for outbound traffic
- The NAT Gateway is working correctly
- The private route table is correctly configured
- The private subnet is associated with the correct route table

**Validation Output Screenshot:**

![Validation Output](<images/output.png>)

---

## Validation Checklist

Use this checklist to confirm the lab was completed successfully:

- Lab-002 environment is already available
- NAT Gateway created in the public subnet
- NAT Gateway state is `Available`
- Elastic IP allocated to the NAT Gateway
- Private route table created or updated
- Route `0.0.0.0/0` points to the NAT Gateway
- Route status shows `Active`
- Private route table associated with the private subnet
- Bastion host is reachable using SSH
- Private EC2 instance is reachable from the bastion host
- `curl google.com` works from the private EC2 instance
- Private EC2 instance still has no public IPv4 address

---

## Commands Used in This Lab

```bash
chmod 400 snakepass.pem
eval "$(ssh-agent -s)"
ssh-add snakepass.pem
ssh -A -i snakepass.pem ec2-user@<public-IP-of-bastion-host>
ssh ec2-user@<private-IP-of-private-instance>
hostname -I
curl google.com
```

---

## Expected Result

At the end of this lab:

- The bastion host remains in the public subnet
- The private EC2 instance remains in the private subnet
- The private EC2 instance still has no public IP address
- The private EC2 instance can access the internet for outbound traffic
- Inbound access to the private EC2 instance is still only possible through the bastion host
- `curl google.com` succeeds from the private EC2 instance

---

## Common Mistakes

- Creating the NAT Gateway in the private subnet instead of the public subnet
- Forgetting that the NAT Gateway must use a public Elastic IP
- Adding the NAT Gateway route to the wrong route table
- Forgetting to associate the private route table with the private subnet
- Testing `curl google.com` from the bastion host instead of the private EC2 instance
- Expecting the private EC2 instance to receive a public IP
- Forgetting to complete Lab-002 before starting Lab-003
- Deleting the Elastic IP before deleting the NAT Gateway

---

## Troubleshooting

| Problem | Likely Cause | Fix |
|---------|--------------|-----|
| NAT Gateway creation fails | Public subnet or Elastic IP configuration is incorrect | Make sure the NAT Gateway is created as a public NAT Gateway in the public subnet |
| Route is not active | Wrong target selected or NAT Gateway is not ready | Wait until the NAT Gateway state is `Available`, then recheck the route |
| Private EC2 cannot access the internet | Private route table is not associated with the private subnet | Associate the private route table with the private subnet |
| `curl google.com` fails from the private instance | NAT route missing or incorrect | Confirm `0.0.0.0/0` points to the NAT Gateway |
| SSH to the private instance fails | Bastion path from Lab-002 is broken | Recheck the bastion host, SSH agent forwarding, and private instance security group from Lab-002 |
| Test works from bastion but not private instance | Test was run from the wrong server | SSH into EC2-B first, then run `curl google.com` |

---

## Cleanup

NAT Gateway can create AWS charges while it is running. Delete it after the lab if you do not need it.

If you want to remove only the Lab-003 extension while keeping Lab-002:

1. Disassociate the private route table from the private subnet if needed.
2. Delete the route `0.0.0.0/0` pointing to the NAT Gateway.
3. Delete the NAT Gateway.
4. Wait until the NAT Gateway deletion is complete.
5. Release the associated Elastic IP if it is no longer needed.

If you want to remove the full environment, follow the cleanup steps in Lab-002 after deleting the NAT Gateway and releasing the Elastic IP.

---

## Conclusion

In this lab, the environment created in Lab-002 was extended by deploying a NAT Gateway in the public subnet and updating the private subnet routing to forward outbound traffic through that NAT Gateway. This enabled the private EC2 instance to access public internet resources without assigning it a public IP address.

The private EC2 instance remained isolated from direct inbound internet access, while still gaining the ability to download packages, perform updates, and reach external services. This demonstrates a common AWS networking pattern where private workloads keep their security boundary while maintaining controlled outbound internet connectivity.

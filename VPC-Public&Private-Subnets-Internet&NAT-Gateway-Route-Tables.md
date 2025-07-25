Here’s a **complete guide** to create a **custom VPC** with:

* ✅ 1 VPC
* ✅ 3 Public Subnets
* ✅ 2 Private Subnets
* ✅ 1 Internet Gateway
* ✅ 1 NAT Gateway
* ✅ Route Tables with proper associations

---

## 📘 Example IP Address Plan

| Component            | CIDR Block      | Availability Zone |
| -------------------- | --------------- | ----------------- |
| **VPC**              | `10.0.0.0/16`   | -                 |
| **Public Subnet 1**  | `10.0.1.0/24`   | `ap-south-1a`     |
| **Public Subnet 2**  | `10.0.2.0/24`   | `ap-south-1b`     |
| **Public Subnet 3**  | `10.0.3.0/24`   | `ap-south-1c`     |
| **Private Subnet 1** | `10.0.101.0/24` | `ap-south-1a`     |
| **Private Subnet 2** | `10.0.102.0/24` | `ap-south-1b`     |

---

## ✅ Step-by-Step Instructions (AWS Console)

---

### 🔹 1. Create a VPC

1. Go to **VPC Dashboard** → **Your VPCs** → **Create VPC**
2. Name: `custom-vpc`
3. IPv4 CIDR block: `10.0.0.0/16`
4. Tenancy: Default
5. Click **Create VPC**

---

### 🔹 2. Create Subnets

#### ➤ Public Subnets

Repeat for each:

* Go to **Subnets** → **Create Subnet**
* Select `custom-vpc`
* Example for Public Subnet 1:

  * Name: `public-subnet-1`
  * AZ: `ap-south-1a`
  * CIDR: `10.0.1.0/24`

Repeat for:

* `public-subnet-2` → AZ `ap-south-1b` → `10.0.2.0/24`
* `public-subnet-3` → AZ `ap-south-1c` → `10.0.3.0/24`

#### ➤ Private Subnets

* `private-subnet-1` → AZ `ap-south-1a` → `10.0.101.0/24`
* `private-subnet-2` → AZ `ap-south-1b` → `10.0.102.0/24`

---

### 🔹 3. Create and Attach Internet Gateway

1. Go to **Internet Gateways** → **Create IGW**
2. Name: `custom-igw`
3. Create → Then **Attach** to `custom-vpc`

---

### 🔹 4. Create Elastic IP for NAT Gateway

1. Go to **Elastic IPs** → **Allocate Elastic IP**
2. Allocate → Note the IP

---

### 🔹 5. Create NAT Gateway

1. Go to **NAT Gateways** → **Create NAT Gateway**
2. Name: `custom-nat`
3. Subnet: Choose **one public subnet**, e.g., `public-subnet-1`
4. Elastic IP: Use the one you created above
5. Click **Create**

---

### 🔹 6. Create Route Tables

#### ➤ Public Route Table

1. Go to **Route Tables** → **Create Route Table**
2. Name: `public-rt`, VPC: `custom-vpc`
3. Edit Routes:

   * Destination: `0.0.0.0/0`
   * Target: **Internet Gateway**
4. **Subnet Associations**:

   * Attach all **public subnets** (`public-subnet-1/2/3`)

#### ➤ Private Route Table

1. Create another route table: `private-rt`
2. Edit Routes:

   * Destination: `0.0.0.0/0`
   * Target: **NAT Gateway**
3. Subnet Associations:

   * Attach **private-subnet-1 and 2**

---

## 🧭 Architecture Overview

```
                          [ Internet ]
                               |
                         +-----v-----+
                         |  IGW      |
                         +-----+-----+
                               |
                          [ VPC: 10.0.0.0/16 ]
                               |
         +--------+------------+--------------+------------+
         |        |            |              |            |
 +-------v+  +----v----+  +----v----+   +------v-----+ +----v-----+
 | Pub AZ1 |  Pub AZ2 |  Pub AZ3 |   | Priv AZ1  | | Priv AZ2 |
 |10.0.1.0 | 10.0.2.0 | 10.0.3.0 |   | 10.0.101.0| |10.0.102.0|
 +--+------+----------+----------+   +-----+------+ +----+----+
    |                                 NAT Gateway
    |                                    |
Public RT: IGW                 Private RT: NAT → EIP
```

---

## ✅ Summary Checklist

| Component               | Attached To     | Route to            |
| ----------------------- | --------------- | ------------------- |
| **Internet Gateway**    | VPC             | Public Route Table  |
| **NAT Gateway**         | Public Subnet   | Private Route Table |
| **Public Route Table**  | Public Subnets  | IGW for `0.0.0.0/0` |
| **Private Route Table** | Private Subnets | NAT for `0.0.0.0/0` |

---

Would you like this setup as **Terraform code**, **AWS CLI commands**, or a **CloudFormation template** next?

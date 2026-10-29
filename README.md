# AWS VPC Peering Project

**Name:** Damor Dharmeshbhai
**Batch:** Aws Internship Batch 1
**Project:** EWB Cloud Labs, Project 01: VPC Peering
**Region:** ap-southeast-2 (Sydney)
**Completion date:** 29 October 2026

---

## 1. Architecture and resource inventory

Two VPCs with non-overlapping CIDRs are connected by a VPC peering connection. A single public bastion in VPC A is the only entry point from the internet. App-A and App-B have no public IP.

- **VPC A** `10.10.0.0/16`: public subnet `10.10.1.0/24` (bastion), private subnet `10.10.2.0/24` (App-A), internet gateway attached
- **VPC B** `10.20.0.0/16`: private subnet `10.20.1.0/24` (App-B), no internet gateway
- **Peering:** `ewb-a-b-peer` between VPC A (requester) and VPC B (accepter)
- **Administration path:** laptop → bastion → App-A or App-B using SSH ProxyJump

| Resource | Name | ID / Address |
|---|---|---|
| VPC A | ewb-vpc-a | vpc-0d35621636b3a1e75 (10.10.0.0/16) |
| VPC B | ewb-vpc-b | vpc-0122fbb915ca6ac7e (10.20.0.0/16) |
| Public subnet A | ewb-a-public | subnet-03ea7ed1d12ca9ade (10.10.1.0/24) |
| Private subnet A | ewb-a-private | subnet-0df16d510fc7bd7ec (10.10.2.0/24) |
| Private subnet B | ewb-b-private | subnet-08692cc574e48e835 (10.20.1.0/24) |
| Internet gateway | ewb-igw-a | igw-077e0f903c26b4435 |
| Route table (public A) | ewb-rt-a-public | rtb-0e5322998a66ac563 |
| Route table (private A) | ewb-rt-a-private | rtb-0eae7f7e64d56d82d |
| Route table (private B) | ewb-rt-b-private | rtb-0b3a8eaef0ca50038 |
| Security group | ewb-sg-bastion | sg-076bc35d109fce980 |
| Security group | ewb-sg-app-a | sg-0545d37e502d0a7a1 |
| Security group | ewb-sg-app-b | sg-079f309d4ef181a66 |
| Instance | ewb-bastion | i-0baafdc00e9e11b08 (10.10.1.10, public IP 3.106.217.20) |
| Instance | ewb-app-a | i-0ecb621e1b82edca5 (10.10.2.10, no public IP) |
| Instance | ewb-app-b | i-07fd33168860fa1c8 (10.20.1.10, no public IP) |
| Peering connection | ewb-a-b-peer | pcx-0c917bf664c0831c2 |

All instances: Amazon Linux 2023, t3.micro, key pair `ewb-lab`, Availability Zone ap-southeast-2a.

---

## 2. VPC A, subnets, internet gateway and route tables (Checkpoint 01)

![VPC A](screenshots/01-vpc-a.png)
![Public subnet](screenshots/02-subnet-a-public.png)
![Private subnet](screenshots/03-subnet-a-private.png)
![Internet gateway attached](screenshots/04-igw-a-attached.png)
![Public route table, initial](screenshots/05-rt-a-public-initial.png)
![Private route table, initial](screenshots/06-rt-a-private-initial.png)

Before peering, `ewb-rt-a-public` had `10.10.0.0/16 → local` and `0.0.0.0/0 → ewb-igw-a`. `ewb-rt-a-private` had only the local route.

**Explanation:** The public subnet is public because its route table has a route to the internet gateway (`0.0.0.0/0 → ewb-igw-a`) and the bastion has a public IP. The private subnet has no internet route and its instances have no public IP, so nobody on the internet can reach them directly.

## 3. VPC B (Checkpoint for VPC B)

![VPC B](screenshots/07-vpc-b.png)
![Private subnet B](screenshots/08-subnet-b-private.png)
![Route table B, initial](screenshots/09-rt-b-private-initial.png)

VPC B has no internet gateway. Its route table had only the local route `10.20.0.0/16`.

---

## 4. Security groups

| Security group | VPC | Inbound rule | Source |
|---|---|---|---|
| ewb-sg-bastion | A | SSH TCP 22 | my public IP /32 |
| ewb-sg-app-a | A | SSH TCP 22 | 10.10.1.10/32 (bastion) |
| ewb-sg-app-a | A | ICMP Echo Request | 10.20.1.10/32 (App-B) |
| ewb-sg-app-b | B | SSH TCP 22 | 10.10.1.10/32 (bastion) |
| ewb-sg-app-b | B | ICMP Echo Request | 10.10.2.10/32 (App-A) |

![Bastion SG](screenshots/10-sg-bastion.png)
![App-A SG](screenshots/11-sg-app-a.png)
![App-B SG](screenshots/12-sg-app-b.png)

Outbound rules are the default allow-all. No rule uses `0.0.0.0/0` for SSH or ICMP.

---

## 5. Instances (Checkpoint 02)

![Instances running](screenshots/13-instances-running.png)
![Bastion details](screenshots/14-bastion-details.png)
![App-A details](screenshots/15-app-a-details.png)
![App-B details](screenshots/16-app-b-details.png)

The bastion has a public IPv4 address. App-A and App-B have an empty Public IPv4 field.

**Test 05, Isolation: PASS.** Both private hosts have no public IPv4.

---

## 6. SSH access

![SSH config](screenshots/17-ssh-config.png)
![SSH to bastion](screenshots/18-ssh-bastion.png)
![SSH to App-A](screenshots/19-ssh-app-a.png)

The private key stays on my laptop and was never copied to any instance. App-A and App-B are reached through `ProxyJump ewb-bastion`.

---

## 7. Test 01: Baseline before peering

![Baseline ping failure](screenshots/20-test01-baseline-ping-fail.png)

From App-A (`10.10.2.10`): `ping -c 4 10.20.1.10` gave **4 transmitted, 0 received, 100% packet loss**. No peering and no route to VPC B existed yet.

**Result: PASS (expected failure).**

---

## 8. Peering (Checkpoint 03)

![Peering active](screenshots/21-peering-active.png)

Connection `ewb-a-b-peer` (`pcx-0c917bf664c0831c2`) was requested from VPC A, accepted in VPC B, and shows status **Active**. Requester CIDR `10.10.0.0/16`, accepter CIDR `10.20.0.0/16`, both in Sydney (ap-southeast-2).

An Active peering alone does not make ping work. Routes and security groups are still needed.

---

## 9. Routes (Checkpoint 04)

| Route table | Destination | Target |
|---|---|---|
| ewb-rt-a-private | 10.10.0.0/16 | local |
| ewb-rt-a-private | 10.20.0.0/16 | pcx-0c917bf664c0831c2 |
| ewb-rt-b-private | 10.20.0.0/16 | local |
| ewb-rt-b-private | 10.10.0.0/16 | pcx-0c917bf664c0831c2 |
| ewb-rt-a-public | 10.10.0.0/16 | local |
| ewb-rt-a-public | 0.0.0.0/0 | igw-077e0f903c26b4435 |
| ewb-rt-a-public | 10.20.0.0/16 | pcx-0c917bf664c0831c2 |

![Private A routes](screenshots/22-rt-a-private-final.png)
![Private B routes](screenshots/23-rt-b-private-final.png)
![Public A routes](screenshots/24-rt-a-public-final.png)

---

## 10. Tests after routes

### Test 02: Forward path (App-A → App-B)
![Forward ping](screenshots/25-test02-forward-ping.png)

`ping -c 4 10.20.1.10` from App-A: **4 transmitted, 4 received, 0% packet loss**, rtt min/avg/max = 0.189/0.278/0.536 ms. **PASS.**

### Test 03: Reverse path (App-B → App-A) and Test 04: Private SSH
![Reverse ping and SSH](screenshots/26-test03-04-reverse-ping-and-ssh.png)

`ssh ewb-app-b` from my laptop opened a shell on App-B through the bastion (`hostname -I` showed `10.20.1.10`). Then `ping -c 4 10.10.2.10` gave **4 received, 0% packet loss**, rtt avg 0.236 ms. **PASS.**

---

## 11. Controlled failure and recovery

1. Removed only the ICMP Echo Request rule from `ewb-sg-app-b`. The SSH rule stayed.
   ![Rule removed](screenshots/27-sg-app-b-icmp-removed.png)
2. Ping from App-A to App-B: **100% packet loss.**
   ![Failed ping](screenshots/28-failed-ping-after-rule-removal.png)
3. Restored the rule: Custom ICMP, Echo Request, source `10.10.2.10/32`.
   ![Rule restored](screenshots/29-sg-app-b-icmp-restored.png)
4. Ping again: **4 received, 0% packet loss.**
   ![Recovery ping](screenshots/30-recovery-ping.png)

**Explanation:** The peering connection and both route tables stayed correct the whole time. The ping failed only because `ewb-sg-app-b` no longer allowed Echo Request from App-A, so App-B dropped the packets. SSH still worked because I kept the SSH rule. After I restored the exact ICMP rule, the replies came back. This shows that routing only gives a path, and the security group decides what traffic is allowed on that path.

---

## 12. Forward path and return path

**Explanation:**

- **Forward path:** App-A sends a packet to `10.20.1.10`. `ewb-rt-a-private` has `10.20.0.0/16 → pcx`, so the packet goes over the peering connection into VPC B. `ewb-sg-app-b` allows Echo Request from `10.10.2.10/32`.
- **Return path:** App-B replies to `10.10.2.10`. `ewb-rt-b-private` has `10.10.0.0/16 → pcx`, so the reply goes back over the peering connection. The security group is stateful, so the reply is allowed automatically.
- Without the return route, the reply would be dropped and the ping would fail even though the forward route exists.

---

## 13. Questions I should be ready to explain

1. **Why is the return route needed?** Replies from App-B need a path back to `10.10.0.0/16`. Without a route in `ewb-rt-b-private`, VPC B has no way to send them to VPC A.
2. **Why is the App-B SSH source the bastion private IP?** With ProxyJump, the bastion opens a new SSH connection to App-B from its private address `10.10.1.10`. App-B never sees my laptop.
3. **Why does Active peering not guarantee a successful ping?** Peering is only the link. Route tables on both sides and the destination security group must also allow the traffic.
4. **Can VPC B use VPC A's internet gateway over peering?** No. Peering is one-to-one and non-transitive, and it does not forward traffic to another VPC's internet gateway.

---

## 14. Cleanup evidence

Resources were deleted in dependency order: instances, peering routes and peering connection, security groups, subnets, route tables, internet gateway, VPCs, key pair, then the local key.

![Instances terminated](screenshots/cleanup/c01-instances-terminated.png)
![Volumes empty](screenshots/cleanup/c02-volumes-empty.png)
![Elastic IPs empty](screenshots/cleanup/c03-elastic-ips-empty.png)
![Peering deleted](screenshots/cleanup/c04-peering-deleted.png)
![Route tables](screenshots/cleanup/c05-route-tables.png)
![Subnets](screenshots/cleanup/c06-subnets.png)
![Internet gateways](screenshots/cleanup/c07-internet-gateways.png)
![VPCs](screenshots/cleanup/c08-vpcs.png)
![Security groups](screenshots/cleanup/c09-security-groups.png)
![Key pairs](screenshots/cleanup/c10-key-pairs.png)
![Laptop cleanup](screenshots/cleanup/c12-laptop-cleanup.png)

Only default-VPC resources remain from this lab. A stopped instance and a key pair named `verticalscaling` remain in the account. They come from an earlier project and are not part of this lab.

---

## 15. What I learned

I learned that VPC peering is only a link: it works only when both VPCs have routes to each other's CIDR and the destination security group allows the traffic. I saw that a missing return route or a missing security group rule makes the ping fail even when the peering is Active. I also learned how a bastion host with SSH ProxyJump lets me manage private servers without public IPs or copying keys to them. Finally, I learned to check things in order (instance, IP, route table, security group) when something does not connect, and to delete resources in the right order to avoid extra cost.

---

## Files in this repository

- `screenshots/` evidence screenshots, in the order of this report
- `screenshots/cleanup/` cleanup screenshots
- `ssh-config.example` SSH config with the bastion IP replaced by a placeholder
- No private keys (`.pem`) are included.

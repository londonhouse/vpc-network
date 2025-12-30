# VPC Standards & Practices: Networking
## Designing an Efficient Hub-and-Spoke Network for Multi-Account AWS

This section of VPC standards and practices focuses on the **core network**. When we talk about efficiency, we’re really trying to answer one fundamental question:

**How do you design a network that scales across many AWS accounts without becoming difficult to manage, hard to observe, or nearly impossible to analyze?**

In my experience, the **hub-and-spoke model** is one of the most efficient ways to design enterprise VPC networking. It cleanly separates responsibilities, reduces repetition, and gives platform and networking teams the leverage they need as environments grow—regardless of speed.

This section focuses on **why that model works so well in multi-account AWS**, and how the underlying components support that goal.

---

## The Core Idea: Separate Network Responsibility from Application Responsibility

In enterprise environments, accounts naturally fall into two broad categories:

### Function-owned accounts
- Networking  
- Shared services  
- Audit and security  
- Platform utilities  

### Business-unit-owned accounts
- Application workloads  
- Data services  
- Product teams  

The hub-and-spoke model is effective because it respects this reality.

Each business unit owns its workloads.  
Each function-owned account provides a centralized capability.

Networking is not something every team should have to reinvent. By centralizing the **network function** while allowing each account to own its **applications**, you gain clarity without sacrificing autonomy.

---

## Hub-and-Spoke Solves a Very Specific Problem

At its center, this model answers one primary question:

**How do you empower network simplicity in a centralized way across a large environment?**

The answer looks like this:

- Each account maintains its own VPC and subnets  
- All network traffic flows through a shared hub  
- Routing, inspection, and outbound control live in one place  

Application teams stay focused on shipping workloads, while networking teams stay focused on traffic flow, security, and reliability.

---

## How This Helps in a Multi-Account Environment

Whether a company’s cloud footprint grows through migration, acquisition, or organically, **poor network design introduces inefficiency quickly**.

Common symptoms include:
- Repeating network appliances in every account  
- Inconsistent routing behavior  
- IP overlap during acquisitions  
- Too many places to troubleshoot the same issue  

The hub-and-spoke model eliminates these problems by creating **predictable network paths**.

Instead of configuring networking independently in every account, teams deploy **one standardized spoke** that connects to a shared hub. From that point forward, network behavior is known, repeatable, and centrally observable.

When you acquire a new team or spin up a new business unit, onboarding becomes **infrastructure as code, not a redesign exercise**.

---

## Transit Gateways Make the Model Work

The major component that enables this model in AWS is the **Transit Gateway**.

The Transit Gateway lives in the **network hub account** and acts as the primary routing component between all spoke VPCs. AWS enables this multi-account connectivity using **AWS Resource Access Manager (RAM)**, allowing each account to attach its VPC to the shared hub without owning it.

### Breaking this down:
- ENIs in private subnets connect to TGW attachments  
- The TGW controls routing between VPCs  
- The hub account controls outbound access and inspection  
- Spoke accounts remain isolated, but connected  

This allows routing to be managed in **one place**, instead of being duplicated across many accounts.

It also enables **controlled east-west traffic**. If application accounts need to communicate with:
- Shared services  
- Logging platforms  
- Key-value stores or secret utilities like HashiCorp Vault  

that routing happens through **known paths with explicit intent**. Nothing is accidentally reachable.

---

## Architecture Explained

In this model, application VPCs typically use **private subnets only**. This creates clarity.

If a workload needs:
- Internet access  
- Shared services  
- External integrations  

it goes through the hub.

This guarantees:
- Consistent inspection  
- Centralized egress control  

---

## Closing: Build for Scale Before You Need It

If you’re operating more than a few AWS accounts, networking stops being a configuration problem and becomes an **organizational problem**.

The hub-and-spoke model provides:
- Efficiency  
- Predictability  
- Clear ownership  
- Scalable onboarding  

It also enables the effective use of Infrastructure as Code to:
- Control CIDR allocation  
- Prevent IP overlap  
- Standardize attachments  
- Spawn new spokes safely and repeatedly  

All of this makes environments easier to audit, troubleshoot, and evolve over time.

This networking approach forms the **foundation** for everything that follows: auditing, logging, identity, and shared services, which are covered in subsequent sections of these VPC standards and practices.


---
title: "Projects"
permalink: /projects/
layout: single
---

### Blue-Green Deployment with Smart Routing on Amazon EKS

**Role:** DevOps Engineer / CDN Team

**Technologies:** Amazon EKS, AWS ALB, Akamai CDN, Kubernetes, GTM, Property Manager, DNS Routing

#### Overview

Led the implementation of a blue-green deployment strategy for [www.marriott.com](http://www.marriott.com), enabling seamless feature rollouts and zero-downtime releases for critical production applications.

#### Problem

Traditional deployments to our production EKS clusters carried the risk of downtime and limited our ability to safely validate new features in a live environment.

#### Solution

Designed and implemented a blue-green deployment workflow using Amazon EKS and Application Load Balancer (ALB), integrated with Akamai CDN. We established parallel “blue” (current) and “green” (new) Kubernetes deployments, allowing traffic to be routed dynamically based on cookies, request headers, or query string parameters (e.g., `?AKAQA=qa23b` or `?AKAQA=green`).

* Special host headers were injected by Akamai CDN and evaluated by the ALB for smart traffic targeting.
* GTM and DNS configuration allowed targeted user groups or QA engineers to access either environment for testing, using simple URL parameters or cookies.
* Routing logic was managed through property manager rules, setting specific variables to control backend environment selection and enabling granular rollout and rollback.

#### Impact

* Achieved zero-downtime deployments, significantly reducing production risk.
* Enabled targeted testing of new features in production-like environments without exposing all users.
* Improved release velocity and confidence, as rollbacks and progressive rollouts became trivial.

#### Key Takeaways

* Integrating CDN-level controls (Akamai) with Kubernetes-native blue-green deployment enabled advanced testing and release patterns.
* Dynamic traffic routing via cookies, headers, and QSPs empowered both QA and business teams with flexible, safe deployment options.



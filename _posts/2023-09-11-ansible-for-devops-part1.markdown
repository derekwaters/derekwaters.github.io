---
layout: post
title:  "Ansible for DevOps - Part 1"
date:   2023-09-11 22:00:00 +1000
categories: ansible devops
---
## The Use Case

A potential automation user with a well-defined pipeline for delivering Kubernetes applications to the cloud had a need to expand their framework to deliver VM-based applications to a public cloud. Their basic requirements were:

- Use existing Terraform automation to provision cloud infrastructure
- Separation of concerns - let the app owners be responsible for pre-rollout checks, application installation and verification testing
- Integration with Service Management tools (specifically ServiceNow)
- Automate as much of the process as possible to speed time-to-market for applications
- Ability to target multiple environments to support blue/green or canary deployments

The customer was aware of Ansible, and other parts of their business were already strong users of Ansible Automation Platform.

## The Plan

The plan was to deploy a full example workflow in AAP. I wanted to demonstrate:
- Integration with Service Management
- GitOps principles
- Automated triggering of automation
- Separation of concerns and RBACs
- Logical branching of workflows

I also wanted to demonstrate “config-as-code” principles so the entire demo system (almost) was built as Ansible playbooks. The intent was to start with a blank OpenShift environment, and then:

1. Deploy the AAP Operator
2. Deploy AAP
3. Deploy (and link) EDA
4. License AAP
5. Deploy Ansible resources for the demo
6. Integrate ServiceNow
7. Trigger Workflow with GitHub

The deployed architecture should look like this:

![Ansible DevOps Demo Architecture](/img/blog1.png)

In the [next post]({% link _posts/2023-09-14-ansible-for-devops-part2.markdown %}) we'll actually start installing Ansible Automation Platform using Ansible.

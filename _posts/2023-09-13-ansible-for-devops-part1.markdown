---
layout: post
title:  "Ansible for DevOps - Part 1"
date:   2023-09-13 22:00:00 +1000
categories: ansible devops
---
## The Use Case

A customer with a well-defined pipeline for delivering Kubernetes applications to the cloud had a need to expand their framework to deliver VM-based applications to a public cloud. Their basic requirements were:

- Use existing Terraform automation to provision cloud infrastructure
- Separation of concerns - let the app owners be responsible for pre-rollout checks, application installation and verification testing
- Integration with Service Management tools (specifically ServiceNow)
- Automate as much of the process as possible to speed time-to-market for applications
- Ability to target multiple environments to support blue/green or canary deployments

The customer was aware of Ansible, and other parts of the business were already strong users of Ansible Automation Platform.

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

{% highlight ansible %}
- name: Log in to the OpenShift API
  redhat.openshift.openshift_auth:
    host: "{{ ocp_host }}"
    username: "{{ ocp_username }}"
    password: "{{ ocp_password }}"
    validate_certs: false
    state: present
  register: openshift_auth_results
{% endhighlight %}

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/

---
layout: post
title:  "Ansible for DevOps - Part 3"
date:   2023-09-15 22:00:00 +1000
categories: ansible devops
---
## Deploy Automation Resources

Now that AAP and EDA are up and running, I start setting up the automation resources needed for the demo. This includes organizations, demo inventories, credentials and projects. The structure of what is created looks something like this:

![AAP Resource Organisation](/img/blog4.png)

The playbook also creates a series of job templates to simulate each of the stages of the application deployment, and an overall workflow to organise them. These stages are:

| Name | Organization | Task |
| ---- | ------------ | ---- |
| 1-create | devplatform | Creates a ServiceNow Change Request |
| 2-prerollout | appowner | Perform any application-specific pre-rollout checks required. |
| S0-scheduled | devplatform | Update Change Request status to “scheduled” |
| S1-started | devplatform | Update Change Request status to “implement” |
| S2-failed | devplatform | Update Change Request status to “canceled” |
| S3-review | devplatform | Update Change Request status to “review” |
| S4-complete | devplatform | Update Change Request status to “closed” with closure code and notes |
| S5-failed | devplatform | Update Change Request status to “canceled” |
| 3-provision | devplatform | Provision the infrastructure required for the application installation |
| 4-postprovision | appowner | Perform application installation and configuration on known infrastructure |
| 5-tvt | appowner | Perform diagnostic tests on the application to ensure deployment has succeeded |
| I1-incident | devplatform | Raise an Incident in ServiceNow |

The final workflow, with branching logic, looks like this:

![Final Workflow](/img/blog5.png)

The workflow job template also supports a parameter to “simulate” a failure in the workflow - rollout_status which can have the following values:

- fail_tvt - cause the workflow to fail at the TVT stage
- fail_precheck - cause the workflow to fail at the Precheck stage
- Any other value - let the workflow succeed

When a workflow template stage fails, the workflow will generate an Incident in ServiceNow (see below).

## GitOps and Webhooks

To enable a webhook on the workflow template, allowing it to be triggered by a GitHub push, I set the webhook_service param to “github” when creating the workflow job template:

{% highlight ansible %}
- name: Create a workflow template
  ansible.controller.workflow_job_template:
    name: "rollout-app-impl"
    description: "Sample App Deployment Workflow with ServiceNow integration and Multiple Teams"
    organization: "{{ org_platform.name }}"
    inventory: "{{ org_platform.name }}-assets"
    webhook_service: "github"
    ...
{% endhighlight %}

However, the returned data from this play doesn’t include the webhook endpoint and the webhook key needed to configure GitHub. It took me a while to find a way to retrieve this. The endpoint can be retrieved by doing a lookup on the ansible.controller.controller_api module for the newly created workflow job template.

The returned data includes the endpoint path in the related.webhook_receiver parameter, so the full URL can be constructed:

{% highlight ansible %}
- name: Load the workflow template settings to get webhook details
  ansible.builtin.set_fact:
    workflow_template: "{{ lookup('ansible.controller.controller_api', 'workflow_job_templates', query_params = { 'name' : 'rollout-app-impl' }, host = aap_host, username = aap_username, password = aap_password, verify_ssl = False) }}"
- name: Set the webhook_receiver URL
  ansible.builtin.set_fact:
    webhook_receiver: "https://{{ aap_host }}{{ workflow_template.related.webhook_receiver }}"
{% endhighlight %}

However, the webhook key isn’t directly returned. This is referenced in the API as a child object of the main workflow_job_template. It turns out you can query this using the ansible.controller.controller_api lookup, even though it looks a little dodgy!

{% highlight ansible %}
- name: Try and get the webhook secret
  ansible.builtin.set_fact:
    key_deets: "{{ lookup('ansible.controller.controller_api',
      'workflow_job_templates/{{ workflow_template.id }}/webhook_key',
      host = aap_host,
      username = aap_username,
      password = aap_password,
      verify_ssl = False) }}"
- name: Set the webkook key
  ansible.builtin.set_fact:
    webhook_key: "{{ key_deets.webhook_key }}"
{% endhighlight %}

We now have the webhook endpoint and key ready to configure in GitHub. Currently this is done manually, but could also be automated.




In the [next post]({% link _posts/2023-09-16-ansible-for-devops-part4.markdown %}) we'll set up some demo RBACs on our organisations.

---
layout: post
title:  "Ansible for DevOps - Part 4"
date:   2023-09-16 22:00:00 +1000
categories: ansible devops
---
## Deploy RBACs

The RBACs defined in the demo are not as fine-grained as should be done in a very security-conscious organisation. In particular, granting the auditor role to the teams against each organization is not ideal. However, for the purposes of the demo, the RBACs show that the app team has visibility and control only over the app team-related job templates, and the platform team has visibility of the app-related job templates, but visibility and control of the remaining job templates and the overall workflow.

Each organization (app and platform) has a single team defined, with requisite roles against the organization. Each team has a single user (Debra Developer on the app team, and Paul Platform on the platform team) with a default password. During a demo, you can log in to the AAP UI using each of these users to show the RBAC model and the restricted access each user has to the overall process. Again, in a properly secured AAP setup, more teams would be added to each organization to further restrict users and roles.

## Integrate ServiceNow

ServiceNow allows you to create a free developer instance. This demo uses one of those instances to create and update Change Requests, and to create Incident Reports. All interactions with ServiceNow use the Ansible servicenow.itsm collection. The hostname, username and password for access to the ServiceNow instance should be passed to the demo playbook as extra_vars.

Note that updating ServiceNow Change Requests requires following the defined change process in ServiceNow, including ensuring that state changes are valid, and including values like ‘Assignment Groups’ at various statuses.

When the workflow executes successfully, a Change Request is created and updated through the entire Change process, as shown below:

![ServiceNow Change Request](/img/blog6.png)

![ServiceNow Change Request Notes](/img/blog7.png)

If something goes wrong with the workflow, an incident is raised in ServiceNow, as shown below:

![ServiceNow Incident](/img/blog8.png)

In the [final post]({% link _posts/2023-09-17-ansible-for-devops-part5.markdown %}) we'll look at future enhancements and list some references.

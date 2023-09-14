---
layout: post
title:  "Ansible for DevOps - Part 2"
date:   2023-09-14 22:00:00 +1000
categories: ansible devops
---
## Deploy AAP

To deploy AAP I first needed to log in to OpenShift. This is done with the redhat.openshift collection, and generates a token used for subsequent automation calls. To do this, the playbook needs the OpenShift API server hostname, and the username and password of an account with API access. These are passed into the playbook as extra_vars.

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

I then install the Operator by creating a namespace, an operator group and then subscribing to the AAP operator itself.

Before installing AAP, we need to wait until the operator is available. Working out how to test for this took a little while to figure out, but the way I settled on was to make Kubernetes info requests to retrieve the Operator Subscription until the list of available operator resources contains more than one item.



Once that’s done, I can install AAP Controller itself. Before deploying EDA, we need to wait until AAP is available. To do this, I try to retrieve the specified Route (Ingress) into AAP. When that is available, I proceed with the installation. 

I also retrieve the generated admin password for AAP. This could have been specified as part of the installation, but it was just as easy to have AAP generate a random one.



## Deploy EDA

The next step is to use the operator to deploy EDA. To do this, I need the AAP URL obtained from the Route previously. This is provided to the operator to install EDA as a Kubernetes spec parameter.



As with AAP, I wait for the EDA Route URL and EDA admin password to be available to determine when EDA is installed.

## License AAP

This is an important step. Without it, the subsequent process of adding Ansible resources works until you get to the point of adding a host to an inventory:



If you haven’t licensed AAP, you get this pretty obscure error message:



If you log in to the AAP Console at this point (with the URL and credentials that the playbook above contains), you see the licensing UI:



I didn’t want to have to manually perform this licensing step in the middle of a fully automated process. It turns out you can automate this with the ansible.controller collection. This needs you to provide a manifest archive file to the automation script:



Instructions on how to generate the manifest from the Red Hat Cloud Console can be found [here](https://docs.ansible.com/automation-controller/4.4/html/userguide/import_license.html#obtain-sub-manifest)

The path to the local zip file used to license AAP should be passed to the playbook as an extra_var.



In the [next post]({% link 2023-09-14-ansible-for-devops-part2.markdown &}) we'll actually start installing Ansible Automation Platform using Ansible.

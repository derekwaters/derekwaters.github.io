---
layout: post
title:  "Ansible for DevOps - Part 2"
date:   2023-09-14 22:00:00 +1000
categories: ansible devops
---
## Deploy AAP

To deploy AAP I first needed to log in to OpenShift. This is done with the redhat.openshift collection, and generates a token used for subsequent automation calls. To do this, the playbook needs the OpenShift API server hostname, and the username and password of an account with API access. These are passed into the playbook as extra_vars.

{% highlight ansible %}
{% raw %}
- name: Log in to the OpenShift API
  redhat.openshift.openshift_auth:
    host: "{{ ocp_host }}"
    username: "{{ ocp_username }}"
    password: "{{ ocp_password }}"
    validate_certs: false
    state: present
  register: openshift_auth_results
{% endraw %}
{% endhighlight %}

I then install the Operator by creating a namespace, an operator group and then subscribing to the AAP operator itself.

Before installing AAP, we need to wait until the operator is available. Working out how to test for this took a little while to figure out, but the way I settled on was to make Kubernetes info requests to retrieve the Operator Subscription until the list of available operator resources contains more than one item.

{% highlight ansible %}
{% raw %}
- name: Wait for the operator to be available
  kubernetes.core.k8s_info:
    api_key: "{{ openshift_auth_results.openshift_auth.api_key }}"
    host: "{{ ocp_host }}"
    validate_certs: false
    kind: Subscription
    namespace: aap
    name: 'ansible-automation-platform'
    api_version: 'operators.coreos.com/v1alpha1'
  register: operatordetails
  until: "operatordetails.resources | length > 0"
  retries: 10
  delay: 30
{% endraw %}
{% endhighlight %}

Once that’s done, I can install AAP Controller itself. Before deploying EDA, we need to wait until AAP is available. To do this, I try to retrieve the specified Route (Ingress) into AAP. When that is available, I proceed with the installation.

I also retrieve the generated admin password for AAP. This could have been specified as part of the installation, but it was just as easy to have AAP generate a random one.

{% highlight ansible %}
{% raw %}
- name: Get AAP admin password
  kubernetes.core.k8s_info:
    api_key: "{{ openshift_auth_results.openshift_auth.api_key }}"
    host: "{{ ocp_host }}"
    validate_certs: false
    api_version: v1
    kind: Secret
    namespace: aap
    name: demo-aap-admin-password
  register: aap_admin_pwd
{% endraw %}
{% endhighlight %}

## Deploy EDA

The next step is to use the operator to deploy EDA. To do this, I need the AAP URL obtained from the Route previously. This is provided to the operator to install EDA as a Kubernetes spec parameter.

{% highlight ansible %}
{% raw %}
- name: Install EDA Controller
  redhat.openshift.k8s:
    api_key: "{{ openshift_auth_results.openshift_auth.api_key }}"
    host: "{{ ocp_host }}"
    validate_certs: false
    api_version: eda.ansible.com/v1alpha1
    kind: EDA
    resource_definition:
      metadata:
        name: demo-eda
        namespace: aap
      spec:
        replicas: 1
        automation_server_url: "{{ aap_route.resources[0].spec.port.targetPort }}://{{ aap_route.resources[0].spec.host }}"
    state: present
{% endraw %}
{% endhighlight %}

As with AAP, I wait for the EDA Route URL and EDA admin password to be available to determine when EDA is installed.

## License AAP

This is an important step. Without it, the subsequent process of adding Ansible resources works until you get to the point of adding a host to an inventory:

{% highlight ansible %}
{% raw %}
- name: Add host to inventory
  ansible.controller.host:
    name: "{{ managed_host }}"
    description: "Local Host for Inventory"
    inventory: "{{ org_platform.name }}-assets"
    state: present
    enabled: true
    variables:
      ansible_connection: local
    tower_host: "{{ aap_host }}"
    tower_username: "{{ aap_username }}"
    tower_password: "{{ aap_password }}"
{% endraw %}
{% endhighlight %}

If you haven’t licensed AAP, you get this pretty obscure error message:

![AAP Not Licensed Error Message](/img/blog2.png)

If you log in to the AAP Console at this point (with the URL and credentials that the playbook above contains), you see the licensing UI:

![AAP License Form](/img/blog3.png)

I didn’t want to have to manually perform this licensing step in the middle of a fully automated process. It turns out you can automate this with the ansible.controller collection. This needs you to provide a manifest archive file to the automation script:

{% highlight ansible %}
{% raw %}
- name: License AAP
  ansible.controller.license:
    manifest: "{{ aap_manifest_path }}"
    state: present
    tower_host: "{{ aap_host }}"
    tower_username: "{{ aap_username }}"
    tower_password: "{{ aap_password }}"
{% endraw %}
{% endhighlight %}

Instructions on how to generate the manifest from the Red Hat Cloud Console can be found [here](https://docs.ansible.com/automation-controller/4.4/html/userguide/import_license.html#obtain-sub-manifest)

The path to the local zip file used to license AAP should be passed to the playbook as an extra_var.



In the [next post]({% link _posts/2023-09-15-ansible-for-devops-part3.markdown %}) we'll deploy some automation resources to our new Ansible installation.

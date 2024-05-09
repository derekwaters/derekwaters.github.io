---
layout: post
title:  "Building Execution Environments... inside Execution Environments"
date:   2024-05-08 14:00:00 +1000
categories: ansible automation platform execution environment eda_configuration
---
## About Execution Environments

Ansible Automation Platform (AAP) introduced the idea of execution environments. These are container images that encapsulate just the libraries, collections and binaries needed to execute Ansible playbooks, and nothing more. This gives advantages in terms of:

- **Scalability** - it's easier to start up many containers to execute automation playbooks rather than wait for one large VM to execute them serially.
- **Portability** - a container image is easy to redeploy across systems, so specialised libraries for specific automation tasks are easily portable.
- **Security** - an execution environment can be built containing only the automation code necessary for a particular automation task, reducing the surface area for malicious attack.

Ansible includes a tool called [ansible-builder](https://www.ansible.com/blog/introduction-to-ansible-builder/) to make the creation of execution environments easy. Behind the scenes, this basically creates a container file definition, and then uses podman to build the container image from the container file definition.

## Automating Execution Environment Creation

Red Hat's automation Community of Practice provide a collection called [ee_utilities](https://github.com/redhat-cop/ee_utilities) which allows you to use Ansible playbooks to generate execution environments using ansible-builder (Note that this collection is also available as validated content in the Ansible Hub [here](https://console.redhat.com/ansible/automation-hub/repo/validated/infra/ee_utilities/)). Instead of defining your execution environment in an execution-environment.yaml file, you can define all of its parameters inside extra vars that get passed to the ee_builder role in the ee_utilities collection. Here's a simple example of what that might look like:

{% highlight yaml %}
{% raw %}
---
- name: Build the ee_aws image (use the ee_builder EE to do so)
  hosts: localhost
  connection: local
  become: false
  gather_facts: false
  collections:
    - infra.ee_utilities
  vars:
    ee_builder_dir_clean: false
    ee_builder_dir: "/tmp/"
    ee_base_registry_username: "{{ registryusername }}"
    ee_base_registry_password: "{{ registrypwd }}"
    ee_base_image: registry.redhat.io/ansible-automation-platform-24/ee-minimal-rhel9:latest
    ee_registry_dest: "{{ registrydest }}"
    ee_registry_username: "{{ registryusername }}"
    ee_registry_password: "{{ registrypwd }}"
    ee_pull_collections_from_hub: false

    ee_list:
      - name: ee_aws
        images:
          base_image:
            name: registry.redhat.io/ansible-automation-platform-24/ee-minimal-rhel9:latest
        alt_name: EE for Management of AWS
        tag: 1.0
        dependencies:
          python:
            - botocore
            - boto3
          galaxy:
            collections:
              - amazon.aws
  roles:
    - infra.ee_utilities.ee_builder
{% endraw %}
{% endhighlight %}

So far, so good. Using the collection means that the target inventory for your playbook needs to be a host that has ansible-builder on it. That normally means either:

1. Standing up a standalone 'builder' host with ansible-builder on it, including creating, maintaining, securing it etc.
2. Dynamically creating a host with ansible-builder installed as part of a playbook, then running the ee_builder role, then tearing down the builder, or
3. Using a Private Automation Hub host, which typically has ansible-builder installed already.

But what about using an execution environment... to build another execution environment?

![We have to go deeper](/img/ee_in_ee/deeper.png)

*(cue the Inception horns)*



## We Have To Go Deeper

What this means is running a container (from an execution environment container image) and then, from inside the container, we're going to build another container. To do that, though, the container runtime needs to *run* another container. This gets referred to as DinD (Docker in Docker) or PinP (Podman in Podman).

And yes, it is possible, though it does require some pretty creative manipulation.

[This article](https://www.redhat.com/sysadmin/podman-inside-container) gives a good deep-dive in to what needs to be done to get a podman container to run another podman container. Note that the repo referred to in that article is out of date, the correct location of the relevant container files is [here](https://github.com/containers/image_build/tree/main/podman).


OK, so that lets us run podman containers inside a podman container. But execution environments are a special-case container image (with defined entry points etc). Can we do the same thing for EEs?

The answer is yes... just.

What we're trying to achieve is something like this:

![EE-in-EE Architecture](/img/ee_in_ee/architecture.png)

We're going to build an execution environment called ee_builder (note this has the "first mover" problem - this one has to be built outside of AAP to begin with). Then we can create job templates to run playbooks using the ee_utilities collection, which will run on the ee_builder Execution Environment, using a 'localhost' inventory. That should allow us to generate new EEs from inside AAP.

## Building ee_builder

The first consideration with building our ee_builder is what tooling it needs. Obviously it needs the ansible-builder Python package, which we can add to the execution-environment.yaml as a dependency. It also needs access to two collections, containers.podman (for manipulating container images) and infra.ee_utilities.

A major problem with a container inside a container is storage and, in particular, overlay filesystems. The container runtime needs access to storage to present to the container, and for storing a cache of container images. For this reason, the execution environment also needs the fuse-overlayfs package installed.

The base image and prerequisites section of your execution-environment.yaml definition file will look something like this:

{% highlight yaml %}
{% raw %}
version: 3

images:
  base_image:
    name: "registry.redhat.io/ansible-automation-platform/ee-minimal-rhel9:2.16.5-2"

dependencies:
  system:
    - fuse-overlayfs
  python:
    - ansible-builder
  galaxy:
    collections:
      - infra.ee_utilities
      - containers.podman
{% endraw %}
{% endhighlight %}

The next problem encountered is that, despite ansible-builder creating and setting a user (user 1000) inside its generated Containerfile, when AAP runs the container image as an execution environment, it runs as root inside the container. This means that all of the overlay filesystem and container runtime configuration need to be specified for the *root* user, not the default *uid=1000* user.

With that done, and a couple of other weird bits of hackery to allow for some temp flag files to be written by the container engine, you end up with an execution environment builder execution environment. The customisation in the ee_builder execution-environment.yaml definition looks something like this:

{% highlight yaml %}
{% raw %}
additional_build_steps:
  prepend_final:
    - RUN echo root:20000:5000 >> /etc/subuid
    - RUN echo root:20000:5000 >> /etc/subgid
    - COPY _build/configs/root_containers.conf /etc/containers/containers.conf
    - RUN mkdir -p /runner/libpod/tmp
    - COPY _build/configs/storage.conf /etc/containers/storage.conf
    - RUN sed -e 's|^#mount_program|mount_program|g' -e '/additionalimage.*/a "/var/lib/shared",' -e 's|^mountopt[[:space:]]*=.*$|mountopt = "nodev,fsync=0"|g' /etc/containers/storage.conf
    - RUN mkdir -p /var/lib/shared/overlay-images /var/lib/shared/overlay-layers /var/lib/shared/vfs-images /var/lib/shared/vfs-layers; touch /var/lib/shared/overlay-images/images.lock; touch /var/lib/shared/overlay-layers/layers.lock; touch /var/lib/shared/vfs-images/images.lock; touch /var/lib/shared/vfs-layers/layers.lock
    - VOLUME /var/lib/containers
    - ENV _CONTAINERS_USERNS_CONFIGURED=""
    - ENV BUILDAH_ISOLATION=chroot
{% endraw %}
{% endhighlight %}

**Note:** this is horribly unoptimised and will generate tons of container layers. "Flattening" the build is a future exercise!

The end result of this process is an ee_builder container image in my image registry:

![ee_builder in quay.io](/img/ee_in_ee/ee_builder.png)

## Building an Execution Environment inside AAP

The final step is to get this working inside AAP itself. This involves creating an Execution Environment and pointing it at the ee_builder image:

![Execution Environment in AAP](/img/ee_in_ee/ee_aap.png)

We then create a simple playbook to generate a new execution environment (with AWS collections and python libraries). A project is created to manage this playbook, and an Inventory containing only 'localhost'. Finally, a job template pulls them all together, using the ee_builder Execution Environment to run a playbook using ee_utilities to generate a new AWS execution environment:

![Execution Environment Build Job](/img/ee_in_ee/job.png)

Once the job runs, the destination quay.io repository now contains a brand new EE built inside another EE:

![ee_aws in quay.io](/img/ee_in_ee/ee_aws.png)

Naturally, in a proper deployment, you would retrieve base images and collections from Private Automation Hub, and would push the built execution environments back into Private Automation Hub, rather than using a public image repo like quay.io.


## Conclusion

Using a Private Automation Hub node to build execution environments, or having a separate dedicated host to do so, are easy ways to automate your execution environment builds. But as AAP moves to be "everything as containers" it seems odd that there isn't an easy way to build execution environments from within an execution environment. I felt it was a technical challenge that I needed to solve to be able to do so, and the method above, while pushing the boundaries a little, finally managed to achieve it.


## References

The execution-environment.yaml definition and other associated files used to build the ee_builder image can be found [here](https://github.com/derekwaters/ansible_devops_demo/tree/main/builder/ee_builder).

The playbook that uses the ee_builder execution environment to build the AWS execution environment can be found [here](https://github.com/derekwaters/ansible_devops_demo/blob/main/builder/build_ee_aws.yml).

[How to use Podman inside a container](https://www.redhat.com/sysadmin/podman-inside-container)

---
layout: post
title:  "Execution Environments in Disconnected Environments"
date:   2023-10-30 22:00:00 +1000
categories: ansible execution environments podman disconnected
---
# Building Execution Environments in a Disconnected Environment

Ansible Automation Platform (AAP) introduced the idea of Execution Environments (EE) - customisable container images that contain all of the components needed to execute an Ansible playbook. EEs improve the consistency, security and scalability of automation tasks. AAP ships with default EEs, but often AAP users need to build their own EEs to include custom modules, libraries and binary packages.

For some users, building EEs needs to take place in a disconnected environment, without general access to the Internet. ansible-builder v3, included with AAP 2.4, gives us the flexibility to customise our EE build process to deal with these sorts of environments.

## Ansible-Builder

The tool used to build execution environments is ansible-builder. You can find the documentation and detailed information about it here. ansible-builder takes a definition file written in YAML (by default, named execution-environment.yml), and uses it to build a container image for your desired execution environment.

Along the way, ansible-builder creates the new image by building a Dockerfile/Containerfile which docker/podman can then turn into a full container image in a container image registry, along with a build context folder with additional resources. Each EE is built as layers on top of a base image, and Red Hat provides both standard and minimal EE container images through Automation Hub or Red Hat’s public image repositories.

The base EE images contain:

- ansible-core
- python
- The python modules / dependencies required for Ansible

On top of that, the execution-environment can define (amongst other things):

- The base image
- Any additional Ansible collections required (dependencies.yml)
- Any additional python modules required (dependencies.txt)
- Any additional rpm libraries needed for the build (bindeps.txt)

ansible-builder executes four stages when building a new EE. A ‘Base’ stage installs the Python, pip and ansible tools required on top of the specified base image. The ‘Galaxy’ stage retrieves any required collections from Galaxy, then the ‘Builder’ stage retrieves any additional Python or system packages required. The ‘Final’ stage is responsible for integrating the outputs of the previous stages into a final EE image.

## Disconnected Environments

If we’re trying to build EEs in a completely disconnected environment, each of the four customisable categories of items above need to be retrieved from somewhere local.

### The Base Image

For the base image, the definition file can reference any local or remote container image registry. That might be a local podman/docker repo, or it might be a Quay or Artifactory repository. As long as the repository supports OCI Container Distribution standard (which is equivalent to the Docker Registry protocol v2), any repository may be used. That also includes Private Automation Hub, which can store container images, including EEs.

### Ansible Collections

In a disconnected environment, Ansible collections can be copied from a local Private Automation Hub, rather than reaching out to Red Hat’s Automation Hub over the internet. This is done by specifying a custom ansible.cfg file which gets injected into the Galaxy stage of the build.

The custom ansible.cfg is created in a ‘files’ subdirectory in the execution environment project. To allow for correct handling of certificates, a certificate authority cert can also be added to the files subdirectory.

These files are then included in the galaxy stage of the build (for the ansible galaxy config) or the base stage of the build (for the certificate) by adding the following to the execution-environment.yml:

{% highlight yaml %}
{% raw %}
additional_build_files:
  - src: files/cert.pem
    dest: configs
  - src: files/ansible.cfg
    dest: configs

additional_build_steps:
  prepend_base:
    - COPY _build/configs/cert.pem /etc/pki/ca-trust/source/anchors/
    - RUN update-ca-trust
  prepend_galaxy:
    - COPY _build/configs/ansible.cfg /etc/ansible/ansible.cfg
{% endraw %}
{% endhighlight %}

Rather than storing Automation Hub tokens in an ansible.cfg that will be committed to source control, we can specify the token at runtime by adding another line to the prepend_galaxy build steps:

{% highlight yaml %}
{% raw %}
prepend_galaxy:
  - ARG ANSIBLE_GALAXY_SERVER_RH_CERTIFIED_REPO_TOKEN
  …

{% endraw %}
{% endhighlight %}

We can then set an environment variable in your build environment:

{% highlight yaml %}
{% raw %}
export ANSIBLE_GALAXY_SERVER_RH_CERTIFIED_REPO_TOKEN=<token from Automation Hub>
{% endraw %}
{% endhighlight %}

And then run ansible-builder telling it to use the given env var:

{% highlight yaml %}
{% raw %}
ansible-builder build --build-arg ANSIBLE_GALAXY_SERVER_RH_CERTIFIED_REPO_TOKEN
{% endraw %}
{% endhighlight %}

### Additional Python Modules

This is where things get interesting. If our build environment doesn’t have access to pypi.org for pip to retrieve Python libraries, we can establish a local Pypi mirror in our disconnected environment using tools like Sonatype Nexus or JFrog Artifactory, or even just a file repository transferred and served up with a web server. The environment you use to run ansible-builder might already be configured to use our local mirror, but when we run ansible-builder we find that pip fails to find our additional modules. Huh? This is because the base image run by ansible-builder has a default pip configuration that will use pypi.org as the repository. How can we get around that?

Fortunately, ansible-builder v3 gives us much more granular control of build step customisation, without having to manually edit Containerfiles, and so we can use that to configure new pip settings for our build process.

Firstly, we define a custom pip.conf file in our EE project directory (say, in a ‘files’ subdirectory):

{% highlight ini %}
{% raw %}
[global]
index=https://my-pypi-mirror-host/nexus/repository/pypi-group/pypi
index-url=https://my-pypi-mirror-host/nexus/repository/pypi-group/simple
trusted-host=my-pypi-mirror-host
{% endraw %}
{% endhighlight %}

Then, in our execution-environment.yml, we define the following:

{% highlight yaml %}
{% raw %}
additional_build_files:
  - src: files/pip.conf
	dest: configs

additional_build_steps:
  prepend_base:
	- COPY _build/configs/pip.conf /etc/pip.conf
{% endraw %}
{% endhighlight %}

The additional_build_files definition takes our pip.conf and injects the file into the base image in the _build/configs/ directory.

The additional_build_steps then, at the ‘prepend_base’ phase, copy our new pip.conf into the default location (/etc/pip.conf) on the base image. All pip operations performed in the build will then use our new custom pip.conf.

### Additional RPM Libraries

When our execution-environment.yaml refers to additional binary dependencies, those dependencies are retrieved using the configured package manager (usually for minimal EEs, this is microdnf, but we can configure any package manager in our execution-environment.yml file). But what remote repos is the builder image using to locate those binary rpms?

As it turns out, the Red Hat base EE images are in turn based on the Red Hat Enterprise Linux Universal Base Image (UBI). The UBI has a single repository config file:

{% highlight ini %}
{% raw %}
/etc/yum.conf/ubi.repo
{% endraw %}
{% endhighlight %}

ubi.repo refers to an Internet-facing repository that can be accessed without needing the client system to be registered with Red Hat. In a disconnected environment, we may have a separate mirrored rpm repo, but how can we tell ansible-builder to use it instead of ubi.repo? Well, we can use a similar process to configuring a custom Python repo. This time, we create a custom repository definition (or copy one from elsewhere):

{% highlight ini %}
{% raw %}
[myrepo]
name=My Repo
baseurl=http://myrepo.example/$basearch/
enabled=1
gpgcheck=0
{% endraw %}
{% endhighlight %}

And then we customise our execution-environment.yml accordingly. This time, though, we also need to remove the default ubi.repo config file so that our custom repo is used:

{% highlight yaml %}
{% raw %}
additional_build_files:
  - src: files/my.repo
	dest: configs

additional_build_steps:
  prepend_base:
	- RUN rm /etc/yum.repos.d/ubi.repo
	- COPY _build/configs/my.repo /etc/yum.repos.d/my.repo
{% endraw %}
{% endhighlight %}

Note that we can just remove the ubi.repo, and then the microdnf calls will attempt to use the configured repos of the ansible-builder host machine. This approach is a bit less maintainable than explicitly injecting the repo definition you want the builder to use.

## Conclusion

Execution Environments provide a great way to isolate and include only the specific tools and libraries needed for automation tasks. However, building new custom execution environments in a disconnected environment can sometimes be tricky to manage. By examining the ansible-builder build process, and using the fine-grained build step configuration of ansible-builder v3, we showed how we can ensure that EEs can be built without external network access.



## References

A sample execution-environment.yml can be found in this repo:

[disconnected-ee.git](https://github.com/derekwaters/disconnected-ee)

[A guide to automation execution environments](https://www.ansible.com/blog/the-anatomy-of-automation-execution-environments)

[Ansible-builder in an offline environment - needs a Red Hat account](https://access.redhat.com/solutions/6955119)

A blog that covers a lot of this material in the ansible-builder v1 world:
[ansible-builder in a disconnected environment](https://cloudautomation.pharriso.co.uk/post/ansible-builder-disconnected/)


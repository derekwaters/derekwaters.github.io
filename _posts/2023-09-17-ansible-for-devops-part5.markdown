---
layout: post
title:  "Ansible for DevOps - Part 5"
date:   2023-09-17 22:00:00 +1000
categories: ansible devops
---
## Future Enhancements

Although this repo contains a couple of roles (one to deploy AAP and EDA, and one to deploy the demo setup), it’s a long way from best-practice Ansible! The code should be refactored into more fine-grained roles, potentially with switches to turn on or off the installation of EDA.

EDA itself doesn’t have a fully production-ready collection for automation yet. This means that, for example, the AAP token generated to allow EDA to connect to the AAP Controller can’t be automatically added to EDA.

I also need to work on detecting the availability of AAP to apply licensing. This sometimes works, sometimes requires rerunning the playbook after a pause.

At the moment, I haven’t automated the configuration of the webhook in github - this has to be done manually. However, the community.general.github_webhook module would allow this process to be automated too.

A further refinement may be to use the details provided in the webhook to make the workflow process succeed or fail rather than using a variable defined on the workflow.


## Resources

A copy of the slides used for the demo presentation are here:

TODO

A video of the demo can be seen here:

TODO

The github repo containing the automation to set up the demo environment can be found [here](https://github.com/derekwaters/ansible_devops_demo)

The github repo containing the “app team” job templates can be found [here](https://github.com/derekwaters/ansible_devops_demo_appteam)


## Thanks!

Thanks for reading. All the posts are collected here:

[The Use Case and Plan]({% link _posts/2023-09-11-ansible-for-devops-part1.markdown %})
[Deploy AAP and EDA and License]({% link _posts/2023-09-14-ansible-for-devops-part2.markdown %})
[Automation Resources, Git and Webhooks]({% link _posts/2023-09-15-ansible-for-devops-part3.markdown %})
[RBACs and ServiceNow Integration]({% link _posts/2023-09-16-ansible-for-devops-part4.markdown %})
[Enhancements and Resources]({% link _posts/2023-09-17-ansible-for-devops-part5.markdown %})


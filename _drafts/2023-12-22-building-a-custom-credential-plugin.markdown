---
layout: post
title:  "Building a Custom Credential Plugin"
date:   2023-12-22 08:00:00 +1000
categories: ansible execution environments credentials aws sts assume role
---
# Building a Custom Credential Plugin

Sometimes the Credential Types in Ansible Automation Platform (AAP) don't quite do what you need. In some cases, you can define a custom Credential Type in AAP to map some credential inputs and inject them into your playbooks as environment variables or extra vars. But sometimes that's not enough and you need to do something more custom.

One such case came up recently. An AAP user was trying to use the amazon.aws collection to manage some cloud infrastructure. But to improve the security posture of their deployment they wanted to use Amazon Web Services (AWS) Security Token Service (STS) to assume a temporary Identity and Access Management (IAM) role to actually perform the automation. Out of the box, the AWS credential type in AAP doesn't support this. 

Fortunately AAP allows you to build custom credentials lookup plugins to do things like this. 

## Custom Credential Plugins

A sample Python code template is available in [this git repository](https://github.com/ansible/awx-custom-credential-plugin-example) showing the general structure a custom credential plugin needs to take:

The plugin code itself is pretty simple. You define the credential inputs in a data structure. These are the fixed pieces of data that will be saved in the AAP database when someone creates an instance of your custom credential. You can define these inputs as “password” type to make them protected in the AAP console. You also define the credential’s “metadata”. These are dynamic values provided to your credential code when AAP actually tries to retrieve credentials, and will be defined in another (built-in) credentials instance. Finally, you define a ‘backend’ function, which is what will be called with all the inputs and metadata needed to actually retrieve a credential value.

Now if you do all this in AAP you might get confused as to why you can create an instance of your new credential type, but when you create or edit a template, and try to select your newly created credential, it's not displayed? This is because the custom plugins need to be an *indirect* credential lookup. A built-in credential instance will in turn use your custom credential instance to look up certain values it needs. For example, you might have a built-in Machine credential that uses your custom plugin to look up its password value from some other secret storage system.

This whole process is a little confusing, but the diagram below explains what's going on:

![ServiceNow Change Request](/img/blog6.png)

DIAGRAM GOES HERE

When AAP tries to populate the Machine Credential when a job is run, it determines that the username field is a lookup, so it accesses the Custom Credential, passing it the ‘identifier = username’ metadata. This is combined with the inputs on the Custom Credential - in this case, just a token value. The Custom Credential backend function then accesses the external secret store, passing the token (from the inputs). The username and password are returned from the external secret store. The Custom Credential then checks the identifier metadata to see what value is being requested, and hence returns the username value, which is then used by the Machine Credential. A similar process occurs when populating the password, but because the identifier is different, the password returned from the secret store is what is returned from the backend function call.

Note that it’s not necessary for all of the values in the Machine Credential to be lookups, so username could be saved with the Machine Credential, but the password is looked up via the Custom Credential. Similarly, depending on the external secret store, the identifier may be passed to that system to only return the single value requested.

## AWS Assume Role Custom Credential

For the AWS assumed role use case, we configure a standard AWS Credential. But rather than specifying an AWS Access Key ID, Secret Key and Security token in the credential, all three values are retrieved by a custom credential based on values returned from a call to AWS’ STS assume_role API.

The AWS Credential looks like this in the AAP console:

IMAGE HERE

The custom STS role credential looks like this in the AAP console (the Access Key ID and the Secret Key belong to the user assuming the IAM role, and the ARN is a tag identifying the role the user will assume):

IMAGE HERE

Templates can then be configured to use the AWS Credential:

SCREENSHOT HERE

## Installing A Plugin

The first difficulty once you've cloned the plugin repo and made your changes is working out how to install your plugin in AAP. The repo does have some instructions. One trick I have discovered is that if you have a local copy of your plugin repo you can run the awx-python command with the local repo rather than referencing a remote git repo:

{% highlight bash %}
awx-python -m pip install .
{% endhighlight %}

To restart the AAP services, you can run this command:

{% highlight bash %}
automation-controller-service restart
{% endhighlight %}

So the entire set of commands to install your plugin from a local folder is:

{% highlight bash %}
awx-python -m pip install .
awx-manage setup_managed_credential_types
automation-controller-service restart
{% endhighlight %}

Note that these steps only work in a traditional AAP install on VMs. My next TODO: is to work out how this is done in a containerised AAP installation where the controller container images are immutable.

## Debugging

Testing custom credential plugins is a little painful and it feels like there should be a better method. Within AAP, the only output you see of a job using your custom credentials is the actual playbook output, none of the output from your custom Python code is visible. Using the built-in AAP logger class doesn’t seem to produce any actual logging output.

Sadly, I resorted to manually opening and writing to a local file on the controller VM. Again this obviously won't work in a containerised AAP system.

Note that my lab environment doesn't have AAP configured to send logs to an external logging system, and it’s possible that this might have made logger logs visible.

## Some Lessons Learnt

### Build an AWS EE

Because the testing playbook actually makes calls to AWS, that means it needs the boto3 and botocore Python libraries to interact with the AWS Application Programming Interface (API). These aren’t in the default AAP execution environments (EE), so I needed to build a new EE including those Python libraries, and the amazon.aws Ansible collection. I’ve included the execution-environment.yml file I used to build the EE in my example git repo (below).

### Caching Results

If you follow the diagram showing how the credentials fields are retrieved from the external secrets system above, you can see that it makes one call to the external secrets system for each external field in the credential. If the values in the external secrets system are consistent, this is fine (other than the wasted effort of making the same API calls multiple times).

However, for the AWS assume_role API, every call returns a new set of AccessKeyId, SecretKey and SecurityToken. So retrieving the AccessKeyId makes a call to assume_role, then retrieving the SecretKey makes another call, which returns a whole new AccessKeyId alongside the SecretKey and SecurityToken. This means that after the three assume_role calls, when you actually try to access AWS, it fails because you’ve got invalid AccessKeyId and SecretKey values from previous calls to assume_role:

DIAGRAM

This means that for the AWS role custom credential we need to store a cache of the resultant credentials so subsequent calls to the backend function can use the cached values. The cache is keyed by a combination of the access key id and the arn (so a user accessing multiple roles will have different cached credentials, and multiple users accessing the same role will also use different credentials). The credentials returned from AWS STS also include an Expiration date, which is also checked to invalidate the cache.

### DateTime Comparison

This led to another interesting bit of Python. The Expiration datetime value returned from AWS STS includes a timezone, but the default Python datetime.datetime.now() function does not, which results in a “comparing naive and aware datetime objects” error. The ‘naive’ datetime value can be converted to a timezone aware one by making the comparison with code like the following:

{% highlight python %}
if (expiration < datetime.datetime.now(expiration.tzinfo)):
{% endhighlight %}

## The Results

In testing, we set up an AWS IAM user with no permissions:

IMAGE

We then set up an IAM role, with a policy allowing the role to be assumed by the IAM user, and with permission to read S3 buckets in the AWS account.

IMAGE

In AAP, we have a test playbook, which simply tries to use the amazon.aws.s3_bucket_info role to get information about S3 buckets:

{% highlight yaml %}
---
- name: Access S3 bucket list in AWS using roles
  hosts: all
  collections:
    - amazon.aws
  
  tasks:
    - name: Testing that the playbook starts
      ansible.builtin.debug:
        msg: "Testing AWS STS assume_role credentials"
        
    - name: Get S3 bucket info with env vars
      amazon.aws.s3_bucket_info:
        access_key: "{{ lookup('env', 'AWS_ACCESS_KEY_ID') }}"
        secret_key: "{{ lookup('env', 'AWS_SECRET_ACCESS_KEY') }}"
        session_token: "{{ lookup('env', 'AWS_SECURITY_TOKEN') }}"
      register: result_with_envvars

    - name: Dump S3 bucket info with env vars
      ansible.builtin.debug:
        var: result_with_envvars

    - name: Get S3 bucket info native
      amazon.aws.s3_bucket_info:
      register: result

    - name: Dump bucket info native
      ansible.builtin.debug:
        var: result
{% endhighlight %}

We create a test Amazon credential with the access key and secret details for the IAM user.

IMAGE

We then create a job running our test playbook with the test credential.

IMAGE

When this job is run, the output indicates that the s3_bucket_info play fails with an AccessDenied error:

IMAGE

Instead, we create a custom AWS Role Credential with our plugin. We use the access key and secret key for our test IAM user, and the ARN for the test IAM role:

IMAGE

Then we create a builtin AWS credential, with each of the three fields using a lookup via our custom credential, and with metadata identifiers of AccessKeyId, SecretAccessKey and SessionToken:

IMAGE

When this AWS credential is attached to a job running our test playbook, the playbook run successfully lets our test user assume our test IAM role, and runs the playbook in the scope of the assumed credentials. The output now shows details about our S3 buckets, as expected:

## Resources

All of the code for this plugin (including the EE build definition and the test playbook) is available on [Github](https://github.com/derekwaters/aws_role_credential_plugin)

More info about AWS STS, role assumption and accessing the AWS API can be found here:

- [Using boto3 to assume an IAM role](https://www.learnaws.org/2022/09/30/aws-boto3-assume-role/)
- [AWS Assume Role API](https://docs.aws.amazon.com/cli/latest/reference/sts/assume-role.html#output)
- [AWS CLI env vars](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-envvars.html)

The template code for building credential plugins can be found on [Github](https://github.com/ansible/awx-custom-credential-plugin-example)

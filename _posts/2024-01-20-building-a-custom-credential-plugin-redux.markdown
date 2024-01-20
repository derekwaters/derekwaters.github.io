---
layout: post
title:  "Building a Custom Credential Plugin (Redux)"
date:   2024-01-20 08:00:00 +1000
categories: ansible execution environments credentials aws sts assume role
---
# Recap

In the [previous]({% link _posts/2023-12-22-building-a-custom-credential-plugin.markdown %}) article, I talked about creating a custom credential plugin that took in a user's AWS access and secret keys, and used them to assume an IAM role to perform automation.

However, saving the access and secret keys in a Credential in AAP isn't the most secure way to do this, so now we'll try and tighten things up. This involves two things: using the AAP environmental configuration to authenticate to the AWS API initially, and providing a specific External ID to the assume_role call.

# Authenticating to the AWS API

Previously we authenticated to the API using an AWS access key and secret key that were stored in an AAP credential. Potentially that means someone with access to the credential could apply it to some other job doing something against the AWS API that we're not expecting.

So instead, we'll make the access key and secret key optional fields on the custom credential. If they're both unset, we instantiate a boto3 Python connection with no keys. Boto3 then looks in a series of alternate places for credentials to access AWS. You can find the full list in [this doc](
https://boto3.amazonaws.com/v1/documentation/api/latest/guide/credentials.html).

In my testing lab, I did this by configuring an AWS credentials definition file for the user account that AAP runs as. This basically meant creating a file at:

{% highlight bash %}
/var/lib/awx/.aws/credentials
{% endhighlight %}

/var/lib/awx is the awx user home directory.

It's important to note that the calls to AWS to build the credential are made _before_ the execution environment is created, so this configuration must be done on the controller nodes, not in the execution environments.

# Changing the Custom Credential

Once we're configured to fallback to environmental AWS credentials, we can amend our Custom Credential plugin accordingly. Firstly, we remove access_key and secret_key from the list of required credential fields, so that they're optional.

Then, when we instantiate the boto connection, we check to see if both the secret and access keys are unset. If so, we instantiate the boto connection without credentials:

{% highlight python %}
if (access_key is None or len(access_key) == 0) and (
    secret_key is None or len(secret_key) == 0):
    # Connect using credentials in the EE
    connection = boto3.client(
        service_name="sts"
    )
else:
    # Connect to AWS using provided credentials
    connection = boto3.client(
        service_name="sts",
        aws_access_key_id=access_key,
        aws_secret_access_key=secret_key
    )
{% endhighlight %}

Note that I found that one of my keys was passed to my function as an empty string rather than None. I suspect this might be something to do with one of the fields being a "password" type, I'm not entirely sure. To account for that, I check to see if both the access and secret keys are either None or length 0 strings.

# External ID

AWS allows you to provide a general "External ID" to the assume_role API call. This External ID can then be used as a parameter to further restrict the ability to assume a role. Details on how to configure AWS to require a specific External ID can be found [here](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-user_externalid.html).

Adding the external ID to our custom credential is trivial. We add a new field to the definition of the credential fields. When we call assume_role, we add the new External ID value:

{% highlight python %}
response = connection.assume_role(
    RoleArn=role_arn,
    RoleSessionName='AAP_AWS_Role_Session1',
    ExternalId=external_id
)
{% endhighlight %}

# Putting It Together

We are now able to configure a custom AWS Role Credential, with no access or secret keys defined, but with an External ID (which can be any arbitrary string value):

![AWS Role Credential with External ID](/img/customcredential2/awscred.png)

We now use this credential as the lookup for the access key, secret key and security token for a builtin AWS Credential. Running our same test playbook to list S3 buckets using our new assumed role shows that we have assumed the IAM role that grants us access temporarily:

![Playbook Run with Successfule Role Assumption](/img/customcredential2/jobsuccess.png)

As before, all of the code for this plugin (including the EE build definition and the test playbook) is available on [Github](https://github.com/derekwaters/aws_role_credential_plugin)

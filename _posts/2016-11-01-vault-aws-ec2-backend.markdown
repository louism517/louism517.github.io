---
title:  "Vault and the AWS EC2 Auth Backend"
date:   2016-11-01 10:00:20 +0000
---

In version 0.6.0 of Vault, Hashicorp introduced an AWS EC2 authentication backend. The [docs][vault-docs] provide a wealth of information, but it took me a few readings to discern the forest for the trees. Once I got past sniggering at the liberal use of the word _nonce_ (which has an altogether [different meaning][saville] in my corner of the world) I came to realise what a useful solution it can provide.

The AWS EC2 backend provides the ability to allow authentication only from EC2 instances which meet certain criteria. For example, you can specify only instances that exist in a given account, or that exhibit a specific role. It is possible to then further restrict authorisation according to EC2 tags associated with the instance.

### Authentication

When an EC2 instance is first launched, an [instance identity document][aws-docs] is generated. This is simply a JSON document that can be retrieved from an endpoint within the metadata service: `http://169.254.169.254/latest/dynamic/instance-identity/document`

It is this document which is used to prove an instance’s identity when authenticating against Vault. Clearly it could be easily spoofed, so AWS provide a PKCS7-signed version of it, which can be verified using their public certificates. It is this version (or alternatively a Base64 encoded document along with a SHA256 signature) which is sent to Vault.

A second piece of information is taken into account when granting authorisation to a requesting instance, and that is the client nonce. The need for this secondary information arises from the fact that access to the identity document is by default granted to any user or process running on an EC2 instance (this can be improved by firewalling the metadata service), and therefore anybody with access to the instance would be able to authenticate.

During the first authentication request by an EC2 instance, the nonce is either provided by the client or auto-generated and returned by Vault. Either way, this nonce _must_ be provided with all subsequent requests for authentication (or until an operator manually intervenes). If authentication is successful a token is returned and it is that which you can use to interact with Vault.

This presents some interesting design decisions. You could:

- discard the nonce altogether after the first use, and accept that no more authentication requests can succeed from the instance.
- keep the nonce in memory in whatever process does the initial requesting, and accept that once that process dies, authentication will be no longer possible.
- store the nonce in a location on disk and accept that anybody with access to the box will be able to authenticate (much like IAM instance profiles).

Coupled with restrictive policies and variable token TTLs, there are a range of solutions you could choose. It depends on the use-case and the sensitivity of the data made available to the instance.

### Authorisation

Vault ‘login’ happens against a defined ‘role’, which has one or more policies attached to it. Access to the role is restricted to EC2 instances that meet a set of requirements.

Vault is able to ascertain a certain amount of information from the decrypted identity document described above and augments this with information it gathers using the AWS API. Using this information, and after performing a perfunctory check that the EC2 instance is actually running, it decides whether the instance should be granted access (i.e. a token) to the role.

At the time of writing there are 4 requirements which can be met:

- the instance must belong to a given AWS account
- the instance must be built from a given AMI (Amazon Machine Image)
- the instance must exhibit a given IAM role
- the instance must exhibit a given IAM instance profile

A combination, or none, of the above can be set.

Further restrictions can be applied through the application of a role_tag. This takes the form of a tag applied to the EC2 instance. An important point to note is that the existence of a role_tag can only _narrow_ the scope of permissions that are allowed, they cannot be used to grant additional privileges.

### Example

With the theory out of the way, let’s run through an example.

We will create a Vault role called `my-role`, with a set of policies attached, to which we’ll allow access to instances running in a specific AWS account. Then we’ll use a role_tag to further restrict the policies available.

Firstly, it needs to be enabled:

```
$ vault auth-enable aws-ec2
```

Secondly, the Vault server itself must be given access to the AWS API in your account, so that it can call in and check instance attributes. If you skip this step then Vault will simply obtain credentials using the default credential chain on the running host:

```
$ vault write auth/aws-ec2/config/client secret_key=vCtSM8ZUEQ3mOFVlYPBQkf2scxxxxXXXX access_key=XXXXXXXXXXX
```

Next, we’ll create `my-role` and associate it with 2 policies: production and dev. Here is where we specify any access restrictions too, in this case we'll bind to the AWS account 1234567890.

```
$ vault write auth/aws-ec2/role/my-role bound_account_id=1234567890 policies=prod,dev 
```

So, at this point, any EC2 instance in the AWS account 1234567890 can request a token that will allow it access to `my-role`, and by extension the production and dev policies.

If we decide (as well you might) that we don’t actually want every instance to have access to the production policy, we can enable role_tags. With role_tags enabled a subset of the policies are awarded to instances that exhibit specific roles, and refused to those that don’t.

The role_tag must be generated by Vault itself. The following command requests a role_tag for `my-role`, which allows the bearer access to the dev policy only:

```
$ vault write /auth/aws-ec2/role/my-role/tag policies=dev
```

It returns something like this:

```
Key          Value
---          -----
tag_key      VaultAccess
tag_value    v1:yarQCqNjtvM=:r=my-role:d=false:m=false:p=default,dev:lzyYRFMPP4lORoHJrvUFl2XrxTDK3vECo9uf7pqEBMw=
```

The _tag_key_ and _tag_value_ are what must be set as the EC2 tag’s key and value respectively. The value itself is an HMAC signed string of which Vault is able to verify authenticity. 

This auth policy for `my-role` then needs to be amended to enable role_tags:

```
$ vault write auth/aws-ec2/role/my-role bound_account_id=1234567890 policies=prod,dev role_tag=VaultAccess
```

Finally, with all this in place, we can now attempt a login to the `my-role` role _only if_ we are on an instance in the AWS account with ID 1234567890 _and_ the EC2 tag ‘VaultAccess’ with the value described above.

```
$ vault write /auth/aws-ec2/login role=my-role \
pkcs7=$(curl -s http://169.254.169.254/latest/dynamic/instance-identity/pkcs7)
nonce=21daf536-99fc-442e-459d-c76c375d444d
```

As mentioned above, the nonce here is optional. If not provided then Vault will return a nonce which must be used for all subsequent authentication attempts from the instance (each nonce is associated with a single instance). If all is well you should be returned a token which you can then use to login, with access to the dev policy only.

There’s a great deal more to the AWS EC2 backend (see the [docs][vault-docs]), but hopefully this provides a good introduction.

[vault-docs]: https://www.vaultproject.io/docs/auth/aws-ec2.html
[aws-docs]: http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-identity-documents.html
[saville]: https://en.wikipedia.org/wiki/Jimmy_Savile

---
title: "AWS IAM Not*"
date: 2021-01-02T20:18:07+06:00
featureImage: https://miro.medium.com/max/4800/1*Ks7GuiPPa2pswNTuITHGSw.png
description: "Sometimes writing AWS IAM policies gets confusing. Especially if our policy authoring is reactive in nature instead of following a proactive permissions strategy."
author: Johannes Verjwijnen
authorThumb: images/team/johannes.jpeg
button_text: Original Source
Button_url: https://blog.deepdives.eu/aws-iam-not-3cabe7887531
---

Sometimes writing [AWS IAM](https://aws.amazon.com/iam/) policies gets confusing. Especially if our policy authoring is reactive in nature instead of following a proactive permissions strategy.

![Image courtesy Amazon Web Services [IAM User Guide](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html)](https://miro.medium.com/max/4800/1*Ks7GuiPPa2pswNTuITHGSw.png#center)

Wow — this is complicated! Additionally some of the documentation and courseware suggest doing both allow and deny-statements.

What how?

Well the [Amazon S3: Limits managing to a specific S3 Bucket](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_examples_s3_deny-except-bucket.html) example in the IAM User Guide shows the idea.

{{< highlight go >}}
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::bucket-name",
                "arn:aws:s3:::bucket-name/*"
            ]
        },
        {
            "Effect": "Deny",
            "NotAction": "s3:*",
            "NotResource": [
                "arn:aws:s3:::bucket-name",
                "arn:aws:s3:::bucket-name/*"
            ]
        }
    ]
}
{{< /highlight >}}

So here we’re allowing all S3 access on to a particular bucket and its contents, and then denying all other actions (for all other services!) and for all other resources.

So how do the NotAction and NotResource -elements work? If we take a look at the policy evaluation order we can see that the second step is “Evaluate all applicable policies”. Thus if a request comes in from a principal that has the above policy attached for reading an item from a DynamoDB table the above policy will be evaluated into a deny.

* The first part of the policy allows S3-actions, so that would be skipped as not matching the request.
* For the second part any DynamoDB-action will match the “NotAction”: “s3:*”-part thus a deny decision is made.
* As a deny overrides any possible allows we will deny the request.

But wait a minute, wasn’t the policy evaluation logic deny by default anyway? Yes it is, and this kind of policy is only needed if you are feeling that you might (accidentally) give someone permissions they shouldn’t have.

There are more sane ways to use NotAction with Allow to, for example, allow full control except deletion.

{{< highlight go >}}
"Effect": "Allow",
"NotAction": "s3:DeleteBucket",
"Resource": "arn:aws:s3:::*",
{{< /highlight >}}

This is way easier than having to explicitly list all S3-actions. But doesn’t that allow all other services? Would a DynamoDB table read be allowed with this kind of policy? Well no, as the resource in that request would be a DynamoDB-table instead of a S3-bucket.

Similarly we can restrict access to a particular resource by using the Deny-NotResource -pattern.

{{< highlight go >}}
"Effect": "Deny",
"Action": "s3:*",
"NotResource": [
    "arn:aws:s3:::OurBucket",
    "arn:aws:s3:::OurBucket/*"
]
{{< /highlight >}}

So here we are denying all S3-actions to all resources except one bucket and its contents. Note that this alone will not give access to that particular resource as an explicit allow is required somewhere else.

Combining Allow and NotResource becomes terribly frightening. Think of an innocent-looking policy like this:

{{< highlight go >}}
"Effect": "Allow",
"Action": "*",
"NotResource": [
    "arn:aws:s3:::SecretBucket",
    "arn:aws:s3:::SecretBucket/*"
]
{{< /highlight >}}

So we’re probably trying to give full access to S3 except for a particular bucket and its contents here. However, we have just given all possible permissions (except access to that bucket) using this policy. And this doesn’t restrict anything as we also gave the permission to change policies themselves :D

The motivation for this blogpost came from a discussion on what NotAction: * or NotResource: * would do. There is a common misconception that they would invert the Effect which naturally isn’t true.

Remember the way that the evaluation logic works. It would first evaluate all applicable policies, thus any request with any action wouldn’t match the NotAction: *-pattern and any request with any resource wouldn’t match the NotResource: *-pattern. Thus they are practically no-ops.

I tested it (for completeness). Created a user with the inline policy:

{{< highlight go >}}
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Deny",
            "Action": "s3:*",
            "NotResource": "*"
        }
    ]
}
{{< /highlight >}}

And then tried to list buckets

{{< highlight go >}}
duvin@iMac ~ % aws s3 ls  --profile notresource
An error occurred (AccessDenied) when calling the ListBuckets operation: Access Denied
{{< /highlight >}}

As anticipated. If we add an Allow-permission that allow works normally as if the deny wouldn’t exist.
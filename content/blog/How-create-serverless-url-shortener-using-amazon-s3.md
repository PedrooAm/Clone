---
title: "How to create a serverless URL shortener using Amazon S3"
date: 2020-01-06T20:18:07+06:00
featureImage: https://miro.medium.com/max/1400/0*zxY_8kAkQKBnO2FO
author: Johannes Verjwijnen
authorThumb: images/team/johannes.jpeg

---

Let’s admit it. We access more and more of the information we need in our daily activities using URLs. These URLs have become longer and more cryptic over time but that’s ok since we simply click on them.

![Photo by Steve Johnson on Unsplash[Unsplash](https://unsplash.com)](https://miro.medium.com/max/1400/0*zxY_8kAkQKBnO2FO#center)

However, sometimes we need to communicate those URLs in a non-internset-native way. For me this has happened during training classes as well as when writing a book. I want the class to download a piece of source code, but the GitHub project file URL is way too long to be correctly retyped by the attendees. Or maybe I want to refer to an online post in the book but printing a very long URL doesn’t make sense.

That’s why we’ve used URL shorteners for quite a while now. Unfortunately, they have their downsides — they may be funded through advertisement sales or offer a freemium version of the product. The features you actually need are diverse; your own domain name, the possibility to customise the URL (instead of being assigned a random string), some come to a surprise to you later on (like the possibility to redirect a shortened URL as the original target has moved).

So why not create your own URL shortener? Now note that I mostly dabble with serverless managed services on AWS for a living so I’m immediately thinking of the following combination:

- API Gateway to handle the requests — just creating a proxy integration to Lambda should suffice.
- Lambda to lookup the redirect URL and return the actual HTTP 301 response redirecting the browser.
- DynamoDB to work as storage for the Lambda function.

I could even run the code using Lambda@Edge and then cache the DynamoDB responses for even better performance!

But… even though that would make a great walkthrough on how Lambda and API Gateway work, it seems a bit overkill for just a URL shortener. Is there anything else?

Did you know that S3 can redirect requests? Of course, you did, at least if you’ve used the static website hosting feature of S3 — the option is right there in configuration screen. You might be surprised that instead of redirecting requests to another object in S3 — or another bucket — you can actually enter any other URL as the redirect target. Additionally the object containing the redirection (the short URL) can be of size 0 bytes (meaning you only pay for storage of the metadata).

So how would we create a shortener like this? My example is going to be without any fancy admin UIs etc, we’re going to create and administer the redirects via the AWS CLI only.

First of course you need to create a bucket. You should name your bucket after the website you’re hosting, so in my case that would be links.verwijnen.fi.

aws s3 mb s3://links.verwijnen.fi
Then we need to enable static website hosting, making the default document ‘default’ and the error document ‘error’.

{{< highlight go >}}
aws s3 website s3://links.verwijnen.fi --index-document default --error-document error
{{< /highlight >}}

We also need to set proper permissions to the content of the bucket. There are several ways to do this, and my favorite is to create a bucket policy allowing everyone GetObject access to the bucket. First we create the policy document (I use the AWS Policy Generator for this as I’m hopeless when trying to remember syntaxes), I’m saving it to policy.json:

{{< highlight go >}}
{
  "Id": "Policy1578299737219",
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Public access to URL shortener bucket",
      "Action": [
        "s3:GetObject"
      ],
      "Effect": "Allow",
      "Resource": "arn:aws:s3:::links.verwijnen.fi/*",
      "Principal": "*"
    }
  ]
}
{{< /highlight >}}

##### Then we set that JSON file as the bucket policy:

{{< highlight go >}}
aws s3api put-bucket-policy --bucket links.verwijnen.fi --policy file://policy.json
{{< /highlight >}}

Then we create a DNS record so that requests to our hostname links.verwijnen.fi get served by the hosted bucket hostname links.
verwijnen.fi.s3-website.eu-north-1.amazonaws.com using Route 53. This is easiest to do via the web console, where we can selected the Hosted Zone we want to edit and then click on “Create Record Set”, edit the name as “links”, leave the type as A and just set Alias to “Yes”. We can then see our bucket in the target options. The routing policy should stay as simple and target health evaluation as “No”.

Now we can create a few redirects, namely the two from above.

{{< highlight go >}}
aws s3api put-object --bucket links.verwijnen.fi --website-redirect-location http://verwijnen.fi/ --key default
aws s3api put-object --bucket links.verwijnen.fi --website-redirect-location http://verwijnen.fi/404.png --key error
{{< /highlight >}}

So now we have the default document redirecting to my website and the error document redirecting to a cute 404 image. 

This can be tested by going to http://links.verwijnen.fi/ and http://links.verwijnen.fi/somethingerroneous.

#### Let’s also create a redirect to this blogpost

{{< highlight go >}}
aws s3api put-object --bucket links.verwijnen.fi --website-redirect-location https://medium.com/@verwijnen/serverless-url-shortener-with-amazon-s3-ffa38ea2a9d6 --key s3url
{{< /highlight >}}

and now it should be available at [http://links.verwijnen.fi/s3url!](http://links.verwijnen.fi/s3url!)]


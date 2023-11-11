---
layout: post
title:  "Go Lambda provided.al2 runtime with multiple binaries"
date:   2023-11-10 10:00:20 +0000
image:  '/images/lambda-logo.png'
description: "How to migrate to the provided.al2 runtime, whilst retaining support for multiple binaries."
tags: [Serverless, AWS]
---

On December 1st 2023 AWS are deprecating the trusty go1.x Lambda runtime.

Don't worry, [runs the messaging](https://aws.amazon.com/blogs/compute/migrating-aws-lambda-functions-from-the-go1-x-runtime-to-the-custom-runtime-on-amazon-linux-2/), just 
upgrade to the new provided.al2 runtime and all shall be well. 
Not only that, you can enjoy Graviton support, and the removal of a superfluous API call in the bargain.

Sounds good, right? Well I'm sure it _is_, but there is a group of people who are out of sorts - those of us who 
package multiple binaries into a single Lambda zip package. 

Now, to be sure, this is not the _proper_ way to Lambda: single-binary, single-package per function has been the dogma for 
some time (although finally some people [are admitting to using Lambdaliths](https://rehanvdm.com/blog/should-you-use-a-lambda-monolith-lambdalith-for-the-api)). 
But it has been _a_ way to Lambda, for some time. There are many serverless use-cases for which latency and package size
are not a concern, where deploying multiple binaries is pragmatic. At [work](https://spaceapegames.com/), we take such an approach.

Regardless of the merits of the approach, it is a fact that the provided.al2 runtime no longer supports it. If you've
found this article then maybe you too are staring down the barrel of splitting hundreds of Lambda functions into multiple
packages. If so, read on, it may save you some effort...

### The Problem

So what even _is_ the problem with multi-binary packages on `provided.al2`? 
The issue is that the runtime requires a single executable, which _must_ be called `bootstrap`. If you need to support 
multiple code-paths, then they must be somehow routed from that entrypoint.

You have a couple of options in dealing with this:

1. Handle the routing of requests to handlers within the `bootstrap` binary. You could do this based on the 
AWS_LAMBDA_FUNCTION_NAME default environment variable, something like:

    ```go
    func main() {
        functionName := os.Getenv("AWS_LAMBDA_FUNCTION_NAME")
        switch functionName {
        case "FirstFunction":
            lambda.Start(firstFunction)
        case "SecondFunction":
            lambda.Start(secondFunction)
        }
        ...
    }
    ```
2. Move to container-based Lambdas. You can have as many binaries as you wish in a container image. This is probably the cleanest way to retain support for multiple binaries, but comes with its own trade-offs.

The downside of both of these approaches is that they require _effort_. Changes will likely be needed to SAM templates, 
to build-chains and to the code itself.

Well good news - there _is_ an easier way.

### A Solution

The solution is to use a _shim_. A shim is a small piece of code that is used to bootstrap a larger piece of code.

The trick here is that the bootstrap binary doesn't have to be a Go program, it just needs to be executable. Which means 
we can drop in a simple shell script - called `bootstrap` - that will route requests to the correct binary:

```bash
#!/bin/bash
exec "${PWD}/${_HANDLER}"
```

Conveniently, the _HANDLER variable will be set to the path of the handler. Which should be exactly the same as was used 
in the go1.x runtime! This is best illustrated by way of an example.

In this current (`go1.x`) snippet, the function handler is located within the `.zip` package at `bin/handle_it`:

```yaml
MyLambdaFunction:
    Type: 'AWS::Serverless::Function'
    Properties:
       Runtime: go1.x
       Handler: bin/handle_it
```

With the shim in place (in the root of the package), we can simply switch one runtime for the other:

```yaml
MyLambdaFunction:
    Type: 'AWS::Serverless::Function'
    Properties:
       Runtime: provided.al2
       Handler: bin/handle_it
```

The `_HANDLER` variable is set to `bin/handle_it` and the shim will `exec` the binary just as before. No other changes
required!

However, it would be myopic to leave it there. As mentioned at the top of this article, there _are_ some advantages in
moving to the `provided.al2` runtime. So let's take a look at how we can take advantage of them.

The `go1.x` runtime requires an additional RPC call to the Lambda Runtime API, on each function invocation. This is
explained in more detail [here](https://aws.amazon.com/blogs/compute/migrating-aws-lambda-functions-from-the-go1-x-runtime-to-the-custom-runtime-on-amazon-linux-2/).
The `provided.al2` runtime removes this requirement, which should result in a small performance improvement. But we can
also remove the RPC component from the packaged binary, which will lead to faster cold-start times. This is achieved by
adding the `lambda.norpc` build tag.

The second is that you should take this opportunity to switch to using Graviton processors for your Lambdas. They come in about 
a third cheaper than x86, _and_ look good on your carbon balance-sheet. To do this, just add `Architectures: [arm64]` to 
your SAM template, and export the environment variable `GOARCH=arm64` to wherever you build your code.

Putting this all together, your template should look something like:

```yaml
MyLambdaFunction:
    Type: 'AWS::Serverless::Function'
    Properties:
       Runtime: provided.al2
       Architectures: [arm64]
       Handler: bin/handle_it
```

And your build command:

```bash
$ GOARCH=arm64 go build -tags lambda.norpc -o bin/handle_it
```

### The Caveat

Of course, there is no such thing as a free lunch. The spawning of a shell process will add precious milliseconds to your
function invocation time. If you have a high-traffic application, or one which is sensitive to latency, then this may 
not be the best solution.

Thanks for reading, and good luck with your migration!
---
title:  "Using Handlers to Test the Golang AWS SDK"
date:   2016-08-09 10:00:20 +0000
---

In the Ruby world to which I am accustomed, testing is made easy for us. In an interpreted language, testing frameworks are able to peek and pry NSA-like into the comings-and-goings of the code at runtime, and Ruby in particular especially lends itself to this type of manipulation.

We forego these luxuries in a compiled language such as Go. My experience of learning to write tests in Go has been something like ambling through a minefield, slowly. On the way to something approaching proficiency I’ve come to appreciate, for instance, the importance of abstracting interfaces so as to make for easier mocking. 

But I’ve also decided that writing tests in Go can be fun. More so than Ruby, tests are _just code_. It is a relative cinch to spin up a fake HTTP server, for example, and sniff the requests that are made to it.

I wanted to find a similarly ‘fun’ way to test some code that uses the AWS SDK. There is surprisingly little information out there as to how to go about this but I found a smattering of comments about ‘handlers’ so decided to try and figure out how to bend them to my will. I’m not suggesting this is the best way to test code that uses the SDK, but it is definitely _a_ way.

Each request made by the AWS SDK goes through a series of stages, with each stage managed by a different set of `Handlers`, known as a `HandlerList`:

```go
type Handlers struct {
    Validate         HandlerList
    Build            HandlerList
    Sign             HandlerList
    Send             HandlerList
    ValidateResponse HandlerList
    Unmarshal        HandlerList
    UnmarshalMeta    HandlerList
    UnmarshalError   HandlerList
    Retry            HandlerList
    AfterRetry       HandlerList
}
```

We are free to manipulate these `HandlerList`s as we see fit, to decorate the request at any given stage of its lifecycle. The `HandlerList` type provides convenience functions to clear some or all of the Handlers in the list, or push an entirely new Handler function to the front or back of the list.

Furthermore, the SDK provides a `Session` type, which can be used to share similar configuration (i.e. [`aws.Config`][aws-config]) between otherwise unrelated service clients. For instance I could create an `s3` and an `ec2` service client using the same `Session` struct. Happily the `Session` struct also allows us to describe a set of Handlers:

```go
type Session struct {
    Config  *aws.Config
    Handlers request.Handlers
}
```

I used this particular technique recently in a simple tool for proxying requests to an AWS Elasticsearch service. The requests being proxied would need to be signed with valid AWS credentials, and there was a requirement to support assumed STS credentials. These STS credentials would need to be refreshed when they were no longer valid (i.e. had passed their expiry date), and it was this particular case which I wished to test.

The full test code can be found [here][aws-test]. Specifically, what it does is to intercept the call to refresh STS credentials (`AssumeRole`) and falsify a response that contains, amongst other things, an `Expiration` date which we can bend to our nefarious means.

The response must be a simulacrum of that which _would_ be sent by the AWS API, were our request ever be allowed to make it there. The structure of the response is easy to find, it is an [`sts.AssumeRoleOutput`][sts-assumeroleoutput]

```go
type AssumeRoleOutput struct {
    AssumedRoleUser *AssumedRoleUser `type:"structure"`
    Credentials *Credentials `type:"structure"`
    PackedPolicySize *int64 `type:"integer"`
}
```

Further perusal of the API docs will uncover the structures for `AssumedRoleUser` and `Credentials`. 

Armed with this information we can start to build a response:

```go
var dummyCreds = &sts.Credentials{
  AccessKeyId:     aws.String("IAMADUMMYIAMACCESSKEY"),
  SecretAccessKey: aws.String("MadeUpSecretAccessKey"),
  Expiration:      aws.Time(time.Now().Add(3 * time.Second)),
  SessionToken:    aws.String("MadeupToken"),
}
var dummyAssumedUser = &sts.AssumedRoleUser{
  Arn:           aws.String("made:up:arn:1233453337:something:"),
  AssumedRoleId: aws.String("madeUpRole"),
}
// Now build the AssumeRoleOuput
var dummyOutput = &sts.AssumeRoleOutput{
  AssumedRoleUser:  dummyAssumedUser,
  Credentials:      dummyCreds,
  PackedPolicySize: aws.Int64(99),
}
```

Notice how we set the `Expiration` of `dumyCreds` to 3 seconds in the future.

A Black Ops descent into the AWS SDK code led to a discovery (the twisted route to which I will not even pretend to remember) that the response to an `AssumeRole` request will be XML. So we just need to marshal our `dummyOutput` into XML and send that as a response, right? 

Almost. There are a couple of extra elements wrapping the response that are absolutely necessary, so we’ll create boilerplate structs for these elements and wrap the whole thing up as `dummyResponse`:

```go
type XMLResponse struct {
  AssumeRoleResponse *XMLResult
}

type XMLResult struct {
  AssumeRoleResult *sts.AssumeRoleOutput
}

var dummyResponse = &XMLResponse{AssumeRoleResponse: &XMLResult{AssumeRoleResult: dummyOutput}}
```

Next we need to fiddle the `HandlerList`s so that our `dummyResponse` is returned in place of an actual response:

```go
func TestStsCredentialGetter(t *testing.T) {
    sess := session.New()
    sess.Handlers.Send.Clear()
    sess.Handlers.Send.PushFront(func(r *request.Request) {
        arn := r.Params.(*sts.AssumeRoleInput).RoleArn

        if *arn != dummyConfig.Arn {
            t.Errorf("Assume role ARN expected: %s, got %s", dummyConfig.Arn, *arn)
        }

        var buf bytes.Buffer
        err := xmlutil.BuildXML(dummyResponse, xml.NewEncoder(&buf))
        if err != nil {
            panic(err)
        }
        r.HTTPResponse = &http.Response{
            StatusCode: 200,
            Body:       ioutil.NopCloser(bytes.NewReader(buf.Bytes())),
            // this for UnmarshalMetaHandler
            Header: http.Header{"X-Amzn-Requestid": []string{"12345254232"}},
        }

    })
```

The first thing we do is to create a new session object and promptly clear all of the handlers associated with the `Send` action (that is, when the request is actually sent). Then we use the `SendFront` function to push an anonymous function of our own to the front of the `Send` queue. It takes a pointer to the `request.Request` object on which it should act as an argument.

Within the function the first thing we do is to check that the correct `RoleArn` has been passed into the call to `sts.AssumeRole`. This is a somewhat pointless test case, but it highlights what can be done here. Then we move onto encoding the dummy response into XML, and wrap it up in an `ioutil.NopCloser` (to satisfy the `request.Body` interface). Note that we also forge an `X-Amzn-Requestid` header, this is just to satisfy one of the many other handlers that this request passes through (`UnmarshalMetaHandler`).

With our cuckold in place, we’re able to get closer to the point of all this - testing our actual code.

A reminder: I want to test that the credentials are refreshed when they reach their `Expiration` time (3 seconds in the future), and _not before_. The type being tested is an `StsCredentialGetter`, which has an exported function `GetCreds()` that should transparently renew credentials through the un-exported function `updateStsCredentials`. The process will therefore be:

   * create an `StsCredentialGetter`, using the doctored session object from above.
   * call `updateStsCredentials` to obtain an initial set of credentials (which will be the _response_ from above).
   * amend the `dummyCredentials` so that upon the next refresh they’ll return a _different_ `AccessKeyId`.
   * call `GetCreds()` and expect it _not_ to refresh and therefore not return the new `AccessKeyId`
   * wait 4 seconds before calling `GetCreds()` once again. This time we _do_ expect the new `AccessKeyId` to be returned.

The following code shows how to do just that:

```go
s := &StsCredentialGetter{
        Config:  dummyConfig,
        session: sess,
    }
    err := s.updateStsCredentials()
    if err != nil {
        t.Fatalf("Error refreshing credentials", err)
    }
    creds, err := s.GetCreds()
    if err != nil {
        t.Fatalf("Error running GetCreds()", err)
    }
    if creds.AccessKeyID != *dummyCreds.AccessKeyId {
        t.Errorf("GetCreds() Access Key expected: %s, got %s", *dummyCreds.AccessKeyId, creds.AccessKeyID)
    }

    // Credentials should not be re-requested until the Expiration time has been reached.
    // Expiration is set to 3 seconds in the future
    dummyCreds.AccessKeyId = aws.String("IAMADUMMYIAMACCESSKEY-NEW")
    creds, err = s.GetCreds()
    if err != nil {
        t.Fatalf("Error running GetCreds()", err)
    }

    if creds.AccessKeyID == *dummyCreds.AccessKeyId {
        t.Error("GetCreds renewed credentials before expiration.")
    }

    // Sleep 4 seconds and request creds again. This time they should renew.
    time.Sleep(4 * time.Second)
    creds, err = s.GetCreds()
    if err != nil {
        t.Fatalf("Error running GetCreds()", err)
    }

    if creds.AccessKeyID != *dummyCreds.AccessKeyId {
        t.Error("GetCreds failed to renew credentials after expiration.")
    }
```

Hopefully this has provided an example of how writing Go tests can provide an opportunity to be inventive, and can even be rewarding (this example actually _worked_, first time!). Thanks for reading.

[aws-test]: https://github.com/louism517/aws-esproxy/blob/master/creds/aws_test.go
[aws-config]: https://docs.aws.amazon.com/sdk-for-go/api/aws/#Config
[sts-assumeroleoutput]: https://docs.aws.amazon.com/sdk-for-go/api/service/sts/#AssumeRoleOutput


---
layout: post
title:  "Rocket Lamb - Run your Rocket webserver in AWS Lambda"
date:   2019-08-25 16:00:00 +0100
categories: rust
tags: lambda rocket aws
summary: >
    An introduction to my first Rust crate, [Rocket Lamb](https://crates.io/crates/rocket_lamb), which lets you easily run a Rocket
    (a popular Rust web framework) webserver as a serverless application in
    AWS Lambda.
---

I've been very interested in running web applications with serverless technologies for a while, and as I'm a primarily a C# developer, I've mostly been using [Amazon.Lambda.AspNetCoreServer](https://github.com/aws/aws-lambda-dotnet/tree/master/Libraries/src/Amazon.Lambda.AspNetCoreServer). This allows me to write my actual application using the standard ASP.NET Core framework and easily plug it into Lambda. But one of the biggest problems I've experienced with serverless is the slow cold starts, particularly with .NET Core.

Since I've been meaning to get into Rust, I thought I'd try it out with Lambda and see how it compares to what I already know. From some limited googling, Rocket looked like the most ergonomic and "familiar" web framework, but I couldn't find any easy way to hook it up to Lambda. So I made one! It's built on the official [lambda rust runtime](https://github.com/awslabs/aws-lambda-rust-runtime) and dispatches requests to Rocket using a [local Client](https://api.rocket.rs/v0.4/rocket/local/struct.Client.html).

<strong>[Check it out on GitHub here.](https://github.com/GREsau/rocket-lamb)</strong> I was very pleased to find that (from what little testing I've done) cold starts are barely noticeable!

## Basic Usage

Let's say you have a simple Rocket application that has a couple of endpoints configured:
{% highlight rust %}
#[macro_use]
extern crate rocket;

fn main() {
    rocket::ignite()
        .mount("/", routes![foo, bar])
        .launch();
}
{% endhighlight %}

All that's needed to hook it up to Lambda to enable it to serve requests from AWS API Gateway (or an Application Load Balancer) is to add Rocket Lamb's [`lambda()`](https://docs.rs/rocket_lamb/0.5.0/rocket_lamb/trait.RocketExt.html#tymethod.lambda) extension method on your instance of `Rocket`, like so:
{% highlight rust %}
#[macro_use]
extern crate rocket;

use rocket_lamb::RocketExt;

fn main() {
    rocket::ignite()
        .mount("/", routes![foo, bar])
        .lambda()
        .launch();
}
{% endhighlight %}

This is really just sugar for creating an implementation of a [`lambda_http::Handler`](https://docs.rs/lambda_http/0.1.1/lambda_http/trait.Handler.html) and passing it to the [`lambda_http::lambda!`](https://docs.rs/lambda_http/0.1.1/lambda_http/macro.lambda.html) macro. This slightly more verbose implementation would behave identically:
{% highlight rust %}
#[macro_use]
extern crate rocket;

use lambda_http::lambda;
use rocket_lamb::RocketHandlerBuilder;

fn main() {
    let rocket = rocket::ignite().mount("/", routes![foo, bar]);
    let builder = RocketHandlerBuilder::new(rocket);
    let handler = builder.into_handler();
    lambda!(handler);
}
{% endhighlight %}

## Binary Responses
Lambda's runtime API involves passing all data (including response bodies) around inside JSON documents. This generally means everything is encoded as UTF-8 text, but it also accepts base64-encoded binary content. Rocket Lamb will automatically base-64 encode any response body which is not valid UTF-8, but in order for this to be handled correctly by AWS, you must enable binary responses on your API Gateway.

The easiest way to do this is by adding a single catch-all binary media type `*/*`, which can be added to your API Gateway in [the AWS console](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-payload-encodings-configure-with-console.html). If you're deploying to AWS using the AWS Severless Application Model, then you can enable binary responses by adding this global setting to your SAM template:
{% highlight yaml %}
Globals:
  Api:
    BinaryMediaTypes:
      - "*/*"
{% endhighlight %}

You should only need to enable binary responses if you're using AWS API Gateway - if you're instead using an Application Load Balancer, it should just work immediately.

## Base Path Handling
When you deploy to API Gateway, you're given a URL like `https://abcd1234.execute-api.eu-west-1.amazonaws.com/Prod`, where `Prod` is the API Gateway deployment stage. Because the stage is part of the path, the server must be aware of it in order to generate correct absolute URLs in responses (e.g. in the Location response header for redirects). This problem also occurs when using an [API Gateway custom domain](https://docs.aws.amazon.com/apigateway/latest/developerguide/how-to-custom-domains.html) with a set base path.

To prevent you from having to handle this base path in the application, Rocket Lamb will automatically figure out the base path from the first request received, and then clone all routes of your `Rocket`, prepending the base path onto them. If you'd rather handle the base path yourself, you can change this behaviour by setting a different [`base_path_behaviour`](https://docs.rs/rocket_lamb/0.6.0/rocket_lamb/struct.RocketHandlerBuilder.html#method.base_path_behaviour).

## Building and Deploying to AWS
The [AWS Lambda Rust Runtime readme](https://github.com/awslabs/aws-lambda-rust-runtime#deployment) goes over deployment in detail, so I don't seen any need to repeat it - go check that out!

You may also want to take a look at the [Example Rocket Lamb API](https://github.com/GREsau/example-rocket-lamb-api#deploying-to-aws-lambda), which uses docker-compose as a cross-platform way to produce a ZIP file ready to be uploaded to lambda. It also has a [Serverless Application Model](https://docs.aws.amazon.com/lambda/latest/dg/serverless_app.html) template file to make it easy to deploy to AWS using CloudFormation.

## Running Locally
There are a couple of ways I've used to run a Rocket Lamb application locally:
1. Output two binaries - one for running in lambda, and one for just running locally. The local one can just run the Rocket server normally, without using Rocket Lamb at all. The [Example Rocket Lamb API](https://github.com/GREsau/example-rocket-lamb-api) shows how this can be done.
2. Use AWS SAM CLI's [`local start-api`](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-cli-command-reference-sam-local-start-api.html) - which is a more realistic environment, but pretty slow as it spins up a new docker container for each request (although [this is being improved](https://github.com/awslabs/aws-sam-cli/pull/1305)).

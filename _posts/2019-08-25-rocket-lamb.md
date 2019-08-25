---
layout: post
title:  "Rocket Lamb - Easily run your Rocket webserver in AWS Lambda"
date:   2019-08-25 16:00:00 +0100
categories: rust
tags: lambda rocket aws
---
I've been very interested in running web applications with serverless technologies for a while, and as I'm a primarily a C# developer, I've mostly been using [Amazon.Lambda.AspNetCoreServer](https://github.com/aws/aws-lambda-dotnet/tree/master/Libraries/src/Amazon.Lambda.AspNetCoreServer). This allows me to write my actual application using the standard ASP.NET Core framework and easily plug it into Lambda. But one of the biggest problems I've experienced with serverless is the slow cold starts, particularly with .NET Core.

Since I've been meaning to get into Rust, I thought I'd try it out with Lambda and see how it feels. From some limited googling, Rocket looked like the most ergonomic and "familiar" web framework, but I couldn't find any easy way to hook it up to Lambda. So I made one! It's built on the official [lambda rust runtime](https://github.com/awslabs/aws-lambda-rust-runtime) and dispatches requests to Rocket using a [local Client](https://api.rocket.rs/v0.4/rocket/local/struct.Client.html).

I was very pleased to find that (from what little testing I've done) cold starts are a complete non-issue!

I'm very open to feedback - this was my first Rust project (other than hello world etc.) so there was a lot of learning-by-doing involved in this!

{% highlight rust %}
#![feature(proc_macro_hygiene, decl_macro)]

#[macro_use] extern crate rocket;
use rocket_lamb::RocketExt;

#[get("/")]
fn hello() -> &'static str {
    "Hello, world!"
}

fn main() {
    rocket::ignite()
        .mount("/hello", routes![hello])
        .lambda() // launch the Rocket as a Lambda
        .launch();
}
{% endhighlight %}

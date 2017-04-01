---
layout: post
title: Dockerization of gRPC service in Ruby
---

In a previous post, I had talked about [Building Microservices using gRPC on Ruby](https://shiladitya-bits.github.io/Building-Microservices-from-scratch-using-gRPC-on-Ruby/). Today, let's talk about how to deploy the same application we built using Docker.
 
If you are new to the concept of *Docker* and *containers*,  

*"Docker automates the repetitive tasks of setting up and configuring development environments so that developers can focus on what matters: building great software."*

Learn more about them on the [official site](https://www.docker.com/what-container).

First, make sure your local environment has Docker engine setup. There are plenty of resources on the official website to download and get your local Docker engine running.

####Setting up the Dockerfile

Create a new file in your project root directory: **`Dockerfile`**

```
FROM ruby:2.3.1

RUN mkdir /snip
WORKDIR /snip
COPY Gemfile /snip
COPY Gemfile.lock /snip

RUN bundle config --global frozen 1

RUN bundle install --without development test
COPY lib /snip/lib
COPY Rakefile /snip

EXPOSE 50052
ENTRYPOINT [ "bundle", "exec"]
CMD ["lib/start_server.rb"]
```








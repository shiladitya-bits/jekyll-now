---
layout: post
title: Dockerization of gRPC service in Ruby
---

In a previous post, I had talked about [Building Microservices using gRPC on Ruby](https://shiladitya-bits.github.io/Building-Microservices-from-scratch-using-gRPC-on-Ruby/). Today, let's talk about how to deploy the same application we built using Docker.
 
If you are new to the concept of *Docker* and *containers*,  

*"Docker automates the repetitive tasks of setting up and configuring development environments so that developers can focus on what matters: building great software."*

Learn more about them on the [official site](https://www.docker.com/what-container).

First, make sure your local environment has Docker engine setup. There are plenty of resources on the official website to download and get your local Docker engine running.

### Setting up the Dockerfile

Create a new file in your project root directory: **`Dockerfile`**

```dockerfile
FROM ruby:2.3.1

RUN mkdir /snip
WORKDIR /snip
COPY Gemfile /snip
COPY Gemfile.lock /snip

RUN bundle config --global frozen 1

RUN bundle install --without development test
COPY lib /snip/lib

EXPOSE 50052
ENTRYPOINT [ "bundle", "exec"]
CMD ["lib/start_server.rb"]
```


### Details behind the scenes

Let's try to understand what we wrote here. It is pretty simple: 

1. `FROM ruby:2.3.1` - just extending our docker image from a ruby image so we don't have to install ruby ourselves here.

2. `snip` directory - This is the directory where our application will be installed

3. `bundle config --global frozen 1` - to make sure Gemfile and Gemfile.lock agree with each other 

4. Note, we could also have done `bundle install --deployment` directly, in which case `bundle config --global frozen 1` is enabled by default.

5. `COPY lib /snip/lib` - copying our application directory.

6. `EXPOSE 50052` - exposing the application server port on the container.
 
7.  `ENTRYPOINT` and `CMD` - tells what command to execute when container is deployed.

### Building a Docker image

To build docker image, simply run :

```console
docker build -t snip .
```

This will build your image with `latest` tag. To build an image with a custom tag,

```console
docker build -t snip:v1 .
```

### Why copy Gemfile and application code separately?

If you notice carefully, we are executing `COPY` on `Gemfile`, and application directory `/lib` separately before and after `bundle install`. We could also have copied all of the content at once and run `bundle install` at the end.

The reason is we would lose out on caching the bundler build!

When we run `docker build`, Docker creates a layer for each command executed. When you execute it a second time it will reuse the cache from previous execution if it did not change.
 
If you copy the application directory before bundle install, there is a high probability that COPY command will not get a cache hit because your application code would keep changing. This will stop using cache, and execute all future commands, including bundle install.

With our current Dockerfile, it will only run bundle install in the next build if you have changes in your `Gemfile` or `Gemfile.lock`. 


### Running the docker image on local

Simple command to run your new image (interactive run -i, closing this process will kill container as well):

```console
docker run -i -t snip:latest
```

***Points to note:***
1. `-i` is to run in interactive mode. `Ctrl+C` will kill your container as well. 
2. `-t` snip:latest runs the snip image with latest tag. Check `docker images` output for all images.
3. Check `docker ps` out for all running containers.

### Issue with git based gens with SSH authentication

Many of you who are trying out Docker in your organizations which have private repositories, you might face an issue during Docker build like this:

Let's say your `Gemfile` had this line which uses SSH authentication for your git based `snip` gem:

```ruby
gem 'snip',:git => "git@github.com:shiladitya-bits/snip.git",:branch => 'master'
```

`docker build` gives the following error:

```console
Fetching git@github.com:shiladitya-bits/snip.git
Host key verification failed.
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
```

The above fails because `git@github.com:shiladitya-bits/snip.git` requires SSH based authentication, and there are no SSH credentials inside the docker machine. Hence, `bundle install` which runs successfully on your host machine(where SSH credentials are present in `~/.ssh` directory), the same command fails to run inside your docker machine. 

There are 2 solutions to this problem:

#### HTTPS Based authentication

Change it to:
```ruby
gem 'snip',:git => "https://github.com/shiladitya-bits/snip",:branch => 'master'
```

This will work as it is for public repositories. For **private** repositories, you will have to setup an [OAuth key](https://help.github.com/articles/creating-a-personal-access-token-for-the-command-line/) for access to your repository, and prepend it to your URL:

```ruby
gem 'snip',:git => "https://<YOUR_OAUTH_KEY>>:x-oauth-basic@github.com/shiladitya-bits/snip",:branch => 'master'
```

#### Solution for SSH based authentication

We need to modify our Dockerfile slightly to copy our ssh keys onto the docker machine as well. Here is a modified version:

```dockerfile
FROM ruby:2.3.1

RUN mkdir /snip
WORKDIR /snip
COPY Gemfile /snip
COPY Gemfile.lock /snip

# Create .ssh directory and copying our ssh key from host machine to docker machine
RUN mkdir /root/.ssh

# NOTE: Make sure your local ~/.ssh/id_rsa* is first copied to your local project working directory
COPY id_rsa* /root/.ssh/

RUN bundle config --global frozen 1

RUN eval "$(ssh-agent -s)"
RUN ssh-keyscan -H github.com >> ~/.ssh/known_hosts

# restricting permission to ssh keys
RUN chmod 0600 ~/.ssh/id_rsa
RUN chmod 0600 ~/.ssh/id_rsa.pub

RUN bundle install --without development test
COPY lib /snip/lib

# Removing our copied ssh keys
RUN rm ~/.ssh/id_rsa*

EXPOSE 50052
ENTRYPOINT [ "bundle", "exec"]
CMD ["lib/start_server.rb"]
```

*Small note*: You need to copy your local `~/.ssh/id_rsa*` to your project working directory. This is due to the restriction of `COPY` command not being able to copy files outside of a build context.
 
The above modifications are all aimed at a simple goal - making a valid ssh key available to the docker machine during bundler build. There are other ways of achieving the same, one of them being [Habitus](http://www.habitus.io/) - a build tool for Docker which helps you host some data on a local server which is available to the docker machine. Habitus works well, but I think it is overengineering to achieve what we want to do in this particular case. The above solution might look like a hack, but it is a one time thing which doesn't hurt much later!  

End of this post! As before, you can find the working code for [snip-service](https://github.com/shiladitya-bits/snip-service) on Github.


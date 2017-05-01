---
layout: post
title: Circuit breaker and monitoring of a gRPC service in Ruby (Part 1)
---

gRPC is a well suited framework for a micro-services architecture, where there are small independent components talking to each other over the network. When I started building multiple components communicating with each other over gRPC, I started realizing there were needs for certain basic tools around a gRPC call (both server side and client side), which were not available out of the box. For example, things like instrumentation of gRPC calls, or a simple health API. I realized I was having to write these components every time I built a new service. This brought me to the idea of building a [grpc-commons](https://github.com/shiladitya-bits/grpc-commons), which will be a set of basic tools required in a micro-services architecture. In this post, I will be focusing on a couple of things - circuit breaker & monitoring.

### Circuit Breaker

A common requirement in an architecture, where independent modules are separate services over the network. Difference between a normal method call and a RPC call over the network is that a RPC call can fail, or become unresponsive. From Martin Fowler's post, 

"*The basic idea behind the circuit breaker is very simple. You wrap a protected function call in a circuit breaker object, which monitors for failures. Once the failures reach a certain threshold, the circuit breaker trips, and all further calls to the circuit breaker return with an error, without the protected call being made at all. Usually you'll also want some kind of monitor alert if the circuit breaker trips.*"

This is how a circuit breaker state diagram usually looks:

![Circuit breaker state diagram](https://martinfowler.com/bliki/images/circuitBreaker/state.png)

### Monitoring

Monitoring is crucial to any software system. I will not be talking about general monitoring systems in this post. I will discuss points of monitoring in a gRPC call that could be setup without much developer effort.

In Part 1, I will try to explain the concept of having a `grpc-commons` and the implementation. Will focus on monitoring in Part 2.

## grpc-commons: Under the hood 

I have mostly been writing gRPC services in Ruby recently. grpc-commons is currently, a set of tools which does the above (and more in future) which would be helpful for a gRPC server / client written in Ruby. I have not packaged them separately (tooling for client & server), but will explain them separately here.
 
**If you look at the generated ruby file for a gRPC service:**
   
![generated rb file for grpc service](https://raw.githubusercontent.com/shiladitya-bits/shiladitya-bits.github.io/master/images/grpc-service-1.png)
   
All the GRPC related methods are inherited from `GRPC::GenericService`.
    
And this is how we call a service defined like above:
   
```ruby
     stub = Snip::UrlSnipService::Stub.new('0.0.0.0:50052', :this_channel_is_insecure)
     req = Snip::SnipRequest.new(url: 'http://shiladitya-bits.github.io')
     resp_obj = stub.snip_it(req)
   ```
   
To know more about implementation details of a basic gRPC client-server call in Ruby, head over to my post (here)[https://shiladitya-bits.github.io/Building-Microservices-from-scratch-using-gRPC-on-Ruby/].
 
I wanted the easiest way to plug in my tools on the client side. the `Snip::UrlSnipService::Stub` class is what gets called when you make the RPC call. So, I went ahead and overrode a new module `GrpcCommons::GenericService` from `GRPC::GenericService`

```ruby
module GrpcCommons
  module GenericService

    def self.included klass

      klass.class_eval do
        include GRPC::GenericService

        # This is an override for the same method in GRPC::GenericService
        def self.rpc_stub_class
          stub_claz = super
          instance_meths = stub_claz.instance_methods(false)
          alt_stub_claz = Class.new(stub_claz) do
            instance_meths.each do |meth|
              define_method(meth) do |*args|
                tracker_name = "#{self.class.name[0..self.class.name.rindex("::")]}:#{meth}"

                stoplight_lambda = Stoplight(tracker_name) {
                  super(*args)
                }.with_cool_off_time(ENV['GRPC_COMMONS_COOLOFF_INTERVAL'].to_i)

                response = StatsD.measure(tracker_name) do
                  stoplight_lambda.run
                end
                response

              end
            end
          end
          alt_stub_claz
        end
      end
    end
  end
end
```

[Link to this file on Github](https://github.com/shiladitya-bits/grpc-commons/blob/master/lib/grpc-commons/generic_service.rb)


If you see, I am wrapping each RPC method defined in the Service class (whenever any Service class includes `GrpcCommons::GenericService` module instead of `GRPC::GenericService`, the `Stub` class gets overriden by the above implementation). Each RPC call will go through a circuit breaker check (using Stoplight gem for this). And the actual `run` method goes through a StatsD call for instrumentation. If you are not aware of StatsD and how it helps in monitoring your apps, [read this blog by Datadog](https://www.datadoghq.com/blog/statsd/). I will try to connect the dots in a future post about how I have been doing my monitoring in the 2nd part. 

Now, let's use this tool given in our gRPC client. This is how a modified service definition file will look like after replacing the `GenericService` module:

![generated rb file for grpc service](https://raw.githubusercontent.com/shiladitya-bits/shiladitya-bits.github.io/master/images/grpc-service-2.png)

Make sure the `grpc-commons` gem is included in your Gemfile. Full code is available in a sample grpc app ([snip](https://github.com/shiladitya-bits/snip-service)), which I have been discussing in [previous posts](https://shiladitya-bits.github.io/Building-Microservices-from-scratch-using-gRPC-on-Ruby/) as well.

Now, you are set to use the new wrapper (with Stoplight and StatsD tracking). There is nothing you need to change in the way the gRPC call is made. Just changing your service definition to use the new module does all the work under the hood!

```ruby
    # The client call is exactly same as before!
    stub = Snip::UrlSnipService::Stub.new('0.0.0.0:50052', :this_channel_is_insecure)
    req = Snip::SnipRequest.new(url: 'http://shiladitya-bits.github.io')
    resp_obj = stub.snip_it(req)
```


---
title: Creating a Fulfillment Integration
---

## Prerequisites

This tutorial assumes that you have [installed bundler](http://bundler.io/#getting-started) and Sinatra, and that you have a working knowledge of [Ruby](http://www.ruby-lang.org/en/), [JSON](http://www.json.org/), [Sinatra](http://www.sinatrarb.com/), and [Rack](http://rack.rubyforge.org).

## Introduction

By now, you should be familiar with the basic concepts of [creating an endpoint](creating_endpoints_tutorial). In this tutorial, we'll walk through creating a fictional - yet more realistic - integration, complete with the [endpoint](terminology#endpoints), JSON request files, and even a dummy API we'll use to simulate our drop-shipper.

## Steps to Build the Integration

We will begin our integration with the simplest possible successful endpoint, and gradually add complexity and functionality.

### Create a Basic Endpoint

As with the more basic [endpoint creation tutorial](creating_endpoints_tutorial), we'll use the Spree [EndpointBase gem](https://github.com/spree/endpoint_base) to create our fulfillment endpoint.

To start with, we need a new directory to house our integration files.

```bash
$ mkdir fulfillment_endpoint
$ cd fulfillment_endpoint```

Within our new `fulfillment_endpoint` directory, we will obviously need to have files to make our integration work correctly. We'll need:

---Gemfile---
```ruby
source 'https://rubygems.org'

gem 'endpoint_base', github: 'spree/endpoint_base'```

---config.ru---
```ruby
require './fulfillment_endpoint'
run FulfillmentEndpoint```

---fulfillment_endpoint.rb---
```ruby
require 'endpoint_base'
require 'multi_json'

class FulfillmentEndpoint < EndpointBase
  post '/drop_ship' do
    process_result 200, { 'message_id' => @message[:message_id] }
  end
end```

This is already enough to function as a working endpoint. Let's create a sample incoming JSON file.

---return_id.json---
```json
{
  "message_id": "518726r85010000001",
  "payload": {
  }
}```

Now install the gems, and start the Sinatra server.
```bash
$ bundle install
$ bundle exec rackup -p 9292```

Open a new Terminal window, navigate to the /fulfillment_endpoint directory, and run:

```bash
$ curl --data @./return_id.json -i -X POST -H 'Content-type:application/json' http://localhost:9292/drop_ship

=> HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
Content-Length: 35
X-Content-Type-Options: nosniff
Server: WEBrick/1.3.1 (Ruby/1.9.3/2012-04-20)
Date: Wed, 10 Jul 2013 16:47:31 GMT
Connection: Keep-Alive

{"message_id":"518726r84910000001"}```

The output (including headers, as we included the `-H` switch in our curl command) does exactly what we expect: it returns a success (200) status message along with the `message_id` of the JSON file we passed.

+++
The sample files for the preceding example are available on [Github](https://github.com/spree/hello_endpoint/tree/master/fulfillment_tutorial/basic_endpoint).
+++

### Make the API Call

This is great, as far as it goes, but it doesn't really show the power of the Spree Integrator. We want our Endpoints to interact with third-party services, not just return status messages. We can approximate this by writing a fake fulfillment API, called DummyShip.

---dummy_ship.rb---
```ruby
module DummyShip
  def self.validate_address(address)
    ## Zipcode must be within a given range.
    unless (20170..20179).to_a.include?(address['zipcode'].to_i)
      halt 406
    end
  end
end```

We'll need to require this API in our `config.ru` file.

---config.ru---
```ruby
require './fulfillment_endpoint'
require './dummy_ship'

run FulfillmentEndpoint```

Of course, we'll need to update our endpoint to interact with the API.

---fulfillment_endpoint.rb---
```ruby
require 'endpoint_base'
require 'multi_json'

class FulfillmentEndpoint < EndpointBase
  post '/drop_ship' do
    process_result 200, { 'message_id' => @message[:message_id] }
  end

  post '/validate_address' do
    address = @message[:payload]['order']['shipping_address']

    begin
      result = DummyShip.validate_address(address)
      process_result 200, { 'message_id' => @message[:message_id], 'message' => "notification:info",
        "payload" => { "result" => "The address is valid, and the shipment will be sent." } }
    rescue Exception => e
      process_result 200, { 'message_id' => @message[:message_id], 'message' => "notification:error",
        "payload" => { "result" => "There was a problem with this address." } }
    end
  end
end```

As you can see, our new `validate_address` service will accept an incoming JSON file and extract the shipping_address, storing it in the `address` variable. It then makes a call to our `DummyShip` API's `validate_address` method, passing in the `message` variable. If there are no exceptions, the endpoint returns a `notification:info` message with a payload indicating that all's well.

If there is an exception, however, the endpoint elegantly rescues the exception, and returns a `notification:error` message with a payload indicating that our address is not valid.

Our admittedly simplistic API does nothing more at this point than make sure the zip code we pass in is within a pre-defined range. If it's not, the API returns a 406 error ("Not Acceptable").

Now we just need a couple of JSON files we can try out. Let's make one that passes an order whose shipping address is within the range, and one which is not.

---good_address.json---
```json
{
  "message": "order:new",
  "message_id": "518726r85010000001",
  "payload": {
    "order": {
      "shipping_address": {
        "firstname": "Chris",
        "lastname": "Mar",
        "address1": "112 Hula Lane",
        "address2": "",
        "city": "Leesburg",
        "zipcode": "20175",
        "phone": "555-555-1212",
        "company": "RubyLoco",
        "country": "US",
        "state": "Virginia"
      }
    }
  }
}```

---bad_address.json---
```json
{
  "message": "order:new",
  "message_id": "518726r85010000001",
  "payload": {
    "order": {
      "shipping_address": {
        "firstname": "Sally",
        "lastname": "Albright",
        "address1": "55 Rye Lane",
        "address2": "",
        "city": "Greensboro",
        "zipcode": "27235",
        "phone": "555-555-1212",
        "company": "Subs and Sandwiches",
        "country": "US",
        "state": "North Carolina"
      }
    }
  }
}```

***
Remember: Sinatra doesn't reload your changes unless you explicitly tell it to. There is a [Sinatra Reloader](http://www.sinatrarb.com/contrib/reloader) gem you can try out on your own, if you like.
***

Time to test it out in curl! First, the address that our API considers valid:

```bash
$ bundle exec rackup -p 9292
$ curl --data @./good_address.json -i -X POST -H 'Content-type:application/json' http://localhost:9292/validate_address

=>HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
Content-Length: 141
X-Content-Type-Options: nosniff
Server: WEBrick/1.3.1 (Ruby/1.9.3/2012-04-20)
Date: Wed, 10 Jul 2013 23:09:47 GMT
Connection: Keep-Alive

{"message_id":"518726r85010000001","message":"notification:info","payload":{"result":"The address is valid, and the shipment will be sent."}}```

Hooray! Our shipment to Leesburg is a go! Now let's try the shipment to Greensboro.

```bash
$ curl --data @./bad_address.json -i -X POST -H 'Content-type:application/json' http://localhost:9292/validate_address

=> HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
Content-Length: 128
X-Content-Type-Options: nosniff
Server: WEBrick/1.3.1 (Ruby/1.9.3/2012-04-20)
Date: Wed, 10 Jul 2013 23:11:16 GMT
Connection: Keep-Alive

{"message_id":"518726r85010000001","message":"notification:error","payload":{"result":"There was a problem with this address."}}```

As we expected, the zip code for this order is outside the API's acceptable range; this shipment can not be sent with the DummyShip fulfillment process.

+++
The sample files for the preceding example are available on [Github](https://github.com/spree/hello_endpoint/tree/master/fulfillment_tutorial/dummy_ship).
+++

### Return Multiple Messages
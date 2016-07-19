---
layout: post
title: "Phoenix Channels In A Google Chrome Extension"
date: 2015-12-16T09:44:47+01:00
---

Phoenix Channels are awesome! With this out of the way let's get down to business.

I switched from a combination of Ruby on Rails, and Faye on NodeJS to Elixir/Phoenix.
Phoenix's Channels javascript is conveniently integrated right from the start in every Phoenix project. You just use it.
But I needed it in a Google Chrome Extension and I did not found a stand-alone javascript version.

I dug for the original javascript and found it in ```PROJECT_PATH/deps/phoenix/web/static/js/phoenix.js``` in the form of ECMAScript 6.
Now Phoenix handles ES6 very elegantly via Brunch, but in my Chrome Extension project I use CoffeeScript and sometimes plain Javascript.
So I took the ES6 file and converted it to Javascript. I tried different online [transpilers](https://en.wikipedia.org/wiki/Source-to-source_compiler) and in the end went with [Babel](https://babeljs.io/repl/).

I wrote a publish-subscribe-client to connect to the server and handle the channels and the subscribe, publish, and unsubscribe actions:

{% highlight coffee %}
class PubsubClient
  constructor: (host, signedToken)->
    # host is something like https://my.server.com
    # signedToken is retrieved by a previous api call from the Phoenix backend

    # Socket comes from the transpiled phoenix.js
    @socket = new Socket(@_connectionPath(host), params: {signedToken: signedToken})
    @socket.connect()
    @channels = {}
  
  subscribe: (channel, receivers, next)->
    # receivers look something like this:
    # { 
    #   'some:event': (msg)-> console.log(msg) 
    #   'some:other_event': (msg)-> console.log(msg)
    #   ...
    # }
    @channels[channel] ||= @socket.channel(channel)
    _.each receivers, (receiver, topic)=>
      @channels[channel].on(topic, (data)->
        console.debug "message #{topic} received on #{channel}"
        receiver(data)
      )
  
    @channels[channel]
    .join()
    .receive('error', (resp)->
      console.error "An error occured on joining '#{channel}'", resp
    )
    .receive('ok', ->
      console.debug "Successfully joined #{channel}"
      next?()
    )
  
  publish: (channel, topic, data = {}, next = ->)->
    @channels[channel] ||= @socket.channel(channel, {})
    @channels[channel]
    .push(topic, data)
    .receive('ok', next)
  
  unsubscribe: (channel, next)->
    if @channels[channel]?
      @channels[channel]
      .leave()
      .receive('ok', (data)=>
        @channels[channel] = null
        next?(data)
      )
  
  _connectionPath: (host)->
    "#{host.replace('http:','ws:').replace('https:','wss:')}/socket"
{% endhighlight %}

There is a lot of room for improvements here. For one the passed follow-up function in form of the ```next```-parameters should be replaced with promises.

But that's a problem for future me.

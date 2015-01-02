---
layout:     post
title:      "I Wrote A Ruby Webserver"
date:       2014-12-28
summary:    "Moar Understanding of the Http Request Cycle"
categories: 
---
I wrote a small, single threaded, non-performant, ruby webserver to better understand where web servers bog down, what limits (or boosts) their performance, and the implications of server architecture in a framework like Rails.

###A Simple REPL
The core of my webserver is a simple REPL (Read Evaluate Print Loop) that sits on top of an instance of Ruby's `TCPServer`.  Once given a port to listen on, `TCPServer` will generate `TCPSocket` objects to represent the bi-directional communication between a browser and our server.

{% highlight ruby %}
  def run
    puts "Connecting on port #{@port}"

    loop do
      socket = @server.accept
      request = verb_and_header(socket)

      http_verb = request[:verb]
      request_uri = request[:request_uri]
      headers = request[:headers]

      if headers.has_key? 'Content-Length'
        http_body = body(socket, headers['Content-Length'].to_i)
        headers['Body'] = http_body
        STDERR.puts http_body 
      end

      if http_verb == 'GET'
        get(request_uri, socket)
      elsif http_verb == 'POST'
        post(request, socket)
      elsif http_verb == 'PUT'
        put(request, socket)
      elsif http_verb == 'DELETE'
        delete(request, socket)
      elsif http_verb == 'HEAD'
        head(request, socket)
      end
    end
  end
{% endhighlight %}

###An HTTP Request/Response Cycle
In my server, I'm letting Ruby's `TCPServer` handle the encoding and decoding of the TCP socket connection.  But since we're dealing with a web browser, we know that we'll need to handle and parse HTTP requests.  In most simple terms, here's an HTTP lifecycle for a GET request:

* The browser issues an HTTP request via a TCP socket connection to our IP address.  Assuming our server is up and running, it will accept the connection and open up a socket for two way communication.

* Once the browser confirms a connection, it sends the request. For example:

{% highlight html %}
GET /index.html HTTP/1.1
Host: www.example.com
User-Agent: Mozilla/5.0 
[blank line here]
{% endhighlight %}

* We'll need to parse the above request, extracting the HTTP verb ('GET' in this case) and the file it's requesting from the server.  The remainder of the request contains the headers, which denote various metadata about the connection.

* Finally, our server should respond in kind along with the contents of the file and then close the connection. For example:
{% highlight html %}
HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 130
Connection: close
[blank line here]
{% endhighlight %}

Let's dive into the last two steps.

###Parsing the Request
Once the socket connection is open between the two computers, we'll need to extract the HTTP verb from the initial request. 

{% highlight ruby %}
  def verb_and_header(socket)
    request = ""
    headers = {}
    while (request_line = socket.gets and request_line != "\r\n")
      request += request_line
    end

    header_params = request.split(/\r?\n/)
    first_line = header_params.shift.split(" ")
    verb = first_line[0]
    request_uri = first_line[1]
    header_params.each do |param|
      key_value = param.split(':')
      headers.merge!(key_value[0] => key_value[1].strip)
    end
    {:verb => verb, :request_uri => request_uri, :headers => headers}
  end
{% endhighlight %}

Since we know how a HTTP/1.1 request will be formatted, we know the http verb will be the first word in the first line, and the requested file will be the following string. Following that will be the header, which we parse and store as key/value pairs in a hash.

###Responding to the Request
Next, back in our REPL we'll need to process the HTTP verb and respond. 

{% highlight ruby %}
  if http_verb == 'GET'
    get(request_uri, socket)
  elsif http_verb == 'POST'
    #post(request, socket)
  elsif http_verb == 'PUT'
    #put(request, socket)
  elsif http_verb == 'DELETE'
    #delete(request, socket)
  elsif http_verb == 'HEAD'
    #head(request, socket)
  end
{% endhighlight %}

In this case, we've got a GET request, so we lookup the requested file and, if it's found, respond with a 200 status code and then print the contents of the file via `IO.copy_stream`.

{% highlight ruby %}
  def get(request_uri, socket)
    path = requested_file(request_uri) 

    if File.directory?(path)
      path = File.join(path, 'index.html')
    end

    # Make sure the file exists and is not a directory
    # before attempting to open it.
    if File.exist?(path) && !File.directory?(path)
      File.open(path, "rb") do |file|
        socket.print "HTTP/1.1 200 OK\r\n" +
                     "Content-Type: #{content_type(file)}\r\n" +
                     "Content-Length: #{file.size}\r\n" +
                     "Connection: close\r\n"

        socket.print "\r\n"

        # write the contents of the file to the socket
        IO.copy_stream(file, socket)
      end
    else
      message = "File not found\n"

      # respond with a 404 error code to indicate the file does not exist
      socket.print "HTTP/1.1 404 Not Found\r\n" +
                   "Content-Type: text/plain\r\n" +
                   "Content-Length: #{message.size}\r\n" +
                   "Connection: close\r\n"

      socket.print "\r\n"

      socket.print message
    end
  end
{% endhighlight %}

###Bottleneck #1: Single Threaded
One clear flaw in my webserver is that it can only open up a single socket at a time.  If a second client made a request while the web server was in the process of dealing with a request from another client, the second client would need to wait until the first process finished.

A production web server like Puma solves this problem by opening up each socket connection in its own `Thread`.  Another solution would be to use Unix forking to run multiple web servers at once (this is the approach of a web server like Unicorn).  Each approach has its pros and cons, but its clear that for a web server to be even slightly usable, it needs some method to handle multiple connections at once.

###Bottleneck #2: Waiting for That Response...
In the full lifecycle of a HTTP web request, most of a server's time is spent waiting for the browser to respond.  So if the code to parse an HTTP request is completed in 2 or 3 milliseconds, waiting for the response might take 10 (or 100) times that long.

The Node.js toolkit offers an alternative architecture that eliminates this "waiting" bottleneck.  Node requires that any function performing I/O must use a callback.  This means that Node can run in a single thread and hop from request to request without waiting for a response from any one client. 

This leads me to...

###Next: Concurrency via Event Based Programming or the Actor Model
Similar to Node.js, Ruby has its own event-based programming library in `EventMachine`.  There are a few web servers built on top of `EventMachine` (Thin being perhaps the most widely used).  It's not really fair to compare Node to Thin, as Rails doesn't enforce an "all I/O via callbacks" rule, but I'm interested in exploring event-based programming as an alternative to Rails' threaded architecture.

Celluloid also offers a framework for concurrency via the Actor model.  I'm going to explore this next.

In the end, waiting for I/O is what really slows down web apps, so that's where optimization can really shine.       









 


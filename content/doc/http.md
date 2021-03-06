# HTTP

*HTTP* is tiny, raw HTTP client - and yet simple and convenient. It
offers a simple way to send requests and read responses. The goal is to
make things simple for everyday use; this can make developers happy :)

The best way to learn about *HTTP* is through examples.

## Simple GET

~~~~~ java
    HttpRequest httpRequest = HttpRequest.get("http://jodd.org");
    HttpResponse response = httpRequest.send();

    System.out.println(response);
~~~~~

All *HTTP* classes offers fluent interface, so you can write:

~~~~~ java
    HttpResponse response = HttpRequest.get("http://jodd.org").send();

    System.out.println(response);
~~~~~

You can build the request step by step:

~~~~~ java
    HttpRequest request = new HttpRequest();
    request
		.method("GET")
		.protocol("http")
		.host("srv")
		.port(8080)
		.path("/api/jsonws/user/get-user-by-id");
~~~~~


## Reading response

When HTTP request is sent, the whole response is stored in `HttpResponse` instance.
You can use response for various stuff. You can get the `statusCode()` or
`statusPhrase()`; or any header attribute.

Most important thing is how to read received response body. You may use one
of the following methods:

+ `body()` - raw body content, always in ISO-8859-1 encoding.
+ `bodyText()` - body text, ie string encoded as specified by "Content-Type" header.
+ `bodyBytes()` - returns raw body as byte array, so e.g. downloaded file
can be saved.

Character encoding used in `bodyText()` is one set in the response headers.
If response does not specify the encoding in it's headers (but e.g. only
on the HTML page), you _must_ specify the encoding with `charset()`
method before calling `bodyText()`.
{: .attn}

Don't forget this :)


## Query parameters

Query parameters may be specified in the URL line (but then they have to
be correctly encoded).

~~~~~ java
    HttpResponse response = HttpRequest
		.get("http://srv:8080/api/jsonws/user/get-user-by-id?userId=10194")
		.send();
~~~~~

Other way is by using `query()` method:

~~~~~ java
    HttpResponse response = HttpRequest
		.get("http://srv:8080/api/jsonws/user/get-user-by-id")
		.query("userId", "10194")
		.send();
~~~~~

You can use `query()` for each parameter, or pass many arguments in one
call (varargs). You can also provide `Map<String, String>` as a
parameter too.

Note: query parameters (as well as headers and form parameters) can be
duplicated. Therefore, they are stored in an array internally. Use method
`removeQuery` to remove some parameter, or overloaded method to
replace parameter.
{: .attn}

Finally, you can reach internal query map, that actually holds all
parameters:

~~~~~ java
    Map<String, Object[]> httpParams = request.query();
    httpParams.put("userId", new String[] {"10194"});
~~~~~

## Basic Authentication

Basic authentication is made easy:

~~~~~ java
    request.basicAuthentication("test", "test");
~~~~~

## POST and form parameters

Looks very similar:

~~~~~ java
    HttpResponse response = HttpRequest
		.post("http://srv:8080/api/jsonws/user/get-user-by-id")
		.form("userId", "10194")
		.send();
~~~~~

As you can see, use `form()` in the same way to specify form parameters.
Everything what is said for `query()` applies to the `form()`.

## Upload files

Again, it's easy: just add file form parameter. Here is one real-world
example:

~~~~ java
    HttpRequest httpRequest = HttpRequest
		.post("http://srv:8080/api/jsonws/dlapp/add-file-entry")
		.form(
			"repositoryId", "10178",
			"folderId", "11219",
			"sourceFileName", "a.zip",
			"mimeType", "application/zip",
			"title", "test",
			"description", "Upload test",
			"changeLog", "testing...",
			"file", new File("d:\\a.jpg.zip")
		);

	HttpResponse httpResponse = httpRequest.send();
~~~~~

And that's really all!

### Monitor upload progress

When uploading large file, it is helpful to monitor the progress. For that
purpose you can use `HttpProgressListener` like this:

~~~~~ java
    HttpResponse response = HttpRequest
        .post("http://localhost:8081/hello")
        .form("file", file)
        .monitor(new HttpProgressListener() {
            @Override
            public void transferred(long len) {
                System.out.println(len/size);
            }
        })
        .send();
~~~~~

Before the upload starts, `HttpProgressListener` calculates the `callbackSize`
- the size of chunk in bytes that will be transfered. By default, this size
equals to 1% of total size. Moreover, it is never less then 512 bytes.

`HttpProgressListener` contains the inner field `size` with the total size
of the request. Note that this is the size of _whole_ request, not only the
files! This is the actual number of bytes that is going to be send, and it is
always a bit larger then file size (due to protocol overhead).

## Headers

Add or reach header parameters with method `header()`. Some common
header parameters are already defined as methods, so you will find
`contentType()` etc.

## GZipped content

Just `unzip()` the response.

~~~~~ java
    HttpResponse response = HttpRequest
		.get("http://www.liferay.com")
		.acceptEncoding("gzip")
		.send();

    System.out.println(response.unzip());
~~~~~

## Use body

You can set request body manually - sometimes some APIs allow to specify
commands in it:

~~~~~ java
    HttpResponse response = HttpRequest
		.get("http://srv:8080/api/jsonws/invoke")
		.body("{'$user[userId, screenName] = /user/get-user-by-id' : {'userId':'10194'}}")
		.basicAuthentication("test", "test")
		.send();
~~~~~

Setting the body discards all previously set `form()` parameters.
{: .attn}

However, using `body()` have more sense on `HttpResponse` object, to see
the received content.

## Charsets and Encodings

By default, query and form parameters are encoded in UTF-8. This can be
changed globally in `JoddHttp`, or per instance:

~~~~~ java
    HttpResponse response = HttpRequest
		.get("http://server/index.html")
		.queryEncoding("CP1251")
		.query("param", "value")
		.send();
~~~~~

You can set form encoding similarly. Moreover, form posting detects
value of **charset** in "Content-Type" header, and if present,
it will be used.

With received content, `body()` method always returns the **raw** string
(encoded as ISO-8859-1). To get string in usable form, use method
`bodyText()`. This method uses provided **charset** from
"Content-Type" header and encodes the body string.


## HttpConnection

Physical HTTP communication is encapsulated by `HttpConnection` interface.
On `send()`, *HTTP* will `open()` connection if not already opened.
Connections are created by the connection provider:
`HttpConnectionProvider`. Default connection provider
is socket-based and it always returns a new `SocketHttpConnection` instance -
that simply wraps a `Socket` and opens it.

It is common to use custom `HttpConnectionProvider`, based on default
implementation. For example, you may extend the `SocketHttpConnectionProvider`
and override `createSocket()` method to return sockets from some pool,
or sockets with different timeout.

Alternatively, you many even provide instance of `HttpConnection` directly,
without any provider.


## Socket

As said, default communication goes through the plain `Socket`. Since it is
a common need to tweak socket behavior before sending data, here are two
examples how you can do it with *HTTP*.

### SocketHttpConnection

Since we know the default type of `HttpConnection`, we can simply get the
instance after explicitly calling the `open()` and cast it:

~~~~~ java
    HttpRequest request = HttpRequest.get()...;
    request.open();
    SocketHttpConnection httpConnection =
        (SocketHttpConnection) request.httpConnection();
    Socket socket = httpConnection.getSocket();
    socket.setSoTimeout(1000);
    ...
    HttpResponse response = request.send();
~~~~~

### SocketHttpConnectionProvider

The other way is to use custom `HttpConnectionProvider` based on `SocketHttpConnectionProvider`. So you may create your own provider like this:

~~~~~ java
    public class MyConnectionProvider extends SocketHttpConnectionProvider {
        protected Socket createSocket(
                SocketFactory socketFactory, String host, int port)
                throws IOException {
            Socket socket = super.createSocket(socketFactory, host, port);
            socket.setSoTimeout(1000);
            return socket;
        }
    }
~~~~~

Now you can use this provider directly in `open()` method:

~~~~~ java
    HttpConnectionProvider connectionProvider = new MyConnectionProvider();
    ...
    HttpRequest request = HttpRequest.get()...;
    HttpResponse response = request.open(connectionProvider).send();
~~~~~

If you want your connection provider to be default one for all your communication,
just assign it to the `JoddHttp.httpConnectionProvider` and you can avoid
the explicit `open()` usage.

## Keep-Alive

By default, all connections are marked as _closed_, to keep servers happy.
*HTTP* allows usage of permanent connections through 'keep-alive' mode.
The `HttpConnection` is opened on the first request and then re-used
in communication session; the socked is not opened again if not needed
and therefore it is reused for several requests.

There are several ways how to do this. The easiest way is the following:

~~~~~~ java
        HttpRequest request = HttpRequest.get("http://jodd.org");
        HttpResponse response = request.connectionKeepAlive(true).send();

        // next request
        request = HttpRequest.get("http://jodd.org/jodd.css");
        response = request.keepAlive(response, true).send();

        ...

        // last request
        request = HttpRequest.get("http://jodd.org/jodd.png");
        response = request.keepAlive(response, false).send();

        // optionally
        //response.close();
~~~~~~

This example fires several requests over the same `HttpConnection`
(i.e. the same socket). When in 'keep-alive' mode, *HTTP* continues
using the existing connection, while paying attention on servers
responses. If server explicitly requires connection to be closed,
*HTTP* will close it and then it will open a new connection to
continue your session. You don't have to worry about this, just
keep calling `keepAlive()` and it will magically do everything
for you in the background. Just don't forget to pass `false` argument
to the last call to indicate server that is the last connection and that
we want to close after receiving the last response. (if for some reasons
the server does not responds correctly, you may close communication
on client side with an explicit call to `response.close()`).
One more thing - if a new connection has to be opened during this
persistent session (when e.g. keep-alive max counter is finished or timeout
expired) the same connection provider will be used as for the initial,
first connection.

## Proxy

`HttpConnectionProvider` also allows you to specify the proxy. Just provide
the `ProxyInfo` instance with the information about the used proxy
(type, address, port, username, password).

*HTTP* supports HTTP, SOCKS4 and SOCKE5 proxy types.


## Parse from InputStreams

Both `HttpRequest` and `HttpResponse` have a method
`readFrom(InputStream)`. Basically, you can parse input stream with
these methods. This is, for example, how you can read request on server
side.

## HttpBrowser

Sending simple requests and receiving response is not enough for situation
when you have to emulate some 'walking' scenario through a target site. For
example, you might need to login, like you would do that in the browser and
than to continue browsing withing current session.

`HttpBrowser` is a tool just for that. It sends requests for you; handles
301 and 302 redirections automatically, reads cookies from the response
and stores into the new request and so on. Usage is simple:

~~~~~ java
	HttpBrowser browser = new HttpBrowser();

	HttpRequest request = HttpRequest.get("www.facebook.com");
	browser.sendRequest(request);

	// request is sent and response is received

	// process the page:
	String page = browser.getPage();

	// create new request
	HttpRequest newRequest = HttpRequest.post(formAction);

	browser.sendRequest(newRequest);
~~~~~

Browser instance handles all the cookies, allowing session to be tracked while
browsing using HTTP and supports keep-alive persistent connections.

## HttpTunnel

*HTTP* is so flexible that you can easily build a HTTP tunnel with it -
small proxy between you and destination. We even give you a base class:
`HttpTunnel` class, that provides easy HTTP tunneling. It opens server
socket on one port and tunnels the whole HTTP traffic to some target
address.

[TinyTunnel][1] is one implementation that simply prints
out the whole communication to the console.


[1]: https://github.com/oblac/tools/blob/master/src/jodd/tools/http/TinyTunnel.java

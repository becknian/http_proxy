# HTTP proxy with caching

An HTTP proxy that accepts HTTP GET requests from clients, fetches the desired content, and returns the requested content to the client. The HTTP proxy also caches data locally in order to reduce the response time.

## Instructions

**To Compile:**

```console
make
```

**To clean:**

```console
make clean
```

**To run:**

```console
./proxy cache_size_in_bytes
```

## Description

### 1. Background

__HTTP GET__  requests  are  used  to  request  data  from  the  specified  source.  For  example,  if  we  want  to  access  a webpage at `http://www.foo.com/bar.html` , our browser will send something similar to the following __HTTP GET__ request.

**GET /bar.html HTTP/1.1**  
**Host: `www.foo.com`**  
**User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:28.0) Gecko/20100101 Firefox/28.0**  
**Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8**  
**Accept-Language: en-US,en;q=0.5**  
**Accept-Encoding: gzip, deflate**  
**Connection: keep-alive**  

In  the  above  example,  the  Firefox  browser  is  the  client,  and Host: `www.foo.com` is  the  server  host  name.  The client is requesting server resource located at /bar.html. Usually, HTTP serversrun on port 80. So, by default, the server  port  number  is  80.  If,  for  example,  the  HTTP  server  runs  on  port  8080,  then  the  URL  would  be `http://www.foo.com:8080/bar.html`, and the Host field in the HTTP request will contain the port number: `www.foo.com:8080`.  
The HTTP server replies with HTTP response. A valid HTTP response includes: (1) the status line, (2) the response header, and (3) the message body. For example, the text below shows an HTTP response. Note that there is an empty line between the response header and the message body. This empty line indicates the end of header fields.  

**HTTP/1.1 200 OK**  
**Date: Thu, 02 Apr 2015 01:51:49 GMT**  
**Server: Apache/2.2.16 (Debian)**  
**Last-Modified: Tue, 10 Feb 2015 17:56:15 GMT**
**Accept-Ranges: bytes**  
**Content-Length: 2693**
**Content-Type: text/html**  

**content of the bar.html...**  

An _HTTP proxy_ is an application that resides between the HTTP client and server.  The HTTP proxy relays the client’s HTTP requests to the HTTP server, and forwards the server’s response back to the client. Proxies are often used when the client cannot directly connect with the HTTP server. If a client is configured to use an HTTP proxy, the HTTP GET request it sends out will not only include path to the resource at the server, but also the server hostname. For example, the request would look like:

**GET `http://www.foo.com/bar.html` HTTP/1.1**  
**Host: `www.foo.com`**  
**User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:28.0) Gecko/20100101 Firefox/28.0**  
**Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8**  
**Accept-Language: en-US,en;q=0.5**  
**Accept-Encoding: gzip, deflate**  
**Connection: keep-alive**  

To request the resource for the client, the _HTTP proxy_ starts a new TCP connection with the server host `www.foo.com` at port number 80 and ask for filebar.html. When the HTTP proxy relays this request from the client, it will have to remove the host name,`http://www.foo.com` from the request line of the _GET_ request, so that the request line specifies only the local resource path at the destination server. When the HTTP response arrives at the proxy, the proxy will forward it to the client.  If the proxy is correctly implemented, a client web-browser should be able to display the requested resource.  

### 2. Multi-threaded HTTP Proxy

The HTTP proxy supports a subset of the HTTP standard. The proxy correctly handles HTTP requests where a single request/response pair is sent over a single TCP connection. Specifically, the proxy operates as follows:

- It creates a TCP server socket to listen for incoming TCP connections on an unused port, and output the host name and port number the proxy is running on.
- When a new connection request comes in, the proxy accepts the connection, establising a TCP connectionwith the client.
- The proxy reads HTTP request sent from the client and prepares an HTTP request to send to the HTTP server.
- The proxy starts a new connection with the HTTP server, and sends its prepared HTTP request to the server.
- The proxy reads the HTTP response and forwards the HTTP response (the status line, the response header,and the entity body).
- The proxy closes the TCP connection with the server.
- The proxy sends the server’s response back to the client via its TCP connection with the client.  This TCP connection with the client reamins open during its communication with the server. The proxy uses _Content-Length_ to determine how much response content to read from the server and forward to the client.
- The proxy closes the connection socket to the client. 

To maintain a high level of throughput even under multiple simultaneous client requests, different requests is handled in different threads(Posix threads), with a new thread created for each client request.

### 3. Testing

First, download a resource located at a URL using the _wget_ command without the proxy. This is the correct resource. Then download the same resource at the same URL with your proxy. Use the _diff_ command to check if the two downloaded resources matches. Suppose your HTTP proxy is started on _remote02.cs.binghamton.edu_ port 47590.  You can run the following Bash shell command in an EMPTY directory on a remote.cs computer to download web resource via your proxy.

```console
$> bash
$> export http_proxy= http://remote02.cs.binghamton.edu:47590 && wget http://www.foo.com/bar.html
```

To download without the HTTP proxy, use the following command:

```console
$> export http_proxy="" && wget http://www.foo.com/bar.htmlNote:
```

You must replace `http://www.foo.com/bar.html` with a valid URL. You may also run your HTTP proxy on your own machine and use Wireshark for debugging. Your web browser also has a cache. A second request for the same URL may be directly served by your browser cache without going through the proxy. Therefore, you are recommended to use _wget_ for debuggingand testing.

### 4. Caching

The _HTTP proxy_ implements caching.  With caching, the proxy stores the responses of past requests in its local storage. If an incoming request matches an entry in the cache, the proxy returns its cached data in its local storage to the client directly. This can greatly reduce the response time and reduce the network bandwidth. The cache size is set as a command line input to the proxy executable. For example, if the maximum cache size is 320,000 bytes, then your proxy should be started as: ___./proxy 320000___. If the cache becomes full, Least Recently Used (LRU) replacement policy is used for selecting an entry in the cache to evict. Upon successfully serving a request, your proxy writes to standard output (stdout) the following items:

**client IP** - client’s IP address in dotted decimal representation.  
**requested URL**  
**cache status** - can only be one of the two values: __CACHE_HIT__ or __CACHE_MISS__  
**content length** - the size of the entity body in bytes.  
**request time** - request processing time in milliseconds from when the connection from the client is accepted to when the connection to the client is closed (i.e., the last bytes were sent to the client).  

These five items of a same request should be printed to a same line and seperated by the “|” character. Below is an example:  

```console
128.226.118.20|http://www.foo.com/bar.html|CACHE_MISS|2693|300  
128.226.118.26|http://www.foo.com/bar.html|CACHE_HIT|2693|2
```

The first request for `http://www.foo.com/bar.html` was not found in the cache. So the resource was fetched from the remote server. Total request time was 300 ms. When a different client with a different IP address requested the same URL, it was a cache hit, and the request time was much smaller, only 2 ms.

.. include:: util/frontpage.rst

.. include:: util/printheaders.rst

.. include:: build/version.rst

.. contents::
   :class: handout

.. include:: util/control.rst

.. include:: util/param.rst

.. raw:: pdf

   PageBreak coverPage

.. class:: heading1

HTTP
====

This chapter covers:

- Protocol basics
- Requests and responses
- HTTP request/response control flow
- Statelessness and idempotence
- Cache related headers

.. container:: handout

   Varnish is designed to be used with HTTP semantics.
   These semantics are specified in the version called HTTP/1.1.
   This chapter covers the basics of HTTP as a protocol, its semantics and the caching header fields most commonly used.

Protocol Basics
---------------

.. figure 17

.. figure:: ui/img/httprequestflow.png
   :align: center
   :width: 100%

   Figure :counter:`figure`: HTTP request/response control flow diagram

- Hyper-Text Transfer Protocol, HTTP, is at the core of the web
- Specified by the IETF, there are two main versions: HTTP/1.1 and HTTP/2(HTTP/0.9 and HTTP/1.0 deprecated now)
- Varnish 4.0 supports HTTP/1.1 and HTTP/1.0

.. container:: handout

   HTTP is a networking protocol for distributed systems.
   It is the foundation of data communication for the web. 
   The development of this standard is done by the IETF and the W3C.
   In 2014, RFCs 723X were published to clarify HTTP/1.1 and they obsolete RFC 2616.

   A new version of HTTP called HTTP/2 has been released under RFC 7540.
   HTTP/2 is an alternative to HTTP/1.1 and does not obsolete the HTTP/1.1 message syntax.
   HTTP's existing semantics remain unchanged.

   The protocol allows multiple requests to be sent in serial mode over a single connection.
   If a client wants to fetch resources in parallel, it must open multiple connections.

Resources and Representations
.............................

- Resource: target of an HTTP request
- A resource may have different representations
- Representation: a particular instantiation of a resource
- A representation may have different states: past, current and desired

.. container:: handout

   Each resource is identified by a Uniform Resource Identifier (URI), as described in Section 2.7 of [RFC7230].
   A resource can be anything and such a thing can have different representations.
   A representation is an instantiation of a resource.
   An origin server, a.k.a. backend,  produces this instantiation based on a list of request field headers, e.g., ``User-Agent`` and ``Accept-encoding``.

   When an origin server produces different representations of one resource, it includes a ``Vary`` response header field.
   This response header field is used by Varnish to differentiate between resource variations.
   More details on this are in the `Vary`_ subsection.

   An origin server might include metadata to reflect the state of a representation.
   This metadata is contained in the validator header fields ``ETag`` and ``Last-Modified``.

   In order to construct a response for a client request, an algorithm is used to evaluate and select one representation with a particular state.
   This algorithm is implemented in Varnish and you can customize it in your VCL code.
   Once a representation is selected, the payload for a 200 (OK) or 304 (Not Modified) response can be constructed.

Requests and Responses
......................

- A request is a message from a client to a server that includes the method to be applied to a requested resource, the identifier of the resource, the protocol version in use and an optional message body
- A method is a token that indicates the method to be performed on a URI
- Standard request methods are: ``GET``, ``POST``, ``HEAD``, ``OPTIONS``, ``PUT``, ``DELETE``, ``TRACE``, or ``CONNECT``
- Examples of URIs are ``/img/image.png`` or ``/index.html``
- Header fields are allowed in requests and responses
- Header fields allow client and servers to pass additional information
- A response is a message from a server to a client that consists of a response status, headers and an optional message body

.. container:: handout

   .. Requests

   The first line of a request message is called `Request-Line`, whereas the first line of a response message is called `Status-Line`.
   The Request-Line begins with a method token, followed by the requested resource (URI) and the protocol version.

   .. Methods

   A request method informs the web server what sort of request this is:
   Is the client trying to fetch a resource (``GET``), update some data (``POST``) at the backend, or just get the headers of a resource (``HEAD``)?
   Methods are case-sensitive.

   .. Headers 

   After the Request-Line, request messages may have an arbitrary number of header fields.
   For example: ``Accept-Language``, ``Cookie``, ``Host`` and ``User-Agent``.

   .. Message Bodies and Responses

   Message bodies are optional but they must comply to the requested method.
   For instance, a ``GET`` request should not contain a request body, but a ``POST`` request may contain one.
   Similarly, a web server cannot attach a message body to the response of a ``HEAD`` request.

   .. Responses

   The Status-Line of a response message consists of the protocol version followed by a numeric status code and its associated textual phrase.
   This associated textual phrase is also called `reason`.
   Important is to know that the reason is intended for the human user.
   That means that the client is not required to examine the reason, as it may change and it should not affect the protocol.
   Examples of status codes with their reasons are: ``200 OK``, ``404 File Not Found`` and ``304 Not Modified``.

   Responses also include header fields after the Status-Line, which allow the server to pass additional information about the response.
   Examples of response header fields are: ``Age``, ``ETag``, ``Cache-Control`` and ``Content-Length``.

   .. note::

      Requests and responses share the same syntax for headers and message body, but some headers are request- or response-specific.

Request Example
...............

::

    GET / HTTP/1.1
    Host: localhost
    User-Agent: Mozilla/5.0 (Macintosh; U; Intel Mac OS X 10.5; fr; rv:1.9.2.16) \
    Gecko/20110319 Firefox/3.6.16
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
    Accept-Language: fr,fr-fr;q=0.8,en-us;q=0.5,en;q=0.3
    Accept-Encoding: gzip,deflate
    Accept-Charset: ISO-8859-1,utf-8;q=0.7,*;q=0.7
    Keep-Alive: 115
    Connection: keep-alive
    Cache-Control: max-age=0

.. container:: handout

   The above example is a typical HTTP request that includes a Request-Line, and headers, but no message body.
   The Request-Line consists of the ``GET`` request for the ``/`` resource and the ``HTTP/1.1`` version.
   The request includes the header fields ``Host``, ``User-Agent``, ``Accept``, ``Accept-Language``, ``Accept-Encoding``, ``Accept-Charset``, ``Keep-Alive``, ``Connection`` and ``Cache-Control``.

   Note that the ``Host`` header contains the hostname as seen by the browser.
   The above request was generated by entering http://localhost/ in the browser.
   Most browsers automatically add a number of headers.

   Some of the headers will vary depending on the configuration and state of the client.
   For example, language settings, cached content, forced refresh, etc.
   Whether the server honors these headers will depend on both the server in question and the specific header.

   The following is an example of an HTTP request using the ``POST`` method, which includes a message body::

     POST /accounts/ServiceLoginAuth HTTP/1.1
     Host: www.google.com
     User-Agent: Mozilla/5.0 (Macintosh; U; Intel Mac OS X 10.5; fr; rv:1.9.2.16) \
     Gecko/20110319 Firefox/3.6.16
     Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
     Accept-Language: fr,fr-fr;q=0.8,en-us;q=0.5,en;q=0.3
     Accept-Encoding: gzip,deflate
     Accept-Charset: ISO-8859-1,utf-8;q=0.7,*;q=0.7
     Keep-Alive: 115
     Connection: keep-alive
     Referer: https://www.google.com/accounts/ServiceLogin
     Cookie: GoogleAccountsLocale_session=en;[...]
     Content-Type: application/x-www-form-urlencoded
     Content-Length: 288

     ltmpl=default[...]&signIn=Sign+in&asts=

Response Example
................

::

    HTTP/1.1 200 OK
    Server: Apache/2.2.14 (Ubuntu)
    X-Powered-By: PHP/5.3.2-1ubuntu4.7
    Cache-Control: public, max-age=86400
    Last-Modified: Mon, 04 Apr 2011 04:13:41 +0000
    Expires: Sun, 11 Mar 1984 12:00:00 GMT
    Vary: Cookie,Accept-Encoding
    ETag: "1301890421"
    Content-Type: text/html; charset=utf-8
    Content-Length: 23562
    Date: Mon, 04 Apr 2011 09:02:26 GMT
    X-Varnish: 1886109724 1886107902
    Age: 17324
    Via: 1.1 varnish
    Connection: keep-alive

    [data]

.. container:: handout

   The example above is an HTTP response that contains a Status-Line, headers and message body.
   The Status-Line consists of the ``HTTP/1.1`` version, the status code ``200`` and the reason ``OK``.
   The response status code informs the client (browser) whether the server understood the request and how it replied to it.
   These codes are fully defined in https://tools.ietf.org/html/rfc2616#section-10, but here is an overview of them:

   - 1xx: Informational – Request received, continuing process
   - 2xx: Success – The action was successfully received, understood, and accepted
   - 3xx: Redirection – Further action must be taken in order to complete the request
   - 4xx: Client Error – The request contains bad syntax or cannot be fulfilled
   - 5xx: Server Error –  The server failed to fulfill an apparently valid request

HTTP Properties
---------------

- HTTP is a stateless protocol
- Properties of methods: safe, idempotent and **cacheable**
- Most common cacheable request methods are ``GET`` and ``HEAD``

.. container:: handout

    HTTP is by definition a stateless protocol meaning that each request message can be understood in isolation.
    Hence, a server MUST NOT assume that two requests on the same connection are from the same user agent unless the connection is secured and specific to that agent.
    
    HTTP/1.1 persists connections by default.
    This is contrary to most implementations of HTTP/1.0, where each connection is established by the client prior to the request and closed by the server after sending the response.
    Therefore, for compatibility reasons, persistent connections may be explicitly negotiated as they are not the default behavior in HTTP/1.0 [https://tools.ietf.org/html/rfc7230#appendix-A.1.2].
    In practice, there is a header called ``Keep-Alive`` you may use if you want to control the connection persistence between the client and the server.

    Safe methods are considered "safe" if they are read-only; i.e., the client request does not alter any state on the server.
    ``GET``, ``HEAD``, ``OPTIONS``, and ``TRACE`` methods are defined to be safe.
    An `idempotent` method is such that multiple identical requests have the same effect as a single request.
    ``PUT``, ``DELETE`` and safe requests methods are idempotent.
    **Cacheable methods** are those that allow to store their responses for future reuse.
    RFC7231 specifies ``GET``, ``HEAD`` and ``POST`` as cacheable.
    However, responses from ``POST`` are very rarely treated as cacheable.
    [https://tools.ietf.org/html/rfc7231#section-4.2]

Web performance
===============

What is it?
-----------

.. container:: handout

	Web performance refers to the speed in which web pages are downloaded and displayed on the user's web browser.
	Faster website download speeds have been shown to increase visitor retention and loyalty and user satisfaction, especially for users with slow internet connections and those on mobile devices. Web performance also leads to less data travelling across the web, which in turn lowers a website's power consumption and environmental impact.
	

Web performance and caching
---------------------------
	
	There are several ways to improve web performance, caching is among them.
	Fetching something over the network is both slow and expensive: large responses require many roundtrips between the client and server, which delays when they are available and can be processed by the browser, and also incurs data costs for the visitor. As a result, the ability to cache and reuse previously fetched resources is a critical aspect of optimizing for performance.
	
Why should you care about web performance?
------------------------------------------

- Abandonment rates: users tend to abandon the website they are browsing on if the response is not fast enough. This happens more often on phones and tablets.
- Customer satisfaction and loyalty: The impact of poor performance on customer loyalty and returning visitors should also cause concern.
- SEO: Google uses page speed as a ranking signal. This means that faster pages rank higher. By doing this, Google is rewarding better user experience with a higher ranking, and demoting poorer user experience. Having an performant site means that you are not unnecessarily dropping in search result position.


	
Introduction to Varnish Cache and Varnish Software
==================================================

Table of contents:

- Benefits of Varnish
- Open source / Free software
- Varnish Software: The company
- What is Varnish Plus?
- Varnish: more than a cache server
- History of Varnish
- Varnish Governance Board (VGB)

.. TODO Comparison of related software solutions such as: Apache mod_security, Squid, Nginx, and Apache Traffic Server (ATS) (reverse and forward proxy, generally comparable to Nginx and Squid).

Varnish Cache and Varnish Plus
------------------------------

.. table 1

.. csv-table:: Table :counter:`table`: Topics Covered in This Book and Their Availability in Varnish Cache and Varnish Plus
   :name: Topics Covered in This Book and Their Availability in Varnish Cache and Varnish Plus
   :delim: ,
   :header-rows: 1
   :widths: 40,30,30
   :file: tables/varnish_cache_plus_offer_diff.csv

.. container:: handout

   .. Open Source / Free Software:

   Varnish Cache is an open source project, and free software. 
   The development process is public and everyone can submit patches, or just take a peek at the code if there is some uncertainty on how does Varnish Cache work.
   There is a community of volunteers who help each other and newcomers. 
   The BSD-like license used by Varnish Cache does not place significant restriction on re-use of the code, which makes it possible to integrate Varnish Cache in virtually any solution.

   Varnish Cache is developed and tested on GNU/Linux and FreeBSD. 
   The code-base is kept as self-contained as possible to avoid introducing out-side bugs and unneeded complexity. 
   Therefore, Varnish uses very few external libraries.

   .. Varnish Software:

   Varnish Software is the company behind Varnish Cache.
   Varnish Software and the Varnish community maintain a package repository of Varnish Cache for several common GNU/Linux distributions.

   .. Varnish Plus:

   Varnish Software also provides a commercial suite called Varnish Plus with software products for scalability, customization, monitoring and expert support services.
   The engine of the Varnish Plus commercial suite is the enhanced commercial edition of Varnish Cache.
   This edition is proprietary and it is called *Varnish Cache Plus*.

   .. Covered in this book:
   
   `Table 1 <#table-1>`_ shows the components covered in this book and their availability for Varnish Cache users and Varnish Plus customers.
   The covered components of Varnish Plus are described in the `Varnish Plus Software Components`_ chapter.
   For more information about the complete Varnish Plus offer, please visit https://www.varnish-software.com/what-is-varnish-plus.

   .. Supported platforms

   At the moment of writing this book, Varnish Cache supports the operating systems and Linux distributions listed in `Table 2 <#table-2>`_.

   .. table 2

   .. csv-table:: Table :counter:`table`: Varnish Cache and Varnish Plus supported platforms
      :name: Varnish Cache and Varnish Plus supported platforms
      :delim: ,
      :header-rows: 1
      :widths: 50,25,25
      :file: tables/supported_platforms.csv

   Varnish Cache and Varnish Plus support only 64-bit systems.

   .. note::

      Varnish Cache Plus should not be confused with Varnish Plus, a product offering by Varnish Software.
      Varnish Cache Plus is one of the software components available for Varnish Plus customers.

Varnish Cache and Varnish Software Timeline
-------------------------------------------

- 2005: Ideas! Verdens Gang (www.vg.no, Norway's biggest newspaper) were looking for alternative cache solutions
- 2006: Work began:
  Redpill Linpro was in charge of project management, infrastructure and supporting development.
  Poul-Henning Kamp did the majority of the actual development.3
- 2006: Varnish 1.0 is released
- 2008: Varnish 2.0 is released
- 2008: ``varnishtest`` is introduced
- 2009: The first Varnish User Group Meeting is held in London
  Roughly a dozen people participate from all around the world
- 2010: Varnish Software is born as a spin-off to Redpill Linpro AS
- 2011: Varnish 3.0 is released
- 2012: The fifth Varnish User Group Meeting is held in Paris
  Roughly 70 people participate on the User-day and around 30 on the developer-day!
- 2012: The Varnish Book is published
- 2013: Varnish Software chosen as a 2013 Red Herring Top 100 Europe company
- 2013: BOSSIE award winner
- 2013: Varnish Software receives World Summit on Innovation & Entrepreneurship Global Hot 100 award
- 2014: Varnish Plus is launched
- 2014: Varnish 4.0 is released
- 2015: Varnish API Engine is released
- 2015: Gartner names Varnish Software as a 2015 ‘Cool Vendor’ in Web-Scale Platforms
- 2015: Varnish Plus supports SSL/TLS

.. container:: handout

   VG, a large Norwegian newspaper, initiated the Varnish project in cooperation with Linpro. 
   The lead developer of the Varnish project, Poul-Henning Kamp, is an experienced FreeBSD kernel hacker.
   Poul-Henning Kamp continues to bring his wisdom to Varnish in most areas where it counts.

   From 2006 throughout 2008, most of the development was sponsored by VG, API, Escenic and Aftenposten, with project management, infrastructure and extra man-power provided by Redpill Linpro.
   At the time, Redpill Linpro had roughly 140 employees mostly centered around consulting services.

   Today Varnish Software is able to fund the core development with income from service agreements, in addition to offering development of specific features on a case-by-case basis.
   The interest in Varnish continues to increase.
   An informal study based on the list of most popular web sites in Norway indicates that about 75% or more of the web traffic that originates in Norway is served through Varnish.

   .. TODO for the author: reference for the informal study?

   .. VGB

   Varnish development is governed by the Varnish Governance Board (VGB), which thus far has not needed to intervene.
   The VGB consists of an architect, a community representative and a representative from Varnish Software.
   
   .. TODO for the editor: confirm the VGB positions
   
   As of November 2015, the VGB positions are filled by Poul-Henning Kamp (Architect), Rogier Mulhuijzen (Community) and Lasse Karstensen (Varnish Software).
   On a day-to-day basis, there is little need to interfere with the general flow of development.

Description of Varnish Cache
============================

What is Varnish?
----------------

.. figure 1

.. figure:: ui/img/reverse_proxy.svg
   :alt: Reverse Proxy
   :align: center
   :width: 50%

   Figure :counter:`figure`: Varnish is more than a reverse proxy

.. container:: handout

   .. What is Varnish?:

   Varnish is a reverse HTTP proxy, sometimes referred to as an HTTP accelerator or a web accelerator.
   A reverse proxy is a proxy server that appears to clients as an ordinary server.
   Varnish stores (caches) files or fragments of files in memory that are used to reduce the response time and network bandwidth consumption on future, equivalent requests.
   Varnish is designed for modern hardware, modern operating systems and modern work loads.

   .. Varnish use
   
   Varnish is more than a reverse HTTP proxy that caches content to speed up your server.
   Depending on the installation, Varnish can also be used as:

   - web application firewall,
   - DDoS attack defender,
   - load balancer,
   - integration point,
   - single sign-on gateway,
   - authentication and authorization policy mechanism,
   - quick fix for unstable backends, and
   - HTTP router.

Varnish is Flexible
...................

Example of Varnish Configuration Language (**VCL**)::

      vcl 4.0;

      backend default {
	  .host = "127.0.0.1";
	  .port = "8080";
      }

      sub vcl_recv {
	  # Do request header transformations here.
	  if (req.url ~ "^/admin") {
	      return(pass);
	  }
      }

.. container:: handout

   Varnish is flexible because you can configure it and write your own caching policies in its Varnish Configuration Language (VCL).
   VCL is a domain specific language based on C.
   VCL is then translated to C code and compiled, therefore Varnish executes lightning fast.
   Varnish has shown itself to work well both on large (and expensive) servers and tiny appliances.

Varnish Distribution
--------------------

Utility programs part of the Varnish distribution:

- ``varnishd``
- ``varnishtest``
- ``varnishadm``
- ``varnishlog``
- ``varnishstat``
- and more

.. container:: handout

   The Varnish distribution includes several utility programs that you will use in this course.
   You will learn how to use these programs as you progress, but it is useful to have a brief introduction about them before we start.

   .. varnishd

   The central block of Varnish is the Varnish daemon ``varnishd``.
   This daemon accepts HTTP requests from clients, sends requests to a backend, caches the returned objects and replies to the client request.

   .. varnishtest

   ``varnishtest`` is a script driven program used to test your Varnish installation.
   ``varnishtest`` is very powerful because it allows you to create client mock-ups, fetch content from mock-up or real backends, interact with your actual Varnish configuration, and assert the expected behavior.
   ``varnishtest`` is also very useful to learn more about the behavior of Varnish.

   .. varnishadm

   ``varnishadm`` controls a running Varnish instance.
   The  ``varnishadm``  utility establishes a command line interface (CLI) connection to ``varnishd``.
   This utility is the only one that may affect a running instance of Varnish.
   You can use ``varnishadm`` to:

   - start and stop ``varnishd``,
   - change configuration parameters,
   - reload the Varnish Configuration Language (VCL),
   - view the most up-to-date documentation for parameters, and
   - more.

   .. varnishlog

   The Varnish log provides large amounts of information, thus it is usually necessary to filter it.
   For example, "show me only what matches X".
   ``varnishlog`` does precisely that.

   .. varnishstat

   ``varnishstat`` is used to access **global counters**.
   It provides overall statistics, e.g the number of total requests, number of objects, and more.
   ``varnishstat`` is particularly useful when using it together with ``varnishlog`` to analyze your Varnish installation.

   .. others

   In addition, there are other utility programs such as ``varnishncsa``, ``varnishtop`` and ``varnishhist``.

Vocabulary
----------

- Client: A client is a computer program that, as part of its operation, relies on sending a request to another computer program (which may or may not be located on another computer). For example, web browsers are clients that connect to web servers and retrieve web pages for display.(wikipedia) 
- Server: In computing, a server is a computer program or a device that provides functionality for other programs or devices, called "clients".(wikipedia)
- Backend: origin server
- Object(in Varnish):
	- Object: local store of HTTP response message
	- Objects in Varnish are stored in memory and addressed by hash keys
	- You can control the hashing

.. container:: handout

	`Objects` are local stores of response messages as defined in https://tools.ietf.org/html/rfc7234.
	They are mapped with a hash key and they are stored in memory.
	References to objects in memory are kept in a hash tree.


Exercise: Install Varnish
-------------------------

- Install Varnish
- Use packages provided by 

  - varnish-software.com for Varnish Cache Plus
  - varnish-cache.org for Varnish Cache

- When you are done, verify your Varnish version, run ``varnishd -V``

.. container:: handout

   You may skip this exercise if already have a well configured environment to test Varnish.
   In case you get stuck, you may look at the proposed solution.

.. table 3

.. csv-table:: Table :counter:`table`: Different Locations of the Varnish Configuration File
   :name: Different Locations of the Varnish Configuration File
   :delim: ;
   :header-rows: 2
   :file: tables/varnish_configuration_files.csv

[1] The file does not exist by default.
Copy it from ``/lib/systemd/system/`` and edit it.

[2] There is no configuration file.
Use the command ``chkconfig varnishlog/varnishncsa on/off`` instead.

[3] There is no configuration file.
Use the command ``systemctl start/stop/enable/disable/ varnishlog/varnishncsa`` instead.

.. container:: handout

   The configuration file is used to give parameters and command line arguments to the Varnish daemon.
   This file also specifies the location of the VCL file.
   Modifications to this file require to run ``service varnish restart`` for the changes to take effect.

   The location of the Varnish configuration file depends on the operating system and whether it uses the ``init`` system of `SysV`, or `systemd`.
   `Table 3 <#table-3>`_ shows the locations for each system installation.

   .. Introduction to apt-get and yum

   To install packages on Ubuntu or Debian, use the command ``apt-get install <package>``, e.g., ``apt-get install varnish``. 
   For CentOS, RHEL or Fedora, use ``yum install <package>``.
   
   You might want to look at `Solution: Install Varnish`_, if you need help.

   If the command ``service varnish restart`` fail, try to start Varnish manually to get direct feedback from the shell.
   Command example::
     
        $ sudo /usr/sbin/varnishd -j unix,user=varnish,ccgroup=varnish \
        -P /var/run/varnish.pid -f /etc/varnish/default.vcl -a :80 -a :6081,PROXY \
        -T 127.0.0.1:6082 -t 120 -S /etc/varnish/secret \
        - s malloc,256MB -F
   

Configure Varnish
-----------------

- Configure Varnish to use real backends

Varnish ``DAEMON_OPTS``::

  -a ${VARNISH_LISTEN_ADDRESS}:${VARNISH_LISTEN_PORT}
  -T ${VARNISH_ADMIN_LISTEN_ADDRESS}:${VARNISH_ADMIN_LISTEN_PORT}

.. container:: handout

   See `Table 3 <#table-3>`_ and locate the Varnish configuration file for your installation.
   Open and edit that file to listen on port ``80`` and have a management interface on port `1234`.

   In Ubuntu and Debian, this is configured with options ``-a`` and ``-T`` of variable  ``DAEMON_OPTS``.
   In CentOS, RHEL, and Fedora, use ``VARNISH_LISTEN_PORT`` and ``VARNISH_ADMIN_LISTEN_PORT`` respectively.

   In order for changes in the configuration file to take effect, `varnishd` must be restarted.
   The safest way to restart Varnish is by using ``service varnish restart``.

   The default VCL file location is ``/etc/varnish/default.vcl``.
   You can change this location by editing the configuration file.
   The VCL file contains your VCL and backend definitions.

   In this book, we use Apache as backend.
   Before continuing, make sure you have Apache installed and configured to listen in port ``8080``.
   See `Appendix F: Apache as Backend`_ if you do not know how to do it.

   Edit ``/etc/varnish/default.vcl`` to use Apache as backend::

     backend default {
       .host = "127.0.0.1";
       .port = "8080";
     }


Test Varnish Using Apache as Backend
....................................

- Run ``http -p Hh localhost``
- Your output should look as::

   # http -p Hh localhost
   GET / HTTP/1.1
   Accept: */*
   Accept-Encoding: gzip, deflate, compress
   Host: localhost
   User-Agent: HTTPie/0.8.0

   HTTP/1.1 200 OK
   Accept-Ranges: bytes
   Age: 0
   Connection: keep-alive
   Content-Encoding: gzip
   Content-Length: 3256
   Content-Type: text/html
   Date: Wed, 18 Mar 2015 13:55:28 GMT
   ETag: "2cf6-5118f93ad6885-gzip"
   Last-Modified: Wed, 18 Mar 2015 12:53:59 GMT
   Server: Apache/2.4.7 (Ubuntu)
   Vary: Accept-Encoding
   Via: 1.1 varnish-plus-v4
   X-Varnish: 32770

.. container:: handout

   You can test your Varnish installation by issuing the command ``http -p Hh localhost``.
   If you see the HTTP response header field ``Via`` containing ``varnish``, then your installation is correct.

   The ``X-Varnish`` HTTP header field contains the Varnish Transaction ID (VXID) of the client request and if applicable, the VXID of the backend transaction that stored in cache the object delivered.
   ``X-Varnish`` is useful to find the correct log entries in the Varnish log.
   For a cache hit, ``X-Varnish`` contains both the ID of the current request and the ID of the request that populated the cache.


VCL Basics
==========

.. TODO for the author:
   Add the robustness principle: liberal to whatever you receive, conservative: consistent in what you send.
   https://en.wikipedia.org/wiki/Robustness_principle

In this chapter, you will learn the following topics:

- The Varnish Configuration Language (VCL) is a domain-specific language
- VCL as a finite state machine
- States as subroutines
- Varnish includes built-in subroutines
- Available functions, legal return actions and variables

.. container:: handout

   .. Definition of VCL

   The Varnish Configuration Language (VCL) is a domain-specific language designed to describe request handling and document caching policies for Varnish Cache.
   When a new configuration is loaded, the VCC process, created by the Manager process, translates the VCL code to C.
   This C code is compiled typically by ``gcc`` to a shared object.
   The shared object is then loaded into the cacher process.

   .. Section overview

   This chapter focuses on the most important tasks to write effective VCL code.
   For this, you will learn the basic syntax of VCL, and the most important VCL built-in subroutines: ``vcl_recv`` and ``vcl_backend_response``.
   All other built-in subroutines are taught in the next chapter.

   .. tip::

      Remember that Varnish has many reference manuals.
      For more details about VCL, check its manual page by issuing ``man vcl``.

Varnish Finite State Machine
----------------------------

.. Todo for the author: consider to create state tables as B.4 State Tables in Real Time Streaming Protocol 2.0 (RTSP).

- VCL workflow seen as a finite state machine – See `Figure 22 <#figure-22>`_ in the book
- States are conceptualized and implemented as subroutines, e.g., ``sub vcl_recv``
- Built-in subroutines start with ``vcl_``, which is a reserved prefix
- ``return (action)`` terminates subroutines, where ``action`` is a keyword that indicates the next step to do

**Snippet from vcl_recv subroutine**

.. include:: vcl/snippet-vcl_recv.vcl
   :literal:

.. TODO for the editor:
   Slides are missing the state machine graph.

.. TODO for the author:
   Add this ``pipe``-example: websockets go in pipe

.. container:: handout

   .. figure 22

   .. figure:: ui/img/simplified_fsm.svg
      :align: center
      :width: 70%

      Figure :counter:`figure`: Simplified Version of the Varnish Finite State Machine

   .. raw:: pdf

      PageBreak

   .. State machine

   VCL is also often described as a finite state machine.
   Each state has available certain parameters that you can use in your VCL code.
   For example: response HTTP headers are only available after ``vcl_backend_fetch`` state.

   `Figure 22 <#figure-22>`_ shows a simplified version of the Varnish finite state machine.
   This version shows by no means all possible transitions, but only a typical set of them.
   `Figure 23 <#figure-23>`_ and
   `Figure 24 <#figure-24>`_ show the detailed version of the state machine for the **frontend** and **backend**  worker respectively.

   .. Subroutines

   States in VCL are conceptualized as subroutines, with the exception of the *waiting* state described in `Waiting State`_
   Subroutines in VCL take neither arguments nor return values.
   Each subroutine terminates by calling ``return (action)``, where ``action`` is a keyword that indicates the desired outcome.
   Subroutines may inspect and manipulate HTTP header fields and various other aspects of each request.
   Subroutines instruct how requests are handled.

   Subroutine example::
   
     sub pipe_if_local {
       if (client.ip ~ local) {
         return (pipe);
       }
     }

   
   To call a subroutine, use the ``call`` keyword followed by the subroutine's name::

     call pipe_if_local;

   .. Built in subroutines

   Varnish has built-in subroutines that are hook into the Varnish workflow.
   These built-in subroutines are all named ``vcl_*``.
   Your own subroutines cannot start their name with ``vcl_``.

The VCL Finite State Machine
----------------------------

- Each request is processed separately
- Each request is independent from others at any given time
- States are related, but isolated
- ``return(action);`` exits one state and instructs Varnish to proceed to the next state
- Built-in VCL code is always present and appended below your own VCL

.. container:: handout

   Before we begin looking at VCL code, it's worth trying to understand the fundamental concepts behind VCL.
   When Varnish processes a request, it starts by parsing the request itself.
   Next, Varnish separates the request method from headers, verifying that it's a valid HTTP request and so on.

   When the basic parsing has completed, the very first policies are checked to make decisions.
   Policies are a set of rules that the VCL code uses to make a decision.
   Policies help to answer questions such as: should Varnish even attempt to find the requested resource in the cache?
   In this example, the policies are in the ``vcl_recv`` subroutine.

VCL Syntax
----------

- VCL files start with ``vcl 4.0;``
- //, # and /\* foo \*/ for comments
- Subroutines are declared with the ``sub`` keyword
- No loops, state-limited variables
- Terminating statements with a keyword for next action as argument of the ``return()`` function, i.e.: ``return(action)``
- Domain-specific

.. TODO for the author:
   give a reference to .el highlighting lisp file for emacs

.. container:: handout

   .. comments

   Starting with Varnish 4.0, each VCL file must start by declaring its version with a special ``vcl 4.0;`` marker at the top of the file.
   If you have worked with a programming language or two before, the basic syntax of Varnish should be reasonably straightforward.
   VCL is inspired mainly by C and Perl.
   Blocks are delimited by curly braces, statements end with semicolons, and comments may be written as in C, C++ or Perl according to your own preferences.

   .. subroutines

   Subroutines in VCL neither take arguments, nor return values.
   Subroutines in VCL can exchange data only through HTTP headers.

   .. TODO for the author: Double check that 'context' is defined.

   VCL has terminating statements, not traditional return values.
   Subroutines end execution when a ``return(*action*)`` statement is made.
   The *action* tells Varnish what to do next.
   For example, "look this up in cache", "do not look this up in the cache", or "generate an error message".
   To check which actions are available at a given built-in subroutine, see the `Legal Return Actions`_ section or see the manual page of VCL.

   .. warning::
   
      If you define your own subroutine and call it from one of the built-in subroutines, executing ``return(foo)`` does not return execution from your custom subroutine to the default function, but returns execution from VCL to Varnish.

VCL Built-in Functions and Keywords
-----------------------------------

**Functions:**

- ``regsub(str, regex, sub)``
- ``regsuball(str, regex, sub)``
- ``ban(boolean expression)``
- ``hash_data(input)``
- ``synthetic(str)``

**Keywords:**

- ``call subroutine``
- ``return(action)``
- ``new()``
- ``set()``
- ``unset()``

All functions are available in all subroutines, except the listed in the table below.

.. table 14

.. csv-table:: Table :counter:`table`: Specific Function Availability
   :name: Specific Function Availability
   :delim: ,
   :header-rows: 1
   :file: tables/subroutine_functions.csv

.. container:: handout

   VCL offers a handful of simple to use built-in functions that allow you to modify strings, add bans, restart the VCL state engine and return control to the Varnish Run Time (VRT) environment.
   This book describes the most important functions in later sections, so the description at this point is brief.

   .. regsub and regsuball

   ``regsub()`` and ``regsuball()`` have the same syntax and does the almost same thing:
   They both take a string ``str`` as input, search it with a regular-expression ``regex`` and replace it with another string.
   The difference between ``regsub()`` and ``regsuball()`` is that the latter changes all occurrences while the former only affects the first match.

   .. ban

   The ``ban(boolean expression)`` function invalidates all objects in cache that match the boolean expression.
   *banning* and *purging* in detailed in the `Cache Invalidation`_ chapter.

VCL Built-in Subroutines
========================

- Cover the VCL built-in subroutines: ``vcl_recv``, ``vcl_pass``, ``vcl_backend_fetch``, ``vcl_backend_response``, ``vcl_hash``, ``vcl_hit``, ``vcl_miss``, ``vcl_deliver``, and ``vcl_synth``
- If your VCL code does not reach a return statement, the built-in VCL subroutine is executed after yours.

.. container:: handout

   This chapter covers the VCL subroutines where you customize the behavior of Varnish.
   However, this chapter does not define caching policies.
   VCL subroutines can be used to: add custom headers, change the appearance of the Varnish error message, add HTTP redirect features in Varnish, purge content, and define what parts of a cached object is unique.

   After this chapter, you should know what all the VCL subroutines can be used for.
   You should also be ready to dive into more advanced features of Varnish and VCL.

   .. Note::

      It is strongly advised to let the default built-in subroutines whenever is possible.
      The built-in subroutines are designed with safety in mind, which often means that they handle any flaws in your VCL code in a reasonable manner.

   .. Tip::

      Looking at the code of built-in subroutines can help you to understand how to build your own VCL code.
      Built-in subroutines are in the file ``/usr/share/doc/varnish/examples/builtin.vcl.gz`` or ``{varnish-source-code}/bin/varnishd/builtin.vcl``.
      The first location may change depending on your distro.

VCL – ``vcl_recv``
------------------

.. TODO for the author:
   Letting client input to decide your caching policy can make DDoS attack

- Normalize client input
- Pick a backend web server
- Re-write client-data for web applications
- Decide caching policy based on client input
- Access Control Lists (ACL)
- Security barriers, e.g., against SQL injection attacks
- Fixing mistakes, e.g., ``index.htlm`` -> ``index.html``

.. container:: handout

   ``vcl_recv`` is the first VCL subroutine executed, right after Varnish has parsed the client request into its basic data structure. 
   ``vcl_recv`` has four main uses:

   #. Modifying the client data to reduce cache diversity. E.g., removing any leading "www." in the ``Host:`` header.
   #. Deciding which web server to use.
   #. Deciding caching policy based on client data.
      For example; no caching POST requests but only caching specific URLs.
   #. Executing re-write rules needed for specific web applications.

   In ``vcl_recv`` you can perform the following terminating actions:

   `pass`: It passes over the cache *lookup*, but it executes the rest of the Varnish request flow.
   `pass` does not store the response from the backend in the cache.

   `pipe`: This action creates a full-duplex pipe that forwards the client request to the backend without looking at the content.
   Backend replies are forwarded back to the client without caching the content.
   Since Varnish does no longer try to map the content to a request, any subsequent request sent over the same keep-alive connection will also be piped.
   Piped requests do not appear in any log.

   `hash`: It looks up the request in cache.

   `purge`: It looks up the request in cache in order to remove it.
   
   `synth` - Generate a synthetic response from Varnish.
   This synthetic response is typically a web page with an error message.
   `synth` may also be used to redirect client requests.

   It's also common to use ``vcl_recv`` to apply some security measures.
   Varnish is not a replacement for intrusion detection systems, but can still be used to stop some typical attacks early. 
   Simple `Access Control Lists (ACLs)`_ can be applied in ``vcl_recv`` too.

   For further discussion about security in VCL, take a look at the Varnish Security Firewall (VSF) application at https://github.com/comotion/VSF.
   The VSF supports Varnish 3 and above.
   You may also be interested to look at the Security.vcl project at https://github.com/comotion/security.vcl.
   The Security.vcl project, however, supports only Varnish 3.x.

   .. tip::

      The built-in ``vcl_recv`` subroutine may not cache all what you want, but often it's better not to cache some content instead of delivering the wrong content to the wrong user.
      There are exceptions, of course, but if you can not understand why the default VCL does not let you cache some content, it is almost always worth it to investigate why instead of overriding it.

.. TOFIX: Here there is an empty page in slides
.. Look at util/strip-class.gawk

Revisiting built-in ``vcl_recv``
................................

.. include:: vcl/default-vcl_recv.vcl
   :literal:
   
   
   
VCL – ``vcl_backend_response``
------------------------------

- Override cache time for certain URLs
- Strip ``Set-Cookie`` header fields that are not needed
- Strip bugged ``Vary`` header fields
- Add helper-headers to the object for use in banning (more information in later sections)
- Sanitize server response
- Override cache duration
- Apply other caching policies

.. container:: handout

   `Figure 24 <#figure-24>`_ shows that ``vcl_backend_response`` may terminate with one of the following actions: *deliver*, *abandon*, or *retry*.
   The *deliver* terminating action may or may not insert the object into the cache depending on the response of the backend.

   Backends might respond with a ``304`` HTTP headers.
   ``304`` responses happen when the requested object has not been modified since the timestamp ``If-Modified-Since`` in the HTTP header.
   If the request hits a non fresh object (see `Figure 2 <#figure-2>`_), Varnish adds the ``If-Modified-Since`` header with the value of ``t_origin`` to the request and sends it to the backend.

   ``304`` responses do not contain a message body.
   Thus, Varnish builds the response using the body from cache.
   ``304`` responses update the attributes of the cached object.

``vcl_backend_response``
........................

**built-in vcl_backend_response**

.. include:: vcl/default-vcl_backend_response.vcl
   :literal:

.. container:: handout

   The ``vcl_backend_response`` built-in subroutine is designed to avoid caching in conditions that are most probably undesired.
   For example, it avoids caching responses with cookies, i.e., responses with ``Set-Cookie`` HTTP header field.
   This built-in subroutine also avoids *request serialization* described in the `Waiting State`_ section.

   To avoid *request serialization*, ``beresp.uncacheable`` is set to ``true``, which in turn creates a ``hit-for-pass`` object.
   The `hit-for-pass`_ section explains in detail this object type.

   If you still decide to skip the built-in ``vcl_backend_response`` subroutine by having your own and returning ``deliver``, be sure to **never** set ``beresp.ttl`` to ``0``.
   If you skip the built-in subroutine and set ``0`` as TTL value, you are effectively removing objects from cache that could eventually be used to avoid *request serialization*.

   .. note::

      Varnish 3.x has a *hit_for_pass* return action.
      In Varnish 4, this action is achieved by setting ``beresp.uncacheable`` to ``true``.
      The `hit-for-pass`_ section explains this in more detail.

.. TOFIX: Here there is an empty page in slides
.. Look at util/strip-class.gawk

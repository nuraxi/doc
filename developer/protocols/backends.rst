.. highlight:: cpp

.. _backend_protocols:

****************
Backend protocol
****************

This section describes the protocol used by DISCOS to communicate with external
backends. Here you can find the protocol definition, the grammar definition
and a simple example implementation of the protocol server as a twisted
protocol. This definition is liberally inspired by the one used by KAT telescope
for device communication ([KAT]_) as we found that work of really good quality.

==================== ===============
**Protocol Version** 1.2
**Last revision**    25/01/2016
**Status**           PRODUCTION
==================== ===============

Introduction
============

*DISCOS BACKEND PROTOCOL* consits of newline separated
text messages sent synchronously over a TCP/IP connection. Messages can be of
two kinds: request and reply. Requests are sent from a client to a server,
while replies are sent from the server in response to a client's request. Each
request must receive one and only one reply message back and in this kind of
topology the client will be tipically represented by a DISCOS component, while
the server is represented by the backend controller. 
This version of the protocol does not impose any constraint to the number of 
clients connecting to a server but leaves to the clients the responsibility of 
orchestrating their requests in a consistent way. Implementations consisting 
of a single client will be the first and foremost ones.

Here you can find a quick list of the requests defined by the protocol, which
will be better described in the next sections:

========================================= ========================================
Request                                   Description
========================================= ========================================
:ref:`backend-protocol-status`            get the backend status code
:ref:`backend-protocol-version`           get the server protocol version
:ref:`backend-protocol-get-configuration` get the backend actual configuration
:ref:`backend-protocol-set-configuration` set a new backend configuration
:ref:`backend-protocol-get-integration`   get the backend integration time
:ref:`backend-protocol-set-integration`   set a new integration time
:ref:`backend-protocol-get-tpi`           ask the backend a total power level read
:ref:`backend-protocol-get-tp0`           ask the backend a zero level read
:ref:`backend-protocol-time`              get the backend time
:ref:`backend-protocol-start`             start the acquisition [at a given time]
:ref:`backend-protocol-stop`              stop the acquisition [at a given time]
:ref:`backend-protocol-set-section`       configure a section of the backend
:ref:`backend-protocol-cal-on`            (de)activate the calibration mark
:ref:`backend-protocol-set-filename`      tell the backend the output file name
:ref:`backend-protocol-convert-data`      instructs the backend to convert file
========================================= ========================================

Handshake
=========

The protocol is a connected protocol. This means that a client must establish a
connection with the server and keep it open for message exchange. In this
preliminary version a very simple handshake is required: on every new connection
the server will write to the client a REPLY message containing the version of
the protocol that the server is implementing. Further decisions are left to the
client.

Example::

  client.open_connection( server_ip, server_port )

  server -> client 
  reply: "!version,1.0.1\r\n"


client now can continue communication or close it because it detects an
incomptible server version

Note that the message is exactly the same one published in reply to the
:ref:`backend-protocol-version` command.


Messaging Protocol
==================

Communication consists of a number of messages, each message consisting of a
line of text.  The protocol supports requests and replies messages, identified
by the first character of the message:
requests are indicated by '**?**', replies by '**!**'. 
Each message is made up of a **type code** followed by the message **name**, and
optionally by one or many message **arguments**; name and arguments are
separated one each other using the character '**,**' . 

A reply is necessary for every request, however the nature of the reply may
change depending on the request:

  * The reply message should have the same name as the request message.
  * The first argument of a reply message should always be a return code. 
  * A return code of **ok** indicates successful processing of the request, 
    while anything else indicates failure. 
  * The recommended failure strings are **invalid** (for malformed requests) 
    and **fail** (for valid requests which could not be processed). 
  * On success, further reply arguments are specific to the type of request.
  * In the case of failure a second argument should describe the failure 
    in more detail and in human-readable form.

+-----------+---------------------------------------------------------------+
|Return Code|Description                                                    |
+===========+===============================================================+
|ok         || Request successfully processed.                              |
|           |  Further arguments are request specific.                      |
+-----------+---------------------------------------------------------------+
|invalid    || Request  malformed. Second argument is a human-readable      |
|           || description of the error.                                    |
+-----------+---------------------------------------------------------------+
|fail       || Valid request that could not be processed.                   |
|           || Second argument is a human-readable description of the error.|
+-----------+---------------------------------------------------------------+

Line Separation
~~~~~~~~~~~~~~~

Each message is terminated by the sequence **CR LF** (carriage-return + line
feed) as specified by the TELNET
standad ([TELNET]_) . This will make the protocol easily usable also for debug
purposes using simple telnet clients.

Message Grammar
~~~~~~~~~~~~~~~

The message grammar is described in extended BNF [EBNF]_ where:

  * Optional items are enclosed in square brackets
  * Items repeating 0 or more times are suffixed with a *
  * Items repeating 1 or more times are suffixed with a +
  * Alternative choices in a production are separated by the '|' symbol
  * Set difference is indicated by the '/' symbol

The grammar is defined as::

    <message> ::= <type> <name> <arguments> <eol>
       <type> ::= "?" | "!"
       <name> ::= alpha (alpha | digit | "-")*
  <arguments> ::= ("" | <separator> <argument>) [<arguments>]
        <eol> ::= newline
  <separator> ::= ","
   <argument> ::= (<plain> | <escape>)+
      <plain> ::= character / <special>
    <special> ::= backslash | null | newline | carriage-return | escape | tab
     <escape> ::= "\" <escapecode>
  <escapecode>::= "\" | "t" | <separator>

Note that arguments can contain spaces and tabs and are limited only by commas
and newlines at the end of the message.

Data Types
==========

Being the protocol string based, whenever we need to transmit other data types
they must be encoded into strings in an unequivocable manner. This is defined
as per the table below:

  * **integer** as formatted by printf("%d",i). i.e. 10 -15
  * **float** as formatted by printf("%f",f) i.e. -1209087123.234234 1.0
  * **boolean** True as 1 and False as 0 i.e. 1, 0
  * **timestamp** is the number of centinanoseconds since epoch. It is the
    format used by the ACS framework and it can be obtained by multiplying
    the unix timestamp in floating point format by 10000000.
    All times are intended to be **UT** i.e. 143092278297088300

Request and Reply Messages
==========================

For each command we give a brief description of how the command can be used and
the description of the reply to the command. We then provide a simple example.

.. _backend-protocol-status:

status
~~~~~~

Asks the status of the backend. The request message has no arguments.
The Reply message has 3 arguments:

  * **timestamp** the timestamp of the answer message according to the backend
    clock
  * **status code** in normal working condition should be **ok**, any other
    value should be used for representing any possible failure state
  * **acquiring** is a boolean value indicating if the backend is performing an
    acquisition, can be 0 for *false* or 1 for *true*

Example communication::

  request: "?status\r\n"
    reply: "!status,ok,1430922782.97088300,ok,0\r\n"

  request: "?status\r\n"
    reply: "!status,ok,1430922782.97088300,clock error,0\r\n"

.. _backend-protocol-version:

version
~~~~~~~

Asks the backend server what version of the protocol it is implementing. The
Request message has no argument. The Reply message has 1 argument:

  * **version id** a string representing the protocol version

Example communication::

  request: "?version\r\n"
    reply: "!version,1.0.1\r\n"

.. _backend-protocol-get-configuration:

get-configuration
~~~~~~~~~~~~~~~~~

Asks the backend server what configuration is loaded at the moment.
Request message has no argument. The Reply message has 1 argument:

  * **configuration id** a string representing the loaded configuration

If the backend has not yet been configured a special value of **unconfigured**
is returned as reply argument.

Example communication::

  request: "?get-configuration\r\n"
    reply: "!get-configuration,K2000\r\n"

  request: "?get-configuration\r\n"
    reply: "!get-configuration,unconfigured\r\n"

.. _backend-protocol-set-configuration:

set-configuration
~~~~~~~~~~~~~~~~~

Instruct the backend to configure itself according to the specified
configuration code given as argument. Reply message has no argument. Request
message has one argument: 

  * **configuration id** a string identifying the configuration to be loaded

Example communication::

  request: "?set-configuration,K2000\r\n"
    reply: "!set-configuration,ok\r\n"

  request: "?set-configuration,nonexistent\r\n"
    reply: "!set-configuration,fail,cannot find configuration 'nonexistent'\r\n"

.. _backend-protocol-get-integration:

get-integration
~~~~~~~~~~~~~~~

Asks the backend server what integration time is configured.
Request message has no argument. The Reply message has 1 argument:

  * **integration time** an integer representing the integration time configured in ms.

If the backend has not yet been configured a special value of **0**
is returned as reply argument.

Example communication::

  request: "?get-integration\r\n"
    reply: "!get-integration,20\r\n"

  request: "?get-integration\r\n"
    reply: "!get-configuration,0\r\n"

.. _backend-protocol-set-integration:

set-integration
~~~~~~~~~~~~~~~

Instruct the backend to configure itself with the specified integration time given 
as argument. Reply message has no argument. Request message has one argument: 

  * **integration time** an integer representing the integration time to be set in ms.

Example communication::

  request: "?set-integration,20\r\n"
    reply: "!set-integration,ok\r\n"

  request: "?set-integration,wrong\r\n"
    reply: "!set-integration,fail,integration time must be an integer number"

.. _backend-protocol-get-tpi:

get-tpi
~~~~~~~

Ask the backend a total power level read.
Request message has no argument. Reply message has as many arguments as the 
number of sections configured in the backend:

  * **TPI_sec0, TPI_sec1, ...** a sequence of floating point numbers representing the
    total power intensity level of each section of the backend

Example communication::

  request: "?get-tpi\r\n"
    reply: "!get-tpi,ok,900.00,1240.00\r\n"

.. _backend-protocol-get-tp0:

get-tp0
~~~~~~~

Ask the backend a total power read with zero level input.
Request message has no argument. Reply message has as many arguments as the 
number of sections configured in the backend:

  * **TP0_sec0, TP0_sec1, ...** a sequence of floating point numbers representing the
    total power intensity level of each section of the backend with terminated input.

Example communication::

  request: "?get-tp0\r\n"
    reply: "!get-tp0,ok,00.00,00.00\r\n"

.. _backend-protocol-time:

time
~~~~

Asks the backend to return its own timestamp, this command should be used to
verify that the backend has an acceptable clock working before issuing time
tagged acquisition commands. Request has no argument. The reply has one only
argument:

  * **timestamp** the timestamp of the answer message according to the backend
    clock

Example communication::

  request: "?time\r\n"
    reply: "!time,ok,1430922782.97088300\r\n"

.. _backend-protocol-start:

start [timestamp]
~~~~~~~~~~~~~~~~~

Tell the backend to start the acquisition. The reply has no parameter. The reqeust has one optional
parameter:

  * **timestamp** the exact time at which the acquisition should start

If given with a timestamp the backend should continue to accept commands while
waiting for the start time. A stop command will cancel any further pending
acquisition. If a new start command is issued while waiting for a start time, the most
recent start command will overwrite the pending one. 

Example communication::

  request: "?start\r\n"
    reply: "!start,ok\r\n"

  request: "?start,1430922782.97088300\r\n"
    reply: "!start,ok\r\n"

  request: "?start,1430922782.97088300\r\n"
    reply: "!start,fail,cannot start at given time\r\n"

.. _backend-protocol-stop:

stop [timestamp]
~~~~~~~~~~~~~~~~

Tell the backend to stop the acquisition. The reply has no parameter. The reqeust has one optional
parameter:

  * **timestamp** the exact time at which the acquisition should stop

If given with a timestamp the backend should continue to accept commands while
waiting for the stop time. If a new stop command is issued while waiting for a stop time, the most
recent stop command will overwrite the pending one. 

Example communication::

  request: "?stop\r\n"
    reply: "!stop,ok\r\n"

  request: "?stop,1430922782.97088300\r\n"
    reply: "!stop,ok\r\n"

  request: "?stop,1430922782.97088300\r\n"
    reply: "!stop,fail,cannot stop at given time\r\n"

.. note::
   In general we note that the correct behaviour of 
   time tagged commands is left as a responsibility to
   the backend itself and not to the protocol. It will be duty of the
   particoular implementation to keep track of pending start and stop timestamps
   during the acquisition process. For example it is possible to have both a
   start timestamp and a stop timestamp issued in the future, and these should
   work as expected.

.. _backend-protocol-set-section:

set-section
~~~~~~~~~~~

Configure a section of the backend, just as explained in 
:ref:`total-power-focus-selector`. The command expects 7 parameters, all of
which are mandatory, but can be substituted by an asterisk '*' meaning that the
corresponding parameter should be left unchanged. The synthax is::

  set-section=sect,startFreq,bw,feed,mode,sampleRate,bins

The server should respond **ok** if the operation succeds, and **fail** if the server
could not perform the operation or the backend does not support this kind of
operation.

Example communication::

  request: "?set-section,1,50.0,200.0,1,CP,10,2048\r\n"
    reply: "!set-section,ok\r\n"

  request: "?set-section,1,*,*,*,*,*,*\r\n"
    reply: "!set-section,ok\r\n"

  request: "?set-section,1,*\r\n"
    reply: "!set-section,fail,set-section needs 7 arguments\r\n"

  request: "?set-section,1,badparam,200.0,1,CP,10,2048\r\n"
    reply: "!set-section,fail,wrong parameter format\r\n"

.. _backend-protocol-cal-on:

cal-on [interleave=0]
~~~~~~~~~~~~~~~~~~~~~

Activate the noise diode for calibration purposes using a cal switching
acquisition mode. The command takes in input one mandatory parameter:

  * **interleave** the number of backend samples to be acquired with the mark
    on and off alternatively. If **0** is given (default) as parameter the mark
    is always off, this can be used to turn the fast mark off.

Example communication::

  request: "?cal-on\r\n"
    reply: "!cal-on,ok\r\n"

  request: "?cal-on,10\r\n"
    reply: "!cal-on,ok\r\n"

  request: "?cal-on,-10\r\n"
    reply: "!cal-on,fail,interleave samples must be a positive int\r\n"

.. _backend-protocol-set-filename:

set-filename
~~~~~~~~~~~~

Tell the backend where to find the telescope file with metadata to be 
merged with the backend data.

  * **filename** absolute path of the file to be merged

Example communication::

  request: "?set-filename,/hi/im/a/file.fits\r\n"
    reply: "!set-filename,ok\r\n"


.. _backend-protocol-convert-data:

convert-data
~~~~~~~~~~~~

.. versionadded:: 1.2

Instructs the backend to convert data. This means that the control system has
successfully closed the auxiliary metadata file which can now be completed with
collected data from the backend.

Example communication::

  request: "?convert-data\r\n"
    reply: "!convert-data,ok\r\n"

Handling Errors
===============

As specified above, the protocol permits to distinguish between two kinds of
errors, both of which are identified in the response messages:

  * **protocol errors** are identified by the response argument **invalid** 
  * **application errors** are identified by the response argument **fail**

Both responses permit a second argument to specify a description of the error.

Example communication::

  request: "?nonexistentcommand\r\n"
    reply: "!nonexistentcommand,invalid,cannot find command\r\n"

  request: "?--asdf\r\n"
    reply: "!--asdf,invalid,invalid characters in command name\r\n"

  request: "ciao\r\n"
    reply: "!ciao,invalid,requests must start with '?'\r\n"

  request: "?start,0\r\n"
    reply: "!start,fail,invalid timestamp\r\n"

  request: "?start,0\r\n"
    reply: "!start,fail,invalid timestamp\r\n"

Reference Implementation
========================

You can find a reference implementation of the protocol at
https://github.com/discos/discos-backend . This package implements all the
logics related to the protocol, including parsing and serialization
of messages, transmission, checks for correctness and error management. 
The package also defines a server implementation which enables a
pluggable protocol to be used. The developer can just look at the
tests (as described in the package docs) in order to define its own protocol
implementation.

Considerations
==============

The proposed protocol is intentionally very simple and little powerful; this
choice is derived from the specs given in the meeting held the 13 / 04 / 2015 at
OaC. 
More complex interactions would require a different protocol resulting in a more
complex definition and different technologies involved. In particular the actual
definition does **not** : 

 * Version the protocol in the protocol itself
 * Permit to send asynchronous messages
 * Permit biderectional requests
 * Permit to send the same message to multiple recipients
 * Enable any security mechanism

References
==========

.. [EBNF] http://www.cl.cam.ac.uk/~mgk25/iso-14977.pdf
.. [KAT] https://casper.berkeley.edu/wiki/images/1/11/NRF-KAT7-6.0-IFCE-002-Rev4.pdf
.. [TELNET] http://www.freesoft.org/CIE/RFC/1123/31.htm


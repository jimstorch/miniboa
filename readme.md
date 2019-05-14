You can contact me at: 'wvzfgbepu@tznvy.pbz'.encode('rot13') 

----

Update May 14, 2019

I recommend anyone interested in Miniboa please use Shmup's fork for Python 2 AND 3 at:

https://github.com/shmup/miniboa

----

## Overview

Miniboa is a bare-bones Telnet server to use as the base for a MUD or similar interactive server.  Miniboa has several nice features for this type of application.

  * Asynchronous - no waiting on player input or state. 
  * Single threaded - light on resources with excellent performance.
  * Runs under your game loop - you decide when to poll for data.
  * Supports 1000 users under Linux and 512 under Windows (untested).     

To use Miniboa, you create a *Telnet Server* object listening at a specified port number.  You have to provide two functions for the server; the first is a handler for new connections and the second is the handler for lost connections.  These handler functions are passed *Telnet Client* objects -- these are your communication paths to and from the individual player's MUD client.

For example, let's say Mike and Joe connect to your MUD server.  Telnet Server will call your *on_connect()* function with Mike's Telnet Client object, and then again with Joe's Telnet Client object.  If Mike's power goes out, Telnet Server will call your *on_disconnect()* function with Mike's Telnet Client object (same exact one).

## Why NOT use Miniboa?

Miniboa provides linemode input only -- in other words, you get a full line of client input at a time, not individual keypresses.  While this works well for the default mode of most Telnet clients, it may not suit authors who wish to add advanced options like auto-complete and multi-line editing.

Miniboa uses _Socket Select_ for greater compatibility between platforms.  This should easily support hundreds of simultaneous users.  If you're targeting a thousand or more simultaneous users on a _Linux-only_ server platform, you may wish to explore a solution using the new _Epoll IO Selector_.

Future versions of Miniboa may add support for both of these features.

----

## Telnet Server

Telnet servers are instances of the TelnetServer class from miniboa.async.  Creating a Telnet Server is pretty simple.  In fact, you can run one with the following three lines of code:

```
from miniboa import TelnetServer
server = TelnetServer()
while True: server.poll()
```

This will launch a server listening on the default port, 7777, that accepts Telnet connections and sends a simple greeting;
```
$ telnet localhost 7777
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
Greetings from Miniboa!  Now it's time to add your code.
```

Initialization arguments for TelnetServer are;

```
server = TelnetServer(port=8888, address='127.0.0.1', on_connect=my_connect_handler, on_disconnect=my_disconnect_handler)
```

  * *port* - the network port you want the server to listen for new connections on.  You should be aware that on Linux, port numbers 1024 and below are restricted for use by normal users.  The default is port 7777.
  * *address* - this is the address of the _local network interface_ you wish the server to listen on.  You can usually omit this parameter (or pass an empty string; '') which causes it to listen on any viable NIC.  Unless your server is directly connected to the Internet (doubtful today) do not set this to the Internet IP address of your server.  The default is an empty string.  
  * *on_connect* - this is the handler function that is called by the server when a new connection is made.  It will be passed the *Client* object of the new user.  The default is a placeholder function that greets the visitor and prints their connection info to stdout. 
  * *on_disconnect* - this is the hander function that is called by the server when connection is lost.  It will be passed the *Client* object of the lost user.  The default is a placeholder function that prints the lost connection info to stdout.
  * *timeout* - length of time to wait for user input during each poll().  Default is 5 milliseconds.


## Server Properties

The follow properties can be read from.  

  * *port* - port the server is listening to. 
  * *address* - address of the local network interface the server is listening to.  See the explanation of this parameter above.

You can set or change these after creating the server: 

  * *on_connect* - handler function for new connections.
  * *on_disconnect* - handler function for lost connections.
  * *timeout* - length of time to wait on user input during a *pol()*.  Default is .005 seconds.  Increasing this value also lowers CPU usage.    


## Server Methods

  * *poll()* - this is where the server looks for new connections and processes existing connection that need to send and/or receive blocks of data with the players.  A call to this method needs to be made from within your program's game loop.
  * *connection_count()* - returns the number of current connections.
  * *client_list()* - returns a list of the current client objects.

Here's a simple example with custom *on_connect()* and *on_disconnect()* handlers:

```
from miniboa import TelnetServer


CLIENTS = []

def my_on_connect(client):
    """Example on_connect handler."""
    client.send('You connected from %s\r\n' % client.addrport())
    if CLIENTS:
        client.send('Also connected are:\r\n')
        for neighbor in CLIENTS:
            client.send('%s\r\n' % neighbor.addrport())
    else:
        client.send('Sadly, you are alone.\r\n')
    CLIENTS.append(client)


def my_on_disconnect(client):
    """Example on_disconnect handler."""
    CLIENTS.remove(client)

server = TelnetServer()
server.on_connect=my_on_connect
server.on_disconnect=my_on_disconnect

print "\n\nStarting server on port %d.  CTRL-C to interrupt.\n" % server.port
while True:
    server.poll()
```


----

## Telnet Clients

Client objects are instances of the TelnetClient class from miniboa.telnet.  These are a mixture of a state machine, send & receive buffers, and some convenience methods.  They are created when a new connection is detected by the TelnetServer and passed to your *on_connect()* and *on_disconnect()* handler functions.  Your application will probably maintain a list (or some other kind of reference) to these clients so it's important to delete references in your on_disconnect handler or else dead ones will not get garbage collected.

The client buffers user's input and breaks it into lines of text that can be retrieved using the *get_command()* method.

## Client Properties

  * *active* - boolean value, True if the client is in good health.  Setting this to False will cause the TelnetServer to drop the user (and then call your *on_disconnect()* function with that client).   
  * *cmd_ready* - this is set to True whenever the user enters some text and then presses the enter key.  The line of text can be obtained by calling the *get_command()* method.
  * *bytes_sent* - number of bytes sent to the client since the session began.
  * *bytes_received* - number of bytes received from the client since the session began.
  * *columns* - Number of columns the client's window supports.  This is set to a default of 80 and then modified if *request_naws()* is called AND the player's client supports NAWS (Negotiate about Window Size). See RFC 1073.
  * *rows* - number of rows the client's window supports.  This is set to a default of 24 and then modified if *request_naws()* is called AND the player's client supports NAWS (Negotiate about Window Size). See RFC 1073.
  * *address* - the client's remote IP address.
  * *port* - the client's port number.
  * *terminal_type* - the client's terminal type.  Defaults to 'unknown terminal' and changed if *request_terminal_type()* is called AND the player's client supports this IAC.  See RFC 779.  


## Client Methods

  * *send()* - append the given text to the client's send buffer which is actually transmitted during a TelnetServer.poll() call.  Python newlines ('\n') are automatically converted to '\r\n' (carriage return + new line) per Telnet specifications. 
  * *send_cc()* - send the given text and convert caret codes into ANSI color sequences.  See the Wiki for a list of caret codes.  See http://code.google.com/p/miniboa/wiki/CaretCodes for a list.
  * *send_wrapped* - send the given text wrapped to the user's terminal width.  Requires a prior NAWS sequence.  Caret codes are converted to ANSI sequences via *send_cc()*.     
  * *get_command()* - returns a line of user input or None (if nothing).  You can also check the property *client.cmd_ready* to see if input is available.  Carriage returns and newlines are stripped.
  * *addrport()* - returns the client's IP address and port number in the format '127.0.0.1:12345'.
  * *idle()* - returns the number of seconds since the user last typed.
  * *duration()* - returns the number of seconds since the user first connected.
  * *password_mode_on()* - request the user's client not to locally echo keystrokes.  It seems that Microsoft's telnet.exe is broken in that you cannot resume local echoing once turned off.    
  * *password_mode_off()* - request the user's client to resume local echo of keystrokes.
  * *request_do_sga()* - Request distant end to Suppress Go-Ahead.  See RFC 858.
  * *request_will_echo()* - Tell the distant end that we would like to echo their text.  See RFC 857.
  * *request_wont_echo()* - Tell the distant end that we would like to stop echoing their text. See RFC 857.
  * *request_naws()* - Request to Negotiate About Window Size.  Results will be stored in the properties *client.columns* and *client.rows*.  See RFC 1073.
  * *request_terminal_type()* - Begins the Telnet negotiations to request the terminal type from the distant end.  Result will be stored in the property client.terminal_type.  See RFC 779. See http://code.google.com/p/miniboa/wiki/TerminalTypes for a list of terminal types that I've found so far.

Keep in mind that *request_naws()* and *request_terminal_type()* are not instantaneous.  When you call them, a special byte sequence is added to the client's send buffer and wont actually transmit until the next *server.poll()* call.  Then the distant end has to reply (assuming they support them) and those replies require another *server.poll()* to process the socket's input.  
   
----

## Demo Code

* hello_demo.py - As short as it gets; creates a server and greets clients with a canned welcome. 
* handler_demo.py - demonstrates how to plug your custom *on_connect()* and *on_disconnect()* functions into the server. 
* chat_demo.py - a simple chat room that handles connections, disconnections, and echoing messages between connected clients. 

----

## License

Copyright 2010 Jim Storch.

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.

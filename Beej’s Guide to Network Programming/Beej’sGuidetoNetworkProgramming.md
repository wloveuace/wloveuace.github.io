

This is a paper that discusses networking programming in C using Sockets , Proper C knowledge is needed before digging in the paper

It is probably at its best when read by individuals who are just starting out with socket programming and are looking for a foothold. It is certainly not the complete guide to sockets programming, by any means

OG paper link: [*6761-sockhelp.pdf](https://www.cs.columbia.edu/~danr/courses/6761/Fall00/hw/pa1/6761-sockhelp.pdf)


---

## ==**Sockets initialization

There is only one header needed for sockets programming **in windows**
	```
	#include <winsock.h>
	```

Before using actual sockets functions we have to initialize  the  Winsock library for a process, allowing it to use Windows Sockets functions and 
specifying the version of Winsock required.

```
WSAStartup() - it takes a version Must be formated using MAKEWORD macro, we use version 2,2 and the returned its a struct holding info about version and pointer to a struct
```

## ==**What is socket**==

*This applies to UNIX and windows*

Unix programs do any sort of I/O, they do it by reading or writing to a file descriptor. A file descriptor is simply an integer associated with an open file. But (and here’s the catch), that file can be a network connection, a FIFO, a pipe, a terminal, a real on-the-disk file, or just about anything else. Everything in Unix is a file! So when you want to communicate with another program over the Internet you’re gonna do it through a file descriptor

in windows they are kernel objects and treated like normal file descriptors just like UNIX

```
socket() - It returns the socket descriptor, and you communicate through it using the specialized functions
```

you might be exclaiming right about now. If it’s a file descriptor, why can’t I just use the normal read() and write() calls to communicate through the socket? 
	You can, but `send` and `recv` offer much greater control over your data transmission.

In this paper we gonna be using DARPA Internet addresses (Internet Sockets), [^1]

## ==**Internet Sockets**==

There are 2 types of internet sockets, there are more but we discussing only 2, One is "**Stream Sockets**"; the other is "**Datagram Sockets**", which may hereafter be referred to as `SOCK_STREAM` and `SOCK_DGRAM`, respectively. Datagram sockets are sometimes called "connectionless sockets"

Stream sockets are reliable two-way connected communication streams. If you output two items into the socket in the order "1, 2", they will arrive in the order "1, 2" at the opposite end.

Examples of Stream sockets usage:
	Telnet
	Web browsers (on port 80)

How do stream sockets achieve this high level of data transmission quality? They use a protocol called "The Transmission Control Protocol", otherwise known as "TCP", TCP makes sure your data arrives sequentially and error-free. You may have heard "TCP" before as the better half of "TCP/IP" where "IP" stands for "Internet Protocol", IP deals primarily with Internet routing and is not generally responsible for data integrity

More about TCP: [[RFC-79355–The Transmission Control Protocol (TCP)]]
More about IP: [[RFC-79155–The Internet Protocol]]

Datagram sockets are connection less connections. One concern about it is if you send a datagram, it may arrive. It may arrive out of order. If it arrives, the data within the packet will be error-free. Datagram sockets also use IP for routing, but they don’t use TCP, they use the "User Datagram Protocol", or "UDP"

Why are they connectionless? Well, basically, it’s because you don’t have to maintain an open connection as you do with stream sockets. You just build a packet, slap an IP header on it with destination information, and send it out. No connection needed.

More about UDP: [[RFC-76855–The User Datagram Protocol (UDP)]]

the **TFTP protocol** says that for each packet that gets sent, the recipient has to send back a packet that says, "I got it!" (an "ACK" packet.) If the sender of the original packet gets no reply in, say, five seconds, he’ll re-transmit the packet until he finally gets an ACK.

The TFTP protocol controls the UDP & TCP packets to avoid losses

## ==**Network Theory**==

it’s time to talk about how networks really work

**Data Encapsulation** Basically, when a packet is born, the packet is wrapped ("encapsulated") in a header by the first protocol (say, the TFTP protocol), then the whole thing (TFTP header included) is encapsulated again by the next protocol (say, UDP), then again by the next (IP), then again by the final protocol on the hardware (physical) layer (say, Ethernet).

When another computer receives the packet, the hardware strips the Ethernet header, the kernel strips the IP and UDP headers, the TFTP program strips the TFTP header, and it finally has the data.

**Layered Network Model** This Network Model describes a system of network functionality that has many advantages over other models. For instance, you can write sockets programs that are exactly the same without caring how the data is physically transmitted (serial, thin Ethernet, AUI, whatever) because programs on lower levels deal with it for you, Those are the layers in the LNM:
	
	• Application
	• Presentation
	• Session
	• Transport
	• Network
	• Data Link
	
	• Application Layer (telnet, ftp, etc.)
	• Host-to-Host Transport Layer (TCP, UDP)
	• Internet Layer (IP and routing)
	• Network Access Layer (Ethernet, ATM, or whatever)

The Application Layer is just about as far from the physical layer as you can imagine it’s the place where users interact with the network.


## ==**Data handling & Data types**==

In sockets we use the basic data types `int , char , unsigned , etc.`

there are two byte orderings: most **significant byte** (sometimes called an "Network byte order"), or **least significant** byte first. The former is called "Host Byte Order"

Network Byte Order: Uses Most significant byte only
Host Byte Order: Can use both Most & Least

Most of the time we use Host Byte Order unless it is specified to be Network Byte Order

```
htons() - Changes from Host Byte Order to Network Byte Order
```


Our 1st struct will be the **sockaddr | sockaddr_in** Struct 
```c
struct sockaddr_in {
sa_family_t sin_family; // Address family (AF_INET)
in_port_t sin_port; // Port number (network byte order)
struct in_addr sin_addr; // IPv4 address (network byte order)
};
```

`AF_INET` represent the IPv4 for `sa_family`,  `sa_data` contains a destination address and port number for the socket. 

`sockaddr_in` is generally a parsed version of `sockaddr` and they are safe to cast between each other

``in_addr`` is a struct holding the IP address
```c
// Internet address (a structure for historical reasons)
 struct in_addr { unsigned long s_addr; };
```

`.sin_addr.S_un.S_addr` : `S_addr` is the **32-bit integer inside the union**, which holds the IP address in network byte order, and used when setting the IP address

`.sin_addr` : - `sin_addr` is the **entire `struct in_addr`**, It represents the IP address as a unit, and is used in functions

## ==**Converting Data types**==

There are two types that you can convert: short (two bytes) and long (four bytes). These functions work for the unsigned variations as well. Say you want to convert a short from Host Byte Order to Network Byte Order. Start with "h" for "host", follow it with "to", then "n" for "network", and "s" for "short": h-to-n-s, or `htons()` (read: "Host to Network Short").

Example:
	• `htons()` – "Host to Network Short"
	• `htonl()` – "Host to Network Long"
	• `ntohs()` – "Network to Host Short"
	• `ntohl()` – "Network to Host Long"

why do `sin_addr` and `sin_port` need to be in Network Byte Order in a `struct sockaddr_in`, but `sin_family` does not? The answer: `sin_addr` and `sin_port` get encapsulated in the packet at the IP and UDP layers, respectively. Thus, they must be in Network Byte Order. However, the `sin_family` field is only used by the kernel to determine what type of address the structure contains, so it must be in Host Byte Order. Also, since `sin_family` does not get sent out on the network, it can be in Host Byte Order

## ==**IP Addresses**==

let’s say you have a struct `sockaddr_in ina`, and you have an IP address "10.12.110.57" that you want to store into it.

```
inet_addr() - converts an IP address in numbers-and-dots notation into an unsigned long, returns the address in Network Byte Order already , you don’t have to call htonl().
```

`inet_addr` returns 0 on fail unlike most socket functions , return -1 on fail[^2]

When you want to reverse the process AKA Network byte to string we use:
```
inet_ntoa() - convert Network to char array[^3], it returns a pointer to a char array
```

## ==**Socket functions**==

**Socket**
```
socket() - Get the network file descriptor, it takes a Doamin(AF_INET) , a type (TCP SOCK_STREAM , UDP SOCK_DGRAM), finally a protocol set to 0. Retruns socket descripter or -1 on error 
```

The global variable `errno` is set to the error’s value (In UNIX)
In windows the error code is at a **Winsock-specific per-thread storage location** maintained by the Winsock DLL (usually **`ws2_32.dll`**).

```
WSAGetLastError() - Gets the Winsock value in Windows
```


Once you have a socket, you might have to associate that socket with a port on your local machine. The port number is used by the kernel to match an incoming packet to a certain process’s socket descriptor.

**bind**
```
bind() – Associates a socket descriptor with a specific IP address and port number. It takes three arguments: the socket descriptor, a pointer to a `sockaddr` structure that contains the address and port information, and the size of that structure.
```

*port and IP are network byte order*

There is a macro called `INADDR_ANY` that contains the IP of the machine in Network byte order, internally its actually just 0

One more port related info, all ports below 1024 are reserved and port number can be upto 65535

**connect**
```
conncet() - connect to a remote host , It takes the socket descripter , A struct to the destination port and IP (sockaddr/sockaddr_in) , sizeof the struct, Returns -1 on fail
```

The kernel will choose a local port for us when connecting, and the site we connect to will automatically get this information from us

`connect()` is a **blocking call by default**: it will wait forever until the TCP handshake succeeds **or fails**.

**Listen**
```
listen() - wait for incoming connections and handle them in some way , it takes the socket desc, an int of number of connections that can wait in connection queue and accepted later, returns -1 on fail [^4]
```

**Accept**
```
accept() - accept 1st connection waiting in queue and connect to it, it takes the socket descripter , a void pointer to empty `sockaddr/_in` struct that will hold the clients IP and Port , a size of the struct , it returns a new socket descripter that we use to send and recv messages or -1 on error
```

Old socket descriptor will still be used to listen for new connections or closed if not needed in anymore

We while loop in the Accept so that if it failed then there are no more clients waiting in queue

- By default, a listening socket is **blocking**.
- `accept()` **waits (blocks)** until a client connects.
- It **never returns `INVALID_SOCKET` just because there are no clients**—it simply waits.

**Send & Recv**
```
Send() - send message to socket descripter , it takes the destination socket descripter , a char pointer of the message , a size of the message, a flags parameter set to 0, returns amount of bytes send or -1 on fail  
```

Sometimes you tell it to send a whole gob of data and it just can’t handle it. It’ll fire off as much of the data as it can, and trust you to send the rest later. Remember, if the value returned by send() doesn’t match the value in len, it’s up to you to send the rest of the string.

if the packet is small (less than 1K or so) it will probably manage to send the whole thing all in one go

```
recv() - Reads message from socket descripter , it takes a socket descripter , a pointer to buffer block , max size of buffer block , flags set to 0 , returns number of bytes read and -1 on fail
```

![[Pasted image 20260315030624.png]]

**Sendto & recvfrom**

Since datagram sockets aren’t connected to a remote host, we need to give a The destination address before we send a packet

```
sendto() - it sends a packet to an IP , it takes a socket descripter of the sender , pointer to the message block , size of the message block , flags set to 0, sockaddr/in contating an IP and port of the destination ,  sizeof the struct , returns amount of bytes sent and -1 on fail
```

In UDP no `listen` , `accept`, `bind` is required since its connection less (kernel handles the bind on server side , It assigns an **ephemeral port** (temporary port from a predefined range).

```
recvfrom() - it recivers a packet from an IP, takes a socket descripter of the local machine , pointer to memory block buffer , max size of buffer , flags set to 0 , pointer to empty `sockaddr/in` that will recive sender's info , size of the struct , returns ammount of bytes recived and -1 on fail
```

if you connect() a datagram socket, you can then simply use send() and recv() for all your transactions (The socket itself is still a datagram socket and the packets still use UDP, but the socket interface will automatically add the destination and source information for you).

**Close & Shutdown**

You’re ready to close the connection on your socket descriptor.

```
close() - closes socket descripter , it takes a socket descripter
```

This will prevent any more reads and writes to the socket. Anyone attempting to read or write the socket on the remote end will receive an error.

Just in case you want a little more control over how the socket closes:

```
shutdown() - used to block reciveing , sending , both , it takes a socket descripter , a flag int
```

![[Pasted image 20260315030558.png]]

If you deign to use shutdown() on unconnected datagram sockets, it will simply make the socket unavailable for further send() and recv() calls (Only if you used connect on a UDP connection)

**getpeername & gethostname**

```
getpeername() - tell you who is at the other end of a connected stream socket, takes a socket descripter that we connected to, pointer to sockaddr/in that will hold info about the other side of the connection , size of the struct
```

```
gethostname() - It returns the name of the computer that your program is running on, it takes a buffer that holds the name , max size of the buffer , returns -1 on fail
```

A question i personally asked my self and its not mentioned in the paper is
**when a server info reads info why does it use the client's socket and send through it as well ?**

![[Pasted image 20260318105611.png]]


## ==**DNS**==

it stands for "Domain Name Service". In human terms we know a site with its name but computers know it by the IP which is used to send , recv , connect , ETC.

You simply pass the string that contains the machine name ("whitehouse.gov") to `gethostbyname()`, and then grab the information out of the returned `struct hostent`

```
gethostbyname() - it gives information about an name like the ip and allias names , takes a pointer to the name , returns a struct holding info about that name or NULL on error
```
![[Pasted image 20260315031807.png]]


## ==**Server and Multi Client handling**==

This section will contain code for a simple server sending "Hello World" and a Client receiving the message

Server Code: [[Server Code]]
Client Code: [[Client Code]]
On server side we:
- get a socket descriptor with socket()
- assign and ip and port with bind()
- listen for clients with listen()
- accept clients with accept()
- send and receive messages

On client side:
- get a socket descriptor with `socket()`
- specify the server’s IP and port in a `sockaddr_in` structure
- connect to the server with `connect()` (OS assigns a local port for client automatically)
	we assign the server IP and port , in client we can use INANNY we use 127.0.0.1 if local
- send and receive messages with `send()` and `recv()`
- close the socket with `closesocket()` and clean up with `WSACleanup()`

## ==**5.3. Datagram Sockets**==

On server side:
- get a socket descriptor with socket()
- assign and ip and port with bind()
- receive info from clients using `recvfrom` and we get info of client aswell `sockaddr_in`
- We can reply using `sendto` and add client's socket and client's info in `sockaddr_in`

On Client side:
- get a socket descriptor with socket()
- initialize server ip and port in `sockaddr_in`
- use `sendto` and add client's socket and server's info  in `sockaddr_in`

## ==**Advanced Techniques and concepts**==

This section talks about advanced techniques and hidden concepts in sockets so you can optimize your sockets use based on this info 

### **Blocking**

When using sockets there are functions that causes the thread to wait until there is something interesting to process For example when there is no client connected to the server , the the accept function will make the thread wait not doing any work until a client connects so it can process him

Waiting is a known concept in operating systems , In windows it genuinely means putting a thread in a wait state where it doesn't consume CPU cycles looping  for an object till its available, instead it uses a notification mechanism.

Functions that cause blocking:
	accept()
	recv()

You can disable blocking by using:
```c
fcntl(
	Socket Descripter,
	File status flags,
	File descriptor flags,
)
```
- **Socket Descriptor** is just the socket handle we wanna set to non-blocking
- **status** is the status we want to change
- **descriptor** is the open or flag we use to set status

To set a socket to non-blocking mode, use:
```c
fcntl(sockfd, F_SETFL, O_NONBLOCK);
```

> this is what we use to set the socket to non-blocking, If you try to read from a non-blocking socket and there’s no data there, it will succeed but return value is -1 and `errno` set to  `EWOULDBLOCK`, **generally not a good idea for busy waiting**

### **Synchronous I/O Multiplexing**

you are a server and you want to listen for incoming connections as well as keep reading from the connections you already have.

No problem, you say, just an accept() and a couple of recv()s. Not so fast, buster! What if you’re blocking on an accept() call? How are you going to recv() data at the same time? "Use non-blocking sockets!" No way! You don’t want to be a CPU hog. What, then?

In this situation we use a function called `select()` , it gives you the power to monitor several sockets at the same time. It’ll tell you which ones are ready for reading, which are ready for writing, and which sockets have raised exceptions

```c
select(  
	int numfds,  
	fd_set *readfds,  
	fd_set *writefds,  
	fd_set *exceptfds,  
	struct timeval *timeout  
)
```
**The function monitors "sets" of file descriptors**
- `numfds` The **highest-numbered file descriptor in any set, plus 1**
  >A **file descriptor (fd)** is essentially a **handle number** used by the operating system to refer to an open resource.

- `readfds` An optional pointer to a set of sockets to be checked for readability.
- `writefds` An optional pointer to a set of sockets to be checked for writability.
- `exceptfds` An optional pointer to a set of sockets to be checked for errors.
- `timeout` The maximum time for **select** to wait

**What are Sets ?**

A **set** (`fd_set`) is a data structure used to store multiple file descriptors (like sockets), so `select()` can monitor them all at once.

Think of it as a **collection (or list) of file descriptors**.

**How do we work with sets?**

You don’t manipulate `fd_set` directly — instead, you use macros:

```c
fd_set set;  
  
FD_ZERO(&set);        // Clear the set  
FD_SET(fd, &set);     // Add a file descriptor  
FD_CLR(fd, &set);     // Remove a file descriptor  
FD_ISSET(fd, &set);   // Check if fd is in the set
```

**How `select()` uses these sets**
1. You **add file descriptors** to the sets (read/write/except).
	- Set fd_Set to zero
	- Add file descripters to the fd_Set
2. Call `select()`.
3. After it returns:
    - The sets are **modified in-place**
    - Only the **ready descriptors remain**

>Before `select()` → sets = **what you want to monitor**    
>After `select()` → sets = **what is ready**
>With this info we can fix our main issue by continue accepting clients and if a previous client sent a message we can process it 

### ==**Professional Server**

New server Model:
- get a socket descriptor with socket()
- assign and ip and port with bind()
- listen for clients with listen()
- create a descriptor set and zero it
- Add the server socket to the descriptor set
- use select() to monitor reads in the server descriptor
- Loop through all sets in FD_SET
- check if a new client joined with FD_ISSET(server socket)
- accept client with accept()
- add client socket to FD_SET
- use select() to check writing to the client socket
- use FD_ISSET to check if client wrote something
- continue the same thing for reading , erroring ETC


---
## Footer

[^1]: There are many kinds of sockets

[^2]: (unsigned)-1 just happens to correspond to the IP address 255.255.255.255! That’s the broadcast address, Always check for the result

[^3]: inet_ntoa() ("ntoa" means "network to ascii")

[^4]: we need to call `bind() ` before we call `listen()` or the kernel will have us listening on a random port.

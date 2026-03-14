

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

`inet_addr` returns -1 on fail unlike most socket functions , return 0 on fail[^2]

When you want to reverse the process AKA Network byte to string we use:
```
inet_ntoa() - convert Network to char array[^3], it returns a pointer to a char array
```

## ==**Socket functions**==

```
socket() - Get the network file descriptor, it takes a Doamin(AF_INET) , a type (TCP SOCK_STREAM , UDP SOCK_DGRAM), finally a protocol set to 0. Retruns socket descripter or -1 on error 
```

The global variable `errno` is set to the error’s value (In UNIX)
In windows the error code is at a **Winsock-specific per-thread storage location** maintained by the Winsock DLL (usually **`ws2_32.dll`**).

```
WSAGetLastError() - Gets the Winsock value in Windows
```


Once you have a socket, you might have to associate that socket with a port on your local machine. The port number is used by the kernel to match an incoming packet to a certain process’s socket descriptor.

```
bind() – Associates a socket descriptor with a specific IP address and port number. It takes three arguments: the socket descriptor, a pointer to a `sockaddr` structure that contains the address and port information, and the size of that structure.
```

*port and IP are network byte order*

There is a macro called `INADDR_ANY` that contains the IP of the machine in Network byte order, internally its actually just 0

One more port related info, all ports below 1024 are reserved and port number can be upto 65535


```
conncet() - To be continued !!!
```

---
## Footer

[^1]: There are many kinds of sockets

[^2]: (unsigned)-1 just happens to correspond to the IP address 255.255.255.255! That’s the broadcast address, Always check for the result

[^3]: inet_ntoa() ("ntoa" means "network to ascii")

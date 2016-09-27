# program with the socket api in python
##Directory
- [Base function](#base-function)
- [C API](#c-api)
- [Some other functions](#some-other-functions)
- [Non_blocking socket](#non_blocking-socket)
	- [select](#select)

###Base function
```python
"server"
sock = socket.socket(family=AF_INET, type=SOCK_STREAM, proto=0, fileno=None)
sock.bind(address)
sock.listen([backlog])
sock.accept()

"client"
sock = socket.socket(family=AF_INET, type=SOCK_STREAM, proto=0, fileno=None)
sock.connect(address)
```
###C API
The python socket api is a straightforward transliteration of c api, so let's see
what's the c api like before we drive into it.

**socket()**
```c
int socket(int family, int type, int protocol);
/*
family : AF_INET, //use the ipv4 address(32 bit) and port(16 bit)
	     AF_INET6, //use the ipv6 address(128bit)
	     AF_LOCAL || AF_UNIX, // used for the unix/linux process communication
	     AF_ROUTE ...
type : SOCK_STREAM,
	   SOCK_DGRAM,
	   SOCK_PACKET,
	   SOCK_SEQPACKET...
protocol : IPPROTO_TCP,
		   IPPROTO_UDP,
		   IPPROTO_SCTP,
		   IPPROTO_TIPC...

The type and protocol are related, for example, the SOCK_STREAM can't combine with IPPROTO_UDP.

When the protocol is 0, it will choose the protocol related to the type automaticly.
*/
```
After we create a socket, we should call bind() to give it an address,
otherwise the system will do the work for you automaticly when connect() or listen() is called.

**bind()**
```c   
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
/*
sockfd : returned by socket

addr : protocol address pointer, the structre is different in different protocol
	ipv4:
		struct sockaddr_in {
			sa_family_t     sin_family;
			in_port_t       sin_port;
			struct in_ addr sin_addr;
		}

		struct in_addr {
			uint32_t s_addr;
		}
	ipv6:
		struct sockaddr_in6 {
			sa_family_t     sin6_family;
			in_port_t       sin6_port;
			uint32_t        sin6_flowinfo;
			struct in6_addr sin6_addr;
			uint32_t        sin6_scope_id;
		}

		struct in6_addr {
			unsigned char s6_addr[16]
		}
	unix:
		#define UNIX_PATH_MAX 108
		struct sockaddr_un {
			sa_family_t sun_family;
			char        sun_path[UNIX_PATH_MAX]
		}

addrlen : the address length
*/
```

**listen() | connect()**
```c
int listen(int sockfd, int backlog);
/*
used for server side
sockfd : same as bind()
backlog : the max number of connection
*/

int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
/*
used for client side
sockfd/addr/addrlen : same as bind()
*/
```

**accept()**
```c
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
/*
sockfd : same as bind()
addr : used for saving client address
addrlen : the address length

return a new socket descriptor, represent a connection
*/
```
In the server life, only one listening socket will be created. 
For the socket that represent a connection, once the connection broken it will be closed.

**read() | write()...**
- read() / write()
- recv() / send()
- readv() / write()
- recvmsg() / recvmsg()
- recvfrom() / sendfrom() 

the defination
```c
#include <unistd.h>
ssize_t read(int fd, void *buf, size_t count);
ssize_t write(int fd, cons void *buf, size_t count);

#include <sys/types.h>
#include <sys/socket.h>
ssize_t recv(int sockfd, void *buf, size_t len, int flags);
ssize_t send(int sockfd, const void *buf, size_t len, int flags);

ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags,
                struct sockaddr *src_addr, socklen_t *addrlen);
ssize_t sendto(int sockfd, const void *buf, size_t len, int flags,
              const struct sockaddr *dest_addr, socklen_t addrlen);

ssize_t recvmsg(int sockfd, struct msghdr *msg, int flags);
ssize_t sendmsg(int sockfd, const struct msghdr *msg, int flags);
```

**close()**
```c
#include <unistd.h>
int close(int fd);
```

Comparing with the c api, we can see the python api are similar except it's 
object-oriented.

Next, let's make a simple practice.
```python
//server
import socket
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM, socket.IPPROTO_TCP)
sock.bind(('127.0.0.1', 8080))
sock.listen(5)
while True:
	connect, address= sock.accept()
	data = bytes()
	while True:
		temp = connect.recv(1024)
		if not temp: break
		data += temp
	connect.sendall(data)
	coonect.close()

//client
import socket
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM, socket.IPPROTO_TCP)
sock.connect(('127.0.0.1', 8080))
sock.sendall(b'Hello, World')
data = sock.recv(1024)
sock.close()
print(data)
```
###Some other functions
**1. socket()**
```
sock = socket.socket(family=AF_INET, type=SOCK_STREAM, proto=0, fileno=None)
```
If fileno is specified, the other arguments are ignored, and the function will return the same socket.

The other arguments is same with c.

####2. socketpair()
```python
socket.socketpair([family,[type[,proto]]])
#Buid a pair of socket for process communication
```

raw c :

```c
int socketpair(int d, int type, int protocol, int sv[2])
//an example for process communication

#include <sys/socket.h>
#include <netinet/in.h>
#include <stdlib.h>
#include <stdio.h>
#include <unistd.h>
void err_sys(const char *errmsg);
int main(void){
	int sockfd[2];
	pid_t pid;
	if ((socketpair(AF_LOCAL, SOCK_STREAM, 0, sockfd))<0)
	err_sys("socketpair");
	if ((pid = fork()) == -1)
	err_sys("fork");
	else if (pid == 0) { 
		/* child process */
	　　char s[BUFSIZ];
		ssize_t n;
	　　close(sockfd[1]); //write port
		if ((n = read(sockfd[0], s, sizeof(s))) <0)
			err_sys("read error!\n");
		printf("read:%s\n",s);
		close(sockfd[0]);
		exit(0);
	} else if (pid > 0) { 
		/* parent process */
		char buf[] = "hello china";
		ssize_t n;
		close(sockfd[0]); //read port
		if((n = write(sockfd[1], buf, sizeof(buf)))<0)
			err_sys("write error!\n");
		close(sockfd[1]);
		wait(NULL);
	}
	return 0;
}
void err_sys(const char *errmsg){
	perror(errmsg);
	exit(1);
}
```
**3. create_connection()**
```python
socket.create_connection(address[, timeout[, source_address]])
```
This is a high level function than connect(), and you can call it to connect a server directly.
**4. fromfd()**
```python
socket.fromfd(fd, family, type, proto=0)
```
Duplicate the file descriptor fd, and set socket options.
**5. getaddrinfo()**
```python
socket.getaddrinfo(host, port, family=0, type=0, proto=0, flags=0)
```
The result is a list of 5-tuples with the following structure:
(family, type, proto, canonname, sockaddr)
**6. setsockopt() | getsockopt()**
```python
socket.getsockopt(level, optname[, buflen])
socket.setsockopt(level, optname, value)
```
raw c:
```c
#include <sys/types.h>
#include <sys/socket.h>
int setsocktopt(int s, int level, int optname, const void * optval, socklen_t optlen);

int getsockopt(int s, int level, int optname, void* optval, socklen_t* optlen);
/*
s : socket descriptor
level : SOL_SOCKET,
		IPPROTO_IP | IPPROTO_IPv6,
		IPPROTO_TCP
optname, optval, optlen
*/
```
Look up this [article](http://blog.csdn.net/chary8088/article/details/2486377) for details!
###Non_blocking socket
The socket is blocking defaultly, which means that if you read | write | accept | connect the process will block. So, we should use multi_thread or multi_process if we talk with multi_client at the same time. However, we could set the socket to non_blocking and do the work in single thread(process).
```python
socket.setblocking(flag)
flag : True | False
```
But, how can we know when the socket is readable or writeable? We need the select.
####select
python document:
> This module provides access to the select() and poll() functions available in most operating systems, devpoll() available on Solaris and derivatives, epoll() available on Linux 2.5+ and kqueue() available on most BSD.

Let's see the details. 

**1. select()**
```python
select.select(rlist, wlist, xlist[, timeout])
#rlist: wait until ready for reading
#wlist: wait until ready for writing
#xlist: wait for an “exceptional condition” 
```
A straighforward interface to the Unix select() system call.

Combine it with socket:

```python
"a simple http server"
import socket
import select

HOST = ('127.0.0.1', 80)
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.bind(HOST)
sock.listen()
rlist = []
rlist.append(sock)
header = "HTTP/1.0 200 OK\r\nContent-Type: text/plain; charset=utf-8\r\n\r\n"
body = "Aha, Hello"
response = (header + body).encode('ascii')
while True:
	rsocks, wsocks, _ = select.select(rlist, [], [])
	for rsock in rsocks:
		if rsock is sock:
			conn, addr = rsock.accept()
			rlist.append(conn) 
		else:
			print(rsock.recv(1024))
			rsock.sendall(response)
			rsock.close()
			rlist.remove(rsock)
```

**2. devpoll | epoll | kqueue | kevent**
```python
poll()    #->poll #supported on most Unix systems
devpoll() #->devpoll #Solaris
epoll()   #->epoll #linux2.5.44+
kqueue()  #->kqueue #BSD
kevent()  #->kevent #BSD
```
These objects are more effective than select, but they are plantform independently.

Just take a look at epoll.
```python
epoll.close()
epoll.closed
epoll.fileno()
epoll.fromfd(fd)
epoll.register(fd[, eventmask])
epoll.modify(fd, eventmask)
epoll.unregister(fd)
epoll.poll(timeout = 1, maxevent = -1)
```

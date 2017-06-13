# portcullis
A secure TCP/IP port reservation tool for Linux and FreeBSD.

TODO : 

1. Finish FreeBSD compatibility in sections marked FreeBSD incompatible in secure_bind.c and sprd.c (add preprocessor directives to change struct ucred).
2. Add make install and the init scripts
3. Update Documentation
4. Testing  all cases

Stretch goals :

1. Add support for UDP ports

To install: 

make install is in the process of being built. Run make install in its current state. Add spr.h to your PATH variable and include it when you need to call secure_bind(3) or secure_close(3). Check the daemon status through syslog. Read our paper (in the report section) to get full background and understanding of project and scope.

Documentation :

SECURE_CLOSE(3)

NAME

secure_close - close domain and TCP file descriptors

SYNOPSIS

#include “spr.h”
#include <unistd.h>

int secure_close(sprFDSet *closeSet);

DESCRIPTION

secure_close() closes the domain file descriptors for the connected and listening end
that is handled by the kernel. Additionally, it closes the established receiving TCP socket
descriptor for the process that called secure_bind().
The parameter for secure_close() consists of a singly linked list node of type spdFDset
*closeSet containing the domain and TCP file descriptors. The file descriptors from
closeSet will be closed and the closing status is returned.

RETURN VALUE

On success, zero is returned. On error, -1 is returned.

SEE ALSO

close(2), secure_bind(3)

SECURE_BIND(3)

NAME

secure_bind - request access to a reserved TCP/IP port under the SPR Daemon

SYNOPSIS

#include “spr.h”

int secure_bind(int portNum, sprFDSet *returnSet);

DESCRIPTION

When the SPR Daemon reserves TCP/IP ports, the socket bound to the port referred to by
portNum with bind(2) is held by the daemon. In order for a process to use the socket, they must
request it from the daemon with secure_bind(). The structure passed into secure_bind() for the
returnSet argument is defined in spr.h as :

typedef struct sprFDSocks {
int recvSock;
int udsListen;
int udsConnect;
} sprFDSet;

The purpose of this struct is to return the socket bound to the port referenced by portNum and
handles to the Unix Domain Sockets (see unix(7)) created to talk to the daemon.
secure_bind() writes the name of a unique Unix Domain Socket to the SPR Daemon’s FIFO.
secure_bind() then creates that Unix Domain Socket and calls listen(2) and accept(2) on it.
Once the daemon uses connect(2) on the other end of the active Unix Domain Socket, both
Unix Domain Sockets are filled in the sprFDSet structure. The SPR Daemon sends its
credentials (process id, user id, group id) across the Unix Domain Socket, which is then verified
by the secure_bind() implementation. secure_bind() then sends the requesting process’
credentials and the desired port, referenced by portNum, to the daemon. The credential passing
is handled by the kernel, described in unix(7). The daemon checks if the requesting process has
access to the requested port (defined in the daemon configuration file). If the process does, the
daemon sends the socket to the requesting process across the Unix Domain Socket, in a
process described in unix(7). secure_bind() then puts that file descriptor into the recvSock field
of the sprFDSet and returns. If the process did not have access to the port or the port was in
use by another process, the daemon closes the Unix Domain Socket. Any socket passed to a
requesting process can be used in any fashion that a socket created with socket(2) and bind(2)
can be used. When desiring to close the socket, the requesting process calls secure_close(3) to
finalize closing the received socket, updating the daemon’s information, and closing the Unix
Domain Sockets.

RETURN VALUE

On success, zero is returned and the fields of the structure returnSet points to are filled out. On
error, -1 is returned, errno is set appropriately, and the fields of the structure pointed to by
returnSet are undefined.

ERRORS

EINVAL: portNum is outside acceptable TCP/IP port range

EADDRINUSE : portNum is being used by another process

EACCESS: Permission to portNum denied by daemon

NOTES

FreeBSD does not use struct ucred (unix(7)) that Linux does.

BUGS

After a process calls secure_close() on a socket descriptor or dies, the port is kept from being
used until the kernel’s TIME_WAIT expires.

SEE ALSO

accept(2), bind(2), connect(2), listen(2), secure_close(3), socket(2), unix(7)

Daemon Documentation

Configuration File

The configuration file should consist of one or more reservations where each reservation is
defined by a line in the configuration file. A valid reservation consists of a set of ports, followed
by a set of intended users and a set of intended groups. These three sets are delimited by
colons, and the elements of each set are delimited by commas. Furthermore, each element of
each set can either be a single value or a range of values. As an example, the reservation line
3416,3500-3700,3410:456-470,433:220,345-350
defines a reservation on port 3416 and all ports in the range 3500-3700 for all users with user
IDs in the range 456-470 along with the user ID 433 and all groups with group IDs in the range
345-350 along with the group ID 220.
Additionally, the user and group sets can be empty. For example, the following reservation lines
are all valid:

3333: 1234:

3333::4567

3333::

Finally, it is acceptable for a port to have more than one relevant reservation line in the
configuration file. As long as a user has a user ID or group ID that is defined in at least one
reservation in the configuration, the user will have permission to bind to the port with
secure_bind().
Integration with init systems
FreeBSD:
After running “make install”, add the following line to /etc/rc.conf:
portcullis_enable=”YES”
Systemd-based systems:
After running “make install”, use the systemctl command to enable the service:
systemctl enable portcullis.service

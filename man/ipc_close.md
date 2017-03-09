# NAME

ipc_close - closes IPC connection in an orderly manner

# SYNOPSIS

**#include &lt;libdill.h>**

**int ipc_close(int **_s_**, int64_t** _deadline_**);**

# DESCRIPTION

IPC protocol is a bytestream protocol (i.e. data can be sent via **bsend()** and received via **brecv()**) for transporting data among processes on the same machine. It is an equivalent to POSIX **AF_LOCAL** sockets.

This function closes an IPC socket cleanly. Unlike **hclose()** it lets the peer know that it is shutting down and waits till the peer acknowledged the shutdown. If this terminal handshake cannot be done it returns error.

# RETURN VALUE

Zero in case of success. On error, it returns -1 and sets _errno_ to one of the values below.

# ERRORS

* **EBADF**: Invalid handle.
* **ECANCELED**: Current coroutine is being shut down.
* **ECONNRESET**: Broken connection.
* **ENOTSUP**: The operation is not supported.
* **ETIMEDOUT**: Deadline was reached.

# EXAMPLE

```c
int s = ipc_connect("/tmp/test.ipc", 10);
int rc = ipc_close(s, -1);
```
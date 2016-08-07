# fdin(3) manual page

## NAME

fdin - waits until file descriptor becomes readable

## SYNOPSIS

```
#include <libdill.h>
int fdin(int fd, int64_t deadline);
```

## DESCRIPTION

Waits until file descriptor becomes readable or until it gets into an error state. Both options cause succeful return from the function. To distinguish between them you have to do subsequent read operation on the file descriptor.

The function also exits if deadline expires.

## RETURN VALUE

The function returns 0 in case of success or -1 in case of error. In the latter case is sets `errno` to one of the following values.

## ERRORS

* `EBADF`: Not a file descriptor.
* `ECANCELED`: Current coroutine is being shut down.
* `EEXIST`: Another coroutine is already blocked on `fdin` with this file descriptor.
* `ETIMEDOUT`: Deadline expired while waiting for the file descriptor.

## EXAMPLE

```
fdin(fd);
```

## SEE ALSO

[fdclean(3)](fdclean.html)
[fdout(3)](fdout.html)

## AUTHORS

Martin Sustrik <sustrik@250bpm.com>

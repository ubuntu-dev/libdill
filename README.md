# libdill: A simple coroutine library for C

## Define a coroutine

Coroutine is a standard C function with `coroutine` modifier applied:

```
coroutine void foo(int arg1, int arg2, int arg3) {
    printf("Hello world!\n");
}
```

## Launch a coroutine

To start a coroutine use 'go' keyword. Expression returns a coroutine handle.

```
coro cr = go(foo(1, 2, 3));
```

Coroutines are cooperatively scheduled. Following functions are used as
scheduler switching points: `go()`, `yield()`, `gocancel()`, `chsend()`,
`chrecv()`, `choose()`, `msleep()` and `fdwait()`.

Keep in mind that each coroutine handle has to be closed using `gocancel()`
otherwise it will result in a memory leak.

In case of error `go()` returns `NULL` and sets `errno` to one of the following
values:

* `ENOMEM`: Not enough memory.

## Cancel a coroutine

To cancel a coroutine use `gocancel()` function. This function deallocates
resources owned by the coroutine, therefore it has to be called even for
coroutines that have already finished executing. Failure to do so results
in resource leaks.

```
int gocancel(coro *crs, int ncrs, int64_t deadline);
```

The function cancels `ncrs` coroutines in the array pointed to be `crs`.
Third argument is a deadline. For detailed information about deadlines check
"Deadlines" section of this manual.

Example:

```
coro crs[2];
crs[0] = go(fx());
crs[1] = go(fx());
int rc = gocancel(crs, 2, now() + 1000);
```

Coroutines being canceled will get grace period to run until the deadline
expires. Afterwards they will be canceled by force, i.e. all blocking calls
in the coroutine will return ECANCELED error.

Exact algorithm of this function is as follows:

1. If all the coroutines are already finished it succeeds straight away.
2. If not so it waits until the deadline. If all coroutines finish during
   this grace period the function succeeds immediately afterwards.
3. Otherwise it "kills" the remaining coroutines. What that means is that
   all function calls in the coroutine in question from that point on will
   fail with `ECANCELED` error.
4. Finally, `gocancel()` will wait until all the "killed" coroutines exit.
   It itself will succeed immediately afterwards.

In case of success `gocancel()` returns 0. In case or error it returns -1 and
sets errno to one of the following values:

* `EINVAL`: Invalid arguments.
* `ECANCELED`: Current coroutine was canceled by its owner. Coroutines in `crs`
   array are cleanly canceled even in the case of this error.

## Yield CPU to other coroutines

Occassionally, a coroutine has to perform a long calculation with no natural
coroutine switching points. Such calculation can block other coroutines from
executing for a long interval of time. To mitigate this problem call `yield()`
function once in a while.

```
int yield(void);
```

In case of success `yield()` returns 0. In case of error it returns -1 and
sets `errno` to one of the following values:

* `ECANCELED`: Current coroutine was canceled by its owner.

## Create a channel

Semantics of channel are identical to semantics of channels in Go language.

To create a channel use 'channel' function:

```
chan channel(size_t item_size, size_t buffer_size);
```

First argument is the size of one element in the channel. Second argument
is number of elements that can be stored in the channel. To get an unbuffered
channel set the second parameter to zero.

For example, this line creates a channel of int-sized elements able to hold
100 items:

```
chan ch = channel(sizeof(int), 100);
```

In case of success the function returns handle to the newly created channel.
In case of error it returns `NULL` and sets `errno` to one of the following
values:

* `ENOMEM`: Not enough memory.

## Sending to a channel

Send a message to channel:

```
int val = 42;
int rc = chsend(ch, &val, sizeof(val));
```
  
## Receiving from a channel

Receive a message from channel:

```
int val;
int rc = chrecv(ch, &val, sizeof(val));
```

## Terminating communication on a channel

Mark a channel as non-functional:

```
int val = -1;
int rc = chdone(ch, &val, sizeof(val));
```

Once this function is called, all attempts to read from the channel will
return the specified value with no blocking. All attems to send to the channel
will result in EPIPE error.

## Duplicating a channel handle

To duplicate a channel handle:

`chan ch2 = chdup(ch);`

## Closing a channel

The channel will be deallocated only after all the clones of the handle
are closed using 'chclose' function.

To close a channel:

`chclose(ch);`

## Working with multiple channels

To multiplex several channel operations use 'choose' function. Its semantics
closely mimic semantics of Golang's 'select' statement.

```
int val1;
int val2 = 42;
struct chclause clauses[] = {
    {ch1, CHRECV, &val1, sizeof(val1)},
    {ch2, CHSEND, &val2, sizeof(val2)}
};
int rc = choose(clauses, 2, now() + 30000);
switch(rc) {
case 0:
    /* val1 received from ch1 */
case 1:
    /* val2 sent to ch2 */
case -1:
    /* error */
}
```

## Deadlines

Libdill uses deadlines rather than timeouts. Deadline is a specific
point in time when the function should be canceled. To create a deadline
use 'now' function. The result it in milliseconds, thus `now() + 1000` means
one second from now onward.

If deadline is reached the function in question fails and sets `errno` to
ETIMEDOUT.

Deadline -1 means "Never time out."

Deadline 0 means: "Perform the operation immediately. If not possile return
ETIMEDOUT."

## Sleeping

To sleep until deadline expires use 'msleep' function:

`int rc = msleep(now() + 1000);`

## Waiting for file descriptors

Wait for a file descriptor:

`int rc = fdwait(fd, FDW_IN | FDW_OUT, now() + 1000);`

Libdill tries to minimise user/kernel mode transitions by caching some
information about file descriptors. After closing a file descriptor you MUST
call `fdclean(fd)` function to clean the associated cache. Also, you MUST use
`mfork` function instead of standard `fork`.

## Coroutine-local storage

A single pointer can be stored in coroutine-local storage.

To set it:

```
void *val = ...;
setcls(val);
```

To retrieve it:

```
void *val = cls();
```

## Debugging

Debugging:

```
gotrace(1); /* starts tracing */
goredump(); /* dumps info about current coroutines and channels */
```

If need be, both functions can be invoked directly from the debugger.

## License

The library is licensed under MIT/X11 license.

# backoff

[![Build Status](https://travis-ci.org/litl/backoff.png?branch=master)](https://travis-ci.org/litl/backoff?branch=master)

Function decoration for back off and retry

This module provides function decorators which can be used to wrap a
function such that it will be retried until some condition is met. It
is meant to be of use when accessing unreliable resources with the
potential for intermittent failures i.e. network resources and external
APIs. Somewhat more generally, it may also be of use for dynamically
polling resources for externally generated content.

## Examples

*Since Kenneth Reitz's [requests](http://python-requests.org) module
has become a defacto standard for HTTP clients in python, networking
examples below are written using it, but it is in no way required by
the `backoff` module.*

### @backoff.on_exception

The `on_exception` decorator is used to retry when a specified exception
is raised. Here's an example using exponential back off when any
`requests` exception is raised:

    @backoff.on_exception(backoff.expo,
                          requests.exceptions.RequestException,
                          max_tries=8)
    def get_url(url):
        return requests.get(url)

### @backoff.on_predicate

The `on_predicate` decorator is used to retry when a particular
condition is true of the return value of the target function.  This may
be useful when polling a resource for externally generated content.

Here's an example which uses a fibonacci sequence back off when the
return value of the target function is the empty list:

    @backoff.on_predicate(backoff.fibo, lambda x: x == [], max_value=13)
    def poll_for_messages(queue):
        return queue.get()

Extra keyword arguments are passed when initializing the
wait generator, so the `max_value` param above is passed as a keyword
arg when initializing the fibo generator.

When not specified, the predicate param defaults to the falsey test,
so the above can more concisely be written:

    @backoff.on_predicate(backoff.fibo, max_value=13)
    def poll_for_message(queue)
        return queue.get()

More simply, a function which continues polling every second until it
gets a non-falsey result could be defined like like this:

    @backoff.on_predicate(backoff.constant, interval=1)
    def poll_for_message(queue)
        return queue.get()

### Using multiple decorators

The `backoff` decorators may also be combined to specify different
back off behavior for different cases:

    @backoff.on_predicate(backoff.fibo, max_value=13)
    @backoff.on_exception(backoff.expo,
                          requests.exceptions.HTTPError,
                          max_tries=4)
    @backoff.on_exception(backoff.expo,
                          requests.exceptions.TimeoutError,
                          max_tries=8)
    def poll_for_message(queue):
        return queue.get()

### Event handlers

Both `backoff` decorators optionally accept event handler functions
using the keyword arguments `on_success`, `on_backoff`, and `on_giveup`.
This may be useful in reporting statistics or performing other custom
logging.

All three handler functions have the same two parameter signature. The
first argument is a tuple consisting of the function reference,
argument list, and keyword dictionary of the invocation being made. The
second argument is a count of number of tries that have occured in the
current invocation.

    def backoff_hdlr(invoc, tries):
        func, args, kwargs = invoc

        print ("Backing off after %s tries calling "
               "function %s with args %s and kwargs %s"
               % (tries, func.__name__, args, kwargs))

    @backoff.on_exception(backoff.expo,
                          requests.exceptions.RequestException,
                          on_backoff=backoff_hdlr)
    def get_url(url):
        return requests.get(url)

#### Multiple handlers per event type

In all cases, iterables of handler functions are also accepted, which
are called in turn.

#### Getting exception info

In the case of the `on_exception` decorator, all `on_backoff` and
`on_giveup` handlers are called from within the except block for the
exception being handled. Therefore exception info is available to the
handler functions via the python standard library, specifically
`sys.exc_info()` or the `traceback` module.

### Logging configuration

Errors and back off and retry attempts are logged to the 'backoff'
logger. By default, this logger is configured with a NullHandler, so
there will be nothing output unless you configure a handler.
Programmatically, this might be accomplished with something as simple
as:

    logging.getLogger('backoff').addHandler(logging.StreamHandler())

The default logging level is ERROR, which corresponds to logging anytime
`max_tries` is exceeded as well as any time a retryable exception is
raised. If you would instead like to log any type of retry, you can
set the logger level to INFO:

    logging.getLogger('backoff').setLevel(logging.INFO)


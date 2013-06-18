#gevent-fastcgi

FastCGI server implementation using **gevent** coroutine-based network library ( <http://www.gevent.org/>).
No need to monkeypatch and slow down your favourite FastCGI server in order to make it "green".

Provides simple request handler API to allow for custom request handlers.
Comes with `gevent_fastcgi.wsgi.WSGIRequestHandler` that uses standard `wsgiref.handlers.BasicCGIHandler`
for running WSGI applications.

Full support for FastCGI protocol connection multiplexing, i.e. it can serve multiple simulteneous requests
over single connection. Requires support on Web-server side.

Can be configured to fork multiple processes to better utilize multi-core CPUs.

Includes adapters for Django and frameworks that use PasteDeploy like Pylons and Pyramid to simplify depolyment.

## Installation

```
$ pip install gevent-fastcgi
```
or
```
$ easy_install gevent-fastcgi
```

## Usage

```python
from gevent_fastcgi.server import FastCGIServer
from gevent_fastcgi.wsgi import WSGIHandler
from myapp import wsgi_app

request_handler = WSGIRequestHandler(wsgi_app)
#request_handler = WSGIRefRequestHandler(wsgi_app)
#request_handler = FastCGIRequestHandler(fastcgi_app)
server = FastCGIServer(('127.0.0.1', 4000), request_handler, max_conns=1024, num_workers=16, multiplex_conn=True)

# To use UNIX-socket instead of TCP
# server = FastCGIServer('/path/to/socket', request_handler, max_conns=4096)

server.serve_forever()
```
### PasteDeploy

Gevent-fastcgi defines three `paste.server_runner` entry points. Each of them will run FastCGIServer with different request
handler implementation:

+ *wsgi*

	`gevent_fastcgi.wsgi.WSGIRequestHandler` will be used to handle requests.
	Application is expected to be a WSGI-application.

+ *wsgiref*

	`gevent_fastcgi.wsgi.WSGIRefRequestHandler` which uses standard `wsgiref.handlers`.
	Application is expected to be a WSGI-application.

+ *fastcgi*

	Application is expected to implement `gevent_fastcgi.interfaces.IRequestHandler` interface.
	It should use `request.stdin` to receive request body and `request.stdout` and/or `request.stderr` to send
	response back to Web-server.


Use it as following:
```
...
[server:main]
use = egg:gevent_fastcgi#wsgi
host = 127.0.0.1
port = 4000
# Unix-socket can be used by specifying path instead of host and port
# socket = /path/to/socket

# The following values are used in reply to Web-server on `FCGI_GET_VALUES` request
#
# Maximum allowed simulteneous connections, i.e. the size of greenlet pool used for connection handlers.
max_conns = 1024

# Fork `num_workers` child processes after socket is bound.
# Must be equal or greate than 1. No children will be actually forked if set to 1 or omitted.
num_workers = 8

# Call specified functions of gevent.monkey module before starting the server
#gevent.monkey.patch_thread = yes
#gevent.monkey.patch_time = yes
#gevent.monkey.patch_socket = yes
#gevent.monkey.patch_ssl = yes
# or
#gevent.monkey.patch_all = yes
...
```
### Django

Add `gevent_fastcgi.adapters.django` to INSTALLED_APPS of settings.py then run the following command (replace host:port with desired values)
```
python manage.py run_gevent_fastcgi host:port
```

### Custom request handlers

Starting from version 0.1.16dev It is possible to use custom request handler with `gevent_fastcgi.server.FastCGIServer`. Such a handler should implement `gevent_fastcgi.interfaces.IRequestHandler` interface and basically is just a callable that accepts single positional argument `request`. `gevent_fastcgi.wsgi` module contains two implementations of `IRequestHandler`. 

Request handler is run in separate greenlet. Request argument passed to request handler callable has the following attributes:

* _environ_ Dictionary containing request environment (NOTE: contains whatever was sent by Web-server via FCGI_PARAM stream, i.e. *does not include current OS-environ variables*)
* _stdin_ File-like object that represents request body, possibly empty
* _stdout_ File-like object that should be used by request handler to send response (including response headers)
* _stderr_ File-like object that can be used to send error information back to Web-server and sys.stderr

The following is examples of request handler implementations:

```python
import os
from zope.interface import implements
from gevent import spawn, joinall
from gevent_subprocess import Popen, PIPE
from gevent_fastcgi.interfaces import IRequestHandler


# WARNING!!!
# CGIRequestHandler is for demonstration purposes only!!!
# IT MUST NOT BE USED IN PRODUCTION ENVIRONMENT!!!

class CGIRequestHandler(object):

    implements(IRequestHandler)

    def __init__(self, root, buf_size=1024):
        self.root = os.path.abspath(root)
        self.buf_size = buf_size

    def __call__(self, request):
        script_name = request.environ['SCRIPT_NAME']
        if script_name.startswith('/'):
            script_name = script_name[1:]
        script_filename = os.path.join(self.root, script_name)

        if script_filename.startswith(self.root) and 
            os.path.isfile(script_filename) and
            os.access(script_filename, os.X_OK):
            proc = Popen(script_filename, stdin=PIPE, stdout=PIPE, stderr=PIPE)
            joinall((spawn(self.copy_stream, src, dest) for src, dest in [
                (request.stdin, proc.stdin),
                (proc.stdout, request.stdout),
                (proc.stderr, request.stderr),
                ]))
        else:
            # report an error
            request.stderr.write('Cannot locate or execute CGI-script %s' % script_filename)

            # and send a reply
            request.stdout.write('\r\n'.join((
                'Status: 404 Not Found',
                'Content-Type: text/plain',
                '',
                'No resource can be found for URI %s' % request.environ['REQUEST_URI'],
                )))
    
    def copy_stream(self, src, dest):
        buf_size = self.buf_size
        read = src.read
        write = dest.write

        while 1:
            buf = read(buf_size)
            if not buf:
                break
            write(buf)


if __name__ == '__main__':
    from gevent_fastcgi.server import FastCGIServer
    
    address = ('127.0.0.1', 8000)
    handler = CGIRequestHandler('/var/www/cgi-bin')
    server = FastCGIServer(address, handler)

    server.serve_forever()
```

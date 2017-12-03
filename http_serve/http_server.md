# 4.1 HTTP Server 的交互流程
1. TCP 链接建立：
  - TCPServer 通过 netutil.bind_sockets 创建监听套接子
  - TCPServer.add_sockets 调用 netutil.add_accept_handler 将监听套接子的可读事件添加导到 I/O 时间循环中
  - 新的客户端链接到来时，创建新的 TCP 链接，并回调 TCPServer.\_handle_connection 方法
  - TCPServer.\_handle_connection 为每个TCP 链接创建一个 IOStream 对象，用于请求和响应的读写
2. HTTP Server 启动：
  - TCPServer.\_handle_connection 调用 HTTPServer 覆盖的 handle_stream 方法，
  创建 HTTP1ServerConnection 对象，并调用 start_serving 方法，启动 HTTP 服务
  - start_serving 在每次 http 请求到来时，创建一个 HTTP1Connection(request conn)对象，
  该类表示一次完整的 htttp 请求响应过程，实现了 http 协议的解析
  - 调用 HTTPServer.start_request 方法，该方法中的 self.request_callback 对象通常时
  Application 对象，进而调用 Application.start_request 方法
  - Application.start_request 返回一个 \_RoutingDelegate 对象，这是一个
  HTTPMessageDelegate 的子类，定义处理请求的接口；在http 请求头到来时创建一个
  httputil.HTTPServerRequest 对象，并根据传递给 Application 路由规则选择合适的 HttpHandler
  - HTTP1Connection.response 以 \_RoutingDelegate 实例为参数，读取请求，并调用
  \_RoutingDelegate 相应方法进行响应
3. \_RoutingDelegate 请求路由:
  -

## 1. TCP 链接建
```python
class TCPServer(object):
    def bind(self, port, address=None, family=socket.AF_UNSPEC, backlog=128,
               reuse_port=False):
       sockets = bind_sockets(port, address=address, family=family,
                              backlog=backlog, reuse_port=reuse_port)
       if self._started:
           self.add_sockets(sockets)
       else:
           self._pending_sockets.extend(sockets)

    def add_sockets(self, sockets):
        if self.io_loop is None:
            self.io_loop = IOLoop.current()

        for sock in sockets:
            self._sockets[sock.fileno()] = sock
            add_accept_handler(sock, self._handle_connection,
                               io_loop=self.io_loop)


    def _handle_connection(self, connection, address):
        try:
            if self.ssl_options is not None:
                stream = SSLIOStream(connection, io_loop=self.io_loop,
                                     max_buffer_size=self.max_buffer_size,
                                     read_chunk_size=self.read_chunk_size)
            else:
                stream = IOStream(connection, io_loop=self.io_loop,
                                  max_buffer_size=self.max_buffer_size,
                                  read_chunk_size=self.read_chunk_size)

            future = self.handle_stream(stream, address)
            if future is not None:
                self.io_loop.add_future(gen.convert_yielded(future),
                                        lambda f: f.result())
        except Exception:
            app_log.error("Error in connection callback", exc_info=True)

    def handle_stream(self, stream, address):
        raise NotImplementedError()
```

## 2. HTTTP Server 启动
### 2.1 HTTPServer
```python
class HTTPServer(TCPServer, Configurable,
                 httputil.HTTPServerConnectionDelegate):
     def initialize(self, request_callback, no_keep_alive=False, io_loop=None,
                  xheaders=False, ssl_options=None, protocol=None,
                  decompress_request=False,
                  chunk_size=None, max_header_size=None,
                  idle_connection_timeout=None, body_timeout=None,
                  max_body_size=None, max_buffer_size=None,
                  trusted_downstream=None):
            self.request_callback = request_callback # application 对象

     def handle_stream(self, stream, address):
             context = _HTTPRequestContext(stream, address,
                                           self.protocol,
                                           self.trusted_downstream)
             conn = HTTP1ServerConnection(
                 stream, self.conn_params, context)
             self._connections.add(conn)
             conn.start_serving(self)

     def start_request(self, server_conn, request_conn):
         if isinstance(self.request_callback, httputil.HTTPServerConnectionDelegate):
             # request_callback 通常是 application 对象
             delegate = self.request_callback.start_request(server_conn, request_conn)
         else:
             delegate = _CallableAdapter(self.request_callback, request_conn)

         if self.xheaders:
             delegate = _ProxyAdapter(delegate, request_conn)

         return delegate

     def on_close(self, server_conn):
         self._connections.remove(server_conn)
```

### 2.2 HTTP1ServerConnection
```python
class HTTP1ServerConnection(object):
    """An HTTP/1.x server."""
    def __init__(self, stream, params=None, context=None):
        """
        :arg stream: an `.IOStream`
        :arg params: a `.HTTP1ConnectionParameters` or None
        :arg context: an opaque application-defined object that is accessible
            as ``connection.context``
        """
        self.stream = stream
        if params is None:
            params = HTTP1ConnectionParameters()
        self.params = params
        self.context = context
        self._serving_future = None

    @gen.coroutine
    def close(self):
        """Closes the connection.
        Returns a `.Future` that resolves after the serving loop has exited.
        """
        self.stream.close()
        # Block until the serving loop is done, but ignore any exceptions
        # (start_serving is already responsible for logging them).
        try:
            yield self._serving_future
        except Exception:
            pass

    def start_serving(self, delegate):
        """Starts serving requests on this connection.
        :arg delegate: a `.HTTPServerConnectionDelegate`
        """
        assert isinstance(delegate, httputil.HTTPServerConnectionDelegate)
        self._serving_future = self._server_request_loop(delegate)
        # Register the future on the IOLoop so its errors get logged.
        self.stream.io_loop.add_future(self._serving_future,
                                       lambda f: f.result())

    @gen.coroutine
    def _server_request_loop(self, delegate):
        try:
            while True:
                # http1 协议解析接口类
                conn = HTTP1Connection(self.stream, False,
                                       self.params, self.context)
                # 调用 http serve 的 start_request 接口，通常是
                # 调用 application.start_request 返回的 HTTPMessageDelegate 子类
                # 用于解析请求和响应
                request_delegate = delegate.start_request(self, conn)
                try:
                    #
                    ret = yield conn.read_response(request_delegate)
                except (iostream.StreamClosedError,
                        iostream.UnsatisfiableReadError):
                    return
                except _QuietException:
                    # This exception was already logged.
                    conn.close()
                    return
                except Exception:
                    gen_log.error("Uncaught exception", exc_info=True)
                    conn.close()
                    return
                if not ret:
                    return
                yield gen.moment
        finally:
            delegate.on_close(self)
```

### 2.3 Application
```python
class Application(ReversibleRouter):
    pass

class ReversibleRouter(Router):
    pass

class Router(httputil.HTTPServerConnectionDelegate):
    """Abstract router interface."""

    def start_request(self, server_conn, request_conn):
            return _RoutingDelegate(self, server_conn, request_conn)


class _RoutingDelegate(httputil.HTTPMessageDelegate):
    def __init__(self, router, server_conn, request_conn):
        self.server_conn = server_conn
        self.request_conn = request_conn
        self.delegate = None
        self.router = router  # type: Router -- Application 对象

    def headers_received(self, start_line, headers):
        # Request 对象，包含了 HTTP1Connection 负责解析HTTP 协议，读写数据
        request = httputil.HTTPServerRequest(
            connection=self.request_conn,
            server_connection=self.server_conn,
            start_line=start_line, headers=headers)

        self.delegate = self.router.find_handler(request)
        return self.delegate.headers_received(start_line, headers)

    def data_received(self, chunk):
        return self.delegate.data_received(chunk)

    def finish(self):
        self.delegate.finish()

    def on_connection_close(self):
        self.delegate.on_connection_close()
```

## 3 HTTP Server 中的抽象基类
### 3.1 HTTPServerConnectionDelegate
- 作用： 请求的处理接口

```python
class HTTPServerConnectionDelegate(object):
    """Implement this interface to handle requests from `.HTTPServer`.
    """

    def start_request(self, server_conn, request_conn):
        """This method is called by the server when a new request has started.
        :arg server_conn: is an opaque object representing the long-lived
            (e.g. tcp-level) connection.
        :arg request_conn: is a `.HTTPConnection` object for a single
            request/response exchange.
        This method should return a `.HTTPMessageDelegate`.
        """
        raise NotImplementedError()

    def on_close(self, server_conn):
        """This method is called when a connection has been closed.
        :arg server_conn: is a server connection that has previously been
            passed to ``start_request``.
        """
        pass
```


### 3.2 HTTPMessageDelegate
- 作用： 请求和响应的解析接口

```python
class HTTPMessageDelegate(object):
    """Implement this interface to handle an HTTP request or response.
    .. versionadded:: 4.0
    """
    def headers_received(self, start_line, headers):
        """Called when the HTTP headers have been received and parsed.
        :arg start_line: a `.RequestStartLine` or `.ResponseStartLine`
            depending on whether this is a client or server message.
        :arg headers: a `.HTTPHeaders` instance.
        Some `.HTTPConnection` methods can only be called during
        ``headers_received``.
        May return a `.Future`; if it does the body will not be read
        until it is done.
        """
        pass

    def data_received(self, chunk):
        """Called when a chunk of data has been received.
        May return a `.Future` for flow control.
        """
        pass

    def finish(self):
        """Called after the last chunk of data has been received."""
        pass

    def on_connection_close(self):
        """Called if the connection is closed without finishing the request.
        If ``headers_received`` is called, either ``finish`` or
        ``on_connection_close`` will be called, but not both.
        """
        pass

```

### 3.3 HTTPConnection
- 作用: 返回响应的接口

```python
class HTTPConnection(object):
    """Applications use this interface to write their responses.
    .. versionadded:: 4.0
    """
    def write_headers(self, start_line, headers, chunk=None, callback=None):
        """Write an HTTP header block.
        :arg start_line: a `.RequestStartLine` or `.ResponseStartLine`.
        :arg headers: a `.HTTPHeaders` instance.
        :arg chunk: the first (optional) chunk of data.  This is an optimization
            so that small responses can be written in the same call as their
            headers.
        :arg callback: a callback to be run when the write is complete.
        The ``version`` field of ``start_line`` is ignored.
        Returns a `.Future` if no callback is given.
        """
        raise NotImplementedError()

    def write(self, chunk, callback=None):
        """Writes a chunk of body data.
        The callback will be run when the write is complete.  If no callback
        is given, returns a Future.
        """
        raise NotImplementedError()

    def finish(self):
        """Indicates that the last body data has been written.
        """
        raise NotImplementedError()
```

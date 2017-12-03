# 4.2 请求处理与路由表
## 1. 路由过程概述
### 1.1 路由对象
1. Application 的请求路由对象是 \_ApplicationRouter 类
  - \_ApplicationRouter 继承自(ReversibleRuleRouter, ReversibleRouter, RuleRouter)
    - RuleRouter: 路由规则实现的基础类，实现了 add_rules，find_handler,
    get_target_delegate 方法
    - ReversibleRouter 抽象基类，定义 reverse_url 接口，用于url的逆向解析
    - ReversibleRuleRouter 继承自上述两个类，扩展了解析结构能够接受的对象
  - RuleRouter 最终接受 \_ApplicationRouter 实例化时传入的 handlers 参数，通过
  add_rules 方法将每条规则保存在一个 Rule 对象内，并维乎着一个包含所有 Rule 对象的列表
2. Rule(matcher, target) 对象接受两个参数
  - matcher: 是Matcher 类的子类，包含着一条路由转发的匹配规则
  - target:
    - 路由规则对应的处理类，通常是一个 RequestHandler 实例
    - target 可能是一个 \_ApplicationRouter 本身，以实现先匹配 host，再匹配路经，
    这样一个层进式匹配逻辑，实现详见下叙路由过程实现
3. Application 的路由对象
  - application 维护的路由规则保存在 self.default_router 属性中，其值是一个 \_ApplicationRouter 实例
  ```python
  self.wildcard_router = _ApplicationRouter(self, handlers)
  self.default_router = _ApplicationRouter(self, [
        Rule(AnyMatches(), self.wildcard_router)
  ])
  ```
  - self.default_router 维护着两层的路由匹配逻辑，第一层是域名匹配，第二层是路经匹配，
  此时 Rule 对像的 target 属性就是 \_ApplicationRouter 本身

### 1.2 路由过程：
1. 路由起点：
  - http serve 启动后最终会再每一个 http 链接上调用 application.start_request 方法，
  其返回一个 \_RoutingDelegate 对象，传递给 HTTP1Connection 对象，用于实现请求的处理
  - \_RoutingDelegate 接受 Application 对象作为参数，存储在 self.router 属性中，
  请求头到来时，调用 \_RoutingDelegate.headers_received 方法，进而调用
  application.find_handler 方法获取请求对应的 handler
2. 路由过程：
  - application.find_handler，调用 self.default_router.find_handler,进而调用
  RuleRouter.find_handler 方法
  - RuleRouter.find_handler 按照从 index=0 遍历规则列表，调用 rule.matcher(request)
  进行规则匹配，匹配成功返回 ruler.target, 并调用 RuleRouter.get_target_delegate 方法
  - 第一匹配时，匹配的是hostname，返回的 ruler.target 是一个 \_RoutingDelegate 实例，
  在 get_target_delegate 方法中继续调用，target.find_handler(request, \*\*target_params)
  进行路经匹配，重复上述过程；此次匹配成功后，target 是一个 RequestHandler 实例，被
  get_target_delegate 方法使用 \_CallableAdapter 包装后直接返回
  - \_CallableAdapter 包装 RequestHandler 以实现 HTTPMessageDelegate 定义的请求处理接口,
  在 body 传输完毕后调用 RequestHandler(HTTPServerRequest) 进入 RequestHandler 处理逻辑


## 3. 路由规则对象
```python
class Rule(object):
    """A routing rule."""

    def __init__(self, matcher, target, target_kwargs=None, name=None):
        """Constructs a Rule instance.
        :arg Matcher matcher: a `Matcher` instance used for determining
            whether the rule should be considered a match for a specific
            request.
        :arg target: a Rule's target (typically a ``RequestHandler`` or
            `~.httputil.HTTPServerConnectionDelegate` subclass or even a nested `Router`,
            depending on routing implementation).
        :arg dict target_kwargs: a dict of parameters that can be useful
            at the moment of target instantiation (for example, ``status_code``
            for a ``RequestHandler`` subclass). They end up in
            ``target_params['target_kwargs']`` of `RuleRouter.get_target_delegate`
            method.
        :arg str name: the name of the rule that can be used to find it
            in `ReversibleRouter.reverse_url` implementation.
        """
        if isinstance(target, str):
            # import the Module and instantiate the class
            # Must be a fully qualified name (module.ClassName)
            target = import_object(target)

        self.matcher = matcher  # Rule(AnyMatches(),
        self.target = target    # self.wildcard_router  +  handlers
        self.target_kwargs = target_kwargs if target_kwargs else {}
        self.name = name

    def reverse(self, *args):
        return self.matcher.reverse(*args)
```


## 2. 路由对象
### 2.1 \_ApplicationRouter
```python
class _ApplicationRouter(ReversibleRuleRouter):
    """Routing implementation used internally by `Application`.
    Provides a binding between `Application` and `RequestHandler`.
    This implementation extends `~.routing.ReversibleRuleRouter` in a couple of ways:
        * it allows to use `RequestHandler` subclasses as `~.routing.Rule` target and
        * it allows to use a list/tuple of rules as `~.routing.Rule` target.
        ``process_rule`` implementation will substitute this list with an appropriate
        `_ApplicationRouter` instance.
    """

    def __init__(self, application, rules=None):
        assert isinstance(application, Application)
        self.application = application
        super(_ApplicationRouter, self).__init__(rules)

    def process_rule(self, rule):
        rule = super(_ApplicationRouter, self).process_rule(rule)

        if isinstance(rule.target, (list, tuple)):
            rule.target = _ApplicationRouter(self.application, rule.target)

        return rule

    def get_target_delegate(self, target, request, **target_params):
        if isclass(target) and issubclass(target, RequestHandler):
            return self.application.get_handler_delegate(request, target, **target_params)

        return super(_ApplicationRouter, self).get_target_delegate(target, request, **target_params)
```

## 2.2 ReversibleRuleRouter
作用: 与路由规则无关，主要是添加了，url 反向解析

```python
class ReversibleRuleRouter(ReversibleRouter, RuleRouter):
    """A rule-based router that implements ``reverse_url`` method.
    Each rule added to this router may have a ``name`` attribute that can be
    used to reconstruct an original uri. The actual reconstruction takes place
    in a rule's matcher (see `Matcher.reverse`).
    """

    def __init__(self, rules=None):
        self.named_rules = {}  # type: typing.Dict[str]
        super(ReversibleRuleRouter, self).__init__(rules)

    def process_rule(self, rule):
        rule = super(ReversibleRuleRouter, self).process_rule(rule)

        if rule.name:
            if rule.name in self.named_rules:
                app_log.warning(
                    "Multiple handlers named %s; replacing previous value",
                    rule.name)
            self.named_rules[rule.name] = rule

        return rule

    def reverse_url(self, name, *args):
        if name in self.named_rules:
            return self.named_rules[name].matcher.reverse(*args)

        for rule in self.rules:
            if isinstance(rule.target, ReversibleRouter):
                reversed_url = rule.target.reverse_url(name, *args)
                if reversed_url is not None:
                    return reversed_url

        return None

a
class ReversibleRouter(Router):
    """Abstract router interface for routers that can handle named routes
    and support reversing them to original urls.
    """
    def reverse_url(self, name, *args):
        raise NotImplementedError()

```

### 2.3 RuleRouter
作用： 路由规则实现的基础类

```python
class RuleRouter(Router):
    """Rule-based router implementation."""

    def __init__(self, rules=None):
        self.rules = []  # type: typing.List[Rule]
        if rules:
            self.add_rules(rules)

    def add_rules(self, rules):
            """Appends new rules to the router.
            :arg rules: a list of Rule instances (or tuples of arguments, which are
                passed to Rule constructor).
            """
            for rule in rules:
                if isinstance(rule, (tuple, list)):
                    assert len(rule) in (2, 3, 4)
                    if isinstance(rule[0], basestring_type):
                        rule = Rule(PathMatches(rule[0]), *rule[1:])
                    else:
                        rule = Rule(*rule)

                self.rules.append(self.process_rule(rule))

    def process_rule(self, rule):
        """Override this method for additional preprocessing of each rule.
        :arg Rule rule: a rule to be processed.
        :returns: the same or modified Rule instance.
        """
        return rule

    def find_handler(self, request, **kwargs):
        for rule in self.rules:
            target_params = rule.matcher.match(request)
            if target_params is not None:
                if rule.target_kwargs:
                    target_params['target_kwargs'] = rule.target_kwargs

                delegate = self.get_target_delegate(
                    rule.target, request, **target_params)

                if delegate is not None:
                    return delegate

        return None

    def get_target_delegate(self, target, request, **target_params):
        """Returns an instance of `~.httputil.HTTPMessageDelegate` for a
        Rule's target. This method is called by `~.find_handler` and can be
        extended to provide additional target types.
        :arg target: a Rule's target.
        :arg httputil.HTTPServerRequest request: current request.
        :arg target_params: additional parameters that can be useful
            for `~.httputil.HTTPMessageDelegate` creation.
        """
        if isinstance(target, Router):
            return target.find_handler(request, **target_params)

        elif isinstance(target, httputil.HTTPServerConnectionDelegate):
            return target.start_request(request.server_connection, request.connection)

        elif callable(target):
            return _CallableAdapter(
                partial(target, **target_params), request.connection
            )

        return None
```


## 1. application 对象
```python
class Application(ReversibleRouter):
    self.wildcard_router = _ApplicationRouter(self, handlers)
    self.default_router = _ApplicationRouter(self, [
          Rule(AnyMatches(), self.wildcard_router)
    ])

    def find_handler(self, request, **kwargs):
        route = self.default_router.find_handler(request)
        if route is not None:
            return route

        if self.settings.get('default_handler_class'):
            return self.get_handler_delegate(
                request,
                self.settings['default_handler_class'],
                self.settings.get('default_handler_args', {}))

        return self.get_handler_delegate(
            request, ErrorHandler, {'status_code': 404})

      def get_target_delegate(self, target, request, **target_params):
          if isclass(target) and issubclass(target, RequestHandler):
              return self.application.get_handler_delegate(request, target, **target_params)

          return super(_ApplicationRouter, self).get_target_delegate(target, request, **target_params)

      def get_handler_delegate(self, request, target_class, target_kwargs=None,
                          path_args=None, path_kwargs=None):
     """Returns `~.httputil.HTTPMessageDelegate` that can serve a request
     for application and `RequestHandler` subclass.
     """
        return _HandlerDelegate(
            self, request, target_class, target_kwargs, path_args, path_kwargs)

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
        self.router = router  # type: Router

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

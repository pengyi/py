# flask flow
- ctx = self.request_context(environ)
  - RequestContext(environ)
   - __init__
     - request = app.request_class(environ)
     - self.url_adapter = app.create_url_adapter(self.request)
     - match_request
       -  url_rule, self.request.view_args = self.url_adapter.match(return_rule=True)
- ctx.push()
   - app_ctx.push()
   - self.session = session_interface.open_session(self.app, self.request)
- response = self.full_dispatch_request()
    - try_trigger_before_first_request_functions
    - preprocess_request
       - url_value_preprocessors((request.endpoint, request.view_args)
       - rv = before_request_funcs()
    - dispatch_request
       - self.view_functions[rule.endpoint](**req.view_args)
    - finalize_request
       - process_response(response)
          - response = after_request_funcs(response) 
          - self.session_interface.save_session(self, ctx.session, response)
- ctx.auto_pop(error)
  - app_ctx.pop(exc)


# local
l = Local()

# these are proxies
request = l('request')
user = l('user')

proxy是一个字符串， self是一个lts对象
def __call__(self, proxy):
    """Create a proxy for a name."""
    return LocalProxy(self, proxy)

# local stack
线程安全的stack， 列表存放在local对象的stack元素中
def __call__(self):
    def _lookup():
        rv = self.top
        if rv is None:
            raise RuntimeError("object unbound")
        return rv

    return LocalProxy(_lookup)

# LocalProxy
def __init__(self, local, name=None):
    object.__setattr__(self, "_LocalProxy__local", local)
    object.__setattr__(self, "__name__", name)
    if callable(local) and not hasattr(local, "__release_local__"):
        # "local" is a callable that is not an instance of Local or
        # LocalManager: mark it as a wrapped function.
        object.__setattr__(self, "__wrapped__", local)


def _get_current_object(self):
    """Return the current object.  This is useful if you want the real
    object behind the proxy at a time for performance reasons or because
    you want to pass the object into a different context.
    """
    if not hasattr(self.__local, "__release_local__"):
        return self.__local()
    try:
        return getattr(self.__local, self.__name__)
    except AttributeError:
        raise RuntimeError("no object bound to %s" % self.__name__)

def __getattr__(self, name):
    if name == "__members__":
        return dir(self._get_current_object())
    return getattr(self._get_current_object(), name)


# context locals
def _lookup_req_object(name):
    top = _request_ctx_stack.top
    if top is None:
        raise RuntimeError(_request_ctx_err_msg)
    return getattr(top, name)


def _lookup_app_object(name):
    top = _app_ctx_stack.top
    if top is None:
        raise RuntimeError(_app_ctx_err_msg)
    return getattr(top, name)


def _find_app():
    top = _app_ctx_stack.top
    if top is None:
        raise RuntimeError(_app_ctx_err_msg)
    return top.app


_request_ctx_stack = LocalStack()
_app_ctx_stack = LocalStack()
current_app = LocalProxy(_find_app) => _app_ctx_stack.top.app
request = LocalProxy(partial(_lookup_req_object, 'request')) => _request_ctx_stack.top.request
session = LocalProxy(partial(_lookup_req_object, 'session')) => _request_ctx_stack.top.session
g = LocalProxy(partial(_lookup_app_object, 'g')) => _app_ctx_stack.top.g



# resource

- _register_view(app, resource, *urls, **kwargs)
 - resource_func = self.output(resource.as_view(endpoint, *resource_class_args, **resource_class_kwargs))
  - as_view
    -  def view(*args, **kwargs):
        - view_class = resource
        - self = view.view_class(*class_args, **class_kwargs)
        - return self.dispatch_request(*args, **kwargs)
 - for decorator in self.decorators resource_func = decorator(resource_func)
 - app.add_url_rule(rule, view_func=resource_func, **kwargs)

# flow
- current_user
  - _get_user()
   - _load_user()
    - 'user_id' not in session
       - _load_from_cookie
         - user_id = decode_cookie(cookie)
         - session['user_id'] = user_id
         - reload_user
       - _load_from_request
         - user = self.request_callback(header)
         - reload_user(user)
       - _load_from_header
         - user = self.header_callback(request)
         - reload_user(user)
    - reload_user
        - user_id = session.get('user_id')
        - user = self.user_callback(user_id)
        - ctx = _request_ctx_stack.top
        - ctx.user = user

#

current_user = LocalProxy(lambda: _get_user())
current_app.login_manager._load_user()

_load_user
---

user_id = session.get('user_id')
if user_id is None:
    ctx.user = self.anonymous_user()
else:
    if self.user_callback is None:
        raise Exception(
            "No user_loader has been installed for this "
            "LoginManager. Add one with the "
            "'LoginManager.user_loader' decorator.")

    # login_manager.user_loader
    user = self.user_callback(user_id)
    if user is None:
        ctx.user = self.anonymous_user()
    else:
        ctx.user = user

is_missing_user_id = 'user_id' not in session
if is_missing_user_id:
    cookie_name = config.get('REMEMBER_COOKIE_NAME', COOKIE_NAME)
    header_name = config.get('AUTH_HEADER_NAME', AUTH_HEADER_NAME)
    has_cookie = (cookie_name in request.cookies and
                    session.get('remember') != 'clear')
    if has_cookie:
        return self._load_from_cookie(request.cookies[cookie_name])
    elif self.request_callback:
        return self._load_from_request(request)
    elif header_name in request.headers:
        return self._load_from_header(request.headers[header_name])




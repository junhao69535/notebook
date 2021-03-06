# werkzeug的Response

werkzeug提供了Response类封装响应。位于werkzeug.wrappers模块里。

## 源码分析

```python
class Response(
    BaseResponse,
    ETagResponseMixin,
    ResponseStreamMixin,
    CommonResponseDescriptorsMixin,
    WWWAuthenticateMixin,
):
    """Full featured response object implementing the following mixins:

    - :class:`ETagResponseMixin` for etag and cache control handling
    - :class:`ResponseStreamMixin` to add support for the `stream` property
    - :class:`CommonResponseDescriptorsMixin` for various HTTP descriptors
    - :class:`WWWAuthenticateMixin` for HTTP authentication support
    """
```

响应对象有以下规则：
* 响应对象是可变对象。
* 在调用freeze()之后，响应对象能被序列化和copy。
* 将同一个响应对象作用于多个WSGI响应对象是安全的。
* 响应对象支持深拷贝（deepcopy）。

可以看出，这个类是通过继承其他类类实现功能的。基本功能由BaseResponse类提供（意味着如果不需要这些混入类的功能，Reponse类和BaseResponse类功能一样），其他功能由Mixin类提供。从继承的类可以看出，我们也可以通过继承新的Mixin类来增加功能。默认下继承了4个Mixin类。
* ETagResponseMixin提供了etag和cache处理功能。
* ResponseStreamMixin提供了流处理功能。
* CommonResponseDescriptorsMixin提供了各种HTTP响应头字段描述器。
* WWWAuthenticateMixin提供了HTTP认证支持。

## BaseResponse源码分析

```python
import warnings

from .._compat import integer_types
from .._compat import string_types
from .._compat import text_type
from .._compat import to_bytes
from .._compat import to_native
from ..datastructures import Headers
from ..http import dump_cookie
from ..http import HTTP_STATUS_CODES
from ..http import remove_entity_headers
from ..urls import iri_to_uri
from ..urls import url_join
from ..utils import get_content_type
from ..wsgi import ClosingIterator
from ..wsgi import get_current_url


def _run_wsgi_app(*args):
    """This function replaces itself to ensure that the test module is not
    imported unless required.  DO NOT USE!
    """
    global _run_wsgi_app
    from ..test import run_wsgi_app as _run_wsgi_app

    return _run_wsgi_app(*args)


def _warn_if_string(iterable):
    """Helper for the response objects to check if the iterable returned
    to the WSGI server is not a string.
    """
    if isinstance(iterable, string_types):
        warnings.warn(
            "Response iterable was set to a string. This will appear to"
            " work but means that the server will send the data to the"
            " client one character at a time. This is almost never"
            " intended behavior, use 'response.data' to assign strings"
            " to the response object.",
            stacklevel=2,
        )


def _iter_encoded(iterable, charset):
    for item in iterable:
        if isinstance(item, text_type):
            # text_type从.._compat导入，默认为str，注：python2中str是字节字符串
            yield item.encode(charset)
        else:
            yield item


def _clean_accept_ranges(accept_ranges):
    if accept_ranges is True:
        return "bytes"
    elif accept_ranges is False:
        return "none"
    elif isinstance(accept_ranges, text_type):
        return to_native(accept_ranges)
    raise ValueError("Invalid accept_ranges value")


class BaseResponse(object):
    """Base response class.  The most important fact about a response object
    is that it's a regular WSGI application.  It's initialized with a couple
    of response parameters (headers, body, status code etc.) and will start a
    valid WSGI response when called with the environ and start response
    callable.

    Because it's a WSGI application itself processing usually ends before the
    actual response is sent to the server.  This helps debugging systems
    because they can catch all the exceptions before responses are started.

    Here a small example WSGI application that takes advantage of the
    response objects::

        from werkzeug.wrappers import BaseResponse as Response

        def index():
            return Response('Index page')

        def application(environ, start_response):
            path = environ.get('PATH_INFO') or '/'
            if path == '/':
                response = index()
            else:
                response = Response('Not Found', status=404)
            return response(environ, start_response)

    Like :class:`BaseRequest` which object is lacking a lot of functionality
    implemented in mixins.  This gives you a better control about the actual
    API of your response objects, so you can create subclasses and add custom
    functionality.  A full featured response object is available as
    :class:`Response` which implements a couple of useful mixins.
    # 正如BaseRequest类一样，由其他Mixin类提供一系列功能。这让我们对响应处理有更大的控制能力，
    # 我们可以通过继承这个类，并继承一些Mixin类提供所需的功能来自定义响应处理。

    To enforce a new type of already existing responses you can use the
    :meth:`force_type` method.  This is useful if you're working with different
    subclasses of response objects and you want to post process them with a
    known interface.
    # 可以通过修改force_type方法能

    Per default the response object will assume all the text data is `utf-8`
    encoded.  Please refer to :doc:`the unicode chapter </unicode>` for more
    details about customizing the behavior.
    # 默认假设处理的文本数据编码为utf-8

    Response can be any kind of iterable or string.  If it's a string it's
    considered being an iterable with one item which is the string passed.
    Headers can be a list of tuples or a
    :class:`~werkzeug.datastructures.Headers` object.

    Special note for `mimetype` and `content_type`:  For most mime types
    `mimetype` and `content_type` work the same, the difference affects
    only 'text' mimetypes.  If the mimetype passed with `mimetype` is a
    mimetype starting with `text/`, the charset parameter of the response
    object is appended to it.  In contrast the `content_type` parameter is
    always added as header unmodified.
    # 通常情况下，mimetype和content_type功能一样，如果mimetype是"text"类型的，
    # 那么会利用charset参数进行编码，而content_type只会原封不动添加到请求头。

    .. versionchanged:: 0.5
       the `direct_passthrough` parameter was added.

    :param response: a string or response iterable.
    :param status: a string with a status or an integer with the status code.
    :param headers: a list of headers or a
                    :class:`~werkzeug.datastructures.Headers` object.
    :param mimetype: the mimetype for the response.  See notice above.
    :param content_type: the content type for the response.  See notice above.
    :param direct_passthrough: if set to `True` :meth:`iter_encoded` is not
                               called before iteration which makes it
                               possible to pass special iterators through
                               unchanged (see :func:`wrap_file` for more
                               details.)
    """

    #: the charset of the response.
    charset = "utf-8"

    #: the default status if none is provided.
    default_status = 200

    #: the default mimetype if none is provided.
    default_mimetype = "text/plain"

    #: if set to `False` accessing properties on the response object will
    #: not try to consume the response iterator and convert it into a list.
    #:
    #: .. versionadded:: 0.6.2
    #:
    #:    That attribute was previously called `implicit_seqence_conversion`.
    #:    (Notice the typo).  If you did use this feature, you have to adapt
    #:    your code to the name change.
    implicit_sequence_conversion = True

    #: Should this response object correct the location header to be RFC
    #: conformant?  This is true by default.
    #:
    #: .. versionadded:: 0.8
    autocorrect_location_header = True

    #: Should this response object automatically set the content-length
    #: header if possible?  This is true by default.
    #:
    #: .. versionadded:: 0.8
    automatically_set_content_length = True

    #: Warn if a cookie header exceeds this size. The default, 4093, should be
    #: safely `supported by most browsers <cookie_>`_. A cookie larger than
    #: this size will still be sent, but it may be ignored or handled
    #: incorrectly by some browsers. Set to 0 to disable this check.
    #:
    #: .. versionadded:: 0.13
    #:
    #: .. _`cookie`: http://browsercookielimits.squawky.net/
    max_cookie_size = 4093

    def __init__(
        self,
        response=None,
        status=None,
        headers=None,
        mimetype=None,
        content_type=None,
        direct_passthrough=False,
    ):
        if isinstance(headers, Headers):
            self.headers = headers
        elif not headers:
            self.headers = Headers()
        else:
            self.headers = Headers(headers)

        if content_type is None:
            if mimetype is None and "content-type" not in self.headers:
                mimetype = self.default_mimetype
            if mimetype is not None:
                mimetype = get_content_type(mimetype, self.charset)  # 判断content_type是否需要增加charset编码
            content_type = mimetype
        if content_type is not None:
            self.headers["Content-Type"] = content_type
        if status is None:
            status = self.default_status
        if isinstance(status, integer_types):
            self.status_code = status  # 状态码
        else:
            self.status = status  # 状态码和状态信息，如"200 OK"

        self.direct_passthrough = direct_passthrough
        self._on_close = []

        # we set the response after the headers so that if a class changes
        # the charset attribute, the data is set in the correct charset.
        if response is None:
            self.response = []
        elif isinstance(response, (text_type, bytes, bytearray)):
            # 如果text_type是unicode，则会把response转成字节序列，
            # 如果automatically_set_content_length为True，
            # 顺便设置Content-Length头
            self.set_data(response)
        else:
            self.response = response

    def call_on_close(self, func):
        """Adds a function to the internal list of functions that should
        be called as part of closing down the response.  Since 0.7 this
        function also returns the function that was passed so that this
        can be used as a decorator.

        .. versionadded:: 0.6
        """
        # 添加func钩子，当response处理后，这个func会被调用
        self._on_close.append(func)
        return func

    def __repr__(self):
        if self.is_sequence:
            body_info = "%d bytes" % sum(map(len, self.iter_encoded()))
        else:
            body_info = "streamed" if self.is_streamed else "likely-streamed"
        return "<%s %s [%s]>" % (self.__class__.__name__, body_info, self.status)

    @classmethod
    def force_type(cls, response, environ=None):
        """Enforce that the WSGI response is a response object of the current
        type.  Werkzeug will use the :class:`BaseResponse` internally in many
        situations like the exceptions.  If you call :meth:`get_response` on an
        exception you will get back a regular :class:`BaseResponse` object, even
        if you are using a custom subclass.
        # 把自定义的response类的对象转换为当前的类（即BaseResponse类）的对象。如果在
        # 一个异常里调用get_response函数，则会得到一个BaseResponse对象，不管是否在使用
        # 自定义response类。

        This method can enforce a given response type, and it will also
        convert arbitrary WSGI callables into response objects if an environ
        is provided::

            # convert a Werkzeug response object into an instance of the
            # MyResponseClass subclass.
            response = MyResponseClass.force_type(response)

            # convert any WSGI application into a response object
            response = MyResponseClass.force_type(response, environ)

        This is especially useful if you want to post-process responses in
        the main dispatcher and use functionality provided by your subclass.

        Keep in mind that this will modify response objects in place if
        possible!

        :param response: a response object or wsgi application.
        :param environ: a WSGI environment object.
        :return: a response object.
        """
        if not isinstance(response, BaseResponse):
            if environ is None:
                raise TypeError(
                    "cannot convert WSGI application into response"
                    " objects without an environ"
                )
            response = BaseResponse(*_run_wsgi_app(response, environ))
        response.__class__ = cls
        return response

    @classmethod
    def from_app(cls, app, environ, buffered=False):
        """Create a new response object from an application output.  This
        works best if you pass it an application that returns a generator all
        the time.  Sometimes applications may use the `write()` callable
        returned by the `start_response` function.  This tries to resolve such
        edge cases automatically.  But if you don't get the expected output
        you should set `buffered` to `True` which enforces buffering.

        :param app: the WSGI application to execute.
        :param environ: the WSGI environment to execute against.
        :param buffered: set to `True` to enforce buffering.
        :return: a response object.
        """
        # 从app获取一个响应。如果需要缓存，即list或tuple，则buffered传True，
        # 否则返回一个generator，无缓存
        return cls(*_run_wsgi_app(app, environ, buffered))

    def _get_status_code(self):
        return self._status_code

    def _set_status_code(self, code):
        self._status_code = code
        try:
            self._status = "%d %s" % (code, HTTP_STATUS_CODES[code].upper())
        except KeyError:
            self._status = "%d UNKNOWN" % code

    status_code = property(
        _get_status_code, _set_status_code, doc="The HTTP Status code as number"
    )
    del _get_status_code, _set_status_code

    def _get_status(self):
        return self._status

    def _set_status(self, value):
        try:
            self._status = to_native(value)
        except AttributeError:
            raise TypeError("Invalid status argument")

        try:
            self._status_code = int(self._status.split(None, 1)[0])
        except ValueError:
            self._status_code = 0
            self._status = "0 %s" % self._status
        except IndexError:
            raise ValueError("Empty status argument")

    status = property(_get_status, _set_status, doc="The HTTP Status code")
    del _get_status, _set_status

    def get_data(self, as_text=False):
        """The string representation of the request body.  Whenever you call
        this property the request iterable is encoded and flattened.  This
        can lead to unwanted behavior if you stream big data.

        This behavior can be disabled by setting
        :attr:`implicit_sequence_conversion` to `False`.

        If `as_text` is set to `True` the return value will be a decoded
        unicode string.

        .. versionadded:: 0.9
        """
        # 获取响应体
        self._ensure_sequence()
        rv = b"".join(self.iter_encoded())
        if as_text:
            rv = rv.decode(self.charset)
        return rv

    def set_data(self, value):
        """Sets a new string as response.  The value set must either by a
        unicode or bytestring.  If a unicode string is set it's encoded
        automatically to the charset of the response (utf-8 by default).
        # 如果value是unicode，会自动以charset编码进行编码
        .. versionadded:: 0.9
        """
        # 设置响应体
        # if an unicode string is set, it's encoded directly so that we
        # can set the content length
        if isinstance(value, text_type):
            value = value.encode(self.charset)
        else:
            value = bytes(value)
        self.response = [value]
        if self.automatically_set_content_length:
            self.headers["Content-Length"] = str(len(value))

    data = property(
        get_data,
        set_data,
        doc="A descriptor that calls :meth:`get_data` and :meth:`set_data`.",
    )

    def calculate_content_length(self):
        """Returns the content length if available or `None` otherwise."""
        try:
            self._ensure_sequence()
        except RuntimeError:
            return None
        return sum(len(x) for x in self.iter_encoded())

    def _ensure_sequence(self, mutable=False):
        """This method can be called by methods that need a sequence.  If
        `mutable` is true, it will also ensure that the response sequence
        is a standard Python list.

        .. versionadded:: 0.6
        """
        if self.is_sequence:
            # if we need a mutable object, we ensure it's a list.
            if mutable and not isinstance(self.response, list):
                self.response = list(self.response)
            return
        if self.direct_passthrough:
            raise RuntimeError(
                "Attempted implicit sequence conversion but the"
                " response object is in direct passthrough mode."
            )
        if not self.implicit_sequence_conversion:
            raise RuntimeError(
                "The response object required the iterable to be a"
                " sequence, but the implicit conversion was disabled."
                " Call make_sequence() yourself."
            )
        self.make_sequence()

    def make_sequence(self):
        """Converts the response iterator in a list.  By default this happens
        automatically if required.  If `implicit_sequence_conversion` is
        disabled, this method is not automatically called and some properties
        might raise exceptions.  This also encodes all the items.

        .. versionadded:: 0.6
        """
        # 把response转换成列表，并调用close方法进行清理
        if not self.is_sequence:
            # if we consume an iterable we have to ensure that the close
            # method of the iterable is called if available when we tear
            # down the response
            close = getattr(self.response, "close", None)
            self.response = list(self.iter_encoded())
            if close is not None:
                self.call_on_close(close)

    def iter_encoded(self):
        """Iter the response encoded with the encoding of the response.
        If the response object is invoked as WSGI application the return
        value of this method is used as application iterator unless
        :attr:`direct_passthrough` was activated.
        """
        # 对response进行charset编码
        if __debug__:
            _warn_if_string(self.response)
        # Encode in a separate function so that self.response is fetched
        # early.  This allows us to wrap the response with the return
        # value from get_app_iter or iter_encoded.
        return _iter_encoded(self.response, self.charset)

    def set_cookie(
        self,
        key,
        value="",
        max_age=None,
        expires=None,
        path="/",
        domain=None,
        secure=False,
        httponly=False,
        samesite=None,
    ):
        """Sets a cookie. The parameters are the same as in the cookie `Morsel`
        object in the Python standard library but it accepts unicode data, too.

        A warning is raised if the size of the cookie header exceeds
        :attr:`max_cookie_size`, but the header will still be set.

        :param key: the key (name) of the cookie to be set.
        :param value: the value of the cookie.
        :param max_age: should be a number of seconds, or `None` (default) if
                        the cookie should last only as long as the client's
                        browser session.
        :param expires: should be a `datetime` object or UNIX timestamp.
        :param path: limits the cookie to a given path, per default it will
                     span the whole domain.
        :param domain: if you want to set a cross-domain cookie.  For example,
                       ``domain=".example.com"`` will set a cookie that is
                       readable by the domain ``www.example.com``,
                       ``foo.example.com`` etc.  Otherwise, a cookie will only
                       be readable by the domain that set it.
        :param secure: If `True`, the cookie will only be available via HTTPS
        :param httponly: disallow JavaScript to access the cookie.  This is an
                         extension to the cookie standard and probably not
                         supported by all browsers.
        :param samesite: Limits the scope of the cookie such that it will only
                         be attached to requests if those requests are
                         "same-site".
        """
        # 调用dump_cookie设置Set-Cookie响应头
        self.headers.add(
            "Set-Cookie",
            dump_cookie(
                key,
                value=value,
                max_age=max_age,
                expires=expires,
                path=path,
                domain=domain,
                secure=secure,
                httponly=httponly,
                charset=self.charset,
                max_size=self.max_cookie_size,
                samesite=samesite,
            ),
        )

    def delete_cookie(self, key, path="/", domain=None):
        """Delete a cookie.  Fails silently if key doesn't exist.

        :param key: the key (name) of the cookie to be deleted.
        :param path: if the cookie that should be deleted was limited to a
                     path, the path has to be defined here.
        :param domain: if the cookie that should be deleted was limited to a
                       domain, that domain has to be defined here.
        """
        self.set_cookie(key, expires=0, max_age=0, path=path, domain=domain)

    @property
    def is_streamed(self):
        """If the response is streamed (the response is not an iterable with
        a length information) this property is `True`.  In this case streamed
        means that there is no information about the number of iterations.
        This is usually `True` if a generator is passed to the response object.

        This is useful for checking before applying some sort of post
        filtering that should not take place for streamed responses.
        """
        try:
            len(self.response)
        except (TypeError, AttributeError):
            return True
        return False

    @property
    def is_sequence(self):
        """If the iterator is buffered, this property will be `True`.  A
        response object will consider an iterator to be buffered if the
        response attribute is a list or tuple.

        .. versionadded:: 0.6
        """
        # 判断response是否缓存，如果response是tuple或list，则是缓存
        return isinstance(self.response, (tuple, list))

    def close(self):
        """Close the wrapped response if possible.  You can also use the object
        in a with statement which will automatically close it.

        .. versionadded:: 0.9
           Can now be used in a with statement.
        """
        if hasattr(self.response, "close"):
            self.response.close()
        for func in self._on_close:
            # 调用close钩子函数
            func()

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_value, tb):
        self.close()

    def freeze(self):
        """Call this method if you want to make your response object ready for
        being pickled.  This buffers the generator if there is one.  It will
        also set the `Content-Length` header to the length of the body.

        .. versionchanged:: 0.6
           The `Content-Length` header is now set.
        """
        # we explicitly set the length to a list of the *encoded* response
        # iterator.  Even if the implicit sequence conversion is disabled.
        self.response = list(self.iter_encoded())
        self.headers["Content-Length"] = str(sum(map(len, self.response)))

    def get_wsgi_headers(self, environ):
        """This is automatically called right before the response is started
        and returns headers modified for the given environment.  It returns a
        copy of the headers from the response with some modifications applied
        if necessary.

        For example the location header (if present) is joined with the root
        URL of the environment.  Also the content length is automatically set
        to zero here for certain status codes.

        .. versionchanged:: 0.6
           Previously that function was called `fix_headers` and modified
           the response object in place.  Also since 0.6, IRIs in location
           and content-location headers are handled properly.

           Also starting with 0.6, Werkzeug will attempt to set the content
           length if it is able to figure it out on its own.  This is the
           case if all the strings in the response iterable are already
           encoded and the iterable is buffered.

        :param environ: the WSGI environment of the request.
        :return: returns a new :class:`~werkzeug.datastructures.Headers`
                 object.
        """
        headers = Headers(self.headers)
        location = None
        content_location = None
        content_length = None
        status = self.status_code

        # iterate over the headers to find all values in one go.  Because
        # get_wsgi_headers is used each response that gives us a tiny
        # speedup.
        for key, value in headers:
            ikey = key.lower()
            if ikey == u"location":
                location = value
            elif ikey == u"content-location":
                content_location = value
            elif ikey == u"content-length":
                content_length = value

        # make sure the location header is an absolute URL
        if location is not None:
            old_location = location
            if isinstance(location, text_type):
                # Safe conversion is necessary here as we might redirect
                # to a broken URI scheme (for instance itms-services).
                location = iri_to_uri(location, safe_conversion=True)

            if self.autocorrect_location_header:
                current_url = get_current_url(environ, strip_querystring=True)
                if isinstance(current_url, text_type):
                    current_url = iri_to_uri(current_url)
                location = url_join(current_url, location)
            if location != old_location:
                headers["Location"] = location

        # make sure the content location is a URL
        if content_location is not None and isinstance(content_location, text_type):
            headers["Content-Location"] = iri_to_uri(content_location)

        if 100 <= status < 200 or status == 204:
            # Per section 3.3.2 of RFC 7230, "a server MUST NOT send a
            # Content-Length header field in any response with a status
            # code of 1xx (Informational) or 204 (No Content)."
            headers.remove("Content-Length")
        elif status == 304:
            remove_entity_headers(headers)

        # if we can determine the content length automatically, we
        # should try to do that.  But only if this does not involve
        # flattening the iterator or encoding of unicode strings in
        # the response.  We however should not do that if we have a 304
        # response.
        if (
            self.automatically_set_content_length
            and self.is_sequence
            and content_length is None
            and status not in (204, 304)
            and not (100 <= status < 200)
        ):
            try:
                content_length = sum(len(to_bytes(x, "ascii")) for x in self.response)
            except UnicodeError:
                # aha, something non-bytestringy in there, too bad, we
                # can't safely figure out the length of the response.
                pass
            else:
                headers["Content-Length"] = str(content_length)

        return headers

    def get_app_iter(self, environ):
        """Returns the application iterator for the given environ.  Depending
        on the request method and the current status code the return value
        might be an empty response rather than the one from the response.

        If the request method is `HEAD` or the status code is in a range
        where the HTTP specification requires an empty response, an empty
        iterable is returned.

        .. versionadded:: 0.6

        :param environ: the WSGI environment of the request.
        :return: a response iterable.
        """
        # 根据请求信息environ，获取响应体，以可迭代对象形式返回
        status = self.status_code
        if (
            environ["REQUEST_METHOD"] == "HEAD"
            or 100 <= status < 200
            or status in (204, 304)
        ):
            iterable = ()
        elif self.direct_passthrough:
            if __debug__:
                _warn_if_string(self.response)
            return self.response
        else:
            iterable = self.iter_encoded()
        return ClosingIterator(iterable, self.close)

    def get_wsgi_response(self, environ):
        """Returns the final WSGI response as tuple.  The first item in
        the tuple is the application iterator, the second the status and
        the third the list of headers.  The response returned is created
        specially for the given environment.  For example if the request
        method in the WSGI environment is ``'HEAD'`` the response will
        be empty and only the headers and status code will be present.

        .. versionadded:: 0.6

        :param environ: the WSGI environment of the request.
        :return: an ``(app_iter, status, headers)`` tuple.
        """
        # 返回（响应体，响应行，响应头）
        headers = self.get_wsgi_headers(environ)
        app_iter = self.get_app_iter(environ)
        return app_iter, self.status, headers.to_wsgi_list()

    def __call__(self, environ, start_response):
        """Process this response as WSGI application.

        :param environ: the WSGI environment.
        :param start_response: the response callable provided by the WSGI
                               server.
        :return: an application iterator
        """
        app_iter, status, headers = self.get_wsgi_response(environ)
        start_response(status, headers)  # 调用WSGI服务器提供的回调函数
        return app_iter

```

### 分析

典型的用法：

```python
from werkzeug.wrappers import Response
def application(environ, start_response):
    response = Response("Hello World!", mimetype="text/plain")
    return response(environ, start_response)
```

这里通过`Response(response="Hello World!", mimetype="text/plain")`实例化了一个Response对象。然后通过调用`response(environ, start_response)`结束处理，返回数据。调用`response(environ, start_response)`实际上是调用__call__(self, environ, start_response)方法。

```python
def __call__(self, environ, start_response):
    app_iter, status, headers = self.get_wsgi_response(environ)
    start_response(status, headers)  # 调用WSGI服务器提供的回调函数
    return app_iter
```

在__call__方法里，调用了get_wsgi_response方法获取了`(app_iter, status, headers)`，即（响应体，响应行，响应头），然后调用WSGI服务器提供的回调函数start_response把响应行和响应头传递过去。了解WSGI服务器的同学都知道WSGI服务器是如何返回响应的：先发送响应体，然后发送响应头，最后通过socket的wfile发送响应体。这就意味着我们必须先调用回调函数start_response之后才能返回响应体。

下面看一下get_wsgi_response是如何处理的：

```python
def get_wsgi_response(self, environ):
        # 返回（响应体，响应行，响应头）
        headers = self.get_wsgi_headers(environ)
        app_iter = self.get_app_iter(environ)
        return app_iter, self.status, headers.to_wsgi_list()
```

可以看出，实际上是调用get_wsgi_headers方法获取响应头，调用get_app_iter获取响应体。

先分析get_wsgi_headers方法：

```python
def get_wsgi_headers(self, environ):
    headers = Headers(self.headers)
    location = None
    content_location = None
    content_length = None
    status = self.status_code

    for key, value in headers:
        ikey = key.lower()
        if ikey == u"location":
            location = value
        elif ikey == u"content-location":
            content_location = value
        elif ikey == u"content-length":
            content_length = value

    if location is not None:
        old_location = location
        if isinstance(location, text_type):
            location = iri_to_uri(location, safe_conversion=True)

        if self.autocorrect_location_header:
            current_url = get_current_url(environ, strip_querystring=True)
            if isinstance(current_url, text_type):
                current_url = iri_to_uri(current_url)
            location = url_join(current_url, location)
        if location != old_location:
            headers["Location"] = location

    if content_location is not None and isinstance(content_location, text_type):
        headers["Content-Location"] = iri_to_uri(content_location)

    if 100 <= status < 200 or status == 204:
        headers.remove("Content-Length")
    elif status == 304:
        remove_entity_headers(headers)

    if (
        self.automatically_set_content_length
        and self.is_sequence
        and content_length is None
        and status not in (204, 304)
        and not (100 <= status < 200)
    ):
        try:
            content_length = sum(len(to_bytes(x, "ascii")) for x in self.response)
        except UnicodeError:
            pass
        else:
            headers["Content-Length"] = str(content_length)

    return headers
```

这么长的代码，实际上就是根据请求信息environ设置响应头。

再分析get_app_iter方法：

```python
def get_app_iter(self, environ):
    # 根据请求信息environ，获取响应体，以可迭代对象形式返回
    status = self.status_code
    if (
        environ["REQUEST_METHOD"] == "HEAD"
        or 100 <= status < 200
        or status in (204, 304)
    ):
        iterable = ()
    elif self.direct_passthrough:
        if __debug__:
            _warn_if_string(self.response)
        return self.response
    else:
        iterable = self.iter_encoded()
    return ClosingIterator(iterable, self.close)
```

这里的代码也很好理解，就是根据请求信息获取响应体，如果请求方法为HEAD，则直接把响应体置空。否则，把响应体包装成可迭代对象返回。最后的ClosingIterator的作用就是返回请求体之前调用hook函数，这就意味着可以提供中间件再返回之前进行处理。

### 总结
可以看出，Response就是根据请求信息设置响应行、响应头和响应体。然后包装成Response对象。


## ETagResponseMixin源码分析
### 分析

```python
from .._compat import string_types
from .._internal import _get_environ
from ..datastructures import ContentRange
from ..datastructures import RequestCacheControl
from ..datastructures import ResponseCacheControl
from ..http import generate_etag
from ..http import http_date
from ..http import is_resource_modified
from ..http import parse_cache_control_header
from ..http import parse_content_range_header
from ..http import parse_date
from ..http import parse_etags
from ..http import parse_if_range_header
from ..http import parse_range_header
from ..http import quote_etag
from ..http import unquote_etag
from ..utils import cached_property
from ..utils import header_property
from ..wrappers.base_response import _clean_accept_ranges
from ..wsgi import _RangeWrapper


class ETagResponseMixin(object):
    """Adds extra functionality to a response object for etag and cache
    handling.  This mixin requires an object with at least a `headers`
    object that implements a dict like interface similar to
    :class:`~werkzeug.datastructures.Headers`.
    # 这个混入类需要原类存在一个类似headers对象

    If you want the :meth:`freeze` method to automatically add an etag, you
    have to mixin this method before the response base class.  The default
    response class does not do that.
    # 如果想自动添加etag到响应，则继承这个类时要放在response基类后面。
    """

    @property
    def cache_control(self):
        """The Cache-Control general-header field is used to specify
        directives that MUST be obeyed by all caching mechanisms along the
        request/response chain.
        """

        def on_update(cache_control):
            if not cache_control and "cache-control" in self.headers:
                del self.headers["cache-control"]
            elif cache_control:
                self.headers["Cache-Control"] = cache_control.to_header()

        return parse_cache_control_header(
            self.headers.get("cache-control"), on_update, ResponseCacheControl
        )

    def _wrap_response(self, start, length):
        """Wrap existing Response in case of Range Request context."""
        if self.status_code == 206:
            # 根据请求返回对应范围的数据，通常用于断点续传或把大文件拆开小文件传输
            self.response = _RangeWrapper(self.response, start, length)

    def _is_range_request_processable(self, environ):
        """Return ``True`` if `Range` header is present and if underlying
        resource is considered unchanged when compared with `If-Range` header.
        """
        # 判断是否提供了Range请求头和资产是否有变化（通过对比If-Range头）
        return (
            "HTTP_IF_RANGE" not in environ
            or not is_resource_modified(
                environ,
                self.headers.get("etag"),
                None,
                self.headers.get("last-modified"),
                ignore_if_range=False,
            )
        ) and "HTTP_RANGE" in environ

    def _process_range_request(self, environ, complete_length=None, accept_ranges=None):
        """Handle Range Request related headers (RFC7233).  If `Accept-Ranges`
        header is valid, and Range Request is processable, we set the headers
        as described by the RFC, and wrap the underlying response in a
        RangeWrapper.

        Returns ``True`` if Range Request can be fulfilled, ``False`` otherwise.
        如果Range请求能完成，返回True，否则返回False
        :raises: :class:`~werkzeug.exceptions.RequestedRangeNotSatisfiable`
                 if `Range` header could not be parsed or satisfied.
        """
        from ..exceptions import RequestedRangeNotSatisfiable

        if accept_ranges is None:
            return False
        self.headers["Accept-Ranges"] = accept_ranges
        if not self._is_range_request_processable(environ) or complete_length is None:
            return False
        parsed_range = parse_range_header(environ.get("HTTP_RANGE"))
        if parsed_range is None:
            raise RequestedRangeNotSatisfiable(complete_length)
        range_tuple = parsed_range.range_for_length(complete_length)
        content_range_header = parsed_range.to_content_range_header(complete_length)
        if range_tuple is None or content_range_header is None:
            raise RequestedRangeNotSatisfiable(complete_length)
        content_length = range_tuple[1] - range_tuple[0]
        # Be sure not to send 206 response
        # if requested range is the full content.
        if content_length != complete_length:
            self.headers["Content-Length"] = content_length
            self.content_range = content_range_header
            self.status_code = 206
            self._wrap_response(range_tuple[0], content_length)
            return True
        return False

    def make_conditional(
        self, request_or_environ, accept_ranges=False, complete_length=None
    ):
        """Make the response conditional to the request.  This method works
        best if an etag was defined for the response already.  The `add_etag`
        method can be used to do that.  If called without etag just the date
        header is set.

        This does nothing if the request method in the request or environ is
        anything but GET or HEAD.

        For optimal performance when handling range requests, it's recommended
        that your response data object implements `seekable`, `seek` and `tell`
        methods as described by :py:class:`io.IOBase`.  Objects returned by
        :meth:`~werkzeug.wsgi.wrap_file` automatically implement those methods.

        It does not remove the body of the response because that's something
        the :meth:`__call__` function does for us automatically.

        Returns self so that you can do ``return resp.make_conditional(req)``
        but modifies the object in-place.

        :param request_or_environ: a request object or WSGI environment to be
                                   used to make the response conditional
                                   against.
        :param accept_ranges: This parameter dictates the value of
                              `Accept-Ranges` header. If ``False`` (default),
                              the header is not set. If ``True``, it will be set
                              to ``"bytes"``. If ``None``, it will be set to
                              ``"none"``. If it's a string, it will use this
                              value.
        :param complete_length: Will be used only in valid Range Requests.
                                It will set `Content-Range` complete length
                                value and compute `Content-Length` real value.
                                This parameter is mandatory for successful
                                Range Requests completion.
        :raises: :class:`~werkzeug.exceptions.RequestedRangeNotSatisfiable`
                 if `Range` header could not be parsed or satisfied.
        """
        environ = _get_environ(request_or_environ)
        if environ["REQUEST_METHOD"] in ("GET", "HEAD"):
            # if the date is not in the headers, add it now.  We however
            # will not override an already existing header.  Unfortunately
            # this header will be overriden by many WSGI servers including
            # wsgiref.
            if "date" not in self.headers:
                self.headers["Date"] = http_date()
            accept_ranges = _clean_accept_ranges(accept_ranges)
            is206 = self._process_range_request(environ, complete_length, accept_ranges)
            if not is206 and not is_resource_modified(
                environ,
                self.headers.get("etag"),
                None,
                self.headers.get("last-modified"),
            ):
                if parse_etags(environ.get("HTTP_IF_MATCH")):
                    self.status_code = 412
                else:
                    self.status_code = 304
            if (
                self.automatically_set_content_length
                and "content-length" not in self.headers
            ):
                length = self.calculate_content_length()
                if length is not None:
                    self.headers["Content-Length"] = length
        return self

    def add_etag(self, overwrite=False, weak=False):
        """Add an etag for the current response if there is none yet."""
        if overwrite or "etag" not in self.headers:
            # generate_etag利用md5对响应体生成etag
            self.set_etag(generate_etag(self.get_data()), weak)

    def set_etag(self, etag, weak=False):
        """Set the etag, and override the old one if there was one."""
        self.headers["ETag"] = quote_etag(etag, weak)

    def get_etag(self):
        """Return a tuple in the form ``(etag, is_weak)``.  If there is no
        ETag the return value is ``(None, None)``.
        """
        return unquote_etag(self.headers.get("ETag"))

    def freeze(self, no_etag=False):
        """Call this method if you want to make your response object ready for
        pickeling.  This buffers the generator if there is one.  This also
        sets the etag unless `no_etag` is set to `True`.
        """
        if not no_etag:
            self.add_etag()
        super(ETagResponseMixin, self).freeze()

    accept_ranges = header_property(
        "Accept-Ranges",
        doc="""The `Accept-Ranges` header. Even though the name would
        indicate that multiple values are supported, it must be one
        string token only.

        The values ``'bytes'`` and ``'none'`` are common.

        .. versionadded:: 0.7""",
    )

    def _get_content_range(self):
        def on_update(rng):
            if not rng:
                del self.headers["content-range"]
            else:
                self.headers["Content-Range"] = rng.to_header()

        rv = parse_content_range_header(self.headers.get("content-range"), on_update)
        # always provide a content range object to make the descriptor
        # more user friendly.  It provides an unset() method that can be
        # used to remove the header quickly.
        if rv is None:
            rv = ContentRange(None, None, None, on_update=on_update)
        return rv

    def _set_content_range(self, value):
        if not value:
            del self.headers["content-range"]
        elif isinstance(value, string_types):
            self.headers["Content-Range"] = value
        else:
            self.headers["Content-Range"] = value.to_header()

    content_range = property(
        _get_content_range,
        _set_content_range,
        doc="""The ``Content-Range`` header as
        :class:`~werkzeug.datastructures.ContentRange` object. Even if
        the header is not set it wil provide such an object for easier
        manipulation.

        .. versionadded:: 0.7""",
    )
    del _get_content_range, _set_content_range

```

### 测试
etag的请求流程，浏览器发送请求，服务端查看请求头有没有If-None-Match字段，如果没有，则生成一个tag，然后作为响应头Etag:etag返回。浏览器会把这个资源和etag缓存下来，当浏览器下一次发送请求时会把这个etag放在If-None-Match请求头部字段。服务端收到If-None-Match请求头部字段，判断是否和之前生成的一样，如果是，返回空响应体，状态码为304。浏览器收到304就知道可以使用本地缓存。

```python
#!coding=utf-8
import hashlib
from werkzeug.wrappers import Response
from werkzeug.serving import run_simple


def app(environ, start_response):
    print environ
    if environ.get("PATH_INFO") == "/":
        response = Response('<html>'
                            '<a href="http://localhost:5000/test">bin</a>'
                            '<script type="text/javascript" src="http://localhost:5000/temp_js.js"></script>'
                            '</html>',
                            headers={"Content-type": "text/html"})
    elif environ.get("PATH_INFO") == "/temp_js.js":
        etag = environ.get("HTTP_IF_NONE_MATCH")  # werkzeug会把If-None-Match改写成HTTP_IF_NONE_MATCH
        if etag:
            etag = etag.replace('"', '')  # 而且这里貌似有个bug，从If-None-Match得到的etag多包了一层双引号""。
        with open("temp_js.js", "rb") as f:
            data = f.read()
        if etag == hashlib.md5(data).hexdigest():
            print u"进入缓存"
            response = Response()
            response.status_code = 304
        else:
            print u"没有进入缓存"
            response = Response(mimetype="application/javascript")
            with open("temp_js.js", "rb") as f:
                response.stream.write(f.read())
            response.add_etag()
    else:
        response = Response()
    return response(environ, start_response)


if __name__ == "__main__":
    run_simple("localhost", 5000, app, threaded=True)
```

### 总结
这个Mixin类提供了Etag和cache的功能，在Etag方面提供了add_etag和get_etag接口。在cache方面提供了cache_control接口。


## ResponseStreamMixin源码分析

```python
class ResponseStreamMixin(object):
    """Mixin for :class:`BaseRequest` subclasses.  Classes that inherit from
    this mixin will automatically get a :attr:`stream` property that provides
    a write-only interface to the response iterable.
    """
    # 提供了一个只写接口给可迭代响应对象

    @cached_property
    def stream(self):
        """The response iterable as write-only stream."""
        return ResponseStream(self)

class ResponseStream(object):
    """A file descriptor like object used by the :class:`ResponseStreamMixin` to
    represent the body of the stream.  It directly pushes into the response
    iterable of the response object.
    """

    mode = "wb+"

    def __init__(self, response):
        self.response = response
        self.closed = False

    def write(self, value):
        if self.closed:
            raise ValueError("I/O operation on closed file")
        self.response._ensure_sequence(mutable=True)
        self.response.response.append(value)
        self.response.headers.pop("Content-Length", None)
        return len(value)

    def writelines(self, seq):
        for item in seq:
            self.write(item)

    def close(self):
        self.closed = True

    def flush(self):
        if self.closed:
            raise ValueError("I/O operation on closed file")

    def isatty(self):
        if self.closed:
            raise ValueError("I/O operation on closed file")
        return False

    def tell(self):
        self.response._ensure_sequence()
        return sum(map(len, self.response.response))

    @property
    def encoding(self):
        return self.response.charset
```

### 分析

```python
class ResponseStreamMixin(object):
    @cached_property
    def stream(self):
        return ResponseStream(self)
```

`@cached_property`装饰器给流提供了缓存功能。实际上是调用ResponseStream。

```python
def write(self, value):
    if self.closed:
        raise ValueError("I/O operation on closed file")
    self.response._ensure_sequence(mutable=True)
    self.response.response.append(value)
    self.response.headers.pop("Content-Length", None)
    return len(value)
```

ResponseStream最核心的就是write方法。_ensure_sequence是BaseResponse类提供的，确保响应体转为list。否则`self.response.response.append(value)`这一步会出错。同时去掉响应头的Content-Length字段，因为在write的时候，这个字段会不断变化，这个字段的设置留到最后再处理。

### 测试
既然ResponseStreamMixin提供了流处理，那么我们可以利用这个发送文件。

```python
from werkzeug.wrappers import Response
from werkzeug.serving import run_simple


def app(environ, start_response):
    response = Response(content_type="application/octet-stream",
                        headers={"Content-Disposition": "attachment; filename=" + "app.py"})
    with open("app.py", "rb") as f:
        response.stream.write(f.read())
    return response(environ, start_response)


if __name__ == "__main__":
    run_simple("localhost", 5000, app, threaded=True)
```


## CommonResponseDescriptorsMixin源码分析

```python
from datetime import datetime
from datetime import timedelta

from .._compat import string_types
from ..datastructures import CallbackDict
from ..http import dump_age
from ..http import dump_header
from ..http import dump_options_header
from ..http import http_date
from ..http import parse_age
from ..http import parse_date
from ..http import parse_options_header
from ..http import parse_set_header
from ..utils import cached_property
from ..utils import environ_property
from ..utils import get_content_type
from ..utils import header_property
from ..wsgi import get_content_length


class CommonResponseDescriptorsMixin(object):
    """A mixin for :class:`BaseResponse` subclasses.  Response objects that
    mix this class in will automatically get descriptors for a couple of
    HTTP headers with automatic type conversion.
    """
    # 这个类的作用就是提供响应对象的一些http属性获取和设置，如mimetype，content-type，content_length等

    @property
    def mimetype(self):
        """The mimetype (content type without charset etc.)"""
        # 获取mimetype属性
        ct = self.headers.get("content-type")
        if ct:
            return ct.split(";")[0].strip()

    @mimetype.setter
    def mimetype(self, value):
        self.headers["Content-Type"] = get_content_type(value, self.charset)

    @property
    def mimetype_params(self):
        """The mimetype parameters as dict. For example if the
        content type is ``text/html; charset=utf-8`` the params would be
        ``{'charset': 'utf-8'}``.

        .. versionadded:: 0.5
        """

        def on_update(d):
            self.headers["Content-Type"] = dump_options_header(self.mimetype, d)

        d = parse_options_header(self.headers.get("content-type", ""))[1]
        return CallbackDict(d, on_update)

    location = header_property(
        "Location",
        doc="""The Location response-header field is used to redirect
        the recipient to a location other than the Request-URI for
        completion of the request or identification of a new
        resource.""",
    )
    age = header_property(
        "Age",
        None,
        parse_age,
        dump_age,
        doc="""The Age response-header field conveys the sender's
        estimate of the amount of time since the response (or its
        revalidation) was generated at the origin server.

        Age values are non-negative decimal integers, representing time
        in seconds.""",
    )
    content_type = header_property(
        "Content-Type",
        doc="""The Content-Type entity-header field indicates the media
        type of the entity-body sent to the recipient or, in the case of
        the HEAD method, the media type that would have been sent had
        the request been a GET.""",
    )
    content_length = header_property(
        "Content-Length",
        None,
        int,
        str,
        doc="""The Content-Length entity-header field indicates the size
        of the entity-body, in decimal number of OCTETs, sent to the
        recipient or, in the case of the HEAD method, the size of the
        entity-body that would have been sent had the request been a
        GET.""",
    )
    content_location = header_property(
        "Content-Location",
        doc="""The Content-Location entity-header field MAY be used to
        supply the resource location for the entity enclosed in the
        message when that entity is accessible from a location separate
        from the requested resource's URI.""",
    )
    content_encoding = header_property(
        "Content-Encoding",
        doc="""The Content-Encoding entity-header field is used as a
        modifier to the media-type. When present, its value indicates
        what additional content codings have been applied to the
        entity-body, and thus what decoding mechanisms must be applied
        in order to obtain the media-type referenced by the Content-Type
        header field.""",
    )
    content_md5 = header_property(
        "Content-MD5",
        doc="""The Content-MD5 entity-header field, as defined in
        RFC 1864, is an MD5 digest of the entity-body for the purpose of
        providing an end-to-end message integrity check (MIC) of the
        entity-body. (Note: a MIC is good for detecting accidental
        modification of the entity-body in transit, but is not proof
        against malicious attacks.)""",
    )
    date = header_property(
        "Date",
        None,
        parse_date,
        http_date,
        doc="""The Date general-header field represents the date and
        time at which the message was originated, having the same
        semantics as orig-date in RFC 822.""",
    )
    expires = header_property(
        "Expires",
        None,
        parse_date,
        http_date,
        doc="""The Expires entity-header field gives the date/time after
        which the response is considered stale. A stale cache entry may
        not normally be returned by a cache.""",
    )
    last_modified = header_property(
        "Last-Modified",
        None,
        parse_date,
        http_date,
        doc="""The Last-Modified entity-header field indicates the date
        and time at which the origin server believes the variant was
        last modified.""",
    )

    @property
    def retry_after(self):
        """The Retry-After response-header field can be used with a
        503 (Service Unavailable) response to indicate how long the
        service is expected to be unavailable to the requesting client.

        Time in seconds until expiration or date.
        """
        value = self.headers.get("retry-after")
        if value is None:
            return
        elif value.isdigit():
            return datetime.utcnow() + timedelta(seconds=int(value))
        return parse_date(value)

    @retry_after.setter
    def retry_after(self, value):
        if value is None:
            if "retry-after" in self.headers:
                del self.headers["retry-after"]
            return
        elif isinstance(value, datetime):
            value = http_date(value)
        else:
            value = str(value)
        self.headers["Retry-After"] = value

    def _set_property(name, doc=None):  # noqa: B902
        def fget(self):
            def on_update(header_set):
                if not header_set and name in self.headers:
                    del self.headers[name]
                elif header_set:
                    self.headers[name] = header_set.to_header()

            return parse_set_header(self.headers.get(name), on_update)

        def fset(self, value):
            if not value:
                del self.headers[name]
            elif isinstance(value, string_types):
                self.headers[name] = value
            else:
                self.headers[name] = dump_header(value)

        return property(fget, fset, doc=doc)

    vary = _set_property(
        "Vary",
        doc="""The Vary field value indicates the set of request-header
        fields that fully determines, while the response is fresh,
        whether a cache is permitted to use the response to reply to a
        subsequent request without revalidation.""",
    )
    content_language = _set_property(
        "Content-Language",
        doc="""The Content-Language entity-header field describes the
        natural language(s) of the intended audience for the enclosed
        entity. Note that this might not be equivalent to all the
        languages used within the entity-body.""",
    )
    allow = _set_property(
        "Allow",
        doc="""The Allow entity-header field lists the set of methods
        supported by the resource identified by the Request-URI. The
        purpose of this field is strictly to inform the recipient of
        valid methods associated with the resource. An Allow header
        field MUST be present in a 405 (Method Not Allowed)
        response.""",
    )

    del _set_property
```

### 分析
这个Mixin类就是提供接口方便获取或设置HTTP响应头字段，如mimetype，content-type，content_length。

```python
response = Response()
print response.mimetype  # text/plain
response.mimetype = "application/octet-stream"
print response.mimetype  # application/octet-stream
```

如果继承了这个Mixin类，可以很方便地获取，设置HTTP响应头字段


## WWWAuthenticateMixin源码分析

```python
class WWWAuthenticateMixin(object):
    """Adds a :attr:`www_authenticate` property to a response object."""

    @property
    def www_authenticate(self):
        """The `WWW-Authenticate` header in a parsed form."""

        def on_update(www_auth):
            if not www_auth and "www-authenticate" in self.headers:
                del self.headers["www-authenticate"]
            elif www_auth:
                self.headers["WWW-Authenticate"] = www_auth.to_header()

        header = self.headers.get("www-authenticate")
        return parse_www_authenticate_header(header, on_update)

```

这个属性实际上是WWWAuthenticate的对象，因此操作的接口都是WWWAuthenticate提供的接口。这个类只提供设置响应头的www-authenticate字段，不提供验证。

说一下这个验证，就明白原理了：
具体定义可以参考RFC 2617中，首先浏览器请求，然后服务端判断请求头是否有`AUTHORIZATION`字段，如果没有，响应头应该添加`WWW-Authenticate`字段，格式为`WWW-Authenticate: Basic realm=“.”`，并把状态码设置为401，返回给浏览器，浏览器会根据这个响应要求用户输入账号和密码，返回把账号和密码的信息请求头`Authorization`字段，格式为`Authorization: Basic YWRtaW46YWRtaW4=`。其中`YWRtaW46YWRtaW4=`是经过base64编码的账号和密码。对应原始的账号密码是admin:admin。服务端通过base64解码并验证登录信息是否正确。

**但是这种方式验证基本不用，因为不安全。**

### 测试

```python
def app(environ, start_response):
    # 验证
    auth = environ.get("HTTP_AUTHORIZATION", None)  # 注意：werkzeug把AUTHORIZATION字段改成了HTTP_AUTHORIZATION
    if not auth:
        response = Response("No auth!", status=401)  # 状态码必须设置为401
        response.www_authenticate.set_basic()  # 响应头设置WWW-Authenticate字段
    else:
        # 此处进行base64解码
        import base64
        auth = auth.split(" ")[1]  # 去掉"Basic"
        data = base64.decodestring(auth).split(":")  # 分隔账号和密码
        user = data[0]
        password = data[1]
        if user == "admin" and password == "123456":
            response = Response("Welcome!" + str(user))
        else:
            response = Response("user or password is incorrect!")
    return response(environ, start_response)
```
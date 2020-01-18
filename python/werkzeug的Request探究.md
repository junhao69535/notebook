# werkzeug的Request探究

werkzeug提供了Request类封装请求。位于werkzeug.wrappers模块里。

```python
class Request(
    BaseRequest,
    AcceptMixin,
    ETagRequestMixin,
    UserAgentMixin,
    AuthorizationMixin,
    CommonRequestDescriptorsMixin,
):
    """Full featured request object implementing the following mixins:

    - :class:`AcceptMixin` for accept header parsing
    - :class:`ETagRequestMixin` for etag and cache control handling
    - :class:`UserAgentMixin` for user agent introspection
    - :class:`AuthorizationMixin` for http auth handling
    - :class:`CommonRequestDescriptorsMixin` for common headers
    """
```

请求对象有以下规则：
* 请求对象是不可变对象，除非修改内部数据结构。
* 请求对象是线程共享的，不是线程安全的。如果在多线程中使用清加锁。这个规则实际上是针对请求对象作为全局变量而言的，如flask，因此flask实现了request线程隔离。
* 请求对象不能被序列化。

可以看出，这个类是通过继承其他类类实现功能的。基本功能由BaseRequest类提供（意味着如果不需要这些混入类的功能，Request类和BaseRequest类功能一样），其他功能由Mixin类提供。从继承的类可以看出，我们也可以通过继承新的Mixin类来增加功能。默认下继承了5个Mixin类。
* AcceptMixin提供了解析请求头功能。
* ETagResponseMixin提供了etag和cache处理功能。
* UserAgentMixin提供了用户代理内省功能。
* AuthorizationMixin提供了HTTP认证处理。
* CommonRequestDescriptorsMixin提供了各种HTTP请求头字段描述器。


## BaseRequest源码分析

```python
import warnings
from functools import update_wrapper
from io import BytesIO

from .._compat import to_native
from .._compat import to_unicode
from .._compat import wsgi_decoding_dance
from .._compat import wsgi_get_bytes
from ..datastructures import CombinedMultiDict
from ..datastructures import EnvironHeaders
from ..datastructures import ImmutableList
from ..datastructures import ImmutableMultiDict
from ..datastructures import ImmutableTypeConversionDict
from ..datastructures import iter_multi_items
from ..datastructures import MultiDict
from ..formparser import default_stream_factory
from ..formparser import FormDataParser
from ..http import parse_cookie
from ..http import parse_options_header
from ..urls import url_decode
from ..utils import cached_property
from ..utils import environ_property
from ..wsgi import get_content_length
from ..wsgi import get_current_url
from ..wsgi import get_host
from ..wsgi import get_input_stream


class BaseRequest(object):
    """Very basic request object.  This does not implement advanced stuff like
    entity tag parsing or cache controls.  The request object is created with
    the WSGI environment as first argument and will add itself to the WSGI
    environment as ``'werkzeug.request'`` unless it's created with
    `populate_request` set to False.
    # 不提供etag解析和cache控制功能，这个对象会添加到WSGI环境。

    There are a couple of mixins available that add additional functionality
    to the request object, there is also a class called `Request` which
    subclasses `BaseRequest` and all the important mixins.

    It's a good idea to create a custom subclass of the :class:`BaseRequest`
    and add missing functionality either via mixins or direct implementation.
    Here an example for such subclasses::

        from werkzeug.wrappers import BaseRequest, ETagRequestMixin

        class Request(BaseRequest, ETagRequestMixin):
            pass

    Request objects are **read only**.  As of 0.5 modifications are not
    allowed in any place.  Unlike the lower level parsing functions the
    request object will use immutable objects everywhere possible.
    # Request对象是只读的，不同于底层解析函数，request对象采用不可变对象实现功能。

    Per default the request object will assume all the text data is `utf-8`
    encoded.  Please refer to :doc:`the unicode chapter </unicode>` for more
    details about customizing the behavior.

    Per default the request object will be added to the WSGI
    environment as `werkzeug.request` to support the debugging system.
    If you don't want that, set `populate_request` to `False`.

    If `shallow` is `True` the environment is initialized as shallow
    object around the environ.  Every operation that would modify the
    environ in any way (such as consuming form data) raises an exception
    unless the `shallow` attribute is explicitly set to `False`.  This
    is useful for middlewares where you don't want to consume the form
    data by accident.  A shallow request is not populated to the WSGI
    environment.
    # 如果shallow是True，则环境会初始化为shallow对象。任何修改环境的操作都会
    # 抛出异常，除非shallow是False。这对那些不想修改数据的中间件特别有用。
    # shallow请求不会在WSGI环境产生。

    .. versionchanged:: 0.5
       read-only mode was enforced by using immutables classes for all
       data.
    """

    #: the charset for the request, defaults to utf-8
    charset = "utf-8"

    #: the error handling procedure for errors, defaults to 'replace'
    encoding_errors = "replace"

    #: the maximum content length.  This is forwarded to the form data
    #: parsing function (:func:`parse_form_data`).  When set and the
    #: :attr:`form` or :attr:`files` attribute is accessed and the
    #: parsing fails because more than the specified value is transmitted
    #: a :exc:`~werkzeug.exceptions.RequestEntityTooLarge` exception is raised.
    #:
    #: Have a look at :ref:`dealing-with-request-data` for more details.
    #:
    #: .. versionadded:: 0.5
    max_content_length = None

    #: the maximum form field size.  This is forwarded to the form data
    #: parsing function (:func:`parse_form_data`).  When set and the
    #: :attr:`form` or :attr:`files` attribute is accessed and the
    #: data in memory for post data is longer than the specified value a
    #: :exc:`~werkzeug.exceptions.RequestEntityTooLarge` exception is raised.
    #:
    #: Have a look at :ref:`dealing-with-request-data` for more details.
    #:
    #: .. versionadded:: 0.5
    max_form_memory_size = None

    #: the class to use for `args` and `form`.  The default is an
    #: :class:`~werkzeug.datastructures.ImmutableMultiDict` which supports
    #: multiple values per key.  alternatively it makes sense to use an
    #: :class:`~werkzeug.datastructures.ImmutableOrderedMultiDict` which
    #: preserves order or a :class:`~werkzeug.datastructures.ImmutableDict`
    #: which is the fastest but only remembers the last key.  It is also
    #: possible to use mutable structures, but this is not recommended.
    #:
    #: .. versionadded:: 0.6
    # 如果需要参数有序，则可以使用ImmutableOrderedMultiDict类，如果需要处理更快，
    # 可以使用ImmutableDict，但不支持一个键多个值。
    parameter_storage_class = ImmutableMultiDict

    #: the type to be used for list values from the incoming WSGI environment.
    #: By default an :class:`~werkzeug.datastructures.ImmutableList` is used
    #: (for example for :attr:`access_list`).
    #:
    #: .. versionadded:: 0.6
    list_storage_class = ImmutableList

    #: the type to be used for dict values from the incoming WSGI environment.
    #: By default an
    #: :class:`~werkzeug.datastructures.ImmutableTypeConversionDict` is used
    #: (for example for :attr:`cookies`).
    #:
    #: .. versionadded:: 0.6
    dict_storage_class = ImmutableTypeConversionDict

    #: The form data parser that shoud be used.  Can be replaced to customize
    #: the form date parsing.
    # 可用自定义表格解析类替代这个类
    form_data_parser_class = FormDataParser

    #: Optionally a list of hosts that is trusted by this request.  By default
    #: all hosts are trusted which means that whatever the client sends the
    #: host is will be accepted.
    #:
    #: Because `Host` and `X-Forwarded-Host` headers can be set to any value by
    #: a malicious client, it is recommended to either set this property or
    #: implement similar validation in the proxy (if application is being run
    #: behind one).
    #:
    #: .. versionadded:: 0.9
    # 由于Host和X-Forwarded-Host可以设置为任意值，因此这两个字段不安全。
    trusted_hosts = None

    #: Indicates whether the data descriptor should be allowed to read and
    #: buffer up the input stream.  By default it's enabled.
    #:
    #: .. versionadded:: 0.9
    disable_data_descriptor = False

    def __init__(self, environ, populate_request=True, shallow=False):
        self.environ = environ
        if populate_request and not shallow:
            self.environ["werkzeug.request"] = self
        self.shallow = shallow

    def __repr__(self):
        # make sure the __repr__ even works if the request was created
        # from an invalid WSGI environment.  If we display the request
        # in a debug session we don't want the repr to blow up.
        args = []
        try:
            args.append("'%s'" % to_native(self.url, self.url_charset))
            args.append("[%s]" % self.method)
        except Exception:
            args.append("(invalid WSGI environ)")

        return "<%s %s>" % (self.__class__.__name__, " ".join(args))

    @property
    def url_charset(self):
        """The charset that is assumed for URLs.  Defaults to the value
        of :attr:`charset`.

        .. versionadded:: 0.6
        """
        return self.charset

    @classmethod
    def from_values(cls, *args, **kwargs):
        """Create a new request object based on the values provided.  If
        environ is given missing values are filled from there.  This method is
        useful for small scripts when you need to simulate a request from an URL.
        Do not use this method for unittesting, there is a full featured client
        object (:class:`Client`) that allows to create multipart requests,
        support for cookies etc.
        # 根据参数创建一个请求对象。用于测试。
        >>> from cStringIO import StringIO
        >>> data = "name=this+is+encoded+form+data&another_key=another+one"
        >>> request = Request.from_values(query_string='foo=bar&blah=blafasel',
        ...    content_length=len(data), input_stream=StringIO(data),
        ...    content_type='application/x-www-form-urlencoded',
        ...    method='POST')
        ...
        >>> request.method
        'POST'

        This accepts the same options as the
        :class:`~werkzeug.test.EnvironBuilder`.

        .. versionchanged:: 0.5
           This method now accepts the same arguments as
           :class:`~werkzeug.test.EnvironBuilder`.  Because of this the
           `environ` parameter is now called `environ_overrides`.

        :return: request object
        """
        from ..test import EnvironBuilder

        charset = kwargs.pop("charset", cls.charset)
        kwargs["charset"] = charset
        builder = EnvironBuilder(*args, **kwargs)
        try:
            return builder.get_request(cls)
        finally:
            builder.close()

    @classmethod
    def application(cls, f):
        """Decorate a function as responder that accepts the request as
        the last argument.  This works like the :func:`responder`
        decorator but the function is passed the request object as the
        last argument and the request object will be closed
        automatically::

            @Request.application
            def my_wsgi_app(request):
                return Response('Hello World!')

        As of Werkzeug 0.14 HTTP exceptions are automatically caught and
        converted to responses instead of failing.

        :param f: the WSGI callable to decorate
        :return: a new WSGI callable
        """
        # 为一个最后一个参数为request的函数提供request对象。如上面的my_wsgi_app中
        # 的request，经过装饰器之后这个request就会被绑定为当前的request对象。

        #: return a callable that wraps the -2nd argument with the request
        #: and calls the function with all the arguments up to that one and
        #: the request.  The return value is then called with the latest
        #: two arguments.  This makes it possible to use this decorator for
        #: both standalone WSGI functions as well as bound methods and
        #: partially applied functions.
        from ..exceptions import HTTPException

        def application(*args):
            request = cls(args[-2])
            with request:
                try:
                    resp = f(*args[:-2] + (request,))
                except HTTPException as e:
                    resp = e.get_response(args[-2])
                return resp(*args[-2:])

        return update_wrapper(application, f)

    def _get_file_stream(
        self, total_content_length, content_type, filename=None, content_length=None
    ):
        """Called to get a stream for the file upload.

        This must provide a file-like class with `read()`, `readline()`
        and `seek()` methods that is both writeable and readable.

        The default implementation returns a temporary file if the total
        content length is higher than 500KB.  Because many browsers do not
        provide a content length for the files only the total content
        length matters.
        # 如果总长度超过500KB，返回一个临时文件对象。因为很多浏览器只提供总长度，而不提供
        # 上传文件的长度。

        :param total_content_length: the total content length of all the
                                     data in the request combined.  This value
                                     is guaranteed to be there.
        :param content_type: the mimetype of the uploaded file.
        :param filename: the filename of the uploaded file.  May be `None`.
        :param content_length: the length of this file.  This value is usually
                               not provided because webbrowsers do not provide
                               this value.
        """
        return default_stream_factory(
            total_content_length=total_content_length,
            filename=filename,
            content_type=content_type,
            content_length=content_length,
        )

    @property
    def want_form_data_parsed(self):
        """Returns True if the request method carries content.  As of
        Werkzeug 0.9 this will be the case if a content type is transmitted.

        .. versionadded:: 0.8
        """
        # 判断请求头部是否提供了CONTENT_TYPE字段
        return bool(self.environ.get("CONTENT_TYPE"))

    def make_form_data_parser(self):
        """Creates the form data parser. Instantiates the
        :attr:`form_data_parser_class` with some parameters.

        .. versionadded:: 0.8
        """
        return self.form_data_parser_class(
            self._get_file_stream,
            self.charset,
            self.encoding_errors,
            self.max_form_memory_size,
            self.max_content_length,
            self.parameter_storage_class,
        )

    def _load_form_data(self):
        """Method used internally to retrieve submitted data.  After calling
        this sets `form` and `files` on the request object to multi dicts
        filled with the incoming form data.  As a matter of fact the input
        stream will be empty afterwards.  You can also call this method to
        force the parsing of the form data.

        .. versionadded:: 0.8
        """
        # 给request对象加载表格数据
        # abort early if we have already consumed the stream
        if "form" in self.__dict__:
            return

        _assert_not_shallow(self)

        if self.want_form_data_parsed:
            content_type = self.environ.get("CONTENT_TYPE", "")
            content_length = get_content_length(self.environ)
            mimetype, options = parse_options_header(content_type)
            parser = self.make_form_data_parser()
            data = parser.parse(
                self._get_stream_for_parsing(), mimetype, content_length, options
            )
        else:
            data = (
                self.stream,
                self.parameter_storage_class(),
                self.parameter_storage_class(),
            )

        # inject the values into the instance dict so that we bypass
        # our cached_property non-data descriptor.
        d = self.__dict__
        d["stream"], d["form"], d["files"] = data

    def _get_stream_for_parsing(self):
        """This is the same as accessing :attr:`stream` with the difference
        that if it finds cached data from calling :meth:`get_data` first it
        will create a new stream out of the cached data.

        .. versionadded:: 0.9.3
        """
        cached_data = getattr(self, "_cached_data", None)
        if cached_data is not None:
            return BytesIO(cached_data)
        return self.stream

    def close(self):
        """Closes associated resources of this request object.  This
        closes all file handles explicitly.  You can also use the request
        object in a with statement which will automatically close it.

        .. versionadded:: 0.9
        """
        files = self.__dict__.get("files")
        for _key, value in iter_multi_items(files or ()):
            value.close()

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_value, tb):
        self.close()

    @cached_property
    def stream(self):
        """
        If the incoming form data was not encoded with a known mimetype
        the data is stored unmodified in this stream for consumption.  Most
        of the time it is a better idea to use :attr:`data` which will give
        you that data as a string.  The stream only returns the data once.
        # 使用data属性可以获取string形式的数据。stream只返回一次。

        Unlike :attr:`input_stream` this stream is properly guarded that you
        can't accidentally read past the length of the input.  Werkzeug will
        internally always refer to this stream to read data which makes it
        possible to wrap this object with a stream that does filtering.
        # 这个方法能保证读取的数据不超过边界。

        .. versionchanged:: 0.9
           This stream is now always available but might be consumed by the
           form parser later on.  Previously the stream was only set if no
           parsing happened.
        """
        _assert_not_shallow(self)
        return get_input_stream(self.environ)

    # environ_property是一个描述器，实现__get__和__set__方法，默认是read-only，
    # 除非指定read-only为False。__get__从environ获取数据。
    input_stream = environ_property(
        "wsgi.input",
        """The WSGI input stream.

        In general it's a bad idea to use this one because you can
        easily read past the boundary.  Use the :attr:`stream`
        instead.""",
    )

    @cached_property
    def args(self):
        """The parsed URL parameters (the part in the URL after the question
        mark).

        By default an
        :class:`~werkzeug.datastructures.ImmutableMultiDict`
        is returned from this function.  This can be changed by setting
        :attr:`parameter_storage_class` to a different type.  This might
        be necessary if the order of the form data is important.
        """
        # wsgi_get_bytes把请求参数字符串编码为字节序列，编码格式和操作系统有关
        return url_decode(
            wsgi_get_bytes(self.environ.get("QUERY_STRING", "")),
            self.url_charset,
            errors=self.encoding_errors,
            cls=self.parameter_storage_class,
        )

    @cached_property
    def data(self):
        """
        Contains the incoming request data as string in case it came with
        a mimetype Werkzeug does not handle.
        """
        # 不依据mimetype来处理数据

        if self.disable_data_descriptor:
            raise AttributeError("data descriptor is disabled")
        # XXX: this should eventually be deprecated.

        # We trigger form data parsing first which means that the descriptor
        # will not cache the data that would otherwise be .form or .files
        # data.  This restores the behavior that was there in Werkzeug
        # before 0.9.  New code should use :meth:`get_data` explicitly as
        # this will make behavior explicit.
        return self.get_data(parse_form_data=True)

    def get_data(self, cache=True, as_text=False, parse_form_data=False):
        """This reads the buffered incoming data from the client into one
        bytestring.  By default this is cached but that behavior can be
        changed by setting `cache` to `False`.

        Usually it's a bad idea to call this method without checking the
        content length first as a client could send dozens of megabytes or more
        to cause memory problems on the server.

        Note that if the form data was already parsed this method will not
        return anything as form data parsing does not cache the data like
        this method does.  To implicitly invoke form data parsing function
        set `parse_form_data` to `True`.  When this is done the return value
        of this method will be an empty string if the form parser handles
        the data.  This generally is not necessary as if the whole data is
        cached (which is the default) the form parser will used the cached
        data to parse the form data.  Please be generally aware of checking
        the content length first in any case before calling this method
        to avoid exhausting server memory.

        If `as_text` is set to `True` the return value will be a decoded
        unicode string.

        .. versionadded:: 0.9
        """
        rv = getattr(self, "_cached_data", None)
        if rv is None:
            if parse_form_data:
                self._load_form_data()
            rv = self.stream.read()
            if cache:
                self._cached_data = rv
        if as_text:
            rv = rv.decode(self.charset, self.encoding_errors)
        return rv

    @cached_property
    def form(self):
        """The form parameters.  By default an
        :class:`~werkzeug.datastructures.ImmutableMultiDict`
        is returned from this function.  This can be changed by setting
        :attr:`parameter_storage_class` to a different type.  This might
        be necessary if the order of the form data is important.

        Please keep in mind that file uploads will not end up here, but instead
        in the :attr:`files` attribute.

        .. versionchanged:: 0.9

            Previous to Werkzeug 0.9 this would only contain form data for POST
            and PUT requests.
        """
        self._load_form_data()
        return self.form

    @cached_property
    def values(self):
        """A :class:`werkzeug.datastructures.CombinedMultiDict` that combines
        :attr:`args` and :attr:`form`."""
        args = []
        for d in self.args, self.form:
            if not isinstance(d, MultiDict):
                d = MultiDict(d)
            args.append(d)
        return CombinedMultiDict(args)

    @cached_property
    def files(self):
        """:class:`~werkzeug.datastructures.MultiDict` object containing
        all uploaded files.  Each key in :attr:`files` is the name from the
        ``<input type="file" name="">``.  Each value in :attr:`files` is a
        Werkzeug :class:`~werkzeug.datastructures.FileStorage` object.

        It basically behaves like a standard file object you know from Python,
        with the difference that it also has a
        :meth:`~werkzeug.datastructures.FileStorage.save` function that can
        store the file on the filesystem.

        Note that :attr:`files` will only contain data if the request method was
        POST, PUT or PATCH and the ``<form>`` that posted to the request had
        ``enctype="multipart/form-data"``.  It will be empty otherwise.

        See the :class:`~werkzeug.datastructures.MultiDict` /
        :class:`~werkzeug.datastructures.FileStorage` documentation for
        more details about the used data structure.
        """
        self._load_form_data()
        return self.files

    @cached_property
    def cookies(self):
        """A :class:`dict` with the contents of all cookies transmitted with
        the request."""
        return parse_cookie(
            self.environ,
            self.charset,
            self.encoding_errors,
            cls=self.dict_storage_class,
        )

    @cached_property
    def headers(self):
        """The headers from the WSGI environ as immutable
        :class:`~werkzeug.datastructures.EnvironHeaders`.
        """
        return EnvironHeaders(self.environ)

    @cached_property
    def path(self):
        """Requested path as unicode.  This works a bit like the regular path
        info in the WSGI environment but will always include a leading slash,
        even if the URL root is accessed.
        """
        raw_path = wsgi_decoding_dance(
            self.environ.get("PATH_INFO") or "", self.charset, self.encoding_errors
        )
        return "/" + raw_path.lstrip("/")

    @cached_property
    def full_path(self):
        """Requested path as unicode, including the query string."""
        return self.path + u"?" + to_unicode(self.query_string, self.url_charset)

    @cached_property
    def script_root(self):
        """The root path of the script without the trailing slash."""
        raw_path = wsgi_decoding_dance(
            self.environ.get("SCRIPT_NAME") or "", self.charset, self.encoding_errors
        )
        return raw_path.rstrip("/")

    @cached_property
    def url(self):
        """The reconstructed current URL as IRI.
        See also: :attr:`trusted_hosts`.
        """
        return get_current_url(self.environ, trusted_hosts=self.trusted_hosts)

    @cached_property
    def base_url(self):
        """Like :attr:`url` but without the querystring
        See also: :attr:`trusted_hosts`.
        """
        return get_current_url(
            self.environ, strip_querystring=True, trusted_hosts=self.trusted_hosts
        )

    @cached_property
    def url_root(self):
        """The full URL root (with hostname), this is the application
        root as IRI.
        See also: :attr:`trusted_hosts`.
        """
        return get_current_url(self.environ, True, trusted_hosts=self.trusted_hosts)

    @cached_property
    def host_url(self):
        """Just the host with scheme as IRI.
        See also: :attr:`trusted_hosts`.
        """
        return get_current_url(
            self.environ, host_only=True, trusted_hosts=self.trusted_hosts
        )

    @cached_property
    def host(self):
        """Just the host including the port if available.
        See also: :attr:`trusted_hosts`.
        """
        # 获取服务器HOST，包含PORT
        return get_host(self.environ, trusted_hosts=self.trusted_hosts)

    query_string = environ_property(
        "QUERY_STRING",
        "",
        read_only=True,
        load_func=wsgi_get_bytes,
        doc="The URL parameters as raw bytestring.",
    )
    method = environ_property(
        "REQUEST_METHOD",
        "GET",
        read_only=True,
        load_func=lambda x: x.upper(),
        doc="The request method. (For example ``'GET'`` or ``'POST'``).",
    )

    @cached_property
    def access_route(self):
        """If a forwarded header exists this is a list of all ip addresses
        from the client ip to the last proxy server.
        """
        # 如果存在转发的标头，则这是从客户端ip到最后一个代理服务器的所有ip地址的列表
        if "HTTP_X_FORWARDED_FOR" in self.environ:
            addr = self.environ["HTTP_X_FORWARDED_FOR"].split(",")
            return self.list_storage_class([x.strip() for x in addr])
        elif "REMOTE_ADDR" in self.environ:
            return self.list_storage_class([self.environ["REMOTE_ADDR"]])
        return self.list_storage_class()

    @property
    def remote_addr(self):
        """The remote address of the client."""
        return self.environ.get("REMOTE_ADDR")

    remote_user = environ_property(
        "REMOTE_USER",
        doc="""If the server supports user authentication, and the
        script is protected, this attribute contains the username the
        user has authenticated as.""",
    )

    scheme = environ_property(
        "wsgi.url_scheme",
        doc="""
        URL scheme (http or https).

        .. versionadded:: 0.7""",
    )

    @property
    def is_xhr(self):
        """True if the request was triggered via a JavaScript XMLHttpRequest.
        This only works with libraries that support the ``X-Requested-With``
        header and set it to "XMLHttpRequest".  Libraries that do that are
        prototype, jQuery and Mochikit and probably some more.
        # 判断这个请求是否由avaScript XMLHttpRequest触发。

        .. deprecated:: 0.13
            ``X-Requested-With`` is not standard and is unreliable. You
            may be able to use :attr:`AcceptMixin.accept_mimetypes`
            instead.
            # 已经遗弃，因为X-Requested-With不是http标准且不可靠。可以使用
            # AcceptMixin.accept_mimetypes来代替r。
        """
        warnings.warn(
            "'Request.is_xhr' is deprecated as of version 0.13 and will"
            " be removed in version 1.0. The 'X-Requested-With' header"
            " is not standard and is unreliable. You may be able to use"
            " 'accept_mimetypes' instead.",
            DeprecationWarning,
            stacklevel=2,
        )
        return self.environ.get("HTTP_X_REQUESTED_WITH", "").lower() == "xmlhttprequest"

    is_secure = property(
        lambda self: self.environ["wsgi.url_scheme"] == "https",
        doc="`True` if the request is secure.",
    )
    is_multithread = environ_property(
        "wsgi.multithread",
        doc="""boolean that is `True` if the application is served by a
        multithreaded WSGI server.""",
    )
    is_multiprocess = environ_property(
        "wsgi.multiprocess",
        doc="""boolean that is `True` if the application is served by a
        WSGI server that spawns multiple processes.""",
    )
    is_run_once = environ_property(
        "wsgi.run_once",
        doc="""boolean that is `True` if the application will be
        executed only once in a process lifetime.  This is the case for
        CGI for example, but it's not guaranteed that the execution only
        happens one time.""",
    )


def _assert_not_shallow(request):
    if request.shallow:
        raise RuntimeError(
            "A shallow request tried to consume form data. If you really"
            " want to do that, set `shallow` to False."
        )

```

### 分析
先分析出现较多的两个描述器：`@cached_property`和`environ_property`。

#### @cached_property

```python
class cached_property(property):
    def __init__(self, func, name=None, doc=None):
        self.__name__ = name or func.__name__
        self.__module__ = func.__module__
        self.__doc__ = doc or func.__doc__
        self.func = func

    def __set__(self, obj, value):
        obj.__dict__[self.__name__] = value

    def __get__(self, obj, type=None):
        if obj is None:
            return self
        value = obj.__dict__.get(self.__name__, _missing)
        if value is _missing:
            value = self.func(obj)
            obj.__dict__[self.__name__] = value
        return value
```

这个描述器的作用是把函数转变为懒惰属性。懒惰属性就是只有当真正调用时才会执行计算且当第一次调用时会计算一次，之后直接获取结果的属性。`cached_property`类继承`property`类，重写了__init__、__set__、和__get__方法。如：

```python
@cached_property
    def host(self):
        return get_host(self.environ, trusted_hosts=self.trusted_hosts)
```

当第一次调用request.host时，会把request对象作为obj，request的类作为type传入__get__，因此request的字典没有host这个属性，因此执行这条语句`value = obj.__dict__.get(self.__name__, _missing)`，value得到的是_missing。然后进入if语句，真正执行函数调用，并把这个结果作为属性写到request的字典里。那么之后执行这条语句`value = obj.__dict__.get(self.__name__, _missing)`时得到的不再是_mising，而是第一次执行得到的结果，所以不会再次执行函数调用了，这也是为什么叫懒惰属性的原因。

#### environ_property

```python
class environ_property(_DictAccessorProperty):
    read_only = True

    def lookup(self, obj):
        return obj.environ


class _DictAccessorProperty(object):
    """Baseclass for `environ_property` and `header_property`."""

    read_only = False

    def __init__(
        self,
        name,
        default=None,
        load_func=None,
        dump_func=None,
        read_only=None,
        doc=None,
    ):
        self.name = name
        self.default = default
        self.load_func = load_func
        self.dump_func = dump_func
        if read_only is not None:
            self.read_only = read_only
        self.__doc__ = doc

    def __get__(self, obj, type=None):
        if obj is None:
            return self
        storage = self.lookup(obj)  # 得到的就是environ
        if self.name not in storage:
            return self.default
        rv = storage[self.name]
        if self.load_func is not None:  # load_func可以对属性进一步处理
            try:
                rv = self.load_func(rv)
            except (ValueError, TypeError):
                rv = self.default
        return rv

    def __set__(self, obj, value):
        if self.read_only:
            raise AttributeError("read only property")
        if self.dump_func is not None:
            value = self.dump_func(value)
        self.lookup(obj)[self.name] = value

    def __delete__(self, obj):
        if self.read_only:
            raise AttributeError("read only property")
        self.lookup(obj).pop(self.name, None)

    def __repr__(self):
        return "<%s %s>" % (self.__class__.__name__, self.name)
```

将请求属性映射到环境变量。实际上就是从environ中获取所需的属性。如查询字符串，直接把"QUERY_STRING"作为参数传给environ_property即可，它把从environ映射查询字符串属性。environ不一定是请求环境，只要提供了environ这个属性即可。如果需要映射的属性不存在，返回default（默认为None）。这些映射的属性都是read-only，除非指定read_only为False。


BaseRequest类就是对environ进一步封装，提供更友好的接口处理请求。如：
* 提供headers懒惰属性，这个属性可以获取所有有关请求头的信息。
* 提供cookies懒惰属性，这个属性解析了请求的cookie。
* 提供files懒惰属性，这个属性获取上传的二进制文件。
* 提供form懒惰属性，这个属性解析了请求表格数据。
* 提供了args懒惰属性，这个属性解析了查询参数。


#### headers

```python
@cached_property
    def headers(self):
        return EnvironHeaders(self.environ)
```

EnvironHeaders实际上还是Headers类，只是限制了headers是只读的。Headers提供了接口和dict基本相同。EnvironHeaders重写了Headers的__getitem__方法，最核心的地方也是这个方法：

```python
def __getitem__(self, key, _get_mode=False):
    if not isinstance(key, string_types):
        raise KeyError(key)
    key = key.upper().replace("-", "_")
    if key in ("CONTENT_TYPE", "CONTENT_LENGTH"):
        return _unicodify_header_value(self.environ[key])
    return _unicodify_header_value(self.environ["HTTP_" + key])
```

__getitem__就是把environ的请求头部数据提取出来。

#### cookies

```python
@cached_property
def cookies(self):
    return parse_cookie(
        self.environ,
        self.charset,
        self.encoding_errors,
        cls=self.dict_storage_class,
    )
```

实际上就是解析environ中的cookie请求头部字段然后存储到`self.dict_storage_class`中。


其他的files、form等处理和上面类似，可自行了解。
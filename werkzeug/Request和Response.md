# Request和Response

## Request和Response简介
werkzeug提供了Request类封装请求，Response类封装响应，都位于werkzeug.wrappers模块里。

一个web app最核心的两个功能就是处理请求和返回响应。Request和Response的用法也很简单：

```python
def app(environ, start_response):
    request = Request(environ)
    ...  # 处理请求
    response = Response()
    return response(environ, start_response)
```

使用environ实例化request，然后进行处理，最后实例化response，返回响应即可。


## Requset
Request究竟做了什么工作？实际上就是对WSGI服务器提供environ进行处理，封装。

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

这些Mixin类都比较简单，有需要可自行探究。下面探究BaseRequest类：

```python
class BaseRequest(object):
    def __init__(self, environ, populate_request=True, shallow=False):
        self.environ = environ
        if populate_request and not shallow:
            self.environ["werkzeug.request"] = self
        self.shallow = shallow
```

__init__方法就是获取了WSGI提供environ，并且将自身赋值给environ的werkzeug.request。下面说一下这个类提供的主要功能：

### args

```python
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
    return url_decode(
        wsgi_get_bytes(self.environ.get("QUERY_STRING", "")),
        self.url_charset,
        errors=self.encoding_errors,
        cls=self.parameter_storage_class,
    )
```

args提供了处理请求的查询字符串的功能。把请求字符串以cls这个参数指定的类的格式存储。

### form

```python
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
```

form提供了处理请求体中的form的功能。

### files

```python
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
```

files和

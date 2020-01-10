# werkzeug库入门

## 简介
> werkzeug German noun: “tool”. Etymology: werk (“work”), zeug (“stuff”)
>
> Werkzeug is a comprehensive WSGI web application library. It began as a simple collection of various utilities for WSGI applications and has become one of the most advanced > WSGI utility libraries.
>
> Werkzeug is Unicode aware and doesn’t enforce any dependencies. It is up to the developer to choose a template engine, database adapter, and even how to handle requests.

这是官网的简介，werkzeug是一个全面WSGI web app库。提供大量的工具用于开发WSGI web app。


## WSGI app介绍
### WSGI app最原始形式
```python
def application(environ, start_response):
    start_response表明开始响应
    start_response("200 OK", [("Content-Type", "text/plain")])
    return ["Hello World!"]
```

WSGI服务器给WSGI web app提供了environ，类型是dict，是关于请求的所有信息。start_response是一个回调函数，app调用这个回调函数给WSGI服务器提供响应行和响应头。

### werkzeug提供的Response包装响应
```python
from werkzeug.wrappers import Response
def application(environ, start_response):
    response = Response("Hello World!", mimetype="text/plain")
    # Response的__call__接受environ和start_response参数
    return response(environ, start_response)
```

werkzeug提供的Response包装响应，Response这个类的__call__方法实际上还是和上面原始的方式一样，解析environ，调用回调函数，返回响应体。

### werkzeug提供Request包装请求
```python
from werkzeug.wrappers import Response, Request
def application(environ, start_response):
    request = Request(environ)
    # Request包装environ，以更好的方式提供请求内容
    text = "Hello %s!" % request.args.get("name", "World")
    response = Response(text, mimetype="text/plain")
    return response(environ, start_response)
```

werkzeug提供Request包装请求，Request包装environ，以更好的方式提供请求内容。


## 教程
### 第一步：创建文件夹
app的文件布局如下：

```
/shortly
    /static
    /templates
    shortly.py
```

### 第二步：基本架构
导入模块：

```python
#!coding=utf-8
import os
import redis
import urlparse
from werkzeug.wrappers import Request, Response
from werkzeug.routing import Map, Rule
from werkzeug.exceptions import HTTPException, NotFound
from werkzeug.wsgi import SharedDataMiddleware
from werkzeug.utils import redirect
from jinja2 import Environment, FileSystemLoader
```

创建基本架构：

```python
class Shortly(object):

    def __init__(self, config):
        self.redis = redis.Redis(config['redis_host'], config['redis_port'])

    def dispatch_request(self, request):
        return Response('Hello World!')

    def wsgi_app(self, environ, start_response):
        request = Request(environ)
        response = self.dispatch_request(request)
        return response(environ, start_response)

    def __call__(self, environ, start_response):
        return self.wsgi_app(environ, start_response)

def create_app(redis_host='localhost', redis_port=6379, with_static=True):
    app = Shortly({
        'redis_host':       redis_host,
        'redis_port':       redis_port
    })
    if with_static:
        app.wsgi_app = SharedDataMiddleware(app.wsgi_app, {
            '/static':  os.path.join(os.path.dirname(__file__), 'static')
        })
    return app

if __name__ == '__main__':
    from werkzeug.serving import run_simple
    app = create_app()
    run_simple('127.0.0.1', 5000, app, use_debugger=True, use_reloader=True)
```

通过工厂函数，可以提供中间件包装app，这里利用共享数据中间件提供了静态文件导出的功能。这里的app被封装了好几层，首先是一个Shortly实例，然后在__call__方法里面调用wsgi_app方法，而wsgi_app方法又用Response又给app封装了，最终返回的是一个Reponse对象，而中间件是在__call__方法和wsgi_app方法之间用SharedDataMiddleware又包装了一层，但不管怎么样，只要它是可调用的即可，因为满足了PEP规范。也正是如此，可以不断利用中间件包装app以提供更多的功能。

上面的app虽然简单，但已经足以运行，利用WSGI服务器可以把这个app跑起来。

```shell
$ python shortly.py
 * Running on http://127.0.0.1:5000/
 * Restarting with reloader: stat() polling
```

### 第三步：环境
既然基本的架构已经弄好了，现在可以提供模板环境了，模板用于渲染html页面。

```python
def __init__(self, config):
    self.redis = redis.Redis(config['redis_host'], config['redis_port'])
    template_path = os.path.join(os.path.dirname(__file__), 'templates')
    self.jinja_env = Environment(loader=FileSystemLoader(template_path),
                                 autoescape=True)

def render_template(self, template_name, **context):
    t = self.jinja_env.get_template(template_name)
    return Response(t.render(context), mimetype='text/html')
```

利用jinja模板引擎给架构提供模板渲染能力。如果前后端分离，可以不使用jinja模板。

### 第四步：路由
werkzeug提供的路由只实现了url和endpoint的一对一映射，而endpoint和处理函数的映射并没有实现，因此需要自行实现。这也是为什么flask需要自行实现路由的原因，flask的路由是基于werkzeug的路由，提供了endpoint和处理函数的一对一映射。

```python
def __init__(self, config):
    self.redis = redis.Redis(config['redis_host'], config['redis_port'])
    template_path = os.path.join(os.path.dirname(__file__), 'templates')
    self.jinja_env = Environment(loader=FileSystemLoader(template_path),
                                 autoescape=True)
    self.url_map = Map([
        Rule('/', endpoint='new_url'),
        Rule('/<short_id>', endpoint='follow_short_link'),
        Rule('/<short_id>+', endpoint='short_link_details')
])
```

上面说了，werkzeug只实现了url和endpoint的映射，因此我们需要自己实现endpoint和处理函数的映射：

```python
def dispatch_request(self, request):
    # 这里的request是Request包装environ后的实例
    adapter = self.url_map.bind_to_environ(request.environ)
    try:
        # 根据请求的url匹配endpoint，如请求路径为http://localhost:5000/foo，则得到如下数据
        # endpoint = 'follow_short_link' values = {'short_id': u'foo'}
        endpoint, values = adapter.match()
        return getattr(self, 'on_' + endpoint)(request, **values)
    except HTTPException, e:
        return e
```

`getattr(self, 'on_' + endpoint)(request, **values)`这就是我们自行实现的endpoint和处理函数的映射。

### 第五步：第一个页面
```python
def on_new_url(self, request):
    error = None
    url = ''
    if request.method == 'POST':
        url = request.form['url']
        if not is_valid_url(url):
            error = 'Please enter a valid URL'
        else:
            short_id = self.insert_url(url)
            return redirect('/%s+' % short_id)
    return self.render_template('new_url.html', error=error, url=url)
```

验证url：

```python
def is_valid_url(url):
    parts = urlparse.urlparse(url)
    return parts.scheme in ('http', 'https')
```

在redis插入url信息：

```python
def insert_url(self, url):
    short_id = self.redis.get('reverse-url:' + url)
    if short_id is not None:
        return short_id
    url_num = self.redis.incr('last-url-id')
    short_id = base36_encode(url_num)
    self.redis.set('url-target:' + short_id, url)
    self.redis.set('reverse-url:' + url, short_id)
    return short_id
```

url编码：

```python
def base36_encode(number):
    assert number >= 0, 'positive integer required'
    if number == 0:
        return '0'
    base36 = []
    while number != 0:
        number, i = divmod(number, 36)
        base36.append('0123456789abcdefghijklmnopqrstuvwxyz'[i])
    return ''.join(reversed(base36))
```

### 第六步：重定向视图
```python
def on_follow_short_link(self, request, short_id):
    link_target = self.redis.get('url-target:' + short_id)
    if link_target is None:
        raise NotFound()
    self.redis.incr('click-count:' + short_id)
    return redirect(link_target)
```

这里通过short_id从redis取出url进行重定向

### 第七步：详情视图
```python
def on_short_link_details(self, request, short_id):
    link_target = self.redis.get('url-target:' + short_id)
    if link_target is None:
        raise NotFound()
    click_count = int(self.redis.get('click-count:' + short_id) or 0)
    return self.render_template('short_link_details.html',
        link_target=link_target,
        short_id=short_id,
        click_count=click_count
    )
```

通过模板渲染对应的视图。

### 第八步：模板文件
layout.html
```html
<!doctype html>
<title>{% block title %}{% endblock %} | shortly</title>
<link rel=stylesheet href=/static/style.css type=text/css>
<div class=box>
  <h1><a href=/>shortly</a></h1>
  <p class=tagline>Shortly is a URL shortener written with Werkzeug
  {% block body %}{% endblock %}
</div>
```

new_url.html
```html
{% extends "layout.html" %}
{% block title %}Create New Short URL{% endblock %}
{% block body %}
  <h2>Submit URL</h2>
  <form action="" method=post>
    {% if error %}
      <p class=error><strong>Error:</strong> {{ error }}
    {% endif %}
    <p>URL:
      <input type=text name=url value="{{ url }}" class=urlinput>
      <input type=submit value="Shorten">
  </form>
{% endblock %}
```

short_link_details.html
```html
{% extends "layout.html" %}
{% block title %}Details about /{{ short_id }}{% endblock %}
{% block body %}
  <h2><a href="/{{ short_id }}">/{{ short_id }}</a></h2>
  <dl>
    <dt>Full link
    <dd class=link><div>{{ link_target }}</div>
    <dt>Click count:
    <dd>{{ click_count }}
  </dl>
{% endblock %}
```

### 第九步：样式文件
/static/style.css
```css
body        { background: #E8EFF0; margin: 0; padding: 0; }
body, input { font-family: 'Helvetica Neue', Arial,
              sans-serif; font-weight: 300; font-size: 18px; }
.box        { width: 500px; margin: 60px auto; padding: 20px;
              background: white; box-shadow: 0 1px 4px #BED1D4;
              border-radius: 2px; }
a           { color: #11557C; }
h1, h2      { margin: 0; color: #11557C; }
h1 a        { text-decoration: none; }
h2          { font-weight: normal; font-size: 24px; }
.tagline    { color: #888; font-style: italic; margin: 0 0 20px 0; }
.link div   { overflow: auto; font-size: 0.8em; white-space: pre;
              padding: 4px 10px; margin: 5px 0; background: #E5EAF1; }
dt          { font-weight: normal; }
.error      { background: #E8EFF0; padding: 3px 8px; color: #11557C;
              font-size: 0.9em; border-radius: 2px; }
.urlinput   { width: 300px; }
```


这个教程简单介绍了werkzeug的基本用法。后面再探究教程里面的细节。
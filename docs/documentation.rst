.. _documentation:

====================
Django Piston's 文档
====================

.. toctree::
   :maxdepth: 4	

----
开始
----

入门Piston非常容易。用Piston写出来的API代码，无论是形式还是表现都与其他Django应用无异。
API代码使用URL映射一组handlers对资源进行定义。

（wrongway在这里强调一下：原文档部分文字含糊，一些概念混淆，比如handler，可能是一个Handler类，也可能是一个Hanlder实例对象，也可能是Handler类的read/update等方法，阅读时请注意）

在入门之前，建议您为API代码创建一个单独的目录，比如 'API' 。

我们的应用结构如下::

    urls.py
    settings.py
    myapp/
       __init__.py
       views.py
       models.py
    api/
       __init__.py
       urls.py
       handlers.py

接下来，在最上层urls.py中定义一个'namespace'，以对应API，如下::

    #!python
    
    urlpatterns = patterns('',
       # all my other url mappings
       (r'^api/', include('mysite.api.urls')),
    )

如上所设，包含API的urls.py将处理所有以'api/'开头的网址。

接下来将展示如何创建资源，以及如何将URL与资源挂钩。

---------------
资源(Resources)
---------------

资源(Resource)指的是代码中映射各种数据的实体。可以是博文、评论以及其他任何数据。

(Wrongway吐槽：其实这一节是讲Handler的...)

下面，我们在handlers.py中创建一个简单的handler::

    #!python

    from piston.handler import BaseHandler
    from myapp.models import Blogpost
    
    class BlogpostHandler(BaseHandler):
       allowed_methods = ('GET',)
       model = Blogpost   
    
       def read(self, request, post_slug):
          ...

Piston通过handler将资源与models进行映射，这个过程中Piston做了很多繁重的幕后工作，为开发者节省了大量精力。

Resource必须是一个类，通常情况下，Resource要实现下列四个方法中的一个或多个：

 :read: 由 **GET** 动作调用，获取对象而不做数据修改(该方法是幂等的)

 :create: 由 **POST** 动作调用，创建新对象(们)并返回该对象(们)(或是返回 :ref:`rc.CREATED`.)

 :update: 由 **PUT** 动作调用，更新某个已存在的对象并返回该对象(或是返回 :ref:`rc.ALL_OK`.)

 :delete: 由 **DELETE** 动作调用，删除某个已存的对象，只返回 :ref:`rc.DELETED` 。

除此之外，我们还可以定义其他我们所需的方法。只要在 ``fields`` 中填写方法名称，该方法就会被调用，
并在调用时自动传入 ``model`` 的实例做为参数。
该方法的返回值将用做该key(方法名称)的值，返回值不做限制，可以为任意类型。

**NB**: 上述自定义的 "resource methods" 应该被 @classmethod 修饰。因为使用这些自定义方法时Piston未必会对Handler进行实例化。
假设你已经定义了一个UserHandler类，并在该类中自定义了一个返回User对象的方法，在这种情况下，Piston是不会调用UserHandler实例的自定义对象方法，
只会调用UserHander类的自定义的类方法。

一个handler即可以表示单独的一个对象，也可以表示多个对象的集合，因此我们可以在read()方法中分别处理这两种情况::

    #!python
    
    from piston.handler import BaseHandler
    from myapp.models import Blogpost
    
    class BlogpostHandler(BaseHandler):
        allowed_methods = ('GET',)
        model = Blogpost   
    
        def read(self, request, blogpost_id=None):
            """
            Returns a single post if `blogpost_id` is given,
            otherwise a subset.
    
            """
            base = Blogpost.objects
            
            if blogpost_id:
                return base.get(pk=blogpost_id)
            else:
                return base.all() # Or base.filter(...)


----------------
发射器(Emitters)
----------------

发射器(Emitters) 表示输出的数据类型，可以是 YAML, JSON, XML, Pickle 或 Django ，分别对应 ``emitters.py`` 中的 ``XMLEmitter``, ``JSONEmitter``, ``YAMLEmitter``, ``PickleEmitter`` 和 ``DjangoEmitter`` 。

编写自己的emitters也很容易，要做的仅仅是创建一个继承 ``Emitter`` 的派生类，然后在其中创建 ``render`` 方法。
render方法接收一个 'request' 参数，该参数是一个请求(request)对象的拷贝。 多了解一下request.GET是很有必要的。 (比如定义回调，JSON的输出)

要将数据进行序列化/渲染，需要调用 ``self.construct()`` ，该方法始终返回一个数据字典，我们就可以对字典做任何想做的操作，再将其返回（返回值须是unicode字符串）。

**NB**: 可以用 ``Emitter.register`` 函式注册Emitters，同样也可以用 ``Emitter.unregister`` 函式来移除注册（假使你想移除一个内置emitter）。

内置的emitter注册::

    #!python
    
    class JSONEmitter(Emitter):
        ...
    
    Emitter.register('json', JSONEmitter, 'application/json; charset=utf-8')

自定义emitter时，可以引入Emitter模块并调用 'register' 对其注册，从而使自定义的emitter生效。
也可以利用同名数据类型(即传入的第一个参数)来覆写内置或是已存在的emitters。

上述实践，使得Piston添加其他形式的Emitter扩展变得非常容易，比如protocal buffers或是CSV。

可以通过 '?format=' GET参数(例如 '/api/blogposts/?format=yaml')为Emitter设置格式。不过新版本的Piston中，我们可以在URL映射中配置
'emitter_format' 关键字参数来设置Emitters（与'format'关键字并不冲突），如下::

    #!python
    
    urlpatterns = patterns('',
       url(r'^blogposts(?P<emitter_format>.+)$', ...),
    )

这样，/blogposts.json 就会使用 JSON emitter。

此外，我们还可以在URL映射中直接设置关键字参数来指定emitter格式::

    #!python
    
    urlpatterns = patterns('',
       url(r'^blogposts$', resource_here, { 'emitter_format': 'json' }),
    )

---------------------
URL映射(Mapping URLs)
---------------------

Piston的URL映射规则与Django无异。以BlogpostHandler为例：

在 urls.py 中::

    #!python
    
    from django.conf.urls.defaults import *
    from piston.resource import Resource
    from mysite.myapp.api.handlers import BlogpostHandler
    
    blogpost_handler = Resource(BlogpostHandler)
    
    urlpatterns = patterns('',
       url(r'^blogpost/(?P<post_slug>[^/]+)/', blogpost_handler),
       url(r'^blogposts/', blogpost_handler),
    )

见上，任何对 /api/blogpost/some-slug-here/ 和 /api/blogposts/ 的访问都映射到BlogpostHandler，
分别对应同一个handler的两种不同数据集。
要注意:一个单独的handler可以处理单个对象，也可以处理多个对象的集合。

.. _anonymous_resources:

匿名资源(Anonymous Resources)
=============================

资源也可以是匿名的("anonymous")。匿名资源即是一种可以实例化的特殊资源，
它用于未经授权(未通过OAuth,Basic或是其他认证handler)的请求。

举个例子，本文前面的BlogpostHandler，允许匿名访问博文。
但我们并不想让匿名用户也能够创建/更新/删除博文，也不想让未经授权的用户看到所有字段。

类似的需求可以通过创建继承自AnonymousBaseHandler(而不是BaseHandler)的新handler来实现。
这样做可以省却很多繁重的工作量。

如下::

    #!python
    
    from piston.handler import AnonymousBaseHandler, BaseHandler
    
    class AnonymousBlogpostHandler(AnonymousBaseHandler):
       model = Blogpost
       fields = ('title', 'content')
    
    class BlogpostHandler(BaseHandler):
       anonymous = AnonymousBlogpostHandler
       # same stuff as before 

我们没必要为了使用匿名handlers，就设置一个继承自BaseHandler的代理handler("proxy handler")。
反而是象上例这般直接指向一个匿名资源，更为实用。

.. _working_with_models:

-------------------------------
使用Models(Working with Models)
-------------------------------

Piston可以绑定某个model，但并不依赖该model。这样做的好处很明显:

* 没有覆写read/create/update/delete时，Piston就提供适用的默认处理(前提是方法已经出现在 ``allow_methods`` 中)。
* 没必要必须指定 ``fields`` 和 ``exclude`` (但你可以这么做，因为它们之间并不是互斥的)
* 如果已经在某个handler中使用了某个model，那么Piston会记住该model的 ``fields``/``exclude`` 设置，并在同样返回该model的其他handlers中继续使用该设置(除非被覆写)。

正如我们之前所见的，在handler中绑定某个model就只需设置 ``model`` 类参数这么简单。

扩展阅读: `为什么Piston要记住之前的handlers所记录的fields信息 <http://bitbucket.org/jespern/django-piston/wiki/FAQ#why-does-piston-use-fields-from-previous-handlers>`_

----------------------------------
配置Handlers(Configuring Handlers)
----------------------------------

Handlers使用用四个参数进行配置。


Model
=====

用于绑定model，查看 :ref:`working_with_models`.

.. _fields_and_exclude:

Fields/Exclude
==============

返回的数据中应包含和排除的字段列表。允许内嵌，可以是外键字段以及多对多字段。

也可以是编译后的正则表达式，例如::

    #!python
    import re
    
    class FooHandler(BaseHandler):
        fields = ('title', 'content', ('author', ('username', 'first_name')))
        exclude = ('id', re.compile('^private_'))

用户可以通过Many2many/ForeignKey字段访问博文，如下::

    class UserHandler(BaseHandler):
        model = User
        fields = ('name', ('posts', ('title', 'date')))

返回的数据会包含用户名称以及该用户发布的博文标题和日期。

对于fields中列表为空的内嵌资源，Piston会使用默认的handler，如下::

    class PostHandler(BaseHandler):
        model = Post
        exclude = ('date',)
    
    class UserHandler(BaseHandler):
        model = User
        fields = ('name', ('posts', ()))

UserHandler会显示一个用户所有博文的所有字段，但不包括博文的发布日期date。

``fields`` 和 ``exclude`` 都不是必须的，二者皆无时Piston也可以使用。

Anonymous
=========

指向可替代的匿名资源。查看 :ref:`anonymous_resources`

--------------------
认证(Authentication)
--------------------

Piston通过一个简单的接口支持可插换的认证，自带两种内置的认证机制，分别是 ``piston.authentication.HttpBasicAuthentication`` 和 ``piston.authentication.OAuthAuthentication`` 。
Basic auth handler非常简单，如果我们要编写自有的认证handler，它无疑很有参考价值。

**Note**: 在apache或nginx下使用 ``piston.authentication.HttpBasicAuthentication`` ，须要在服务器或虚拟主机配置时添加
``WSGIPassAuthorization On`` 指令，否则django-piston不会
从 ``request.META`` 的 ``HTTP_AUTHORIZATION`` 中读取认证数据。详看: http://code.google.com/p/modwsgi/wiki/ConfigurationDirectives#WSGIPassAuthorization.

认证Handler必须是一个类，且必须实现 ``is_authenticated`` 和 ``challenge`` 两个方法。

``is_authenticated`` 只接收一个参数，即Django接收的 ``request`` 对象的一个拷贝。
该对象包含认证一个用户所需的全部信息，比如 ``request.META.get('HTTP_AUTHENTICATION')``.

认证成功之后，该函式必须将 ``request.user`` 设为正确的 ``django.contrib.auth.models.User`` 对象。
这样做是考虑到随后的handlers要确认是哪个用户已经登录。

该函式必须返回True或False，以表示该用户是否登录成功。

对于认证失败的情况，就交由 ``challenge`` 处理。

``challenge`` 不接收任何参数，必须返回一个包含正确cahllenge指令的 ``HttpResponse`` 对象。
对于Basic auth来说，该函式返回一个带有 ``WWW-Authenticate`` 报头的，状态码为401的空响应(response)。
并通知接收端要提供认证信息。

针对匿名handlers，Piston提供了一个特珠的类： ``piston.authentication`` 中的 ``NoAuthentication`` 。
该类始终返回 ``is_authenticated`` 为 True。

OAuth
=====

OAuth是首选的认证方式，因为它能分辨消费者"consumers"，也就是说，它能识别使用该API的已认证的应用。
Piston深知和尊重这一点，并对此善加利用，比如当我们使用 @throttle 装饰器时，
它于底层限制每一个消费者应用，维系服务运转，即使当中某个服务已经达到限制配额。

-------------------------
表单验证(Form Validation)
-------------------------

Django拥有一套出色的内置表单验证机制，Piston对此善加利用。

我们可以对某个方法使用@validate装饰器，该装饰器接收两个参数。
第一个参数是必须的，即用于验证的表单，第二个参数是可选的，即提交数据的动作。
对于 ``create`` ，默认的动作是 'POST' ，对于 ``update`` ，动作就是 'PUT' 。

举个例子(使用ModelForm)::

    #!python
    
    from django import forms
    from piston.utils import validate
    from mysite.myapp.models import Blogpost
        
    class BlogpostForm(forms.ModelForm):
        class Meta:
            model = Blogpost
	...

    @validate(BlogpostForm)
    def create(request, ...):
        ...

使用一个普通form::

    #!python
    
    from django import forms
    from piston.utils import validate
    
    class DataForm(forms.Form):
        data = forms.CharField(max_length=128)
        is_private = forms.BooleanField(default=True, required=False)

    ...

    @validate(DataForm, 'PUT')
    def update(...):
        ...


若某个方法被@validate装饰，那么发送给该方法的数据如果没有通过表单本身的 ``is_clean`` 方法验证，
Piston就会禹客户端返回错误，而不运行任何操作。如果通过了验证，
表单对象就会附加到请求(request)对象中。
然后我们可以通过 ``request.form`` 获取表单(可以进一步通过cleaned_data取得验证后的数据)，如下例::

    #!python
    
    @validate(EchoForm, 'GET')
    def read(self, request):
        return {'msg': request.form.cleaned_data['msg']}


-------------------------------------------------------
辅助方法，工具类 & 装饰器(Helpers, utils & @decorators)
-------------------------------------------------------

出于方便，Piston提供了一组辅助和工具方法。
其中一个便是 ``piston.utils`` 中的 ``rc`` ，它包含了一组标准的响应返回。
在Piston的动作中做为响应码返回给客户端，以表示某个特定的状态。

但在最新版本的Piston， ``rc`` 返回的则是一个 **新的** HttpResponse的实例（此前版本仅仅是返回状态码），
使用如下::

    #!python
    
    resp = rc.CREATED
    resp.write("Everything went fine!")
    return resp
    
    resp = rc.CREATED
    resp.write("This will not have the previous 'fine' text in it.")
    return resp


新版本的Piston在返回码上的更改是后端兼容的，因为Piston仅仅是覆写了 ``__getattr__`` 方法以返回一个新实例，而非之前的符号。

==================   ================================   ======================
变量(Variable)       结果(Result)                       描述(Description)
==================   ================================ 	======================
rc.ALL_OK            200 OK                           	操作成功(Everything went well)
rc.CREATED           201 Created			对象创建成功(Object was created)
rc.DELETED           204 (Empty body, as per RFC2616)	对象删除成功(Object was deleted)
rc.BAD_REQUEST       400 Bad Request                    客户端请求错误或无法理解(Request was malformed/not understood)
rc.FORBIDDEN         401 Forbidden                   	权限不足(Permission denied)
rc.NOT_FOUND         404 Not Found                    	资源未找到(Resource not found)
rc.DUPLICATE_ENTRY   409 Conflict/Duplicate           	对象已存在(Object already exists)
rc.NOT_HERE          410 Gone                         	对象不存在(Object does not exist)
rc.NOT_IMPLEMENTED   501 Not Implemented              	动作不可用(Action not available)
rc.THROTTLED         503 Throttled                    	客户端请求受限(Request was throttled)
==================   ================================ 	======================

--------------------------
限制客户端请求(Throttling)
--------------------------

有些场合，我们并不想让用户在短时间内过于频繁地调用某个动作。
这时，可以让Piston在全局的基础层面上限制用户请求，在限制期限内有效地阻止客户端访问。

Piston在使用OAuth的情况下，会利用OAhth在每个消费者应用的底层对客户端访问进行限制。
若没有使用OAuth，受限规则会取决于已登录用户；对于匿名请求，受限规则则取决于客户端IP地址。

限制客户端请求是通过添加@throttle装饰器实现的。该装饰器有两个必填参数和一个可选参数。

第一个参数是某个时间段内允许的请求数量。

第二个参数就是该时间段的时长，以秒为单位。

第三个参数是一个可选的字符串，
会被添加到缓存key中。我们可以利用该参数为一组或一个动作进行限制。不提供该参数的话，Piston默认只对当前动作进行限制。

举个例子::

    #!python
    
    @throttle(5, 10*60)
    def create(...):


在这个例子中，客户端在10分钟内调用'create'的次数会限制在5次。

也可以对动作分组限制::

    #!python
    
    @throttle(5, 10*60, 'user_writes')
    def create(...):
    
    @throttle(5, 10*60, 'user_writes')
    def update(...):

----------------------------------
生成文档(Generating Documentation)
----------------------------------

公开发布API时提供相应的API使用文档。写文档本来就很沉闷繁琐，更改代码后再同步更新文档就更是如此。

幸运地是，Piston为此已经做了很多工作，使得我们可以方便地书写文档。

``piston.doc`` 库提供了一组方法，方便我们使用Django的Views和Template生成文档。

``generate_doc`` 函式返回一个 ``HandlerDocumentation`` 实例，该实例有下列几个方法::

* .model (get_model) 返回该handler的名称。
* .doc (get_doc) 返回给定的handler的docstring。
* .get_methods 返回一组可用的方法列表。该方法接收一个可选的关键字参数 ``include_defaults`` (默认为False)，在参数为True且并没有重写默认方法的情况下，该方法的返回列表会包含默认方法。我们想使用默认方法并想将其包含在文档中时，该参数就能派上用场。

``get_methods`` 包含一系列有趣的 ``HandlerMethod`` 。

* .signature (get_signature) 返回 //signature// 方法, 前两个固定参数是Piston自动指定的 'self' 和 'request' 。客户端无须指定这两个参数，所以不必关心。还有一个可选的关键字参数 ``parse_optional`` (默认为 True)，用来将默认为None的关键字参数转变成 "<optional>" 。
* .doc (get_doc) 返回某个动作的docstring，所以我们应该保证handler/action都有详细的文档描述。
* .iter_args() 会返回一个结构为(参数名，默认参数值/None)的二元组。如果默认参数值也是None，该参数值会被转换成字符串'None'。这样做是为了避免和None相混淆.因此我们就能区分返回的默认参数值终究是None还是空。

例子::

    #!python
    
    from piston.handler import BaseHandler
    from piston.doc import generate_doc
    
    class BlogpostHandler(BaseHandler):
        model = Blogpost
    
        def read(self, request, post_slug=None):
            """
            Reads all blogposts, or a specific blogpost if
            `post_slug` is supplied.
            """
            ...
    
        @staticmethod
        def resource_uri():
            return ('api_blogpost_handler', ['id'])
    
    doc = generate_doc(BlogpostHandler)
    
    print doc.name # -> 'BlogpostHandler'
    print doc.model # -> <class 'Blogpost'>
    print doc.resource_uri_template # -> '/api/post/{id}'
    
    methods = doc.get_methods()
    
    for method in methods:
        print method.name # -> 'read'
        print method.signature # -> 'read(post_slug=<optional>)'
    
        sig = ''
    
        for argn, argdef in method.iter_args():
            sig += argn
    
            if argdef:
                sig += "=%s" % argdef
    
            sig += ', '
    
        sig = sig.rstrip(",")
        
    print sig # -> 'read(repo_slug=None)'
    

资源URL(Resource URIs)
======================

每个资源都可以对应一个URI。调用资源的 .resource_uri() 方法就可以在Handler中访问该资源。

详见 [[FAQ#what-is-a-uri-template|FAQ: What is a URI Template]].

-----------
测试(Tests)
-----------

Piston的初始测试案例放在在tests/下。要运行该测试需要安装zc.buildout，并使用zc.buildout创建一个Django下的隔离环境。
该案例有两组测试，分别是 tests/bin/test-1.0 和 tests/bin/test-1.1 ，各自对应不同版本的Django。完成下面两个步骤就可以运行测试。

运行测试很简单::

    $ python bootstrap.py 
    Creating directory './bin'.
    [snip]
    Generated script './bin/buildout'.
    
    $ ./bin/buildout -v
    Develop: 'tests/..'
    Getting distribution for 'djangorecipe'.
    Got djangorecipe 0.17.3.
    Getting distribution for 'zc.recipe.egg'.
    Got zc.recipe.egg 1.2.2.
    Uninstalling django-1.0.
    Installing django-1.0.
    django: Downloading Django from: http://www.djangoproject.com/download/1.0.2/tarball/
    Generated script './bin/django-1.0'.
    Generated script './bin/test-1.0'.
        
    $ ./bin/test-1.0
    Creating test database...
    [snip]
    ...
    ----------------------------------------------------------------------
    Ran 6 tests in 0.283s
    
    OK
    Destroying test database...


运行buildout时一定要加上-v参数，之所以这么做，是因为不使用 "-v" 参数的话，在django下创建测试脚本时会导致脚本一直挂起。

如果您愿意为Piston贡献力量，欢迎您添加更多测试。因为当前测试仅仅覆盖了基本操作，还有很多欠缺之处。

------------------------
接收数据(Receiving data)
------------------------

Piston运行在HTTP协议上，对于post的数据处理得很好，对于其他格式(JSON或YAML)的数据也处理得不错。

Piston既能接收链值对数据，也很容易接收结构化数据。
Piston会尝试通过一组"loader"来反序列化传入的非form数据，这些"loader"取决于客户端指定的内容类型(Content-type)。

举个例子，如果我们给一个handler发送JSON数据，JSON内容类型(Content-Type)是 "application/json" ，Piston会做下面两件事情：

# 将反序列化后的数据存放在 ``request.data`` 中
# 将 ``request.content_type`` 设为 ``application/json`` 。对于form提交的数据而言，``request.content_type`` 则总是 None.

可以如下这般使用 (查看  `testapp/handlers.py <http://bitbucket.org/jespern/django-piston/src/7042cd328873/tests/test_project/apps/testapp/handlers.py>`_)::

    #!python

        def create(self, request):
            if request.content_type:
                data = request.data
                
                em = self.model(title=data['title'], content=data['content'])
                em.save()
                
                for comment in data['comments']:
                    Comment(parent=em, content=comment['content']).save()
                    
                return rc.CREATED
            else:
                super(ExpressiveTestModel, self).create(request)
    
如果我们传送下列JSON数据，也可以被很好的处理::

    #!python
    
    {"content": "test", "comments": [{"content": "test1"}, {"content": "test2"}], "title": "test"}


handler接收的任何数据都会被正常的反序列化，所以只要内容一致，无论格式是YAML还是XML，handler都能正常处理。

如果handler不接受客户端发送的数据（可能需要其他格式），可以用 ``utils.require_mime`` 装饰器来指定某种数据类型来解决这个问题。

该装饰器可以接收四种数据格式，以其短语名称来指定，分别是 'json', 'yaml', 'xml' 和 'pickle' 。

比如::

    #!python
    class SomeHandler(BaseHandler):
        @require_mime('json', 'yaml')
        def create(...
    
        @require_extended
        def update(...

.. _streaming:

-------------
流(Streaming)
-------------

Piston支持流输出至客户端。不过默认情况下该功能是被禁用的，原因是:

* Django下的流输出会中止 ``ConditionalGetMiddleware`` 和 ``CommonMiddleware`` 。

要绕过这一点不利因素，Piston提供了两个代理中间件("proxy middleware classes")，它们在流输出的情况下不会运行，因此不会在客户端收到数据之前
查看和截断数据。如果不使用这两个中间件，Django就会跟踪输出的内容(以计算E-Tags和Content-Length)，因此会导致随后的接收的peek内容为空。

在 ``piston.middleware`` 中有两个类用于替换 ``ConditionalGetMiddleware`` 和 ``CommonMiddleware`` 。

在 settings.py::

    #!python
    
    MIDDLEWARE_CLASSES = (
       # ...
        'piston.middleware.ConditionalMiddlewareCompatProxy',
        'piston.middleware.CommonMiddlewareCompatProxy',
       # ...
    )


支除任何对 ``ConditionalGetMiddleware`` 和 ``CommonMiddleware`` 的引用，或者令这两中间件无效。
若有其他中间件需要在流输出之前查看数据，也要使用代理中间件对其进行封装，如下::

    #!python
    
    from piston.middleware import compat_middleware_factory
    
    class MyMiddleware(...):
        ...
    
    MyMiddlewareCompatProxy = compat_middleware_factory(MyMiddleware)


然后在setting.py设置 ``MyMiddlewareCompatProxy`` 以取代原有中间件。

-------------------------------
配置项(Configuration variables)
-------------------------------

Piston 可以使用如下几个配置项来在某些方面做精细地控制，而不必修改代码。

==============================   ==========
Setting                          Meaning
==============================   ==========
settings.PISTON_EMAIL_ERRORS	 如果Piston崩溃，是否给管理员发邮件记录回溯信息(类似Django在 DEBUG = True 时会显示调试信息一样)
settings.PISTON_DISPLAY_ERRORS   Piston崩溃之后, 是否在客户端显示简短的回溯信息（包括预期的函数名）
settings.PISTON_STREAM_OUTPUT    为True时，Piston会通知Django将内容流输出至客户端，不过之前请详细阅 :ref:`streaming` 。
==============================   ==========

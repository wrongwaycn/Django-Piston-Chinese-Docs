.. Django Piston documentation master file, created by
   sphinx-quickstart on Sat Jul 23 19:25:38 2011.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

欢迎查看Django Piston文档
=========================

当前稳定版本为0.2.2 <http://bitbucket.org/jespern/django-piston/downloads/django-piston-0.2.2.tar.gz>`_.

**NEW**: 讨论组托管于 `Google Groups <http://groups.google.com/group/django-piston>`_.

生产版本
--------

当前开发版版本为 ``0.3dev`` ，该文档正是根据此版本而建。

用于Django中创建RESTful API的轻型框架
-------------------------------------

Piston是一个相对轻型的Django应用，用于为网站创建应用编程接口(API)。

Piston有如下特点：

* 与Django原生机制相互结合
* 支持OAuth，同时也支持 Basic/Digest 和自定义认证
* 无须与models绑定，支持任何资源
* 支持JSON, YAML, Python Pickle & XML (和 `HATEOAS <http://blogs.sun.com/craigmcc/entry/why_hateoas>`_.)
* 自带一组易用性好，重用性强的Python类库
* 推崇正确地使用HTTP (还有状态码, ...)
* 内置表单验证 (可选，由Django本身实现)，对客户端请求进行限制等功能
* 支持流输出 :ref:`streaming <streaming>`, 只占用很少的内存
* 不影响用户原来的开发方式。

**NB**: Piston内置OAuth，但并不要求用户必须使用OAuth。
Piston附带了一些关于如何使用OAuth简单的例子，开发者可以做为参考。(consumer/token models,urls等等)

社区知识库
----------

通过社区知识库，开发者能够为Piston的发展提供新鲜空气，创建更多活跃的讨论。

安装
----

官方稳定版是 ``0.2.2`` ，主干代码托管在bitbucket上，由于本文档所描述的版本是开发版而非稳定版，因此在使用Piston时留意当前版本信息。

Jesper Noehr 另外创建了一个分支 : `django-piston-oauth2 <https://bitbucket.org/jespern/django-piston-oauth2>`_ 
该分支对oauth2提供了支持，同样也托管在 `Github <https://github.com/django-piston/django-piston-oauth2>`_

文档
----

.. toctree::
   :maxdepth: 4

   documentation

完整的例子
----------

例子比文档更有说服力：

 * urls.py::

     #!python
     
     from django.conf.urls.defaults import *
     from piston.resource import Resource
     from piston.authentication import HttpBasicAuthentication
     
     from myapp.handlers import BlogPostHandler, ArbitraryDataHandler
     
     auth = HttpBasicAuthentication(realm="My Realm")
     ad = { 'authentication': auth }
     
     blogpost_resource = Resource(handler=BlogPostHandler, **ad)
     arbitrary_resource = Resource(handler=ArbitraryDataHandler, **ad)
     
     urlpatterns += patterns('',
         url(r'^posts/(?P<post_slug>[^/]+)/$', blogpost_resource), 
         url(r'^other/(?P<username>[^/]+)/(?P<data>.+)/$', arbitrary_resource), 
     )

 * 与之对应的 handlers.py::

     #!python
     
     import re
     
     from piston.handler import BaseHandler
     from piston.utils import rc, throttle
     
     from myapp.models import Blogpost
     
     class BlogPostHandler(BaseHandler):
         allowed_methods = ('GET', 'PUT', 'DELETE')
         fields = ('title', 'content', ('author', ('username', 'first_name')), 'content_size')
         exclude = ('id', re.compile(r'^private_'))
         model = Blogpost
     
         @classmethod
         def content_size(self, blogpost):
             return len(blogpost.content)
     
         def read(self, request, post_slug):
             post = Blogpost.objects.get(slug=post_slug)
             return post
     
         @throttle(5, 10*60) # allow 5 times in 10 minutes
         def update(self, request, post_slug):
             post = Blogpost.objects.get(slug=post_slug)
     
             post.title = request.PUT.get('title')
             post.save()
     
             return post
     
         def delete(self, request, post_slug):
             post = Blogpost.objects.get(slug=post_slug)
     
             if not request.user == post.author:
                 return rc.FORBIDDEN # returns HTTP 401
     
             post.delete()
     
             return rc.DELETED # returns HTTP 204
     
     class ArbitraryDataHandler(BaseHandler):
         methods_allowed = ('GET',)
     
         def read(self, request, username, data):
             user = User.objects.get(username=username)
     
             return { 'user': user, 'data_length': len(data) }


以上就是一个完整例子。

寻求帮助
--------

Piston拥有良好的文档（wrongway吐槽：我看未必）以及一个正在不断完善的FAQ（wrongway再吐：没多少内容）。当您发现有用信息时请添加到FAQ中。

开始阅读 :ref:`documentation`

内容
----

.. toctree::
   :maxdepth: 2

   documentation
   faq

目录
====

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`


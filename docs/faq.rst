
FAQ
===

.. toctree::


How do I...
-----------

...

Piston为何要从之前的handler中获取fields信息？
---------------------------------------------

我们创建某个绑定model的handler时，Piston会自动通过metaclass对handler进行注册。接下来，
如果其他hander返回该model的一个对象，且这个handler本身并没有定义fields，那么Piston就会使用之前捆绑该model的handler中的fields来提供内容。

举个例子，下面这个handler::

    #!python
    
    class FooHandler(BaseHandler):
        model = Foo
        fields= ('id', 'title')


绑定了 'Foo' 。接下来::

    #!python
    
    def read(...):
       f = Foo.objects.get(pk=1)
       return { 'foo': f }


Piston将返回'f'的'id'和'title'字段。但这两个字段并不是我们想要的，就可以象下面这样定义字段::

    #!python
    
    class OtherHandler(BaseHandler):
        fields = ('something', ('foo', ('title', 'content')))
    
        def read(...):
            f = Foo.objects.get(pk=1)
            return { 'foo': f }


这样，Piston会返回'f'的'title'和'content'，而不是之前的'id'和'title'。关于fields内嵌的深度，Piston是不做限制的。

如果我们想重置metaclass和内嵌的fields，只要用 ('foo', ()) 就可以一次取出model实例的所有字段内容。
   
为什么即便我明确指定了id，但是Piston仍然会忽略'id'？
----------------------------------------------------

**NB**: 在新版本的Piston中， ``fields`` 指定的字段包含信息会覆盖 ``exclude`` 中的字段排除信息。

Piston排除models的ID，其充分理由是 *通常情况下* ID应该做为内部非公开属性，不应该暴露给用户。
在 *必须* 包含'id'的情况下，我们可以重设'exclude'来实现::

    #!python

    class SomeHandler(BaseHandler):
        exclude = ()

如果想覆写默认的全局设置，可以用下列的代码::

    #!python
    
    from piston.handler import BaseHandler, AnonymousBaseHandler
    
    BaseHandler.fields = AnonymousBaseHandler.fields = ()


什么是URI模板？
---------------

http://bitworking.org/projects/URI-Templates/] 定义了如何构造访问WEB资源的URL。根据下列URL模板：

``http://www.yourproject.com/api/post/{id}/``

以及变量值

``id := 1``

所得到的网址就是：

``http://www.yourproject.com/api/post/1/``

提示和技巧
----------

如 Stephan Preeker (http://groups.google.com/group/django-piston/browse_thread/thread/1ca2fd1c89f3df4e) 中提到的，
我们可以在已认证的handler中调用匿名handler，如下::

    #!python
    
    class handler( .. ) 
        def read(self, request, id): 
            self.anonymous.read(id=id) 


上述代码能够工作，是因为 ``self.anonymous`` 指向匿名handler，因此我们可以直接引用匿名方法。

上报错误
--------

见 http://bitbucket.org/jespern/django-piston/issues/


哪些项目在使用Piston？
----------------------

Piston最初在Bitbucket内部使用，随后将其开源。Bitbucket的API已经实现版本化，大家可以在 http://api.bitbucket.org/ 找到最新版本。

如果您感觉Piston还不错，不妨将您应用Piston的网站做为案例添加进来

* http://dpaste.de/ 使用piston提供简单的API，用于从桌面或TextMate粘贴代码片段。

* http://www.bachigua.com/ 使用Piston提供一组API，用于实现类似 "iGoogle" 的小组件门户 (用Django写的)

* http://urbanairship.com/ 将Piston用于它们的API，最初是为移动设备提供访问（比如iphone推送通知以及app内置付款）。

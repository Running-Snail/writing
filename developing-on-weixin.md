# 微信开发的那些事儿
最近我们开通了BigTech服务号，拥有各种各样的接口权限，再加上微信这个平台，可以做很多很多有意思的东西。也因为推广产品和公众号的要求，不得不接触到了微信开发，然后把一些我自己真实的经历和经验在这里分享给大家。

根据开发者文档，获取公众号的 `access_token` 便是第一件要做的事。

获取access_token很简单，使用一个get的http请求就可以做到了

```
GET https://api.weixin.qq.com/cgi-bin/token?grant_type=client_credential&appid=APPID&secret=APPSECRET
```

为此纸豪封装了一个python的类 `Weixin` 使用requests库来获取，相关代码如下

``` python
def handle_response(self, response):
    return response.json()

def retrive_access_token(self):
    """\
        retrive the access token from weixin api(not cached)
        should "NOT" use this on your own
    """
    params = {
        'grant_type': 'client_credential',
        'appid': self.appid,
        'secret': self.appsecret
    }
    r = requests.get(urls.ACCESS_TOKEN, params=params)
    return self.handle_response(r)
```

但是呢，微信每日获取 `access_token` 的次数是有限制的，每天只能获取2000次，要是每次要用到 `access_token` 的地方都要去获取的话，2000次分分钟就用完了，而且会导致不同用户不同时间访问服务可能不可用的状态。

这就涉及到了缓存的话题，而且 `access_token` 很有可能在很多不同的地方用到。每过一段时间就去刷新缓存。

这里纸豪用的是memcache，他足够简单，也足够强大。

## Cache Daemon
缓存程序是一直要运行的，所以要写一个后台程序来运行，考虑到微信摇缓存的东西还很多，所以要使用多线程的方式来缓存，每一个线程控制一种资源的缓存。

daemon使用的是Sander Marechal's的开源Daemon代码。

我们要写自己的Daemon的话，只需要继承他的Daemon类就好了

``` python
class BigtechWeixinDaemon(Daemon):
    def __init__(self, pidfile, mem_addr):
        super(BigtechWeixinDaemon, self).__init__(pidfile)
        self.cache_threads = {}
        self.mem = memcache.Client(mem_addr)
```

对于每一种需要缓存的资源，使用用一个线程类来管理

``` python
class CacheThread(threading.Thread):
    def __init__(self, name):
        super(CacheThread, self).__init__()
        self.name = name
        self.running = True
        self.interval = 10  # 缓存间隔
        self.cache = None  # 缓存的数据
        self.controller = None  # 指的是daemon类的实例
```
重写的run函数里面需要依次调用 `update` 方法来更新缓存，用 `save_cache` 方法来存到memcache里面去，update函数里面来更新cache和interval的值

``` python
def update(self):
    raise NotImplementedError()

def save_cache(self):
    self.controller.get_memcache_client().set(self.name, self.cache)

def run(self):
    while self.running:
        try:
            self.update()
            self.save_cache()
            time.sleep(self.interval)
        except Exception, e:
            logging.exception(e)
            self.interval = 60
```
接下来只需要对应不同的资源继承CacheThread类，再重写update方法就好了，例如缓存access_token的代码如下

``` python
class AccessTokenCacher(CacheThread):
    NAME = 'bigtech_weixin_access_token'

    def __init__(self, appid, appsecret):
        super(AccessTokenCacher, self).__init__(
            name=AccessTokenCacher.NAME)
        self.appid = appid
        self.appsecret = appsecret
        self.weixin = Weixin(app_id=appid, app_secret=appsecret)

    def update(self):
        logging.info('start update access token')
        response = self.weixin.retrive_access_token()
        logging.info('retrive result')
        logging.info(response)
        if not Weixin.is_ok(response):
            logging.info('retrive access token failed, will retry in 30s')
            self.interval = 30
            return

        try:
            self.interval = response['expires_in']
            self.cache = response
        except KeyError:
            self.interval = 7200
```
好了，最后一件事就是把这些线程加入daemon里，在daemon启动的时候开启这些线程就好了。在BigtechWeixinDaemon里面添加这些方法

``` python
def add_cacher(self, cacher):
    self.cache_threads[cacher.name] = cacher
    cacher.set_controller(self)

def start_all_cacher(self):
    for name, thread in self.cache_threads.iteritems():
        thread.start()

def get_cache(self, name):
    return self.cache_threads[name].cache

def get_memcache_client(self):
    return self.mem

def run(self):
    self.start_all_cacher()
    while 1:
        time.sleep(60)
```
大家一定注意到了run里面的死循环和time.sleep

死循环好理解是为了让程序不要退出，但是如果光光写一个while 1: 再pass，实际在运行的时候，服务器的cpu占用率就会飙升到100%，加了time.sleep之后cpu占用率就下降到正常水平了。

## xml
通过看微信的开发者手册发现他们使用的都是xml格式的数据来来进行通讯的。读写xml很简单，无论是使用etree还是sax解析，都可以获取里面的数据。纸豪在实际开发的时候却踩了无数的坑，在这里把心酸史都讲述出来。

**ToUserName和FromUserName**。这两个可以从微信服务器发过来的数据里面获取到，在返回数据的时候，一定一定要记得这两个域的值要 **反过来** ，即返回的http response里面ToUserName的值要写在FromUserName里面，FromUserName的值要写在ToUserName里面。如果没有反过来会莫名其妙的收不到公众号返回的信息，也没有任何的提示。

**xml不是标准的xml**。标准的xml前面是要带 `<?xml version="1.0" encoding="UTF-8" ?>`的，但是微信接收的数据中不能包含这个标签，否则微信会提示你该公众号暂时无法提供服务。而如果大家使用任何xml writer的话，是很有可能在你不知情的情况下给你在xml前面加上这个标签。所以纸豪建议大家直接就使用字符串的模版来构建xml字符串，像这样

``` python
XML_TEMPLATE = (
    u'<xml>'
    u'<ToUserName><![CDATA[{to_user_name}]]></ToUserName>'
    u'<FromUserName><![CDATA[{from_user_name}]]></FromUserName>'
    u'<CreateTime>{create_time}</CreateTime>'
    u'<MsgType><![CDATA[text]]></MsgType>'
    u'<Content><![CDATA[{content}]]></Content>'
    u'</xml>'
)
```
再使用字符串的format就好了～

## match-handle 方式
微信中经常遇到的场景就是用户发给公众号一条消息，公众号回复相应的内容，并且这个需求会不停的变化，会有各种各样的匹配方式。纸豪这里采用的是一种match-handle的方式来处理。

所谓match-handle，就是针对发来的一条消息，根据消息的内容，类型，附加信息等进行匹配，让匹配上的handler来进行处理。其实就是定义一系列的规则，当消息符合某个规则时进行相应的处理。

MsgHandler类非常的简单，只是定义一个Handler的抽象类

``` python
class MsgHandler(object):
    def __init__(self):
        super(MsgHandler, self)

    def match(self, msg):
        raise NotImplementedError()

    def handle(self, msg):
        raise NotImplementedError()

    def handle_not_match(self, msg):
        raise NotImplementedError()
```
hanlde_not_match用来处理没有匹配的情况

有了MsgHandler就需要有管理MsgHandler的类，MsgHandlerController代码如下

``` python
class MsgHandleController(object):
    """docstring for MsgHandleController"""
    def __init__(self):
        super(MsgHandleController, self).__init__()
        self.handlers = []

    def add_handler(self, handler):
        self.handlers.append(handler)

    def handle(self, msg):
        for h in self.handlers:
            if h.match(msg):
                return h.handle(msg)
            else:
                try:
                    return h.handle_not_match(msg)
                except NotImplementedError:
                    pass
        return False
```
这个controller的设计就是一个列表管理，可以加入相应的handler，通过遍历handler对一个msg进行处理

这里是设计成只能有一个handler来处理msg（通过 return h.handle(msg)）

也可以设计成多个handler对一个msg进行处理

## 授权登录
授权登录也是微信的一个大头，通过它可以创建微信内的h5应用，获取用户的信息。

经过长时间纸豪的实践，授权登录可以被设计成一个独立的模块，每次有新的h5要使用到授权登录模块时，只需要使用这个模块的url就可以进行用户的授权。这样做的好处是非常方便的将网页授权加入到应用中去。而且也使这个微信公众号的用户保存在同一个数据库中，数据公用性好，而且减少了重复代码的书写，一旦成熟稳定下来了，就不需要怎么维护就可以保证功能的实现。

说到底，到底怎么做这个呢。

1. 首先需要一个url来发起授权登录（也就是跳转到微信的授权页面）
2. 其次需要一个处理用户授权的url（此时微信会发回来code，通过微信的接口来获取用户的信息，最后使用session来保存用户的数据）
3. 最后一步就是跳转回应用里

在第一步里，要进行用户是否已经授权过的判断（通过session来判断，微信的浏览器在微信没有退出的时候会保持自己的session会话，如果只是退出微信而不是杀掉微信进程的话，session会话是不会被关闭的）

第二步，获取用户信息的时候需要用之前说到已经缓存的 `access_token` 来获取web的 `access_token` ，再来获取用户的信息，最后装进session中去，**为了保证用户的信息安全，请使用加密过的服务器端session**

第三步，跳转回应用中去，在这一步需要注意，往回跳转的时候，有时候需要带一些参数在url上，此时就需要在认证处理的url中加入一个回跳时的参数

## JSSDK
使用jssdk的时候需要签名signature，noncestr，timestamp，这些数据的生成都需要后端来产生，生成signature的时候需要jssdk_ticket才能生成，而 `jssdk_ticket` 和 `access_token` 一样每天有获取次数的限制，所以跟 `access_token` 一样需要缓存。直接在之前提到过的缓存Daemon中添加相应的CacheThread即可。

在一些h5的项目合作中往往前端和后台需要协作完成jssdk的授权部分。纸豪觉得这一部分可以分离出一个独立的后台模版，在渲染的时候加入前端代码，在这里timestamp，noncestr，timestamp等都已经配置完毕，降低了前端和后台之间的沟通成本。

## 如何调试
调试微信是一个非常困难的事情，因为微信相对封闭，目前也只能在微信传输助手中输入url再来点击打开，才能在微信浏览器中打开相应的页面。

在使用了微信授权的h5中调试又会困难一分，因为此时页面已经没法在外部浏览器打开。

调试微信的要点，纸豪总结了以下几点

1. 多打印变量log，保存在后台的log文件中，调试的时候经常检查这些变量
2. 多进行单元测试，将一个个单元拆分出来测试要比整体测试要方便快速省力
3. 提供前端展示页，不需要用户授权，仅仅用于测试前端页面是否有bug

## 参考
[微信开发者文档](http://mp.weixin.qq.com/wiki/home/index.html)

[Sander Marechal's的开源Daemon代码](http://www.jejik.com/articles/2007/02/a_simple_unix_linux_daemon_in_python/)



tia---
layout: post
title: "python routes"
description: ""
category: 
tags: []
---
{% include JB/setup %}

## Ruotes简介

Routes解决了一个Web开发中经常遇到的问题，那就是怎么将url请求map到app，也就是，你怎么能说'/blog/2008/01/08'做这件事，而'/login'却做另外一个。许多Web框架都有一个内置的调度系统。比如，'/A/B/C'表示在目录B中读文件C，或者在'A.B'模块中调用B类的C方法。这样看，毫无问题。但是，当你要重新组织urls的时候，改动就会很大。

Routes却另辟蹊径。它将url层次和action分离，你可以按照你想要的方式去连接它们。如果你要改变一个特定的url，只要改变route map的一行代码，而不用改变action的逻辑。甚至你可以将多个url指向相同的action。Routes最早起源于Ruby on Rails，到现在已经有很大的不同了。

Ruotes是Pylons框架最初的调度系统，也是CherrPy的一种调度。所有的Web框架都能用它来处理整个url架构或者是url的subtree。它也可以将subtree指向其他的调度，TurboGear 2在Pylons就是这么实现的。

## 搭建Routes

Routes安装可以用pip或者easy_insatll，也可以下载源码用python setup.py install安装。

Pylons中，在*myapp/config/routing.py*模块的make_map函数中定义routes，下面是一个典型的配置：
{% highlight py linenos %}
from routes import Mapper
map = Mapper()
map.connect(None, "/error/{action}/{id}", controller="error")
map.connect("home", "/", controller="main", action="index")
# ADD CUSTOM ROUTES HERE
map.connect(None, "/{controller}/{action}")
map.connect(None, "/{controller}/{action}/{id}")
{% endhighlight %}
第1行和第2行创建一个mapper。

第3行匹配以'/error'开头的三层url，将controller参数设置成常量。url请求'/error/images/arrow.jpg'会产生：
{% highlight py linenos %}
{"controller": "error", "action": "images", "id": "arrow.jpg"}
{% endhighlight %}

第4行匹配'/', controller和action参数都设置成了常量，还有route名称'home'，它可以用来生成url。（其他route用None代替name。建议生成url的route都命名，其他route没有必要。）

第6行匹配任意的两层url，第7行匹配任意三层url。当我们不想为每个action都定义不同的route时，这么做是很有用的。当然，如果你为么个action都定义了route，就可以删掉这两行了。

注意，'/error/images/arrow.jpg'可以匹配第3行和第7行。mapper是按顺序匹配，所以这个url匹配第3行。

如果没有route能匹配url，mapper会返回'matach failed'，在Pylons就是'404 Not Found'。

Route path必须以'/'开头。

#### Requirements

Route path可以使用正则表达式。有两种方式：一是行内，二是使用requirements参数。
{% highlight py linenos %}
map.connect(R"/blog/{id:\d+}")
map.connect(R"/download/{platform:windows|mac}/{filename}")
{% endhighlight %} 
等同于：
{% highlight py linenos %}
map.connect("/blog/{id}", requirements={"id": R"\d+"}
map.connect("/download/{platform}/{filename}",
    requirements={"platform": R"windows|mac"})
{% endhighlight %} 

为了包含'\'，正则表达式使用了R""。如果不用R，就要使用'\\'。

{% highlight py linenos %}
m.connect("archives/{year}/{month}/{day}", controller="archives",
          action="view", year=2004,
          requirements=dict(year=R"\d{2,4}", month=R"\d{1,2}"))

NUMERIC = R"\d+"
map.connect(..., requirements={"id": NUMERIC})

ARTICLE_REQS = {"year": R"\d\d\d\d", "month": R"\d\d", "day": R"\d\d"}
map.connect(..., requirements=ARTICLE_REQS)
{% endhighlight %}

#### Conditions

Conditions决定匹配何种请求。conditions参数是一个字典，有三个key：
methon

大写的HTTP方法列表，请求的方法必须是其中某一种。

sub_domain

可以使subdomian列表，True，False或者None。如果是列表，请求必须调用列表中的subdomain。如果是True，请求必须包括任意一个subdomain。如果是False或者None，不匹配subdomain。

function

验证request的函数。它必须是func(envirion, match_dict) => bool这种类型。如果匹配成功返回True，否则返回False。第一个参数是WSGI环境变量，第二个参数是匹配成功后返回的变量。函数还可以改变mtach。

例子：
{% highlight py linenos %}
# Match only if the HTTP method is "GET" or "HEAD".
m.connect("/user/list", controller="user", action="list",
          conditions=dict(method=["GET", "HEAD"]))

# A sub-domain should be present.
m.connect("/", controller="user", action="home",
          conditions=dict(sub_domain=True))

# Sub-domain should be either "fred" or "george".
m.connect("/", controller="user", action="home",
          conditions=dict(sub_domain=["fred", "george"]))

# Put the referrer into the resulting match dictionary.
# This function always returns true, so it never prevents the match
# from succeeding.
def referals(environ, result):
    result["referer"] = environ.get("HTTP_REFERER")
    return True
m.connect("/{controller}/{action}/{id}",
    conditions=dict(function=referals))
{% endhighlight %}

#### Wildcard routes

path变量默认情况不匹配'/'。这是为了保证每个path精确匹配某一层url。你可以用requirements去覆盖：
{% highlight py linenos %}
map.connect("/static/{filename:.*?}")
{% endhighlight %}
匹配'/static/foo.jpg'，'/static/bar/foo.jpg'等url。

#### Format extensions

path中包含'{.format}'会匹配可选的扩展名（如.html或者.json），在'.'之后设置format变量。例如：
{% highlight py linenos %}
map.connect('/entries/{id}{.format}')
{% endhighlight %}
匹配'/entries/1'和'/entires/1.mp3'。你可以用requirements去限制匹配的扩展名：
{% highlight py linenos %}
map.connect('/entries/{id:\d+}{.format:json}')
{% endhighlight %}
匹配'/entries/1'和'/entires/1.json'，但是不匹配'/entires/1.mp3'。

#### Submappers


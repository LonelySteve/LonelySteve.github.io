---
layout: post
title: 有关 Requests 库中多部分编码(Multipart-Encoded) 自定义 boundary 问题
tags:
 - 原创
 - Python 
 - Requests
 - boundary
date: 2018-07-29 # 强制回溯到正确的发布时间 
---

## 前言

这个问题其实在使用HTTP库时很少能碰到，一般人也没有这种需求，但是出于某些原因，较真的我在这里折腾了几个小时，一开始我没想到是 Requests 本身的问题，后来，我把它的源码给翻了出来：

<!-- more -->

```Python

    @staticmethod
    def _encode_files(files, data):
        """Build the body for a multipart/form-data request.

        Will successfully encode files when passed as a dict or a list of
        tuples. Order is retained if data is a list of tuples but arbitrary
        if parameters are supplied as a dict.
        The tuples may be 2-tuples (filename, fileobj), 3-tuples (filename, fileobj, contentype)
        or 4-tuples (filename, fileobj, contentype, custom_headers).
        """
        if (not files):
            raise ValueError("Files must be provided.")
        elif isinstance(data, basestring):
            raise ValueError("Data must not be a string.")

        # 一些别的blabla这里省略
        body, content_type = encode_multipart_formdata(new_fields)

        return body, content_type

```

Requests 在处理files参数时，最终是调用以上的函数进行处理的。

但是其实 encode_multipart_formdata() 函数的原型是这个样子的：

```Python

def encode_multipart_formdata(fields, boundary=None):
    """
    Encode a dictionary of ``fields`` using the multipart/form-data MIME format.

    :param fields:
        Dictionary of fields or list of (key, :class:`~urllib3.fields.RequestField`).

    :param boundary:
        If not specified, then a random boundary will be generated using
        :func:`mimetools.choose_boundary`.
    """
    body = BytesIO()
    if boundary is None:
        boundary = choose_boundary()

    for field in iter_field_objects(fields):
        body.write(b('--%s\r\n' % (boundary)))

        writer(body).write(field.render_headers())
        data = field.data

        if isinstance(data, int):
            data = str(data)  # Backwards compatibility

        if isinstance(data, six.text_type):
            writer(body).write(data)
        else:
            body.write(data)

        body.write(b'\r\n')

    body.write(b('--%s--\r\n' % (boundary)))

    content_type = str('multipart/form-data; boundary=%s' % boundary)

    return body.getvalue(), content_type


```

这个是urllib3库的函数，显然，它本身支持对boundary的自定义，原来Requests为了自身代码的简洁，默认了自动生成boundary这一行为，但是这可能会给使用Content Type进行自定义boundary的人造成困惑，Requests自身没有警告提示，在官方文档中也并未提及。

## 解决方案

### 一、手动修改 Requests 库的函数，让它支持在方法层面对boundary自定义

这个相对而言麻烦一些，你需要对Requests库本身的架构有所理解，但是其实也难不到哪去，我成功实现在标准Requests库中加入了boundary这一参数，例如，上述的Requests库中静态函数的定义被我修改成：

```Python

    @staticmethod
    def _encode_files(files, data ,boundary=None):
        """Build the body for a multipart/form-data request.

        Will successfully encode files when passed as a dict or a list of
        tuples. Order is retained if data is a list of tuples but arbitrary
        if parameters are supplied as a dict.
        The tuples may be 2-tuples (filename, fileobj), 3-tuples (filename, fileobj, contentype)
        or 4-tuples (filename, fileobj, contentype, custom_headers).
        """
        if (not files):
            raise ValueError("Files must be provided.")
        elif isinstance(data, basestring):
            raise ValueError("Data must not be a string.")

        # 一些别的blabla这里省略
        body, content_type = encode_multipart_formdata(new_fields,boundary=boundary)

        return body, content_type

```

然后层层传递这个boundary参数，你就可以在方法层面自定义boundary了

```Python

import requests,string,random

boundary="----WebKitFormBoundary" + "".join(random.sample(string.ascii_letters+string.digits,16))
headers={
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/67.0.3396.99 Safari/537.36",
    "Referer": "https://www.xxxx.com/account/face/upload",
    "Content-Type":"multipart/form-data; " + boundary,
    "Origin": "https://www.xxxx.com"
}

# 这里的data可以是open()打开的文件流，还可以是一切具有read()方法的对象
fields={"dopost":(None,"save"),"DisplayRank":(None,"10000"),"face":("blob",data,"image/jpeg")}

r = requests.post(account.PENDANT_UPDATEFACE,files=fields,headers=headers,boundary=boundary)  # 这里可以传递 boundary 参数了

print(r.text)

```

### 二、使用 requests-toolbelt 第三方库

其实这个问题早就有人提出来了，详见[#issues 1997](https://github.com/requests/requests/issues/1997)

根据官方的说法，他们认为requests可以帮你处理好headers，你不必要去自定义boundary，如果你执意这么做的话，可以使用[requests-toolbelt](https://gitlab.com/sigmavirus24/toolbelt)这个第三方库，它提供了相关的功能。

[requests-toolbelt](https://gitlab.com/sigmavirus24/toolbelt)使用pip即可安装：

    pip install requests-toolbelt

基本用法：

```Python

import requests
import random,string
from requests_toolbelt import MultipartEncoder

fields={'file':('test.png',data,"image/png"),'file_id':"0"} # 这里的data可以是open()打开的文件流，还可以是一切具有read()方法的对象

m=MultipartEncoder(fields=fields,boundary='----WebKitFormBoundary'+''.join(random.sample(string.ascii_letters+string.digits,16)))

headers={
    "Host":"xxxx",
    "Connection":"keep-alive",
    "Content-Type":m.content_type
}

r=requests.post('https://xxxx/api/upload',headers=headers,data=m)

print(r.text)

```

这是推荐的方案，值得一提的是，这个方案和我的方法是兼容的哦 :>
---
layout: post
title: "解决fetch跨域缺失header问题"
description: "fetch缺失header的原因和解决办法"
header_image: /assets/img/2019-05-08-01.png
keywords: "fetch-missing-header-when-cors"
tags: []
---
{% include JB/setup %}
![img](/assets/img/2019-05-08-01.png)

## 背景

前端js网络请求的时候经常会有跨域问题，可以通过让服务器支持跨域来解决。最近在开发Electron应用时遇到header丢失问题。

## Header为什么丢失

简单的请求通过`fetch`这个API非常方便，他是Promise形式的，相比node的`http/https`模块和传统的XMLHTTPRequest书写更简便。

根据API说明，他有以下限制：

>The Promise returned from fetch() won’t reject on HTTP error status even if the response is an HTTP 404 or 500. Instead, it will resolve normally (with ok status set to false), and it will only reject on network failure or if anything prevented the request from completing.

>By default, fetch won't send or receive any cookies from the server, resulting in unauthenticated requests if the site relies on maintaining a user session (to send cookies, the credentials init option must be set).
Since Aug 25, 2017. The spec changed the default credentials policy to same-origin. Firefox changed since 61.0b13.

除此之外，跨域的时候还会出现header缺失问题，只能返回部分header头

>If a request is made for a resource on another origin which returns the CORs headers, then the type is cors. cors and basic responses are almost identical except that a cors response restricts the headers you can view to `Cache-Control`, `Content-Language`, `Content-Type`, `Expires`, `Last-Modified`, and `Pragma`.

* Cache-Control
* Content-Language
* Content-Type
* Expires
* Last-Modified
* Pragma

如果你要获取`Content-Length`的话，就比较尴尬了。

笔者有一个场景如下：
>通过HTTTP的HEAD方法，判定一个URL资源是否可用，可用的话，文件大小是多少，这里就需要获取`Content-Length`，在测试工具和其他语言内请求都是正常，唯独使用`fetch`接口出问题，根据API介绍，可以给服务端配置`access-control-expose-headers`来解决，这对于无法控制后端代码的情况下，是不可接受的。

最终解决办法是放弃使用fetch，接口，手动包装XMLHTTPRequest为Promise：

```javascript
// fetch存在header跨域缺失问题，如果需要获取全部header则不能使用fetch
checkUrlByFetch = (link) => {
    return fetch(link, {
        method: "HEAD",
    }).then(response => {
        const headers = response.headers
        headers.forEach(console.log);
        const result = { ok: response.ok, link: link }
        if (response.ok) {
            console.log('有效链接:' + link)
            try {
                const length = headers.get('content-length');
                const contentType = headers.get('content-type');
                //application/vnd.android.package-archive
                console.log(`类型：${contentType}, 字节数:${length}`)
                result.type = contentType;
                result.length = length;
            } catch (error) {
                console.log('无效链接，忽略')
                console.error(error);
            }
        } else {
            console.log('无效链接，忽略:' + link)
        }
        return result;
    })
}

// 通过XMLHttpRequest获取，可以得到所有header
checkUrlByAjax = (link) => {
    return new Promise(resolve => {
        const request = new XMLHttpRequest();
        const result = { ok: false, link: link };
        request.onload = (e) => {
            // Get the raw header string
            const headers = request.getAllResponseHeaders();
            console.log(headers);
            if (request.status === 200) {
                const length = request.getResponseHeader('content-length');
                const contentType = request.getResponseHeader('content-type');
                console.log(`类型：${contentType}, 字节数:${length}`)
                result.type = contentType;
                result.length = length;
            }
            result.ok = request.status === 200;
        }
        request.onloadend = (e) => {
            resolve(result)
        }
        request.open('HEAD', link);
        request.send();
    })
}

```

## 小结

* [https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)
* [https://stackoverflow.com/questions/48413050/missing-headers-in-fetch-response](https://stackoverflow.com/questions/48413050/missing-headers-in-fetch-response)
* [https://developers.google.com/web/updates/2015/03/introduction-to-fetch#response_types](https://developers.google.com/web/updates/2015/03/introduction-to-fetch#response_types)
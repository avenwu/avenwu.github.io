---
layout: post
title: "Metaweblog在Android上使用"
description: "如何在Android平台使用MetaWeblog"
category: 
tags: [android,metaweblog]
---
{% include JB/setup %}

metaweblog是一个博客接口协议，目前主流的博客平台均支持该协议，比如博客园，CSDN，WordPress等。通过约定的协议可以不用登陆相应博客网站，直接用pc客户端直接发布博客文章。  
在android上当然也可以使用，利用xml-rpc的java实现库org.apache.xmlrpc:xmlrpc-client可以实现通信。

###配置
首先下载xmprpc及其依赖库，这里我用的是gradle管理依赖库：

	compile ('org.apache.xmlrpc:xmlrpc-client:3.1.3'){
        exclude module: 'xml-apis'
    }

由于xml-apis已经包含在android核心框架内，因此为了避免重复的依赖手动声明不包含即可。  
另外如果你的项目使用了其他的三方库，可能还会有一些错误，比如META-INF中的文件冲突：

	packagingOptions {
        exclude 'META-INF/DEPENDENCIES'
        exclude 'META-INF/LICENSE'
        exclude 'META-INF/NOTICE'
    }
    
一般都比较好处理直接根据错误信息做相应调整；

###使用
关于博客平台支持的协议接口一般都可以在官网找到入口，这里以博客园为例：

[http://www.cnblogs.com/services/metaweblog.aspx#Post](http://www.cnblogs.com/services/metaweblog.aspx#Post)

主要是对博客的增删查改几个借口比较重要：

- blogger.deletePost
- blogger.getUsersBlogs
- metaWeblog.editPost
- metaWeblog.getCategories
- metaWeblog.getPost
- metaWeblog.getRecentPosts
- metaWeblog.newMediaObject
- metaWeblog.newPost
- wp.newCategory

利用xmlrpc可以很方便的调用借口，根据要求传入不同的参数，下面有几个测试接口：

{% highlight java %}
private XmlRpcClient getClient() throws MalformedURLException{
    XmlRpcClientConfigImpl config = new XmlRpcClientConfigImpl();
    config.setServerURL(new URL("http://www.cnblogs.com/avenwu/services/metablogapi.aspx"));
    XmlRpcClient client = new XmlRpcClient();
    client.setConfig(config);
    return client;
}
public void testMetaWeblogGetCategories() throws MalformedURLException, XmlRpcException{
    Object data = getClient().execute("metaWeblog.getCategories", new Object[]{"test", "你的用户名", "你的密码"});
    CategoryInfo[] result = Parser.parseCategory(data);
    Log.d("testMetaWeblog", result.toString());
}
public void testMetaWeblogGetPost() throws MalformedURLException, XmlRpcException{
    Object data = getClient().execute("metaWeblog.getRecentPosts", new Object[]{"test","你的用户名","你的密码", 10});
    Post[] result = Parser.parseRecentPost(data);
    Log.d("testMetaWeblog", result.toString());
}
public void testMetaWeblogGetUsesBlogs() throws MalformedURLException, XmlRpcException{
    Object data = getClient().execute("blogger.getUsersBlogs", new Object[]{"test","你的用户名","你的密码"});
    BlogInfo[] result = Parser.parseBloInfo(data);
    Log.d("testMetaWeblog", result.toString());
}

public void testMetaWeblogNewPost()  throws MalformedURLException, XmlRpcException{
    Object data = getClient().execute("metaWeblog.newPost", new Object[]{"用户id","你的用户名","你的密码", getPost(), false});
    Log.d("testMetaWeblog", data.toString());
}
public Map<String, Object> getPost(){
    Map<String, Object> post = new HashMap<String, Object>();
    post.put("dateCreated", Calendar.getInstance().getTime());
    post.put("description","#Test Post with metaweblog");
    post.put("title", "Test");
    return post;
}

public void testMetaWeblogDeletePost()  throws MalformedURLException, XmlRpcException {
    Object data = getClient().execute("blogger.deletePost", new Object[]{"test","文章id","你的用户名","你的密码", false});
    Log.d("testMetaWeblog", data.toString());
}
{% endhighlight %}

###小结
相关资料不是很多，但是使用上其实并不难，因为apache已经做了封装。

###参考
1. [http://www.ibm.com/developerworks/library/x-metablog/](http://www.ibm.com/developerworks/library/x-metablog/)
2. [http://www.cnblogs.com/services/metaweblog.aspx#Post](http://www.cnblogs.com/services/metaweblog.aspx#Post)

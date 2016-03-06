---
layout: post
title: "再破博客园登录"
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-06-19.jpg
description: ""
category: 
tags: [博客园登陆模拟]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-06-19.jpg)

4月以来博客园悄然修改了登陆的接口，导致我等屁民开发的客户端生生登陆不了。趁着周末重新对登陆进行了抓包分析，总算搞定，可以歇一口气：）

## 分析

截止目前登陆页面地址是这样的[http://passport.cnblogs.com/user/signin?ReturnUrl=http%3A%2F%2Fwww.cnblogs.com%2F](http://passport.cnblogs.com/user/signin?ReturnUrl=http%3A%2F%2Fwww.cnblogs.com%2F)

眼尖的园友应该发现了第一个变化，即登陆地址成了use/signin。当然肯定不止这一出修改。

之前登陆采用的是表单提交，现在登陆请求采用了ajax，利用post方式将加密后的用户名，密码拼接成json串发给服务器。

## 处理
我是怎么知道的，当然不是猜的，

详细的js代码可以直接看到：

{% highlight javascript %}

            var encrypt = new JSEncrypt();
            encrypt.setPublicKey('MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCp0wHYbg/NOPO3nzMD3dndwS0MccuMeXCHgVlGOoYyFwLdS24Im2e7YyhB0wrUsyYf0/nhzCzBK8ZC9eCWqd0aHbdgOQT6CuFQBMjbyGYvlVYU2ZP7kG9Ft6YV6oc9ambuO7nPZh+bvXH0zDKfi02prknrScAKC0XhadTHT3Al0QIDAQAB');
            var encrypted_input1 = encrypt.encrypt($('#input1').val());
            var encrypted_input2 = encrypt.encrypt($('#input2').val());
            var ajax_data = {
                input1: encrypted_input1,
                input2: encrypted_input2,
                remember: $('#remember_me').prop('checked')
            };

            if(enable_captcha){
                var captchaObj = $("#captcha_code_input").get(0).Captcha;
                ajax_data.captchaId = captchaObj.Id;
                ajax_data.captchaInstanceId = captchaObj.InstanceId;
                ajax_data.captchaUserInput = $("#captcha_code_input").val();
            }
            is_in_progress = true;
            $.ajax({
                url: ajax_url,
                type: 'post',
                data: JSON.stringify(ajax_data),
                contentType: 'application/json; charset=utf-8',
                dataType: 'json',
                headers: {
                    'VerificationToken': 'cZ0PISksjsWGEbj4IhANzxSXoXmLr9zNWVBzTNuy5khrwm0akh5Eo9XTrmoHt_RzFOKbWD2jOaibj7r_bZZlPLAx81c1:G8OVJmEYy2z1FJJUwvvy_mS3HLR-AYitaaf3eCXHUI8LIwAPjxnpkXqgR32zqSRli_gid77jDtaUlOjGgob8TjdIOq41'
                },
                success: function (data) {                    
                    if (data.success) {
                        $('#tip_btn').html('登录成功，正在重定向...');
                        location.href = return_url;
                    } else {
                        $('#tip_btn').html(data.message + "<br/><br/>联系 contact@cnblogs.com");
                        is_in_progress = false;
                        if(enable_captcha)
                        {
                            captchaObj.ReloadImage();
                        }
                    }
                },
                error: function (xhr) {
                    is_in_progress = false;
                    $('#tip_btn').html('抱歉！出错！联系 contact@cnblogs.com');
                }
            });
    
{% endhighlight %}

## 用户名，密码加密处理
虽然不是很懂js，但是这并不妨碍分析，这里利用了<script src="/scripts/jsencrypt.min.js"></script>里面的加密函数，只知道大致用的是RSA加密，公钥的话js中已经给出了，直接拿来用就可以。本来想用java实现，加密函数内容太多，实现不容易，所以仍然用的这段原生的js加密、解密函数；

现在java中js库需要解决，查了下相关资料Moliza出了一个引擎Rhino，文档到是很多，执行一个简单的js语句应该没问题，但这里需要导入整个js库文件，没找的简单可行的方法，只能迂回前进，直接用weview来加密，写一段js代码，调用上面提到的js加密库，效果也不错，可以正常解析：
{% highlight java %}

    private final static String ENCRYPT = "javascript:encryptLoginInfo('%s','%s')";

    public void login() {
        WebView webview = new WebView(mContext);
        webview.getSettings().setJavaScriptEnabled(true);
        webview.addJavascriptInterface(new Android(), "android");
        webview.loadUrl(PAGE);
        webview.setWebViewClient(new WebViewClient() {
            @Override
            public void onPageFinished(WebView view, String url) {
                Logger.d("HTML URL=" + url);
                if (PAGE.equals(url) && !isLogining) {
                    view.loadUrl(String.format(ENCRYPT, mListener.setName(), mListener.setPassword()));
                }
            }
        });
    }
    
{% endhighlight %}

## 抓取头部数据
除了用户数据加密外，这里还需要添加一个头部VerificationToken,伪造是不可能了，只能访问登陆页，利用正则匹配出来。
{% highlight java %}

 Document doc = Jsoup.parse(html);
            String url = doc.select("#c_login_logincaptcha_CaptchaImage").attr("src");
            if (!TextUtils.isEmpty(url)) {
                Logger.d("skip auto login as captcha is needed");
                return "";
            }
            Element script = doc.select("script").get(2);
            Pattern p = Pattern.compile("(?is)'VerificationToken': '(.+?)'"); // Regex for the value of the key
            Matcher m = p.matcher(script.html()); // you have to use html here and NOT text! Text will drop the 'key' part
            String VerificationToken;
            if (m.find()) {
                VerificationToken = m.group(1);
            } else {
                return "";
            }
            
{% endhighlight %}

## 验证码抓取
同理夜间的验证码登陆也需要调整，因为这个也变了，但原理是一样的，分析html内的标签找到验证码所在的img标签，获取src即图片生成地址。

{% highlight java %}

 Document doc = Jsoup.parse(html);
            String url = doc.select("#LoginCaptcha_CaptchaImage").attr("src");
            if (TextUtils.isDigitsOnly(url)) {
                return params;
            }
            //BotDetect.Init('LoginCaptcha', '6c5b12a021d8460fa8bc87cdf96f0d80', 'captcha_code_input', true, true, true, true, 1200, 7200, 0, true);
            Matcher matcher = Pattern.compile("(?is)BotDetect.Init\\('LoginCaptcha', '(.+?)',").matcher(html);
            if (matcher.find()) {
                params.captchaInstanceId = matcher.group(1);
            }
{% endhighlight %}

## 结语
基本上每个细节点都变了，所以需要正对登陆环节重新分析，分析出每一个所需的元素后想办法模拟出来。
---
layout:     post
title:      "iOS 中使用模板引擎渲染 HTML 界面"
subtitle:   "ios-html-template-engine"
date:       2016-02-23
header-img: "img/bg8.jpg"
author:     "CMB"
tags:
    - iOS
    - HTML

---

在 `iOS` 实际的开发中，使用 `UIWebView` 来加载数据使用的场景特别多。很多时候我们会动态的从服务器获取一段 `HTML` 的内容，然后 `App` 这边动态的处理这段 `HTML` 内容用于展示在 `UIWebView` 上。使用到的 `API` 接口为：

```
- (void)loadHTMLString:(NSString *)string baseURL:(NSURL *)baseURL;
```

由于 `HTML` 内容通常是变化的，所以我们需要动态生成 `HTML` 代码。通常我们从服务器端获取到标题、时间、作者和对应的内容，然后我们需要对这些数据处理之后拼接成一段 `HTML` 字符串。对于传统的做法是将上面的需要替换的内容填写一些占位符，放到指定的文件中如（content.html）,如下所示：

```
<!DOCTYPE html>
<html>
    <head>
        <title>key_title</title>
    </head>
    <body>
        <div>
            <div>
                 <h2>key_title</h2>
                 <div>key_date key_author</div>
                 <hr/>
            </div>
            <div>key_content</div>
       </div>
    </body>
</html>
```

然后在指定的地方使用如下的方式动态生成 `HTML` 代码：

```
- (NSString *)loadHTMLByStringFormat:(NSDictionary *)data
{
    NSString *templatePath = [[NSBundle mainBundle] pathForResource:@"content" ofType:@"html"];
    NSMutableString *html = [[NSMutableString alloc] initWithContentsOfFile:templatePath encoding:NSUTF8StringEncoding error:nil];
    [html replaceOccurrencesOfString:@"key_title" withString:data[@"title"] options:NSCaseInsensitiveSearch range:NSMakeRange(0, html.length)];
    [html replaceOccurrencesOfString:@"key_author" withString:data[@"author"] options:NSCaseInsensitiveSearch range:NSMakeRange(0, html.length)];
    [html replaceOccurrencesOfString:@"key_date" withString:data[@"date"] options:NSCaseInsensitiveSearch range:NSMakeRange(0, html.length)];
    [html replaceOccurrencesOfString:@"key_content" withString:data[@"content"] options:NSCaseInsensitiveSearch range:NSMakeRange(0, html.length)];
    return html;
}
```

在实际的使用中发现还是存在不少的问题，比如我们需要对数据进行预先处理的时候需要写大量的

```
- (NSUInteger)replaceOccurrencesOfString:(NSString *)target withString:(NSString *)replacement options:(NSStringCompareOptions)options range:(NSRange)searchRange;
```

这样的替换，而且对于一些特殊的字符还需要进行特殊处理等，实在不是太友好，这样就需要一个引擎来专门处理这些事情，本文主要介绍 `MGTemplateEngine` 和 `GRMustache` 的使用。

### 使用模板引擎

#### MGTemplateEngine的使用

`MGTemplateEngine` 是 `Matt Gemmell` 的作品，它是一个比较流行的模板引擎，它的模板语言比较类似于 `Smarty` 、 `FreeMarker` 和 `Django` 。另外它可以支持自定义的 `Filter` （以便实现自定义的渲染逻辑），需要依赖正则表达式的工具类 `RegexKit` 。

#### 1、创建模板

```
<!DOCTYPE html>
<html>
    <head>
        <title></title>
    </head>
    <body>
        <div>
            <div>
                <div>
                    <h2></h2>
                    <div> </div>
                    <hr/>
                </div>
                <div><article class="post-container post-container--single" itemscope itemtype="http://schema.org/BlogPosting">
  <header class="post-header">
    <div class="post-meta">
      <time datetime="2015-01-10 20:16:21 +0800" itemprop="datePublished" class="post-meta__date date">2015-01-10</time> &#8226; <span class="post-meta__tags tags">iOS</span>
    </div>
    <h1 class="post-title">iOS中JavaScript和OC交互</h1>
  </header>

  <section class="post">
    <p>在iOS开发中很多时候我们会和UIWebView打交道，目前国内的很多应用都采用了UIWebView的混合编程技术，最常见的是微信公众号的内容页面。前段时间在做微信公众平台相关的开发，发现很多应用场景都是利用HTML5和UIWebView来实现的。</p>

<h3>机制</h3>

<p>Objective-C语言调用JavaScript语言，是通过UIWebView的
<code>- (NSString *)stringByEvaluatingJavaScriptFromString:(NSString *)script;</code>的方法来实现的。该方法向UIWebView传递一段需要执行的JavaScript代码最后获取执行结果。</p>

<p>JavaScript语言调用Objective-C语言，并没有现成的API，但是有些方法可以达到相应的效果。具体是利用UIWebView的特性：在UIWebView的内发起的所有网络请求，都可以通过delegate函数得到通知。  </p>

<h3>示例</h3>

<p>下面提供一个简单的例子介绍如何相互的调用，实现的效果是在界面上点击一个链接，然后弹出一个对话框判断是否登录成功。</p>

<p><img src="/images/uiwebview_js/uiwebview_js_demo.png" alt="uiwebview_js_demo.png"></p>

<p>（1）示例的HTML的源码如下：</p>
<figure class="highlight"><pre><code class="language-text" data-lang="text">&lt;html&gt;
    &lt;head&gt;
        &lt;meta http-equiv=&quot;content-type&quot; content=&quot;text/html;charset=utf-8&quot; /&gt;
        &lt;meta http-equiv=&quot;X-UA-Compatible&quot; content=&quot;IE=Edge&quot; /&gt;
        &lt;meta content=&quot;always&quot; name=&quot;referrer&quot; /&gt;
        &lt;title&gt;测试网页&lt;/title&gt;
    &lt;/head&gt;
    &lt;body&gt;
        &lt;br /&gt;
        &lt;a href=&quot;devzeng://login?name=zengjing&amp;password=123456&quot;&gt;点击链接&lt;/a&gt;
    &lt;/body&gt;
&lt;/html&gt;
</code></pre></figure>
<p>（2）UIWebView Delegate回调方法为：</p>
<figure class="highlight"><pre><code class="language-text" data-lang="text">- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType
{
    NSURL *url = [request URL];
    if([[url scheme] isEqualToString:@&quot;devzeng&quot;]) {
        //处理JavaScript和Objective-C交互
        if([[url host] isEqualToString:@&quot;login&quot;])
        {
            //获取URL上面的参数
            NSDictionary *params = [self getParams:[url query]];
            BOOL status = [self login:[params objectForKey:@&quot;name&quot;] password:[params objectForKey:@&quot;password&quot;]];
            if(status)
            {
                //调用JS回调
                [webView stringByEvaluatingJavaScriptFromString:@&quot;alert(&#39;登录成功!&#39;)&quot;];
            }
            else
            {
                [webView stringByEvaluatingJavaScriptFromString:@&quot;alert(&#39;登录失败!&#39;)&quot;];
            }
        }
        return NO;
    }
    return YES;
}
</code></pre></figure>
<p>说明：</p>

<p>1、同步和异步的问题</p>

<p>（1）Objective-C调用JavaScript代码的时候是同步的</p>

<p><code>- (NSString *)stringByEvaluatingJavaScriptFromString:(NSString *)script;</code></p>

<p>（2）JavaScript调用Objective-C代码的时候是异步的</p>

<p><code>- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType;</code></p>

<p>2、常见的JS调用</p>

<p>（1）获取页面title</p>

<p><code>NSString *title = [webview stringByEvaluatingJavaScriptFromString:@&quot;document.title&quot;];</code></p>

<p>（2）获取当前的URL</p>

<p><code>NSString *url = [webview stringByEvaluatingJavaScriptFromString:@&quot;document.location.href&quot;];</code></p>

<p>3、使用第三方库</p>

<p><code>https://github.com/marcuswestin/WebViewJavascriptBridge</code></p>

<h3>使用案例</h3>

<p>1、动态将网页上的图片全部缩放</p>

<p>JavaScript脚本如下：</p>
<figure class="highlight"><pre><code class="language-text" data-lang="text">function ResizeImages() {
    var myImg, oldWidth;
    var maxWidth = 320;
    for(i = 0; i &lt; document.images.length; i++) {
        myImg = document.images[i];
        if(myImg.width &gt; maxWidth) {
            oldWidth = myImg.width;
            myImg.width = maxWidth;
            myImg.heith = myImg.height*(maxWidth/oldWidth);
        }
    }
}
</code></pre></figure>
<p>在iOS代码中添加如下代码：</p>
<figure class="highlight"><pre><code class="language-text" data-lang="text">[webView stringByEvaluatingJavaScriptFromString:  
 @&quot;var script = document.createElement(&#39;script&#39;);&quot;   
 &quot;script.type = &#39;text/javascript&#39;;&quot;   
 &quot;script.text = \&quot;function ResizeImages() { &quot;   
     &quot;var myimg,oldwidth;&quot;  
     &quot;var maxwidth=380;&quot; //缩放系数   
     &quot;for(i=0;i &lt;document.images.length;i++){&quot;   
         &quot;myimg = document.images[i];&quot;  
         &quot;if(myimg.width &gt; maxwidth){&quot;   
             &quot;oldwidth = myimg.width;&quot;   
             &quot;myimg.width = maxwidth;&quot;   
             &quot;myimg.height = myimg.height * (maxwidth/oldwidth);&quot;   
         &quot;}&quot;   
     &quot;}&quot;
 &quot;}\&quot;;&quot;   
 &quot;document.getElementsByTagName(&#39;head&#39;)[0].appendChild(script);&quot;];
[webView stringByEvaluatingJavaScriptFromString:@&quot;ResizeImages();&quot;];
</code></pre></figure>
<h3>参考资料</h3>

<p>1、<a href="http://blog.devtang.com/blog/2012/03/24/talk-about-uiwebview-and-phonegap/">《关于UIWebView和PhoneGap的总结》</a></p>

<p>2、<a href="http://www.uml.org.cn/mobiledev/201108181.asp">《iOS开发之Objective-C与JavaScript的交互 》</a></p>

  </section>
</article>

<section class="read-more">


   <div class="read-more-item">
       <span class="read-more-item-dim">最近的文章</span>
       <h2 class="post-list__post-title post-title"><a href="/blog/ios-html-template-engine.html" title="link to iOS中使用模板引擎渲染HTML界面">iOS中使用模板引擎渲染HTML界面</a></h2>
       <p class="excerpt">在iOS实际的开发中，使用UIWebView来加载数据使用的场景特别多。很多时候我们会动态的从服务器获取一段HTML的内容，然后App这边动态的处理这段HTML内容用于展示在UIWebView上。使用到的API接口为：- (void)loadHTMLString:(NSString *)string baseURL:(NSURL *)baseURL;由于HTML内容通常是变化的，所以我们需要动态生成HTML代码。通常我们从服务器端获取到标题、时间、作者和对应的内容，然后我们需要对这些数据处...&hellip;</p>
       <div class="post-list__meta"><time datetime="2015-01-17 21:22:04 +0800" class="post-list__meta--date date">2015-01-17</time> &#8226; <span class="post-list__meta--tags tags">iOS</span><a class="btn-border-small" href=/blog/ios-html-template-engine.html>继续阅读</a></div>
   </div>




   <div class="read-more-item">
       <span class="read-more-item-dim">更早的文章</span>
       <h2 class="post-list__post-title post-title"><a href="/blog/ios8-touch-id.html" title="link to iOS8中使用TouchID校验用户身份">iOS8中使用TouchID校验用户身份</a></h2>
       <p class="excerpt">在iOS8中，开发者们可使用向第三方应用开放了Touch ID权限的API，以便他们在应用中使用指纹认证来完成用户认证部分。相当一部分的APP（如印象笔记、新版QQ）以及在升级后采用了Touch ID来验证用户身份，用以替代过去使用一般密码或者PIN码，如下图所示：（1）新版QQ：（2）印象笔记高级版本：本文主要介绍如何在应用中集成Touch ID来校验用户的身份。集成步骤1、环境要求（1）开发环境：Xcode 6（iOS8 SDK）（2）设备要求：iPhone 5s、iPhone 6 (...&hellip;</p>
       <div class="post-list__meta"><time datetime="2014-12-07 13:09:47 +0800" class="post-list__meta--date date">2014-12-07</time> &#8226; <span class="post-list__meta--tags tags">iOS</span><a class="btn-border-small" href=/blog/ios8-touch-id.html>继续阅读</a></div>
   </div>

</section>

<section class="post-comments">




</section>

</div>
            </div>
        </div>
    </body>
</html>
```

#### 2、渲染生成 `HTML` 字符串

```
- (NSString *)loadHTMLByMGTemplateEngine:(NSDictionary *)data
{
    NSString *templatePath = [[NSBundle mainBundle] pathForResource:@"template" ofType:@"html"];
    MGTemplateEngine *engine = [MGTemplateEngine templateEngine];
    [engine setMatcher:[ICUTemplateMatcher matcherWithTemplateEngine:engine]];
    [engine setObject:data[@"title"] forKey:@"title"];
    [engine setObject:data[@"author"] forKey:@"author"];
    [engine setObject:data[@"date"] forKey:@"date"];
    [engine setObject:data[@"content"] forKey:@"content"];
    return [engine processTemplateInFileAtPath:templatePath withVariables:nil];
}
```

#### 3、说明

（1）`MGTemplateEngine` 提供的示例程序是运行在 `Mac OS` 上的，如果要使用到 `iOS` 上面需要引入 `Foundation` 框架

（2）对于运行在 `Xcode6` 以上的环境下创建的工程由于没有 `PCH` 文件可能会报错，需要在 `MGTemplateEngine` 的各个头文件中引入 `Foundation` 框架

（3）`MGTemplateEngine` 在 `GitHub` 上的地址为 `https://github.com/mattgemmell/MGTemplateEngine` 。

### `GRMustache` 的使用

#### 相比MGTemplateEngine来说GRMustache简单不少，

#### 1、处理模板文件

模板文件和 `MGTemplateEngine` 的一样。

#### 2、渲染生成 `HTML` 字符串

```
- (NSString *)loadHTMLByGRMustache:(NSDictionary *)data
{
    NSString *templatePath = [[NSBundle mainBundle] pathForResource:@"template" ofType:@"html"];
    NSString *template = [NSString stringWithContentsOfFile:templatePath encoding:NSUTF8StringEncoding error:nil];
    return [GRMustacheTemplate renderObject:data fromString:template error:nil];
}
```

#### 3、说明

（1）`renderObject` 使用的数据的 `key` 必须要和模板中的占位符一一对应起来

（2）`GRMustache` 在 `GitHub` 上的地址为 `https://github.com/groue/GRMustache`

### 参考资料

1、[《MGTemplateEngine - Templates with Cocoa》](http://mattgemmell.com/mgtemplateengine-templates-with-cocoa/)

2、[《MGTemplateEngine 模版引擎简单使用》](http://blog.csdn.net/crazy_srufboy/article/details/21748995)

3、[《GRMustache Document》](http://mustache.github.io/mustache.5.html)
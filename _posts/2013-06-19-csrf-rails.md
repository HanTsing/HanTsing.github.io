---
layout: post
title: "CSRF Rails"
description: ""
category: rails
tags: [rails, csrf]
---

{% include JB/setup %}

###CSRF攻击以及Rails防范措施

首先来了解一下什么是CSRF，CSRF是伪造客户端请求的一种攻击，CSRF的英文全称是Cross Site Request Forgery，字面上的意思是跨站点伪造请求。


**CSRF攻击是源于WEB的隐式身份验证机制！WEB的身份验证机制虽然可以保证一个请求是来自于某个用户的浏览器，但却无法保证该请求是用户批准发送的！**

 要理解这句话，首先要了解一下浏览器共享cookie的原理。 同窗口浏览器因为是同一个进程, 所以cookie是共享的，多个Tab访问的同一个网站，每次请求访问这个网站，都会带上此网站的cookie。
 
 我们理解了这个原理之后，就可以通过下面的列子来演示一个CSRF攻击。
 首先用户在访问A网站，这时会生产这个用户在A网站的cookie，假如用户重新打开一个Tab访问B网站，点击了一个恶意链接，而这个恶意链接是访问A网站的请求，那么此时这个请求就会带上A网站的cookie信息，这个恶意请求就会以用户的名义来进行请求，形成了一次CSRF恶意攻击。
 
 下面的图片演示了这次攻击流程
 ![Alt text](http://images.cnblogs.com/cnblogs_com/anic/csrf3.PNG)


 ####接下来看看rails是如何防止CSRF攻击的

Rails內建了CSRF防禦功能，也就是所有的POST請求，都必須加上一個安全驗證碼。在app/controllers/application_controller.rb你會看到以下程式啟用這個功能：
<pre><code>
class ApplicationController &lt; ActionController::Base
  protect_from_forgery
end
</code></pre>

<pre><code>
#lib/action_view/helpers/form_tag_helper.rb
  def token_tag(token)
    if token == false || !protect_against_forgery?
      ''
    else
      token ||= form_authenticity_token
      tag(:input, :type => "hidden", :name => request_forgery_protection_token.to_s, :value => token)
    end
  end
</code></pre>


form_authenticity_token 这个方法会往session里面加入csrf_token信息。

<pre><code>
#actionpack/lib/action_controller/metal/request_forgery_protection.rb
  def form_authenticity_token
    session[:_csrf_token] ||= SecureRandom.base64(32)
  end
</code></pre>


这样提交表单时就会验证CSRF ApplicationController 中的 protect_from_forgery方法就会验证csrf_token的有效性。

<pre><code>
#actionpack/lib/action_controller/metal/request_forgery_protection.rb
  def protect_from_forgery(options = {})
    self.forgery_protection_strategy = protection_method_class(options[:with] || :null_session)
    self.request_forgery_protection_token ||= :authenticity_token
    prepend_before_action :verify_authenticity_token, options
  end
</code></pre>



然后调用了verify_authenticity_token方法 最终通过这个verified_request? 来验证crsf_token

<pre><code>
 # The actual before_action that is used. Modify this to change how you handle unverified requests.
  def verify_authenticity_token
    unless verified_request?
      logger.warn "Can't verify CSRF token authenticity" if logger
       handle_unverified_request
    end
  end
</code></pre>


<pre><code>
  def verified_request?
    !protect_against_forgery? || request.get? || request.head? ||
    form_authenticity_token == params[request_forgery_protection_token] ||
    form_authenticity_token == request.headers['X-CSRF-Token']
  end
</code></pre>

以下几种情况是合法的请求会验证通过

1, 跳过不验证的
2, GET HEAD 请求
3, csrf_token 和参数中 authenticity_token 值相同的
4, http header 中 X-CSRF-Token 和 csrf_token 的值相同的

如果不是合法请求 Rails就会抛出一个ActionController:InvalidAuthenticityToken的错误。并且会重设session

<pre><code>
      def handle_unverified_request
        reset_session
      end
</code></pre>

Rails Session CookieStore 原理
 
在rails后端调试下 session, 打印出来的结果是一个hash, 以hui800 为例, 先反向得到 session 数据, 用firebug可以看到
hui800的cookie中有一个 _hui800_session, 如下:
<pre><code>
BAh7B0kiD3Nlc3Npb25faWQGOgZFRkkiJTM5MzhjMDFlYTA1NjljOGIwZWJjODhkMjZmYWM0ODQ4BjsAVEkiEF9jc3JmX3Rva2VuBjsARkkiMXJ5OEVHb2pvMnhWZmdFaGpHeTNhRk5JQkZwRE5oOFpXekY5Ymo5WVNkVDg9BjsARg%3D%3D--7805882eaed11466c84bd490e829a1e2516013e9
</code></pre>

我们可以通过 Marshal.load Base64.decode64(data) 后会得到一个hash, 这个就是后端的 session数据
<pre><code>
pry(main)> Marshal.load Base64.decode64(s)
 {"session_id"=>"3938c01ea0569c8b0ebc88d26fac4848",
 "_csrf_token"=>"ry8EGojo2xVfgEhjGy3aFNIBFpDNh8ZWzF9bj9YSdT8="}
</code></pre>
这个'_csrf_token'就是rails生成的session值，因为rails的session cookiestore所以是存在浏览器的cookie中的

Layout中也有一段<%= csrf_meta_tags %>是給JavaScript讀取驗證碼用的。
但页面如果有缓存的话，我的做法会给cookie设置一个csrf token，ajax请求来读取cookie的值，然后加到请求参数中。

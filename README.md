# RuCTF 2014 Quals - Web400 writeup

> Hi! We have unconfirmed information about our agent and developer of our site. His tasks includes processing requests from civilians. We think that he is double agent and works for irRational Security Agency. Help us to find evidence about that.  
> Auth: team:pass

Ok, what interesting information do we have about the task?

> His [agent's] tasks includes processing requests from civilians.

*Client-side attack? XSS?*

> Cookie named "ssid"

*Some kind of sessions?*

Let's try to fill the form fields with `<script>alert('xss')</script>` and make request.

![](https://raw.github.com/dscheg/ructf-2014-quals-web400-writeup/master/img/1.png)

![](https://raw.github.com/dscheg/ructf-2014-quals-web400-writeup/master/img/2.png)

Nothing... View page source.

![](https://raw.github.com/dscheg/ructf-2014-quals-web400-writeup/master/img/3.png)

We have something interesting here - hidden information for agents about request: IP address and User-Agent.
Maybe it isn't encoded? Try to inject script into User-Agent (with Firefox we can use 'Modify Headers' extension).

![](https://raw.github.com/dscheg/ructf-2014-quals-web400-writeup/master/img/4.png)

Submit the form.

![](https://raw.github.com/dscheg/ructf-2014-quals-web400-writeup/master/img/5.png)

Voila! XSS works. But we have some limitations here:

1. HTTPOnly cookies so we cannot steal session id.
2. Content-Security-Policy header with "default-src 'self'" which disallow all attemps to open connections to other domains.

Yet another limitation - hint for the task:

> It seems that agent and system developer have disabled all outbound connections from agent's computer to the Internet ;-)

What can we do with this? CSRF can bypass these limitations, but we have no target to it.
What else doesn't need outbound connection? Session Fixation! Let's try it.
But, but, but... HTTPOnly cookies. We cannot overwrite agent's ssid by our own! Or... we can?
After some investigation we can reveal that here is a possibility to set cookies to non-root path, '/msg' for example.

Let's check that it works. Go to page /msg (or submit the form). Run this simple script in browser console.

`document.cookie='ssid=wtf';path=/msg'`

![](https://raw.github.com/dscheg/ructf-2014-quals-web400-writeup/master/img/6.png)

Check our cookies.

![](https://raw.github.com/dscheg/ructf-2014-quals-web400-writeup/master/img/7.png)

Now we have 2 cookies with name **ssid** set. F5 the page. And...

![](https://raw.github.com/dscheg/ructf-2014-quals-web400-writeup/master/img/8.png)

Server returns Set-Cookie header in response with new ssid! This means that our fixated cookie is more significant.

Now all that we need to do - create payload which set cookies to all known paths. Here it is:

`--><script>function a(p){document.cookie='ssid=3c87db1f7acc4a2a8758cf49dd74617965c393927f7753e78232f81b875e7e70;path=/'+p};a('msg');a('auth');a('login');</script>`

Submit the form. After several seconds reload the page, and we got it - secret page!

![](https://raw.github.com/dscheg/ructf-2014-quals-web400-writeup/master/img/9.png)

**P.S.** Task can be solved with retrieving information through DNS requests to attacker's DNS server (agent is bad admin and outgoing connections on 53 port works). See writeup here: http://ahack.ru/write-ups/ructf-quals-14.htm
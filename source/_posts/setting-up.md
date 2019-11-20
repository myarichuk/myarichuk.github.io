title: 'Setting up the blog, a meta-post!'
tags:
  - Hexo.io
  - Blog
categories:
  - Meta
author: Michael Yarichuk
date: 2019-11-18 08:28:00
top_img: hexo-logo.jpeg
cover: /2019/11/18/setting-up/hexo-logo.png
---

After taking a look at a couple of more "mainstream" blogging systems, I was looking for a way to do some blogging and not deal with over-engineered systems that are bloated with unnecessary features. I didn't want to spend time in understanding the details required to actually tweak those systems and customize them to my liking.
And then I found static website generators like Jekyll and Hexo. After choosing Hexo because it used a more familiar toolset, I found out that I actually understood how it worked without investing too much time. And with my (VERY!) limited knowledge of web programming I could tweak it and make my blog to look and feel the way I wanted it. 
"Great..." I hear you say "... now how do I do that?"

### Setting up
The idea behind [Hexo.io](https://hexo.io/) is simple. The Hexo system compiles with Node.js a static website from a template and it's pages from markdown files.
Setting up a minimal blog is easy. After [installing Node.js](https://nodejs.org/en/download/) on your system, we simply need to execute

``` bash
$ npm install hexo-cli -g
$ hexo init blog
```

That's it! The last command will create a folder named 'blog' with necessary files to generate your website (blog?)
The next step is to actually check that it works. Simply, execute the following.

``` bash
$ hexo generate
$ hexo server
```

Now you have a local webserver running that will host your blog and you can test it. (by default it would be http://localhost:4000)

### Customizing it
Setting up is nice, but nobody really wants a website with default look and feel. So, now what?
{% asset_img fun-begins.jpg %}

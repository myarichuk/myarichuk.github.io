---
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

First step would be to design the look and feel of the website, which can be done in two ways.
- If you have some knowledge in web development, you can have a go at creating your own theme for Hexo. This would probably require some web design skills with CSS and knowledge of one of the supported Javascript template engines (such as [EJS](https://github.com/hexojs/hexo-renderer-ejs), [Haml](https://github.com/hexojs/hexo-renderer-haml), [Jade](https://github.com/hexojs/hexo-renderer-jade), or [Pug](https://github.com/maxknee/hexo-render-pug). A good starting place would be [in this article](http://www.codeblocq.com/2016/03/Create-an-Hexo-Theme-Part-1-Index/) and [Hexo's documentation article](https://hexo.io/docs/themes.html) can be useful as well.
- If you are like me and don't know that much about web development, you can simply get one of [existing themes](https://hexo.io/themes/) and tweak it to your liking. At the very minimum, it would involve fiddling with *_config.yml* file inside the theme's folder, but tweaking around page templates and CSS is also possible, of course.

For example, in case of this blog, I took an [existing theme - "Butterfly"](https://github.com/jerryc127/hexo-theme-butterfly) and tweaked it a little.

### Creating content
Finally, when the tweaking is complete it is time to actually write something, this is why why wanted a blog in the first place, no? </br>
Running the following console command will create an empty post with specified title - which is simply a markdown file at **[Hexo root]/source/_posts/[post title].md**


Console command to create a new post:
``` bash
$ hexo new post <title>
```
You can read more about writing in Hexo in [relevant documentation article](https://hexo.io/docs/writing.html).

### Publishing it
After finishing writing a post or two, it is time to publish the website.
We would start by executing the following command in the console:
``` bash
$ hexo generate
```
This would "compile" markdown and theme's templates into a static website, which then can be deployed to a hosting service. 
<br/>
_Note: the destination of such "compilation" would be **[Hexo root]/public folder**_
<br/>


Now, after we generated our static website, it is time to deploy it. Before doing that, we would need to configure Hexo's deploy command in the **_config.yml** file, a command that can be used to deploy compiled website to one or more destinations.
For example, the following will enable deployment to a git repository
``` yml
deploy:
  type: git
  repo: [repository url]
  branch: [branch name]
```

You can read more about the deploy command [in it's documentation article](https://hexo.io/docs/one-command-deployment).

So, that's it. Have fun blogging! <br/>
Actually, I think this blog post is a bit longer than it could (or should!) be, but I wanted to be thorough, especially since this is my first blog post :)

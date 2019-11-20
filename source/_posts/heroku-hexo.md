---
title: Hosting Hexo.io in Heroku
date: 2019-11-21 01:16:43
tags:
  - Hexo.io
  - Blog
categories:
  - Meta
author: Michael Yarichuk
top_img: heroku-logo.png
cover: /2019/11/21/heroku-hexo/heroku-logo.png
---

When trying to set up my blog to be hosted in Heroku, I set up it so I can push int Github repo, then Heroku will pull the code and deploy it. Locally it seemed to work fine with Hexo's server, so I was a bit surprised when my blog failed. Heroku logs have shown the following:
```
2019-11-20T22:39:25.000000+00:00 app[api]: Build succeeded
2019-11-20T22:39:27.173507+00:00 heroku[web.1]: Stopping all processes with SIGTERM
2019-11-20T22:39:27.241329+00:00 heroku[web.1]: Process exited with status 143
2019-11-20T22:39:29.990618+00:00 heroku[web.1]: Starting process with command `npm start`
2019-11-20T22:39:33.485877+00:00 heroku[web.1]: State changed from starting to crashed
2019-11-20T22:39:33.556555+00:00 heroku[web.1]: State changed from crashed to starting
2019-11-20T22:39:33.470506+00:00 heroku[web.1]: Process exited with status 1
2019-11-20T22:39:33.322139+00:00 app[web.1]: npm ERR! missing script: start
```

Apparently, Heroku tries to execute **start** script on Hexo website, so adding the following script to **package.json** solved this issue.
``` json
"scripts": {
	"start": "hexo server -p $PORT"
	// the rest of the scripts
}
```
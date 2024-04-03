---
title: "Heroku docker FastAPI"
date:  2020-11-03T12:00:00+00:00
tags:
  - heroku
  - fastapi
  - docker
---

1. Make sure you have the Heroku CLI and GIT

2. See this [repo](https://github.com/askblaker/fastapi-docker-heroku) or the spoiler below:

```
git clone git@github.com:askblaker/fastapi-docker-heroku.git
cd fastapi-docker-heroku
heroku create <your-app-name>
heroku git:remote <your-app-name>
heroku stack:set container
git push heroku main
```

2. Enjoy your api at https://your-app-name.herokuapp.com

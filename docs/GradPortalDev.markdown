---
layout: post
title:  "Development for IU Graduate portal"
date:   2024-03-04 14:27:49 +0300
---

# Development for IU Graduate portal

Here we cover how to get started with developing for the portal.

###  Project components

As any web app, the portal has frontend and backed components. However it additionally has a telegram bot that is ran alongside it.

### Technological stack

- Python FastAPI
- Next.js
- Docker
- PostgreSQL
- Prisma ORM
- Telebot
- Lets Encrypt

### Disclaimer

*Currently the project may be in an unstable state -- it is not ready for usage.*

*You may want to wait until the issues have been resolved, [track the issue here][bad_issue].*

### How to run

Here is how to start developing IU graduate portal:
* Clone repository
* Create virtual env `python -m venv .`
* Activate environment [using an appropriate method][pythonvenv]
* Install requirements `pip install -r requirements.txt.`
* Make an .env file {% highlight bash %}
        DATABASE_HOST=
        DATABASE_PORT=
        DATABASE_PASSWORD=
        DATABASE_NAME=
        DATABASE_USERNAME=
        SECRET_KEY=
        ALGORITHM=
        ACCESS_TOKEN_EXPIRE_MINUTES=
        MAIL_USERNAME=
        MAIL_PASSWORD=
        MAIL_FROM=
        MAIL_PORT=
        MAIL_SERVER=
        MAIL_FROM_NAME={% endhighlight %}
* Put [telegram bot token][tgbottoken] in `/app/telegram/data.py`
* Change telegram admin to your id in `/app/telegram/admin/data.py`
* Put url of your database into `/app/prisma/schema.prisma`
* Run `prisma generate`, `prisma db push`
* From root directory run `uvicorn app.main:app`
* From frontend directory run `npm install`, `dev npm start`

Keep in mind that current active branch is `moving-to-fastapi`!

### API Documentation

As the portal is moving towards FastAPI, you can see API documentation by following the steps above and visiting `http://127.0.0.1:8000/docs`.

You can track the process of moving to the FastAPI in [this pull request][fastapipr].

### Deployment

We do not yet provide specific steps for deployment. However there are some Dockerfiles in the repository which may help you with your endeavours.

You can track the progress on the docker deployment [here][bad_issue].

### Contribution

Feel free to contribute to this repository by submitting a Pull Request or an Issue.



[portal]: https://github.com/TheSharpOwl/inno-alumni-portal
[bad_issue]: https://github.com/TheSharpOwl/inno-alumni-portal/issues/36
[live_demo]: https://graduates.innopolis.university/
[demo_yt]: https://www.youtube.com/watch?v=PwiZH98iqJ8
[pythonvenv]: https://docs.python.org/3/tutorial/venv.html#creating-virtual-environments
[tgbottoken]: https://core.telegram.org/bots#how-do-i-create-a-bot
[fastapipr]: https://github.com/TheSharpOwl/inno-alumni-portal/pull/35

<!--{% highlight bash %}
bash be bash
{% endhighlight %}-->





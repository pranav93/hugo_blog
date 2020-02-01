---
title: "Pushing Error Traceback On Slack Channel In Python"
date: 2016-11-13T19:01:09+05:30
draft: false
tags: ["python", "traceback"]
---

Wouldn't it be awesome if we could get the error trace on slack. For this first of all we'll have to write a handler that will compose the message the way slack likes.

```python
import logging
import slackweb


class SlackHandler(logging.Handler):

    def __init__(self, url, channel, username, icon_emoji):
        logging.Handler.__init__(self)
        self.slack_obj = slackweb.Slack(url=url)
        self.channel = channel
        self.username = username
        self.icon_emoji = icon_emoji

    def emit(self, record):
        self.slack_obj.notify(
            text=record.exc_text or record.msg,
            channel="#{}".format(self.channel),
            username=self.username,
            icon_emoji=self.icon_emoji
        )

# Usage: Add handler as,
# 'slack_jabber': {
#     'class': 'SlackHandler',
#     'url': 'https://hooks.slack.com/services/XYZ',
#     'channel': 'custom',
#     'username': 'log_jabber',
#     'icon_emoji': ':ghost:'
# },
#  and use it in logger
```

And we’ll configure the logger in config file,

```python
'slacker': {
  'class': 'utils.slacker.SlackHandler',
  'url': 'https://hooks.slack.com/services/xyz',
  'channel': 'pranavdev',
  'username': 'crash_jabber',
  'icon_emoji': ':ghost:'
},
```

As, you can see hook is needed so that our messages are passed to a channel in slack. You can find information about how to create an incoming wehook here https://api.slack.com/

At last, call your logger when exception occurs as,

```python
slacker.exception(exc)
```

And you will get the stack trace on your channel.

The final project is at https://github.com/pranav93/slack_trace

hi bill, here's some code for a feature i'm finishing today

its a bundled 'recent activity' email notification system for [Cobudget](http://cobudget.co/) [Beta](http://beta.cobudget.co/)

### context:

our designer, derek, has been thinking about how cobudget's email notification system should be like. [he came up with this](https://docs.google.com/document/d/15N5UqHo649pqzBoNN5r1hTubbtTRlCLsfH_RyfHIDDs/edit?usp=sharing)

in this system, users choose which events they wanna hear about, and how often they wanna hear about it.

here's a GIF of the UI

![GIF of the UI](http://g.recordit.co/W0nB035S3Y.gif)

the UI design is still in progress, but the server-side code is done -- and that's what i'd like to show you.

### code sample

you can [view the pull request here](https://github.com/cobudget/cobudget-api/pull/129) if you like, but in case you don't want to sift through all the noise, i've prepared guided walkthroughs to explain this feature's two main server-side functions:

  1. **[walkthrough: how customized email notifications are delivered to cobudget users](./cobudget-rake.md)**, and

  2. **[walkthrough: how users update their email notification settings](./subscription-trackers-controller.md)**

i've picked this code sample, because it's fresh in my head, and because it's been a particularly interesting problem to solve. it's involved timezones, scheduled jobs, delayed jobs, event collection, subscriptions, efficient database queries, email notifications, and tests at the model-level, service-level, and controller-level.

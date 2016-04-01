hi bill, here's some code for a feature i'm finishing today

its a bundled 'recent activity' email notification system for [Cobudget](http://cobudget.co/) [Beta](http://beta.cobudget.co/)

**edit [03-31-16]**: though all tests were passing in our test environment, when i pushed up to our staging server, emails weren't being sent. silly mistake of mine -- our test environment is configured to deliver emails synchronously, but our production environment is configured to enqueue requested email deliveries to be delivered asynchronously. and it turns out that enqueued method calls cannot accept ActiveRecord objects as parameters. i've updated the code to accommodate for this. i've also done some rewording and added a brief technical overview to make the walkthroughs easier to walk through.


### context:

our designer, derek, has recently redesigned cobudget's email notification system. [he came up with this](https://docs.google.com/document/d/15N5UqHo649pqzBoNN5r1hTubbtTRlCLsfH_RyfHIDDs/edit?usp=sharing).

the goal here is to notify users of group activity they care about without flooding their email.

in this system, users choose which events they wanna hear about, and how often they wanna hear about it. they receive these notifications via email -- as 'bundles' of recent activity

here's a GIF of the UI

![GIF of the UI](http://g.recordit.co/W0nB035S3Y.gif)

the UI design is still in progress, but the server-side code is done -- and that's what i'd like to show you.

### code sample

you can [view the pull request here](https://github.com/cobudget/cobudget-api/pull/129) if you like, but in case you don't want to sift through all the noise, i've prepared guided walkthroughs to explain this feature's two main server-side functions:

  1. **[walkthrough: how customized email notifications are delivered to cobudget users](./cobudget-rake.md)**, and

  2. **[walkthrough: how users update their email notification settings](./subscription-trackers-controller.md)**

you may find it helpful to have this **[brief technical overview of cobudget](./brief-technical-overview.md)** open in a separate window while reading through the walkthroughs.

**note**: most of these walkthrough files will contain the subheadings *quick reference* and *relevant tests*. under the *quick reference* subheadings, you'll find snippets of files that the main piece of code references (in case you're curious). *relevant tests* is self-explanatory :)

i've picked this code sample, because it's fresh in my head, and because it's been a particularly interesting problem to solve. it's involved timezones, scheduled jobs, delayed jobs, event collection, subscriptions, efficient database queries, email notifications, and tests at the model-level, service-level, and controller-level.

*walkthrough: how customized email notifications are delivered to cobudget users - part 1 / 4*

### `rake cobudget:deliver_recent_activity_digest`

this is where everything starts. this rake task will be set to run every 10 minutes on our heroku server. when it runs, it sends customized notification emails to every user on cobudget who is due for delivery.

```ruby
namespace :cobudget do
  desc "delivers recent activity digest emails to users"
  task deliver_recent_activity_digest: :environment do
    DeliverRecentActivityDigest.run!
  end
end
```

**[GOTO `DeliverRecentActivityDigest#run!`](./deliver-recent-activity-digest.md)**

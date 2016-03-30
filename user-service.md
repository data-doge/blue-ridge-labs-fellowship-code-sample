*walkthrough: how customized email notifications are delivered to cobudget users - part 3 / 5*

### `UserService#send_recent_activity_email`

uses an instance of `RecentActivityService` to (1) check if a user has `recent_activity`, and (2) if they do, send an email to them

```rb
class UserService
  def self.send_recent_activity_email(user: )
    recent_activity = RecentActivityService.new(user: user)
    if recent_activity.is_present?
      UserMailer.recent_activity(user: user, recent_activity: recent_activity).deliver_later
    end
  end

  ...
end
```

**[OK sweet, how does `RecentActivityService` work?](./recent-activity-service.md)**

**and also, [what is `UserMailer#recent_activity` doing?](./user-mailer.md)**

---

### relevant tests

#### `UserService`

![meow](http://i.imgur.com/AwdMkLX.png)

```rb
require 'rails_helper'

describe "UserService" do
  after { ActionMailer::Base.deliveries.clear }

  ...

  describe "#send_recent_activity_email(user:)" do
    let!(:current_time) { DateTime.now.utc }
    let!(:user) { create(:user) }
    let!(:group) { create(:group) }
    let!(:membership) { create(:membership, member: user, group: group) }
    # notification_frequency set to 'hourly' by default
    let!(:subscription_tracker) { user.subscription_tracker }

    before do
      subscription_tracker.update(recent_activity_last_fetched_at: current_time - 1.hour)
    end

    context "recent activity exists" do
      it "sends email to user" do
        Timecop.freeze(current_time - 30.minutes) do
          create(:bucket, status: "draft", group: group, target: 420)
        end

        Timecop.return
        UserService.send_recent_activity_email(user: user)
        expect(ActionMailer::Base.deliveries.length).to eq(1)
      end
    end

    context "recent activity doesn't exist" do
      it "does not send email to user" do
        UserService.send_recent_activity_email(user: user)
        expect(ActionMailer::Base.deliveries.length).to eq(0)
      end
    end
  end
end
```

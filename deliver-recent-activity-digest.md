*walkthrough: how customized email notifications are delivered to cobudget users - part 2 / 4*

### `DeliverRecentActivityDigest#run!`

when called, goes through every user with active memberships, and if they are due for a delivery, it (1) asks `UserMailer` to send them a recent activity email, and (2) schedules the next delivery

```rb
class DeliverRecentActivityDigest
  def self.run!
    current_time = DateTime.now.utc
    User.with_active_memberships.each do |user|
      if user.subscription_tracker.ready_to_fetch?(current_time: current_time)
        UserMailer.recent_activity(user: user).deliver_later
        user.subscription_tracker.update_next_fetch_time_range!
      end
    end
  end
end
```

**[GOTO `UserMailer#recent_activity`](./user-mailer)**

---

#### quick reference

##### `User`

```rb
require 'securerandom'

class User < ActiveRecord::Base
  ...

  after_create :create_default_subscription_tracker

  ...

  has_one :subscription_tracker,                   dependent: :destroy

  ...

  scope :with_active_memberships, -> { joins(:memberships).where(memberships: {archived_at: nil}) }

  ...

  private
    ...

    def create_default_subscription_tracker
      SubscriptionTracker.create(
        user: self,
        recent_activity_last_fetched_at: DateTime.now.utc.beginning_of_hour
      )
    end
end
```

##### `SubscriptionTracker`

```rb
class SubscriptionTracker < ActiveRecord::Base
  belongs_to :user

  ...

  def next_recent_activity_fetch_scheduled_at
    interval = case notification_frequency
      when "hourly" then 1.hour
      when "daily" then 1.day
      when "weekly" then 1.week
    end
    recent_activity_last_fetched_at + interval if interval
  end

  def ready_to_fetch?(current_time: )
    notification_frequency != "never" && current_time >= next_recent_activity_fetch_scheduled_at
  end

  ...

  def update_next_fetch_time_range!
    update(recent_activity_last_fetched_at: next_recent_activity_fetch_scheduled_at)
  end

  ...
end
```


---

#### relevant tests

##### `DeliverRecentActivityDigest`

![meow](http://i.imgur.com/nKisJq1.png)

```rb
require 'rails_helper'

describe "DeliverRecentActivityDigest" do
  let!(:current_time) { DateTime.now.utc }
  let!(:user) { create(:user) }
  let!(:subscription_tracker) { user.subscription_tracker }

  before do
    subscription_tracker.update(notification_frequency: "hourly")
    subscription_tracker.update(recent_activity_last_fetched_at: current_time - 1.hour)
  end

  after do
    Timecop.return
    ActionMailer::Base.deliveries.clear
  end

  describe "#run!" do
    context "user has no active memberships" do
      it "does not send a recent activity notification email to the user" do
        Timecop.freeze(current_time + 1.minute) do
          DeliverRecentActivityDigest.run!
          expect(ActionMailer::Base.deliveries).to be_empty
        end
      end

      it "does not schedule the next recent activity fetch" do
        Timecop.freeze(current_time + 1.minute) do
          DeliverRecentActivityDigest.run!
          expect(subscription_tracker.reload.recent_activity_last_fetched_at).to eq(current_time - 1.hour)
        end
      end
    end

    context "user has at least one active membership" do
      let!(:group) { create(:group) }
      let!(:membership) { create(:membership, member: user, group: group) }

      context "current_time is before user's next scheduled fetch" do
        it "does not send a recent activity notification email to the user" do
          Timecop.freeze(current_time - 1.minute) do
            DeliverRecentActivityDigest.run!
            expect(ActionMailer::Base.deliveries).to be_empty
          end
        end

        it "does not schedule the next recent activity fetch" do
          Timecop.freeze(current_time - 1.minute) do
            DeliverRecentActivityDigest.run!
            expect(subscription_tracker.reload.recent_activity_last_fetched_at).to eq(current_time - 1.hour)
          end
        end
      end

      context "current_time is after user's next scheduled fetch" do
        context "recent activity exists" do
          before do
            Timecop.freeze(current_time - 1.minute) do
              create(:bucket, group: group, status: "draft", name: "le bucket", user: user)
            end
          end

          it "sends a recent activity notification email to the user" do
            Timecop.freeze(current_time + 1.minute) do
              DeliverRecentActivityDigest.run!
              expect(ActionMailer::Base.deliveries.length).to eq(1)

              sent_email = ActionMailer::Base.deliveries.first

              expect(sent_email.to).to include(user.email)
              expect(sent_email.subject).to include("My recent activity on Cobudget")
              expect(sent_email.body).to include("le bucket")
            end
          end

          it "updates user's subscription_tracker's recent_activity_last_fetched_at timestamp" do
            Timecop.freeze(current_time + 1.minute) do
              DeliverRecentActivityDigest.run!
              expect(subscription_tracker.reload.recent_activity_last_fetched_at).to eq(current_time)
            end
          end
        end

        context "recent activity does not exist" do
          it "does not send a recent activity notification email to the user" do
            Timecop.freeze(current_time + 1.minute) do
              DeliverRecentActivityDigest.run!
              expect(ActionMailer::Base.deliveries).to be_empty
            end
          end

          it "updates user's subscription_tracker's recent_activity_last_fetched_at timestamp" do
            Timecop.freeze(current_time + 1.minute) do
              DeliverRecentActivityDigest.run!
              expect(subscription_tracker.reload.recent_activity_last_fetched_at).to eq(current_time)
            end
          end
        end
      end
    end
  end
end
```

##### `SubscriptionTracker`

![meow](http://i.imgur.com/4o0kjoD.png)

```rb

require 'rails_helper'

RSpec.describe SubscriptionTracker, type: :model do
  ...

  let!(:parisian_user) { create(:user, utc_offset: +60) }

  ...

  describe "#next_recent_activity_fetch_scheduled_at" do
    let(:subscription_tracker) { parisian_user.subscription_tracker }
    let(:six_am_today) { DateTime.now.utc.beginning_of_day + 6.hours }

    def subscription_tracker_with_last_fetch_today_at_six_am(notification_frequency: )
      subscription_tracker.update(notification_frequency: notification_frequency)
      subscription_tracker.update(recent_activity_last_fetched_at: six_am_today)
      subscription_tracker
    end

    context "if notification_frequency is 'never'" do
      it "returns nil" do
        tracker = subscription_tracker_with_last_fetch_today_at_six_am(notification_frequency: "never")
        expect(tracker.next_recent_activity_fetch_scheduled_at).to be_nil
      end
    end

    context "if notification_frequency is 'hourly'" do
      it "returns datetime 1 hour after last fetch" do
        tracker = subscription_tracker_with_last_fetch_today_at_six_am(notification_frequency: "hourly")
        expect(tracker.next_recent_activity_fetch_scheduled_at).to eq(six_am_today + 1.hour)
      end
    end

    context "if notification_frequency is 'daily'" do
      it "returns datetime 1 day after last fetch" do
        tracker = subscription_tracker_with_last_fetch_today_at_six_am(notification_frequency: "daily")
        expect(tracker.next_recent_activity_fetch_scheduled_at).to eq(six_am_today + 1.day)
      end
    end

    context "if notification_frequency is 'weekly'" do
      it "returns datetime 1 week after last fetch" do
        tracker = subscription_tracker_with_last_fetch_today_at_six_am(notification_frequency: "weekly")
        expect(tracker.next_recent_activity_fetch_scheduled_at).to eq(six_am_today + 1.week)
      end
    end
  end

  ...
end
```

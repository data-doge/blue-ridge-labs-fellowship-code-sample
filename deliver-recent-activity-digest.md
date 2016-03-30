*walkthrough: how customized email notifications are delivered to cobudget users - part 2 / 5*

### `DeliverRecentActivityDigest#run!`

when called, goes through every user with active memberships who is due for a delivery, and (1) asks `UserService` to send them customized emails, and (2) schedule their next delivery

```rb
class DeliverRecentActivityDigest
  def self.run!
    current_time = DateTime.now.utc
    User.with_active_memberships.each do |user|
      if user.subscription_tracker.ready_to_fetch?(current_time: current_time)
        UserService.send_recent_activity_email(user: user)
        user.subscription_tracker.update_next_fetch_time_range!
      end
    end
  end
end
```

<a href="./user-service.md" style="color: red; font-weight: bold;">OK! how does `UserService#send_recent_activity_email` work?</a>

---

### quick reference

#### `User`

```rb
require 'securerandom'

class User < ActiveRecord::Base
  ...

  after_create :create_default_subscription_tracker

  has_many :groups, through: :memberships
  has_many :memberships, foreign_key: "member_id", dependent: :destroy
  has_one :subscription_tracker,                   dependent: :destroy
  has_many :allocations,                           dependent: :destroy
  has_many :comments,                              dependent: :destroy
  has_many :contributions,                         dependent: :destroy
  has_many :buckets,                               dependent: :destroy

  scope :with_active_memberships, -> { joins(:memberships).where(memberships: {archived_at: nil}) }

  ...

  private
    ...

    def create_default_subscription_tracker
      SubscriptionTracker.create(
        user: self,
        notification_frequency: "hourly",
        recent_activity_last_fetched_at: DateTime.now.utc.beginning_of_hour
      )
    end
end
```

#### `SubscriptionTracker`

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

#### `schema`

```rb
create_table "subscription_trackers", force: :cascade do |t|
  t.integer  "user_id",                                                               null: false
  t.boolean  "comments_on_buckets_user_authored",                  default: true,     null: false
  t.boolean  "comments_on_buckets_user_participated_in",           default: true,     null: false
  t.boolean  "new_draft_buckets",                                  default: true,     null: false
  t.boolean  "new_live_buckets",                                   default: true,     null: false
  t.boolean  "new_funded_buckets",                                 default: true,     null: false
  t.boolean  "contributions_to_live_buckets_user_authored",        default: true,     null: false
  t.boolean  "contributions_to_live_buckets_user_participated_in", default: true,     null: false
  t.boolean  "funded_buckets_user_authored",                       default: true,     null: false
  t.datetime "recent_activity_last_fetched_at"
  t.string   "notification_frequency",                             default: "hourly", null: false
  t.datetime "created_at",                                                            null: false
  t.datetime "updated_at",                                                            null: false
end

add_index "subscription_trackers", ["user_id"], name: "index_subscription_trackers_on_user_id", using: :btree

create_table "users", force: :cascade do |t|
  t.string   "email",                  default: "", null: false
  t.string   "encrypted_password",     default: "", null: false
  t.string   "reset_password_token"
  t.datetime "reset_password_sent_at"
  t.datetime "remember_created_at"
  t.integer  "sign_in_count",          default: 0,  null: false
  t.datetime "current_sign_in_at"
  t.datetime "last_sign_in_at"
  t.string   "current_sign_in_ip"
  t.string   "last_sign_in_ip"
  t.datetime "created_at"
  t.datetime "updated_at"
  t.string   "name"
  t.text     "tokens"
  t.string   "provider"
  t.string   "uid"
  t.string   "confirmation_token"
  t.integer  "utc_offset"
  t.datetime "confirmed_at"
  t.datetime "joined_first_group_at"
end
```

---

### relevant tests

#### `DeliverRecentActivityDigest`

![meow](http://i.imgur.com/YSr3gYl.png)

```rb
require 'rails_helper'

describe "DeliverRecentActivityDigest" do
  after { Timecop.return }

  describe "#run!" do
    before { allow(UserService).to receive(:send_recent_activity_email) }

    context "user has no active memberships" do
      before { create(:user) }

      it "does nothing" do
        expect(UserService).not_to receive(:send_recent_activity_email)
        DeliverRecentActivityDigest.run!
      end
    end

    context "user has at least one active membership" do
      before { make_user_group_member }

      context "current_time is before user's next scheduled fetch" do
        it "does nothing" do
          time = (user.subscription_tracker.next_recent_activity_fetch_scheduled_at - 1.minute).utc
          Timecop.freeze(time) do
            expect(UserService).not_to receive(:send_recent_activity_email)
            DeliverRecentActivityDigest.run!
          end
        end
      end

      context "current_time is after user's next scheduled fetch" do
        it "calls fetches activity for user and updates their subscription_tracker's next fetch_time_range" do
          old_next_recent_activity_scheduled_at = user.subscription_tracker.next_recent_activity_fetch_scheduled_at

          time = (old_next_recent_activity_scheduled_at + 1.minute).utc
          Timecop.freeze(time) do
            expect(UserService).to receive(:send_recent_activity_email)
            DeliverRecentActivityDigest.run!
            expect(user.subscription_tracker.reload.recent_activity_last_fetched_at).to eq(old_next_recent_activity_scheduled_at)
          end
        end
      end
    end
  end
end
```

#### `SubscriptionTracker`

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

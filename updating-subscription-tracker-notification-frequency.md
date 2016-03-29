### updating the `notification_frequency` of a user's `SubscriptionTracker`

if a user changes their preferred `notification_frequency`, we want to schedule their next 'bundled recent activity' notification email, and set said email's 'time_range' for 'recent_activity' in accordance with their time zone. this is managed in the `SubscriptionTracker`'s `after_update` callback

```rb
class SubscriptionTracker < ActiveRecord::Base
  belongs_to :user
  after_update :update_recent_activity_last_fetched_at_if_notification_frequency_changed
  validates :notification_frequency, inclusion: { in: %w(never hourly daily weekly) }

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

  def next_fetch_time_range
    if recent_activity_last_fetched_at && next_recent_activity_fetch_scheduled_at
      recent_activity_last_fetched_at..next_recent_activity_fetch_scheduled_at
    end
  end

  ...

  private
    def update_recent_activity_last_fetched_at_if_notification_frequency_changed
      if self.notification_frequency_changed?
        time_now = DateTime.now.utc
        datetime = case self.notification_frequency
          when "hourly" then time_now.beginning_of_hour.utc
          when "daily" then (time_now.in_time_zone((self.user.utc_offset || 0) / 60).beginning_of_day + 6.hours).utc
          when "weekly" then (time_now.in_time_zone((self.user.utc_offset || 0) / 60).beginning_of_week + 6.hours).utc
        end
        self.update_columns(recent_activity_last_fetched_at: datetime)
      end
    end
end
```

[OK! i see how `bundled recent activity` notification emails are rescheduled when a user changes their preferred `notification_frequency`, now bring me back to the `SubscriptionTrackersController`](./subscription-trackers-controller.md)

---

### quick reference

#### `schema`

```rb
ActiveRecord::Schema.define(version: 20160329050826) do
  ...

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

  ...
end

```

---

### relevant tests

![meow](http://i.imgur.com/UgmAO0p.png)

```rb
require 'rails_helper'

RSpec.describe SubscriptionTracker, type: :model do
  # at this particular time in the world ..
  let!(:current_utc_time) { DateTime.parse("2015-11-05T05:10:00Z") }
  # for the parisians (UTC offset +60 min) it is currently 6:10AM
  let!(:parisian_user) { create(:user, utc_offset: +60) }
  # for the oaklanders (UTC offset -480 min) it is currently 9:10PM
  let!(:oakland_user) { create(:user, utc_offset: -480) }
  # and for the aucklanders (UTC offset +720 min) it is currently 6:10PM
  let!(:auckland_user) { create(:user, utc_offset: +720) }

  after { Timecop.return }

  def six_am_today_for_user_in_utc(user)
    (DateTime.now.utc.in_time_zone(user.utc_offset / 60).beginning_of_day + 6.hours).utc
  end

  def six_am_beginning_of_week_for_user_in_utc(user)
    (DateTime.now.utc.in_time_zone(user.utc_offset / 60).beginning_of_week + 6.hours).utc
  end

  def update_user_subscriptions(notification_frequency: )
    User.find_each do |user|
      user.subscription_tracker.update(notification_frequency: notification_frequency)
    end
  end

  context "notification_frequency updated" do
    context "to never" do
      it "sets recent_activity_last_fetched_at to nil" do
        Timecop.freeze(current_utc_time) do
          update_user_subscriptions(notification_frequency: "never")
          expect(parisian_user.reload.subscription_tracker.recent_activity_last_fetched_at).to eq(nil)
          expect(oakland_user.reload.subscription_tracker.recent_activity_last_fetched_at).to eq(nil)
          expect(auckland_user.reload.subscription_tracker.recent_activity_last_fetched_at).to eq(nil)
        end
      end
    end

    context "to hourly" do
      it "sets recent_activity_last_fetched_at to the beginning of this hour" do
        Timecop.freeze(current_utc_time) do
          update_user_subscriptions(notification_frequency: "never")
          update_user_subscriptions(notification_frequency: "hourly")
          beginning_of_the_hour = DateTime.parse("2015-11-05T05:00:00Z")
          expect(parisian_user.reload.subscription_tracker.recent_activity_last_fetched_at).to eq(beginning_of_the_hour)
          expect(oakland_user.reload.subscription_tracker.recent_activity_last_fetched_at).to eq(beginning_of_the_hour)
          expect(auckland_user.reload.subscription_tracker.recent_activity_last_fetched_at).to eq(beginning_of_the_hour)
        end
      end
    end

    context "to daily" do
      it "sets recent_activity_last_fetched_at to 6am today (user's local time)" do
        Timecop.freeze(current_utc_time) do
          update_user_subscriptions(notification_frequency: "daily")
          expect(parisian_user.reload.subscription_tracker.recent_activity_last_fetched_at).to eq(six_am_today_for_user_in_utc(parisian_user))
          expect(oakland_user.reload.subscription_tracker.recent_activity_last_fetched_at).to eq(six_am_today_for_user_in_utc(oakland_user))
          expect(auckland_user.reload.subscription_tracker.recent_activity_last_fetched_at).to eq(six_am_today_for_user_in_utc(auckland_user))
        end
      end
    end

    context "to weekly" do
      it "sets recent_activity_last_fetched_at to 6am of the beginning of the week (monday) (user's local time)" do
        Timecop.freeze(current_utc_time) do
          update_user_subscriptions(notification_frequency: "weekly")
          expect(parisian_user.reload.subscription_tracker.recent_activity_last_fetched_at).to eq(six_am_beginning_of_week_for_user_in_utc(parisian_user))
          expect(oakland_user.reload.subscription_tracker.recent_activity_last_fetched_at).to eq(six_am_beginning_of_week_for_user_in_utc(oakland_user))
          expect(auckland_user.reload.subscription_tracker.recent_activity_last_fetched_at).to eq(six_am_beginning_of_week_for_user_in_utc(auckland_user))
        end
      end
    end
  end

  ...
end
```

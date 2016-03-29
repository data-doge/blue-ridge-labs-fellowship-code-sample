### `UserMailer#recent_activity`

here, we prepare the email for delivery

```rb
class UserMailer < ActionMailer::Base
  ...

  def recent_activity(user:, recent_activity:)
    @user = user
    @recent_activity = recent_activity
    @last_fetched_at = @user.subscription_tracker.last_fetched_at_formatted
    mail(to: user.name_and_email,
         from: "Cobudget Updates <updates@cobudget.co>",
         subject: "My recent activity on Cobudget - from #{@last_fetched_at}"
    )
  end
end
```

[OK, and what does the html template for this email look like?](./recent-activity-email-template.md)

---

### quick reference

#### `SubscriptionTracker`

```rb
class SubscriptionTracker < ActiveRecord::Base
  belongs_to :user

  ...

  def last_fetched_at_formatted
    case notification_frequency
      when "hourly" then recent_activity_last_fetched_at.in_time_zone((user.utc_offset || 0) / 60).strftime("%l:%M%P").strip
      when "daily" then "yesterday"
      when "weekly" then "last week"
    end
  end

  ...
end
```

#### `User`

```rb
require 'securerandom'

class User < ActiveRecord::Base
  ...

  validates :name, presence: true
  validates_format_of :email, :with => /\A[^@]+@([^@\.]+\.)+[^@\.]+\z/

  def name_and_email
    "#{name} <#{email}>"
  end

  ...
end
```

---

### relevant tests

#### `SubscriptionTracker`

![meow](http://i.imgur.com/eM6WWRV.png)

```rb
require 'rails_helper'

RSpec.describe SubscriptionTracker, type: :model do
  # at this particular time in the world ..
  let!(:current_utc_time) { DateTime.parse("2015-11-05T05:10:00Z") }
  # for the parisians (UTC offset +60 min) it is currently 6:10AM
  let!(:parisian_user) { create(:user, utc_offset: +60) }

  ...

  describe "#last_fetched_at_formatted" do
    let(:subscription_tracker) { parisian_user.subscription_tracker }

    context "notification_frequency is 'hourly'" do
      it "returns local time formatted as %l:%M%P" do
        subscription_tracker.update(recent_activity_last_fetched_at: current_utc_time)
        expect(subscription_tracker.last_fetched_at_formatted).to eq("6:10am")
      end
    end

    context "notification_frequency is 'daily'" do
      it "returns 'yesterday'" do
        subscription_tracker.update(notification_frequency: "daily")
        expect(subscription_tracker.last_fetched_at_formatted).to eq("yesterday")
      end
    end

    context "notification_frequency is 'weekly'" do
      it "returns 'weekly'" do
        subscription_tracker.update(notification_frequency: "weekly")
        expect(subscription_tracker.last_fetched_at_formatted).to eq("last week")
      end
    end

    context "notification_frequency is 'never'" do
      it "returns 'never'" do
        subscription_tracker.update(notification_frequency: "never")
        expect(subscription_tracker.last_fetched_at_formatted).to be_nil
      end
    end
  end
end
```

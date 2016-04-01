*walkthrough: how users update their email notification settings - part 1(a) / 1*

### `SubscriptionTrackersController#update_email_settings`

a signed-in user updates (1) the types of events they want to subscribe to, and (2) how often they want 'bundled recent activity' email notifications about those events, by making a `POST` request to `/subscription_trackers`. these requests are handled by the `SubscriptionTrackersController`

#### `SubscriptionTrackersController`

```rb
class SubscriptionTrackersController < AuthenticatedController
  api :POST, '/subscription_trackers/update_email_settings', 'updates users email settings'
  def update_email_settings
    subscription_tracker = current_user.subscription_tracker
    subscription_tracker.update(subscription_tracker_params)
    if subscription_tracker.valid?
      render json: [subscription_tracker]
    else
      render json: subscription_tracker.errors.full_messages, status: 422
    end
  end

  private
    def subscription_tracker_params
      params.require(:subscription_tracker).permit(
        :comments_on_buckets_user_authored,
        :comments_on_buckets_user_participated_in,
        :new_draft_buckets,
        :new_live_buckets,
        :new_funded_buckets,
        :contributions_to_live_buckets_user_authored,
        :contributions_to_live_buckets_user_participated_in,
        :funded_buckets_user_authored,
        :notification_frequency
      )
    end
end
```

updating these attributes is trivial -- **[but updating a `subscription_tracker`'s `notification_frequency` is a little interesting](./updating-subscription-tracker-notification-frequency.md)**

**[OK, GOBACKHOME](./README.md)**

---

### quick reference

#### `AuthenticatedController`

```rb
class AuthenticatedController < ApplicationController
  before_action :authenticate_user!
end
```

#### `SubscriptionTracker`

```rb
class SubscriptionTracker < ActiveRecord::Base
  belongs_to :user
  validates :notification_frequency, inclusion: { in: %w(never hourly daily weekly) }

  ...
end
```

#### `User`

```rb
class User < ActiveRecord::Base

  ...

  after_create :create_default_subscription_tracker

  ...

  has_one :subscription_tracker,                   dependent: :destroy

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

#### `SubscriptionTrackerSerializer`

```rb
class SubscriptionTrackerSerializer < ActiveModel::Serializer
  embed :ids, include: true
  attributes :id,
             :comments_on_buckets_user_authored,
             :comments_on_buckets_user_participated_in,
             :new_draft_buckets,
             :new_live_buckets,
             :new_funded_buckets,
             :contributions_to_live_buckets_user_authored,
             :contributions_to_live_buckets_user_participated_in,
             :funded_buckets_user_authored,
             :notification_frequency
end
```

#### `routes`

```rb
Rails.application.routes.draw do
  ...

  scope path: 'api/v1', defaults: { format: :json } do
    ...

    resources :subscription_trackers do
      collection do
        post :update_email_settings
      end
    end
  end

  ...
end
```

---

### relevant tests

![meow](http://i.imgur.com/9iCHf1f.png)

#### `SubscriptionTrackersController`

```rb
require "rails_helper"

describe SubscriptionTrackersController, :type => :controller do
  describe "#update_email_settings" do
    let(:user) { create(:user) }
    let(:subscription_tracker) { user.subscription_tracker }
    let(:valid_params) {{
      subscription_tracker: {
        comments_on_buckets_user_authored: true,
        comments_on_buckets_user_participated_in: false,
        new_draft_buckets: true,
        new_live_buckets: false,
        new_funded_buckets: false,
        contributions_to_live_buckets_user_authored: true,
        contributions_to_live_buckets_user_participated_in: true,
        funded_buckets_user_authored: false,
        notification_frequency: "weekly"
      }
    }}

    let(:invalid_params) {{
      subscription_tracker: {
        notification_frequency: "anti-dramatic refridgerator"
      }
    }}

    context "user signed in" do
      before { request.headers.merge!(user.create_new_auth_token) }

      context "valid params" do
        before { post :update_email_settings, valid_params }

        it "returns http status 'success'" do
          expect(response).to have_http_status(:success)
        end

        it "updates subscription_tracker" do
          expect(SubscriptionTracker.find_by(valid_params[:subscription_tracker]).id).to eq(subscription_tracker.id)
        end

        it "returns subscription_tracker as json" do
          expect(parsed(response)["subscription_trackers"][0]["id"]).to eq(subscription_tracker.id)
        end
      end

      context "invalid params" do
        before { post :update_email_settings, invalid_params }

        it "returns http status 'unprocessable'" do
          expect(response).to have_http_status(422)
        end

        it "does not update subscription_tracker" do
          expect(subscription_tracker.notification_frequency).not_to eq("anti-dramatic refridgerator")
        end
      end
    end

    context "user not signed in" do
      it "returns http status 'unauthorized'" do
        post :update_email_settings, valid_params
        expect(response).to have_http_status(:unauthorized)
      end
    end
  end
end
```

*walkthrough: how customized email notifications are delivered to cobudget users - part 3(b) / 4*

### `RecentActivityService`

when requested, this service returns an `activity` `Hash` containing all new activity in a user's group since the last time activity was fetched for them. it only returns activity that the user has subscribed to.

```rb
class RecentActivityService
  attr_accessor :comments_on_bucket_you_participated_in,
                :comments_on_buckets_user_authored,
                :contributions_to_live_buckets_user_authored,
                :funded_buckets_user_authored,
                :contributions_to_live_buckets_user_participated_in,
                :new_funded_buckets,
                :new_draft_buckets,
                :new_live_buckets,
                :subscription_tracker,
                :time_range,
                :buckets_user_participated_in,
                :buckets_user_commented_on_ids,
                :buckets_user_contributed_to_ids,
                :user_buckets,
                :user_group_buckets,
                :users_active_groups,
                :user,
                :activity

  def initialize(user:)
    @user = user
    @activity = {}
  end

  def for_group(group)
    activity[group] ||= {
      contributions_to_live_buckets_user_authored:        collection_scoped_to_group(collection: contributions_to_live_buckets_user_authored,        group: group),
      funded_buckets_user_authored:                       collection_scoped_to_group(collection: funded_buckets_user_authored,                       group: group),
      comments_on_buckets_user_authored:                  collection_scoped_to_group(collection: comments_on_buckets_user_authored,                  group: group),
      comments_on_buckets_user_participated_in:           collection_scoped_to_group(collection: comments_on_buckets_user_participated_in,           group: group),
      new_live_buckets:                                   collection_scoped_to_group(collection: new_live_buckets,                                   group: group),
      new_draft_buckets:                                  collection_scoped_to_group(collection: new_draft_buckets,                                  group: group),
      contributions_to_live_buckets_user_participated_in: collection_scoped_to_group(collection: contributions_to_live_buckets_user_participated_in, group: group),
      new_funded_buckets:                                 collection_scoped_to_group(collection: new_funded_buckets,                                 group: group)
    }
  end

  def is_present?
    return false unless subscription_tracker.subscribed_to_any_activity?
    user.active_groups.map { |g| for_group(g).values }.flatten.compact.any?
  end

  private
    def collection_scoped_to_group(collection:, group:)
      scoped_collection =
        if collection.table_name == "comments" || collection.table_name == "contributions"
          collection.joins(:bucket).where(buckets: {group_id: group.id})
        elsif collection.table_name == "buckets"
          collection.where(group: group)
        end
      scoped_collection if scoped_collection && scoped_collection.any?
    end

    def contributions_to_live_buckets_user_authored
      if subscription_tracker.contributions_to_live_buckets_user_authored
        contributions_to_live_buckets_user_authored ||= Contribution.where(
          created_at: time_range,
          bucket: user_buckets.where(funded_at: nil)
        )
      end
    end

    def funded_buckets_user_authored
      if subscription_tracker.funded_buckets_user_authored
        funded_buckets_user_authored ||= user_buckets.where(funded_at: time_range)
      end
    end

    def comments_on_buckets_user_authored
      if subscription_tracker.comments_on_buckets_user_authored
        comments_on_buckets_user_authored ||= Comment.where(bucket: user_buckets, created_at: time_range)
      end
    end

    def comments_on_buckets_user_participated_in
      if subscription_tracker.comments_on_buckets_user_participated_in
        comments_on_buckets_user_participated_in ||= Comment.where(bucket: buckets_user_participated_in, created_at: time_range)
      end
    end

    def new_live_buckets
      if subscription_tracker.new_live_buckets
        new_live_buckets ||= user_group_buckets.where(status: "live", live_at: time_range)
      end
    end

    def new_draft_buckets
      if subscription_tracker.new_draft_buckets
        new_draft_buckets ||= user_group_buckets.where(status: "draft", created_at: time_range)
      end
    end

    def contributions_to_live_buckets_user_participated_in
      if subscription_tracker.contributions_to_live_buckets_user_participated_in
        contributions_to_live_buckets_user_participated_in ||= Contribution.where(
          created_at: time_range,
          bucket: buckets_user_participated_in.where(funded_at: nil)
        )
      end
    end

    def new_funded_buckets
      if subscription_tracker.new_funded_buckets
        new_funded_buckets ||= user_group_buckets.where(status: "funded", funded_at: time_range)
                                                 .where.not(user: user)
      end
    end

    ###############################################################

    def subscription_tracker
      subscription_tracker ||= user.subscription_tracker
    end

    def time_range
      time_range ||= subscription_tracker.next_fetch_time_range
    end

    def buckets_user_participated_in
      buckets_user_participated_in ||= Bucket.where(id: (buckets_user_commented_on_ids + buckets_user_contributed_to_ids).uniq)
    end

    def buckets_user_commented_on_ids
      Bucket.where.not(user: user).joins(:comments).where(comments: {user: user}).pluck(:id)
    end

    def buckets_user_contributed_to_ids
      Bucket.where.not(user: user).joins(:contributions).where(contributions: {user: user}).pluck(:id)
    end

    def user_buckets
      user_buckets ||= user.buckets
    end

    def user_group_buckets
      user_group_buckets ||= Bucket.where(group: user.active_groups)
    end
end
```

**[GOBACKTO: `UserMailer#recent_activity`](./user-mailer.md)**

---

#### quick reference

##### `User`

```rb
require 'securerandom'

class User < ActiveRecord::Base
  ...

  has_one :subscription_tracker,                   dependent: :destroy

  ...

  def active_groups
    Group.joins(:memberships).where(memberships: {member: self, archived_at: nil})
  end

  ...
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

  def subscribed_to_any_activity?
    comments_on_buckets_user_authored || comments_on_buckets_user_participated_in || new_draft_buckets || new_live_buckets || new_funded_buckets || contributions_to_live_buckets_user_authored || contributions_to_live_buckets_user_participated_in || funded_buckets_user_authored || recent_activity_last_fetched_at
  end

  def next_fetch_time_range
    if recent_activity_last_fetched_at && next_recent_activity_fetch_scheduled_at
      recent_activity_last_fetched_at..next_recent_activity_fetch_scheduled_at
    end
  end

  ...
end
```

---

#### relevant tests

##### `RecentActivityService`

![meow](http://i.imgur.com/faCZn7W.png)

```rb
require 'rails_helper'

describe "RecentActivityService" do
  let(:current_time) { DateTime.now.utc }
  let(:user) { create(:user) }
  let(:group) { create(:group) }
  let(:membership) { create(:membership, member: user, group: group) }
  # notification_frequency set to 'hourly' by default
  let(:subscription_tracker) { user.subscription_tracker }

  before do
    Timecop.freeze(current_time - 70.minutes) do
      create(:allocation, user: user, group: group, amount: 20000)
      subscription_tracker.update(recent_activity_last_fetched_at: current_time - 1.hour)

      @bucket_user_participated_in = create(:bucket, group: group, target: 420, status: "live")
      create(:comment, user: user, bucket: @bucket_user_participated_in)

      @bucket_user_authored = create(:bucket, group: group, user: user, target: 420, status: "live")
      @bucket_user_authored_to_be_fully_funded = create(:bucket, group: group, user: user, target: 420, status: "live")
    end
  end

  after { Timecop.return }

  context "recent_activity exists" do
    before do
      # make some old activity
      Timecop.freeze(current_time - 70.minutes) do
        # create 1 comments on @bucket_user_participated_in
        create_list(:comment, 1, bucket: @bucket_user_participated_in)

        # create 1 comments on @bucket_user_authored
        create_list(:comment, 1, bucket: @bucket_user_authored)

        # create 1 contributions for @bucket_user_participated_in
        create_list(:contribution, 1, bucket: @bucket_user_participated_in)

        # create 1 contributions for @bucket_user_authored
        create_list(:contribution, 1, bucket: @bucket_user_authored)

        # create 1 contribution for@bucket_user_authored_to_be_fully_funded
        create(:contribution, bucket:@bucket_user_authored_to_be_fully_funded)

        # create 1 new draft_buckets
        create_list(:bucket, 1, status: "draft", group: group, target: 420)

        # create 1 new live_buckets
        create_list(:bucket, 1, status: "live", group: group, target: 420)

        # create 1 new funded_buckets
        create_list(:bucket, 1, status: "live", group: group, target: 420).each do |bucket|
          create(:contribution, bucket: bucket, amount: 420)
        end
      end

      # make some new activity
      Timecop.freeze(current_time - 30.minutes) do
        # create 2 comments on @bucket_user_participated_in
        create_list(:comment, 2, bucket: @bucket_user_participated_in)

        # create 2 comments on @bucket_user_authored
        create_list(:comment, 2, bucket: @bucket_user_authored)

        # create 2 contributions for @bucket_user_participated_in
        create_list(:contribution, 2, bucket: @bucket_user_participated_in)

        # create 2 contributions for @bucket_user_authored
        create_list(:contribution, 2, bucket: @bucket_user_authored)

        # create 2 contributions for @bucket_user_authored_to_be_fully_funded
        create(:contribution, bucket:@bucket_user_authored_to_be_fully_funded)
        create(:contribution,
          bucket:@bucket_user_authored_to_be_fully_funded,
          amount:@bucket_user_authored_to_be_fully_funded.amount_left
        )

        # create 2 new draft_buckets
        create_list(:bucket, 2, status: "draft", group: group, target: 420)

        # create 2 new live_buckets
        create_list(:bucket, 2, status: "live", group: group, target: 420)

        # create 2 new funded_buckets
        create_list(:bucket, 2, status: "live", group: group, target: 420).each do |bucket|
          create(:contribution, bucket: bucket, amount: 420)
        end
      end
    end

    context "user subscribed to all recent_activity" do
      it "returns all recent_activity as a hash" do
        Timecop.freeze(current_time) do
          recent_activity = RecentActivityService.new(user: user)
          activity = recent_activity.for_group(group)
          expect(activity[:comments_on_buckets_user_participated_in].length).to eq(2)
          expect(activity[:comments_on_buckets_user_authored].length).to eq(2)
          expect(activity[:contributions_to_live_buckets_user_authored].length).to eq(2)
          expect(activity[:contributions_to_live_buckets_user_participated_in].length).to eq(2)

          expect(activity[:funded_buckets_user_authored].length).to eq(1)

          expect(activity[:new_draft_buckets].length).to eq(2)
          expect(activity[:new_live_buckets].length).to eq(2)
          expect(activity[:new_funded_buckets].length).to eq(2)

          expect(recent_activity.is_present?).to eq(true)
        end
      end
    end

    context "user not subscribed to any recent_activity" do
      it "returns nil instead" do
        subscription_tracker.update(
          comments_on_buckets_user_authored: false,
          comments_on_buckets_user_participated_in: false,
          new_draft_buckets: false,
          new_live_buckets: false,
          new_funded_buckets: false,
          contributions_to_live_buckets_user_authored: false,
          contributions_to_live_buckets_user_participated_in: false,
          funded_buckets_user_authored: false
        )

        recent_activity = RecentActivityService.new(user: user)

        expect(recent_activity.for_group(group)).to eq({
          comments_on_buckets_user_participated_in: nil,
          comments_on_buckets_user_authored: nil,
          contributions_to_live_buckets_user_authored: nil,
          funded_buckets_user_authored: nil,
          contributions_to_live_buckets_user_participated_in: nil,
          new_funded_buckets: nil,
          new_draft_buckets: nil,
          new_live_buckets: nil
        })

        expect(recent_activity.is_present?).to eq(false)
      end
    end
  end

  context "user subscribed to all recent_activity, but none exists" do
    it "returns nil instead" do
      recent_activity = RecentActivityService.new(user: user)

      expect(recent_activity.for_group(group)).to eq({
        comments_on_buckets_user_participated_in: nil,
        comments_on_buckets_user_authored: nil,
        contributions_to_live_buckets_user_authored: nil,
        funded_buckets_user_authored: nil,
        contributions_to_live_buckets_user_participated_in: nil,
        new_funded_buckets: nil,
        new_draft_buckets: nil,
        new_live_buckets: nil
      })

      expect(recent_activity.is_present?).to eq(false)
    end
  end
end
```

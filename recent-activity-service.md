### `RecentActivityService`

checks to see what types of events a user is subscribed to, and prepares an `activity` `Hash` for each of the `user`s groups

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
      return nil unless collection && collection.any?
      if collection.table_name == "comments" || collection.table_name == "contributions"
        collection.joins(:bucket).where(buckets: {group_id: group.id})
      elsif collection.table_name == "buckets"
        collection.where(group: group)
      end
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

[OK! now that i know how `RecentActivityService` prepares a hash of `activity` for a given user, i'm ready to go back to `UserService#send_recent_activity_email`](./user-service.md)

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

  ...

  def active_groups
    Group.joins(:memberships).where(memberships: {member: self, archived_at: nil})
  end

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

#### `schema`

```rb
ActiveRecord::Schema.define(version: 20160329050826) do

  # These are extensions that must be enabled in order to support this database
  enable_extension "plpgsql"

  ...

  create_table "buckets", force: :cascade do |t|
    t.datetime "created_at"
    t.datetime "updated_at"
    t.string   "name"
    t.text     "description"
    t.integer  "user_id"
    t.integer  "target"
    t.integer  "group_id"
    t.string   "status",            default: "draft"
    t.datetime "funding_closes_at"
    t.datetime "funded_at"
    t.datetime "live_at"
  end

  add_index "buckets", ["group_id"], name: "index_buckets_on_group_id", using: :btree
  add_index "buckets", ["user_id"], name: "index_buckets_on_user_id", using: :btree

  create_table "comments", force: :cascade do |t|
    t.text     "body",       null: false
    t.integer  "user_id"
    t.integer  "bucket_id"
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
  end

  add_index "comments", ["bucket_id"], name: "index_comments_on_bucket_id", using: :btree
  add_index "comments", ["user_id"], name: "index_comments_on_user_id", using: :btree

  create_table "contributions", force: :cascade do |t|
    t.integer  "user_id"
    t.integer  "bucket_id"
    t.datetime "created_at"
    t.datetime "updated_at"
    t.integer  "amount",     null: false
  end

  add_index "contributions", ["bucket_id"], name: "index_contributions_on_bucket_id", using: :btree
  add_index "contributions", ["user_id"], name: "index_contributions_on_user_id", using: :btree

  ...

  create_table "groups", force: :cascade do |t|
    t.string   "name"
    t.datetime "created_at"
    t.datetime "updated_at"
    t.string   "currency_symbol", default: "$"
    t.string   "currency_code",   default: "USD"
  end

  create_table "memberships", force: :cascade do |t|
    t.integer  "group_id",                    null: false
    t.integer  "member_id",                   null: false
    t.boolean  "is_admin",    default: false, null: false
    t.datetime "created_at"
    t.datetime "updated_at"
    t.datetime "archived_at"
  end

  add_index "memberships", ["group_id"], name: "index_memberships_on_group_id", using: :btree
  add_index "memberships", ["member_id"], name: "index_memberships_on_member_id", using: :btree

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

  add_index "users", ["email"], name: "index_users_on_email", unique: true, using: :btree
  add_index "users", ["reset_password_token"], name: "index_users_on_reset_password_token", unique: true, using: :btree
end
```

---

### relevant tests

#### `RecentActivityService`

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

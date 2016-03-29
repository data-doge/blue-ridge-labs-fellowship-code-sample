#### `recent_activity.html.erb`

the corresponding template for this email looks like this:

```html
<body style="font-family: Arial; font-size: 12px; padding-left: 20px; padding-top: 20px;">
  <h1 style="font-weight: normal; margin-bottom: 5px">Here's what you missed...</h1>

  <p style="margin-top: 0; margin-bottom: 20px">
    Activity since <%= @last_fetched_at %> (when you last logged in)
  </p>

  <ul style="list-style: none; padding-left: 0">
    <% @user.active_groups.each do |group| %>
      <li class="group-activity">
        <% activity = @recent_activity.for_group(group) %>

        <h2 style="font-weight: normal">In <%= group.name %></h2>

        <ul style="list-style: none">
          <% if activity[:contributions_to_live_buckets_user_authored] %>
            <%= render partial: "user_mailer/recent_activity_partials/contributions_to_live_buckets_user_authored", locals: {contributions_grouped_by_bucket: activity[:contributions_to_live_buckets_user_authored].group_by { |c| c.bucket }, user: @user} %>
          <% end %>

          <% if activity[:funded_buckets_user_authored] %>
            <%= render partial: "user_mailer/recent_activity_partials/funded_buckets_user_authored", locals: {buckets: activity[:funded_buckets_user_authored], user: @user} %>
          <% end %>

          <% if activity[:comments_on_buckets_user_authored] %>
            <%= render partial: "user_mailer/recent_activity_partials/comments_on_buckets_user_authored", locals: {comments_grouped_by_bucket: activity[:comments_on_buckets_user_authored].group_by { |c| c.bucket }, user: @user} %>
          <% end %>

          <% if activity[:comments_on_buckets_user_participated_in] %>
            <%= render partial: "user_mailer/recent_activity_partials/comments_on_buckets_user_participated_in", locals: {comments_grouped_by_bucket: activity[:comments_on_buckets_user_participated_in].group_by { |c| c.bucket }, user: @user} %>
          <% end %>

          <% if activity[:new_live_buckets] %>
            <%= render partial: "user_mailer/recent_activity_partials/new_live_buckets", locals: {buckets: activity[:new_live_buckets], membership: @user.membership_for(group)} %>
          <% end %>

          <% if activity[:new_draft_buckets] %>
            <%= render partial: "user_mailer/recent_activity_partials/new_draft_buckets", locals: {buckets: activity[:new_draft_buckets]} %>
          <% end %>

          <% if activity[:contributions_to_live_buckets_user_participated_in] %>
            <%= render partial: "user_mailer/recent_activity_partials/contributions_to_live_buckets_user_participated_in", locals: {contributions_grouped_by_bucket: activity[:contributions_to_live_buckets_user_participated_in].group_by { |c| c.bucket }, user: @user} %>
          <% end %>

          <% if activity[:new_funded_buckets] %>
            <%= render partial: "user_mailer/recent_activity_partials/new_funded_buckets", locals: {buckets: activity[:new_funded_buckets], user: @user} %>
          <% end %>
        </ul>
      </li>
    <% end %>
  </ul>

  <footer style="font-size: 10px; margin-top: 20px">
    &lt;3 from cobudget. to change your email settings <a href=<%= "#{root_url}#/email_settings" %>>click here</a>
  </footer>
</body>
```

and here are the partials it is rendering:
  - [`_contributions_to_live_buckets_user_authored.html.erb`]('./recent_activity_partials/_contributions_to_live_buckets_user_authored.html.erb')
  - [`_funded_buckets_user_authored.html.erb`]('./recent_activity_partials/_funded_buckets_user_authored.html.erb')
  - [`_comments_on_buckets_user_authored.html.erb`]('./recent_activity_partials/_comments_on_buckets_user_authored.html.erb')
  - [`_comments_on_buckets_user_participated_in.html.erb`]('./recent_activity_partials/_comments_on_buckets_user_participated_in.html.erb')
  - [`_new_live_buckets.html.erb`]('./recent_activity_partials/_new_live_buckets.html.erb')
  - [`_new_draft_buckets.html.erb`]('./recent_activity_partials/_new_draft_buckets.html.erb')
  - [`_contributions_to_live_buckets_user_participated_in.html.erb`]('./recent_activity_partials/_contributions_to_live_buckets_user_participated_in.html.erb')
  - [`_new_funded_buckets.html.erb`]('./recent_activity_partials/_new_funded_buckets.html.erb')

---

### here's what the email looks like:

![meow](http://i.imgur.com/mvwVqAr.png)

### and here's the mailer preview that rendered the email above

```rb
class UserMailerPreview < ActionMailer::Preview
  def recent_activity
    user = generate_user
    group = Group.create(name: Faker::Company.name)
    membership = Membership.create(member: user, group: group)
    generate_recent_activity_for(membership: membership)
    recent_activity = RecentActivityService.new(user: user)
    UserMailer.recent_activity(user: user, recent_activity: recent_activity)
  end

  private
    def generate_recent_activity_for(membership: )
      current_time = DateTime.now.utc
      user = membership.member
      group = membership.group
      # note: notification_frequency set to "hourly" by default
      subscription_tracker = user.subscription_tracker

      Allocation.create(user: user, group: group, amount: 20000)
      subscription_tracker.update(recent_activity_last_fetched_at: current_time - 1.hour)

      bucket_user_participated_in = generate_bucket(group: group)
      generate_comment(user: user, bucket: bucket_user_participated_in)
      bucket_user_participated_in_to_be_fully_funded = generate_bucket(group: group, status: "live")
      generate_contribution(user: user, bucket: bucket_user_participated_in_to_be_fully_funded)

      live_bucket_user_authored = generate_bucket(group: group, user: user, status: "live")
      bucket_user_authored_to_be_fully_funded = generate_bucket(group: group, user: user, status: "live")


      Timecop.freeze(current_time - 30.minutes) do
        # create 2 comments on bucket_user_participated_in
        generate_comment(bucket: bucket_user_participated_in)
        generate_comment(bucket: bucket_user_participated_in)

        # create 2 comments on live_bucket_user_authored
        generate_comment(bucket: live_bucket_user_authored)
        generate_comment(bucket: live_bucket_user_authored)

        # create 2 contributions for bucket_user_participated_in
        generate_contribution(bucket: bucket_user_participated_in)
        generate_contribution(bucket: bucket_user_participated_in)

        # create 2 contributions for live_bucket_user_authored
        generate_contribution(bucket: live_bucket_user_authored)
        generate_contribution(bucket: live_bucket_user_authored, user: user)

        # create 2 contributions for bucket_user_participated_in_to_be_fully_funded
        generate_contribution(bucket: bucket_user_participated_in_to_be_fully_funded)
        generate_contribution(
          bucket: bucket_user_participated_in_to_be_fully_funded,
          amount: bucket_user_participated_in_to_be_fully_funded.amount_left
        )

        # create 2 contributions for bucket_user_authored_to_be_fully_funded
        generate_contribution(bucket: bucket_user_authored_to_be_fully_funded)
        generate_contribution(
          bucket: bucket_user_authored_to_be_fully_funded,
          amount: bucket_user_authored_to_be_fully_funded.amount_left
        )

        # create 2 new draft_buckets
        generate_bucket(status: "draft", group: group, target: 420)

        # create 2 new live_buckets
        generate_bucket(status: "live", group: group, target: 420)
      end

      Timecop.return
    end

    def generate_user
      User.create(name: Faker::Name.name, email: Faker::Internet.email, password: "password")
    end

    def generate_bucket(user: nil, group:, status: "draft", target: 420)
      if user.nil?
        user = generate_user
        group.add_member(user)
      end
      Bucket.create(name: Faker::Lorem.sentence, description: Faker::Lorem.paragraph, target: target, user: user, group: group, status: status)
    end

    def generate_comment(user: nil, bucket:)
      if user.nil?
        user = generate_user
        group = bucket.group
        group.add_member(user)
      end
      Comment.create(user: user, bucket: bucket, body: Faker::Lorem.sentence)
    end

    def generate_contribution(user: nil, bucket:, amount: 1)
      if user.nil?
        user = generate_user
        group = bucket.group
        group.add_member(user)
        Allocation.create(user: user, group: group, amount: amount)
      end
      Contribution.create(user: user, bucket: bucket, amount: amount)
    end
end
```

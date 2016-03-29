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

### [and here's the mailer preview that rendered the email above](./user-mailer-preview.md)

---

[OK! i understand how customized 'bundled recent activity' email notifications are sent out to Cobudget users, now take me back to the main README plz](./README.md)

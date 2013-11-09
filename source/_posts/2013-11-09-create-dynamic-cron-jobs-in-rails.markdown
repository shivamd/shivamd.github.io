---
layout: post
title: "Create Dynamic Cron Jobs in Rails"
date: 2013-11-09 03:38
comments: true
categories: ["rails","cron", "delayed-jobs"]
---

I have used the likes of [Delayed Jobs][1] and [Sidekiq][2] in previous apps, but for an app I am working on I had to create cron jobs based on user settings. I needed someway to iterate over the user's settings and create the crons needed. I came across the [whenever][3] gem that makes it easy to create cron jobs. With a little tweaking I was able to create cron jobs based on a user's setting. 

### Gem installation

In your Gemfile: 

{% codeblock lang:ruby %}
gem 'whenever', require :false
{% endcodeblock %}
The reason we add require false is because whenever creates cron jobs outside of the rails app. 

Next run the following command from the root of your app: 
{% codeblock lang:ruby %}
bundle exec wheneverize
{% endcodeblock %}
This will create a schedule.rb in the config file, this is where your logic for setting up crons will be. You can have a look at this [railscasts][4] on how to setup crons. 
### Requiring the rails app

In order to have access to your models inside the file, you need to require the application. This can be done with the following line.

{% codeblock lang:ruby config/scheduler.rb %}
require "./"+ File.dirname(__FILE__) + "/environment.rb"
{% endcodeblock %}

This will load up the application which gives us access to our rails models. Now let's say you have many users with different reminder settings.

{% codeblock lang:ruby config/scheduler.rb %}
require "./"+ File.dirname(__FILE__) + "/environment.rb"
set :output, "#{path}/log/cron.log" #logs
@users = User.all
@users.each do |user|
  every user.reminder_frequency.to_sym, at: user.reminder_time do  
    runner "user.send_reminder"
  end 
end
{% endcodeblock %}
**reminder_frequency** can contain for example: "sunday" or "weekday"  
**reminder_time** contains the timestamp: "6:00pm"

This will create the cron jobs for every user. 
Running the following command will confirm that: 
{% codeblock lang:bash%}
bundle exec whenever
{% endcodeblock %}
There are many different solutions out there for scheduled jobs, whenever makes it really easy with it's syntax and simplicity.
Let me know if you have any questions.

[1]:https://github.com/collectiveidea/delayed_job
[2]:https://github.com/mperham/sidekiq
[3]:https://github.com/javan/whenever
[4]:http://railscasts.com/episodes/164-cron-in-ruby 

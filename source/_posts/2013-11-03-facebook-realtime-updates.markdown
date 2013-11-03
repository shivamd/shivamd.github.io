---
layout: post
title: "Facebook Realtime Updates in a Rails Application"
date: 2013-11-03 00:01
comments: true
categories: ["code", "api", "facebook"]
---

Since Facebook has rolled out realtime updates it has been easier to gather information. Instead of hitting the API on a set time interval, we can just wait for Facebook to ping us and go grab the information we need. This all sounds good in theory but as I had to integrate realtime updates for an application I am working on, there were quite a few pitfalls and I wish I knew some of these before hand.

### Setting up a Facebook application

I am not going to walk you through creating a Facebook application, [Ryan Bates][1] already does a good job with it. Once you have that setup and the permissions you need for this application head over to your [Facebook applications][2] dashboard. Over here select the application in which you would like to enable realtime updates and hit the edit button on the top right. This should take you to a page with the basic information of your application, and on the right under settings there should be 'realtime updates'.

{% img center /images/realtime_settings.png %}

As you can see I have a user object and the fields I want to follow. I have selected all the fields for the demo. You should only select the fields your app has permissions for, otherwise it would be pointless as you won't be able to get the information. 
For the callback URL you should point it to the route of the realtime action, but you must be thinking how will I test this locally? That's where [Ngrok][3] comes in. Ngrok creates a tunnel from the public internet to a port on your local machine. This is perfect for a situation like this where Facebook has to verify that you are authorized. 
The information on the site is self explanatory, once installed have it listen to the same port as your server. 

{% codeblock lang:bash %}
rails s -p 3000 
# on a different Terminal window
path/to/ngrok 3000
{% endcodeblock %}
Ngrok will give you a URL which you can paste in to the callback section followed by the route to the realtime action. For the verify token any random string will do. 

### Setting up the controller

Now create a facebook controller with the real_time method. 

{% codeblock lang:ruby app/controllers/facebook_controller.rb %}
   def real_time
    if request.method == "GET"
      if params['hub.mode'] =='subscribe' && params['hub.verify_token'] =='stringToken'
        render :text => params['hub.challenge']
      else
        render :text => 'Failed to authorize facebook challenge request'
      end
    elsif request.method == "POST"
      #do stuff with information 
    end
  end
{% endcodeblock %}

And the routes should look like this: 
{% codeblock lang:ruby config/routes.rb %}
    namespace :facebook do
      match "/real_time", :action => :real_time, :as => 'facebook_subscription', :via => [:get,:post]
    end
{% endcodeblock %}

Now that we have the route set up for both GET&POST, when we click on test on the realtime updates page Facebook will make a get request to this route ensuring we are authorized. The default hub.mode set by Facebook is subscribe and you must make sure that the verify_token matches what you have entered on Facebook. If all went well Facebook will prompt you with the following message:


{% img center /images/realtime_success.png %}

### And we are done...not really. 

Your application is running, realtime updates are setup and you have a test account to play around with. On every change the user makes to the field you have subscribed to, Facebook will ping your servers. However it will only do so with the Facebook ID, the fields that have changed and a timestamp. This means that we still have to query the API and this can be quite a pain when it comes to things like comments or photos. The good thing is I have banged my head on this for the past week and should be able to alleviate a lot of your troubles. 

Facebook example response for relationship change:

{% codeblock lang:ruby %}
{"object"=>"user", "entry"=>[{"uid"=>"100928374283421", "id"=>"100928374283421", "time"=>1383426584, "changed_fields"=>["relationship_status", "feed"]}], "facebook"=>{"object"=>"user", "entry"=>[{"uid"=>"100928374283421", "id"=>"100928374283421", "time"=>1383426584, "changed_fields"=>["relationship_status", "feed"]}]}}
{% endcodeblock %}
Now in order to actually grab the change we have to make a call to the Facebook API requesting this information. You will only be able to get this information if you have the right permissions. 

{% codeblock lang:ruby %}
  #you'll need the HTTParty gem
  HTTParty.get("https://graph.facebook.com/#{uid}?fields=relationship_status&access_token=" + auth_token)
{% endcodeblock %}
This gives the information for the relationship status. This seems and is straight forward and these are the building pieces for every change. However there are some caveats. Below are snippets in order to get events,friends,posts,photos&comments. 

### Events
Facebook example response for events change:

{% codeblock lang:ruby %}
{"object"=>"user", "entry"=>[{"uid"=>"100928374283421", "id"=>"100928374283421", "time"=>1383426584, "changed_fields"=>["events", "feed"]}], "facebook"=>{"object"=>"user", "entry"=>[{"uid"=>"100928374283421", "id"=>"100928374283421", "time"=>1383426584, "changed_fields"=>["events", "feed"]}]}}
{% endcodeblock %}

And the query to get all event ID's and the respective information: 

{% codeblock lang:ruby %}
  #where token is the user's auth token
  event_ids = FbGraph::Query.new("SELECT eid, rsvp_status FROM event_member WHERE uid = #{uid}").fetch(access_token: token)
  event_ids.each do |event_id|
    event_info = FbGraph::Query.new("SELECT creator, eid, all_members_count, attending_count, not_replied_count, declined_count, unsure_count, privacy, host, location, name, description, venue, pic, start_time, end_time FROM event WHERE eid=#{event_id['eid']}").fetch(access_token: token)
  end
{% endcodeblock %}

### Friends 
Facebook example response for friends change: 
{% codeblock lang:ruby %}
{"object"=>"user", "entry"=>[{"uid"=>"100928374283421", "id"=>"100928374283421", "time"=>1383426584, "changed_fields"=>["friends", "feed"]}], "facebook"=>{"object"=>"user", "entry"=>[{"uid"=>"100928374283421", "id"=>"100928374283421", "time"=>1383426584, "changed_fields"=>["friends", "feed"]}]}}
{% endcodeblock %}

And the query to get all friend's related information: 

{% codeblock lang:ruby %}
#token is the user's auth token
friends = HTTParty.get("https://graph.facebook.com/" + uid + "/friends?access_token=" + token + "&fields=birthday,age_range,gender,first_name,last_name,name,id").parsed_response["data"]
friends.each do |friend|
  friend_data = HTTParty.get("https://graph.facebook.com/" + friend["id"] + "?access_token=" + token + "&fields=about,age_range,birthday,link,bio,events,gender,interested_in,location,name,relationship_status,significant_other,first_name,interested_in,last_name,locale,location,username,family,likes,locations,movies,picture,photos,posts,security_settings").parsed_response
end
{% endcodeblock %}

### Posts,Photos&Comments 

Now this is where things start to get complicated. As you can see Facebook sends 'feed' as a changed field with every column.With every change to anything on Facebook it needs to update the feed and in order to get the photos,posts and comments we must query the feed or stream in the Facebook Query Language. Despite being able to get posts on the 'status' changed field and photos on the 'photos' changed field, I found it best to look for the feed change and make a query.

One great thing with the Facebook Query Language is that you can add conditions to your queries.When making a query after a realtime update, you can put a conditional for updated time to be just before the timestamp given by Facebook.That's what I thought too, but after a lot of digging Facebook doesn't give you the right information. The only way around this is to get all the posts, sort them by updated time and then use ruby's built in methods to select the posts before a certain time. 

Facebook example response for feed: 
{% codeblock lang:ruby %}
#as you can see it gives us the time
{"object"=>"user", "entry"=>[{"uid"=>"100928374283421", "id"=>"100928374283421", "time"=>1383426584, "changed_fields"=>["feed"]}], "facebook"=>{"object"=>"user", "entry"=>[{"uid"=>"100928374283421", "id"=>"100928374283421", "time"=>1383426584, "changed_fields"=>["feed"]}]}}
{% endcodeblock %}

And the query to get the latest posts: 
{% codeblock lang:ruby %} 
latest_posts = FbGraph::Query.new("SELECT comment_info,post_id,message,actor_id,attachment,is_hidden,like_info,message_tags,permalink,privacy,created_time,updated_time,share_count,target_id,type,with_tags from stream WHERE source_id=me() and (message or attachment) ORDER BY updated_time DESC LIMIT 100").fetch(access_token:token)
posts = latest_posts.select { |post| post["updated_time"] > (time-30) }.select{ |post| post["message"].present? || post["attachment"]["media"] }
{% endcodeblock %}
What's going on here? In the first query, I find all the posts of the user limit them to 100, order them by updated time and make sure that either message or attachment(photo) is present. In the next line I take only the posts that have an updated time 30 seconds(approximate time it takes Facebook to ping your servers) before the timestamp. And select the ones that either have a message or photos. Now we have a bunch of posts, time to decipher whether it is a status update, photo or comment. 

{% codeblock lang:ruby %}
#posts from the previous codeblock 
posts.each |post|
  if post["message"].present? 
    #create post
  elsif post["attachment"] && post["attachment"]["media"]
    #create photo with post["attachment"]["media"] passed as a parameter
  end
end
{% endcodeblock %}

The post is fairly simple from here, we already have the attributes we need. Now in order to get comments for post, we can do the following query: 

{% codeblock lang:ruby %}
#you can sort by time if needed 
comments = FbGraph::Query.new("SELECT id, user_likes, text, post_id, fromid, likes, time FROM comment WHERE post_id = '#{post_id}' ").fetch(access_token:token)
{% endcodeblock %}
The use case for me is to update all the comments for the posts and add the new ones, hence I query for all of them. Facebook gives you the flexibility with the query language for any use case possible. Posts,and comments done and now time for photos. 
After rigourous testing, Facebook gives only 9 photos. Let's say a user added an album, added 50 photos to this with normal queries that I am about to display Facebook will only give 9 photos back. The worst part is there's no way to decipher if there are more photos. One trick I found is, if Facebook gives you 9 photos that means there are more photos. Hence I find all the photos in the album. 

{% codeblock lang:ruby %}
#for each post from the previous block that has attachments
photo_ids = post.map{|i| i["photo"]["fbid"] }
if photo_ids.count == 9 
  # get data for first photo
  id = photo_ids.first
  photo_data = HTTParty.get("https://graph.facebook.com/#{id}?access_token=#{user.auth_token}")
  #parse album ID and get all photo id's for that album
  album_id = photo_data["link"].match(/(a\.)([^\.]+)/)[-1]
  response = HTTParty.get("https://graph.facebook.com/#{album_id}/photos?fields=id&access_token=#{user.auth_token}")
  photo_ids = response.parsed_response["data"].map { |i| i["id"] } 
end
{% endcodeblock %}
I get the photo_ids from the first line, and if there are 9 which means more photos I get all photo IDs from the album. 
Iterating over all the photo ID's to get the related informaton: 
{% codeblock lang:ruby %}
photo_ids.each do |id| 
  photo_data = HTTParty.get("https://graph.facebook.com/#{id}?access_token=#{user.auth_token}")
  #and for comments 
  if photo_data.has_key?("comments")
    photo_data["comments"]["data"].each do |comment|
      #use comment information
    end
  end
end
{% endcodeblock %}
As you can see there's a lot of edge cases here, so you must test thoroughly. Both manually and with automated tests. One good thing about the photo query is that it gives you comment information too. The following takes care of status updates, comments on them, photos posted on the wall, comments to those and if the user posts an album. The only part I haven't been able to get is realtime comment on a photo which is inside an album. I am currently working on that, and will update this post if I find a way. 

### Wrapping it all up. 

Despite it's limitations Facebook has done a great job with the realtime updates and [Facebook Query Language][4]. I can't stress this point enough, test thoroughly. Manually and with automated testing such as Rspec. I have written tests for this and might showcase it in a later blog post. Whenever you get stuck, instead of searching StackOverflow which at times can help, the best approach I found is to use the [Graph Explorer][5].

Hope that was useful, let me know if you have any questions.


[1]: http://railscasts.com/episodes/360-facebook-authentication?view=asciicast
[2]: https://developers.facebook.com/apps
[3]: https://ngrok.com/
[4]: https://developers.facebook.com/docs/reference/fql/
[5]: https://developers.facebook.com/tools/explorer

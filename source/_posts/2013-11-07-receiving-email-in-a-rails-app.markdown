---
layout: post
title: "Receiving Email in a Rails App"
date: 2013-11-07 02:07
comments: true
categories: ["code", "rails", "email"]
---

Receiving email in a Rails app can be tricky. The Action Mailer library has minimal support for this and it can be quite tricky. This is where SendGrid's [parse API][1] comes in along with the gem released by ThoughtBot called [Griddler][2].

### Setup

Start of by adding the griddler gem to your gemfile.

```
gem "griddler"
```
Followed by a bundle install. If you run rake routes, you will see Griddler has created a route to receive mails on.
If you need to edit this route, you can paste it in your routes file. 

{% codeblock lang:ruby config/routes.rb %}
post '/email_processor' => 'griddler/emails#create'
{% endcodeblock %}

By default Griddler will look for an EmailProcessor class. You can create one in the models folder.

{% codeblock lang:ruby app/models/email_processor.rb %}
class EmailProcessor 
  def self.process(email)
    #create models, process reports etc.
  end
end
{% endcodeblock %}

These are the following attributes available on the email received. 
{% codeblock %}
.to 
.from
.subject
.body
.raw_text
.raw_html
.raw_body
.attachments
.headers
.raw_headers
{% endcodeblock %}


### Email server setup

For my app I ended up using SendGrid, which has an add-on for Heroku. Once your account is setup, go the [parse webhook settings page][8].

{% img center /images/sendgrid_settings.png %}


**In order for this to work you need a domain name**. Your site doesn't necessarily have to be deployed on it. Enter the domain you are going to use in the domain field. For the URI enter the route where you want to receive mails. I have used [Ngrok][3] as I wanted to test it in development.

The last part needed in order to receive emails is to point your domain's MX record to, in this case, mx.sendgrid.net.
What is a MX record?
The full form is Mail Exchanger record. It specifies how email should be routed for a particular domain. It specifies a mail server, in this case sendgrid, for accepting emails for your domain.
Go to your domain settings and create a MX record which points to mx.sendgrid.net. An example for [Namecheap][4] would look like this. 
You will also need to setup an email for your domain, by default namecheap creates an info@yourdomain.com.

{% img center /images/namecheap.png %}

And that's it! Start your rails server along with ngrok or a localtunnel. Give it a go send an email to the address you created in the domain settings.
In my case this would be info@shivamdaryanani.com and boom you should have received a post message from Sendgrid. You would want to put a debugger inside the method to play around with email object that Griddler gives you.
The SendGrid API expects a 200 response from your endpoint. If it doesn't get it or if your site is down, it tries up to 3 times in set intervals until it reaches your app. Which is great if your site is under maintenance.

### Why would I want to receive email?

That's a good question. A lot of sites create their support tickets in this way. Once you email support@domain.com, it parses the email and creates the ticket. Also here is a list of creative ideas:

- [The Birdy][5]
- [Nudge Mail][6]
- [IDoneThis][7]

Hope this was useful let me know if you come across any roadblocks.

[1]: http://sendgrid.com/docs/API_Reference/Webhooks/parse.html
[2]: http://github.com/thoughtbot/griddler
[3]: http://ngrok.com/ 
[4]: http://namecheap.com
[5]: https://thebirdy.com/
[6]: http://www.nudgemail.com/
[7]: https://idonethis.com/
[8]: http://sendgrid.com/developer/reply

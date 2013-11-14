---
layout: post
title: "Creating Email Accounts on Your Domain Dynamically"
date: 2013-11-14 09:40
comments: true
categories: ["rails", "dns", "domain"]
---

For an app I am working on, I needed to create an email account on my domain based on the user's input. For example the user signed up as 'johndoe', I had to create an email for them johndoe@mydomain.com. Initially I thought this would be difficult, but it turned out to be simple thanks to [DNSimple][1]. 

### Attempt#1

After searching stackoverflow for a few hours, I posted a [question][3] on asking how can this be done. Thanks to [Ludovic][2] who mentioned I can use the API of my host service. I didn't know it would be that easy and luckily my host service, [Namecheap][4], has an [API][5]. I signed up for a key and within 24 hours got access to it. I was able to [set up email forwarding][6], but due to the API being in beta stage there are a few caveats. First the API is only available during specific hours, which wasn't too bad. However the API was expected to be used through a set IP address and this was a deal breaker. In order to get this up on Heroku meant using the [Proximo][7] addon which is too expensive for my budget. I scraped this and starting looking for an easier way. 

### DNSimple To The Rescue

I had heard a lot of about DNSimple and their ease on managing domain settings. This topic always confuses me, but their brilliant UI and well documented options make it very easy to use. The best part was they have a full featured [API][8] which is exactly what I needed. DNSimple comes with a 30day trial, but I signed up for their bronze plan. The request needed to get all email forwards for you app:

{% codeblock lang:bash %}
curl  -H 'X-DNSimple-Token: youraddress@gmail.com:APIToken' \
      -H 'Accept: application/json' \
      https://dnsimple.com/domains/example.com/email_forwards
{% endcodeblock %}

This is a get request with the headers notifying that you want a JSON response and a header specifying your email and API token. You can obtain your API token from the [account page][9], don't confuse this with the domain key that DNSimple provides. 
In order to create an email forward, you will need to make a post request: 
{% codeblock lang:bash %}
curl  -H 'X-DNSimple-Token: youraddress@gmail.com:APIToken' \
      -H 'Accept: application/json' \
      -H 'Content-Type: application/json' \
      -X POST \
      -d '{
          "email_forward": {
            "from": "john",
            "to": "someone@example.com"
          }
      }' \
      https://dnsimple.com/domains/example.com/email_forward
{% endcodeblock %}
This time we have an additional header, setting the content-type to be JSON as we will be sending JSON. We also need to specify a post request, which is indicated by -X. Finally we have to send the data with -d. All emails that get sent to john@example.com will be forwarded to someone@example.com.

You can send a post request with headers in a Rails app using the ```system``` method(you'll paste in the same script)  but I prefer to make a post request using the [HTTParty][10] gem as it's cleaner. The same post request in ruby: 

{% codeblock lang:ruby %}
require 'httparty'
url = "https://dnsimple.com/domains/example.com/email_forwards"
HTTParty.post(url, 
              body: { "email_forward" => 
                           { 
                             "from" => "john",
                              "to" => "someone@example.com" 
                           } 
                    }.to_json, 
              headers: {
                        'X-DNSimple-Token' => 'youraddress@gmail.com:APIToken',
                        'Accept' => 'application/json',
                        'Content-Type' => 'application/json',
                       }
             )
{% endcodeblock %} 

That will create the same result as the script above,  don't forget to call ```to_json``` on the body.

[1]: http://dnsimple.com
[3]: http://stackoverflow.com/questions/19866040/how-to-create-an-email-address-on-your-domain-dynamically
[2]: http://stackoverflow.com/users/1385773/ludovic
[4]: http://www.namecheap.com/
[5]: http://developer.namecheap.com/docs/
[6]: http://developer.namecheap.com/docs/doku.php?id=api-reference:domains.dns:setcustom
[7]: https://addons.heroku.com/proximo
[8]: http://developer.dnsimple.com/
[9]: https://dnsimple.com/account
[10]:https://github.com/jnunemaker/httparty


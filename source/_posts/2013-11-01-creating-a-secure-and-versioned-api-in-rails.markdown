---
layout: post
title: "Creating a Secure and Versioned API in Rails"
date: 2013-11-01 23:55
comments: true
categories: ["code", "rails", "api"]
---

Been working on a side project with a friend and we were at the point where we needed to build an API for the code we had written. The idea was to build an API which can be accessed by our customers with an API key. 

After some research, I found couple  of gems that make it easy to do this. [Rocket Pants][1] and [Versionist][2]. In order to reduce the dependencies, I decided to build it myself with the help of [RailsCasts][3]. 

### Creating the API. 

We will start off by creating the routes for it. 

{% codeblock lang:ruby config/routes.rb %}

namespace :api do 
  namespace :v1 do 
     namespace :search do 
       get "youtube"
     end
  end
end

{% endcodeblock %}

This block of code gives us the following route: 
```
/api/v1/search/youtube
```


Now in order to create the controller, since they are namespaced under api and the version number you must put the controller inside the two modules. 


{% codeblock lang:ruby app/controllers/api/v1/search_controller.rb%}
module Api
  module V1
    class SearchController < ApplicationController
      
      def youtube
        #code that searches for youtube videos
      end

    end
  end
end
{% endcodeblock %}

Now doing the following should give you a list of YouTube results in a JSON format. 

{% codeblock lang:bash %}
curl yoururl.com/api/v1/search/youtube?query=funny
{% endcodeblock %}

This setup gives us the ability to easily add new versions of the API while maintaining backwards compatibility. The only thing is that the API is not secure. 

### Securing the API. 

There are a few ways you can do this. Instead of doing it with OAuth I just created an access token given to each application created. The access token is created using a SecureRandom.hex string. 


{% codeblock lang:ruby app/models/user.rb %}
class User < ActiveRecord::Base 
  #code for authentication 

  has_many :applications
end
{% endcodeblock %}

{% codeblock lang:ruby app/models/application.rb %}

class Application < ActiveRecord::Base
  before_create :generate_access_token

  belongs_to :user

  private

  def generate_access_token
    begin 
      self.access_token = SecureRandom.hex
    end while self.class.exists?(access_token: access_token)
  end
end

{% endcodeblock %}

The way I set this up is that a user can have many applications, and each application belongs to a user. Before an application is created there is a unique key which is generated restricing access to the API. 
There are other columns you can add to the model if needed such as permissions, expires_at etc.

{% codeblock lang:ruby app/controllers/api/v1/search_controller.rb%}
module Api
  module V1
    class SearchController < ApplicationController
      before_filter :restrict_access
      
      def youtube
        #code that searches for youtube videos
      end

      private 

      def restrict_access
        app = Application.find_by_access_token(params[:access_token])
        head :unauthorized unless app
      end

    end
  end
end
{% endcodeblock %}

Updating the controller in order to send an unauthorized response if the access key is invalid. Now in order to make a request the user has to send in the right access_token in the parameters. Also make sure to not make the access_token field accessible using attr_accessible or in rails 4 you can do the following: 

{% codeblock lang:ruby app/controllers/applications.rb%}

class ApplicationsController < ApplicationController
  #code for controller
  
  def create 
    @application = current_user.applications.build(application_params)
    #code to save application
  end

  private 

  def application_params
    params.require(:application).permit(:name)
  end

end
{% endcodeblock %}

Instead of directly passing in the params[:application] we pass the build method a method itself. This private method makes sure to only permit the application name.

Since the output of my controllers were already in JSON there was no need for [RABL][4]. Also for those looking for something lightweight can have a look at [Rails-API][5]. This is enough to get you started if you any questions leave comments and I shall address them.

[1]: https://github.com/filtersquad/rocket_pants
[2]: https://github.com/bploetz/versionist
[3]: https://railscasts.com
[4]: https://github.com/nesquena/rabl
[5]: https://github.com/rails-api/rails-api

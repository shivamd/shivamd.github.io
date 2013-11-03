---
layout: post
title: "Nested Attributes in Rails 4"
date: 2013-11-03 23:24
comments: true
categories: ["code", "rails", "forms"]
---

Often when creating a web application there comes a time where two models are related and you want to create one form where you can add both the attributes. Let's say you have a topic model which has many questions, and you want to create the topic and questions for it in one form. Rails makes this easy with the accepts_nested_attributes_for method. 

### Models 

{% codeblock lang:ruby app/models/topic.rb %}
class Topic < ActiveRecord::Base 
  has_many :questions
end
{% endcodeblock %}

{% codeblock lang:ruby app/models/question.rb %}
class Question < ActiveRecord::Base 
  belongs_to :topic
end
{% endcodeblock %}

In order for the topic form to be able to add questions, we need the following line:
{% codeblock lang:ruby app/models/topic.rb %}
class Topic < ActiveRecord::Base 
  has_many :questions
  accepts_nested_attributes_for :questions, allow_destroy: true
end
{% endcodeblock %}

Accepts nested attributes is just a shortcut, it defines a dynamic attribute {field_name}_attributes so you can automatically assign them to an association. The allow destroy lets you destroy question objects in this case through the form, it is set to false from the start. There are other methods you can use with this, such as reject_if, limit and  update_only. More information can be found from the [source][1].

### Form

{% codeblock lang:ruby %}
<%= form_for @topic do |f| %>

  <div id="name" class="field">
    <%= f.text_field :name, placeholder: "Name" %>
  </div>

  <div id="questions" class="field">
    <%= f.fields_for :questions do |builder| %>
      <div class="question">
        <%= builder.text_field :content, placeholder: "Question" %>
      </div>
    <% end %>
  </div>
  
  <%= f.submit "Create topic" %>

<% end %>
{% endcodeblock %}
This creates a simple form which allows you to create a topic with a name and add one question to it.

### Controller & Routes
The controller method for create is your regular create method, nothing fancy. That's why I love Rails.

{% codeblock lang:ruby app/controllers/topics_controller.rb %}
class TopicsController < ApplicationController

  def new 
    @topic = Topic.new
    @question = @topic.questions.build
  end

  def create
    @topic = Topic.new(topic_params)
    unless @topic.save 
      render :new
    else
      redirect_to root_path, notice: "Successfully created a topic"
    end
  end

  private 

  def topic_params
    params.require(:topic).permit(:name, questions_attributes: [:content])
  end
end

{% endcodeblock %}

The new method creates the instances for both topic and questions. In the create action we instantiate a topic passing in a method. This is a private method needed in Rails 4 to make sure only the allowed attributes are used. We require the topic parameters and permit the name for a topic and the questions_attributes. Now all we need are the routes and this will work.

{% codeblock lang:ruby config/routes.rb %}
  resources :topics
{% endcodeblock %}

Now we have a simple version of adding questions to a topic, but we still don't have validations.

### Validations

It is good practice to have your validations on both the database level and on the Rails model's itself. The validation for the association is not as straightforward as it seems.  A common approach would look like this. 


{% codeblock lang:ruby app/models/topic.rb %}
class Topic < ActiveRecord::Base 
  has_many :questions
  accepts_nested_attributes_for :questions, allow_destroy: true

  validates :name, presence: true
end
{% endcodeblock %}

{% codeblock lang:ruby app/models/question.rb %}
class Question < ActiveRecord::Base 
  belongs_to :topic
  validates :content, :topic_id,  presence: true
end
{% endcodeblock %}

These are simple validations that would normally pass but they won't because of the nested forms. You need to tell Rails that the two model relations describe the same relationship but from the opposite direction. Rails provides a method for this, inverse_of. Rails has a method for everything. 

{% codeblock lang:ruby app/models/topic.rb %}
class Topic < ActiveRecord::Base 
  has_many :questions, inverse_of :topic
  accepts_nested_attributes_for :questions, allow_destroy: true

  validates :name, presence: true
end
{% endcodeblock %}

{% codeblock lang:ruby app/models/question.rb %}
class Question < ActiveRecord::Base 
  belongs_to :topic, inverse_of :questions
  validates :content, :topic,  presence: true
end
{% endcodeblock %}

As you can see we indicated the inverse of relation from both sides. Also now the Question model validates the topic itself instead of the ID.

There you have it, a form that accepts nested attributes in Rails 4 with validations.


[1]: http://apidock.com/rails/ActiveRecord/NestedAttributes/ClassMethods/accepts_nested_attributes_for

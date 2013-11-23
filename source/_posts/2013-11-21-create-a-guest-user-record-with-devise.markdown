---
layout: post
title: "Creating a Guest User Record with Devise"
date: 2013-11-21 02:04
comments: true
categories: ["devise", "rails"]
---

At times you need the user to check out your application without having to sign up. Play around with the features and if they are interested they can sign up. It's easy to create a guest record on visit of the page, but the difficult part is in transferring the data from the guest to the signed up user. 

### Setup

Assuming you have a default devise setup on a user, you need to override the ```current_user``` method devise provides out of the box.

First we'll need to add a guest column to our user model.
{% codeblock %}
rails g migration AddGuestToUser guest:boolean
{% endcodeblock %}

{% codeblock lang:ruby %}
class AddGuestToUser < ActiveRecord::Migration
  def change
    add_column :users, :guest, :boolean
  end
end
{% endcodeblock %}

### Overwrite the CurrentUser Method

{% codeblock lang:ruby app/controllers/application_contorller.rb %}
class ApplicationController < ActionController::Base
  protect_from_forgery
  before_filter :authenticate_user!
  helper_method :current_user

  def current_user 
    super || guest_user
  end

  private

  def guest_user
    User.find(session[:guest_user_id].nil? ? session[:guest_user_id] = create_guest_user.id : session[:guest_user_id])
  end

  def create_guest_user
    user = User.new { |user| user.guest = true }
    user.email = "guest_#{Time.now.to_i}#{rand(99)}@example.com"
    user.save(:validate => false)
    user
  end
end
{% endcodeblock %}

Since we use the ```current_user``` method in our application, we default to normal behaviour. If there's none we create a guest_user. The logic is simple,
if the ```session[:guest_user_id]``` is nil we create one otherwise we use that id itself.
The ```create_guest_user``` method instantiates a user and within a block sets the guest attribute to true. The reason we do that is because guest is not accessible as an attribute and setting it normally would throw a mass assignment error.
You probably have some logic in the navigation bar when going to the homepage, hence the guest user record gets created whenever you make your first reference to current user.

### Transferring data to signed up user.
Let's say the user finds your application beneficial and decides to sign up. Here is the code needed to transfer the information.

{% codeblock lang:ruby app/controllers/registrations_controller.rb %}
class RegistrationsController < Devise::RegistrationsController
  def new
    super
  end

  def create
    @user = User.new(params[:user])
    if @user.save
      current_user.move_to(@user) if current_user && current_user.guest?
      sign_up("user", @user)
      redirect_to user_root_path
    else
      render :new
    end
  end
end
{% endcodeblock %}

We want the new action to be the same, hence we call ```super```. However we are overriding the create action. After saving the user we call move to on the ```current_user``` if they are a guest. ```move_to``` is a method defined on the user model. 

{% codeblock lang:ruby app/models/user.rb %}
class User < ActiveRecord::Base
 #user logic 

 def move_to(user)
  todos.update_all(user_id: user.id)
 end

 #more user logic
end
{% endcodeblock %}

In this case we move all the todos to the user who has just signed up. Once that is done we sign_up the user using devise's built in ```sign_up``` method. 
You might be thinking about deleting the guest user right now, but you should not in case you have relations dependent to it. You best bet would be to create a cron that deletes the guest user's every 24 hours.

{% codeblock lang:ruby lib/tasks/scheduler.rake %}
desc "Remove guest accounts more than a week old."
task :guest_cleanup => :environment do
  User.where(guest: :true).destroy_all
end
{% endcodeblock %}

And there it is, guest users who can interact with your app.

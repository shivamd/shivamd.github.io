---
layout: post
title: "Harnessing the Power of Ctags in Vim"
date: 2013-11-16 22:51
comments: true
categories: ["code", "vim"]
---
What is Ctags? It's a tool that creates an index file of functions, variables, class names and so on based on the language. This makes it easy to navigate a code base, and makes you feel like a badass.

At times you come across a method and have no idea where in the source code it is. Imagine with a press of a command it takes you there immediately. You are writing Rails code and want to look at the documentation for let's say ```validates```, instead of doing a google search just use Ctags! You no longer have to get scared of a large code base.

### The setup.

Ctags come shipped in with MacOS but you'll want the exuberant Ctags. 

{% codeblock lang:bash %}
brew install ctags
{% endcodeblock %}

If you did the installation right, you should get the following output when running ```ctags --version```

{% codeblock %}
Exuberant Ctags 5.8, Copyright (C) 1996-2009 Darren Hiebert
Compiled: Nov 15 2013, 00:46:27  Addresses: <dhiebert@users.sourceforge.net>,
http://ctags.sourceforge.net  Optional compiled features: +wildcards, +regex
{% endcodeblock %}

### Indexing a directory 
Now that we have Ctags installed, go to the root file of your code base and run the following command:
{% codeblock lang:bash %}
ctags -R
{% endcodeblock %}

The ```-R``` is to recurse through the subdirectories. Now there should be a tags file created in the directory. If you are using Git for version control, I recommend putting this in your global gitignore. 

### Indexing your Ruby Gems.
This is best done through the ruby gem ```gem-ctags```

{% codeblock lang:bash %}
gem install ctags
{% endcodeblock %}

And then run: 
{% codeblock lang:bash %}
gem ctags
{% endcodeblock %}
This will index all the gems and you won't have to ever do this again. From now on whenever you install a gem, it will get automatically indexed.

### Auto Indexing.
Despite the awesomeness of Ctags you might be thinking you have to always manually index a directory. Whenever you pull code from Github, merge in branches etc. This issue has been solved by [the great Tim Pope][1]. The article explains how to setup auto indexing using git hooks. There is one point that I missed out, he says to make the files executable. To do that: 

{% codeblock lang:bash %}
chmod +x file_name
{% endcodeblock %}
You must do this for all the files as he states.
Now any time you pull, merge or commit the tags file will get updated. 

However this won't work yet for existing repositories, you must reinitialize them with ```git init``` followed by ```git ctags```
### Navigating. 
With all that out of the way, on to the fun stuff. Open a file place your cursor on a word. Let's say you're in the post class and you see the word User. Navigating to that word and pressing <C-]>"control and close bracket", will jump you to the User class. You can keep diving deep into the code base with this. At times you can go a bit too deep,the good news is that Vim keeps a history of things. Hit ```:tags``` to access that. 

{% codeblock lang:bash %}
 # TO tag         FROM line  in file/text
 1  1 User                6  ~/programming/contractjobs/teen_w/app/models/crons/daily_alerts_email.rb
 2  1 devise              6  app/models/user.rb
 3  1 attr_accessible    10  app/models/user.rb
{% endcodeblock %}

This is my current list, you can move back and forth between this by <C-o>"control O" and <C-i>"control i". As you can see I have ```attr_accessible``` in my list, this is a method in the rails framework. The one thing I absolutely love about Ctags is that it makes navigating your gems' source code very easy. My rails knowledge has increased really fast thanks to this. As the say, the best way to improve as a developer is too read other people's code. 

You can prefix the ctag command with a ```g```  to get a full list of matches to choose from. 
This should be enough to get you started there are a lot of other commands ```help ctags``` to find out more. 
[For Sublime Text users][2].

[1]: http://tbaggery.com/2011/08/08/effortless-ctags-with-git.html
[2]: https://github.com/SublimeText/CTags

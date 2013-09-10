---
layout: post
title: "How many lines of code is your project?"
date: 2013-09-11 00:34
comments: true
categories: ["code", "bash"]
---

A client just asked me, how many lines of code is the project?
Despite being an absurd question, and having no relation with the project at all this made me really curious to find out.
After a quick web search, I stumbled upon this piece of code (Bare in find this is for a rails app).
{% codeblock %}
  find ./app -type f | xargs cat | wc -l
{% endcodeblock %}
What does this code do?
{% codeblock %} find {% endcodeblock %} is a shell method that searches through the files.
The next part, {% codeblock %} ./app -type f {% endcodeblock %} searches the app directory. You can search any directory you want, this looks at the app directory which in a rails app holds majority of the code you write.
After that you list the type you want. If you search without the type you get both directories and files. In this case we just want files, hence the "f". This is an alias for "file".
{% codeblock %} | xargs {% endcodeblock %}
We need to use xargs over here which reads the STDIN stream data and coverts each line into space separated arguments to the command.
{% codeblock %} cat | {% endcodeblock %}
reads the output and pipes it out to the next step of commands
{% codeblock %} wc -l {% endcodeblock %}
"wc" is short for word count. This gives 3 results. The first being the number of new lines, the second being the number of words and last the number of characters. However we are interested in the lines, hence the command "-l".
This is a cool little script to get an overview of the size of the codebase.

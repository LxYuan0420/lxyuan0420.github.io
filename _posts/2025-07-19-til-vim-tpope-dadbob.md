---
layout: post  
title: "TIL: Exploring Databases in Vim with vim-dadbod"  
description: "Work with SQLite directly inside Vim with no context switching"  
---

Work has slowed down a bit recently, and it feels like a good time to kinda
wander around and try new stuff. Today I learned about `tpope/vim-dadbod` and
it turned out to be a surprisingly handy tool.

It's a Vim plugin that lets you connect to databases and run SQL queries
directly inside your editor. No GUI. No switching away from
terminal.

### Connecting to SQLite

I tested it with a local SQLite file. To connect, just run:

{% highlight bash %}
:DB sqlite:///Users/exampleuser/db/dev.sqlite3
{% endhighlight %}

That opens the connection. Then you can browse tables using:

{% highlight bash %}
:DBUI
{% endhighlight %}

It gives you a sidebar with tables and schemas. Use `:DBUIClose` to exit when you're done.

### Query Workflow

What I like most is pressing `S` on a table to open a dedicated query split. You can write SQL there and run it by visually selecting the query and pressing:

{% highlight bash %}
\S  " leader + S
{% endhighlight %}

The result shows up automatically in a split at the bottom. No need to leave the editor or context switch.

Example query:

{% highlight bash %}
SELECT * FROM users WHERE name LIKE '%Alex%' ORDER BY id DESC LIMIT 5;
{% endhighlight %}

### Bonus: SQL from Comments

Sometimes I jot down a messy comment like:

{% highlight bash %}
select all columns from users table where name like %Alex% and order by id desc limit 5
{% endhighlight %}

Then I visually select it and ask an LLM to rewrite it into valid SQL. It gives back the correct command. I paste it, run it, and move on.

### Final Thoughts

It works smoothly with my existing Vim setup including plugins like:

* `tpope/vim-dadbod`
* `kristijanhusak/vim-dadbod-ui`
* `kristijanhusak/vim-dadbod-completion`

Querying a local SQLite file from inside Vim feels seamless and fits perfectly into a terminal-based workflow.

If you're in Vim most of the day anyway, this just makes sense.

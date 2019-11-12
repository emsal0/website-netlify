---
title: "First post!"
date: 2017-02-27T17:17:13:19-08:00
draft: false
---

<font color="red">
**This post reflects the technology used in an earlier version of this website.**
</font>

<p>Hey, it's the first post!</p>
<p>Over the reading break, I had the opportunity to patch together a blog for this website. I hadn't done straight-up full-stack web dev in a while, so it was nice to just bang out a full application with nice functionality and a decent interface. I've wanted to make my own full website for myself since I was about 7 years old, so I guess that's also some sort of an achievement!</p>
<p>I was inspired to make this blog separate from any of my social media and other external accounts because I realized that you can only depend so much on external services; the life of this website won't be tied to the life of any of those. <a href="https://medium.com/matter/the-web-we-have-to-save-2eb1fe15a426">Hossein Derakhshan's post</a> about the state of internet content solidified these sentiments for me; I hope that someday soon, more people will start their own blogs again.</p>
<p>More technical details about how I built this thing are available below; if you're less interested in that, then welcome!</p>
<!--more-->
<p>The whole thing is made with Flask, which I really enjoyed working with. Its simplicity gives you a lot of freedom to modify the behaviour of an app while understanding the flow of your code's execution, and its rich library support allows you to skip the task of doing the nitty-gritty work of implementing really common features like an admin interface.</p>
<p>My tech stack is essentially some of the standard stuff you'd see in a Python web app:</p>
<ul>
<li><code>Flask</code> with <code>jinja2</code></li>
<li><code>SQLAlchemy</code> as an ORM</li>
<li><code>alembic</code> for migrations</li>
<li><code>Flask-Admin</code> for the admin interface</li>
<li><code>Flask-Login</code> for access control</li>
</ul>
<p>I have it all set up in a private GitHub repo, but I managed to go about coding the thing without revealing any compromising information, so maybe I can eventually work toward open-sourcing something so people can set up their own blogs really easily! This website does need a lot of improvement before I can do that, though. Here's hoping!</p>


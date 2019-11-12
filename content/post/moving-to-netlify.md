---
title: "Moving to Hugo and Netlify"
date: 2019-11-12T13:26:46-08:00
draft: false
---

I just moved this website over from my makeshift homemade setup on my self-hosted Digital Ocean box to a more convenient stack. See <a href="{{< ref path="/post/firstpost.md" >}}">the very first post</a> here to see what the old stack looked like.

I'm using [Hugo](https://gohugo.io) with the [whiteplain theme](https://github.com/taikii/whiteplain), keeping some of the simplicity of the old design. I'm currently hosting on [Netlify](https://www.netlify.com)

It takes a nontrivial amount of work to migrate a website across setups like this. Hopefully, my documenting the tasks I had to get through would be helpful for others wanting to do something similar.

* Migrating each post and reformatting to make sure it works in the Hugo markdown engine
* Aliasing, to make sure links in the old URL scheme redirects to the same posts. I paid more attention to this because a reference to the Twitter clustering post appeared in a book, [Practical Web Scraping for Data Science: Best Practices and Examples with Python](https://books.google.ca/books?id=rkBWDwAAQBAJ&pg=PA6&lpg=PA6&dq=emsal.me&source=bl&ots=6-2bKpr5Jc&sig=ACfU3U3olhqvsG9FUxwIQAtPTrz_FxjAcA&hl=en&sa=X&ved=2ahUKEwjNqabGz-XlAhV1CTQIHUq-DQ8Q6AEwBnoECAkQAQ#v=onepage&q=emsal.me&f=false) by Seppe vanden Broucke and Bart Baesens.
* Mucking around with DNS
* Migrating the LetsEncrypt certificates

I'd say the work paid off, though. I no longer have to manage my own Postgres instance and an admin interface to store my blog posts: everything's managed through Git now. Hugo also makes it a lot easier to do things like reference other posts, tag my posts, arrange them into categories, and more things that I'm probably forgetting.

Netlify also has a great service for hosting static websites and I'm honestly interested with just playing around with it more.

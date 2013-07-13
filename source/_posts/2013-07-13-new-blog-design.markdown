---
layout: post
title: "New blog design"
date: 2013-07-13 18:10
comments: true
categories: blog, reflexive
---

I seem to have spent quite a long time of recent attempting to redesign various
blog pages, and it made me realise that I had always planned to put some effort
into the design of my own blog.

<!-- more -->

This blog runs on the rather excellent [octopress](http://octopress.org), which
has a rather nice standard blog theme. However, its success has meant that more
and more sites on the web have the same look. This isn't ideal if you're
attempting to make your blog even slightly memorable.

Since I last investigated changing the theme, a rather wonderful website
[opthemes](http://opthemes.com/) has appeared with a good selection of themes
you can just drop into your octopress site.

I particularly liked Aron Cedercrantz’s [BlogTheme](https://github.com/rastersize/BlogTheme),
so decided to fork it, and apply some of my own customisations.

## Installing an octopress theme

Installing a theme in octopress is simple. Firstly add it as a submodule:

{% codeblock %}
git submodule add https://github.com/sammyd/BlogTheme.git .themes/BlogTheme
git submodule update --init
{% endcodeblock %}

And then you need to install the theme in the source:

{% codeblock %}
bundle exec rake install\[BlogTheme\]
{% endcodeblock %}

## Customising a theme

Once you have the theme submodule installed, you can then fiddle around with the
theme's source in `.themes/BlogTheme`, running the rake task each time you want
to pull the changes over to the blog source. Remember you'll have to regenerate
the blog after doing this, or as I prefer, have the watch task running:

{% codeblock %}
bundle exec rake watch
{% endcodeblock %}

## Design

Design isn't my speciality. I've attempted to choose fonts and colours which are
coherent and easy to read. In theory anybody that comes across this blog will be
here for the content, not the design.

Hope you appreciate it - let me know in the comments or on twitter.

If you've enjoyed this post why not follow me on twitter -
[@iwantmyrealname](https://twitter.com/iwantmyrealname).

sam
x
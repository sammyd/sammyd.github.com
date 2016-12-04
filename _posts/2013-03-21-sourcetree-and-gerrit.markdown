---
layout: post
title: "SourceTree and Gerrit"
date: 2013-03-21 08:16
comments: true
categories: [git, tips]
---

{% img center /images/2013-03-21-gerrit-sourcetree.png %}

[SourceTree](http://sourcetreeapp.com/) is a free GUI for git and mercurial
produced by the people at [Atlassian](http://www.atlassian.com/) and I love it.
It's brilliant at loads of things, but I particularly find it useful for repo
visualisation.



One of the cool features of SourceTree is that you can add arbitrary commit text
replacements. For example, if you use Jira as a project tracking tool, SourceTree
can find references to issues and turn them into links:

{% img center /images/2013-03-21-jira-commit-message.png %}

<!-- more -->

## Jira Links

Setting this up is really simple:

1. In SourceTree, when viewing your repo click the Settings button on the right
hand side of the tool bar.
2. Then click Advanced.
3. You can then Add a Commit Text Replacement. Set the two required fields:

{% img center /images/2013-03-21-jira-link.png %}


It really is that easy.


## Links to Gerrit Change Ids

It's also possible to add custom regex-based text replacements. For example, if
you use [gerrit](https://code.google.com/p/gerrit/) (a brilliant open source
git-based code review tool) then commit messages can have change-ids added. These
allow for multiple changesets be linked together for the same commit - an essential
part of the code review process.

Gerrit is a web-based tool, with each changset having its own URL. Using the
SourceTree regex text replacement, we can change the change-ids present in the
commit messages into links:

{% img center /images/2013-03-21-gerrit-commit-message.png %}

To set this up, we add a new Commit Text Replacement (as with the aforementioned
Jira link), but select the type to be other.

We use the following regex pattern, which picks out the 40 character hex-hash,
prefixed with a capital I:

{% codeblock Regex Pattern: %}
(I[a-f0-9]{40})
{% endcodeblock %}

And then the replacement is a correctly constructed HTML link:

{% codeblock Replace With: lang:html %}
<a href="https://ourdevserver.com/gerrit/#/q/$1,n,z">$1</a>
{% endcodeblock %}

This uses the gerrit query URL to search for the requested gerrit ID.

I find this tip really useful to see who reviewed commits which have been through
gerrit, and to add reviewers to commits I have just pushed. Hope it helps somebody
else as well.

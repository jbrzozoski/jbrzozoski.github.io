---
layout: post
title:  "Why can't git show just annotated tags?"
date:   2022-01-06 00:01:00 -0500
categories: computers
---
I use git a lot. Both for work and for home stuff. At one job, a tyrannical devops lead forced us all to install his git aliases, which were numerous and absurd. Except one that I fell in love with:

```
alias.lg=log --graph --pretty=format:"%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset" --abbrev-commit --date=relative
```

Seriously, go put that in your git config now and thank me later.

I now use `git lg --all` constantly to see what's going on in my repos and upstream.

Tonight I decided I wanted something kinda prettified like that to let me look at annotated release tags.

Well, it turns out the `git tag` command doesn't have a way to say "annotated only". It always includes simple tags.

The closest command which can tell the difference is `git for-each-ref --format="%(objecttype)" refs/tags`.

So, I started to put something together, and got slightly out of hand, and ended up with a script I named [git-lt](https://gist.github.com/jbrzozoski/c8db1375a8a4aaf4cc8a9d94a4bc8f88).

I just put a copy of that in my local path. I hope it finds a lot of use.

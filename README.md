About
=====

This repository contains the sources for CAF's Dev Blog.


Yekyll Setup for Local Testing
==============================

You will need to install ruby, nodjs, rdiscount, and yekyll:

```
$PKG_MGR install ruby nodejs
gem install jekyll rdiscount
```

To browse the blog locally on port 4000, run:

```
jekyll serve --watch
```


Create a New Blog Post
======================

To add a new blog post, you will need to create a new file with current date
and post title as file name (all lowercase and using `-` instead of
whitespaces). The new file must contain a Yekyll header followed by Markdown
formatted text.  To get the boilerplate code out of the way, you can use the
file `template.md`:

```
cp template.md _posts/$(date +%F)-awesome-news.md
vim $(date +%F)-awesome-news.md
```

Fill out the `title`, `author`, and `tags` fields accordingly. In case you are
not registered as author yet, add your author short name (as key), display
name, email, and GitHub account name to the file `_config.yml`.

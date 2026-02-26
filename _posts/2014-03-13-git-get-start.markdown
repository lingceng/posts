---
layout: post
title: "Git Get Started"
date: 2014-03-13 07:21
comments: true
categories: [tool, git]
---
Config name and email

```bash
git config --global user.name "John Doe"
git config --global user.email "johndoe@example.com"
```

Get help

```bash
git config --help
git help config
man git-config
```

Start up

```bash
git init
git add .
git commit -m "Depot Scaffold"

add and commit
git commit -a -m "Depot Scaffold"
```

Compare

```bash
git diff HEAD^ HEAD

# show diff stat
git diff --stat

# show the tree-like view
git log --graph --oneline --all
```

Specify the file path

```bash
git diff HEAD^ HEAD app/models/product.rb
```

Git ammend the last commit

```bash
git commit -amend
```

Powerful edit commit command

```bash
git rebase --interactive HEAD^5
```

Git include delelted files

```bash
git add -A
```

Creates a remote named "origin" pointing at your GitHub repository

```bash
git remote add origin https://github.com/username/Hello-World.git
```

Sends your commits in the "master" branch to GitHub

```bash
git push origin master
```

Pull down changes

```bash
git pull orgin master
```

---
title: Deploying a Gatsby project to Github Pages
date: '2019-08-07'
published: true
layout: post
tags: ['gatsby', 'github']
category: software
---

The scope of this article is small. I will be showing how to publish a Gatsby project to your personal or your organization's Github Pages site.

## The Manual Way

There are lots of options when it comes to building and deploying Gatsby projects to a number of web hosting sites, but the prevailing option for deploying to Github Pages that I've found, ```gh-pages```, has given me a surprising number of problems when it comes to deploying to my personal page (this one), so here's how to do it without scripts.

### Create the repository

1. Create a new Github repo named ```YourAccount.github.io```
2. In the root folder of your Gatsby project, run
    ```
    gatsby build
    ```
3. Initialize a new git repo in the public folder (this is the folder in which Gatsby places the build output).
    ```
    cd public
    git init
    git add .
    git commit -m "Initial Commit"
    ```
4. Add your new Github repository as the remote, and push.
    ```
    git remote add origin https://github.com/YourAccount/YourAccount.github.io.git
    git push origin master
    ```
You've done it. Navigate to ```YourAccount.github.io``` and you will see your lovely website sitting there for all the world to see.

### Push updates
When it comes time to push an update, assuming you are in the project root folder, run:
```
gatsby build
cd public
git add .
git commit -m "Updates"
git push origin master
```

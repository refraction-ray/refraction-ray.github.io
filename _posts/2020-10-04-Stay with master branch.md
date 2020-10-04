---
layout: post
title: "Stay with master branch"
date: 2020-10-04
excerpt: "How to still use master as the default branch on GitHub"
tags: [misc]
comments: true
---

* toc
{:toc}

## Introduction

There is a [discussion](https://github.com/pmmmwh/react-refresh-webpack-plugin/issues/113) on GitHub about why or why not to replace master as the default branch name. I don't want to comment on this topic, but my personal choice is to stick with master branch on both my old projects and new projects in the future. Therefore, in this very short post, I make a note on how to maintain master branch on GitHub (and possibly on git if git itself shifts the default from master to main in the future).

## GitHub

* **General setup**: go to [https://github.com/settings/repositories], under "Repository default branch", delete main and fill in master, click Update button, DONE.
* **For individual repo**:, one could first create an empty repo on GitHub which essentially has no branch information. And we just ``git push -u origin master`` as before. Just ignore the instruction given by GitHub on the repo page such as ``git branch -M main``, and go through the exact same workflow as before.
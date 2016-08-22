---
layout: post
title: Lessons learned after updating from Rails 3 to Rails 4
date: 2016-08-22 22:32
categories:
---

Recently I was busy with a task to update quite a big Rails 3 application to Rails 4. Nearly two months were spent on
it. After updating Rails and all dependent gems, the biggest problem was to fix all the tests. In this article I will
describe what lessons I have learned while working on it.

### 1. Dependencies are bad

Some gems have more bugs, some have less, but anyway almost all of them have bugs. Less gems we have - less problems.
Sometimes it is easier just to add a gem, that will help us to solve a problem we are working on, instead of creating
our own code. As a result with the gem we add dependencies, that can cause problems later.

For example, maybe because of a mistake or lack of experience, there are some gems that depend on Rails 3 and do not
have that dependency in their gemspec file. In order to find a problem I needed to dive into their code. I had that kind
of a problem with [sunspot](https://github.com/sunspot/sunspot) version 2.0.0, there is no dependency on Rails 3 in
gemspec, but it does not work with Rails 4. There are later versions, but we could not update to it because of we use
Solr 3 and version 2.1.0 already depends on Solr 4. To fix the problem I have created a branch with cherry-picks of all
commits with Rails 4 support - [https://github.com/sunspot/sunspot/pull/1](https://github.com/sevaorlov/sunspot/pull/1).
I have also asked to release version 2.0.1, but they did not want to bother with it -
[https://github.com/sunspot/sunspot](https://github.com/sunspot/sunspot/issues/777).

For most other gems it was easier, I just needed to update a gem to latest version, though, I still needed to spend time
investigating a problem.

#### Keep it simple. Do not add gems unless you really need it.
<br>

### 2. Patches are bad

We have several patches for various gems including Rails and for each patch I needed to investigate whether the problem
is already fixed or not. Some patches are in our codebase, some are implemented as a fork with additional commit.
Sometimes these patches are not even ours, we just pick them because do not want to spend time creating our own patch.
In anyway these patches rely on gems internal implementation and in order to update a gem you also need to update the
patch for it.

If you are not satisfied with some gem behaviour - try to create some workaround without touching that particular
behaviour. For example we use [Squeel gem](https://github.com/activerecord-hackery/squeel) for queries in Active Record
and there are some problems with complex queries. Instead of creating a patch we use simple Arel syntax or even plain
SQL for these queries.

#### Keep it simple. Try to create a workaround instead of a patch.
<br>

These lessons are quite obvious and were already mentioned across the internet, but still not everybody follow them.

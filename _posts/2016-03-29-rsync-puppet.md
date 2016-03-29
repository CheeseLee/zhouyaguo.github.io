---
layout: post
title: "使用rsync搭建内部puppet repo"
comments: true
categories:
- puppet
---

- `rsync --list-only rsync://yum.puppetlabs.com/packages/yum/` 查看列表
- `mkdir ~/puppet-repo && rsync -avrtH --delete rsync://yum.puppetlabs.com/packages/yum/ ~/puppet-repo` 只同步yum的

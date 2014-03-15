---
layout: post
title: "通过travis-ci同步github到SAE"
description: ""
category: 
tags: [sae travis-ci]
---

最近使用Python Django开发了一个小Web应用，部署到了Sina AppEngine(简称SAE)。SAE通过svn来部署代码，但是我自己用github来管理应用。原因是我更喜欢使用git，并且也能更好地利用github这个社区，更不用提git有[成功的Git分支模型](http://www.juvenxu.com/2010/11/28/a-successful-git-branching-model/)这样的东西。但是，这就造成了github与SAE上的代码同步的问题。如果每次代码更新都要在git与svn上提交两次，这是难以想象的。

实际上追根究底，我只想每次提交代码后能同样更新到SAE上。可SAE上代码部署却只支持svn。

这时，我想到了[travis-ci](https://travis-ci.org/)，这个github的好基友 ;)

travis-ci可以在你每次向github上的某个repository提交代码后，帮你做一些事情，通过github的hook实现。比如：运行测试代码，并通知你测试结果。

事实上，每次你得到的都是一个完整的基于openstack的Ubuntu虚拟机，你可以写脚本执行任何动作，比如，安装软件，编译并测试代码，部署代码到生产环境等。更多关于travis-ci的介绍可以参考网上的资料，或travis-ci的[官方文档](http://about.travis-ci.org/docs/).

我让travis-ci做的就是：

- 从SAE上用svn checkout出代码；
- 将我上次在github上提交的代码打成一个patch；
- 然后在svn代码库里apply这个patch；
- 用svn commit到SAE上；

如下是我的travis-ci的配置

```YAML
language: python

python:
- "2.7"

env:
global:
    - secure: "MENTH3dKpndihQ1WAu1Twtv0bUeCcRPgGTbjx6F3lpaWQ9/JCuq+SksRZZ9M\nnCmoPYHmWPRrwj5LKA2P56mWHmyyd1EO7X9MKNI5hSivW3rJIvldkHP259j1\nf5LGNB1hDlfiGbI+/YkNAj07QnSeYCMxfJNMzSM4tfgscp6TuEY="

# command to install dependencies
install:
- pip install -r requirements.txt --use-mirrors

# command to run tests
script: nosetests 

after_script:
- env
- ./sync-git-to-sae.sh holysoros@163.com server
```

`sync-git-to-sae.sh`是我写的脚本.

```bash
#!/bin/sh
##################################################
# Brief       Sync your git commitment to SAE.
# Author      LiJunjie, holysoros@gmail.com
# Usage       sync-git-to-sae.sh <username> [src dir]
# Version     0.00
# Date        11-06-16 16:09:25
##################################################

# username of SAE
username="$1"

# src_dir is the root directory of all things which
# you want to deploy to SAE svn repository.
if [ -z "$2" ]; then
    src_dir="."
else
    src_dir="$2"
fi

svn_dir=/var/tmp/svn_dir

# remove the deleted items from svn repository
svn_rm_deleted() {
    deleted_items=`svn status | grep ^! | awk '{ print $2}'`
    if ! [ -z "$deleted_items" ]; then
        svn rm "$deleted_items"
    fi
}

# add new items from svn repository
svn_add_new() {
    new_items=`svn status | grep ^? | awk '{ print $2}'`
    if ! [ -z "$new_items" ]; then
        svn add "$new_items"
    fi
}


echo "Checkout svn repository from SAE"
# SVN_PASSWD variable come from travis encrypted environment variables
svn co https://svn.sinaapp.com/holyweibo/ "$svn_dir" --username "$username" --password "$SVN_PASSWD" --no-auth-cache || exit 1


echo "Sync from git repository"

cd $src_dir
git diff --relative --no-color HEAD^..HEAD >/var/tmp/diff.patch

cd $svn_dir/1
patch -p1 < /var/tmp/diff.patch

cd $svn_dir/1
svn_rm_deleted
svn_add_new


echo "Deploy to SAE"
svn ci -m "$TRAVIS_COMMIT" --username "$username" --password "$SVN_PASSWD" --no-auth-cache || exit 1


echo "Done"
```

从SAE上checkout与commit代码时都需要提供安全密码，你肯定不希望是明文，travis-ci早就提供了很好的方法[Encryption keys](http://about.travis-ci.org/docs/user/encryption-keys/)。按照文档中的步骤做，文档说的不清楚的一点是`travis encrypt "something to encrypt"`，详细的例子是：

在本地执行

    travis encrypt "SVN_PASSWD=helloworld"

然后把输出加到`.travis.yaml`中就可以了。之后，就可以在travis-ci的脚本中引用环境变量`SVN_PASSWD`，变量的值是解密后的内容，正如我的`sync-git-to-sae.sh`中所做的。

有了这个之后，每次我再修改代码，就只需要提交到github上就可以了。travis-ci会把修改同步到SAE的svn repository。

---
layout: post
title: "Migrating from TFS + TFVC to VSTS + git"
date: 2018-7-15
comments: true
sharing: true
category: [VSTS, git]
---

It seems like everybody uses git for source control nowadays. Heck, Microsoft is using git instead of its own TFVC. Although I've been using GitHub for open source projects for years now, most of my day is still spent in TFS. I finally got the green-light to move one fairly large project from TFS with TFVC to VSTS with git, this post is about my experience during the move.

## Requirements
* Migrate one repository with few branches from TFS 2017 to VSTS
* Permanent move; no need to keep TFS and VSTS in sync after conversion
* Migrate history for all branches being moved

The first two requirements are quite easy to achieve but the last one proved to be tricky. Luckily, we have [git-tfs](https://github.com/git-tfs/git-tfs/).

## git-tfs
git-tfs is a two-way bridge between TFS and git. It essentially converts changesets to commits and vice-versa. For the purpose of this article, I'm going to focus on TFS > git direction only.

## Preparation
* [Install git](https://git-scm.com/downloads)
* Install a minimal version of Visual Studio 2015. [VS2017 is not supported](https://github.com/git-tfs/git-tfs/issues/1054) by git-tfs yet.
* Install git-tfs. You can use Chocolatey or download the binaries from the [releases page](https://github.com/git-tfs/git-tfs/releases)
* As of writing of this post, version v0.29.0 is the current version. Note that I was not able to use this version as it was failing to load the TFS assemblies from my installation. I had to use v0.28.0.
* Since I downloaded the binaries and dropped them in an arbitrary folder, I had to add git-tfs folder to my path variable.

<pre class="brush: ps">
set PATH=%PATH%;c:\lpains\git-tfs
</pre>

* In VSTS, [create a team project](https://docs.microsoft.com/en-us/vsts/organizations/accounts/create-team-project?view=vsts). Make sure to select git as the source control protocol.
* [Create a new git repository](https://docs.microsoft.com/en-us/vsts/git/create-new-repo?view=vsts) in your project and save the repository URL for later use.

## Execution

"Clone" your parent branch from TFS. This will create a git repository with all commits and correct branch and relations from TFS. You may want to provide a .gitignore or a user map file to further optimize the clone to your needs. For more options, check [clone command documentation](https://github.com/git-tfs/git-tfs/blob/master/doc/commands/clone.md).

<pre class="brush: ps">
mkdir c:\lpains\Project
cd c:\lpains\Project
git tfs clone https://tfs/tfs/DefaultCollection $/Project/Path/To/ParentBranch --branches=all
</pre>

This will take a while so...

![Coffe time]({{ site.url }}/images/posts/coffee-business-cat.jpg)

git-tfs suggests cleaning git-tfs metadata. This is optional but will leave you with a cleaner repository.

<pre class="brush: ps">
git filter-branch -f --msg-filter "sed 's/^git-tfs-id:.*$//g'" -- --all
</pre>

Delete old branches by deleting the `.git/refs/original` folder.

In order to push your repository to VSTS, you will need to add an origin reference to your newly created git repository

<pre class="brush: ps">
git remote add origin https://MySubscription.visualstudio.com/MyProject/_git/Repository
</pre>

Lastly, we push the git repository to VSTS. I had one commit in my remote repo (added readme.md file) and tried to perform a pull. That caused a `refusing to merge unrelated histories` error. In this case, I do want to merge the histories so I repeated the pull command with `--allow-unrelated-histories` flag

<pre class="brush: ps">
git pull origin --allow-unrelated-histories
git push --all origin
</pre>

## Conclusion
Git-tfs is great. Even though the project maintenance seems to have slowed considerably, I still highly recommend the tool if you are moving from TFVC to git. Feel free to post your experience in the comments section below.

Cheers!
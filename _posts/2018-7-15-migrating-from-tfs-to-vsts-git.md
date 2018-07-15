---
layout: post
title: "Migrating from TFS + TFVC to VSTS + git"
date: 2018-7-15
comments: true
sharing: true
categories: VSTS git
---

It seems like everybody uses git for source control nowadays. Heck, Microsoft is using git instead of its own TFVC. Although I've been using github for open source projects for years now, most of my day is still spent in TFS. I finally got the green-light to move one fairly large project from TFS with TFVC to VSTS with git, this post is about my experience during the move.

## Requirements
* Migrate one repository with few branches from TFS 2017 to VSTS
* Permanent move; no need to keep TFS and VSTS in sync after conversion
* Migrate history for all branches being moved

The first two requirements are quite easy to achieve but the last one proved to be tricky. Luckily, we have [git-tfs](https://github.com/git-tfs/git-tfs/).

## git-tfs
git-tfs is a two-way bridge between TFS and git. It essentially converts changesets to commits and vice-versa. For the purpose of this article, I'm going to focus on TFS > git direction only.

## Preparation
1. [Install git](https://git-scm.com/downloads)
2. Install a minimal version of Visual Studio 2015. [VS2017 is not supported](https://github.com/git-tfs/git-tfs/issues/1054) by git-tfs yet.
3. Install git-tfs. You can use Chocolatey or download the binaries from the [releases page](https://github.com/git-tfs/git-tfs/releases)
    1. As of writing of this post, version v0.29.0 is the current version. Note that I was not able to use this version as it was failing to load the TFS assemblies from my installation. I had to use v0.28.0.
    2. Since I downloaded the binaries and dropped them in an arbitrary folder, I had to add git-tfs folder to my path variable.
4. In VSTS, [create a tem project](https://docs.microsoft.com/en-us/vsts/organizations/accounts/create-team-project?view=vsts). Make sure to select git as the source control protocol.
5. [Create a new git repository](https://docs.microsoft.com/en-us/vsts/git/create-new-repo?view=vsts) in your project and save the repository url for later use.

<pre class="brush: ps">
set PATH=%PATH%;c:\lpains\git-tfs
</pre>

## Execution

1. "Clone" a repository from TFS. This will create a git repository with all commits from TFS. For more options, check [clone command documentation](https://github.com/git-tfs/git-tfs/blob/master/doc/commands/clone.md).
    1. You may want to provide a .gitignore or a user map file to further optimize the clone to your needs. Again, check the docs.

<pre class="brush: ps">
mkdir c:\lpains\Project
cd c:\lpains\Project
git tfs clone https://tfs/tfs/DefaultCollection $/Project/Path/To/ParentBranch --branches=all
</pre>

2. This will take a while so...

![Coffe time]({{ site.url }}/images/posts/coffee-business-cat.gif)

3. Clean git-tfs metadata
<pre class="brush: ps">
git filter-branch -f --msg-filter "sed 's/^git-tfs-id:.*$//g'" -- --all
</pre>

4. Delete old branchs by deleting the `.git/refs/original` folder.

5. In order to push your repository to VSTS, you will need to add a origin reference to your newly created git repository

<pre class="brush: ps">
git remote add origin https://MySubscription.visualstudio.com/MyProject/_git/Repository
</pre>

6. Lastly, we push the git repository to VSTS. 
    1. I had one commit in my remote repo (added readme.md file) and tried to perform a pull. That caused a `refusing to merge unrelated histories` error. In this case, I do want to merge the histories so I repeated the pull command with `--allow-unrelated-histories` flag

<pre class="brush: ps">
git pull origin --allow-unrelated-histories
git push --all origin
</pre>

## Build/Release
Build and Release are mostly the same as TFS so I will skip describing the steps for them.

## Conculsion
Git-tfs is great. Even though the project maintenance seems to have slowed considerably, I still highly recommend the tool if you are moving from TFVC to git. Feel free to post your experience in the comments section below.

Cheers!
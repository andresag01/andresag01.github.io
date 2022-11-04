---
layout: post
title:  "Understanding Git (Part III): Workflows"
date:   2022-11-17 11:21:09 +0100
---

{::options parse_block_html="true" /}

Git is an extremely flexible and versatile tool to effectively and reliably
version control files. However, there are a million ways to organize
projects with Git, so it is important to agree on general usage guidelines. For
example, good guidelines tell us where is the latest stable version of the
code, how to propose changes or new features, etc. People sometimes call those
guidelines a _workflow_.

A Git workflow is much like the "coding style" rules from a software
project. Git does not enforce the guidelines, just as programming languages
like C or Java do not enforce coding styles, although everyone contributing
to the project is *strongly* encouraged to follow them to ensure that Git is
used in a coherent, consistent and productive fashion.

In the remainder of this post, I will briefly describe two Git workflows. I had
to use either of these workflows in virtually all the private and open-source
projects that I contributed to. I assume that you read parts [I]({% post_url
2022-11-09-understanding-git-p1 %}) and [II]({% post_url
2022-11-11-understanding-git-p2 %}) of this series, so you know your branches,
merges, remotes, etc.

{% include understanding-git-contents.md %}

# Feature Branch Workflow

The idea is that `main` is a long-lived branch that is always kept in working
order. As a user, you can always be sure that the latest `main` will compile
and is broadly expected to work. It is also common to run regressions, such as
nightly tests, in a Continuous Integration (CI) system to sanity check and
ensure quality.

Contributors are required to propose changes via branches. A contributor's
workflow is typically like this:

1. Update your local copy of the repository with `git fetch origin`. Assume
that `origin` is the remote's name.
1. Create a branch off the latest commit in `main` and switch to that branch
with `git checkout origin/main -b my-feature-branch`.
1. Make your changes.
1. Create commits in the branch `my-feature-branch`.
1. Push your feature branch to the remote when you are happy with the changes:
`git push origin my-feature-branch`.

Other contributors are able to see your feature branch with the proposed
changes at this point. Ideally, the changes will be reviewed giving rise to
discussions. Websites, like [GitHub](https://github.com),
[BitBucket](https://bitbucket.com) and [GitLab](https://gitlab.com), have a
great feature to assist with the review process: _Pull Requests (PR)_ (GitLab
calls them _Merge Requests (MR)_!). In this context, a PR is simply a "formal"
request to merge your feature branch into another branch (typically `main`).
Contributors will be able to review your changes, add comments and request
amendments. You can upload those changes by creating more commits in
`my-feature-branch` and then pushing with `git push origin my-feature-branch`
as usual.

PRs are merged after the changes are reviewed and discussed -- and hopefully
agreed! -- by clicking a button on the web UI. When that happens, the feature
branch is merged by automatically creating a merge commit in `main` (think
`git merge` from [here]({% post_url 2022-11-09-understanding-git-p1 %})!).

As discussed before, a branch is simply a pointer to a commit and we can create
(and throw-away!) as many branches as we like. There is no need to keep
maintaining and updating feature branches -- hence why I never use `git pull`!
--, so it is desirable to delete feature branches, like
`my-feature-branch`, after the PR is merged to avoid having lots of unused
branches cluttering the repository. In fact, this is so common that web UIs
normally have an option to automatically delete the feature branch after a PR
is merged. Alternatively, you can manually delete a branch like this:

```sh
# Delete branch locally
$ git branch -D my-feature-branch
# Delete remote branch
$ git push origin --delete my-feature-branch
```

<div class="panel-info">
<div class="panel-info-title">
**Using Pull Requests (PR) and Merge Requests (MR)**
</div>
<div class="panel-info-body">
First of all, PRs and MRs are basically the same thing: GitHub and BitBucket
call them PRs and GitLab calls them MRs. You will be glad to know that these
are created and used through the graphical web UI which is fairly intuitive.
However, each website works in a slightly different way, so you need to look at
their specific documentation to see how to create and merge PRs, make comments,
approve, etc.

The key benefit of PRs is to create good spaces for reviewing and discussing
the proposed changes before these are integrated into `main`. However, reviews
can be lenghty and it is likely that your feature branch diverges from `main`
by the time everything is agreed and ready to merge. If that is the case, you
may need to incorporate the latest changes from `origin/main` into your
feature branch by either rebasing or merging.
</div>
</div>

# Forking Workflow

The feature branch workflow is great with a single remote repository and a few
contributors. However, a couple of problems pop up as the number of
contributors grows or when projects are open-source. First, each contributor
will work on at least one feature branch, and usually many more, so the number
of _active_ branches tends to grow very quickly cluttering the repository.
And second, and perhaps most importantly in open-source projects, you do not
want every contributor to have permissions to directly modify the project's
official remote repository -- what if someone deletes `main`?!

The forking workflow neatly addresses both problems. The idea is that only a
small, select, trusted (!) group of core contributors has permissions to manage
and modify the project's remote repository -- including merging PRs! Everyone
else has either no permission (if it is a private project) or read-only
permissions. Contributors with read-only permissions are able to clone the
repository (with `git clone`) and _fork_ it.

A fork is simply a copy in another remote of the project's original remote
repository. For example, lets say that I would like to contribute to the Mbed
TLS project which is hosted in GitHub
[here](https://github.com/Mbed-TLS/mbedtls), so I clone their
repository in my local computer and `origin` is the only remote:

```sh
$ git clone https://github.com/Mbed-TLS/mbedtls.git
$ git remote -v
origin  https://github.com/Mbed-TLS/mbedtls.git (fetch)
origin  https://github.com/Mbed-TLS/mbedtls.git (push)
```

I wanted to propose changes to Mbed TLS. I had read-only access to their
repository, so I forked Mbed TLS using the "fork" button in GitHub's web UI.
This creates my own copy of Mbed TLS in another remote
[here](https://github.com/andresag01/mbedtls). I then configure a second remote
in my local copy of Mbed TLS like this (notice the different URL!):

```sh
$ git remote add andresag https://github.com/andresag01/mbedtls.git
$ git fetch andresag
```

So my local repository is now aware of two remotes:

```sh
$ git remote -v
andresag        https://github.com/andresag01/mbedtls.git (fetch)
andresag        https://github.com/andresag01/mbedtls.git (push)
origin  https://github.com/Mbed-TLS/mbedtls.git (fetch)
origin  https://github.com/Mbed-TLS/mbedtls.git (push)
```

The remote `origin` is Mbed TLS's official repository that I can only read
while `andresag` is my fork which I have permission to both read and write. But
both remotes are entirely different, so commits pushed into `origin` will not
affect `andresag` and viceversa!

Back to the original task: how do we propose a change using the fork? We first
create the change commits and push them to the fork. Then we use PRs! I like
using the feature branch workflow in my forks, so the steps would be like this
in the Mbed TLS example:

1. Update local repository: `git fetch origin`
1. Create feature branch: `git checkout origin/main -b my-feature-branch`
1. Make your changes.
1. Create commits in the branch `my-feature-branch`
1. Push feature branch to my for: `git push andresag my-feature-branch`
1. Create PR using the GitHub web UI as described
[here](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/creating-a-pull-request).
[Here](https://github.com/Mbed-TLS/mbedtls/pull/2496) is an example of a PR I
created a long time ago for Mbed TLS from my fork.

The PR is then merged as usual after the review and discussion process.

# Closing Thoughts on Workflows

The general workflows described above recommend, very broadly, how to use
Git. However, using Git in a project has many more practical considerations.
Here are a few things to look out for when contributing to, creating
your own or simply using a project versioned with Git:

* Projects regularly produce official releases. The release is taken from a
commit that is usually tracked by either a Git branch or a
[tag](https://git-scm.com/book/en/v2/Git-Basics-Tagging). Ideally, you want to
take an officially released, presumably stable, version of the project if you
are a user.
* The main branch is usually called `main`, `master` or `primary`. The latest
commit in this branch is normally a stable version of the project as discussed
above. Projects sometimes have a `development` or `dev` branch too where PRs
and new features are merged using the feature branch workflow. In this case,
the development branch may be unstable and it is only merged into `main` when
an official release is produced.
* CIs normally executes long-running tests in the `main` or `dev` branch (or
both). Sometimes the CI also runs shorter sanity checks in feature branches
after a PR is created. Ideally, the PR is only merged if the CI tests pass.

# Conclusion

I described two workflows to help us use Git in a consistent and productive
way. As mentioned along the way, I had to use either of these two approaches
virtually everywhere I worked with Git. Also, I tend to use the feature branch
workflow for my own forks to keep things organized. However, I only scratched
the surface! There are many other approaches to using Git with their own
advantages and disadvantages, so have a look at the way others use Git and
learn from it!

This is the last post in this Git series (after parts [I]({% post_url
2022-11-09-understanding-git-p1 %}) and [II]({% post_url
2022-11-11-understanding-git-p2 %})). Hopefully, these articles are helpful if
you are getting started with Git or even if you have been using it for some
time but perhaps it had not fully clicked. Happy times using Git!

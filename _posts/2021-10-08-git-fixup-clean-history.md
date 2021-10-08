---
layout: post
title: Clean git history - using git fixup commits, aliases and more
comments: true
---


# Clean git history - using git fixup commits, aliases and more

## Introduction

Git has many different features that make it convenient to preserve clean version control history and collaborate with others without affecting productivity.

One very useful feature is git [--fixup](https://git-scm.com/docs/git-commit#_options) commits that have been there since forever, but not used by many as often as it should.

The main focus of this blog post will talk about some use cases and practical examples of how we use it at work and why it makes the review process much more pleasant.

Additionally, there are some other nice features that can be used in conjunction with git `--fixup` commits that I'll briefly cover as well.

As bonus, I'll also provide some examples, such as:
- Scripts that I wrote which make it more convenient to use
- Using git aliases
- Making some these commands available through Atlassian's SourceTree

## What is a git fixup commit?

A git fixup commit is a special commit that fixes another commit up the tree.

Git automatically prefixes them with `fixup!` and then with the same commit message it fixes.

Let's say we have the following tree:
```
* 05dcacd - (HEAD -> my_new_feature) F3
* 1a69fa6 - F2
* 744b43b - F1
* a308372 - (master) A3
* 0cd736a - A2
* 4fa144c - A1
* ea977df - * Initial commit
```

We submit a PR (Pull Request) with the changes made in the `my_new_feature` branch. Our dear colleague then reviewed it and commented: "Can we please rename this variable to foo?".

There are different ways to approach this request:
1. Create a new commit saying: "Rename variable per code review."
2. Rebase the commit where this change was first introduced and `git push --force`
3. Or, create a `--fixup` commit

Approaching option [1] will make our git history messy and hard to follow what this new feature branch was introducing. merge-commits solves this problem, however if this pattern continues, the git history will become too noisy and rather confusing.

Approaching option [2] sounds reasonable when working by oneself, but when working with colleagues, imagine there are multiple requests for changes within the same PR. If we `git push --force` after applying the many changes, it means our colleague will need to do another review iteration, increasing the chances of missing things or potential bugs that were introduced while rebasing.

Approaching option [3], solves options [1] and [2]. How? Incremental changes that can be automatically cleaned up prior to merging.

Let's say we want to fix the commit: `* 744b43b - F1` where the variable that was requested to be renamed was first introduced. The way we would do it, is first making the changes, staging them and then executing the following command:
```sh
git commit --fixup 744b43b
```

The git tree will now look as follows:
```
* f7b34e7 - (HEAD -> my_new_feature) fixup! F1
* 05dcacd - F3
* 1a69fa6 - F2
* 744b43b - F1
* a308372 - (master) A3
* 0cd736a - A2
* 4fa144c - A1
* ea977df - * Initial commit
```

All that is left is pushing the changes and attaching a link where our colleague made the comment to the new `fixup!` commit: `* f7b34e7 - fixup! F1`.

At this point our colleague is happy with our changes and approves the PR. Prior to merging, we need to clean up the `fixup!` commits. How do we do this? Using [--autosquash](https://git-scm.com/docs/git-rebase#_options) by executing the following command:
```sh
git rebase --autosquash --interactive a308372
```

This will bring us into the following interactive rebase mode:
```
pick 744b43b F1
fixup f7b34e7 fixup! F1
pick 1a69fa6 F2
pick 05dcacd F3

# Rebase a308372..f7b34e7 onto a308372 (4 commands)
#
# Commands:
# p, pick <commit> = use commit
# r, reword <commit> = use commit, but edit the commit message
# e, edit <commit> = use commit, but stop for amending
# s, squash <commit> = use commit, but meld into previous commit
# f, fixup <commit> = like "squash", but discard this commit's log message
...
```

Pay attention to how the `--autosquash` automatically sorted the `fixup!` commit under the commit it was fixing and also flagged as `fixup`, meaning that as soon as we save, the fixup commit will be squashed into its upper commit without changing its commit log message.

As the PR review at this point is approved and the rebase is done, it's now safe to `git push --force` and merge it.

This is very useful instead of doing a regular git rebase, as it makes it easier for our colleague to see how we applied the requested changes incrementally.

## Taking it a step further using Git Aliases

In some cases it's not as convenient having to type these commands, especially when there are no aliases for `--autosquash` or `--fixup`. Most people I know just use system aliases for all their git commands (e.g. `gcm` for `git commit -m`). I personally don't do that, as it could potentially collide with other tooling on my system. Git offers the option to create [Git Aliases](https://git-scm.com/book/en/v2/Git-Basics-Git-Aliases), where it's still possible to write `git` commands whilst doing the extra work behind the scene.

Let's create two helpful aliases for the git `fixup` use cases.

In the `$HOME` directory there should be a file called `.gitconfig`. We can add aliases there by using `git config` commands (e.g. `git config --global alias.fixup 'commit --fixup $1'`), or simply editing the `.gitconfig` file manually (which is my preferred way in order to see the changes better). Let's add there the following two aliases:
```ini
[alias]
	fixup = "!git commit --fixup $1 #"
	fixup-squash = "!EDITOR=true git rebase --autosquash -i $1 #"
```

Pay attention to the `EDITOR=true` environment variable before the `git rebase --autosquash` command. For whatever odd reason, auto-squashing fixup commits don't work unless the `-i` flag is passed for interactive rebase mode. This hack essentially bypasses the interactive rebase, so that it will automatically squash the fixup commits without having to do the extra step of exiting interactive mode.

Now we can create fixup commits and auto-squashing them like this:
```sh
git fixup 744b43b
git fixup-squash a308372
```

Note that for the `fixup-squash` command it is possible to also pass the most recent referenced branch name (I'm intentionally not using the term `parent branch`, as I'll explain later why). This will save us the extra effort of providing a specific commit hash, e.g. since `my_new_feature` branch started from `master`, it's also possible to write the autosquash command as: `git fixup-squash master`.

It would have been nice making the `git fixup-squash master` command more dynamic by not having to type `master`, wouldn't it? We can fake this by trying to get the nearest branch name that resides on a branch other than the current branch. Here is a script I wrote that attempts to retrieve it:
```sh
#!/bin/sh
# Attempt to get the nearest local branch name
# https://stackoverflow.com/questions/3161204/how-to-find-the-nearest-parent-of-a-git-branch
default_branch=$(git remote show $(git remote get-url origin 2>/dev/null) | awk '/HEAD branch/ {print $NF}')
current_branch=$(git branch --show-current)
if [ "$default_branch" = "$current_branch" ]; then
    printf $default_branch
else
    parent=$(git show-branch | sed "s/].*//" | grep "\*" | grep -v "$(git rev-parse --abbrev-ref HEAD)" | head -n1 | sed "s/^.*\[//")
    if [ -z "$parent" ]; then
        printf $default_branch
    else
        printf $parent
    fi
fi
```

Now let's create an alias that invokes this script:
```ini
[alias]
	parent = "!sh ~/.git/parent.sh #"
	fixup = "!git commit --fixup $1 #"
	fixup-squash = "!EDITOR=true git rebase --autosquash -i $1 #"
```

If you read through the stackoverflow thread attached in the script, it will make more sense why there is no such a thing a parent branch in git. This being said, use caution when writing commands such as `git fixup-squash $(git parent)`. I always check the output of the `git parent` command before executing the more dynamic approach. You may wonder why this dynamic approach is even needed if we can just write the branch name? well, we can create an alias for it too, so that we don't have to provide the additional parameters:
```ini
[alias]
	fixup-autosquash = "!git fixup-squash $(git parent) #"
```

Then we can just execute `git fixup-autosquash` and all the `fixup!` commits will get magically squashed.

## Bonus

We have seen above how nice it is automating some of these commands. Here I would like to introduce to you some other nice git aliases I often use:
```ini
[alias]
	current-branch = "!git rev-parse --abbrev-ref HEAD #"
	default-branch = "!git remote show $(git remote get-url origin) | awk '/HEAD branch/ {print $NF}' #"
	remote-open = "!open $(git remote get-url origin | sed -E 's/git@github\\.com:(.*)\\.git/https:\\/\\/github.com\\/\\1/') #"
	loge = "!git log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit #"
	cleaner = "!git clean -xffd; git submodule foreach --recursive git clean -xffd; git reset --hard; git submodule foreach --recursive git reset --hard; git submodule update --init --recursive #"
	tmp = "!git commit -m \"tmp! $1\" #"
	tmp-squash = "!hashes=$(git log $1..HEAD --grep='tmp!' --oneline | cut -d' ' -f1 | tr '\n' ',' | sed 's/.$//') && git-filter-repo --commit-callback \"$(cat ~/.git/filter-repo-tmp-commits.py | sed \"s/REPLACE_ME_HASHES/$hashes/\")\" --force #"
	dummy = "!touch $(openssl rand -hex 10); git add .; git commit -m \"$1\" #"
```

| Command           | Description                                |
|:------------------|:-------------------------------------------|
| `current-branch`  | gets the current branch name |
| `default-branch`  | gets the default remote branch name (assuming there is only one) |
| `remote-open`     | open a tab to the github's repository (ssh cloned) |
| `loge`            | show git logs in a nice visual mode |
| `cleaner`         | cleans a repository deeply including its sub-modules or any ignored files |
| `tmp`             | creates a commit prefixed with `tmp! [commit_message]` (useful for commits that require alteration prior to merging) |
| `dummy`           | creates a dummy commit for testing purposes |

Here are some practical examples:
```sh
git current-branch
git checkout $(git default-branch)
git remote-open
git loge
git cleaner
git tmp "Change the dates prior to merging"
git dummy "testing commit"
```

### Atlassian's SourceTree

I often use SourceTree when doing complex rebasing or just want to have more visuals. I can launch a repository in it by executing from within the terminal the `stree` command.

Unfortunately SourceTree doesn't have a feature for `--fixup` nor `--autosquash` commands. Fortunately, we can create [Custom Actions](https://confluence.atlassian.com/sourcetreekb/using-git-in-custom-actions-785323500.html) by going to `Settings` > `Custom Actions` -> `Add` name the `Menu Caption` as `fixup` and then add for `Script to run` -> `git` and for `Parameters` -> `fixup $SHA`. Now it's possible to right click any commit -> `Custom Actions` -> `Repository Actions` -> `fixup` and it will pick the SHA from the selected commit.

It's also possible to do the same for `--autosquash` by creating another custom action calling it `fixup-squash` and doing the same as above, just that for `Parameters` -> `fixup-squash $SHA`.

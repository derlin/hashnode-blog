---
title: "Seriously, You Need to Learn Git"
seoDescription: "Most developers do not take the time to learn git. Let me prove it and give some advice on how to really start learning git."
datePublished: Mon Jul 22 2024 07:00:43 GMT+0000 (Coordinated Universal Time)
cuid: clywn11rt00050aifcgw72he8
slug: seriously-you-need-to-learn-git
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1721567945264/63d08a01-70c8-4567-b67a-2a8b0758e7c5.jpeg
tags: productivity, git, learning, opinion

---

*WARNING: ranting incoming.*

I have seen countless articles about git (one of the hottest subjects out here), yet I am still puzzled at how few developers actually took the time to *learn* git.

## You do not know git

To clarify what I mean by that, let's play a game. If I ask you to create a feature branch, commit some changes, and make a pull request, I am sure everything is fine. But now, let me vary the game:

* level one: you can't use any external tool such as Gitlab or your favorite IDE. No clickops, terminal only. This holds for all the other levels.
    
* level two: you have to split your changes into 5 commits. Bonus: the files are from only two files. And please, don't use the old trick of copying all the changes somewhere, and then pasting them one by one to "prepare" each commit.
    
* level three: you pushed your commits 1 to 5, but your reviewer asks to change something in commit 2, reorder commits 3-4, meld commit 5 into 4, change the commit message of commit 1, and rebase your branch to master.
    
* level four: you made a mistake during the level three operation. You need to "recover" and start again, but you already pushed your changes, so a force pull from the origin won't work.
    
* bonus: a bug was introduced into the codebase, but you have no idea when. How do you use git to find the commit that started it all?
    

All of this is doable when you understand the concepts and inner workings of git - (I wanted to write "*when you start actually giving a f\*\*k about the tool you use every day*").

Why bother? Well, the operations I outlined in the game can:

1. speed up your development process,
    
2. make your codebase cleaner and your reviews easier: if all the developers in your team learn how to properly split commits and organize them, you can start thinking about conventional commits, automated changelogs, atomic commits, continuous deployment, clean review processes, etc.
    
3. make you fit for open-source: yup, you'll need all that if you want one day to contribute to the Linux kernel or other important codebases!
    
4. save your life when inevitably someone (you?) screws up a git repo
    
5. \[*... ask ChatGPT to fill up the list ...*\]
    

Don't get me wrong. I am not saying everyone should be a complete master of git, but when 99% of the codebases worldwide are managed through git and "gitting" is a daily activity, I believe spending some time understanding the power of it may be worth it. I myself do not know all the git commands by heart, but I know enough to be aware of the possibilities (and what to google for).

## But you can start learning

Now that I (I hope!) convinced you, here are some pointers on how to start.

### Start with the basics

Start by learning git the hard way. Stop using your IDE or git UI for a while, until you feel at ease with all the basic operations directly using the command line. Do not learn by heart, but ask yourself what git is doing under the hood. For example, "staging area", "branches" and "remote" should be crystal clear. You should also understand what's inside the `.git/config` , how `.gitignore` works, and what are *refs*.

Here are some of what I consider "basic" commands:

* the obvious `clone`, `pull`, `push`, `fetch`
    
* remote management: `git remote add|remove|show origin`
    
* `git status`, and how to add(remove) files/folders to the staging area (e.g. `git rm --cached -r foo/`)
    
* commit (with short and long messages), and especially *amend* the last commit (e.g. `git commit --amend --reset-author`). Also, ensure you know how to revert the latest commit(s) (`git reset --hard/--soft`).
    
* basic branch and tag management: `git checkout [-b] ...`, `git branch ...`, `git tag ...`. Note that checkout works not only with branches but also with files!
    
* stashing (please, do it for yourself!): `git stash` / `git stash pop`, ...
    
* merge and basic rebase: `git merge`, `git rebase`
    
* understand `git log`, (or better, `git log --graph --format=oneline`) and view a specific ref (`git show`, `git revparse`)
    
* some other handy commands: `git diff`, `git revparse`,...
    

Note: this is a very generic advice. When I started programming, my first Java class was not about primitives, it was about `java` and `javac`. I started with a Notepad and a terminal, and only 4 weeks into the class was I allowed to use an IDE. This was brilliant and stayed with me ever since.

### Get comfortable with rebasing

[`git rebase`](https://git-scm.com/docs/git-rebase) is so powerful it is blinding. You could spend months just playing with it. I personally only know 2 things by heart: (a) how to rebase a branch onto another (e.g. `git rebase master` from a feature branch) and (b) how to do interactive rebase.

Play with interactive rebase yourself, and get familiar with the options: `pick`, `reword`, `edit`, `squash`, `fixup`. Learn how to handle conflicts during rebase, and how to abort.

If you fear you'll mess up, don't forget nothing is undoable in git. You can stop an interactive rebase at any point using the `git rebase --abort` command. If you already finished the rebase, you still have the reflog ;).

### Experiment and learn how to undo anything

Trying things out is difficult when you fear about messing up. But anything you do on your local repository can be rolled back. You may have already learned about `git reset` and `git pull --force`, so the next step is to get acquainted with the reflog.

Reference logs (*reflogs*), keep the history of when the tips of branches and other references in the local repository are modified. For example, when you create/delete a branch, do a rebase, merge something, etc. Each log is associated with an SHA, that you can use to go back in time exactly as you do with commits, using `git reset <reflog SHA>`.

I'll let you figure out the rest by yourself! But now that you are aware of its existence, there is no excuse to try things: merge, rebase, delete, mess up all you want, you're covered.

### Be curious

When you return to using your IDE / git GUI (tip: try [lazygit](https://github.com/jesseduffield/lazygit), it's AMAZING), always ask yourself what command(s) are used in the background.

Do a `git --help` once in a while, and read about the commands you never encountered. For example, have a look at `git blame`, `git bissect`, etc.

Ask your peers how they use git, and learn from them. Share your learnings and spread the knowledge.

## Up your game

Now that you're familiar with git, it is time to step up your game. Think of how it could help you streamline your processes and how it can benefit you and your team.

As a bonus, here are additional advice:

* *Sign* your commits using GPG keys (this will annotate your commits with a nice "verified" badge on GitHub)
    
* Improve your pull requests by following the [conventional commits](https://conventionalcommits.org) convention, and your reviews by following the [conventional comments](https://conventionalcomments.org/).
    
* Keep a clean git history using squashing, and think about adopting [atomic commits](https://dev.to/samuelfaure/how-atomic-git-commits-dramatically-increased-my-productivity-and-will-increase-yours-too-4a84)
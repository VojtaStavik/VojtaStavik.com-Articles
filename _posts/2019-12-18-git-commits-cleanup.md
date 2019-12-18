---
layout: post
title: "Clean Git History without Terminal"
filename: "2019-12-18-git-commits-cleanup.md"
social-image: "/images/git-cleanup/git-cleanup-1.png"
---

![Git cleanup](/images/git-cleanup/git-cleanup-1.png)

Your commits represent and encapsulate your added value to the project you're working on. They are meant to _forever_ live in the history of your repo. They are your legacy to all the programmers who will work on the project after you.

It's almost magical when you think about it. For example, you can still find [the very first commit from Chris Lattner in the Swift repo](https://github.com/apple/swift/commit/afc81c1855bf711315b8e5de02db138d3d487eeb). If you have (a lot of) time, you can go commit by commit from this one and re-experience the whole history of the Swift project.

A commit history doesn't need to tell the true story of the development process with all its hiccups and try-and-errors. I actually think it **shouldn't** tell the real story. 

**A good git history tells an idealized version of the story where we knew exactly what we were doing.** Let me show you how to achieve this.

<!-- more -->

## Tools

Some people find interacting with `git` directly from the terminal intimidating. For the purpose of this article, I'll be using my `git` GUI of choice, [Fork](https://git-fork.com), for all `git` operations. However, everything I'm doing here can be achieved from the terminal, too.

## My Git Workflow

This is my typical git workflow when working on a new pull request:

1. Create a new branch for the task.

2. Work on the task. Commit more/less _randomly_ with commit messages like `"WIP"`, `"experiment"`, `"keep this??"`, etc. 

3. Push to the server regularly, at least once a day.

4. When done with the task, reset all commits and create new ones representing the idealized version of the story. This usually takes just a couple of minutes. The most difficult part is to come up with meaningful commit messages.

5. Force-push the branch. ⚠️ This can be done safely only when I'm the only person working on the PR.

6. Open a PR, ask for a review.

7. Incorporate the feedback from the review using `fixup!` commits.

8. Apply fixup commits, force-push one more time, merge the branch 🎉


## Resetting the History

Find the root commit of your branch and select ```Reset 'branch name' to Here```.

![Git cleanup](/images/git-cleanup/git-cleanup-2.png)

Select `Soft` or `Mixed` for the reset type. This doesn't discard your current changes, it only resets the commits. **Don't select `Hard`⚠️ if you don't want to discard all your changes!**

![Git cleanup](/images/git-cleanup/git-cleanup-3.png)

You can now see all your changes at once in `Unstaged changes`.

![Git cleanup](/images/git-cleanup/git-cleanup-4.png)


## Recreating the Commits

Now, the fun part: Go through the changes and stage only the ones that make a nice small meaningful commit.

For example, let's put all changes related to updating Pods to a separate commit:

![Git cleanup](/images/git-cleanup/git-cleanup-5.png)

![Git cleanup](/images/git-cleanup/git-cleanup-6.png)

You don't have to always stage all changes in a file. Simply select the lines you want to stage and press `Stage`:

![Git cleanup](/images/git-cleanup/git-cleanup-7.png)

![Git cleanup](/images/git-cleanup/git-cleanup-9.png)


If you find a change that shouldn't be there, feel free to discard it. No more accidentally committed changes!

![Git cleanup](/images/git-cleanup/git-cleanup-8.png)

Repeat these steps until you have all the changes committed.

**Voila! We now have nice and meaningful commits:**

![Git cleanup](/images/git-cleanup/git-cleanup-10.png)

We can now push the new commits to the server and open a PR. We need to use "force" push because we want to rewrite the existing commits on the remote branch.

![Git cleanup](/images/git-cleanup/git-cleanup-11.png)

> ⚠️ There's nothing wrong with using force-push extensively in your git workflow. However, it can be dangerous in situations when you're not the only person working on the given branch (or just yourself working from multiple computers). 

> Always be careful when doing this operation. I strongly recommend learning more about how to use `push --force` and `push --force-with-lease` safely before you start using this workflow.

## Incorporating PR Feedback

OK, the PR has been opened for a while and we gathered some valuable feedback from our teammates. How to fix all the issues they found while keeping the commit history nice and readable? Meet `fixup!` commits. They are meant to exist only temporarily and their only purpose is to fix another, already existing commit.

After making the required changes, we need to identify to which commit(s) the new changes relate.

We will commit the changes with the name `fixup! <the name of the commit we fix>`. We repeat this process with all the new changes.

![Git cleanup](/images/git-cleanup/git-cleanup-12.png)

If you want your teammates to review the fixups you've made, you can push the fixup commits to the server and ask them for another review.

## Applying All `fixup!` Commits

Once the whole team is happy with the PR, there's one final step. Of course, we don't want to have the `fixup!` commits on `master`!

We select the root commit of our branch and choose `Rebase 'branch name' to Here Interactively`.

![Git cleanup](/images/git-cleanup/git-cleanup-13.png)

Fork opens a new modal dialog and it automatically offers to squash the `fixup!` commits with their related commits.

![Git cleanup](/images/git-cleanup/git-cleanup-14.png)

The only remaining step is to force-push the updated commits and merge the PR 🎉

## Tips & Ideas
- The interactive rebase functionality also allows you to change the order of the commits, reword, or even remove certain commits completely.

- When your work touches `xcodeproj` file a lot, it can be sometimes difficult to correctly identify and split all the changes in this file into separate commits. I always try to commit changes like adding a target or a file immediately.

- If a part of your task is renaming of certain elements, it's easier to commit the related changes immediately, rather than recreating the "Rename xxx" commit later.
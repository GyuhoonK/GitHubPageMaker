---
layout: post
current: post
cover: assets/built/images/git/merge-rebase.png
navigation: True
title: merge vs. rebase
date: 2022-05-14 22:30:00 +0900
tags: [git]
class: post-template
subclass: 'post tag-git'
author: GyuhoonK
---

merge와 rebase 비교하기

# Merge & Rebase 

Git에서 브랜치를 합치는 두 가지 방법으로 merge와 rebase가 있습니다. 이 둘은 서로 다른 브랜치를 하나의 브랜치로 합친다는 공통점을 갖지만, 이외의 기능은 많은 차이를 갖습니다.

## git merge

> Join two or more development histories together
>
> Incorporates changes from the named commits (since the time their histories diverged from the current branch) into the current branch. This command is used by *git pull* to incorporate changes from another repository and can be used by hand to merge changes from one branch into another.

대상 브랜치의 커밋 내용을 현재 브랜치로 포함시킨다는 것이 `git merge` 의 가장 큰 특징입니다.  서로 다른 브랜치 간에서만 사용되는 것이 아니라, 저장소 간의 내용을 하나로 통합하는 경우에도 사용됩니다. 즉, `git pull`에서도 `git merge`를 이용합니다.

git document는 아래와 같은 상황을 예시로 `git merge` 를 설명합니다.

```
	    A---B---C topic
	   /
D---E---F---G master
```

이 때, `topic`, `master`의 공통 조상이 되는 commit인 `E` 를 **base**라고 부릅니다.

현재 브랜치(current branch)가 `master` 일 때, `git merge topic` 을 실행하면, 아래와 같은 브랜치 히스토리가 생성됩니다.

```bash
$ git checkout master
$ git merge topic
```

```
	    A---B---C topic
  	 /         \
D---E---F---G---H master
```

`base`로부터 현재 `topic` 브랜치의 current commit인 `C`까지의 모든 변화(`E-A-B-C`)를 `master` 브랜치의 current commit이었던 `G`로부터 재실행하게 됩니다. 그리고 이를 `H` 라는 commit으로 생성합니다.

이 때, 충돌(conflict)가 발생하게 되면 `git merge` 를 입력한 사용자는 충돌을 해결하고 `merge` 를 이어가거나(`git merge --continue`), 이전 상태로 되돌려야만 합니다(`git merge --abort`).

## git rebase

`rebase` 는 `<upstream>, <branch>`를 명시할 수 있습니다. `<branch>` 옵션은 명시하지 않으면 현재 명령어를 실행하는 브랜치(current branch)로 입력됩니다.

```bash
$ git rebase <upstream> <branch>
```

> Reapply commits on top of another base tip

> If `[branch]` is specified, *git rebase* will perform an automatic `git switch [branch]` before doing anything else. Otherwise it remains on the current branch. All changes made by commits in the current branch but that are not in `[upstream]` are saved to a temporary area.The commits that were previously saved into the temporary area are then reapplied to the current branch, one by one, in order. Note that any commits in HEAD which introduce the same textual changes as a commit in HEAD..`[upstream]` are omitted (i.e., a patch already accepted upstream with a different commit message or timestamp will be skipped).

`[upstream]`에 존재하지 않고, current branch(`[branch]`)에 존재하는 commit은 `patch`라는 임시 공간에 저장됩니다.  이후 해당 `patch`에 저장되있는 commit은 **순서대로** `upstream`의 current commit에서부터 적용됩니다. 이 과정은 공통 commit인 `base` 를 upstream의 current commit으로 변경하는 작업으로 이해할 수 있습니다.

	      A---B---C topic
	     /
	D---E---F---G master
위와 같은 상황에서 아래 명령어를 실행하면,  `topic` 브랜치의 base를  `master` 의 current commit `G` 로 변경합니다.

```bash
$ git checkout topic
$ git rebase master # git rebase master topic
```

```
              A'--B'--C' topic
             /
D---E---F---G master
```

current branch인 `topic` 의 `A-B-C` 커밋들은 `patch` 에 잠시 저장되어 있다가, `G` 에서부터  `patch`의 커밋 내용들을 `master` 브랜치에 **순서대로** 적용합니다.

`git merge`가 `A-B-C`를  **한번에** 적용하여 `G--H` 로 commit했던 것과 다른 부분입니다.

이 때, 만약 upstream(`matser`)의 커밋 중 일부가 branch(`topic`)의 커밋 내용을 포함하고 있는 경우, 해당 커밋 내용은 건너뛰고(skipped) 진행됩니다.

 ```
       A---B---C topic
      /
 D---E---A'---F master
 # A in topic is same to A' in master 
 ```

```
               B'---C' topic
              /
D---E---A'---F master
# A in topic is skipped when `git rebase master`
```

하나의 브랜치(upstream)으로 부터 여러 개의 브랜치가 생성되었을 때, rebase는 이를 간결하게 표현해 줄 수 있습니다. 이는 `--onto`옵션을 이용합니다.

> The current branch is reset to `[upstream]`, or [newbase] if the --onto option was supplied. This has the exact same effect as `git reset --hard `[upstream]`` (or [newbase]). ORIG_HEAD is set to point at the tip of the branch before the reset.

> --onto [newbase]
>
> Starting point at which to create the new commits. If the --onto option is not specified, the starting point is <upstream>. May be any valid commit, and not just an existing branch name.

```
o---o---o---o---o  master
     \
      o---o---o---o---o  next
                       \
                        o---o---o  topic
```

만약   `topic`의 변화 내용만을 `master` 에 병합하고 싶다면 우리는 `topic`이 `master` 브랜치로부터 분화(forked)된 것으로 변경해야합니다. rebase의 `--onto` 옵션을 이용하면  `topcic` 의 base branch를  `master`로 변경할 수 있습니다.

```bash
$ git rebase --onto master next topic
```

newbase인 `master` 브랜치로부터 새로운 커밋들을 rebase하게 됩니다. 이 경우, topic이 master로부터 forked되지 않았으므로 수정되게 됩니다.

즉, `master` 브랜치가  `topic`의 base로 변경됩니다.

```
o---o---o---o---o  master
    |            \
    |             o'--o'--o'  topic
     \
      o---o---o---o---o  next
```

아래의 경우처럼 브랜치 간 관계가 복잡한 경우에도 rebase를 이용하면 관계를 단순하게 조작할 수 있습니다.

```
                        H---I---J topicB
                       /
              E---F---G  topicA
             /
A---B---C---D  master
```

```bash
$ git rebase --onto master topicA topicB
```

```
             H'--I'--J'  topicB
            /
            | E---F---G  topicA
            |/
A---B---C---D  master
```

# Purpose

`merge`와 `rebase`는 두 개 이상의 브랜치를 합친다는 공통점은 있지만 동작 방식과 그 결과는 다릅니다. 따라서 목적에 따라 다르게 사용해야합니다.

> #### Git Rebase
>
> - Streamlines a potentially complex history.
> - Avoids merge commit “noise” in busy repos with busy branches.
> - Cleans intermediate commits by making them a single commit, which can be helpful for DevOps teams.
>
> #### Git Merge
>
> - Simple and familiar.
> - Preserves complete history and chronological order.
> - Maintains the context of the branch.

`rebase`는 복잡한 히스토리를 단순화하는데 주요한 목적이 있으나, 이러한 과정에서 정확한 forked 정보는 소실될 수 있습니다. 또한 `merge`에 비해서 사용이 어렵고 복잡합니다.

반대로 `merge`는 fork, commit, merge에 대한 모든 히스토리가 그대로 남아있어 추후에 트래킹에서의 이점이 있을 수 있습니다. 그러나 모든 히스토리가 남기 때문에 `rebase`에 비해 히스토리가 복잡해지는 단점이 있습니다.



[참고]

[Git Document](https://git-scm.com/)

[Git Rebase vs. Git Merge: Which Is Better?](https://www.perforce.com/blog/vcs/git-rebase-vs-git-merge-which-better)

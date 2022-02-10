---
title: Copy a Git repo to a new one
date: 2022-02-10 14:41:08
tags: git
---

For some reason we need to copy all the things from one git repository to another, include the commit history. Below will show an instruction for how to do it.

<!-- more -->

1. Create a new repo
   This most by click the "create" button in some GUI or just do **git init** to create an empty repo

> Note: "empty repo" means no need to create a initial commit, otherwise you need force override this commit later

2. Clone the new repo to local

```
git clone https://github.com/new.git
```

3. Switch remote url to the old repo

```
git remote set-url origin https://github.com/old.git
```

4. Fetch all from old repo

```
git fetch
```

The **git fetch** will only fetch the master branch, but if the old repo contains more branches, we also need to fetch them to local

```
git branch -r | grep -v '\->' | sed "s,\x1B\[[0-9;]*[a-zA-Z],,g" | while read remote; do git branch --track "${remote#origin/}" "$remote"; done
```

Now we got everything we want in our local repo

5. Swtich back to new repo

```
git remote set-url origin https://github.com/new.git
```

6. Push everything to new repo

```
git push --all
```

Also we need push all tags to remote repo

```
git push --tags
```

Done

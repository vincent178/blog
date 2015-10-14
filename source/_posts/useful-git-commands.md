title: 好用的 git 命令
date: 2015-10-14 11:20:06
tags:
---

1. Create Branch
```bash
git checkout -b <branch-name>

git branch   <branch-name>
git checkout <branch-name>
```

2. Check Branches
```bash
git branch    #=> check local branches
git branch -r #=> check remote branches
```

3.  Delete Local Branch
```bash
git branch -d <branch-name>
git branch -D <branch-name> # use '-D' means force to delete branch
```

4. Delete Remote Branch
```bash
git push origin --delete <branch-name>
```

5. Update your remote branch list
```bash
git remote prune <remote-name>
```

6. Remove untracked files in your working directory
```bash
git clean -f
```

7. Remove tracked files in your working directory
```bash
git reset --HARD
```

8. Remove tracked and deleted files in your working directory
```bash
git rm $(git ls-files --deleted)
```

9. Stash Usage
```bash
git stash save <name>
git stash pop  <name>
git stash drop
```

10. Rollback to a specific commit 
Git reset without the --hard option resets the commit history but not the files. With the --hard option also files in working tree are reset.
```bash
git reset --hard <tag|branch|commit id>
```

11. Ignoring changes in tracked files
```bash
git update-index --assume-unchanged <file>
# If you wanna start tracking changes again
git update-index --no-assume-unchanged <file> 
```

12. Speed up while lots of files need to track <br />
Usually git-gc runs very quickly while providing good disk space utilization and performance. 
This option will cause git-gc to more aggressively optimize the repository at the expense of taking much more time. 
```bash
git gc --aggressive
```

13. Global git ignore
```bash
git config --global core.excludesfile '~/.gitignore'
```


# Git Notes

### Git Commands

#### Show the local changes that have been made
```git status```

#### Add the file(s) to the index staged for the next commit
```git add <file>```

#### Add only files that are tracked and changed
```git add -u *```

#### Commit (record) the changes to the repository
```git commit -m 'descriptive commit message'```

#### Push the recorded changes to the repository
```git push origin <branch-name>```

#### Pull any changes to the repository into your local from master
```
git fetch origin
git merge origin/master
```

#### Update branch from master
```git merge master```

#### Undo a commit that hasn't been pushed yet
```git reset --soft HEAD^```

#### Undo all commits that haven't been pushed yet
```git reset --soft HEAD^^```

# Unix

##Fedora

Give user sudo privileges: Add user to wheel group.

```
gpasswd wheel -a username
```

#### SCP

Copy local file to remote location:

```
scp -i $path/to/keyfile a-file.txt chris@remoteMachine:/desired/remote/destination
```

Copy remote file to local:

```
scp -i $path/to/keyfile chris@remoteMachine:/desired/remote/sourceFile /desired/local/destination
```
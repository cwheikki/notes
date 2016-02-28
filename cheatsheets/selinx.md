# SELinux Notes

See current settings:
```getenforce```

Adjust settings:
```
setenforce 0 // Permissive
setenforce 1 // Enforcing
```

View all configs:
```getsebool -a```

modify settings
```vim /etc/sysconfig/selinux```

Audit2Allow:
```grep nginx /var/log/audit/audit.log | audit2allow```
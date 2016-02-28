# NginX Notes

```
service nginx start

service nginx stop

service nginx reload

service nginx restart
```

### nginx config files
```
/etc/nginx/nginx.conf
/etc/nginx/sites-available/*.conf
/etc/nginx/sites-enabled/*.conf
```


### Systemd Setup
Service file
/etc/systemd/system/heik-io-node
.service
```
[Service]
ExecStart=/bin/node /home/heik-io-node/heik.io/server.js
Restart=always
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=heik.io
User=heik-io-node
Group=heik-io-node
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```

Start the service

systemctl enable heik-io-node
systemctl start heik-io-node
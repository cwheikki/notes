## Setting up NginX with Node.js

I used Fedora (20.0) for my setup

Requirements:
    A server with systemd
    nginx
    git
    Root access

Disable Apache
```
// Shut down Apache
service httpd stop
// Remove from boot cycle so it doesn't start up during server boot
systemctl disable httpd
// To restart later, if desired: systemctl enable httpd
```

dnf install nginx git nodejs

// start nginx
service nginx start
// enable nginx to start upon boot
systemctl enable nginx

Should be able to see test page at http://1.2.3.4 (your server IP address)

useradd -mrU heik-io-node

passwd <password>

su heik-io-node

// Deploy node application
// gulp

// Symbolic link between sites-available config and sites-enabled

```
ln -s /etc/nginx/sites-available/nginxsite.com.conf /etc/nginx/sites-enabled/nginxsite.com.conf
```

restart nginx
```
service nginx restart
```

Resources:
https://www.digitalocean.com/community/tutorials/understanding-nginx-server-and-location-block-selection-algorithms
http://handyjs.org/article/the-kick-ass-guide-to-deploying-nodejs-web-apps-in-production
# Initial Fedora Setup
Ref: https://www.digitalocean.com/community/tutorials/initial-setup-of-a-fedora-22-server


    >> ssh root@IPAddress
    >> adduser chris
    >> passwd chris
    >> gpasswd -a chris wheel

From local machine, add public SSH key

    >> cat ~/.ssh/id_rsa.pub | ssh chris@remote_host "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

Disallow root login and password auth

    >> sudo dnf update
    >> sudo dnf install -y vim

Find PermitRootLogin directive. Uncomment and set to no

`>> sudo vim /etc/ssh/sshd_config`

    PermitRootLogin no
    ...
    PasswordAuthentication no

Save and exit. Reload config.

` >> sudo systemctl reload sshd `

Configure time zone

 `>> sudo ln -sf /usr/share/zoneinfo/US/Central /etc/localtime`

Verify the date

    >> date
    Sun Feb 28 17:44:12 CST 2016


Enable Firewall

    >> sudo dnf install -y iptables-services
    >> sudo systemctl enable iptables
    >> sudo systemctl start iptables

Save rules to persist after reboot
(Saved to /etc/sysconfig/iptables)
` >> sudo /usr/libexec/iptables/iptables.init save `

Allow HTTP and HTTPS traffic

` >> sudo vim /etc/sysconfig/iptables `

Add 2 new rules, one for port 80 and one for port 443 after the rule for port 22 (SSH)

    ...
    -A INPUT -p tcp -m state --state NEW -m tcp --dport 80 -j ACCEPT
    -A INPUT -p tcp -m state --state NEW -m tcp --dport 443 -j ACCEPT
    ...

Activate new rule set

`>> sudo systemctl restart iptables`
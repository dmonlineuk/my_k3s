# Logging

Desire is to send infrastructure logs to a syslog receiver. Since each node is set up with a Ubuntu server minimal install, the steps are straightforward enough:

```bash
# Apt update and upgrade, then install rsyslog
sudo apt update && sudo apt upgrade -y && sudo apt install rsyslog vim -y

# Create the conf file for sending log messages to remote receiver
sudo vim /etc/rsyslog.d/60-remote.conf
```

```conf
*.* action(
    type="omfwd"
    target="syslog-01"
    port="514"
    protocol="tcp"
    queue.type="linkedList"
    queue.filename="remote"
    action.resumeRetryCount="-1"
)
```

```bash
# Check the configuration has no errors
sudo rsyslogd -N1

# Restart rsyslog
sudo systemctl restart rsyslog && sudo systemctl status rsyslog

# Using lnav on the syslog receiver, you should see messages related
# to the service restart. Additionally, send a test log message, e.g.
logger -p user.notice -t k3s-lab "test message from $(hostname)"
```

Note this does not encrypt anything so messages are being sent in the clear. This is acceptable for a test lab, but needs further action for production.

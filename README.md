# Devops_intern_Assignment
#. Launch EC2 Instance
Log in to AWS Console → EC2.

Click Launch Instance.

Choose:
Ubuntu  (Free Tier eligible)
Instance type: t2.micro
Key pair: Create or use existing
Network settings: Allow SSH (port 22)
Launch the instance.

After launching instance i have ssh to it in Moobaxterm.

After that i have created a new user by the command **sudo adduser devops_intern**

To allow **devops_intern** to run administrative commands without being prompted for a password, I added a sudoers configuration.


```**sudo visudo -f /etc/sudoers.d/devops_intern**
```

Inside the file, I added:

```**devops_intern ALL=(ALL) NOPASSWD:ALL**
```

This ensures the user has full sudo privileges.

I updated the hostname to make the server easily identifiable.

```**sudo hostnamectl set-hostname prashant-devops**
```
Then updated /etc/hosts:

```**sudo nano /etc/hosts**
```
Replaced old hostname with:

127.0.0.1   prashant-devops
Rebooted the system:
```sudo reboot
```
Reconnect using SSH afterward.

#Part 2

I installed Nginx, which serves static content by default.

```**sudo apt update
sudo apt install nginx -y**
```
After installation, the service starts automatically.

The goal was to display my name, the EC2 instance ID (fetched from metadata), and uptime.

```**sudo tee /var/www/html/index.html > /dev/null <<EOF
<html>
<body>
<h1>DevOps Intern Task</h1>
<h2>Name: Prashant Giri</h2>
<h2>Instance ID: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)</h2>
<h2>Uptime: $(uptime -p)</h2>
</body>
</html>
EOF**
```
I accessed it through:
```http://PUBLIC_IP
```

#part 3

The script outputs date, uptime, CPU usage, memory usage, disk usage, and top processes.


```**sudo tee /usr/local/bin/system_report.sh > /dev/null <<'EOF'
#!/bin/bash
echo "===================="
echo "Date & Time: $(date)"
echo "Uptime: $(uptime -p)"
echo "CPU Usage: $(top -bn1 | grep 'Cpu(s)' | awk '{print 100-$8}')%"
echo "Memory Usage: $(free | awk '/Mem/ {printf \"%.2f\", $3/$2 * 100}')%"
echo "Disk Usage: $(df -h / | awk 'NR==2{print $5}')"
echo "Top 3 CPU-consuming processes:"
ps -eo pid,pcpu,comm --sort=-pcpu | head -n 4
echo ""
EOF**
```
Make executable:

```**sudo chmod 755 /usr/local/bin/system_report.sh**
```

I scheduled it to run every 5 minutes and store logs to /var/log/system_report.log.

```**sudo crontab -e**
```

Added:

```***/5 * * * * /usr/local/bin/system_report.sh >> /var/log/system_report.log 2>&1**
```

#Part 4

Since the Ubuntu AMI did not include AWS CLI by default, I installed AWS CLI v2 manually.
This allowed me to interact with AWS CloudWatch from the EC2 instance.

```**cd /tmp
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
After installation, I verified it using aws --version.
**
```
To authenticate my instance with AWS services, I configured the AWS CLI using IAM access keys.

```**aws configure**
```
I entered:

AWS Access Key ID

AWS Secret Access Key

Default region: ap-south-1

Output format: json

This allowed the instance to authenticate and push logs to CloudWatch.

A Log Group is a top-level container in CloudWatch where logs are stored.

I created the required log group named:

```**/devops/intern-metrics**
```
Command:

```**aws logs create-log-group \
  --log-group-name /devops/intern-metrics \
  --region ap-south-1**
```

  Inside the log group, CloudWatch needs a log stream, which acts like a sub-folder where logs are appended.


```**aws logs create-log-stream \
  --log-group-name /devops/intern-metrics \
  --log-stream-name intern-stream \
  --region ap-south-1**
```

  CloudWatch expects logs in a JSON format with timestamps.
  
Since /var/log/system_report.log is a plain text file, I converted it into a JSON events file using Python.

```**python3 - <<'EOF' > /tmp/events.json
import time, json
events=[]
now=int(time.time()*1000)
for line in open('/var/log/system_report.log'):
    events.append({"timestamp": now, "message": line.strip()})
print(json.dumps(events))
EOF**
```

With the log group, log stream, and events file ready, I pushed the log data into CloudWatch Logs:

```**aws logs put-log-events \
  --log-group-name /devops/intern-metrics \
  --log-stream-name intern-stream \
  --log-events file:///tmp/events.json \
  --region ap-south-1**
```
After running this command successfully, the monitoring script logs appeared in the CloudWatch Logs console.

#part 5 (OPTIONAL)

While cron works fine for periodic execution, systemd timers offer better reliability, centralized logging, dependency handling, and self-healing via service restarts.
Using systemd makes the monitoring script behave more like a proper Linux service, which is the standard in modern DevOps environments.

Create the systemd service
```
**sudo tee /etc/systemd/system/system_report.service > /dev/null <<EOF**
**[Unit]
Description=Run system report script

[Service]
Type=oneshot
ExecStart=/usr/local/bin/system_report.sh >> /var/log/system_report.log 2>&1
EOF**
```

Create the systemd timer

```**sudo tee /etc/systemd/system/system_report.timer > /dev/null <<EOF
[Unit]
Description=Run system report every 5 minutes

[Timer]
OnBootSec=1min
OnUnitActiveSec=5min
Persistent=true

[Install]
WantedBy=timers.target
EOF**
```
Enable and start the timer

```**sudo systemctl daemon-reload
sudo systemctl enable --now system_report.timer**
```

Check next scheduled run

```**systemctl list-timers | grep system_report**
```

To add alerting capability, I integrated AWS SES so that an email is sent when the disk usage exceeds a defined threshold (80%).
This brings real monitoring behavior into the system: whenever the server is low on disk space, an automated notification email is sent to:

connecttoprashantgiri@gmail.com

Before sending emails, AWS SES requires:

Verifying the sender email

Using the correct region (ap-south-1)

Once verified, the script uses AWS CLI to send an alert.

Updated system_report.sh with alert logic

```**sudo tee /usr/local/bin/system_report.sh > /dev/null <<'EOF'
#!/bin/bash
TIMESTAMP="$(date '+%Y-%m-%d %H:%M:%S %Z')"
HOSTNAME="$(hostname)"

echo "===================="
echo "System Report - ${TIMESTAMP}"
echo "===================="
echo "Uptime: $(uptime -p)"
CPU_USED=$(top -bn1 | grep "Cpu(s)" | awk '{print 100-$8}')
echo "CPU Usage: ${CPU_USED}%"
MEM_USAGE=$(free | awk '/Mem:/ {printf(\"%.2f\", $3/$2 * 100)}')
echo "Memory Usage: ${MEM_USAGE}%"
DISK_PCT=$(df --output=pcent / | tail -1 | tr -dc '0-9')
DISK_HUMAN=$(df -h / | awk 'NR==2 {print $5}')
echo "Disk Usage: ${DISK_HUMAN}"
echo "Top 3 CPU-consuming processes:"
ps -eo pid,pcpu,comm --sort=-pcpu | sed -n '1,4p'
echo ""
**
**# Send email if disk usage > 80%
if [ "$DISK_PCT" -ge 80 ]; then
  aws ses send-email \
    --from "connecttoprashantgiri@gmail.com" \
    --destination "ToAddresses=connecttoprashantgiri@gmail.com" \
    --message "Subject={Data=Disk Alert},Body={Text={Data=Disk usage exceeded 80% on ${HOSTNAME} at ${TIMESTAMP}}}" \
    --region ap-south-1
  echo "ALERT SENT (Disk ${DISK_HUMAN}) at ${TIMESTAMP}"
fi
EOF**
Make script executable again:
**sudo chmod 755 /usr/local/bin/system_report.sh**
**
```
How AWS SES Was Configured

Open AWS Console → Amazon SES (Region: ap-south-1)

Go to Verified Identities

Click Create Identity → select Email

Enter connecttoprashantgiri@gmail.com

SES sends a verification email → click the link

Once status becomes Verified, SES can send and receive email

These are all the steps which i have followed and i have completed the task and attached the screenshots.

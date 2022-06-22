# LinuxAvPOC
Antivirus for Linux server and detailed report in AWS CloudWatch.

Here our end goal is to create a cron job which scans the whole system or whatever the directory you want to scan. Once the scan is done it will send the scan report to AWS cloudwatch. And if the malware is detected then CloudWatch will instantiate the Alarm.

Open Source cross platform AV: ClamAV - supported by Cisco
Website: https://www.clamav.net/

**Install Clam AV on Linux**
```
1. sudo apt update
2. sudo apt-get install clamav clamav-daemon -y
```
After the installation is complete, you'll need to stop the daemon, so you can update the ClamAV database manually. Stop the daemon with the command:
 
```
#Stop Clamav
sudo systemctl stop clamav-freshclam
#update clam
sudo freshclam 
# if daemon not started automatically
sudo systemctl start clamav-freshclam 
```
**Manual database update steps:(Optional Step)**
```
-- sudo wget https://database.clamav.net/daily.cvd # get daily database update
-- sudo cp daily.cvd /var/lib/clamav/
```
**Test clamscan with the below commands**
```
sudo clamscan --infected --detect-pua=yes --recursive pathOfDirectoryToScan
```

**Full Scan**

```
sudo clamscan \
  --suppress-ok-results \
  --recursive \
  --log=/var/log/clamav-scan.log \
  --cross-fs=no \
  /
```

 
**Test using a sample malware file (**https://www.eicar.org/?page_id=3950**)**
```
1. wget -P ~/ http://www.eicar.org/download/eicar.com
2. mv eicar.com ~/home/XXX-USER/infectedfolder/
3. sudo clamscan --infected --remove --recursive ~/home/XXX-USER/infectedfolder/
```


**Local Recursive Scan**
```
clamscan  --suppress-ok-results  --recursive  
 ```

**How to set ClamAV to scan automatically?**

Now we'll create a bash script that will scan the /home/ubuntu/infectedfolder directory and then create a cron job to run it everyday. 

**vim /home/ubuntu/clamscan_daily.sh**
```
#!/bin/bash
LOGFILE="/var/log/clamav/clamav-antivirus.log";
DIRTOSCAN="/home/ubuntu/infectedfolder";

for S in ${DIRTOSCAN}; do
 DIRSIZE=$(du -sh "$S" 2>/dev/null | cut -f1);
 echo "Starting scan of "$S" directory.
 Directory size: "$DIRSIZE".";
 sudo clamscan -ri  --suppress-ok-results --detect-pua=yes "$S" >> "$LOGFILE";
 #find /var/log/clamav/ -type f -mtime +30 -exec rm {} \;
 MALWARE=$(tail "$LOGFILE"|grep Infected|cut -d" " -f3);

  if [ "$MALWARE" -ne "0" ];then
     echo "MALWARE Detected";
  fi
done
exit 0
```

Give that file executable permissions with the command:
sudo chmod u+x /home/ubuntu/clamscan_daily.sh

Create the cron job with the command:
```sudo crontab -e ```
At the bottom of the file, add the following line to run the scan every day at 1 am:
```
1 1 * * * sudo /bin/bash /home/ubuntu/clamscan_daily.sh > /dev/null 2>&1
```
Add Logs into AWS CloudWatch 
	If you have already installed aws cloudwatch agent then you can skip this steps 

**Download & Install**
```
wget https://s3.us-west-1.amazonaws.com/amazoncloudwatch-agent-us-west-1/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
sudo dpkg -i -E ./amazon-cloudwatch-agent.deb
```

**Setup with existing configuration**
```
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```

**Manually edit configuration json**

sudo nano /opt/aws/amazon-cloudwatch-agent/bin/config.json	(Verify logs json has timezone property)

```
{
     "agent": {
         "run_as_user": "root"
     },
     "logs": {
         "logs_collected": {
             "files": {
                 "collect_list": [
                     {
                         "file_path": "/var/log/clamav/clamav-antivirus.log",
                         "log_group_name": "ClamAvLogs_domain.com",
                         "log_stream_name": "{instance_id}"
                     }
                 ]
             }
         }
     }
 }
```

**Start service with updated configuration json**
```
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -s -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json


sudo service amazon-cloudwatch-agent restart
```

**Logs for debugging**
```
cat /var/log/amazon/amazon-cloudwatch-agent/amazon-cloudwatch-agent.log

```


More resource - create a cron job for running virus scanning automatically

https://www.techrepublic.com/article/how-to-install-and-use-clamav-on-ubuntu-server-20-04/

https://www.rapidspike.com/blog/how-to-send-log-files-to-aws-cloudwatch-ubuntu/

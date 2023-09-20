---
title: "Mastodon Storage Management"
date: 2023-09-15T21:41:58-07:00
draft: true
summary: Manage the storage that Mastodon uses
series: Personal Mastodon Server
---

We will build on the monitoring enabled by the CloudWatch agent installed in the previous post in the series. Make sure
the agent is installed and correctly running and uploading logs and metrics.

# Create a Disk Usage Alarm

Without an alarm that your disk usage is getting high you won't know until your server stops responding. Recovering from
100% disk usage can be difficult, so ideally you can take action before then.

Go to the AWS CloudWatch console and view all alarms. Click on the "Create alarm" button.

## Select Metric

Go to the CWAgent metric namespace. Select the disk_used_percent metric for your Mastodon instance and path "/". Search 
all the metrics for disk_used_percent to make it easier to find. Don't use any metric that is not specifically for the
"/" path.

Verify the data in the graph looks how you expect and then click "Select metric".

## Specify Metric and Conditions

Set the statistic type to Average and the Period to 5 minutes.

Under conditions set the threshold type to static. Select greater and then enter the percent value you want to be 
notified at. I choose 80 for my alarm.

Click "Next".

## Configure Actions

Select "In alarm" for the alarm state trigger. Either create the new topic "Default_CloudWatch_Alarms_Topic" or select
it if already existing. You can also use a different name for the topic than the AWS default.

If you created the topic enter your email address to be subscribed to receive emails for all alarms sent to this topic.
If you used a previously created topic I recommend verifying the subscription for you to receive emails for the alarms.
You will receive an email confirming your subscription to the topic and need to click the link in the email to verify.

Click "Next"

## Finish Alarm

Give your alarm a name and optionally a description. Click "Next".

Verify the alarm details and when good click "Create alarm".

# Automate Cleaning Mastodon Storage

The previous post in this series about Mastodon Federation mentioned a tool to remove old media saved on your server
that can be safely cleaned up and fetched again as needed. This tool can be automated to run periodically to keep
your disk usage lower.

## Run Script Using Cron

Run everything in this section as the Mastodon user in the home directory.

Create the script media-remove.sh with the following contents:

```shell
date
echo "Running tootctl media remove"
cd ~/live
RAILS_ENV=production bin/tootctl media remove --concurrency 1
date
echo "Complete"
echo ""
```

Make sure to add execute permissions to the script.

Edit the cron file with `crontab -e` and add the line:

```shell
0 8 * * * PATH=$HOME/.rbenv/bin:$HOME/.rbenv/shims:$PATH $HOME/media-remove.sh >> /var/log/media-remove/output.log 2>&1
```

This runs the script every day at 08:00 UTC. Change the time to when you expect your server to not be in use.

## Log the Media Remove Output

Run this section as your user that can sudo.

Create the directory media-remove in /var/log for the output of the script. Change the owner of this directory to the
*mastodon* user.

Create a file /etc/logrotate.d/media-remove to rotate the logs and add the following content:

```shell
/var/log/media-remove/*.log {
        monthly
        rotate 12
        compress
        missingok
        notifempty
}
```

Configure the CloudWatch agent to upload the "/var/log/media-remove/output.log*" files. Restart the agent to get the 
updated config.

# Increase the Disk Space of Your Instance

If you are finding that your disk space is running low even with the media remove script running you may need to 
increase the amount of disk space your instance has. 30 GB should be enough to start but if you follow a lot of users
that post a lot of images or videos you can increase as needed.

Create a snapshot of your instance first!

## Modify the EBS Volume

Go to the AWS console and navigate to your EBS volume. Modify the volume to your new desired size. 

When the volume modification state is "optimizing" or "completed" you can proceed.

## Expand the Linux File System

After modifying the volume you must expand the file system in Linux. Follow this guide from the 
[AWS Documentation][Expand Linux Volume].

If you waited until the disk space was entirely consumed then you may get errors while expanding the file system. You
can follow this [tutorial][No Space Tutorial] to mount /tmp to a temporary file system in memory.

[Expand Linux Volume]: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/recognize-expanded-volume-linux.html
[No Space Tutorial]: https://repost.aws/knowledge-center/ebs-volume-size-increase

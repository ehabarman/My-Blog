---
title: Building Custom Desktop Notificatations for Gmail in Linux
date: 2020-08-29T19:24:43.755Z
draft: false
categories: python
tags:
  - Python Linux
author: Ehab Arman
authorImage: ""
image: uploads/gmail-email-notification.png
comments: true
share: true
type: post
---
Everyone has their preferences, and emails are no exception to this(for me at least). Unfortunately, I couldn't find a package or script to manage and customize my mail notifications. Therefore, I decided to build my own. This article explains how to build and customize a python script for Gmail notifications. I will show what you need to know and do, as for the full customization, it is up to your preferences.

# Installing required packages

##### Note: I'm using Ubuntu, so I will be using apt for installing packages.

## libnotify-bin

Libnotify is a library that sends desktop notifications to a notification daemon. We will use "**notify-send**" to display our notifications on the desktop. In order to install it, run the following command in terminal:

> `sudo apt install libnotify-bin`

## Python & Python packages

First we will need to install Python, Python version is up to you, I will be using python3:

> `sudo apt install python3`

After you have installed Python, you will need to install PIP (package management system for software packages written in Python):

> `sudo apt install python3-pip`

Finally, we will need google API to work with Gmail in python:

> `pip3 install --upgrade google-api-python-client google-auth-httplib2 google-auth-oauthlib`

# Building the script

## Gmail authorization

We need to get authorization for our script. To do that, we need to use google API:

* Go [here](https://developers.google.com/gmail/api/quickstart/python).
* Click **Enable the Gmail API** then choose **Desktop app** and click create.
* Download **credentials.json.**

The **credentials.json** contains details to get token from google API. The token will be used in getting emails from your mail box. Now use it in the following script to generate the token, which will be used to access your mail:

> ```python
> from google_auth_oauthlib.flow import InstalledAppFlow
> from google.auth.transport.requests import Request
> import pickle
> import os.path
>
> def gmail_auth():
>     credentials = None
>     token_pickle = 'path/to/token.pickle'
>     credentials_json = 'path/to/credentials.json'
>     scope = 'https://www.googleapis.com/auth/gmail.readonly' # Means that your authorization is only for reading the mail
>     if os.path.exists(token_pickle):
>         with open(token_pickle, 'rb') as token:
>             credentials = pickle.load(token)
>     # If there are no (valid) credentials available, let the user log in.
>     if not credentials or not credentials.valid:
>         if credentials and credentials.expired and credentials.refresh_token:
>             credentials.refresh(Request())
>         else:
>             flow = InstalledAppFlow.from_client_secrets_file(credentials_json, scope)
>             credentials = flow.run_local_server(port=0)
>         # Save the credentials for the next run
>         with open(token_pickle, 'wb') as token:
>             pickle.dump(credentials, token)
>     return credentials
> ```

**Note:** The first token access will require you signing in via a link. The code snippet above will store that token in token.pickle file, which means you will need to do the sign in for the first time and from their the script will auto refresh the token for you.

## Retrieving mails

Now that we have created credentials with the required authorization, we can interact with Gmail and retrieve our new emails:

>
>
> ```python
> from googleapiclient.discovery import build
>
> MAIL_SUBJECT = 'snippet'
> FOLDERS = ['INBOX']
> MESSAGES_TOTAL = "messagesTotal"
> WAIT_TIME = 30
>
> def main():
>     credentials = gmail_auth()
>     service = build('gmail', 'v1', credentials=credentials)
>     profile = service.users().getProfile(userId='me').execute()
>     prev_emails_count = profile[MESSAGES_TOTAL]
>     while True:
>         profile = service.users().getProfile(userId='me').execute()
>         current_emails_count = profile[MESSAGES_TOTAL]
>         if prev_emails_count < current_emails_count:
>             new_emails_number = current_emails_count - prev_emails_count
>             new_emails = get_mails_subjects(new_emails_number, service)
>             send_notifications(new_emails)
>         prev_emails_count = current_emails_count
>         sleep(WAIT_TIME)
>
>
> def get_mails_subjects(new_emails_count, service):
>     new_emails = []
>     mails = service.users().messages().list(userId='me', labelIds=FOLDERS, maxResults=new_emails_count).execute()
>     for mail in mails["messages"]:
>         mail_id = mail['id']
>         mail_details = service.users().messages().get(userId='me', id=mail_id).execute()
>         new_emails.append(mail_details[MAIL_SUBJECT])
>     return new_emails
>
> main()
> ```

If you noticed in the code above is getting all the new emails in the INBOX folder from the email every 30 seconds, the emails will be passed to `send_notifications(new_emails)`, which is responsible for sending the notifications to the desktop. This is not the most reliable or elegant way to do it, but it is simple enough for our purpose here. In this part, you can do some changes to match your preferences or need.

In my case, I'm interested to be notified about only emails related to my work and from specific people, therefore, I usually filter the new emails by sender. In addition, I'm getting my emails from two Gmail accounts instead of one. Lastly, the notifications that I like to see should contains basic info about the sender and subject only.

## Displaying notifications

Remember the `libnotify-bin` It is time to use it at last. Here is a very basic implementation of the `send_notifications` function:

> ```python
> ICON_PATH = '{}/gmail.png'.format(SCRIPT_DIR)
> NOTIFICATION_SOUND = '/usr/share/sounds/ubuntu/stereo/message.ogg'
> WAIT_TIME = 5
>
> def send_notifications(notifications):
>     for notification in notifications:
>         subprocess.call(['notify-send', '-i', str(ICON_PATH), '-t', str(WAIT_TIME * 1000), *[notification]])
>         subprocess.call(['paplay', NOTIFICATION_SOUND])
> ```

The code snippet will send a pop notification to desktop that will last for 5 seconds, while at the same time, it will play a small audio.

Here, you can make the changes you desire. For example, I divided the emails I'm interested in to 3 categories and assigned a different audio for each one. In addition, I added a different icon for each of them.

## Making the script into a service

At the end of the day, What we did so far is creating our notification script, but we still need to run it manually. There is more than a way to fix this. One of them it to make the script a cronjob, it will save you the trouble of running it and you can even remove the while loop and the sleep from the script since the cronjob will handle it for you.

Another way to do it is to make the script into a service in the daemon. I will go with this one, since it has a little more steps you need to be aware of. Basically you will need to add the following service file to your daemon:

`custom_notifications.service`

> ```editorconfig
> [Unit]
> Description=Gmail Notifications
> After=systemd-user-sessions.service,systemd-journald.service
>
> [Service]
> ExecStart=/path/to/script/gmail-notifications
> Environment="DISPLAY=:0" "XAUTHORITY=/home/your_user/.Xauthority"
> RemainAfterExit=yes
>
> [Install]
> WantedBy=multi-user.target
> ```

Since the notifications uses other system services like OS session & network. You will need to add them here before you start your service or it will fail at the start. Of course you can handle it in the script as well, but I prefer to keep my code clean and leave environment issue to the daemon.

If you desire to run the code periodically and get rid of the while loop and sleep from the script, then add the following file to your daemon:

`custom_notifications.timer`

> ```editorconfig
> [Unit]
> Description=Gmail Notifications timer
>
> [Timer]
> OnUnitActiveSec=30s
> OnBootSec=30s
>
> [Install]
> WantedBy=timers.target
> ```





## Summary

So far, we have discussed how to build Gmail desktop notifications. Still, we can create and add many things that suit our work or preferences, all it takes is to spare some time and dedication to make our life easier and simpler.

I hope you are inspired to build your custom notifications and add your ideas. I enjoyed working, testing and writing about this topic. If you have some interesting ideas, let me know in the comments.

Thank you so much for reading.
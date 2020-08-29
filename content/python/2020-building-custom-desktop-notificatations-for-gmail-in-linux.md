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
Everyone has their preferences, emails are no exception for this rule -for me at least-. Unfortunately, I couldn't find a package or script to manage and customize my mail notifications. Therefore, I decided to built my own. This article is to explain how to build and customize a python script for Gmail notifications.

# Installing required packages

##### **note:** I'm using Ubuntu, so I will be using **apt** for installing packages.

## libnotify-bin

libnotify is a library that sends desktop notifications to a notification daemon. We will use "**notify-send**" to display our notifications on the desktop. In order to install it, run the following command in terminal:

> `sudo apt install libnotify-bin`

## Python & Python packages

First we will need to install python, python version is up to you, I will be using python3:

> `sudo apt install python3`

After you have installed python, you will need to install PIP (package management system for software packages written in Python):

> `sudo apt install python3-pip`

Finally, we will need google api to work with gmail in python:

> `pip3 install --upgrade google-api-python-client google-auth-httplib2 google-auth-oauthlib google-cloud-pubsub`

# Building the script

## Gmail authorization

We need to get authorization for our script. To do that:

* Go to: **https://developers.google.com/gmail/api/quickstart/python**
* Click **Enable the Gmail API** then choose **Desktop app** and click create.
* Download **credentials.json**

The credentials.json contains details to get token for accessing your Gmail. Now use it in the following script to generate token.pickle, which will be used to access your mail

>

## Retrieving mails

Now that we have created credentials with the required authorization, we can interact with Gmail and retrieve our mails:

> ```
> from googleapiclient.discovery import build
>
> def main():
>     credentials = gmail_auth()
>     service = build('gmail', 'v1', credentials=credentials)
>     profile = service.users().getProfile(userId='me').execute()
>     prev_mails_count = profile[MESSAGES_TOTAL]
>     while True:
>         profile = service.users().getProfile(userId='me').execute()
>         new_mails_count = profile[MESSAGES_TOTAL]
>         if prev_mails_count < new_mails_count:
>             subjects = []
>             new_mails = new_mails_count - prev_mails_count
>             get_mails_subjects(new_mails, subjects, service)
>             send_notifications(subjects)
>         prev_mails_count = new_mails_count
>         sleep(WAIT_TIME)
>
>
> def get_mails_subjects(new_mails, subjects, service):
>     mails = service.users().messages().list(userId='me', labelIds=FOLDERS, maxResults=new_mails).execute()
>     for mail in mails["messages"]:
>         mail_id = mail['id']
>         mail_details = service.users().messages().get(userId='me', id=mail_id).execute()
>         subjects.append(mail_details[MAIL_SUBJECT])
> ```
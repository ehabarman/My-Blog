---
title: Building Custom Desktop Notificatations for Gmail in Linux
date: 2020-08-29T19:24:43.755Z
draft: true
categories: python
tags:
  - Python
author: Ehab Arman
authorImage: ""
image: uploads/gmail-email-notification.png
comments: true
share: true
type: post
---


Everyone has their preferences, emails are no exception for this rule -for me at least-. Unfortunately, I couldn't find a package or script to manage and customize my mail notifications. Therefore, I decided to built my own. This article is to explain how to build and customize a python script for Gmail notifications.

# Installing required packages

**note:** I'm using Ubuntu, so I will be using **apt** for installing packages.

* #### libnotify-bin

  libnotify is a library that sends desktop notifications to a notification daemon. We will use "**notify-send**" to display our notifications on the desktop. In order to install it, run the following command in terminal:

  > `sudo apt install libnotify-bin`
* #### P**ython & Python packages**

  First we will need to install python, python version is up to you, I will be using python3:

  > `sudo apt install python3`

  After you have installed python, you will need to install PIP (package management system for software packages written in Python):

  > `sudo apt install python3-pip`

  Finally, we will need google api to work with gmail in python:

  > `pip3 install --upgrade google-api-python-client google-auth-httplib2 google-auth-oauthlib google-cloud-pubsub`



# Building the script

* #### Gmail authorization

  We need to get authorization for our script. To do that:

  * Go to: **https://developers.google.com/gmail/api/quickstart/python**
  * Click **Enable the Gmail API** then choose **Desktop app** and click create.
  * Download **credentials.json**

  The credentials.json contains details to get token for accessing your Gmail. Now use it in the following script to generate token.pickle, which will be used to access your mail

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
* #### Retrieving mails
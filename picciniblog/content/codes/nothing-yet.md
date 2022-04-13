---
author:
  name:  "Patrick Piccini"
date: 2022-04-12T21:00:50-03:00
linktitle: Code - Email com Google Authentication
type:
- post
- posts
title: Code - Email com Google Authentication
weight: 10
series:
- Hugo 101
---

### Voce pode clonar o repositório: [email-google-auth](https://github.com/patrickpiccini/email-google-auth).

##### _GoogleAuthenticator.py_
~~~python

import os.path
from google.auth.transport.requests import Request
from google.oauth2.credentials import Credentials
from google_auth_oauthlib.flow import InstalledAppFlow
from googleapiclient.discovery import build
from googleapiclient.errors import HttpError


def authenticator(client_secret_file, api_service_name, api_version, *scopes):
    CLIENT_SECRET_FILE = client_secret_file
    API_SERVICE_NAME = api_service_name
    API_VERSION = api_version
    SCOPES = [scope for scope in scopes[0]]

    creds = None

    if os.path.exists('token.json'):
        creds = Credentials.from_authorized_user_file(
            'token.json', SCOPES)
    
    if not creds or not creds.valid:
        if creds and creds.expired and creds.refresh_token:
            creds.refresh(Request())
        else:
            flow = InstalledAppFlow.from_client_secrets_file(
                CLIENT_SECRET_FILE, SCOPES)
            creds = flow.run_local_server(port=0)
        # Save the credentials for the next run
        with open('token.json', 'w') as token:
            token.write(creds.to_json())

    try:
        # Call the Gmail API
        service = build(API_SERVICE_NAME, API_VERSION, credentials=creds)
        print(API_SERVICE_NAME, 'service created successfully')
        return service

    except HttpError as error:
        # TO DO(developer) - Handle errors from gmail API.
        print('An error occurred: {}'.format(error))
~~~

##### _SendEmail.py_
~~~python
import base64
from email import errors
from GoogleAuthenticator import authenticator
from email.mime.text import MIMEText

CLIENT_SECRET_FILE = 'DocGoogle/credentials.json'
API_NAME = 'gmail'
API_VERSION = 'v1'
SCOPES = ['https://mail.google.com/']

service = authenticator(CLIENT_SECRET_FILE, API_NAME, API_VERSION, SCOPES)

message = MIMEText('Python Mail test using API Google')
message['from'] = "your_email@gmail.com"
message['to'] = 'recipient@gmail.com'
message['subject'] = 'API Google'
raw_string = base64.urlsafe_b64encode(message.as_string())

try:
    message = service.users().messages().send(userId='me', body={'raw': raw_string}).execute()
    print ('Message Id: {}').format(message['id'])
except errors.HttpError as error:
     print ('An error occurred: {}').format(error)').format(error)

~~~
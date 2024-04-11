---
layout: post
title:  "Development for LeadsCalendar project"
date:   2024-03-04 14:27:49 +0300
---

# Development for LeadsCalendar project

Here we cover how to get started with developing for the project.

### Logic of the program

We propose for everything to be simple: backend in flask, frontend in react. Essentially the project is just stitching mutiple APIs together.

The flow of the user working with the app is as follows:
1. Login using google account
2. See the calendar of the host with the option to switch to their own calendar or display events side-by-side
3. Specify desired timeslot, press "pay button", be redirected to payment options page
4. Pay either using binance or paypal, be redirected back to the calendar
5. Both calendars now contain the paid event

### Connecting Google API

The main part of the project is of course the calendar. However the problem is twofold:

1. We need to be able to schedule a meeting for the host of the website
2. We need to schedule the same meeting for the client that pays

The problem that we run into here is that this essentially requires two sets of credentials -- one for the host and one for the client.

We propose to approach this problem with the following idea:
* For the host create a service account and pass it through an .env file into the container with the server
* Authorize all clients via google OAuth

#### Creating a service account
This passage cites [this helpful resource](https://stackoverflow.com/questions/72342943/web-application-using-google-calendar-api) for your convenience.

1. Create a service account on GCS > Create a key and download the json (as credentials.json or whatever you want)
2. In the calendar you want to access, go to Settings > Settings for my calendars > Select calendar you want to use > Go to "Access permissions for events" > + Add people and groups > Paste in your service account email > Set permissions to Make changes to events
3. Navigate to Integrate calendar and copy the calendarId (it will not be primary as when using your own account but most likely your email address)
4. Store your credentials.json in your project (only for testing)
5. Set an environment variable called GOOGLE_APPLICATION_CREDENTIALS to the path of your credentials.json. I use an .env file and load it using python-dotenv but feel free to do as you please. (ex. GOOGLE_APPLICATION_CREDENTIALS=credentials.json)
6. Store your calendarId somewhere (I stored it in config.py)
7. Build whatever process you need starting with service = build('calendar', 'v3'). The auth is done in the background using this method.

Now you can use the key to fetch the calendar:
{% highlight python %}
import config
from googleapiclient.discovery import build
from datetime import datetime

def get_calendar_events():
    service = build('calendar', 'v3')
    now = datetime.utcnow().isoformat() + 'Z'
    events_result = service.events().list(calendarId=config.MAIL_USERNAME, timeMin=now).execute()
    events = events_result.get('items', [])
    return events
{% endhighlight %}

More on the service accounts [in the official docs](https://cloud.google.com/iam/docs/service-account-overview).

####  Using OAuth to authorize users

For this there are quite a few thing needed, most of the steps are available [in the official docs](https://developers.google.com/calendar/api/quickstart/python), which I will also cite here for convenience.
1. Create google cloud project ([steps here](https://developers.google.com/workspace/guides/create-project))
2. Enable OAuth in the project in the Menu > APIs & Services > OAuth consent screen.
3. Create OAuth token by visiting Menu > APIs & Services > Credentials > Create Credentials > OAuth client ID.

Now you can use the token to perform OAuth logins for the users and fetch their calendar events:
{% highlight python %}
import datetime
import os.path

from google.auth.transport.requests import Request
from google.oauth2.credentials import Credentials
from google_auth_oauthlib.flow import InstalledAppFlow
from googleapiclient.discovery import build
from googleapiclient.errors import HttpError

# If modifying these scopes, delete the file token.json.
SCOPES = ["https://www.googleapis.com/auth/calendar.readonly"]


def main():
  """Shows basic usage of the Google Calendar API.
  Prints the start and name of the next 10 events on the user's calendar.
  """
  creds = None
  # The file token.json stores the user's access and refresh tokens, and is
  # created automatically when the authorization flow completes for the first
  # time.
  if os.path.exists("token.json"):
    creds = Credentials.from_authorized_user_file("token.json", SCOPES)
  # If there are no (valid) credentials available, let the user log in.
  if not creds or not creds.valid:
    if creds and creds.expired and creds.refresh_token:
      creds.refresh(Request())
    else:
      flow = InstalledAppFlow.from_client_secrets_file(
          "credentials.json", SCOPES
      )
      creds = flow.run_local_server(port=0)
    # Save the credentials for the next run
    with open("token.json", "w") as token:
      token.write(creds.to_json())

  try:
    service = build("calendar", "v3", credentials=creds)

    # Call the Calendar API
    now = datetime.datetime.utcnow().isoformat() + "Z"  # 'Z' indicates UTC time
    print("Getting the upcoming 10 events")
    events_result = (
        service.events()
        .list(
            calendarId="primary",
            timeMin=now,
            maxResults=10,
            singleEvents=True,
            orderBy="startTime",
        )
        .execute()
    )
    events = events_result.get("items", [])

    if not events:
      print("No upcoming events found.")
      return

    # Prints the start and name of the next 10 events
    for event in events:
      start = event["start"].get("dateTime", event["start"].get("date"))
      print(start, event["summary"])

  except HttpError as error:
    print(f"An error occurred: {error}")


if __name__ == "__main__":
  main()
{% endhighlight %}

### Connecting Binance API
This largely follows the steps of the [official docs](https://developers.binance.com/docs/binance-pay/), and cites them.

1. Create [a binance merchant account](https://merchant.binance.com/en)
2. Get the API key for the merchant account in the administrative interface
3. Now you can create orders using the API key ([see example](https://github.com/binance/binance-pay-connector-python/blob/master/examples/pay/merchant/new_order.py))

{% highlight python %}
import logging
from binance.pay.merchant import Merchant as Client
from binance.pay.lib.utils import config_logging


config_logging(logging, logging.DEBUG)

key = ""
secret = ""

client = Client(key, secret)

parameters = {
    "env": {"terminalType": "MINI_PROGRAM"},
    "merchantTradeNo": "2223",
    "orderAmount": 1.00,
    "currency": "USDT",
    "goods": {
        "goodsType": "01",
        "goodsCategory": "0000",
        "referenceGoodsId": "abc001",
        "goodsName": "apple",
        "goodsUnitAmount": {"currency": "USDT", "amount": 1.00},
    },
    "shipping": {
        "shippingName": {"firstName": "Joe", "lastName": "Don"},
        "shippingAddress": {"region": "NZ"},
    },
    "buyer": {"buyerName": {"firstName": "cz", "lastName": "zhao"}},
}

response = client.new_order(parameters)
print(response)
{% endhighlight %}


### Connecting PayPal API
The paypal integration is trickier than binance or google due the absense of any PayPal python libraries. They still have a REST API, but it is suggested to wrap that API in order to not create a mess.

1. Create a [paypal developer account](https://developer.paypal.com/home/)
2. Get client ID and client secret by [logging into the dashboard](https://developer.paypal.com/dashboard/) > Apps & Credentials > Create App

Now you can obtain temporary access tokens using the id and secret ([see more here](https://developer.paypal.com/api/rest/)):
{% highlight python %}
import requests

url = "https://api-m.sandbox.paypal.com/v1/oauth2/token"
headers = {
    "Content-Type": "application/x-www-form-urlencoded"
}
data = {
    "granttype": "clientcredentials"
}
auth = ("CLIENTID", "CLIENTSECRET")

response = requests.post(url, headers=headers, data=data, auth=auth)

responsebody = response.json()
accesstoken = responsebody["accesstoken"]
expiresin = responsebody"expires_in"

print("Access Token:", accesstoken)
print("Expires In:", expiresin)
{% endhighlight %}

Now you can use the token to create orders ([see more here](https://developer.paypal.com/docs/api/orders/v2/)):

{% highlight python %}
import requests

headers = {
    'Content-Type': 'application/json',
    'PayPal-Request-Id': '7b92603e-77ed-4896-8e78-5dea2050476a',
    'Authorization': 'Bearer 6V7rbVwmlM1gFZKW_8QtzWXqpcwQ6T5vhEGYNJDAAdn3paCgRpdeMdVYmWzgbKSsECednupJ3Zx5Xd-g',
}

data = '{ "intent": "CAPTURE", "purchase_units": [ { "reference_id": "d9f80740-38f0-11e8-b467-0ed5f89f718b", "amount": { "currency_code": "USD", "value": "100.00" } } ], "payment_source": { "paypal": { "experience_context": { "payment_method_preference": "IMMEDIATE_PAYMENT_REQUIRED", "brand_name": "EXAMPLE INC", "locale": "en-US", "landing_page": "LOGIN", "shipping_preference": "SET_PROVIDED_ADDRESS", "user_action": "PAY_NOW", "return_url": "https://example.com/returnUrl", "cancel_url": "https://example.com/cancelUrl" } } } }'

response = requests.post('https://api-m.sandbox.paypal.com/v2/checkout/orders', headers=headers, data=data)
{% endhighlight %}


### First project steps

After obtaining API tokens one should create a project:

{% highlight bash %}
npx create-react-app react-flask-app
cd react-flask-app
mkdir api
cd api
python3 -m venv venv
source venv/bin/activate
{% endhighlight %}

And the development can begin.

### Suggested Quality control

In order to better control the quality of the project, relevant tools should be installed as early as possible.

Before proceeding please [enable github actions](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository) in the repository:
* Navigate to main page of the repository
* Enable github actions in Setting > Actions > General > Actions permissions

#### Ruff

Ruff is a super-fast and conveniet linter for python. It even can autofix a lot of small issues!
Use [ruff github action](https://github.com/ChartBoost/ruff-action) to set it up.

#### Hadolint

Hadolint is a linter for Dockerfiles. It may help avoid some of the more common errors and inconsistencies.
Use [hadolint github action](https://github.com/hadolint/hadolint-action) to set it up.

#### Eslint

Eslint is an extremely popular linter for javascript code. It needs to be used in any modern project.
Use [eslint gihub action](https://github.com/marketplace/actions/action-eslint) to set it up.

### Proposed backend API endpoints

* /login -- allows to login using google oauth
* /logout -- allows to logout
* /calendar/server -- returns calendar of the host
* /calendar/client -- returns calendar of the client
* /payment -- returns payment options
* /payment/binance -- allows to pay using binance
* /payment/paypal -- allows to pay using paypal
* /event -- allows to propose an event or confirm it with payment

### Proposed UI components

React has a lot of UI libraries that one could use, so use what you are well-aquainted with. However, most of them do not include the google-calendarish component that this project requires. For that try using [react-big-calendar](https://github.com/jquense/react-big-calendar?ref=retool-blog.ghost.io).


### Dockerization

In order to dockerize the solution feel free to use something along [these lines](https://blog.miguelgrinberg.com/post/how-to-dockerize-a-react-flask-project):
{% highlight docker %}
FROM python:3.9
WORKDIR /app

COPY api/requirements.txt api/api.py api/.flaskenv ./
RUN pip install -r ./requirements.txt
ENV FLASK_ENV production

EXPOSE 5000
CMD ["gunicorn", "-b", ":5000", "api:app"]
{% endhighlight %}

{% highlight docker %}
# Build step #1: build the React front end
FROM node:16-alpine as build-step
WORKDIR /app
ENV PATH /app/node_modules/.bin:$PATH
COPY package.json yarn.lock ./
COPY ./src ./src
COPY ./public ./public
RUN yarn install
RUN yarn build

# Build step #2: build an nginx container
FROM nginx:stable-alpine
COPY --from=build-step /app/build /usr/share/nginx/html
COPY deployment/nginx.default.conf /etc/nginx/conf.d/default.conf
{% endhighlight %}





<!--{% highlight bash %}
bash be bash
{% endhighlight %}-->





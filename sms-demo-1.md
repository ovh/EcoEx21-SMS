# How to manage a simple SMS campaign using the OVHcloud SMS API
- We are using routes described in https://api.ovh.com/console/#/sms
- SMS documentation: https://docs.ovh.com/gb/en/sms/

#### 1. Loading the credentials
- Visit [https://api.ovh.com/createToken/?GET=/sms/*&POST=/sms/*&PUT=/sms/*&DELETE=/sms/*&GET=/sms](https://api.ovh.com/createToken/?GET=/sms/*&POST=/sms/*&PUT=/sms/*&DELETE=/sms/*&GET=/sms) to get your credentials.
  - You can restrict the patterns according your needs


```python
# -*- encoding: utf-8 -*-

import yaml

# application_key: xxxx
# application_secret: xxxx
# consumer_key: xxxx
#
with open("credentials.yml", 'r') as file:
    credentials = yaml.safe_load(file)
```

#### 2. Authenticating to the API
- We are using the SDK: https://github.com/ovh/python-ovh


```python
import ovh

client = ovh.Client(
    endpoint='ovh-eu',
    application_key=credentials["application_key"],
    application_secret=credentials["application_secret"],
    consumer_key=credentials["consumer_key"],
)
```

#### 3. Getting your account
https://api.ovh.com/console/#/sms#GET


```python
# List your SMS accounts
service_names = client.get('/sms')

# Use the first SMS account
service_name = service_names[0] # e.g. sms-ab987654-1
```

#### 4. Create the campaign
https://api.ovh.com/console/#/sms/%7BserviceName%7D/batches#POST
- `from` is the sender linked to your account
  - It can be a E.164 virtual number or an alphanumeric name
  - You can go to your control panel https://www.ovh.com/auth/ to manage your virtual numbers and senders
  - https://docs.ovh.com/gb/en/sms/launch_first_sms_campaign/#step-2-create-a-sender_1
- `to` is a list of E.164 number recipents
- `message` is the message sent to each receivers using `GSM-7`/`GSM 03.38` charset
- `noStop` tells if each sent SMS should NOT have a stop clause
  - The clause is appended to the message by our backend
  - The stop clause is **mandatory** for marketing campaigns
  - e.g. `STOP 36184` preceded by a newline character


```python
import json

campaign_name = "SMS BlackFriday promotions"

params = {
    "name":     campaign_name,
    "from":     "+33700000002",
    "to":       ["+33600000001"],
    "message":  "SMS promotions, -13% on each packs",
    "noStop":   False, #optional
}

batch = client.post('/sms/%s/batches' % (service_name), **params)

# Display the created batch
print(json.dumps(batch, indent=2, sort_keys=True))
```

#### 5. Waiting for batch completion
https://api.ovh.com/console/#/sms/%7BserviceName%7D/batches/%7Bid%7D#GET


```python
import time

batch_id = batch["id"]
last_status = batch["status"]

while True:
    batch = client.get('/sms/%s/batches/%s' % (service_name, batch_id))
    if batch["status"] == "COMPLETED":
        print("The batch is completed")
        break
    if batch["status"] == "FAILED":
        print("The batch is failed")
        break

    if batch["status"] != last_status:
        print("Status: %s" % batch["status"])
        last_status = batch["status"]

    print(".", end="")
    time.sleep(10) # wait 10s
```

#### 6. Show completed batch


```python
if len(batch["errors"]) == 0:
    print("No errors on receivers during the processing of the batch.\n")
else:
    print("The following receivers have not been processed:")
    for error in batch["errors"]:
        print("%s: %s" % (error["receiver"], error["message"]))



print(json.dumps(batch, indent=2, sort_keys=True))
```

#### 7. Get campaign statistics
https://api.ovh.com/console/#/sms/%7BserviceName%7D/batches/%7Bid%7D/statistics#GET


```python
statistics = client.get('/sms/%s/batches/%s/statistics' % (service_name, batch_id))

print("Estimated credits:                                 %s" % statistics["estimatedCredits"])
print("Used credits:                                      %s" % statistics["credits"])
print("Number of SMS discarded during the processing:     %d" % len(batch["errors"]))
print("Number of pending SMS:                             %d" % statistics["pending"])
print("Number of sent SMS:                                %d" % statistics["sent"])
print("Number of delivered SMS to receivers:              %d" % statistics["delivered"])
print("Number of not delivered SMS to receivers:          %d" % statistics["failed"])
print("Number of SMS that received a STOP from receivers: %d" % statistics["stoplisted"])

```

#### 8. To go furthermore, you can download a CSV containing all the details about the SMS sent.
- Ask the export using https://api.ovh.com/console/#/sms/%7BserviceName%7D/document#GET
  - You must fill the `batchID` filter
  - From the response you get the document ID
- Get your document on https://api.ovh.com/console/#/me/document/%7Bid%7D#GET
  - Poll the document every 5s until the size is greater than 0
  - Then download the CSV using the `getUrl` from the response

File content example:
```
id;date;sender;receiver;ptt;operatorCode;descriptionDlr;tag;message;sentAt;deliveredAt
10000065;2021-09-28 15:57:30;"+337xxxxxxxx";+336xxxxxxxx;0;20815;"Success";"";"SMS promotions, -13% on each packs
STOP 36184";2021-09-28 15:57:32;2021-09-28 15:57:37
```

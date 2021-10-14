# How to manage a customized SMS campaign using the OVHcloud SMS API
- We are using routes described in https://api.ovh.com/console/#/sms
- SMS documentation: https://docs.ovh.com/gb/en/sms/

#### 1. Loading the credentials
- Visit [https://api.ovh.com/createToken/?POST=/me/*&PUT=/me/*&GET=/me/*&GET=/sms/*&POST=/sms/*&PUT=/sms/*&DELETE=/sms/*&GET=/sms](https://api.ovh.com/createToken/?POST=/me/*&PUT=/me/*&GET=/me/*&GET=/sms/*&POST=/sms/*&PUT=/sms/*&DELETE=/sms/*&GET=/sms) to get your credentials.
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
# List your accounts
service_names = client.get('/sms')

# Use the first account
service_name = service_names[0] # e.g. sms-ab987654-1
```

#### 4. Create a document used to upload your CSV
https://api.ovh.com/console/#/me/document#POST


```python
campaign_name = "SMS BlackFriday promotions"
filename = "blackfriday.csv"

document = client.post('/me/document',
    name=filename,
    tags=[ #optional: you can set tags=None
        {
            "key": "sms_account",
            "value": service_name,
        },
        {

            "key": "sms_campaign",
            "value": campaign_name,
        },
    ],
)

document_id = document["id"]
print("Document created with id %s" % document_id)
```

#### 5. Set the expiration of the document
The CSV will be copied during the receivers resource creation in the next steps so it can be auto-removed after 24h.

https://api.ovh.com/console/#/me/document/%7Bid%7D#PUT


```python
# Set expiration of the document since it will copied during the receivers resource creation
from datetime import datetime
from datetime import timedelta
from datetime import timezone

expiration_date = datetime.now(timezone.utc) + timedelta(days=1)

params = {
    "expirationDate": expiration_date.astimezone().isoformat(),
}

client.put('/me/document/%s' % document_id, **params)
```

#### 6. Upload the CSV
We are using the PUT URL from the created document.


```python
import requests

url = document["putUrl"]

with open(filename) as f:
    content = f.read()

response = requests.put(url, data = content)
if not response.ok:
    print(response.text)

```

#### 7. Create a receivers from the previous document
https://api.ovh.com/console/#/sms/%7BserviceName%7D/receivers#POST

- This action performs checks on the input CSV:
  - Checks the CSV is encoded in UTF-8
  - Checks the `number` colunm contains E.164 numbers
  - Checks the custom colunms' texts are compatible with the `GSM-7`/`GSM 03.38` charset
  - etc.
- Then the CSV is copied to the slot ID in your SMS space


```python
import json

receivers = client.post('/sms/%s/receivers' % service_name,
    autoUpdate=False,
    description=campaign_name,
    documentId=document_id,
    slotId=1,
)

print(json.dumps(receivers, indent=2, sort_keys=True))
```

#### 8. Create the campaign
https://api.ovh.com/console/#/sms/%7BserviceName%7D/batches#POST
- `from` is the sender linked to your account
  - It can be a E.164 virtual number or an alphanumeric name
  - You can go to your control panel https://www.ovh.com/auth/ to manage your virtual numbers and senders
- `slotID` is the slot ID from the `receivers` object
- `message` is the template message sent to each receivers using `GSM-7`/`GSM 03.38` charset
  - The text substitution is done by defining the column name surrounded by the `#` character (e.g. `#name#`)
- `noStop` tells if each sent SMS should NOT have a stop clause
  - The clause is appended to the message by our backend
  - The stop clause is **mandatory** for marketing campaigns
  - e.g. `STOP 36184` preceded by a newline character


```python
params = {
    "name":     campaign_name,
    "from":     "+33700000001",
    "slotID":   str(receivers["slotId"]),
    "message":  "#name# here is your redeem code #redeem_code# for SMS packs",
    "noStop":   False,
}

batch = client.post('/sms/%s/batches' % (service_name), **params)

# Display the created batch
print(json.dumps(batch, indent=2, sort_keys=True))
```

#### 9. Waiting for batch completion
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

#### 10. Show completed batch


```python
if len(batch["errors"]) == 0:
    print("No errors on receivers during the processing of the batch.")
else:
    print("The following receivers have not been processed:")
    for error in batch["errors"]:
        print("%s: %s" % (error["receiver"], error["message"]))



print(json.dumps(batch, indent=2, sort_keys=True))
```

#### 11. Get campaign statistics
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

#### 12. To go furthermore, you can download a CSV containing all the details about the SMS sent.
- Ask the export using https://api.ovh.com/console/#/sms/%7BserviceName%7D/document#GET
  - You must fill the `batchID` filter
  - From the response you get the document ID
- Get your document on https://api.ovh.com/console/#/me/document/%7Bid%7D#GET
  - Poll the document every 5s until the size is greater than 0
  - Then download the CSV using the `getUrl` from the response

File content example:
```
id;date;sender;receiver;ptt;operatorCode;descriptionDlr;tag;message;sentAt;deliveredAt
10000066;2021-09-28 16:16:53;"+337xxxxxxxx";+336xxxxxxxx;0;20815;"Success";"";"John Doe here is your redeem code 1WzEys8BYDK for SMS packs
STOP 36184";2021-09-28 16:16:54;2021-09-28 16:17
```

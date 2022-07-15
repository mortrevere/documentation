# Forward Logs to SEKOIA.IO via HTTPS

To push your events to SEKOIA.IO, you can use our HTTPS log collector. It's the simpler way to send us your logs, especially for non-syslog system.

For the forwarding, two options are available:

- The safer one is to send your events as a structured JSON payload. By using JSON, you ensure to avoid serialization issues.
- The alternative is to send your events as line-oriented records.

## Push our events to SEKOIA.IO as JSON payload

To send us events, you should set `Content-Type` HTTP header to `application/JSON`.

The following fields are currently handled by SEKOIA.IO’S HTTPS log collector:

| Field         | Mandatory? | Type     | Description                                                                                            |
|---------------|------------|----------|--------------------------------------------------------------------------------------------------------|
| `intakey_key` | Yes        | String   | Intake to which you would like to push events to                                                       |
| `json`        | Yes        | String   | The actual log payload. If you want to push structured JSON logs, please send them as quoted JSON here |
| `@timestamp`  | No         | Datetime | Event date if you want to push your own date (fallback is to use the reception’s date)                 |


When pushing your event, the collector returns the event's identifier in the SEKOIA.IO detection workflow, as a JSON payload.

To push text events, one can just `POST` content to `https://intake.sekoia.io`:

```python
import requests

content = {"intake_key": "YOUR_INTAKE_KEY", "JSON": "[764008:0] info: 198.51.100.10 example.org. A IN"}
response = requests.post("https://intake.sekoia.io", JSON=content)
print(response.text) # (1)
```

1. Will print  `{"event_id": "uuid"}`

To push structured data to SEKOIA.IO, you can push your payload as quoted JSON in the `POST`ed payload:

```python
import requests
import JSON

structured_log = {"key": "value"}
content = {"intake_key": "YOUR_INTAKE_KEY", "JSON": JSON.dumps(structured_log)}
response = requests.post("https://intake.sekoia.io", JSON=content)
print(response.text) # (1)
```

1. Will print  `{"event_id": "uuid"}`

For numerous events, you can use the alternative endpoint `/batch`. This endpoint accepts a set of events:

```python
import requests

content = {"intake_key": "YOUR_INTAKE_KEY", "JSONs": ["[764008:0] info: 198.51.100.10 example.org. A IN", "[764023:0] info: 2.34.100.56 text.org. A IN"]}
response = requests.post("https://intake.sekoia.io/batch", JSON=content)
print(response.text) # (1)
```

1. Will print  `{"event_ids": ["uuid1", "uuid2"]}`

## Push our events to SEKOIA.IO as line-oriented records

If you are not able to send your events as a structured JSON payload, you can use the `plain` endpoint.

The following headers are handled by SEKOIA.IO’S HTTPS log collector:

| Header                       | Mandatory? | Type     | Description                                                                            |
|------------------------------|------------|----------|----------------------------------------------------------------------------------------|
| `X-SEKOIAIO-INTAKE-KEY`      | Yes        | String   | Intake to which you would like to push events to                                       |
| `X-SEKOIAIO-EVENT-TIMESTAMP` | No         | Datetime | Event date if you want to push your own date (fallback is to use the reception’s date) |


To push one event, just POST content to `https://intake.sekoia.io/plain`

```python
import requests

headers = {"X-SEKOIAIO-INTAKE-KEY": "YOUR_INTAKE_KEY"}
content = "[764008:0] info: 198.51.100.10 example.org. A IN"
response = requests.post("https://intake.sekoia.io/plain", data=content, headers=headers)
print(response.text) # (1)
```

1. Will print  `{"event_id": "uuid"}`

For numerous events, you can use the alternative endpoint `/batch`. The events should be separated by the line feed character (`U+000A` or `\n`):

```python
import requests

headers = {"X-SEKOIAIO-INTAKE-KEY": "YOUR_INTAKE_KEY"}
events = ["[764008:0] info: 198.51.100.10 example.org. A IN", "[764023:0] info: 2.34.100.56 text.org. A IN"]
content = "\n".join(events)
response = requests.post("https://intake.sekoia.io/plain/batch", data=content, headers=headers)
print(response.text) # (1)
```

1. Will print  `{"event_ids": ["uuid1", "uuid2"]}`
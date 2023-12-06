# Python3 Flask Instrumentation
In this doc I identify the following issues with the [OTel Python getting started docs](https://opentelemetry.io/docs/instrumentation/python/getting-started/):
- 🔔 auto-instrumentation is broken on main:latest, even for `flask<3`

After the supporting analysis of [the logs](./demo.log), there is a proposal for working getting started instructions.

# Current getting started instructions

### On macOS with Python 3.11.6
#### pip freeze
Running `pip install 'flask<3'` from a clean venv yields:
```
blinker==1.7.0
click==8.1.7
Flask==2.3.3
itsdangerous==2.1.2
Jinja2==3.1.2
MarkupSafe==2.1.3
Werkzeug==3.0.1
```

## Current setup
In the [auto-instrumentation section of the OTel Python docs](https://opentelemetry.io/docs/instrumentation/python/getting-started/#instrumentation) have setup instructions that say:
```sh
pip install opentelemetry-distro
opentelemetry-bootsrap -a install
```

### Replicating the setup instructions
#### Input
Run the server as in the docs:
```sh
OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED=true \
    opentelemetry-instrument \
    --traces_exporter console \
    --metrics_exporter console \
    --logs_exporter console \
    --service_name dice-server \
    flask run -p 8080 | demo.log
```
^ notice the output tee'd to demo.log, that will be used to quantify claims. I've included [sample logs in this repo](./demo.log), with the "* Debug mode: off" line removed so the reader can `jq < demo.log` for more detail.

#### curl a few times
in Bash:
```sh
for i in {1..3}; do curl http://localhost:8080/rolldice; done
```
or in Fish
```sh
for i in (seq 1 128); curl http://localhost:8080/rolldice; end
```

#### Output
```json
 {
    "body": "Anonymous player is rolling the dice: 5",
    "severity_number": "<SeverityNumber.WARN: 13>",
    "severity_text": "WARNING",
    "attributes": {
        "otelSpanID": "0",
        "otelTraceID": "0",
        "otelTraceSampled": false,
        "otelServiceName": "dice-server"
    },
    "dropped_attributes": 0,
    "timestamp": "2023-12-06T18:31:18.219932Z",
    "trace_id": "0x00000000000000000000000000000000",
    "span_id": "0x0000000000000000",
    "trace_flags": 0,
    "resource": "BoundedAttributes({'telemetry.sdk.language': 'python', 'telemetry.sdk.name': 'opentelemetry', 'telemetry.sdk.version': '1.21.0', 'service.name': 'dice-server', 'telemetry.auto.version': '0.42b0'}, maxlen=None)"
}
```
^ note "trace_id" and "span_id" `0x00...` with ID attrs `"0"` in the trace of the /rolldice endpoint.

notice the versions used:
- `sdk.version: 1.21.0`
- `auto.version: 0.42b0`

#### 💢 Docs claim (1): trace and span IDs will be populated

##### When tested on latest
The traces and spans are zeroed out.

##### Documentation excerpt
In the [OTel getting started docs](https://opentelemetry.io/docs/instrumentation/python/getting-started/#run-the-instrumented-app), dropping down the _expected output_ section, the docs show non-zero trace and span IDs:
```json
{
    "body": "Anonymous player is rolling the dice: 3",
    "severity_number": "<SeverityNumber.WARN: 13>",
    "severity_text": "WARNING",
    "attributes": {
        "otelSpanID": "5c2b0f851030d17d",
        "otelTraceID": "db1fc322141e64eb84f5bd8a8b1c6d1f",
        "otelServiceName": "dice-server"
    },
    "timestamp": "2023-10-10T08:14:32.631195Z",
    "trace_id": "0xdb1fc322141e64eb84f5bd8a8b1c6d1f",
    "span_id": "0x5c2b0f851030d17d",
    "trace_flags": 1,
    "resource": "BoundedAttributes({'telemetry.sdk.language': 'python', 'telemetry.sdk.name': 'opentelemetry', 'telemetry.sdk.version': '1.17.0', 'service.name': 'dice-server', 'telemetry.auto.version': '0.38b0'}, maxlen=None)"
}
```
^ notice the populated trace and span IDs. The reader may also notice the different versions of the SDK. I'm not sure if that aught to matter vis the instructions in the docs.
versions in docs:
- `sdk.version: 1.17.0`
- `auto.version: 0.38b0`

The versions in the docs are not latest, which will be installed by following `pip install opentelemetry-distro` as stated in docs.

#### 💢 Docs claim (2): http attributes will be populated

##### When tested on latest
The speified cattributes are missing entirely.

##### Documentation excerpt
The docs in the same section as (1) above also claim:
```json
    "attributes": {
        "http.method": "GET",
        "http.server_name": "127.0.0.1",
        "http.scheme": "http",
        "net.host.port": 8080,
        "http.host": "localhost:8080",
        "http.target": "/rolldice?rolls=12",
        "net.peer.ip": "127.0.0.1",
        "http.user_agent": "curl/8.1.2",
        "net.peer.port": 58419,
        "http.flavor": "1.1",
        "http.route": "/rolldice",
        "http.status_code": 200
    },
```
^ these logs should appear. However `grep "http\.method" < demo.log` yields no results.

#### 💢 Docs claim (3): request duration will be measured with a histogram

##### When tested on latest
The histogram is missing entirely.

##### Documentation excerpt
The docs claim we'll see this output, under the second _View example output_ dropdown.
```json
    "data": {
    "aggregation_temporality": 2,
    "data_points": [
        {
        "attributes": {
            "http.flavor": "1.1",
            "http.host": "localhost:5000",
            "http.method": "GET",
            "http.scheme": "http",
            "http.server_name": "127.0.0.1",
            "http.status_code": 200,
            "net.host.port": 5000
        },
        "bucket_counts": [0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0],
        "count": 1,
        "explicit_bounds": [
            0, 5, 10, 25, 50, 75, 100, 250, 500, 1000
        ],
        "max": 1,
        "min": 1,
        "start_time_unix_nano": 1666077040063027610,
        "sum": 1,
        "time_unix_nano": 1666077098181107419,
```

### Current setup docs issue summary
- Auto-instrumentation is broken, even for `flask<3`.

# Proposed getting started instructions
With a Python3 > 3.6 installation, in Fish:jh
```sh
python3 -m venv venv
source venv/bin/activate.fish
pip install 'werkzeug<3'
pip install 'flask<3'
pip install opentelemetry-distro
pip install opentelemetry-instrumentation-flask
```
^ run pip installs verbosely like this because pip does not use argument order.

Doing so yields:
```
Flask==2.3.3
Werkzeug==2.3.8
```

... proposal under construction ...

## Server
```sh
flask run -p 8080
```

## Client
```sh
curl http://localhost:8080/rolldice
```

### Expected result
You should get random numbers between 1 and 6 immediately.

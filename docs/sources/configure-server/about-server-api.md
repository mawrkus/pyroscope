---
description: Learn about the Pyrocope server API
menuTitle: Server API Overview
title: Pyroscope Server HTTP API Reference
weight: 20
---

# Pyroscope Server HTTP API Reference

Pyroscope server exposes an HTTP API for querying profiling data and ingesting profiling data from other sources.

## Authentication

Grafana Pyroscope does not include an authentication layer. Operators should use an authenticating reverse proxy for security.

In multi-tenant mode, Pyroscope requires the X-Scope-OrgID HTTP header set to a string identifying the tenant. This responsibility should be handled by the authenticating reverse proxy. For more information, refer to the [multi-tenancy documentation]({{< relref "./about-tenant-ids" >}}).

## Ingestion

There is one primary endpoint: POST /ingest. It accepts profile data in the request body and metadata as query parameters.

The following query parameters are accepted:

| Name               | Description                             | Notes                           |
|:-------------------|:----------------------------------------|:--------------------------------|
| `name`             | application name                        | required |
| `from`             | UNIX time of when the profiling started | required                        |
| `until`            | UNIX time of when the profiling stopped | required                        |
| `format`           | format of the profiling data            | optional (default is `folded`)  |
| `sampleRate`       | sample rate used in Hz                  | optional (default is 100 Hz)    |
| `spyName`          | name of the spy used                    | optional                        |
| `units`            | name of the profiling data unit         | optional (default is `samples`  |
| `aggregrationType` | type of aggregration to merge profiles  | optional (default is `sum`)     |


`name` specifies application name. For example:
```
my.awesome.app.cpu{env=staging,region=us-west-1}
```

The request body contains profiling data, and the Content-Type header may be used alongside format to determine the data format.

Some of the query parameters depend on the format of profiling data. Pyroscope currently supports three major ingestion formats.

### Text Formats

These formats handle simple ingestion of profiling data, such as `cpu` samples, and typically don't support metadata (e.g., labels) within the format. All necessary metadata is derived from query parameters, and the format is specified by the `format` query parameter.

**Supported Formats:**

- **Folded**: Also known as `collapsed`, this is the default format. Each line contains a stacktrace followed by the sample count for that stacktrace. For example:
```
foo;bar 100
foo;baz 200
```

- **Lines**: Similar to `folded`, but it represents each sample as a separate line rather than aggregating samples per stacktrace. For example:
```
foo;bar
foo;bar
foo;baz
foo;bar
```

### pprof format

The `pprof` format is a widely used binary profiling data format, particularly prevalent in the Go ecosystem.

When using this format, certain query parameters have specific behaviors:

- **format**: This should be set to `pprof`.
- **name**: This parameter contains the _prefix_ of the application name. Since a single request might include multiple profile types, the complete application name is formed by concatenating this prefix with the profile type. For instance, if you send CPU profiling data and set `name` to `my-app{}`, it will be displayed in pyroscope as `my-app.cpu{}`.
- **units**, **aggregationType**, and **sampleRate**: These parameters are ignored. The actual values are determined based on the profile types present in the data (refer to the "Sample Type Configuration" section for more details).

#### Sample Type Configuration

Pyroscope server inherently supports standard Go profile types such as `cpu`, `inuse_objects`, `inuse_space`, `alloc_objects`, and `alloc_space`. When dealing with software that generates data in pprof format, you may need to supply a custom sample type configuration for Pyroscope to interpret the data correctly.

For an example Python script to ingest a pprof file with a custom sample type configuration, see **[this Python script](https://github.com/grafana/pyroscope/tree/main/examples/api/ingest_pprof.py).**

To ingest pprof data with custom sample type configuration, modify your requests as follows:
* Set Content-Type to `multipart/form-data`.
* Upload the profile data in a form file field named `profile`.
* Include the sample type configuration in a form file field named `sample_type_config`.

A sample type configuration is a JSON object formatted like this:

```json
{
  "inuse_space": {
    "units": "bytes",
    "aggregation": "average",
    "display-name": "inuse_space_bytes",
    "sampled": false
  },
  "alloc_objects": {
    "units": "objects",
    "aggregation": "sum",
    "display-name": "alloc_objects_count",
    "sampled": true
  },
  "cpu": {
    "units": "samples",
    "aggregation": "sum",
    "display-name": "cpu_samples",
    "sampled": true
  },
  // pprof supports multiple profiles types in one file,
  //   so there can be multiple of these objects
}
```

Explanation of sample type configuration fields:

- **units**
  - Supported values: `samples`, `objects`, `bytes`
  - Description: Changes the units displayed in the frontend. `samples` = CPU samples, `objects` = objects in RAM, `bytes` = bytes in RAM.
- **display-name**
  - Supported values: Any string.
  - Description: This becomes a suffix of the app name, e.g., `my-app.inuse_space_bytes`.
- **aggregation**
  - Supported values: `sum`, `average`.
  - Description: Alters how data is aggregated on the frontend. Use `sum` for data to be summed over time (e.g., CPU samples, memory allocations), and `average` for data to be averaged over time (e.g., memory in-use objects).
- **sampled**
  - Supported values: `true`, `false`.
  - Description: Determines if the sample rate (specified in the pprof file) is considered. Set to `true` for sampled events (e.g., CPU samples), and `false` for memory profiles.

This configuration allows for customized visualization and analysis of various profile types within Pyroscope.

### Uploading a pprof File to Pyroscope Server Using profilecli

Using `profilecli`, you can easily upload your pprof files to the Pyroscope server. `profilecli` is a command-line utility that simplifies the process by allowing you to directly upload profiles with a single command. Below is a guide on how you can use `profilecli` for uploading a pprof file:

#### Prerequisites
- Ensure you have `profilecli` installed on your system.
- Have the pprof file you want to upload ready.

#### Steps to Upload

1. **Identify the pprof file and Pyroscope server details.**

   - Path to your pprof file: `path/to/your/pprof-file.pprof`
   - Pyroscope server URL: If using Grafana Cloud, for example, `https://profiles-prod-001.grafana.net`. For local instances, it could be `http://localhost:4040`.

2. **Determine Authentication Details (if applicable).**

   - If using cloud or authentication is enabled on your Pyroscope server, you will need the following:
     - Username: `<username>`
     - Password: `<password>`

3. **Specify any Extra Labels (Optional).**

   - You can add additional labels to your uploaded profile using the `--extra-labels` flag.
   - The name of the application that the profile was captured from can be provided via the `service_name` label (defaults to `profilecli-upload`).

4. **Construct and Execute the Upload Command.**

   - Here's a basic command template:
     ```
     ./profilecli upload \
         --url=<pyroscope_server_url> \
         --username=<username> \
         --password=<password> \
         --extra-labels=<label_name>=<label_value> \
         <pprof_file_path>
     ```

   - Modify the placeholders (`<pyroscope_server_url>`, `<username>`, etc.) with your actual values.

   - Example command:
     ```
     ./profilecli upload \
         --url=https://profiles-prod-001.grafana.net \
         --username=my_username \
         --password=my_password \
         path/to/your/pprof-file.pprof
     ```

   - Example command with extra labels:
     ```
     ./profilecli upload \
         --url=https://profiles-prod-001.grafana.net \
         --username=my_username \
         --password=my_password \
         --extra-labels=service_name=my_application_name \
         --extra-labels=cluster=us \
         path/to/your/pprof-file.pprof
     ```

5. **Check for Successful Upload.**

   - After running the command, you should see a confirmation message indicating a successful upload. If there are any issues, `profilecli` will provide error messages to help you troubleshoot.

#### Additional Tips

- Use the `--verbose` flag if you need more detailed logs during the upload process.
- The `--extra-labels` flag can be used multiple times to add several labels.
- For help with `profilecli` commands, you can always use `./profilecli --help`.

Using `profilecli` streamlines the process of uploading profiles to Pyroscope, making it a convenient alternative to manual HTTP requests.

### JFR format

This is the [Java Flight Recorder](https://openjdk.java.net/jeps/328) format, typically used by JVM-based profilers, also supported by our Java integration.

When this format is used, some of the query parameters behave slightly different:
* `format` should be set to `jfr`.
* `name` contains the _prefix_ of the application name. Since a single request may contain multiple profile types, the final application name is created concatenating this prefix and the profile type. For example, if you send cpu profiling data and set `name` to `my-app{}`, it will appear in pyroscope as `my-app.cpu{}`.
* `units` is ignored, and the actual units depends on the profile types available in the data.
* `aggregationType` is ignored, and the actual aggregation type depends on the profile types available in the data.

JFR ingestion support uses the profile metadata to determine which profile types are included, which depend on the kind of profiling being done. Currently supported profile types include:
* `cpu` samples, which includes only profiling data from runnable threads.
* `itimer` samples, similar to `cpu` profiling.
* `wall` samples, which includes samples from any threads independently of their state.
* `alloc_in_new_tlab_objects`, which indicates the number of new TLAB objects created.
* `alloc_in_new_tlab_bytes`, which indicates the size in bytes of new TLAB objects created.
* `alloc_outside_tlab_objects`, which indicates the number of new allocated objects outside any TLAB.
* `alloc_in_new_tlab_bytes`, which indicates the size in bytes of new allocated objects outside any TLAB.

#### JFR with labels

In order to ingest JFR data with dynamic labels, you have to make the following changes to your requests:
* use an HTTP form (`multipart/form-data`) Content-Type.
* send the JFR data in a form file field called `jfr`.
* send `LabelsSnapshot` protobuf message in a form file field called `labels`.

```protobuf
message Context {
    // string_id -> string_id
    map<int64, int64> labels = 1;
}
message LabelsSnapshot {
  // context_id -> Context
  map<int64, Context> contexts = 1;
  // string_id -> string
  map<int64, string> strings = 2;
}

```
Where `context_id` is a parameter [set in async-profiler](https://github.com/pyroscope-io/async-profiler/pull/1/files#diff-34c624b2fbf52c68fc3f15dee43a73caec11b9524319c3a581cd84ec3fd2aacfR218)

### Examples

Here's a sample code that uploads a very simple profile to pyroscope:

{{< code >}}

```curl
printf "foo;bar 100\n foo;baz 200" | curl \
-X POST \
--data-binary @- \
'http://localhost:4040/ingest?name=curl-test-app&from=1615709120&until=1615709130'

```

```python
import requests
import urllib.parse
from datetime import datetime

now = round(datetime.now().timestamp()) / 10 * 10
params = {'from': f'{now - 10}', 'name': 'python.example{foo=bar}'}

url = f'http://localhost:4040/ingest?{urllib.parse.urlencode(params)}'
data = "foo;bar 100\n" \
"foo;baz 200"

requests.post(url, data = data)
```

{{< /code >}}


Here's a sample code that uploads a JFR profile with labels to pyroscope:

{{< code >}}

```curl
curl -X POST \
  -F jfr=@profile.jfr \
  -F labels=@labels.pb  \
  "http://localhost:4040/ingest?name=curl-test-app&units=samples&aggregationType=sum&sampleRate=100&from=1655834200&until=1655834210&spyName=javaspy&format=jfr"
```

{{< /code >}}

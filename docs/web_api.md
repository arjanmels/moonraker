# API

Most API methods are supported over both the Websocket and HTTP transports.
File Transfer and "/access" requests are only available over HTTP. The
Websocket is required to receive server generated events such as gcode
responses.  For information on how to set up the Websocket, please see the
Appendix at the end of this document.

### HTTP API Overview

Moonraker's HTTP API could best be described as "RESTish".  Attempts are
made to conform to REST standards, however the dynamic nature of
Moonraker's API registration along with the desire to keep consistency
between mulitple API protocols results in an HTTP API that does not
completely adhere to the standard.

Moonraker is capable of parsing request arguments from the both the body
(either JSON or form-data depending on the `Content-Type` header) and from
the query string.  All arguments are grouped together in one data structure,
with body arguments taking precedence over query arguments.  Thus
if the same argument is supplied both in the body and in the
query string the body argument would be used. It is left up to the client
developer to decide exactly how they want to provide arguments, however
future API documention will make recommendations.  As of March 1st 2021
this document exclusively illustrates arguments via the query string.

All successful HTTP requests will return a json encoded object in the form of:

`{result: <response data>}`

Response data is generally an object itself, however for some requests this
may simply be an "ok" string.

Should a request result in an error, a standard error code along with
an error specific message is returned.

#### Query string type hints

By default all arguments passed via the query string are represented as
strings.  Most endpoint handlers know the data type for each of their
arguments, thus they can perform conversion from a string type if necessary.
However some endpoints accept arguments of a "generic" type, thus the
client is responsible for specifying the type if "string" is not desirable.
This is not a problem for websocket requests as the JSON parser can extract
the appropriate type.  HTTP requests must provide "type hints" in these
scenarios.  Moonraker supplies support for the following query string type hints:
- int
- bool
- float
- json
The `json` type hint can be specified to pass an array or an object via
the query string.  Remember to percent encode the json string so that
the query string is correctly parsed.

Type hints may be specified by post-fixing them to a key, with a ":"
separating the key and the hint.  For example, lets assume that we
have a request that takes `seconds` (integer) and `enabled` (boolean)
arguments.  The query string with type hints might look like:
```
?seconds:int=120&enabled:bool=true
```
A query string that takes a `value` argument with which we want to
assing an object, `{foo: 21.5, bar: "hello"}` might look like:
```
?value:json=%7B%22foo%22%3A21.5%2C%22bar%22%3A%22hello%22%7D
```
As you can see, a percent encoded json string is not human readable,
thus using this functionality should be seen as a "last resort."  If at
all possible clients should attempt to put these arguments in the body
of a request.

### Websocket API Overview

The Websocket API is based on JSON-RPC, an encoded request should look
something like:
```json
{
    "jsonrpc": "2.0",
    "method": "API method",
    "params": {"arg_one": 1, "arg_two": true},
    "id": 354
}
```

The `params` field may be left out if the API request takes no arguments.
The `id` should be a unique integer value that has no chance of colliding
with other JSON-RPC requests.  The `method` is the API method, as defined
for each API in this document.

A successful request will return a response like the following:
```json
{
    "jsonrpc": "2.0",
    "result": {"res_data": "success"},
    "id": 354
}
```
The `result` will generally contain an object, but as with the HTTP API in some
cases it may simply return a string.  The `id` field will return an id that
matches the one provided by the request.

Requests that result in an error will receive a properly formatted
JSON-RPC response:
```json
{
    "jsonrpc": "2.0",
    "error": {"code": 36000, "message": "Error Message"},
    "id": 354
}
```
Some errors may not return a request ID, such as an improperly formatted request.

The `test/client` folder includes a basic test interface with example usage for
most of the requests below.  It also includes a basic JSON-RPC implementation
that uses promises to return responses and errors (see json-rcp.js).

## Printer Administration

### Get Klippy host information:
- HTTP command:\
  `GET /printer/info`

- Websocket command:\
  `{jsonrpc: "2.0", method: "printer.info", id: <request id>}`

- Returns:\
  An object containing the build version, cpu info, Klippy's current state.

    ```json
    {
      state: "<klippy state>",
      state_message: "<current state message>",
      hostname: "<hostname>",
      software_version: "<version>",
      cpu_info: "<cpu_info>",
      klipper_path: "<moonraker use only>",
      python_path: "<moonraker use only>",
      log_file: "<moonraker use only>",
      config_file: "<moonraker use only>",
    }
    ```

### Emergency Stop
- HTTP command:\
  `POST /printer/emergency_stop`

- Websocket command:\
  `{jsonrpc: "2.0", method: "printer.emergency_stop", id: <request id>}`

- Returns:\
  `ok`

### Restart the host
- HTTP command:\
  `POST /printer/restart`

- Websocket command:\
  `{jsonrpc: "2.0", method: "printer.restart", id: <request id>}`

- Returns:\
  `ok`

### Restart the firmware (restarts the host and all connected MCUs)
- HTTP command:\
  `POST /printer/firmware_restart`

- Websocket command:\
  `{jsonrpc: "2.0", method: "printer.firmware_restart", id: <request id>}`

- Returns:\
  `ok`

## Printer Status

### List available printer objects:
- HTTP command:\
  `GET /printer/objects/list`

- Websocket command:\
  `{jsonrpc: "2.0", method: "printer.objects.list", id: <request id>}`

- Returns:\
  An a list of "printer objects" that are currently available for query
  or subscription.  This list will be passed in an "objects" parameter.

  ```json
  { objects: ["gcode", "toolhead", "bed_mesh", "configfile",....]}
  ```

### Query printer object status:
- HTTP command:\
  `GET /printer/objects/query?gcode`

  The above will fetch a status update for all gcode attributes.  The query
  string can contain multiple items, and specify individual attributes:

  `?gcode=gcode_position,busy&toolhead&extruder=target`

- Websocket command:\
  `{jsonrpc: "2.0", method: "printer.objects.query", params:
    {objects: {gcode: null, toolhead: ["position", "status"]}},
     id: <request id>}`

  Note that an empty array will fetch all available attributes for its key.

- Returns:\
  An object where the top level items are "eventtime" and "status".  The
  "status" item contains data about the requested update.

  ```json
  {
    eventtime: <klippy time of update>,
    status: {
      gcode: {
        busy: true,
        gcode_position: [0, 0, 0 ,0],
        ...},
      toolhead: {
        position: [0, 0, 0, 0],
        status: "Ready",
        ...},
      ...}
    }
  ```
See [printer_objects.md](printer_objects.md) for details on the printer objects
available for query.

### Subscribe to printer object status:
- HTTP command:\
  `POST /printer/objects/subscribe?connection_id=123456789&
   gcode=gcode_position,bus&extruder=target`

   Note:  The HTTP API requires that a `connection_id` is passed via the query
   string or as part of the form.   This should be the
   [ID reported](#get-websocket-id) from a currently connected websocket. A
   request that includes only the `connection_id` argument will cancel the
   subscription on the specified websocket.

- Websocket command:\
  `{jsonrpc: "2.0", method: "printer.objects.subscribe", params:
    {objects: {gcode: null, toolhead: ["position", "status"]}},
    id: <request id>}`

    Note that if `objects` is an empty object then the subscription will
    be cancelled.

- Returns:\
  Status data for objects in the request, with the format matching that of
  the `/printer/objects/query`:

  ```json
  {
    eventtime: <klippy time of update>,
    status: {
      gcode: {
        busy: true,
        gcode_position: [0, 0, 0 ,0],
        ...},
      toolhead: {
        position: [0, 0, 0, 0],
        status: "Ready",
        ...},
      ...}
    }
  ```
See [printer_objects.md](printer_objects.md) for details on the printer objects
available for subscription.

Status updates for subscribed objects are sent asynchronously over the
websocket.  See the `notify_status_update` notification for details.

### Query Endstops
- HTTP command:\
  `GET /printer/query_endstops/status`

- Websocket command:\
  `{jsonrpc: "2.0", method: "printer.query_endstops.status", id: <request id>}`

- Returns:\
  An object containing the current endstop state, with each attribute in the
  format of `endstop:<state>`, where "state" can be "open" or "TRIGGERED", for
  example:

```json
  {x: "TRIGGERED",
   y: "open",
   z: "open"}
```

### Query Server Info
- HTTP command:\
  `GET /server/info`

- Websocket command:
  `{jsonrpc: "2.0", method: "server.info", id: <request id>}`

- Returns:\
  An object containing the server's state, structured as follows:

```json
  {
    klippy_connected: <bool>,
    klippy_state: <string>,
    plugins: [<strings>]
  }
```
  Note that `klippy_state` will match the `state` value received from
  `/printer/info`. The `klippy_connected` item tracks the state of the
  connection to Klippy. The `plugins` key will return a list of all
  enabled plugins.  This can be used by clients to check if an optional
  plugin is available.

### Get Server Configuration
- HTTP command:\
  `GET /server/config`

- Websocket command:
  `{jsonrpc: "2.0", method: "server.config", id: <request id>}`

- Returns:\
  An object containing the server's configuration, structured as follows:

```json
{
    config: {
        server: {
            ...
        },
        authorization: {
            ...
        },
        ...
    }
}
```

### Fetch stored temperature data
- HTTP command:\
  `GET /server/temperature_store`

- Websocket command:
  `{jsonrpc: "2.0", method: "server.temperature_store", id: <request id>}`

- Returns:\
  An object where the keys are the available temperature sensor names, and with
  the value being an array of stored temperatures.  The array is updated every
  1 second by default, containing a total of 1200 values (20 minutes).  The
  array is organized from oldest temperature to most recent (left to right).
  Note that when the host starts each array is initialized to 0s.
  ```json
  {
      extruder: {
          temperatures: [],
          targets: [],
          powers: []
      },
      temperature_fan my_fan: {
          temperatures: [],
          targets: [],
          speeds: [],
      },
      temperature_sensor my_sensor: {
          temperatures: []
      }
  }
  ```

### Fetch stored gcode info
- HTTP command:\
  `GET /server/gcode_store`

  Optionally, a `count` argument may be added to specify the number of
  responses to fetch. If omitted, the entire gcode store will be sent
  (up to 1000 responses).

  `GET /server/gcode_store?count=100`

- Websocket command:
  `{jsonrpc: "2.0", method: "server.gcode_store", id: <request id>}`

  OR
  `{jsonrpc: "2.0", method: "server.gcode_store",
   params: {count: <integer>} id: <request id>}`

- Returns:\
  An object with the field `gcode_store` that contains an array
  of objects.  Each object will contain a `message` field and a
  `time` field:
```json
  {
    gcode_store: [
      {
        message: <string>,
        time: unix_time_stamp,
        type: <string>
      }, ...
    ]
  }
```
Each `message` field contains a gcode response received at the time
indicated in the `time` field. Note that the time stamp refers to
unix time (in seconds).  This can be used to create a JavaScript
`Date` object:
```javascript
for (let resp of result.gcode_store) {
  let date = new Date(resp.time * 1000);
  // Do something with date and resp.message ...
}
```
The `type` field will either be "command" or "response".

### Restart Server
- HTTP command:\
  `POST /server/restart`

- Websocket command:
  `{jsonrpc: "2.0", method: "server.restart", id: <request id>}`

- Returns:\
  `"ok"` upon receipt of the restart request.  After the request
  is returns, the server will restart.  Any existing connection
  will be disconnected.  A restart will result in the creation
  of a new server instance where the configuration is reloaded.

## Get Websocket ID
- HTTP command:\
  Not Available

- Websocket command:
  `{jsonrpc: "2.0", method: "server.websocket.id", id: <request id>}`

- Returns:\
  This connected websocket's unique identifer in the format shown below.
  Note that this API call is only available over the websocket.

```json
  {
    websocket_id: <int>
  }
```

## Gcode Controls

### Run a gcode:
- HTTP command:\
  `POST /printer/gcode/script?script=<gc>`

  For example,\
  `POST /printer/gcode/script?script=RESPOND MSG=Hello`\
  Will echo "Hello" to the terminal.

- Websocket command:\
  `{jsonrpc: "2.0", method: "printer.gcode.script",
    params: {script: <gc>}, id: <request id>}`

- Returns:\
  An acknowledgement that the gcode has completed execution:

  `ok`

### Get GCode Help
- HTTP command:\
  `GET /printer/gcode/help`

- Websocket command:\
  `{jsonrpc: "2.0", method: "printer.gcode.help",
    params: {script: <gc>}, id: <request id>}`

- Returns:\
  An object where they keys are gcode handlers and values are the associated
  help strings.  Note that help strings are not available for basic gcode
  handlers such as G1, G28, etc.

## Print Management

### Print a file
- HTTP command:\
  `POST /printer/print/start?filename=<file name>`

- Websocket command:\
  `{jsonrpc: "2.0", method: "printer.print.start",
    params: {filename: <file name>, id:<request id>}`

- Returns:\
  `ok` on success

### Pause a print
- HTTP command:\
  `POST /printer/print/pause`

- Websocket command:\
  `{jsonrpc: "2.0", method: "printer.print.pause", id: <request id>}`

- Returns:\
  `ok`

### Resume a print
- HTTP command:\
  `POST /printer/print/resume`

- Websocket command:\
  `{jsonrpc: "2.0", method: "printer.print.resume", id: <request id>}`

- Returns:\
  `ok`

### Cancel a print
- HTTP command:\
  `POST /printer/print/cancel`

- Websocket command:\
  `{jsonrpc: "2.0", method: "printer.print.cancel", id: <request id>}`

- Returns:\
  `ok`

## Machine Commands

### Shutdown the Operating System
- HTTP command:\
  `POST /machine/shutdown`

- Websocket command:\
  `{jsonrpc: "2.0", method: "machine.shutdown", id: <request id>}`

- Returns:\
  No return value as the server will shut down upon execution

### Reboot the Operating System
- HTTP command:\
  `POST /machine/reboot`

- Websocket command:\
  `{jsonrpc: "2.0", method: "machine.reboot", id: <request id>}`

- Returns:\
  No return value as the server will shut down upon execution

### Restart a system service
Restarts a system service via `sudo systemctl restart <name>`. Currently
only `moonraker`, `klipper`, and `webcamd` are allowed.

- HTTP command:\
  `POST /machine/services/restart?service=<service_name>`

- Websocket command:\
  `{jsonrpc: "2.0", method: "machine.services.restart",
  params: {service: "service name"}, id: <request id>}`

- Returns:\
  `ok` when complete.  Note that if `moonraker` is chosen, the return
  value will be sent prior to the restart.

### Stop a system service
Stops a system service via `sudo systemctl stop <name>`. Currently
only `webcamd` and `klipper` are allowed.

- HTTP command:\
  `POST /machine/services/stop?service=<service_name>`

- Websocket command:\
  `{jsonrpc: "2.0", method: "machine.services.stop",
  params: {service: "service name"}, id: <request id>}`

- Returns:\
  `ok` when complete

### Start a system service
Starts a system service via `sudo systemctl start <name>`. Currently
only `webcamd` and `klipper` are allowed.

- HTTP command:\
  `POST /machine/services/start?service=<service_name>`

- Websocket command:\
  `{jsonrpc: "2.0", method: "machine.services.start",
  params: {service: "service name"}, id: <request id>}`

- Returns:\
  `ok` when complete

### Get Process Info
Returns system usage information about the moonraker process.

- HTTP command:\
  `GET /machine/proc_info`

- Websocket command:\
  `{jsonrpc: "2.0", method: "machine.proc_info", id: <request id>}`

- Returns:\
  An object in the following format:
  ```json
  {
      proc_info: [
          {
              time: <system time of sample>,
              cpu_usage: <usage percent>,
              memory: <memory_used>,
              mem_units: "<kB, mB, etc>"
          },
          ...
      ],
      throttled_state: {
          bits: <throttled bits>,
          flags: ["flag1", "flag2", ...]
      }
  }
  ```
  Process information is sampled every second.  The `proc_info` field
  will return up to 30 samples, each sample with the following fields:
  - `time`: Time of the sample (in seconds since the Epoch)
  - `cpu_usage`: A floating point value between 0-100, representing the
    CPU usage of the Moonraker process.
  - `memory`: Integer value representing the current amount of memory
    allocated in RAM (resident set size).
  - `mem_units`: A string indentifying the units of the value in the
    `memory` field.  This is typically "kB", but not guaranteed.
  If the system running Moonraker supports `vcgencmd` then Moonraker
  will check the current throttled flags via `vcgencmd get_throttled`
  and report them in the `throttled_state` field:
  - `bits`: An integer value that represents the bits reported by
    `vcgencmd get_throttled`
  - `flags`: Descriptive flags parsed out of the bits.  One or more
    of the following flags may be reported:
    - "Under-Voltage Detected"
    - "Frequency Capped"
    - "Currently Throttled"
    - "Temperature Limit Active"
    - "Previously Under-Volted"
    - "Previously Frequency Capped"
    - "Previously Throttled"
    - "Previously Temperature Limited"
  The first four flags indicate an active throttling condition,
  whereas the last four indicate a previous condition (may or
  may not still be active).  If `vcgencmd` is not available the
  `throttled_state` will report `null`.

## File Operations

Most file operations are available over both APIs, however file upload,
file download, and file delete are currently only available via HTTP APIs.

Moonraker organizes different local directories into "roots".  For example,
gcodes are located at `http:\\host\server\files\gcodes\*`, otherwise known
as the "gcodes" root.  The following roots are available:
- gcodes
- config
- config_examples (read-only)
- docs (read-only)

Write operations (upload, delete, make directory, remove directory) are
only available on the `gcodes` and config roots.  Note that the `config` root
is only available if the "config_path" option has been set in Moonraker's
configuration.

### List Available Files
Walks through a directory and fetches all files.  All file names include a
path relative to the specified "root".  Note that if the query st

- HTTP command:\
  `GET /server/files/list?root=gcodes`

  If the query string is omitted then the command will return
  the "gcodes" file list by default.

- Websocket command:\
  `{jsonrpc: "2.0", method: "server.files.list", params: {root: "gcodes"}
  , id: <request id>}`

  If `params` are are omitted then the command will return the "gcodes"
  file list.

- Returns:\
  A list of objects containing file data in the following format:

```json
[
  {filename: "file name",
   size: <file_size>,
   modified: <unix_time>,
   ...]
```

### Get GCode Metadata
  Get file metadata for a specified gcode file.  If the file is located in
  a subdirectory, then the file name should include the path relative to
  the "gcodes" root.  For example, if the file is located at:\
  `http://host/server/files/gcodes/my_sub_dir/my_print.gcode`
  Then the filename should be `my_sub_dir/my_print.gcode`.

- HTTP command:\
  `GET /server/files/metadata?filename=<filename>`

- Websocket command:\
  `{jsonrpc: "2.0", method: "server.files.metadata", params: {filename: "filename"}
  , id: <request id>}`

- Returns:\
  Metadata for the requested file if it exists.  If any fields failed
  parsing they will be omitted.  The metadata will always include the file name,
  modified time, and size.

```json
  {
    filename: "file name",
    size: <file_size>,
    modified: <unix_time>,
    slicer: "Slicer Name",
    slicer_version: "<version>",
    first_layer_height: <mm>,
    first_layer_bed_temp: <C>,
    first_layer_extr_temp: <C>,
    layer_height: <mm>,
    object_height: <mm>,
    estimated_time: <time_in_seconds>,
    filament_total: <mm>,
    gcode_start_byte: <byte_location_of_first_gcode_command>,
    gcode_end_byte: <byte_location_of_last_gcode_command>,
    thumbnails: [
      {
        width: <in_pixels>,
        height: <in_pixels>,
        size: <length_of_string>,
        data: <base64_string>
      }, ...
    ]
  }
```

### Get directory information
Returns a list of files and subdirectories given a supplied path.
Unlike `/server/files/list`, this command does not walk through
subdirectories.

- HTTP command:\
  `GET /server/files/directory?path=gcodes/my_subdir&extended=true`

  If the query string is omitted then the command will return
  the "gcodes" file list by default.

- Websocket command:\
  `{jsonrpc: "2.0", method: "server.files.get_directory",
   params: {path: "gcodes/my_subdir", extended: true} ,
   id: <request id>}`

  If the "params" are omitted then the command will return
  the "gcodes" file list by default.

The `extended` argument is optional, and defaults to false. If
specified and set to true, then data returned for gcode files
will also include metadata if it is available.

- Returns:\
  An object containing file and subdirectory information in the
  following format:

```json
  {
    files: [
      {
        filename: "file name",
        size: <file_size>,
        modified: <unix_time>
      }, ...
    ],
    dirs: [
      {
        dirname: "directory name",
        modified: <unix_time>
      }
    ]
  }
```

### Make new directory
Creates a new directory at the specified path.

- HTTP command:\
  `POST /server/files/directory?path=gcodes/my_new_dir`

- Websocket command:\
  `{jsonrpc: "2.0", method: "server.files.post_directory", params:
   {path: "gcodes/my_new_dir"}, id: <request id>}`

Returns:\
`ok` if successful

### Delete directory
Deletes a directory at the specified path.

- HTTP command:\
  `DELETE /server/files/directory?path=gcodes/my_subdir`

- Websocket command:\
  `{jsonrpc: "2.0", method: "server.files.delete_directory", params:
   {path: "gcodes/my_subdir"} , id: <request id>}`

  If the specified directory contains files then the delete request
  will fail, however it is possible to "force" deletion of the directory
  and all files in it with and additional argument in the query string:\
  `DELETE /server/files/directory?path=gcodes/my_subdir&force=true`

  OR to the JSON-RPC params:\
  `{jsonrpc: "2.0", method: "get_directory", params:
   {path: "gcodes/my_subdir", force: True}, id: <request id>}`

  Note that a forced deletion will still check in with Klippy to be sure
  that a file in the requested directory is not loaded by the virtual_sdcard.

- Returns:\
`ok` if successful

### Move a file or directory
Moves a file or directory from one location to another. Note that the following
conditions must be met for a move successful move:
- The source must exist
- The source and destinations must have the same "root" directory
- The user (typically "Pi") must have the appropriate file permissions
- Neither the source nor destination can be loaded by the virtual_sdcard.
  If the source or destination is a directory, it cannot contain a file
  loaded by the virtual_sdcard.

When specifying the `source` and `dest`, the "root" directory should be
prefixed. Currently the only supported roots are "gcodes/" and "config/".

This API may also be used to rename a file or directory.   Be aware that an
attempt to rename a directory to a directory that already exists will result
in *moving* the source directory to the destination directory.

- HTTP command:\
  `POST /server/files/move?source=gcodes/my_file.gcode
  &dest=gcodes/subdir/my_file.gcode`

- Websocket command:\
  `{jsonrpc: "2.0", method: "server.files.move", params:
   {source: "gcodes/my_file.gcode",
   dest: "gcodes/subdir/my_file.gcode"}, id: <request id>}`

### Copy a file or directory
Copies a file or directory from one location to another.  A successful copy has
the pre-requesites as a move with one exception, a copy may complete if the
source file/directory is loaded by the virtual_sdcard.  As with the move API,
the source and destination should have the root prefixed.

- HTTP command:\
  `POST /server/files/copy?source=gcodes/my_file.gcode
   &dest=gcodes/subdir/my_file.gcode`

- Websocket command:\
  `{jsonrpc: "2.0", method: "server.files.copy", params:
   {source: "gcodes/my_file.gcode", dest: "gcodes/subdir/my_file.gcode"},
   id: <request id>}`

### Gcode File Download
- HTTP command:\
  `GET /server/files/gcodes/<file_name>`

- Websocket command:\
  Not Available

- Returns:\
  The requested file

### File Upload
Upload a file.  Currently files may be uploaded to the "gcodes" or "config"
root, with "gcodes" being the default location.  If one wishes to upload
to a subdirectory, the path may be added to the upload's file name
(relative to the root). If the directory does not exist an error will be
returned.  Alternatively, the "path" argument may be set, as explained
below.

- HTTP command:\
  `POST /server/files/upload`

  The file to be uploaded should be added to the FormData per the XHR spec.
  The following arguments may be added to the form:
  - root: The root location in which to upload the file.  Currently this may
    be "gcodes" or "config".  If not specified the default is "gcodes".
  - path: This argument may contain a path (relative to the root) indicating
    a subdirectory to which the file is written. If a "path" is present, the
    server will attempt to create any subdirectories that do not exist.
  Arguments available only for the "gcodes" root:
  - print: If set to "true", Klippy will attempt to start the print after
    uploading.  Note that this value should be a string type, not boolean. This
    provides compatibility with Octoprint's legacy upload API.

- Websocket command:\
  Not Available

- Returns:\
  The file name along with a successful response.
  ```json
  {'result': "file_name"}
  ```
  If the supplied root is "gcodes", a "print_started" attribute is also
   returned.
  ```json
  {'result': "file_name", 'print_started': <boolean>}
  ```

### Gcode File Delete
Delete a file in the "gcodes" root.  A relative path may be added to the file
to delete a file in a subdirectory.
- HTTP command:\
  `DELETE /server/files/gcodes/<file_name>`

- Websocket command:\
  `{jsonrpc: "2.0", method: "server.files.delete_file", params:
   {path: "gcodes/<file_name>"}, id: <request id>}`

   If the gcode file exists within a subdirectory, the relative
   path should be included in the file name.

- Returns:\
  The HTTP request returns the name of the deleted file.

### Download included config file
- HTTP command:\
  `GET /server/files/config/<file_name>`

- Websocket command:\
  Not Available

- Returns:\
  The requested file

### Delete included config file
Delete a file in the "config" root.  A relative path may be added to the file
to delete a file in a subdirectory.
- HTTP command:\
  `DELETE /server/files/config/<file_name>`

- Websocket command:\
  `{jsonrpc: "2.0", method: "server.files.delete_file", params:
   {path: "config/<file_name>}, id: <request id>}`

- Returns:\
  The HTTP request returns the name of the deleted file.

### Download a config example
- HTTP command:\
  `GET /server/files/config_examples/<file_name>`

- Websocket command:\
  Not Available

- Returns:\
  The requested file

### Download Klipper documentation
- HTTP command:\
  `GET /server/files/docs/<file_name>`

- Websocket command:\
  Not Available

- Returns:\
  The requested file

### Download klippy.log
- HTTP command:\
  `GET /server/files/klippy.log`

- Websocket command:\
  Not Available

- Returns:\
  klippy.log

### Download moonraker.log
- HTTP command:\
  `GET /server/files/moonraker.log`

- Websocket command:\
  Not Available

- Returns:\
  moonraker.log

## Authorization

Untrusted Clients must use a key to access the API by including it in the
`X-Api-Key` header for each HTTP Request.  The API below allows authorized
clients to receive and change the current API Key.

### Get the Current API Key
- HTTP command:\
  `GET /access/api_key`

- Websocket command:\
  Not Available

- Returns:\
  The current API key

### Generate a New API Key
- HTTP command:\
  `POST /access/api_key`

- Websocket command:\
  Not available

- Returns:\
  The newly generated API key.  This overwrites the previous key.  Note that
  the API key change is applied immediately, all subsequent HTTP requests
  from untrusted clients must use the new key.

### Generate a Oneshot Token

Some HTTP Requests do not expose the ability the change the headers, which is
required to apply the `X-Api-Key`.  To accomodiate these requests it a client
may ask the server for a Oneshot Token.  Tokens expire in 5 seconds and may
only be used once, making them relatively for inclusion in the query string.

- HTTP command:\
  `GET /access/oneshot_token`

- Websocket command:
  Not available

- Returns:\
  A temporary token that may be added to a requests query string for access
  to any API endpoint.  The query string should be added in the form of:
  `?token=randomly_generated_token`

## Database APIs
The following endpoints provide access to Moonraker's ldbm database.  The
database is divided into "namespaces", each client may define its own
namespace to store information.  From the client's point of view, a
namespace is an "object".  Items in the database are accessed by providing
a namespace and a key.  A key may be specifed as string, where a "." is a
delimeter to access nested objects. Alternatively the key may be specified
as an array of strings, where each string references a nested object.
This is useful for scenarios where your namespace contain keys that include
a "." character.

Moonraker reserves "moonraker" and "gcode_metadata" namespaces.  Clients may
read from these namespaces but they may not modify them.

For example, assume the following object is stored in the "superclient"
namespace:

```json
{
    settings: {
        console: {
            enable_autocomple: True
        }
    }
    theme: {
        background_color: "black"
    }
}
```
One may access the "enable_autocomplete" field by supplying `superclient` as
the "namespace" parameter and `settings.console.enable_autocomplete` or
`["settings", "console", "enable_autocomplete"]` as the "key" parameter for
the request.  The entire settings object could be accessed by providing
`settings` or `["settings"]` as the key parameter.  The entire namespace
may be read by omitting the "key" parameter, however as explained below it
is not possible to modify a namespace without specifying a key.

### List Namespaces
Lists all available namespaces.

- HTTP command:\
  `GET /server/database/list`

- Websocket command:\
  `{jsonrpc: "2.0", method: "server.database.list", id: <request id>}`

- Returns:\
  An object containing an array of namespaces in the following format:
  ```json
  {
      'namespaces': ["namespace1", "namespace2"...]
  }
  ```

### Get Database Item
Retreives an item from a specified namespace. The `key` parameter may be
omitted, in which case an object representing the entire namespace will
be returned in the `value` field.  If the `key` is provided and does not
exist in the database an error will be returned.

- HTTP command:\
  `GET /server/database/item?namespace=my_namespace&key=item.location`

- Websocket command:\
  `{jsonrpc: "2.0", method: "server.database.get_item", params:
   {namespace: "my_namespace", key: "item.location"}, id: <request id>}`



- Returns:\
  An object containing the following fields:
  ```json
  {
      'namespace': "requested_namespace",
      'key': "requested_key"
      'value': <value at key>
  }
  ```

### Add Database Item
Inserts an item into the database.  If the namespace does not exist
it will be created.  If the key specifies parent objects, all parents
will be created if they do not exist.  If the key exists it will be
overwritten with the provided `value`.  The key parameter must be provided,
it is not possible to assign a value directly to a namespace.

- HTTP command:\
  `POST /server/database/item?namespace=my_namespace&key=item.location&value:int=100`

- Websocket command:\
  `{jsonrpc: "2.0", method: "server.database.post_item", params:
   {namespace: "my_namespace", key: "item.location", value: 100},
   id: <request id>}`

- Returns:\
  An object containing the following fields:
  ```json
  {
      'namespace': "requested_namespace",
      'key': "requested_key"
      'value': <the value inserted>
  }
  ```

### Delete Database Item
Deletes an item from the database at the specified key. If the key does not
exist in the database an error will be returned.  If the deleted item results
in an empty namespace, the namespace will be removed from the database.

- HTTP command:\
  `DELETE /server/database/item?namespace=my_namespace&key=item.location`

- Websocket command:\
  `{jsonrpc: "2.0", method: "server.database.delete_item", params:
   {namespace: "my_namespace", key: "item.location"}, id: <request id>}`


- Returns:\
  An object containing the following fields:
  ```json
  {
      'namespace': "requested_namespace",
      'key': "requested_key"
      'value': <value at deleted key>
  }
  ```

## Update Manager APIs
The following endpoints are available when the `[update_manager]` plugin has
been configured:

### Get update status
Retreives the current state of each "package" available for update.  Typically
this will consist of information regarding `moonraker`, `klipper`, a `client`,
and `system` packages.  If moonraker has not yet received information from
Klipper then its status will be omitted.  If a client has not been configured
then its status will also be omitted.  One may request that the update info
be refreshed by sending a `refresh=true` argument.  Note that the refresh
parameter is ignored if an update is in progress or if a print is in progress.
In these cases the current status will be returned immediately.

- HTTP command:\
  `GET /machine/update/status?refresh=false`

  If the query string is omitted then "refresh" will default
  to false.

- Websocket command:\
  `{jsonrpc: "2.0", method: "machine.update.status",
   params: {refresh: false} , id: <request id>}`

  If the "params" are omitted then "refresh" will default to false.

- Returns:\
  Status information in the following format:
  ```json
  {
      'version_info': {
          'moonraker': {
              branch: <string>,
              remote_alias: <string>,
              version: <string>,
              remote_version: <string>,
              current_hash: <string>,
              remote_hash: <string>,
              is_valid: <bool>,
              is_dirty: <bool>,
              detached: <bool>,
              debug_enabled: <bool>
          },
          'klipper': {
              branch: <string>,
              remote_alias: <string>,
              version: <string>,
              remote_version: <string>,
              current_hash: <string>,
              remote_hash: <string>,
              is_valid: <bool>,
              is_dirty: <bool>,
              detached: <bool>,
              debug_enabled: <bool>
          },
          'client_name_1': {
              name: <string>,
              version: <string>,
              remote_version: <string>
          },
          'system': {
              package_count: <int>,
              package_list: <array>
          }
      },
      busy: false,
      github_rate_limit: <int>,
      github_requests_remaining: <int>
      github_limit_reset_time: <int>,
  }
  ```
  - The `busy` field is set to true if an update is in progress.  Moonraker
    will not allow concurrent updates.
  - The `github_rate_limit` is the maximum number of github API requests
    the user currently has.  An unathenticated user typically has 60
    requests per hour.
  - The `github_requests_remaining` is the number of API request the user
    currently has remaining.
  - The `github_limit_reset_time` is reported as seconds since the epoch.
    When this time is reached the user's limit will be reset.
  - The `moonraker` and `klipper` objects have the following fields:
    - `branch`: the name of the current git branch.  This should typically
      be "master".
    - `remote_alias`: the alias for the remote.  This should typically be
      "origin".
    - `version`:  version of the current repo on disk
    - `remote_version`: version of the latest available update
    - `current_hash`: hash of the most recent commit on disk
    - `remote_hash`: hash of the most recent commit pushed to the remote
    - `is_valid`: True if installation is a valid git repo on the master branch
      and an "origin" set to the official remote
    - `is_dirty`: True if the repo has been modified
    - `detached`: True if the repo is currently in a detached state
    - `debug_enabled`: True when "enable_repo_debug" has been configured.  This
      will bypass repo validation, allowing detached updates, and updates from
      a remote/origin other than "origin/master".
  - Multiple `client` fields may be present.  Web clients have the following
    fields:
    - `name`: Name of the configured client
    - `version`:  version of the installed client.
    - `remote_version`:  version of the latest release published to GitHub
    A `git_repo` client will have fields that match that of `klipper` and
    `moonraker`
  - The `system` object has the following fields:
    - `package_count`: The number of system packages available for update
    - `package_list`: An array containing the names of packages available
      for update

### Update Moonraker
Pulls the most recent version of Moonraker from GitHub and restarts
the service.  If "include_deps" is set to `true` an attempt will be made
to install new packages (via apt-get) and python dependencies (via pip).
Note that Moonraker uses semantic versioning to check for dependency changes
automatically, so it is generally not necessary to set `include_deps`
to `true`.  If an update is requested while a print is in progress then
this request will return an error.

- HTTP command:\
  `POST /machine/update/moonraker?include_deps=false`

  If the query string is omitted then "include_deps" will default
  to false.

- Websocket command:\
  `{jsonrpc: "2.0", method: "machine.update.moonraker",
   params: {include_deps: false}, id: <request id>}`

  If the "params" are omitted then "include_deps" will default to false.

- Returns:\
  `ok` when complete

### Update Klipper
Pulls the most recent version of Klipper from GitHub and restarts
the service.  If "include_deps" is set to `true` an attempt will be made
to install new packages (via apt-get) and python dependencies (via pip).
At the moment there is no method for automatically checking for updated
Klipper dependencies, so clients might wish to make this option available
to users via the UI. If an update is requested while a print is in progress
then this request will return an error.

- HTTP command:\
  `POST /machine/update/klipper?include_deps=false`

  If the query string is omitted then "include_deps" will default
  to false.

- Websocket command:\
  `{jsonrpc: "2.0", method: "machine.update.klipper",
   params: {include_deps: false}, id: <request id>}`

  If the "params" are omitted then "include_deps" will default to false.

- Returns:\
  `ok` when complete

### Update Client
If one more more `[update_manager client client_name]` sections have
been configured this endpoint can be used to install the most recently
published release of the client.  If an update is requested while a
print is in progress then this request will return an error.  The
`name` argument is requred, it's value should match the `client_name`
of the configured section.

- HTTP command:\
  `POST /machine/update/client?name=client_name`

- Websocket command:\
  `{jsonrpc: "2.0", method: "machine.update.client",
   params: {name: "client_name"}, id: <request id>}`

- Returns:\
  `ok` when complete

### Update System Packages
Upgrades the system packages.  Currently only `apt-get` is supported.
If an update is requested while a print is in progress then this request
will return an error.

- HTTP command:\
  `POST /machine/update/system`

- Websocket command:\
  `{jsonrpc: "2.0", method: "machine.update.system",
   id: <request id>}`

- Returns:\
  `ok` when complete

## Power APIs
The APIs below are available when the `[power]` plugin has been configured.

### Get Devices
- HTTP command:\
  `GET /machine/device_power/devices`

- Websocket command:\
  `{"jsonrpc":"2.0","method":"machine.device_power.devices","id":"1"}`

- Returns:\
  An array of objects containing info for each configured device.
  ```json
  {
    devices: [
      {
        device: <device_name>,
        status: <device_status>,
        type: <device_type>
      }, ...
    ]
  }
  ```

### Get Device Status
- HTTP command:\
  `GET /machine/device_power/status?dev_one&dev_two`

- Websocket command:\
  `{"jsonrpc":"2.0","method":"machine.device_power.status","id":"1",
    "params":{"dev_one":null, "dev_two": null}}`

- Returns:\
  An object containing status for each requested device
  ```json
  {
    dev_one: <device_status>,
    dev_two: <device_status>,
    ...
  }
  ```

### Power On Device(s)
- HTTP command:\
  `POST /machine/device_power/on?dev_one&dev_two`

- Websocket command:\
  `{"jsonrpc":"2.0","method":"machine.device_power.on","id":"1",
    "params":{"dev_one":null, "dev_two": null}}`

- Returns:\
  An object containing status for each requested device
  ```json
  {
    dev_one: <device_status>,
    dev_two: <device_status>,
    ...
  }
  ```

### Power Off Device(s)
- HTTP command:\
  `POST /machine/device_power/off?dev_one&dev_two`

- Websocket command:\
  `{"jsonrpc":"2.0","method":"machine.device_power.off","id":"1",
    "params":{"dev_one":null, "dev_two": null}}`

- Returns:\
  An object containing status for each requested device
  ```json
  {
    dev_one: <device_status>,
    dev_two: <device_status>,
    ...
  }
  ```

## Octoprint API emulation
Partial support of Octoprint API is implemented with the purpose of
allowing uploading of sliced prints to a moonraker instance.
Currently we support Slic3r derivatives and Cura with Cura-Octoprint.

### Version information
- HTTP command:\
  `GET /api/version`

- Returns:\
  An object containing simulated Octoprint version information
  ```json
  {
    server: "1.5.0",
    api: "0.1",
    text: "Octoprint (Moonraker v0.3.1-12)"
  }
  ```

### Server status
- HTTP command:\
  `GET /api/server`

- Returns:\
  An object containing simulated Octoprint server status
  ```json
  {
    server: "1.5.0",
    safemode: <None or "settings">
  }
  ```

### Login verification & User information
- HTTP command:\
  `GET /api/login`

- Returns:\
  An object containing stubbed Octoprint login/user verification
  ```json
  {
    _is_external_client: false,
    _login_mechanism: "apikey",
    name: "_api",
    active: true,
    user: true,
    admin: true,
    apikey: null,
    permissions: [],
    groups: ["admins", "users"],
  }
  ```

### Get settings
- HTTP command:\
  `GET /api/settings`

- Returns:\
  An object containing stubbed Octoprint settings.
  The webcam route is hardcoded to Fluidd/Mainsail default path.
  We say we have the UFP plugin installed so that Cura-Octoprint will
  upload in the preferred UFP format.
  ```json
  {
    plugins: {
      UltimakerFormatPackage: {
        align_inline_thumbnail: false,
        inline_thumbnail: false,
        inline_thumbnail_align_value: "left",
        inline_thumbnail_scale_value: "50",
        installed: true,
        installed_version: "0.2.2",
        scale_inline_thumbnail: false,
        state_panel_thumbnail: true
      }
    },
    feature: {
      sdSupport: false,
      temperatureGraph: false
    },
    webcam: {
      flipH: false,
      flipV: false,
      rotate90: false,
      streamUrl: "/webcam/?action=stream",
      webcamEnabled': true
    }
  }
  ```

### File Upload
- HTTP command:\
  `POST /api/files/local`
  Otherwise identical to the standard Moonraker `POST /server/files/upload` API.

### Get Job status
- HTTP command:\
  `GET /api/job`

- Returns:\
  An object containing stubbed Octoprint Job status
  ```json
  {
    job: {
      file: {name: null},
      estimatedPrintTime: null,
      filament: {length: null},
      user: None
    },
    progress: {
      completion: null,
      filepos: null,
      printTime: null,
      printTimeLeft: null,
      printTimeOrigin: null
    },
    state: <One of "Offline","Error", "Operational", "Printing", "Paused">
  }
  ```

### Get Printer status
- HTTP command:\
  `GET /api/printer`

- Returns:\
  An object containing Octoprint Printer status
  ```json
  {
    temperature: {
      <list of heater names: "bed", "tool<n>">: {
        actual: <actual temp>,
        offset: 0,
        target: <target temp>
      }
    },
    state: {
      text': state,
      flags': {
        operational: <bool>,
        paused: <bool>,
        printing: <bool>,
        cancelling: <bool>,
        pausing: False,
        error: <bool>,
        ready: <bool>,
        closedOrError: <bool>
      }
    }
  }
  ```

### Send GCode command
- HTTP command:\
  `POST /api/printer/command`

  JSON payload with parameter:
  * commands: List of GCode strings

- Returns:\
  An blank JSON object
  ```json
  {}
  ```

### List Printer profiles
- HTTP command:\
  `GET /api/printerprofiles`

- Returns:\
  An object containing simulates Octoprint Printer profile
  ```json
  {
    profiles: {
      _default: {
        id: "_default",
        name: "Default",
        color: "default",
        model: "Default",
        default': true,
        current': true,
        heatedBed: <true if "heater_bed" heater exists>,
        heatedChamber: <true if "chamber" heater exists>
      }
    }
  }
  ```

## History APIs
The APIs below are avilable when the `[history]` plugin has been configured.

### Get job list
- HTTP command:\
  `GET /server/history/list?limit=50&start=50&since=1&before=5`

- Websocket command:\
  `{"jsonrpc":"2.0","method":"server.history.list","id":"1","params":{}}`

  All arguments are optional. Arguments are as follows:
  - `before` All jobs before this UNIX timestamp
  - `limit` Number of prints to return
  - `since` All jobs after this UNIX timestamp
  - `start` Record number to start from (i.e. 10 would start at the 10th print)

- Returns an array of jobs that have been printed
  ```json
  {
      count: <number of prints>,
      jobs: [
        {
            "job_id": <unique job id>,
            "end_time": <end_time>,
            "filament_used": <filament_used>,
            "filename": <filename>,
            "metadata": {}, // Object containing metadata at time of job
            "print_duration": <print_duration>,
            "status": <status>,
            "start_time": <start_time>,
            "total_duration": <total_duration>
        },
        ...
      ]
  }
  ```

### Get a single job
- HTTP command:\
  `GET /server/history/job?uid=<id>`

- Websocket command:\
  `{"jsonrpc":"2.0","method":"server.history.get_job","id":"1",
  "params":{"uid": <id>}}`

- Returns:
  Data associated with the job ID in the following format:
  ```json
  {
      "job": {
        "job_id": <unique job id>,
        "end_time": <end_time>,
        "filament_used": <filament_used>,
        "filename": <filename>,
        "metadata": {}, // Object containing metadata at time of job
        "print_duration": <print_duration>,
        "status": <status>,
        "start_time": <start_time>,
        "total_duration": <total_duration>
      }
  }
  ```

### Delete job
- HTTP command:\
  `DELETE /server/history/job?uid=<id>`

  Note: it is possible to replace the "uid" argument with `all=true`
  to delete all jobs.

- Websocket command:\
  `{"jsonrpc":"2.0","method":"server.history.delete_job","id":"1",
  "params":{"uid": <id>}}`

  Note: it is possible to replace the "uid" argument with `"all": true`
  to delete all jobs.

- Returns an array of deleted ids
```json
[
    id1,
    id2,
    ...
]
```

## Websocket notifications
Printer generated events are sent over the websocket as JSON-RPC 2.0
notifications.  These notifications are sent to all connected clients
in the following format:

`{jsonrpc: "2.0", method: <event method name>}`

OR

`{jsonrpc: "2.0", method: <event method name>, params: [<event parameter>]}`

If a notification has parameters,  the `params` value will always be
wrapped in an array as directed by the JSON-RPC standard.  Currently
all notifications available are broadcast with either no parameters
or a single parameter.

### Gcode response:
All calls to gcode.respond() are forwarded over the websocket.  They arrive
as a "gcode_response" notification:

`{jsonrpc: "2.0", method: "notify_gcode_response", params: ["response"]}`

### Status subscriptions:
Status Subscriptions arrive as a "notify_status_update" notification:

`{jsonrpc: "2.0", method: "notify_status_update", params: [<status_data>]}`

The structure of the status data is identical to the structure that is
returned from an object query's "status" attribute.

### Klippy Ready:
Notify clients when Klippy has reported a ready state

`{jsonrpc: "2.0", method: "notify_klippy_ready"}`

### Klippy Shutdown:
Notify clients when Klippy has reported a shutdown state

`{jsonrpc: "2.0", method: "notify_klippy_shutdown"}`

### Klippy Disconnected:
Notify clients when Moonraker's connection to Klippy has terminated

`{jsonrpc: "2.0", method: "notify_klippy_disconnected"}`

### File List Changed
When a client makes a change to the virtual sdcard file list
(via upload or delete) a notification is broadcast to alert all connected
clients of the change:

`{jsonrpc: "2.0", method: "notify_filelist_changed",
 params: [<file changed info>]}`

The <file changed info> param is an object in the following format, where
the "action" is the operation that prompted the change, and the "item"
contains information about the item that has changed:

```json
{action: "<action>",
  item: {
    path: "<file or directory path>",
    root: "<root_name>",
    size: <file size>,
    modified: "<date modified>"
 }
```
Note that file move and copy actions also include a "source item" that
contains the path and root of the source file or directory.
```json
{action: "<action>",
  item: {
    path: "<file or directory path>",
    root: "<root_name>",
    size: <file size>,
    modified: "<date modified>"
 },
  source_item: {
    path: "<file or directory path>",
    root: "<root_name>"
  }
}
```

The following `actions` are currently available:
- `upload_file`
- `delete_file`
- `create_dir`
- `delete_dir`
- `move_item`
- `copy_item`

### Metadata Update
When a new file is uploaded via the API a websocket notification is broadcast
to all connected clients after parsing is complete:

`{jsonrpc: "2.0", method: "notify_metadata_update", params: [metadata]}`

Where `metadata` is an object in the following format:

```json
{
  filename: "file name",
  size: <file size>,
  modified: "last modified date",
  slicer: "Slicer Name",
  first_layer_height: <in mm>,
  layer_height: <in mm>,
  object_height: <in mm>,
  estimated_time: <time in seconds>,
  filament_total: <in mm>,
  thumbnails: [
    {
      width: <in pixels>,
      height: <in pixels>,
      size: <length of string>,
      data: <base64 string>
    }, ...
  ]
}
```

### Update Manager Responses
The update manager will send asyncronous messages to the client during an
update:

`{jsonrpc: "2.0", method: "notify_update_response", params: [response]}`

Where `response` is an object int he following format:
```json
{
    application: <string>,
    proc_id: <int>,
    message: <string>,
    complete: <boolean>
}
```
- The `application` field contains the name of application currently being
  updated.  Generally this will be either "moonraker", "klipper", "system",
  or "client".
- The `proc_id` field contains a unique id associated with the current update
  process.  This id is generated for each update request.
- The `message` field contains an asyncronous message sent during the update
  process.
- The `complete` field is set to true on the final message sent during an
  update, indicating that the update completed successfully.  Otherwise it
  will be false.

### Update Manager Refreshed
The update manager periodically auto refreshes the state of each application
it is tracking.  After an auto refresh has completed the following
notification is broadcast:

`{jsonrpc: "2.0", method: "notify_update_refreshed", params: [update_info]}`

Where `update_info` is an object that matches the response from an
[update status](#get-update-status) request.

### CPU Throttled
If the system supports throttled CPU monitoring Moonraker will send the
following notification when it detectes an active throttled condition.

`{jsonrpc: "2.0", method: "notify_cpu_throttled", params: [throttled_state]}`

Where `throtled_state` is an object that matches the `throttled_state` in the
response from a [process info](#get-process-info) request.  It is possible
for clients to receive this notification multiple times if the system
repeatedly transitions between an active and inactive throttled condition.

### History Changed
If the `[history]` module is enabled the following notification is sent when
a job is added or finished:

`{jsonrpc: "2.0", method: "notify_history_changed", params: [history_state]}`

Where `history_state` is an object in the following format:
```json
{
    "action": "added" or "finished",
    "job": <job object>
}
```
The `job` field matches the object returned when requesting
[job data](#get-a-single-job).
# Appendix

## Websocket setup
All transmissions over the websocket are done via json using the JSON-RPC 2.0
protocol.  While the websever expects a json encoded string, one limitation
of Eventlet's websocket is that it can not send string encoded frames.  Thus
the client will receive data om the server in the form of a binary Blob that
must be read using a FileReader object then decoded.

The websocket is located at `ws://host:port/websocket`, for example:
```javascript
var s = new WebSocket("ws://" + location.host + "/websocket");
```

It also should be noted that if authorization is enabled, an untrusted client
must request a "oneshot token" and add that token's value to the websocket's
query string:

```
ws://host:port/websocket?token=<32 character base32 string>
```

This is necessary as it isn't currently possible to add `X-Api-Key` to a
Websocket object's request header.

The following startup sequence is recommened for clients which make use of
the websocket:
1) Attempt to connect to `/websocket` until successful using a timer-like
   mechanism
2) Once connected, query `/printer/info` (or `printer.info`) for the ready
   status.
   - If the response returns an error (such as 404), set a timeout for
     2 seconds and try again.
   - If the response returns success, check the result's `state` attribute
     - If `state == "ready"` you may proceed to request status of printer objects
       make subscriptions, get the file list, etc.
     - If `state == "error"` then Klippy has experienced an error
       - If an error is detected it might be wise to prompt the user.  You can
         get a description of the error from the `state_message` attribute
     - If `state == "shutdown"` then Klippy is in a shutdown state.
     - If `state == "startup"` then re-request printer info in 2s.
- Repeat step 2 until Klipper reports ready.
- Client's should watch for the `notify_klippy_disconnected` event.  If it reports
  disconnected then Klippy has either been stopped or restarted.  In this
  instance the client should repeat the steps above to determine when
  klippy is ready.

## Basic Print Status
An advanced client will likely use subscriptions and notifications
to interact with Moonraker, however simple clients such as home automation
software and embedded devices (ie: ESP32) may only wish to monitor the
status of a print.  Below is a high level walkthrough for receiving print state
via polling.

- Set up a timer to poll at the desired interval.  Depending on your use
  case, 1 to 2 seconds is recommended.
- On each cycle, issue the following request:
  - `GET http://host/printer/objects/query?webhooks&virtual_sdcard&print_stats`\
    Or via json-rpc:\
    `{'jsonrpc': "2.0", 'method': "printer.objects.query", 'params':
    {'objects': {'webhooks': null, 'virtual_sdcard': null,
    'print_stats': null}}, id: <request id>}`
- If the request returns an error or the returned `result.status` is an empty
  object printer objects are not available for query.  Each queried object
  should be available in `result.status`.  The client should check to make
  sure that all objects are received before proceeding.
- Inspect `webhooks.ready`.  If the value is not "ready" the printer
  is not available.  `webhooks.message` contains a message pertaining
  to the current state.
- If the printer is ready, inspect `print_stats.state`.  It may be one
  of the following values:
  - "standby": No print in progress
  - "printing":  The printer is currently printing
  - "paused":  A print in progress has been paused
  - "error":  The print exited with an error.  `print_stats.message`
    contains a related error message
  - "complete":  The last print has completed
- If `print_stats.state` is not "standby" then `print_stats.filename`
  will report the name of the currently loaded file.
- `print_stats.filename` can be used to fetch file metadata.  It
  is only necessary to fetch metadata once per print.\
  `GET http://host/server/files/metadata?filename=<filename>`\
  Or via json-rpc:\
  `{jsonrpc: "2.0", method: "server.files.metadata",
  params: {filename: "filename"}
  , id: <request id>}`\
  If metadata extraction failed then this request will return an error.
  Some metadata fields are only populated for specific slicers, and
  unsupported slicers will only return the size and modifed date.
- There are multiple ways to calculate the ETA, this example will use
  file progress, as it is possible calculate the ETA with or without
  metadata.
  - If `metadata.estimated_time` is available, the eta calculation can
    be done as:
    ```javascript
    // assume "result" is the response from the status query
    let vsd = result.status.virtual_sdcard;
    let prog_time = vsd.progress * metadata.estimated_time;
    let eta = metadata.estimated_time - prog_time
    ```
    Alternatively, one can simply subtract the print duration from
    the estimated time:
    ```javascript
    // assume "result" is the response from the status query
    let pstats = result.status.print_status;
    let eta = metadata.estimated_time - pstats.print_duration;
    if (eta < 0)
      eta = 0;
    ```
  - If no metadata is available, print duration and progress can be used to
    calculate the ETA:
    ```javascript
    // assume "result" is the response from the status query
    let vsd = result.status.virtual_sdcard;
    let pstats = result.status.print_stats;
    let total_time = pstats.print_duration / vsd.progress;
    let eta = total_time - pstats.print_duration;
    ```
- It is possible to query additional object if a client wishes to display
  more information (ie: temperatures).  See
  [printer_objects.md](printer_objects.md) for more information.

## Bed Mesh Coordinates
The [bed_mesh](printer_objects.md#bed_mesh) printer object may be used
to generate three dimensional coordinates of a probed area (or mesh).
Below is an example (in javascript) of how to transform the data received
from a bed_mesh object query into an array of 3D coordinates.
```javascript
// assume that we have executed an object query for bed_mesh and have the
// result.  This example generates 3D coordinates for the probed matrix,
// however it would work with the mesh matrix as well
function process_mesh(result) {
  let bed_mesh = result.status.bed_mesh
  let matrix = bed_mesh.probed_matrix;
  if (!(matrix instanceof Array) ||  matrix.length < 3 ||
      !(matrix[0] instanceof Array) || matrix[0].length < 3)
      // make sure that the matrix is valid
      return;
  let coordinates = [];
  let x_distance = (bed_mesh.mesh_max[0] - bed_mesh.mesh_min[0]) /
    (matrix[0].length - 1);
  let y_distance = (bed_mesh.mesh_max[1] - bed_mesh.mesh_min[1]) /
    (matrix.length - 1);
  let x_idx = 0;
  let y_idx = 0;
  for (const x_axis of matrix) {
    x_idx = 0;
    let y_coord = bed_mesh.mesh_min[1] + (y_idx * y_distance);
    for (const z_coord of x_axis) {
      let x_coord = bed_mesh.mesh_min[0] + (x_idx * x_distance);
      x_idx++;
      coordinates.push([x_coord, y_coord, z_coord]);
    }
    y_idx++;
  }
}
// Use the array of coordinates visualize the probed area
// or mesh..
```

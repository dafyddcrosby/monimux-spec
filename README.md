# Monimux Spec .01 (20131211)
written by Dafydd Crosby

Note - until spec has reached 1.0, consider the design decisions malleable ;-)

# Design Notes

The monimux metric aggregator spec has several design goals.

* Simple, machine-readable interfaces for both input and output, allowing wholesale replacement of the program while requiring minimal (preferably none) modification of the surrounding scripts. This will allow minimal rewriting should the program need to be rewritten or reimplemented in another language.
* Interfaces that allow one to write collectors and redirectors in any language they so choose (Perl, Ruby, C, etc.)
* Avoid "magic" configuration, instead going for easily configurable (and explicit) interfaces. It should be clear where failure resides.

# Program-wide

All program input/output should be presented in Javascript Object Notation (JSON). This is chosen for its ability to be easily parsed in most modern programming languages (in many cases it comes standard with the language), as well as being both succinct and human readable. This extends to metric collection, muxed metric output, configuration, logs, and any error output.

# Program configuration

Program configuration a single JSON object in a file.

While there may be implementation-specific configuration options, there are several mandatory configuration options to implement.

* `collector_dir`: The directory where collector configurations are stored.
* `collector_interval`: The default interval (in seconds) that collectors will use when the interval is not explicitly set in the collector configuration.
* `collector_timeout`: The default amount of time (in seconds) that collectors will be given to emit their output and terminate before forced termination.
* `redirector_dir`: The directory where redirector configurations are stored.
* `redirector_socket_dir`: The directory where redirector file sockets are stored. The directory must already exist and have appropriate permissions.
* `redirector_timeout`: The amount of time between when a redirector is sent SIGTERM and SIGKILL.
* `log_path`: The path to use for log location.

An example configuration:
```json
{
  "collector_dir": "/etc/monimuxer/collectors/",
  "collector_interval": 10,
  "collector_timeout": 5,
  "redirector_dir": "/etc/monimuxer/redirectors/",
  "redirector_socket_dir": "/tmp/",
  "redirector_timeout": 5,
  "log_path": "/var/log/monimuxer.log"
}
```

Should these options not be set in the configuration file by the administrator, the defaults can be determined by the implementation.

The default location of the configuration file is implementation-specific. However, the administrator will be able to set a custom location using the command line interface.

# Command Line interface

As some languages do not implement gettext, *how* the options are called is implementation specific. However, the following options should be implemented:

* Getting the implementation version (as well as the monimux spec version). (e.g. --version)
* Setting the file location of the configuration file. (e.g. --config <file>)
* Enable debug output in logs. (e.g. --debug)
* The ability to turn off daemonizing (if the implementation daemonizes). (e.g. --foreground)
* Get a listing of program arguments (e.g. --help)

The following options are suggestions, but are not required by the implementation.

* List collectors that are available
* List redirectors that are available

# Collectors

Collector configuration is a single JSON object in a file that has a .json suffix. The basename of the configuration file is the collector name.

```json
{
  "desc": "A MySQL stats collector (note - the desc argument is optional)",
  "exec": "/opt/monimuxer/collectors/mysql.rb",
  "config": { "socket": "/tmp/mysql.socket" },
  "env": { "PATH": "/usr/bin/" },
  "interval": 20,
  "timeout": 60
}
```

The "exec" argument is required. It presents the absolute path to the script to be executed. 

The "config" argument, if present, is given as an argument to the script to be exec'd. If no "config" argument is present, an empty JSON object is given as an argument to the script to be exec'd.

The "env" argument, if present, will set the environment variables for the script to be exec'd. 

The "interval" argument, if present, sets the frequency of script execution in seconds.

The "timeout" argument, if present, sets how long, in seconds, the exec'd script is given to emit output and terminate before forced termination.

If the configuration JSON is invalid, the collector will not run. This should be logged.

The program that is used when run as exec should output the gathered metrics as a plain JSON object to STDOUT.

Error messages that prevent the collection of some or all metrics should be constructed using JSON output to STDERR.

```json
{ "msg": "Cannot connect to MySQL database" }
```

Output that is not valid JSON will be noted in the logs (it is implementation-specific as to whether the corrupt output is included in the log 'msg', though is not required)

If the collector terminates before outputting any JSON to STDERR or STDOUT, it will be noted in the logs, as this is considered a collector error.

If a collector cannot retrieve metrics and this is not considered an error, it should print an empty JSON object to STDOUT and exit with a return code of 0. This will not trigger a log write.

# Redirectors (name may be changed to improve conceptual clarity)

Redirectors are scripts that are given a UNIX socket to connect to and then, after some form of processing, redirect to some other destination. An example is a script that takes the metrics and then sends them to a Graphite server.

Redirector configuration is a single JSON object in a file that has a .json suffix. The basename of the configuration file is the collector name.

```json
{
  "exec": "/opt/monimuxer/redirectors/graphite.rb",
  "config": { "host": "graphite.example.com" },
  "timeout": 5
}
```

The "exec" argument is necessary. It is the absolute path to the redirector script.

The "config" argument, if present, is given as an argument to the script to be exec'd. There is a special variable used to give the redirector the socket called 'mm_socket'. If this is present in "config", it will be overwritten with the socket path, however the name of the variable should keep name collision to a minimum.

The "timeout" argument, if present, overrides the default setting for how long after the script has to terminate after being given SIGTERM to when it is given SIGKILL.

```bash
/opt/monimuxer/redirectors/graphite.rb "{\"mm_socket\": \"/tmp/234mert\", \"host\": \"graphite.example.com\"}"
```

If the configuration JSON is invalid, the redirector will not run. This should be logged.

The redirector will receive valid JSON. An example of the input:

```json
{
  "mysql": {
    "rows": 200000
  },
  "memcached": {
    "hits": 2343,
    "misses": 300
  }
}
```

# Logs

Log messages are output as JSON, a newline between each message.

Example:
```json
{ "lvl": "DEBUG", "msg": "Starting up monimuxer" }
{ "lvl": "INFO", "msg": "Monimuxer version n" }
{ "lvl": "INFO", "msg": "Using configuration file at /etc/monimuxer.json" }
{ "lvl": "ERROR", "collector": "mysql", "msg": "Cannot connect to database" }
{ "lvl": "ERROR", "collector": "dies_before_outputting_error", "msg": "Program exited with exit code 2, no output given." }
{ "lvl": "ERROR", "collector": "outputs_bad_json", "msg": "Program output unparseable JSON to STDOUT." }
{ "lvl": "ERROR", "redirector": "bad_config", "msg": "Invalid JSON for redirector configuration" }
```

"lvl" is the message severity level. It can be DEBUG, INFO, WARN, or ERROR.

"collector" is the name of the collector.

"redirector" is the name of the redirector.

"msg" is the message. It can be either from the collector, redirector, or main program.

The advantages of using JSON for logging is that log servers (e.g. Logstash) can easily digest these logs, and that the logs should compress fairly easily.

# Handling signals

* `SIGHUP` - Rereads main configuration file. Once the configuration file is parsed, values are changed to match the new configuration. This then results in the re-reading of collector and redirector configuration. Currently running collectors are allowed to finish collection (up to their alloted timeout).
* `SIGTERM` - allow currently running collectors to finish collection (up to their alloted timeout), followed by the killing of the redirectors (using SIGTERM, followed by SIGKILL after redirector timeout has elapsed). The program then exits returning 0
* `SIGUSR1` - rotates log file, writing current log to log_path.<timestamp>, where timestamp is an ISOxxx timestamp

# Known Unknowns and TODO's

* Redirectors need a better name, as it doesn't seem to convey that it is sending the muxed data elsewhere.
* It is unclear if the socket approach to redirectors is fully robust. There will need to be testing on this to ensure it is the right approach.
* Need to determine if setting ENV is feasible for collectors.
* Handling non-ascii (ie UTF8) output
* Need to decide on adding args to collector config. Seems to spoil the purity of "JSON everything", but it would prevent the writing of shims that do this simple action.

# Changelog

* .01 - 20131211 - Initial draft

# Configuration

## Debugging

In case a model plugin doesn't work correctly (ios, procurve, etc.), you can
enable live debugging of SSH/Telnet sessions. Just add a `debug` option
containing the value true to the `input` section. The log files will be created
depending on the parent directory of the logfile option.

The following example will log an active ssh/telnet session
`/home/oxidized/.config/oxidized/log/<IP-Address>-<PROTOCOL>`. The file will be
truncated on each consecutive ssh/telnet session, so you need to put a `tailf`
or `tail -f` on that file!

```yaml
log: /home/oxidized/.config/oxidized/log

# ...

input:
  default: ssh, telnet
  debug: true
  ssh:
    secure: false
  http:
    ssl_verify: true
```

## Privileged mode

To start privileged mode before pulling the configuration, Oxidized needs to
send the enable command. You can globally enable this, by adding the following
snippet to the global section of the configuration file.

```yaml
vars:
   enable: S3cre7
```

## Removing secrets

To strip out secrets from configurations before storing them, Oxidized needs the
`remove_secret` flag. You can globally enable this by adding the following
snippet to the global section of the configuration file.

```yaml
vars:
  remove_secret: true
```

Device models that contain substitution filters to remove sensitive data will
now be run on any fetched configuration.

As a partial example from ios.rb:

```ruby
  cmd :secret do |cfg|
    cfg.gsub! /^(snmp-server community).*/, '\\1 <configuration removed>'
    # ...
    cfg
  end
```

The above strips out snmp community strings from your saved configs.

**NOTE:** Removing secrets reduces the usefulness as a full configuration backup, but it may make sharing configs easier.

## Disabling SSH exec channels

Oxidized uses exec channels to make information extraction simpler, but there
are some situations where this doesn't work well, e.g. configuring devices. This
feature can be turned off by setting the `ssh_no_exec`
variable.

```yaml
vars:
  ssh_no_exec: true
```

## Disabling SSH keepalives

Oxidized SSH input makes use of SSH keepalives to prevent timeouts from slower
devices and to quickly tear down stale sessions in larger deployments. There
have been reports of SSH keepalives breaking compatibility with certain OS
types. They can be disabled using the `ssh_no_keepalive` variable on a per-node
basis (by specifying it in the source) or configured application-wide.

```yaml
vars:
  ssh_no_keepalive: true
```

## SSH Auth Methods

By default, Oxidized registers the following auth methods: `none`, `publickey` and `password`. However you can configure this globally, by groups, models or nodes.

```yaml
vars:
  auth_methods: [ "none", "publickey", "password", "keyboard-interactive" ]
```

## Public Key Authentication with SSH

Instead of password-based login, Oxidized can make use of key-based SSH
authentication.

You can tell Oxidized to use one or more private keys globally, or specify the
key to be used on a per-node basis. The latter can be done by mapping the
`ssh_keys` variable through the active source.

Global:

```yaml
vars:
  ssh_keys: "~/.ssh/id_rsa"
```

Per-Node:

```yaml
# ...
map:
  name: 0
  model: 1
vars_map:
  enable: 2
  ssh_keys: 3
# ...
```

If you are using a non-standard path, especially when copying the private key
via a secured channel, make sure that the permissions are set correctly:

```bash
foo@bar:~$ ls -la ~/.ssh/
total 20
drwx------ 2 oxidized oxidized 4096 Mar 13 17:03 .
drwx------ 5 oxidized oxidized 4096 Mar 13 21:40 ..
-r-------- 1 oxidized oxidized  103 Mar 13 17:03 authorized_keys
-rw------- 1 oxidized oxidized  399 Mar 13 17:02 id_ed25519
-rw-r--r-- 1 oxidized oxidized   94 Mar 13 17:02 id_ed25519.pub
```

Finally, multiple private keys can be specified as an array of file paths, such
as `["~/.ssh/id_rsa", "~/.ssh/id_another_rsa"]`.

## SSH Proxy Command

Oxidized can `ssh` through a proxy as well. To do so we just need to set
`ssh_proxy` variable with the proxy host information and optionally set the
`ssh_proxy_port` with the SSH port if it is not listening on port 22.

This can be provided on a per-node basis by mapping the proper fields from your
source.

An example for a `csv` input source that maps the 4th field as the `ssh_proxy`
value and the 5th field as `ssh_proxy_port`.

```yaml
# ...
map:
  name: 0
  model: 1
vars_map:
  enable: 2
  ssh_proxy: 3
  ssh_proxy_port: 4
# ...
```

## SSH enabling legacy algorithms

When connecting to older firmware over SSH, it is sometimes necessary to enable
legacy/disabled settings like KexAlgorithms, HostKeyAlgorithms, MAC or the
Encryption.

These settings can be provided on a per-node basis by mapping the ssh_kex,
ssh_host_key, ssh_hmac and the ssh_encryption fields from you source.

```yaml
# ...
map:
  name: 0
  model: 1
vars_map:
  enable: 2
  ssh_kex: 3
  ssh_host_key: 4
  ssh_hmac: 5
  ssh_encryption: 6
# ...
```

## FTP Passive Mode

Oxidized uses ftp passive mode by default. Some devices require passive mode to
be disabled. To do so, we can set `input.ftp.passive` to false - this will make
use of FTP active mode.

```yaml
input:
  ftp:
    passive: false
```

## Advanced Configuration

Below is an advanced example configuration.

You will be able to (optionally) override options per device.
The router.db format used is `hostname:model:username:password:enable_password`.
Hostname and model will be the only required options, all others override the
global configuration sections.

Custom model names can be mapped to an oxidized model name with a string or
a regular expression.


```yaml
---
username: oxidized
password: S3cr3tx
model: junos
interval: 3600 #interval in seconds
log: ~/.config/oxidized/log
debug: false
threads: 30 # maximum number of threads
# use_max_threads:
# false - the number of threads is selected automatically based on the interval option, but not more than the maximum
# true - always use the maximum number of threads
use_max_threads: false
timeout: 20
retries: 3
prompt: !ruby/regexp /^([\w.@-]+[#>]\s?)$/
crash:
  directory: ~/.config/oxidized/crashes
  hostnames: false
vars:
  enable: S3cr3tx
groups: {}
extensions:
  oxidized-web:
    load: true
    # Bind to any IPv4 interface
    listen: 0.0.0.0
    # Bind to port 8888 (default)
    port: 8888
    # Prefix prod to the URL, so http://oxidized.full.domain/prod/
    url_prefix: prod
    # virtual hosts to listen to (others will be denied)
    vhosts:
      - localhost
      - 127.0.0.1
      - oxidized
      - oxidized.full.domain
pid: ~/.config/oxidized/oxidized.pid
input:
  default: ssh, telnet
  debug: false
  ssh:
    secure: false
output:
  default: git
  git:
      user: Oxidized
      email: oxidized@example.com
      repo: "~/.config/oxidized/oxidized.git"
source:
  default: csv
  csv:
    file: ~/.config/oxidized/router.db
    delimiter: !ruby/regexp /:/
    map:
      name: 0
      model: 1
      username: 2
      password: 3
    vars_map:
      enable: 4
model_map:
  cisco: ios
  juniper: junos
  !ruby/regexp /procurve/: procurve
```

## Advanced Group Configuration

For group specific credentials

```yaml
groups:
  mikrotik:
    username: admin
    password: blank
  ubiquiti:
    username: ubnt
    password: ubnt
```

Model specific variables/credentials within groups

```yaml
groups:
  foo:
    models:
      arista:
        username: admin
        password: password
        vars:
          ssh_keys: "~/.ssh/id_rsa_foo_arista"
      vyatta:
        vars:
          ssh_keys: "~/.ssh/id_rsa_foo_vyatta"
  bar:
    models:
      routeros:
        vars:
          ssh_keys: "~/.ssh/id_rsa_bar_routeros"
      vyatta:
        username: admin
        password: pass
        vars:
          ssh_keys: "~/.ssh/id_rsa_bar_vyatta"
```

For mapping multiple group values to a common name, you can use strings and
regular expressions:

```yaml
group_map:
  alias1: groupA
  alias2: groupA
  alias3: groupB
  alias4: groupB
  !ruby/regexp /specialgroup/: groupS
  aliasN: groupZ
  # ...
```

add group mapping to a source

```yaml
source:
  # ...
  <source>:
    # ...
    map:
      model: 0
      name: 1
      group: 2
```

For model specific credentials

You can add 'username: nil' if the device only expects a Password at prompt.

```yaml
models:
  junos:
    username: admin
    password: password
  ironware:
    username: admin
    password: password
    vars:
      enable: enablepassword
  apc_aos:
    username: apc
    password: password
  cisco:
    username: nil
    password: pass
```

### Options (credentials, vars, etc.) precedence:
From least to most important:
- global options
- model specific options
- group specific options
- model specific options in groups
- options defined on single nodes

More important options overwrite less important ones if they are set.

## oxidized-web: RESTful API and web interface

The RESTful API and web interface are enabled by installing the `oxidized-web`
gem and configuring the `extensions.oxidized-web:` section in the configuration
file. You can set the following parameter:
- `load`: `true`/`false`: Enables or disables the `oxidized-web` extension
  (default: `false`)
- `listen`: Specifies the interface to bind to (default: `127.0.0.1`). Valid
  options:
  - `127.0.0.1`: Allows IPv4 connections from localhost only
  - `'[::1]'`: Allows IPv6 connections from localhost only
  - `<IPv4-Address>` or `'[<IPv6-Address>]'`: Binds to a specific interface
  - `0.0.0.0`: Binds to any IPv4 interface
  - `'[::]'`:  Binds to any IPv4 and IPv6 interface
- `port`: Specifies the TCP port to listen to (default: `8888`)
- `url_prefix`: Defines a URL prefix (default: no prefix)
- `vhosts`: A list of virtual hosts to listen to. If not specified, it will
  respond to any virtual host.

> [!NOTE]
> The old syntax `rest: 127.0.0.1:8888/prefix` is still supported but
> deprecated. It produces a warning and won't be suported in future releases.
>
> If the `rest` configuration is used, the extensions.oxidized-web will be
> ignored.

> [!NOTE]
> You need oxidized-web version 0.16.0 or later to use the
> extentions.oxidized-web configuration


```yaml
# Listen on http://[::1]:8888/
extensions:
  oxidized-web:
    load: true
    listen: '[::1]'
    port: 8888
```

```yaml
# Listen on http://127.0.0.1:8888/
extensions:
  oxidized-web:
    load: true
    listen: 127.0.0.1
    port: 8888
```

```yaml
# Listen on http://[2001:db8:0:face:b001:0:dead:beaf]:8888/oxidized/
extensions:
  oxidized-web:
    load: true
    listen: '[2001:db8:0:face:b001:0:dead:beaf]'
    port: 8888
    url_prefix: oxidized
```

```yaml
# Listen on http://10.0.0.1:8000/oxidized/
extensions:
  oxidized-web:
    load: true
    listen: 10.0.0.1
    port: 8000
    url_prefix: oxidized
```

```yaml
# Listen on any interface to http://oxidized.rocks:8888 and
# http://oxidized:8888
extensions:
  oxidized-web:
    load: true
    listen: '[::]'
    url_prefix: oxidized
    vhosts:
     - oxidized.rocks
     - oxidized
```

## Triggered backups

A node can be moved to head-of-queue via the REST API `GET/PUT
/node/next/[NODE]`. This can be useful to immediately schedule a fetch of the
configuration after some other event such as a syslog message indicating a
configuration update on the device.

In the default configuration this node will be processed when the next job
worker becomes available, it could take some time if existing backups are in
progress. To execute moved jobs immediately a new job can be added
automatically:

```yaml
next_adds_job: true
```

This will allow for a more timely fetch of the device configuration.

## Disabling DNS resolution

In some instances it might not be desirable to attempt to resolve names of
nodes. One such use case is when nodes are accessed through an SSH proxy, where
the remote end resolves the names differently than the host on which Oxidized
runs would.

Names can instead be passed verbatim to the input:

```yaml
resolve_dns: false
```

## Environment variables

You can use some environment variables to change default root directories values.

* `OXIDIZED_HOME` may be used to set oxidized configuration directory, which defaults to `~/.config/oxidized`
* `OXIDIZED_LOGS` may be used to set oxidzied logs and crash directories root, which default to `~/.config/oxidized`

## Logging
Oxidized supports parallel logging to different systems (appenders). The
following appenders are currently supported:
- `stderr`: log to standard error (this is the default)
- `stdout`: log to standard output
- `file`: log to a file
- `syslog`: log to syslog

> `stderr` and `stdout` are mutually exclusive and will produce a warning if used
> simultaneously.

> Note: `syslog` currently produces two timestamps because of an issue in
> [Sematic Logger](https://github.com/reidmorrison/semantic_logger/issues/316).

> You can configure as many file appenders as you wish.

You can set a log level globally and/or for each appender.
- The global log level will limit which log messages are accepted, depending
  on their level.
  - The default global log level is `:info`.
  - If you set `debug: true` in the configuration, the global log level will be
    forced to `:debug`.
- The appender log level limits which log messages are displayed by the
  appender, depending on their level.
  - The default is `:trace`.


> Available log levels: `:trace`, `:debug`, `:info`, `:warn`,
> `:error` and `:fatal`

Here is a configuration example logging `:error` to syslog, `:warn` to stdout
and `:info` to `~/.config/oxidized/info.log`:

```yaml
logger:
  # Default level
  # level: :info
  appenders:
    - type: syslog
      level: :error
    - type: stdout
      level: :warn
    - type: file
      # Default level is :trace, so we get the logs in the default level (:info)
      file: ~/.config/oxidized/info.log
```

If you want to log :trace to a file and `:info` to stdout, you must set the
global log level to `:trace`, and limit the stdout appender to `:info`:

```yaml
logger:
  level: :trace
  appenders:
    - type: stdout
      level: :info
    - type: file
      file: ~/.config/oxidized/trace.log
```

### Change log level
You can change the global log level of oxidized by sending a SIGUSR2 to
the process:
```
kill -SIGUSR2 424242
```
It will rotate between the log levels and log a warning with the new level
(you won't see the warning when the log level is `:fatal` or `:error`):
```
2025-06-30 15:25:27.972881 W [109750:2640] SemanticLogger -- Changed global default log level to :warn
```

If you specified a log level for an appender, this log level won't be
changed.

> :warning: **Warning** This currently does not work when oxidized-web is used
> and will kill the whole oxidized application. This will be corrected in a
> future release of oxidized-web.

### Dump running threads
With the SIGTTIN signal, oxidized will log a backtrace for each of its threads.
```
kill -SIGTTIN 424242
```

The threads used to fetch the configs are named `Oxidized::Job 'hostname'`:

```
2025-06-30 15:32:22.293047 W [110549:2640 core.rb:76] Thread Dump -- Backtrace:
/home/xxx/oxidized/lib/oxidized/core.rb:76:in `sleep'
/home/xxx/oxidized/lib/oxidized/core.rb:76:in `block in run'
(...)
2025-06-30 15:32:22.293409 W [110549:Oxidized::Job 'host2' ssh.rb:127] Thread Dump -- Backtrace:
/home/xxx/oxidized/lib/oxidized/input/ssh.rb:127:in `sleep'
/home/xxx/oxidized/lib/oxidized/input/ssh.rb:127:in `block (2 levels) in expect'
```

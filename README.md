# HAProxy Basics

## Install

On Debian-based systems:

```
apt-get install haproxy 
```

On macOS:

```
brew install haproxy
```

## Configuration File

Create a valid `haproxy.cfg` file on your local machine. For example:

```
mkdir -p ~/haproxy
```

```
cat <<EOF > ~/haproxy/haproxy.cfg
global
    log 127.0.0.1:514  local0  info

defaults
    log     global
    mode    http
    timeout connect 5000ms
    timeout client  50000ms
    timeout server  50000ms

frontend myfrontend
  bind *:80
  default_backend myservers

backend myservers
  server server1 127.0.0.1:1080 check
EOF
```

This is set up to forward HTTP requests from port 80 to a backend server running on port 1080. The following configuration details are defined:

### Global 

`log 127.0.0.1:514  local0  info`: Instructs HAProxy to send logs to the Syslog server listening at 127.0.0.1:514. Messages are sent with facility local0, which is one of the standard, user-defined Syslog facilities.

`local0`: This specifies the syslog facility to be used for the logs. Syslog facilities are used to separate logs from different sources. Common facilities include `local0` to `local7`, `auth`, `cron`, `daemon`, `kern`, and `user`. Using `local0` ensures that HAProxy logs are isolated from other logs, making them easier to filter and analyze.

### Defaults

`log global`: Uses the global logging settings defined in the global section.

### Frontend

`mode http`: Sets the operating mode to HTTP, indicating that HAProxy will process HTTP traffic.

`timeout connect 5000ms`: Sets the maximum time to wait for a connection to a backend server to 5000 milliseconds (5 seconds).

`timeout client 50000ms`: Sets the maximum inactivity time on the client side to 50000 milliseconds (50 seconds).

`timeout server 50000ms`: Sets the maximum inactivity time on the server side to 50000 milliseconds (50 seconds).

`bind *:80`: Binds HAProxy to listen on all network interfaces on port 80 for incoming HTTP requests.

`default_backend myservers`: Forwards all incoming requests to the `myservers` backend.

### Backend

`server server1 127.0.0.1:1080 check`: Defines a single server named `server1` with IP address 127.0.0.1 and port 1080. The `check` option enables health checks to ensure the server is available.

## Logs

To setup basic logging for HAProxy, install rsyslog:

On Debian-based systems:

```
apt-get install rsyslog
```

On macOS:

```
brew install rsyslog
```

Update the config file in `/usr/local/etc/rsyslog.conf`:

```
# Load necessary modules
module(load="imudp")  # for UDP syslog reception
module(load="imtcp")  # for TCP syslog reception

# Enable UDP listener on port 514
input(type="imudp" port="514")

# Enable TCP listener on port 514
input(type="imtcp" port="514")

# Define a log file for all incoming messages
*.* /var/log/remote-syslog.log

# Provide the local host with permissions to log remotely
$AllowedSender UDP, 127.0.0.1
$AllowedSender TCP, 127.0.0.1

# Define a log file for HAProxy specific log messages
local0.* /var/log/haproxy.log
```

Start rsyslog:

```
sudo /usr/local/opt/rsyslog/sbin/rsyslogd -n -f /usr/local/etc/rsyslog.conf -i /usr/local/var/run/rsyslogd.pid
```

Verify port 514 is open for syslog messages:

```
netstat -an | grep 514   
tcp4       0      0  *.514                  *.*                    LISTEN     
tcp6       0      0  *.514                  *.*                    LISTEN     
udp4       0      0  *.514                  *.*                               
udp6       0      0  *.514                  *.*                               
```

## Start HAProxy

To run haproxy as a service:

```
brew services start haproxy
```

Or, if you don't want/need a background service you can just run:

```
/usr/local/opt/haproxy/bin/haproxy -f ~/haproxy/haproxy.cfg
```

## Test HAProxy response

To test the response from HAProxy through port 80, send some light traffic:

Light Traffic

```
for i in $(seq 1 100); do curl -s -o /dev/null "http://localhost"; done
```

Or a single request:

```
curl -v http://localhost
```

Response

```
*   Trying [::1]:80...
* connect to ::1 port 80 failed: Connection refused
*   Trying 127.0.0.1:80...
* Connected to localhost (127.0.0.1) port 80
> GET / HTTP/1.1
> Host: localhost
> User-Agent: curl/8.4.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< access-control-allow-headers: Content-Type, api_key, Authorization
< date: Sat, 22 Jun 2024 17:33:14 GMT
< access-control-allow-methods: GET, POST, DELETE, PUT, PATCH, OPTIONS
< access-control-allow-origin: https://editor.swagger.io
< content-length: 17
< 
{ "rsp": "ok" }

* Connection #0 to host localhost left intact
```

## View Log

You may need to manually create the log file in:

```
sudo touch /var/log/haproxy.log
```

Then try to tail the file:

```
tail -f /var/log/haproxy.log
```
 
Log Format (default:)

```
2024-06-22T13:53:21-07:00 localhost haproxy[4452]: Connect from 127.0.0.1:50408 to 127.0.0.1:80 (myfrontend/HTTP)
```

## Links

https://www.haproxy.com/blog/haproxy-configuration-basics-load-balance-your-servers

https://www.haproxy.com/blog/introduction-to-haproxy-logging

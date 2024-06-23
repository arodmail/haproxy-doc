# HAProxy Basics

This guide sets up HAProxy on a local system, in a basic way, to get started quickly with a simple configuration. In this approach, HAProxy forwards requests to a backend service. 

![haproxy-basic.png](img%2Fhaproxy-basic.png)

The guide includes suggested steps to send HAProxy logs to a local Syslog server. This was tested on macOS, so specific commands to install and configure on macOS are listed. The equivalent commands in the guide for Debian-based systems should have the same effect. 

While additional considerations should be given to security in production environments, a basic approach to enable SSL connections using TLSv1.3 is also explored by the guide, for local development and testing purposes. 

## Goals

Setting up HAProxy to forward HTTP requests to a backend service can be driven by several architectural goals:

**A. Single Point of Access**: Clients interact with a single endpoint (HAProxy), simplifying DNS management and reducing the complexity of client configuration.

**B. Backend Abstraction**: Backend services can be moved, added, or removed without impacting the client as long as the HAProxy configuration is updated accordingly.

**C. Health Checks**: Checks on a backend service ensure only healthy endpoints receive traffic.

**D. SSL Termination**: HAProxy can handle SSL termination, decrypting HTTPS requests before forwarding them to a backend service. This offloads the computational overhead of SSL/TLS encryption from the backend.

**E. Centralized Logging**: HAProxy can provide detailed logs of all HTTP requests and responses, which helps in monitoring traffic, debugging issues, and analyzing usage patterns.

## 1. Install

On Debian-based systems:

```
apt-get install haproxy 
```

On macOS:

```
brew install haproxy
```

## 2. Configure

Create a valid `haproxy.cfg` file on your local machine. For example:

```
mkdir ~/haproxy
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

This is set up to forward HTTP requests from port 80 to a backend server running on port 1080. 

The `haproxy.cfg` file has 4 sections:

### a - Global 

`log 127.0.0.1:514  local0  info`: Instructs HAProxy to send logs to the Syslog server listening at 127.0.0.1:514. Messages are sent with facility local0, which is one of the standard, user-defined Syslog facilities.

`local0`: This specifies the syslog facility to be used for the logs. Syslog facilities are used to separate logs from different sources. Common facilities include `local0` to `local7`, `auth`, `cron`, `daemon`, `kern`, and `user`. Using `local0` ensures that HAProxy logs are isolated from other logs, making them easier to filter and analyze.

### b - Defaults

`log global`: Uses the global logging settings defined in the global section.

### c - Frontend

`mode http`: Sets the operating mode to HTTP, indicating that HAProxy will process HTTP traffic.

`timeout connect 5000ms`: Sets the maximum time to wait for a connection to a backend server to 5000 milliseconds (5 seconds).

`timeout client 50000ms`: Sets the maximum inactivity time on the client side to 50000 milliseconds (50 seconds).

`timeout server 50000ms`: Sets the maximum inactivity time on the server side to 50000 milliseconds (50 seconds).

`bind *:80`: Binds HAProxy to listen on all network interfaces on port 80 for incoming HTTP requests.

`default_backend myservers`: Forwards all incoming requests to the `myservers` backend.

### d - Backend

`server server1 127.0.0.1:1080 check`: Defines a single server named `server1` with IP address 127.0.0.1 and port 1080. The `check` option enables health checks to ensure the server is available.

## 3. Logging

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

## 4. Start 

To run haproxy as a service:

```
brew services start haproxy
```

Or, if you don't want/need a background service you can just run:

```
/usr/local/opt/haproxy/bin/haproxy -f ~/haproxy/haproxy.cfg
```

## 5. Test 

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

In this test, the `{ "rsp": "ok" }` response is from the backend service that HAProxy forwards requests to. 

## 6. View Log

You may need to manually create the log file in:

```
sudo touch /var/log/haproxy.log
```

Then, while running tests, try to tail the file:

```
tail -f /var/log/haproxy.log
```
 
Log Format (default:)

```
2034-06-22T13:53:21-07:00 localhost haproxy[4452]: Connect from 127.0.0.1:50408 to 127.0.0.1:80 (myfrontend/HTTP)
```

## 7. Enable HTTPS

For local development and testing purposes, you can enable HTTPS with HAProxy. This approach generates a self-signed certificate that your browser will trust for a sample domain: `hacktothefuture.com`, or other domain name. For this you need to create a local Certificate Authority (CA), use it to sign the certificate, and then add the CA's certificate to your browser's trusted certificates. 

The high-level steps are:

* Domain registration
* Create your own certificate authority (i.e., become a CA)
* Create a certificate signing request (CSR) for the server
* Sign the server's CSR with your CA key
* Install the server certificate on the server
* Install the CA certificate on the client


### Step 1: Register a domain

To bypass the Domain Name System (DNS) for the specified domain, register a domain in your local `/etc/hosts` file, as in:

```
127.0.0.1	hacktothefuture.com
```

This entry causes requests to `hacktothefuture.com` to be directed to the loopback address `127.0.0.1`  instead of looking up the domain on the internet.

### Step 2: Create a Local Certificate Authority (CA)

Generate the private key for your CA:
```
openssl genpkey -algorithm RSA -out myCA.key -aes256
```

Note: This step prompts for a pass phrase:

```
Enter PEM pass phrase:
```

Generate the CA certificate:

```
openssl req -x509 -new -key myCA.key -days 3650 -out myCA.pem -subj "/C=US/ST=State/L=City/O=Organization/OU=Unit/CN=MyLocalCA"
```

### Step 3: Generate a Self-Signed Certificate Using Your CA

Generate a private key for your server:

```
openssl genpkey -algorithm RSA -out server.key -aes256
```

Generate a certificate signing request (CSR):

```
openssl req -new -key server.key -out server.csr -subj "/C=US/ST=State/L=City/O=Organization/OU=Unit/CN=hacktothefuture.com"
```

Create a configuration file for the certificate with the Subject Alternative Name (SAN), in a file named `san.cnf` as follows:

```
[ req ]
distinguished_name = req_distinguished_name
req_extensions = req_ext
prompt = no

[ req_distinguished_name ]
C = US
ST = State
L = City
O = Organization
OU = Unit
CN = hacktothefuture.com

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = hacktothefuture.com
```

Sign the CSR with your CA's private key to create the certificate:

```
openssl x509 -req -in server.csr -CA myCA.pem -CAkey myCA.key -CAcreateserial -out server.crt -days 365 -extfile san.cnf -extensions req_ext
```

Combine the server key and certificate into a single file:

```
cat server.key server.crt > ~/haproxy/haproxy.pem
```

### Step 4: Add the CA Certificate to Your Browser's Trusted Certificates

#### For macOS:

- Open the Keychain Access application.
- Drag and drop `myCA.pem` into the login or system keychain.
- Double-click the certificate and expand the "Trust" section.
- Set "When using this certificate" to "Always Trust."

#### For Debian-based systems (e.g., Ubuntu):

Copy the CA certificate to `/usr/local/share/ca-certificates/`:

```
sudo cp myCA.pem /usr/local/share/ca-certificates/myCA.crt
```

Update the CA certificates:

```
sudo update-ca-certificates
```

### Step 5: Restart HAProxy

Update the `frontend` section in `haproxy.cfg` to bind to port 443 and use the trusted SSL certificate:

```
global
    log 127.0.0.1:514  local0  info 
defaults
    log     global
    mode    http
    timeout connect 5000ms
    timeout client  50000ms
    timeout server  50000ms

frontend myfrontend
  bind *:443 ssl crt /Users/alex/haproxy/haproxy.pem 
  default_backend myservers

backend myservers
  server server1 127.0.0.1:8000 check
```

Before

```
  bind *:80
```

After

```
  bind *:443 ssl crt /Users/alex/haproxy/haproxy.pem 
```

Restart the HAProxy process:

```
/usr/local/opt/haproxy/bin/haproxy -f ~/haproxy/haproxy.cfg
```

Note: You may be prompted for the PEM pass phrase to restart the process:

```
Enter PEM pass phrase:
```

### Step 6: Test

```
curl -v https://hacktothefuture.com/
```

Response

```
> curl -v https://hacktothefuture.com 
*   Trying 127.0.0.1:443...
* Connected to hacktothefuture.com (127.0.0.1) port 443
* ALPN: curl offers h2,http/1.1
* (304) (OUT), TLS handshake, Client hello (1):
*  CAfile: /etc/ssl/cert.pem
*  CApath: none
* (304) (IN), TLS handshake, Server hello (2):
* (304) (IN), TLS handshake, Unknown (8):
* (304) (IN), TLS handshake, Certificate (11):
* (304) (IN), TLS handshake, CERT verify (15):
* (304) (IN), TLS handshake, Finished (20):
* (304) (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / AEAD-AES256-GCM-SHA384
* ALPN: server accepted h2
* Server certificate:
*  subject: C=US; ST=State; L=City; O=Organization; OU=Unit; CN=hacktothefuture.com
*  start date: Jun 23 15:01:17 2024 GMT
*  expire date: Jun 23 15:01:17 2025 GMT
*  subjectAltName: host "hacktothefuture.com" matched cert's "hacktothefuture.com"
*  issuer: C=US; ST=State; L=City; O=Organization; OU=Unit; CN=MyLocalCA
*  SSL certificate verify ok.
* using HTTP/2
* [HTTP/2] [1] OPENED stream for https://hacktothefuture.com/
* [HTTP/2] [1] [:method: GET]
* [HTTP/2] [1] [:scheme: https]
* [HTTP/2] [1] [:authority: hacktothefuture.com]
* [HTTP/2] [1] [:path: /]
* [HTTP/2] [1] [user-agent: curl/8.4.0]
* [HTTP/2] [1] [accept: */*]
> GET / HTTP/2
> Host: hacktothefuture.com
> User-Agent: curl/8.4.0
> Accept: */*
> 
< HTTP/2 200 
< access-control-allow-headers: Content-Type, api_key, Authorization
< date: Sun, 23 Jun 2024 16:09:32 GMT
< access-control-allow-methods: GET, POST, DELETE, PUT, PATCH, OPTIONS
< access-control-allow-origin: https://editor.swagger.io
< content-length: 17
< 
{ "rsp": "ok" }
```

A secure connection is established using TLSv1.3. This curl response also shows a successful connection and communication with the local server configured to use HTTPS with a self-signed certificate. The detailed steps of the TLS handshake, certificate verification, and HTTP/2 communication are included in the verbose output from curl.

**Browser test**

To verify a secure connection is established by a web browser, and it trusts the self-signed certificate, go to the secure URL: 

https://hacktothefuture.com/ 

![trusted-cert.png](img%2Ftrusted-cert.png)

## Links

HAProxy Configuration Basics: Load Balance Your Servers

https://www.haproxy.com/blog/haproxy-configuration-basics-load-balance-your-servers

Introduction to HAProxy Logging

https://www.haproxy.com/blog/introduction-to-haproxy-logging

SSL / TLS

https://www.haproxy.com/documentation/haproxy-configuration-tutorials/ssl-tls/

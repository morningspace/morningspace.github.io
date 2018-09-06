HAProxy is an open source application used as TCP/HTTP Load Balancer and for proxy solutions. It can distribute the workload across multiple Elasticsearch nodes thus improving the overall performance and reliability.

In this guide, we will discuss the process of setting up a high availability load balancer using HAProxy to control the traffic of SSL enabled Elasticsearch by separating HTTPS requests across multiple Elasticsearch nodes.

## Install HAProxy

To install HAProxy, run the following commands on RedHat based systems:

    # yum update
    # yum install haproxy

Or on Debian based systems:

    # apt-get update
    # apt-get install haproxy

Make sure your HAProxy version is higher than or equal to 1.5, because according to HAProxy release information, SSL is natively supported starting from HAproxy version 1.5.

You can run the following command to make sure haproxy is installed properly:

    # which haproxy
    /usr/sbin/haproxy

and check the version by:

    # haproxy -v
    HA-Proxy version 1.5.18 2016/05/10
    Copyright 2000-2016 Willy Tarreau <willy@haproxy.org>

Now, let's configure the haproxy to be auto-started so that can be started after reboot. Make sure the haproxy service has a functional systemd init script located at /etc/systemd/system/multi-user.target.wants/haproxy.service. Then use the following command to enable the service:

    # systemctl enable haproxy.service

Reload the systemd daemon to take effect:

    # systemctl daemon-reload

And check the service status to make sure it has been enabled

    # service haproxy.service status
    Redirecting to /bin/systemctl status haproxy.service
    ● haproxy.service - HAProxy Load Balancer
       Loaded: loaded (/usr/lib/systemd/system/haproxy.service; enabled; vendor preset: disabled)
       Active: inactive (dead)

Note it has been enabled but is not in active status. We will configure HAProxy before start it.

## Configure HAProxy

After HAProxy is installed, let us create a setup in which we have three Elasticsearch nodes and one HAProxy instance.

To start, backup the original sample configuration file at /etc/haproxy

    # cp /etc/haproxy/haproxy.cfg{,.original}

### Configure HAProxy Logs

For future debugging purpose, we need to enable logging feature in HAProxy. Open the HAProxy configuration file `/etc/haproxy/haproxy.cfg` to make sure the following line is defined under `global` section.

    log         127.0.0.1 local2

Then, enable UDP syslog reception in `/etc/rsyslog.conf` configuration file to separate log files for HAProxy under /var/log directory. Open the `rsyslog.conf` file and uncomment the following lines:

    # Provides UDP syslog reception
    $ModLoad imudp
    $UDPServerRun 514

Then, create a separate file `haproxy.conf` under `/etc/rsyslog.d/` directory to configure log specific for HAProxy. Append the following line to the file:

    local2.*	/var/log/haproxy.log

Restart the rsyslog service to take effect.

### Configure front & backend with SSL enabled

To enable SSL, the most typical use case is `SSL Termination`. To have HAProxy handle the SSL connection, it means having the SSL Certificate live on HAProxy server. HAProxy uses a single file to hold private key, signed certificate, and any optinally intermediate CA certificates in PEM format. If your private key has passphrase, HAProxy will prompt you for it on each start and stop, which is not suitable for a daemon. So, we also need to remove the password from the key.

In our case, suppose all certificates and keys are stored under /etc/elasticsearch/certs. Let's use elasticsearch-admin.key as an example. You can use the openssl rsa command to remove the passphrase.

    # openssl rsa -in elasticsearch-admin.key -out elasticsearch-admin-no-pass.key

Concatenate the certificate and key files together (in that order) as below

    # cat elasticsearch-admin.crt.pem elasticsearch-admin-no-pass.key | tee elasticsearch-admin-crt-key.pem

To use the generated certificate and key files, add the following part in `global` section:

    # default directory to fetch SSL CA certificates
    ca-base /etc/elasticsearch/certs
    # default directory to fetch SSL certifictes
    crt-base /etc/elasticsearch/certs

Then, define the frontend as below:

    frontend main
      bind *:443     ssl no-sslv3 crt elasticsearch-admin-crt-key.pem
      default_backend app

It configures an HTTPS endpoint. We tell HAProxy what interface and port to listen to(*:443), to use HTTPS(ssl directive) but not SSL 3.0(no-sslv3), and where to find the certificate bundle.

Next, define the backend as below:

    backend app
      balance roundrobin
      server  esnode1 9.112.244.23:9200
      server  esnode2 9.112.244.65:9200
      server  esnode3 9.112.244.95:9200

It configures the three Elasticsearch nodes with their IP addresses and ports, and use roundrobin as the balance algorithm.

### Configure health check

To make sure the backend servers are healthy, we will configure HAProxy to check their health by continuously requesting a specific URL endpoint provided by Elasticsearch. Let's change the backend definition as below:

    backend app
      balance roundrobin
      option  httpchk GET /
      option  log-health-checks
      server  esnode1 9.112.244.23:9200 ssl no-sslv3 verify required ca-file ca/chain-ca.pem crt elasticsearch-admin-crt-key.pem check inter 10s fall 3 rise 2
      server  esnode2 9.112.244.65:9200 ssl no-sslv3 verify required ca-file ca/chain-ca.pem crt elasticsearch-admin-crt-key.pem check inter 10s fall 3 rise 2
      server  esnode3 9.112.244.95:9200 ssl no-sslv3 verify required ca-file ca/chain-ca.pem crt elasticsearch-admin-crt-key.pem check inter 10s fall 3 rise 2

We use `httpcheck` option to specify what service endpoint will be used for health check. We can also use `log-health-checks` option to enable logging of health checks status updates.

The `check` option after each server definition enables health checks on the backend server instances. The `inter` parameter sets the interval between two consecutive health checks. The `fall` parameter states that the server will be considered as dead after specified number of consecutive unsuccessful health checks. The `rise` parameter states that the server will be considered as operational after specified number of consecutive successful health checks.

We also enable the SSL for health check by the `ssl` option with proper certificates being provided.

## Start HAProxy

Before start, you can have HAProxy validate the configuration file by running the below command:

    # haproxy -f /etc/haproxy/haproxy.cfg -c

If there's no validation errors, we can run the below command to start the HAProxy service:

    # service haproxy start

Then, monitor the log by checking haproxy.log:

    # tail -f /var/log/haproxy.log

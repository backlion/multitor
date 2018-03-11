# multitor

## Releases

|            **STABLE RELEASE**            |           **TESTING RELEASE**            |
| :--------------------------------------: | :--------------------------------------: |
| [![](https://img.shields.io/badge/Branch-master-green.svg)]() | [![](https://img.shields.io/badge/Branch-testing-orange.svg)]() |
| [![](https://img.shields.io/badge/Version-v1.2.1-lightgrey.svg)]() | [![](https://img.shields.io/badge/Version-v1.2.1-lightgrey.svg)]() |
| [![Build Status](https://travis-ci.org/trimstray/multitor.svg?branch=master)](https://travis-ci.org/trimstray/multitor) | [![Build Status](https://travis-ci.org/trimstray/multitor.svg?branch=testing)](https://travis-ci.org/trimstray/multitor) |

## Description

A tool that lets you **create multiple TOR** instances with a **load-balancing** traffic between them by **HAProxy**. It's provides one single endpoint for clients. In addition, you can **view** previously running **TOR** processes and create a **new identity** for all or selected processes.

> The **multitor** has been completely rewritten on the basis of:
>
> - **Multi-TOR** project written by *Jai Seidl*: [Multi-TOR](https://github.com/jseidl/Multi-TOR)
> - original source is (*Sebastian Wain* project): [Distributed Scraping With Multiple Tor Circuits](http://blog.databigbang.com/distributed-scraping-with-multiple-tor-circuits/)

## Parameters

Provides the following options:

```bash
  Usage:
    multitor <option|long-option>

  Examples:
    multitor --init 2 --user debian-tor --socks-port 9000 --control-port 9900
    multitor --show-id --socks-port 9000

  Options:
        --help                      show this message
        --debug                     displays information on the screen (debug mode)
    -i, --init <num>                init new tor processes
    -s, --show-id                   show specific tor process id
    -n, --new-id                    regenerate tor circuit
    -u, --user <string>             set the user (only with -i|--init)
        --socks-port <port_num|all> set socks port number
        --control-port <port_num>   set control port number
        --proxy <socks|http>        set tor load balancer
```

## Requirements

**<u>Multitor</u>** uses external utilities to be installed before running:

- [tor](https://www.torproject.org/)
- [netcat](http://netcat.sourceforge.net/)
- [haproxy](https://www.haproxy.org/)
- [polipo](https://www.irif.fr/~jch/software/polipo/)

## Install/uninstall

It's simple - for install:

```bash
./setup.sh install
```

For remove:

```bash
./setup.sh uninstall
```

> - symlink to `bin/multitor` is placed in `/usr/local/bin`
> - man page is placed in `/usr/local/man/man8`

## Use example

### Creating processes

Then an example of starting the tool:

```bash
multitor --init 2 -u debian-tor --socks-port 9000 --control-port 9900
```

Creates new **Tor** processes and specifies the number of processes to create:

- `--init 2`

Specifies the user from which new processes will be created (the user must exist in the system):

- `-u debian-tor`

Specifies the port number for **Tor** communication. Increased by 1 for each subsequent process:

- `--socks-port 9000`

Specifies the port number of the **Tor** process control. Increased by 1 for each subsequent process:

- `--control-port 9900`

### Reviewing processes

Examples of obtaining information about a given **Tor** process created by **multitor**:

```bash
multitor --show-id --socks-port 9000
```

We want to get information about a given **Tor** process:

- `--show-id`

> You can use the **all** value to display all processes.

Specifies the port number for communication. Allows you to find the process after this port number:

- `--socks-port 9000`

### New Tor identity

If there is a need to create a new identity:

```bash
multitor --new-id --socks-port 9000
```

We set up creating a new identity for **Tor** process:

- `--new-id`

> You can use the **all** value to regenerate identity for all processes.

Specifies the port number for communication. Allows you to find the process after this port number:

- `--socks-port 9000`

### Proxy

See [Load balancing](#lb).

### Output example

So if We created 2 **TOR** processes by **Multitor** example output will be given:

![multitor_output](doc/img/multitor_output.png)

## Load balancing {#lb}

**Multitor** uses two techniques to create a load balancing mechanism -  these are **socks proxy** and **http proxy**. Each of these types of load balancing is good but its purpose is slightly different.

For browsing websites (generally for **http/https** traffic) it is recommended to use **http proxy**. In this configuration, the **polipo** service is used, which has many very useful functions (including cache memory) which in the case of **TOR** is not always well-aimed. In addition, we are confident in better handling of ssl traffic.

The **socks proxy** type is also reliable, however, when browsing websites through **TOR** nodes it can cause more problems.

**Multitor** uses **HAProxy** to create a local proxy server for all created **TOR** or **Polipo** instances and distribute traffic between them. The default configuration is in `templates/haproxy-template.cfg`.

> **HAProxy** uses **16379** to communication, so all of your services to use the load balancer should have this port number.

### SOCKS Proxy

Communication architecture:

```bash
 Client
    |
    |
 HAProxy (127.0.0.1:16379)
    |
    |--------> TOR Instance (127.0.0.1:9000)
    |
    |--------> TOR Instance (127.0.0.1:9001)
```

To run the load balancer you need to add the `--proxy socks` parameter to the command specified in the example.

```bash
multitor --init 2 -u debian-tor --socks-port 9000 --control-port 9900 --proxy socks
```

After launching, let's see the working processes:

```bash
netstat -tapn | grep LISTEN | grep "tor\|haproxy\|polipo"
tcp        0      0 127.0.0.1:9000          0.0.0.0:*               LISTEN      28976/tor
tcp        0      0 127.0.0.1:9001          0.0.0.0:*               LISTEN      29039/tor
tcp        0      0 127.0.0.1:9900          0.0.0.0:*               LISTEN      28976/tor
tcp        0      0 127.0.0.1:9901          0.0.0.0:*               LISTEN      29039/tor
tcp        0      0 127.0.0.1:16379         0.0.0.0:*               LISTEN      29104/haproxy
tcp        0      0 127.0.0.1:16380         0.0.0.0:*               LISTEN      29104/haproxy
```

In order to test the correctness of the setup, you can run the following command:

```bash
for i in $(seq 1 4) ; do printf "req %2d: " "$i" ; curl -k --location --socks5 127.0.0.1:16379 http://ipinfo.io/ip ; done
req  1: 5.254.79.66
req  2: 178.175.135.99
req  3: 5.254.79.66
req  4: 178.175.135.99
```

Communication through **socks proxy** takes place without a cache (except browsers that have their own cache). **Curl** and other low-level programs should work without any problems.

### HTTP Proxy

Communication architecture:

```bash
 Client
    |
    |
 HAProxy (127.0.0.1:16379)
    |
    |--------> Polipo Instance (127.0.0.1:8000)
    |             |
    |             |---------> TOR Instance (127.0.0.1:9000)
    |
    |--------> Polipo Instance (127.0.0.1:8001)
                  |
                  |---------> TOR Instance (127.0.0.1:9001)
```

To run the load balancer you need to add the `--proxy http` parameter to the command specified in the example.

```bash
multitor --init 2 -u debian-tor --socks-port 9000 --control-port 9900 --proxy http
```

After launching, let's see the working processes:

```bash
netstat -tapn | grep LISTEN | grep "tor\|haproxy\|polipo"
tcp        0      0 127.0.0.1:9000          0.0.0.0:*               LISTEN      32168/tor
tcp        0      0 127.0.0.1:9001          0.0.0.0:*               LISTEN      32246/tor
tcp        0      0 127.0.0.1:9900          0.0.0.0:*               LISTEN      32168/tor
tcp        0      0 127.0.0.1:9901          0.0.0.0:*               LISTEN      32246/tor
tcp        0      0 127.0.0.1:16379         0.0.0.0:*               LISTEN      32327/haproxy
tcp        0      0 127.0.0.1:16380         0.0.0.0:*               LISTEN      32327/haproxy
tcp        0      0 127.0.0.1:8000          0.0.0.0:*               LISTEN      32307/polipo
tcp        0      0 127.0.0.1:8001          0.0.0.0:*               LISTEN      32320/polipo
```

In order to test the correctness of the setup, you can run the following command:

```bash
for i in $(seq 1 4) ; do printf "req %2d: " "$i" ; curl -k --location --proxy 127.0.0.1:16379 http://ipinfo.io/ip ; done
req  1: 178.209.42.84
req  2: 185.100.85.61
req  3: 178.209.42.84
req  4: 185.100.85.61
```

In the default configuration, the **Polipo** cache has been **turned off** (look at the configuration template). If you set the network configuration in the browser so that the traffic passes through **HAProxy**, you must remember that browsers have their **own cache,** which can cause that each entry to the page will be from the same IP address. This is not a big problem because it is not always the case. After clearing the browser cache again, the web server will receive the request from a different IP address.

You can check it for example in the firefox browsers by installing the "*Empty Cache Button by mvm*" add-on and enter the [http://myexternalip.com/](http://myexternalip.com/) website.

### Port convention

The port numbers for the **TOR** are set by the user using the `--socks-port` parameter. Additionally, the standard port on which **HAProxy** listens is **16379**. **Polipo** uses ports **1000** smaller than those set for **TOR**.

### HAProxy stats interface

If you want to view traffic statistics, go to http://127.0.0.1:16380/stats.

### Polipo configuration interface

If you wat to view or changed **Polipo** params, got to [http://127.0.0.1:8000/polipo/config](http://127.0.0.1:8000/polipo/config) (remember the right port number).

### Gateway

If you are building a gateway for **TOR** connections, you can put **HAProxy** on an external IP address by changing the `bind` directive in **haproxy-template.cfg**:

```bash
bind              0.0.0.0:16379 name proxy
```

## Password authentication

**Multitor** uses password for authorization on the control port. The password is generated automatically and contains 18 random characters - it is displayed in the final report after the creation of new processes.

## Logging

After running the script, the `log/` directory is created and in it the following files with logs:

- `<script_name>.<date>.log` - all `_logger()` function calls are saved in it
- `stdout.log` - a standard output and errors from the `_init_cmd()` and other function are written in it

## Important

If you use this tool in other scripts where the output is saved everywhere, not on the screen, remember that you will not be able to use the generated password. I will correct this in the next version. If you do not use regenerate function of single or all **TOR** circuits with a password, you can safely restart the **multitor** which will do it for you.

## Limitations

- each **Tor** process needs a certain number of memory. If the number of processes is too big, the oldest one will be automatic killed by the system
- **Polipo** is no longer supported but it is still a very good and light proxy. It is very good for such applications. In the next version I will give you the option to choose a different solution

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## Project architecture

    |-- LICENSE.md                 # GNU GENERAL PUBLIC LICENSE, Version 3, 29 June 2007
    |-- README.md                  # this simple documentation
    |-- CONTRIBUTING.md            # principles of project support
    |-- .gitignore                 # ignore untracked files
    |-- .travis.yml                # continuous integration with Travis CI
    |-- setup.sh                   # install multitor on the system
    |-- bin
        |-- multitor               # main script (init)
    |-- doc                        # includes documentation, images and manuals
        |-- man8
            |-- multitor.8         # man page for multitor
    |-- etc                        # contains configuration files
    |-- lib                        # libraries, external functions
    |-- log                        # contains logs, created after init
    |-- src                        # includes external project files
        |-- helpers                # contains core functions
        |-- import                 # appends the contents of the lib directory
        |-- __init__               # contains the __main__ function
        |-- settings               # contains multitor settings
    |-- templates                  # contains examples and template files
        |-- haproxy-template.cfg   # example of HAProxy configuration
        |-- polipo-template.cfg    # example of Polipo configuration

## License

GPLv3 : <http://www.gnu.org/licenses/>

**Free software, Yeah!**

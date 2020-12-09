# Install Guide

## Build krakend from source

**Requirements**

- go >= 1.12
- git
- Internet connection
- make

```
make build
```

it will generate `./krakend` binary file. You can build it locally and then copy `krakend` to your server.

## Install

**Copy binary to target directory**

```
mkdir /opt/gateway
mv ./krakend /opt/gateway/krakend
```

**Make config file**

```
cd /opt/gateway
vi config.json
```

```
{
    "version": 2,
    "name": "My lovely gateway",
    "port": 80,
    "cache_ttl": "3600s",
    "timeout": "3s",
    "extra_config": {
      "github_com/devopsfaith/krakend-gologging": {
        "level":  "DEBUG",
        "prefix": "[KRAKEND]",
        "syslog": false,
        "stdout": true
      },
      "github_com/devopsfaith/krakend-metrics": {
        "collection_time": "60s",
        "proxy_disabled": false,
        "router_disabled": false,
        "backend_disabled": false,
        "endpoint_disabled": false,
        "listen_address": ":8090"
      }
    },
    "endpoints": [
        {
            "endpoint": "/supu",
            "method": "GET",
            "headers_to_pass": ["Authorization", "Content-Type"],
            "backend": [
                {
                    "host": [
                        "http://127.0.0.1:8000"
                    ],
                    "url_pattern": "/__debug/supu",
                    "extra_config": {
                        "github.com/devopsfaith/krakend-martian": {
                            "fifo.Group": {
                                "scope": ["request", "response"],
                                "aggregateErrors": true,
                                "modifiers": [
                                {
                                    "header.Modifier": {
                                        "scope": ["request", "response"],
                                        "name" : "X-Martian",
                                        "value" : "ouh yeah!"
                                    }
                                },
                                {
                                    "header.RegexFilter": {
                                        "scope": ["request"],
                                        "header" : "X-Neptunian",
                                        "regex" : "no!",
                                        "modifier": {
                                            "header.Modifier": {
                                                "scope": ["request"],
                                                "name" : "X-Martian-New",
                                                "value" : "some value"
                                            }
                                        }
                                    }
                                }
                                ]
                            }
                        }
                    }
                }
            ]
        }
    ]
}
```

**Check application**

```
/opt/gateway/krakend check -c /opt/gateway/config.json
```

**Create a service**

```
vi /etc/systemd/system/gateway.service
```

```
[Unit]
Description=gateway
After=syslog.target
After=network.target
Wants=xml-convertor.service
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=1
User=root
WorkingDirectory=/opt/gateway
ExecStart=krakend run -c config.json
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=analytics-api_gateway
AmbientCapabilities=CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target
```

**logging**

configure log rotation from syslog with `gateway` prefix

## Start service

```
systemctl enable gateway
systemctl start  gateway
```
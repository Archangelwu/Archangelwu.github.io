# ubuntu server + docker + ipvlan + multi-postfix images

這邊記錄一下設定記錄：

新增兩組虛擬 IP：10.1.11.41 與 10.1.11.42 

```shell=
postfix@testpostfix:~$ sudo cat /etc/netplan/00-installer-config.yaml
[sudo] password for postfix:
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens160:
      addresses:
        - 10.1.11.40/24
        - 10.1.11.41/24
        - 10.1.11.42/24
      nameservers:
        addresses:
          - 8.8.8.8
        search: []
      routes:
      - to: default
        via: 10.1.11.254
  version: 2
```

新增 ipvlan 給 docker 使用：

```shell
sudo docker network create -d ipvlan   --subnet=10.1.11.0/24   --gateway=10.1.11.254   -o ipvlan_mode=l2   -o parent=ens160   my_ipvlan_network
```

確認一下：

```shell=
postfix@testpostfix:~$ sudo docker network ls
NETWORK ID     NAME                DRIVER    SCOPE
210bee15d171   bridge              bridge    local
95a1d897ef1d   host                host      local
1afb7ea28cb4   my_ipvlan_network   ipvlan    local
884ced2539e7   none                null      local
```

docker image list

```shell=
postfix@testpostfix:~$ sudo docker image ls
REPOSITORY          TAG       IMAGE ID       CREATED        SIZE
postfix-with-nano   latest    2c46959bea3a   2 weeks ago    265MB
boky/postfix        latest    96c3f55740bf   3 weeks ago    231MB
hello-world         latest    d2c94e258dcb   9 months ago   13.3kB
```

docker run 吧！

```shell=
sudo docker run -d --name postfix-41 --network my_ipvlan_network --ip 10.1.11.41 postfix-with-nano

sudo docker run -d --name postfix-42 --network my_ipvlan_network --ip 10.1.11.42 postfix-with-nano
```

檢查一下 docker network inspect

```shell=
postfix@testpostfix:~$ sudo docker network inspect my_ipvlan_network
[
    {
        "Name": "my_ipvlan_network",
        "Id": "1afb7ea28cb4a27f3176733dcfc4277cde7c0e727116b845648d8c5b817b9a9e",
        "Created": "2024-01-31T07:10:04.838777596Z",
        "Scope": "local",
        "Driver": "ipvlan",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "10.1.11.0/24",
                    "Gateway": "10.1.11.254"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "9d9e72d885c1e5ca709355000aaca8c85ef58d8e94d00dafd6d0bb609ec5dc24": {
                "Name": "postfix-42",
                "EndpointID": "20ed5eb5eb65703ab427a46c49ac6469769bf75fe1da5f02c4bbb4e24300fe62",
                "MacAddress": "",
                "IPv4Address": "10.1.11.42/24",
                "IPv6Address": ""
            },
            "a4b34c08b51b7f5d9db03729672b46f2efbd9056588ed03cac7561e86b472dd4": {
                "Name": "postfix-41",
                "EndpointID": "30ceba1d778a436d03f8c39861072d4c635967a5046b0d6ca85fc46da6812d94",
                "MacAddress": "",
                "IPv4Address": "10.1.11.41/24",
                "IPv6Address": ""
            }
        },
        "Options": {
            "ipvlan_mode": "l2",
            "parent": "ens160"
        },
        "Labels": {}
    }
]
```

:::info
檢查 docker inspect postfix-41

但因為資料頗多，所以請進入詳細資訊察看唷！
:::
:::spoiler
```shell=
postfix@testpostfix:~$ sudo docker inspect postfix-41
[
    {
        "Id": "a4b34c08b51b7f5d9db03729672b46f2efbd9056588ed03cac7561e86b472dd4",
        "Created": "2024-01-31T07:12:33.807732199Z",
        "Path": "/bin/sh",
        "Args": [
            "-c",
            "/scripts/run.sh"
        ],
        "State": {
            "Status": "running",
            "Running": true,
            "Paused": false,
            "Restarting": false,
            "OOMKilled": false,
            "Dead": false,
            "Pid": 6246,
            "ExitCode": 0,
            "Error": "",
            "StartedAt": "2024-01-31T07:12:34.436544421Z",
            "FinishedAt": "0001-01-01T00:00:00Z",
            "Health": {
                "Status": "healthy",
                "FailingStreak": 0,
                "Log": [
                    {
                        "Start": "2024-01-31T08:33:34.266873086Z",
                        "End": "2024-01-31T08:33:36.470178968Z",
                        "ExitCode": 0,
                        "Output": "Postfix check...\nDKIM check...\nAll OK!\n"
                    },
                    {
                        "Start": "2024-01-31T08:34:06.47802743Z",
                        "End": "2024-01-31T08:34:08.663768321Z",
                        "ExitCode": 0,
                        "Output": "Postfix check...\nDKIM check...\nAll OK!\n"
                    },
                    {
                        "Start": "2024-01-31T08:34:38.670854477Z",
                        "End": "2024-01-31T08:34:40.86043903Z",
                        "ExitCode": 0,
                        "Output": "Postfix check...\nDKIM check...\nAll OK!\n"
                    },
                    {
                        "Start": "2024-01-31T08:35:10.867525338Z",
                        "End": "2024-01-31T08:35:13.072293924Z",
                        "ExitCode": 0,
                        "Output": "Postfix check...\nDKIM check...\nAll OK!\n"
                    },
                    {
                        "Start": "2024-01-31T08:35:43.080300317Z",
                        "End": "2024-01-31T08:35:45.269648162Z",
                        "ExitCode": 0,
                        "Output": "Postfix check...\nDKIM check...\nAll OK!\n"
                    }
                ]
            }
        },
        "Image": "sha256:2c46959bea3a9c85f7a21e96460dc8c16ca3bc05ff2b0d3e08d781e9b96076b0",
        "ResolvConfPath": "/var/lib/docker/containers/a4b34c08b51b7f5d9db03729672b46f2efbd9056588ed03cac7561e86b472dd4/resolv.conf",
        "HostnamePath": "/var/lib/docker/containers/a4b34c08b51b7f5d9db03729672b46f2efbd9056588ed03cac7561e86b472dd4/hostname",
        "HostsPath": "/var/lib/docker/containers/a4b34c08b51b7f5d9db03729672b46f2efbd9056588ed03cac7561e86b472dd4/hosts",
        "LogPath": "/var/lib/docker/containers/a4b34c08b51b7f5d9db03729672b46f2efbd9056588ed03cac7561e86b472dd4/a4b34c08b51b7f5d9db03729672b46f2efbd9056588ed03cac7561e86b472dd4-json.log",
        "Name": "/postfix-41",
        "RestartCount": 0,
        "Driver": "overlay2",
        "Platform": "linux",
        "MountLabel": "",
        "ProcessLabel": "",
        "AppArmorProfile": "docker-default",
        "ExecIDs": [
            "88b380562270af6f7497cbea5bf4fea38338c7c99a30aa5421e9dea405b2a6d5"
        ],
        "HostConfig": {
            "Binds": null,
            "ContainerIDFile": "",
            "LogConfig": {
                "Type": "json-file",
                "Config": {}
            },
            "NetworkMode": "my_ipvlan_network",
            "PortBindings": {},
            "RestartPolicy": {
                "Name": "no",
                "MaximumRetryCount": 0
            },
            "AutoRemove": false,
            "VolumeDriver": "",
            "VolumesFrom": null,
            "ConsoleSize": [
                30,
                150
            ],
            "CapAdd": null,
            "CapDrop": null,
            "CgroupnsMode": "private",
            "Dns": [],
            "DnsOptions": [],
            "DnsSearch": [],
            "ExtraHosts": null,
            "GroupAdd": null,
            "IpcMode": "private",
            "Cgroup": "",
            "Links": null,
            "OomScoreAdj": 0,
            "PidMode": "",
            "Privileged": false,
            "PublishAllPorts": false,
            "ReadonlyRootfs": false,
            "SecurityOpt": null,
            "UTSMode": "",
            "UsernsMode": "",
            "ShmSize": 67108864,
            "Runtime": "runc",
            "Isolation": "",
            "CpuShares": 0,
            "Memory": 0,
            "NanoCpus": 0,
            "CgroupParent": "",
            "BlkioWeight": 0,
            "BlkioWeightDevice": [],
            "BlkioDeviceReadBps": [],
            "BlkioDeviceWriteBps": [],
            "BlkioDeviceReadIOps": [],
            "BlkioDeviceWriteIOps": [],
            "CpuPeriod": 0,
            "CpuQuota": 0,
            "CpuRealtimePeriod": 0,
            "CpuRealtimeRuntime": 0,
            "CpusetCpus": "",
            "CpusetMems": "",
            "Devices": [],
            "DeviceCgroupRules": null,
            "DeviceRequests": null,
            "MemoryReservation": 0,
            "MemorySwap": 0,
            "MemorySwappiness": null,
            "OomKillDisable": null,
            "PidsLimit": null,
            "Ulimits": null,
            "CpuCount": 0,
            "CpuPercent": 0,
            "IOMaximumIOps": 0,
            "IOMaximumBandwidth": 0,
            "MaskedPaths": [
                "/proc/asound",
                "/proc/acpi",
                "/proc/kcore",
                "/proc/keys",
                "/proc/latency_stats",
                "/proc/timer_list",
                "/proc/timer_stats",
                "/proc/sched_debug",
                "/proc/scsi",
                "/sys/firmware",
                "/sys/devices/virtual/powercap"
            ],
            "ReadonlyPaths": [
                "/proc/bus",
                "/proc/fs",
                "/proc/irq",
                "/proc/sys",
                "/proc/sysrq-trigger"
            ]
        },
        "GraphDriver": {
            "Data": {
                "LowerDir": "/var/lib/docker/overlay2/76946cbcce99a58e88f585d5b1ca2793e99e522f31c03db064db2b3a0f260dfd-init/diff:/var/lib/docker/overlay2/d07c2fa903e074b6d7135f0ab43ae1f42d0a7e266a1385acce76c9dc83be3dea/diff:/var/lib/docker/overlay2/9060bfc6830e4e5e3a5213cd6f0573dddced0ff68ab652520e2ff8e43ae75dff/diff:/var/lib/docker/overlay2/b1f9cc3b11e1792673981ecdd1b5455d82888820bae30d6c2d97230643399e0a/diff:/var/lib/docker/overlay2/6082ed8183a4bf0eff85857e7c8574af5556c582c55b1ca082251cc4b853696f/diff:/var/lib/docker/overlay2/71546895a548d653778ac8e7e8d06e43bbf2b3388730d8def7b6eb82dc92d8c1/diff:/var/lib/docker/overlay2/36efb30976fc98dc5414969d9a1829e8afa8ef89bd6334348ec62904e5a74fa3/diff:/var/lib/docker/overlay2/229d9381673cf269067fcdcb1416cf6dfb07f2ec692f67ff1e6b8818bc85b2da/diff:/var/lib/docker/overlay2/a412ea4a467d1276277734dd69efa6c24437028b69d0572839c996d8d7ada279/diff:/var/lib/docker/overlay2/975008f36759e342f5628f68fb60d9ded9e448b50bfef14d7e16c981d6f180d4/diff:/var/lib/docker/overlay2/01d67e3ca13651add9e6a73a8466a6adfa44a68e5d717fbe49a7bfd4db715b65/diff:/var/lib/docker/overlay2/1df773259c8c2376c3915556a76a9f602420f898d9368379e6c38d9c8f994dbc/diff:/var/lib/docker/overlay2/03d0ba6bc5b67f4232211e1eae9c57d72dbfe6330b87922f2c806a9aedf96f05/diff:/var/lib/docker/overlay2/26f3b33e56960c4a8f9087a509503fb579a94d9662e37513a58c0c777266861a/diff",
                "MergedDir": "/var/lib/docker/overlay2/76946cbcce99a58e88f585d5b1ca2793e99e522f31c03db064db2b3a0f260dfd/merged",
                "UpperDir": "/var/lib/docker/overlay2/76946cbcce99a58e88f585d5b1ca2793e99e522f31c03db064db2b3a0f260dfd/diff",
                "WorkDir": "/var/lib/docker/overlay2/76946cbcce99a58e88f585d5b1ca2793e99e522f31c03db064db2b3a0f260dfd/work"
            },
            "Name": "overlay2"
        },
        "Mounts": [
            {
                "Type": "volume",
                "Name": "fb4562d31c6f3c25d56265d56b26b390e71d67bde5ce87bfcfc2b6f843baa909",
                "Source": "/var/lib/docker/volumes/fb4562d31c6f3c25d56265d56b26b390e71d67bde5ce87bfcfc2b6f843baa909/_data",
                "Destination": "/etc/postfix",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            },
            {
                "Type": "volume",
                "Name": "1329a84a64539c7694361da23e2fb5608fe7328cd83993a6cf740767b1c904c0",
                "Source": "/var/lib/docker/volumes/1329a84a64539c7694361da23e2fb5608fe7328cd83993a6cf740767b1c904c0/_data",
                "Destination": "/var/spool/postfix",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            },
            {
                "Type": "volume",
                "Name": "0f2e6430f2f2c6b2c4b3ba710ac4bda80cd86fe5e873aa00eb86404fae93ee56",
                "Source": "/var/lib/docker/volumes/0f2e6430f2f2c6b2c4b3ba710ac4bda80cd86fe5e873aa00eb86404fae93ee56/_data",
                "Destination": "/etc/opendkim/keys",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            }
        ],
        "Config": {
            "Hostname": "a4b34c08b51b",
            "Domainname": "",
            "User": "root",
            "AttachStdin": false,
            "AttachStdout": false,
            "AttachStderr": false,
            "ExposedPorts": {
                "25/tcp": {},
                "587/tcp": {}
            },
            "Tty": false,
            "OpenStdin": false,
            "StdinOnce": false,
            "Env": [
                "ALLOW_EMPTY_SENDER_DOMAINS=true",
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
            ],
            "Cmd": [
                "/bin/sh",
                "-c",
                "/scripts/run.sh"
            ],
            "Healthcheck": {
                "Test": [
                    "CMD-SHELL",
                    "/scripts/healthcheck.sh"
                ],
                "Interval": 30000000000,
                "Timeout": 5000000000,
                "StartPeriod": 10000000000,
                "Retries": 3
            },
            "Image": "postfix-with-nano",
            "Volumes": {
                "/etc/opendkim/keys": {},
                "/etc/postfix": {},
                "/var/spool/postfix": {}
            },
            "WorkingDir": "/tmp",
            "Entrypoint": null,
            "OnBuild": null,
            "Labels": {
                "maintainer": "Bojan Cekrlic - https://github.com/bokysan/docker-postfix/"
            }
        },
        "NetworkSettings": {
            "Bridge": "",
            "SandboxID": "83d5b1aa24de928f05889b880dcb3f84003cae3318e3c050819f74856382cdbe",
            "HairpinMode": false,
            "LinkLocalIPv6Address": "",
            "LinkLocalIPv6PrefixLen": 0,
            "Ports": {},
            "SandboxKey": "/var/run/docker/netns/83d5b1aa24de",
            "SecondaryIPAddresses": null,
            "SecondaryIPv6Addresses": null,
            "EndpointID": "",
            "Gateway": "",
            "GlobalIPv6Address": "",
            "GlobalIPv6PrefixLen": 0,
            "IPAddress": "",
            "IPPrefixLen": 0,
            "IPv6Gateway": "",
            "MacAddress": "",
            "Networks": {
                "my_ipvlan_network": {
                    "IPAMConfig": {
                        "IPv4Address": "10.1.11.41"
                    },
                    "Links": null,
                    "Aliases": [
                        "a4b34c08b51b"
                    ],
                    "NetworkID": "1afb7ea28cb4a27f3176733dcfc4277cde7c0e727116b845648d8c5b817b9a9e",
                    "EndpointID": "30ceba1d778a436d03f8c39861072d4c635967a5046b0d6ca85fc46da6812d94",
                    "Gateway": "10.1.11.254",
                    "IPAddress": "10.1.11.41",
                    "IPPrefixLen": 24,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "",
                    "DriverOpts": null
                }
            }
        }
    }
]
```
:::

:::info
檢查 docker inspect postfix-42

但因為資料頗多，所以請進入詳細資訊察看唷！
:::
:::spoiler
```shell=
postfix@testpostfix:~$ sudo docker inspect postfix-42
[
    {
        "Id": "9d9e72d885c1e5ca709355000aaca8c85ef58d8e94d00dafd6d0bb609ec5dc24",
        "Created": "2024-01-31T07:12:47.67018071Z",
        "Path": "/bin/sh",
        "Args": [
            "-c",
            "/scripts/run.sh"
        ],
        "State": {
            "Status": "running",
            "Running": true,
            "Paused": false,
            "Restarting": false,
            "OOMKilled": false,
            "Dead": false,
            "Pid": 6758,
            "ExitCode": 0,
            "Error": "",
            "StartedAt": "2024-01-31T07:12:48.294569751Z",
            "FinishedAt": "0001-01-01T00:00:00Z",
            "Health": {
                "Status": "healthy",
                "FailingStreak": 0,
                "Log": [
                    {
                        "Start": "2024-01-31T08:37:01.092736502Z",
                        "End": "2024-01-31T08:37:03.286081006Z",
                        "ExitCode": 0,
                        "Output": "Postfix check...\nDKIM check...\nAll OK!\n"
                    },
                    {
                        "Start": "2024-01-31T08:37:33.295664813Z",
                        "End": "2024-01-31T08:37:35.476397105Z",
                        "ExitCode": 0,
                        "Output": "Postfix check...\nDKIM check...\nAll OK!\n"
                    },
                    {
                        "Start": "2024-01-31T08:38:05.481791838Z",
                        "End": "2024-01-31T08:38:07.665621658Z",
                        "ExitCode": 0,
                        "Output": "Postfix check...\nDKIM check...\nAll OK!\n"
                    },
                    {
                        "Start": "2024-01-31T08:38:37.673258026Z",
                        "End": "2024-01-31T08:38:39.845017225Z",
                        "ExitCode": 0,
                        "Output": "Postfix check...\nDKIM check...\nAll OK!\n"
                    },
                    {
                        "Start": "2024-01-31T08:39:09.851427724Z",
                        "End": "2024-01-31T08:39:12.0492291Z",
                        "ExitCode": 0,
                        "Output": "Postfix check...\nDKIM check...\nAll OK!\n"
                    }
                ]
            }
        },
        "Image": "sha256:2c46959bea3a9c85f7a21e96460dc8c16ca3bc05ff2b0d3e08d781e9b96076b0",
        "ResolvConfPath": "/var/lib/docker/containers/9d9e72d885c1e5ca709355000aaca8c85ef58d8e94d00dafd6d0bb609ec5dc24/resolv.conf",
        "HostnamePath": "/var/lib/docker/containers/9d9e72d885c1e5ca709355000aaca8c85ef58d8e94d00dafd6d0bb609ec5dc24/hostname",
        "HostsPath": "/var/lib/docker/containers/9d9e72d885c1e5ca709355000aaca8c85ef58d8e94d00dafd6d0bb609ec5dc24/hosts",
        "LogPath": "/var/lib/docker/containers/9d9e72d885c1e5ca709355000aaca8c85ef58d8e94d00dafd6d0bb609ec5dc24/9d9e72d885c1e5ca709355000aaca8c85ef58d8e94d00dafd6d0bb609ec5dc24-json.log",
        "Name": "/postfix-42",
        "RestartCount": 0,
        "Driver": "overlay2",
        "Platform": "linux",
        "MountLabel": "",
        "ProcessLabel": "",
        "AppArmorProfile": "docker-default",
        "ExecIDs": null,
        "HostConfig": {
            "Binds": null,
            "ContainerIDFile": "",
            "LogConfig": {
                "Type": "json-file",
                "Config": {}
            },
            "NetworkMode": "my_ipvlan_network",
            "PortBindings": {},
            "RestartPolicy": {
                "Name": "no",
                "MaximumRetryCount": 0
            },
            "AutoRemove": false,
            "VolumeDriver": "",
            "VolumesFrom": null,
            "ConsoleSize": [
                30,
                150
            ],
            "CapAdd": null,
            "CapDrop": null,
            "CgroupnsMode": "private",
            "Dns": [],
            "DnsOptions": [],
            "DnsSearch": [],
            "ExtraHosts": null,
            "GroupAdd": null,
            "IpcMode": "private",
            "Cgroup": "",
            "Links": null,
            "OomScoreAdj": 0,
            "PidMode": "",
            "Privileged": false,
            "PublishAllPorts": false,
            "ReadonlyRootfs": false,
            "SecurityOpt": null,
            "UTSMode": "",
            "UsernsMode": "",
            "ShmSize": 67108864,
            "Runtime": "runc",
            "Isolation": "",
            "CpuShares": 0,
            "Memory": 0,
            "NanoCpus": 0,
            "CgroupParent": "",
            "BlkioWeight": 0,
            "BlkioWeightDevice": [],
            "BlkioDeviceReadBps": [],
            "BlkioDeviceWriteBps": [],
            "BlkioDeviceReadIOps": [],
            "BlkioDeviceWriteIOps": [],
            "CpuPeriod": 0,
            "CpuQuota": 0,
            "CpuRealtimePeriod": 0,
            "CpuRealtimeRuntime": 0,
            "CpusetCpus": "",
            "CpusetMems": "",
            "Devices": [],
            "DeviceCgroupRules": null,
            "DeviceRequests": null,
            "MemoryReservation": 0,
            "MemorySwap": 0,
            "MemorySwappiness": null,
            "OomKillDisable": null,
            "PidsLimit": null,
            "Ulimits": null,
            "CpuCount": 0,
            "CpuPercent": 0,
            "IOMaximumIOps": 0,
            "IOMaximumBandwidth": 0,
            "MaskedPaths": [
                "/proc/asound",
                "/proc/acpi",
                "/proc/kcore",
                "/proc/keys",
                "/proc/latency_stats",
                "/proc/timer_list",
                "/proc/timer_stats",
                "/proc/sched_debug",
                "/proc/scsi",
                "/sys/firmware",
                "/sys/devices/virtual/powercap"
            ],
            "ReadonlyPaths": [
                "/proc/bus",
                "/proc/fs",
                "/proc/irq",
                "/proc/sys",
                "/proc/sysrq-trigger"
            ]
        },
        "GraphDriver": {
            "Data": {
                "LowerDir": "/var/lib/docker/overlay2/22e9615c504fe25fe8e14905a17a6ae03b482671b74b622f89bae69bdb6cf58a-init/diff:/var/lib/docker/overlay2/d07c2fa903e074b6d7135f0ab43ae1f42d0a7e266a1385acce76c9dc83be3dea/diff:/var/lib/docker/overlay2/9060bfc6830e4e5e3a5213cd6f0573dddced0ff68ab652520e2ff8e43ae75dff/diff:/var/lib/docker/overlay2/b1f9cc3b11e1792673981ecdd1b5455d82888820bae30d6c2d97230643399e0a/diff:/var/lib/docker/overlay2/6082ed8183a4bf0eff85857e7c8574af5556c582c55b1ca082251cc4b853696f/diff:/var/lib/docker/overlay2/71546895a548d653778ac8e7e8d06e43bbf2b3388730d8def7b6eb82dc92d8c1/diff:/var/lib/docker/overlay2/36efb30976fc98dc5414969d9a1829e8afa8ef89bd6334348ec62904e5a74fa3/diff:/var/lib/docker/overlay2/229d9381673cf269067fcdcb1416cf6dfb07f2ec692f67ff1e6b8818bc85b2da/diff:/var/lib/docker/overlay2/a412ea4a467d1276277734dd69efa6c24437028b69d0572839c996d8d7ada279/diff:/var/lib/docker/overlay2/975008f36759e342f5628f68fb60d9ded9e448b50bfef14d7e16c981d6f180d4/diff:/var/lib/docker/overlay2/01d67e3ca13651add9e6a73a8466a6adfa44a68e5d717fbe49a7bfd4db715b65/diff:/var/lib/docker/overlay2/1df773259c8c2376c3915556a76a9f602420f898d9368379e6c38d9c8f994dbc/diff:/var/lib/docker/overlay2/03d0ba6bc5b67f4232211e1eae9c57d72dbfe6330b87922f2c806a9aedf96f05/diff:/var/lib/docker/overlay2/26f3b33e56960c4a8f9087a509503fb579a94d9662e37513a58c0c777266861a/diff",
                "MergedDir": "/var/lib/docker/overlay2/22e9615c504fe25fe8e14905a17a6ae03b482671b74b622f89bae69bdb6cf58a/merged",
                "UpperDir": "/var/lib/docker/overlay2/22e9615c504fe25fe8e14905a17a6ae03b482671b74b622f89bae69bdb6cf58a/diff",
                "WorkDir": "/var/lib/docker/overlay2/22e9615c504fe25fe8e14905a17a6ae03b482671b74b622f89bae69bdb6cf58a/work"
            },
            "Name": "overlay2"
        },
        "Mounts": [
            {
                "Type": "volume",
                "Name": "b5eaab2a271f6499bdb3d5f6e160e8a38d00f2b690d6179fbfeb9b93f76c8e92",
                "Source": "/var/lib/docker/volumes/b5eaab2a271f6499bdb3d5f6e160e8a38d00f2b690d6179fbfeb9b93f76c8e92/_data",
                "Destination": "/etc/opendkim/keys",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            },
            {
                "Type": "volume",
                "Name": "0d2ac6cc18210e41a61ea5dd32a2bbf7dae51ceddcd2cb82b754879910326f3d",
                "Source": "/var/lib/docker/volumes/0d2ac6cc18210e41a61ea5dd32a2bbf7dae51ceddcd2cb82b754879910326f3d/_data",
                "Destination": "/etc/postfix",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            },
            {
                "Type": "volume",
                "Name": "54032da9684dd5ec45b9243f7962596e1fb6817f8f9467421c3162ab8d224b75",
                "Source": "/var/lib/docker/volumes/54032da9684dd5ec45b9243f7962596e1fb6817f8f9467421c3162ab8d224b75/_data",
                "Destination": "/var/spool/postfix",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            }
        ],
        "Config": {
            "Hostname": "9d9e72d885c1",
            "Domainname": "",
            "User": "root",
            "AttachStdin": false,
            "AttachStdout": false,
            "AttachStderr": false,
            "ExposedPorts": {
                "25/tcp": {},
                "587/tcp": {}
            },
            "Tty": false,
            "OpenStdin": false,
            "StdinOnce": false,
            "Env": [
                "ALLOW_EMPTY_SENDER_DOMAINS=true",
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
            ],
            "Cmd": [
                "/bin/sh",
                "-c",
                "/scripts/run.sh"
            ],
            "Healthcheck": {
                "Test": [
                    "CMD-SHELL",
                    "/scripts/healthcheck.sh"
                ],
                "Interval": 30000000000,
                "Timeout": 5000000000,
                "StartPeriod": 10000000000,
                "Retries": 3
            },
            "Image": "postfix-with-nano",
            "Volumes": {
                "/etc/opendkim/keys": {},
                "/etc/postfix": {},
                "/var/spool/postfix": {}
            },
            "WorkingDir": "/tmp",
            "Entrypoint": null,
            "OnBuild": null,
            "Labels": {
                "maintainer": "Bojan Cekrlic - https://github.com/bokysan/docker-postfix/"
            }
        },
        "NetworkSettings": {
            "Bridge": "",
            "SandboxID": "8df630dc332d8c53d5c8592630c6372cd525c0eccdbd71737e84d44c9596717c",
            "HairpinMode": false,
            "LinkLocalIPv6Address": "",
            "LinkLocalIPv6PrefixLen": 0,
            "Ports": {},
            "SandboxKey": "/var/run/docker/netns/8df630dc332d",
            "SecondaryIPAddresses": null,
            "SecondaryIPv6Addresses": null,
            "EndpointID": "",
            "Gateway": "",
            "GlobalIPv6Address": "",
            "GlobalIPv6PrefixLen": 0,
            "IPAddress": "",
            "IPPrefixLen": 0,
            "IPv6Gateway": "",
            "MacAddress": "",
            "Networks": {
                "my_ipvlan_network": {
                    "IPAMConfig": {
                        "IPv4Address": "10.1.11.42"
                    },
                    "Links": null,
                    "Aliases": [
                        "9d9e72d885c1"
                    ],
                    "NetworkID": "1afb7ea28cb4a27f3176733dcfc4277cde7c0e727116b845648d8c5b817b9a9e",
                    "EndpointID": "20ed5eb5eb65703ab427a46c49ac6469769bf75fe1da5f02c4bbb4e24300fe62",
                    "Gateway": "10.1.11.254",
                    "IPAddress": "10.1.11.42",
                    "IPPrefixLen": 24,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "",
                    "DriverOpts": null
                }
            }
        }
    }
]
```
:::


察看 postfix image 裡頭的 main.cf 與 master.cf

```shell=
sudo docker exec -it postfix-41 /bin/bash
```

:::info
cat /etc/postfix/master.cf

但因為資料頗多，所以請進入詳細資訊察看唷！
:::
:::spoiler
```shell=
postfix@testpostfix:~$ sudo docker exec -it postfix-41 /bin/bash
root@a4b34c08b51b:/tmp# cat /etc/postfix/master.cf
#
# Postfix master process configuration file.  For details on the format
# of the file, see the master(5) manual page (command: "man 5 master" or
# on-line: http://www.postfix.org/master.5.html).
#
# Do not forget to execute "postfix reload" after editing this file.
#
# ==========================================================================
# service type  private unpriv  chroot  wakeup  maxproc command + args
#               (yes)   (yes)   (no)    (never) (100)
# ==========================================================================
smtp      inet  n       -       n       -       -       smtpd
#smtp      inet  n       -       n       -       1       postscreen
#smtpd     pass  -       -       n       -       -       smtpd
#dnsblog   unix  -       -       n       -       0       dnsblog
#tlsproxy  unix  -       -       n       -       0       tlsproxy
submission inet n       -       n       -       -       smtpd
#  -o syslog_name=postfix/submission
#  -o smtpd_tls_security_level=encrypt
#  -o smtpd_sasl_auth_enable=yes
#  -o smtpd_tls_auth_only=yes
#  -o smtpd_reject_unlisted_recipient=no
#  -o smtpd_client_restrictions=$mua_client_restrictions
#  -o smtpd_helo_restrictions=$mua_helo_restrictions
#  -o smtpd_sender_restrictions=$mua_sender_restrictions
#  -o smtpd_recipient_restrictions=
#  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
#  -o milter_macro_daemon_name=ORIGINATING
#smtps     inet  n       -       n       -       -       smtpd
#  -o syslog_name=postfix/smtps
#  -o smtpd_tls_wrappermode=yes
#  -o smtpd_sasl_auth_enable=yes
#  -o smtpd_reject_unlisted_recipient=no
#  -o smtpd_client_restrictions=$mua_client_restrictions
#  -o smtpd_helo_restrictions=$mua_helo_restrictions
#  -o smtpd_sender_restrictions=$mua_sender_restrictions
#  -o smtpd_recipient_restrictions=
#  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
#  -o milter_macro_daemon_name=ORIGINATING
#628       inet  n       -       n       -       -       qmqpd
pickup    unix  n       -       n       60      1       pickup
cleanup   unix  n       -       n       -       0       cleanup
qmgr      unix  n       -       n       300     1       qmgr
#qmgr     unix  n       -       n       300     1       oqmgr
tlsmgr    unix  -       -       n       1000?   1       tlsmgr
rewrite   unix  -       -       n       -       -       trivial-rewrite
bounce    unix  -       -       n       -       0       bounce
defer     unix  -       -       n       -       0       bounce
trace     unix  -       -       n       -       0       bounce
verify    unix  -       -       n       -       1       verify
flush     unix  n       -       n       1000?   0       flush
proxymap  unix  -       -       n       -       -       proxymap
proxywrite unix -       -       n       -       1       proxymap
smtp      unix  -       -       n       -       -       smtp
relay     unix  -       -       n       -       -       smtp
        -o syslog_name=postfix/$service_name
#       -o smtp_helo_timeout=5 -o smtp_connect_timeout=5
showq     unix  n       -       n       -       -       showq
error     unix  -       -       n       -       -       error
retry     unix  -       -       n       -       -       error
discard   unix  -       -       n       -       -       discard
local     unix  -       n       n       -       -       local
virtual   unix  -       n       n       -       -       virtual
lmtp      unix  -       -       n       -       -       lmtp
anvil     unix  -       -       n       -       1       anvil
scache    unix  -       -       n       -       1       scache
postlog   unix-dgram n  -       n       -       1       postlogd
#
# ====================================================================
# Interfaces to non-Postfix software. Be sure to examine the manual
# pages of the non-Postfix software to find out what options it wants.
#
# Many of the following services use the Postfix pipe(8) delivery
# agent.  See the pipe(8) man page for information about ${recipient}
# and other message envelope options.
# ====================================================================
#
# maildrop. See the Postfix MAILDROP_README file for details.
# Also specify in main.cf: maildrop_destination_recipient_limit=1
#
maildrop  unix  -       n       n       -       -       pipe
  flags=DRhu user=vmail argv=/usr/bin/maildrop -d ${recipient}
#
# ====================================================================
#
# Recent Cyrus versions can use the existing "lmtp" master.cf entry.
#
# Specify in cyrus.conf:
#   lmtp    cmd="lmtpd -a" listen="localhost:lmtp" proto=tcp4
#
# Specify in main.cf one or more of the following:
#  mailbox_transport = lmtp:inet:localhost
#  virtual_transport = lmtp:inet:localhost
#
# ====================================================================
#
# Cyrus 2.1.5 (Amos Gouaux)
# Also specify in main.cf: cyrus_destination_recipient_limit=1
#
#cyrus     unix  -       n       n       -       -       pipe
#  user=cyrus argv=/cyrus/bin/deliver -e -r ${sender} -m ${extension} ${user}
#
# ====================================================================
# Old example of delivery via Cyrus.
#
#old-cyrus unix  -       n       n       -       -       pipe
#  flags=R user=cyrus argv=/cyrus/bin/deliver -e -m ${extension} ${user}
#
# ====================================================================
#
# See the Postfix UUCP_README file for configuration details.
#
uucp      unix  -       n       n       -       -       pipe
  flags=Fqhu user=uucp argv=uux -r -n -z -a$sender - $nexthop!rmail ($recipient)
#
# Other external delivery methods.
#
ifmail    unix  -       n       n       -       -       pipe
  flags=F user=ftn argv=/usr/lib/ifmail/ifmail -r $nexthop ($recipient)
bsmtp     unix  -       n       n       -       -       pipe
  flags=Fq. user=bsmtp argv=/usr/lib/bsmtp/bsmtp -t$nexthop -f$sender $recipient
scalemail-backend unix	-	n	n	-	2	pipe
  flags=R user=scalemail argv=/usr/lib/scalemail/bin/scalemail-store ${nexthop} ${user} ${extension}
mailman   unix  -       n       n       -       -       pipe
  flags=FR user=list argv=/usr/lib/mailman/bin/postfix-to-mailman.py
  ${nexthop} ${user}

```
:::

:::info
cat /etc/postfix/main.cf

但因為資料頗多，所以請進入詳細資訊察看唷！
:::
:::spoiler
```shell=
root@a4b34c08b51b:/tmp# cat /etc/postfix/main.cf
# See /usr/share/postfix/main.cf.dist for a commented, more complete version


# Debian specific:  Specifying a file name will cause the first
# line of that file to be used as the name.  The Debian default
# is /etc/mailname.
#myorigin = /etc/mailname

smtpd_banner = $myhostname ESMTP $mail_name (Debian/GNU)
biff = no

# appending .domain is the MUA's job.
append_dot_mydomain = no

# Uncomment the next line to generate "delayed mail" warnings
#delay_warning_time = 4h

readme_directory = no

# See http://www.postfix.org/COMPATIBILITY_README.html -- default to 3.6 on
# fresh installs.
compatibility_level = 3.6



# TLS parameters
smtpd_tls_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
smtpd_tls_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
smtpd_tls_security_level=may

smtp_tls_CApath=/etc/ssl/certs
smtp_tls_security_level = may
smtp_tls_session_cache_database = lmdb:${data_directory}/smtp_scache


smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination
#myhostname = localhost.j2xen23xrnaetdgy434mngmvif.cx.internal.cloudapp.net
alias_maps = lmdb:/etc/aliases
alias_database = lmdb:/etc/aliases
mydestination =
#relayhost =
mynetworks = 127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
mailbox_size_limit = 0
recipient_delimiter = +
inet_interfaces = all
inet_protocols = all
manpage_directory = /usr/share/man
default_database_type = lmdb
relay_domains =
header_size_limit = 4096000
smtpd_delay_reject = yes
smtpd_helo_required = yes
smtpd_client_restrictions = permit_mynetworks,reject
smtpd_helo_restrictions = permit_mynetworks,reject_invalid_helo_hostname,permit
smtpd_sender_restrictions = permit_mynetworks,reject
smtp_sasl_mechanism_filter = scram,scram,scram,scram,scram,digestmd5,crammd5,ntlm,login,plain,anonymous
message_size_limit = 0
myhostname = a4b34c08b51b
```
:::

查看 netstat -ntulp

```shell=
root@a4b34c08b51b:/tmp# netstat -ntulp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.11:40865        0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:25              0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:587             0.0.0.0:*               LISTEN      -
tcp6       0      0 :::25                   :::*                    LISTEN      -
tcp6       0      0 :::587                  :::*                    LISTEN      -
udp        0      0 127.0.0.11:53203        0.0.0.0:*                           -
```


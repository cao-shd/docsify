- 安装

```shell
yum install bind-chroot bind-utils -y
```

- 修改配置文件

```shell
vi /etc/named.conf

options {
        # listen-on port 53 { any; };
        listen-on port 53 { any; };
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        recursing-file  "/var/named/data/named.recursing";
        secroots-file   "/var/named/data/named.secroots";
        # allow-query     { any; };
        allow-query     { any; };

        /*
         - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
         - If you are building a RECURSIVE (caching) DNS server, you need to enable
           recursion.
         - If your recursive DNS server has a public IP address, you MUST enable access
           control to limit queries to your legitimate users. Failing to do so will
           cause your server to become part of large scale DNS amplification
           attacks. Implementing BCP38 within your network would greatly
           reduce such attack surface
        */
        recursion yes;

        dnssec-enable yes;
        dnssec-validation yes;

        /* Path to ISC DLV key */
        bindkeys-file "/etc/named.root.key";

        managed-keys-directory "/var/named/dynamic";

        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
        type hint;
        file "named.ca";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";

```

- 修改域名的区域文件

```shell
vi /etc/named.rfc1912.zones

zone "xxx.com" IN {
    type master;
    file "xxx.com.zone";
    allow-update { none; };
};
zone "131.168.192.in-addr.arpa" {
    type master;
    file "192.168.131.arpa";
    allow-update { none; };
};
```

- 设置正向解析文件

```shell
cd /var/named
cp -a named.localhost xxx.com.zone

$TTL 1D
@       IN SOA  xxx.com. root.xxx.com. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        NS      ns.xxx.com.
ns      IN  A   192.168.131.10
www     IN  A   192.168.131.10
```

- 设置反向解析文件

```shell
cd /var/named
cp -a named.loopback  192.168.131.arpa

$TTL 1D
@       IN SOA  xxx.com. root.xxx.com. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        NS      ns.xxx.com.
ns      A       192.168.131.10
10      PTR     www.xxx.com.
```

- 验证

```shell
nslookup
```

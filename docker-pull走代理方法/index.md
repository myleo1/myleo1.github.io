# docker pull走代理方法


## STEP1

进入编辑systemd配置

```bash
nano /usr/lib/systemd/system/docker.service
```

## SETP2

在systemd配置文件的最后的[install]前加入如下几行(配置代理)

```bash
Environment=HTTP_PROXY=127.0.0.1:1087
Environment=HTTPS_PROXY=127.0.0.1:1087
Environment=NO_PROXY=localhost,127.0.0.1,172.17.0.0/16,192.168.56.0/24,10.96.0.0/16
```

## SETP3

```bash
systemctl daemon-reload
```

```bash
systemctl restart docker
```



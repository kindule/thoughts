---
title: docker-exec-error
date: 2018-09-29 10:31:38
tags:
---

```shell
[root@products-core /root]
$docker exec -it confluence2 sh
rpc error: code = 13 desc = invalid header field value "oci runtime error: exec failed: container_linux.go:247: starting container process caused \"process_linux.go:75: starting setns process caused \\\"fork/exec /proc/self/exe: no such file or directory\\\"\"\n"
```


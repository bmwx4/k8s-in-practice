给容器磁盘做限制

```bash
$ xfs_quota -x -c 'project -s -p /data2/hostpath/test 1234' /data2
$ xfs_quota -x -c 'limit -p bhard=100G 123' /data2
```

```bash
# xfs_quota -x -c "report -h" /data2
Project quota on /data2 (/dev/sdb1)
                        Blocks              
Project ID   Used   Soft   Hard Warn/Grace   
---------- ---------------------------------
123      0   100G   100G  00 [———]
```

## Linux磁盘分区

**fdisk -l**

![1585189815107](C:\Users\lushuang\AppData\Roaming\Typora\typora-user-images\1585189815107.png)

**fdisk /dev/sdb**

*输入m---n---p---1--回车---回车---p---w*

**fdisk  -l**

![1585190023420](C:\Users\lushuang\AppData\Roaming\Typora\typora-user-images\1585190023420.png)

**格式化磁盘**

*mkfs.ext4 /dev/vdb1*

**挂盘**

*mount /dev/sdb1 /data*

**查看分区UUID**

*ls -l /dev/disk/by-uuid/*

**永久挂盘**

```shell
UUID=4c2c090d-4228-49fc-9cbe-3920b3bf287c /                       ext4    defaults        1 1
UUID=927bf6a4-c63f-47ca-9d6b-5f8f334c0307            /data                      ext4    defaults        1 2
~ 
```




# ChCore Lab5：文件系统与Shell



## 练习1

> 实现tfs_mknode和tfs_namex。





## 练习2

> 实现tfs_file_read和tfs_file_write。提示：由于数据块的大小 为PAGE_SIZE，因此读写可能会牵涉到多个页面。读取不能超过文件大 小，而写入可能会增加文件大小（也可能需要创建新的数据块）。



## 练习3

> 实现tfs_load_image函数。需要通过之前实现的函数进行目录和文件 的创建，以及数据的读写。





## 练习4

> 实 现 在user/tmpfs/tmpfs_main.c中 的fs_dispatch和 在user/tmpfs/tmpfs_server.c中 的 所 有fs_server_xxx处 理函数，附表给出了fs_dispatch的分发规则。



|     请求ID      |        函数        |      功能      |
| :-------------: | :----------------: | :------------: |
|   FS_REQ_SCAN   |   fs_server_scan   | 列出文件或目录 |
|  FS_REQ_MKDIR   |  fs_server_mkdir   |    创建目录    |
|  FS_REQ_CREAT   |  fs_server_creat   |    创建文件    |
|  FS_REQ_RMDIR   |  fs_server_rmdir   |    移除目录    |
|  FS_REQ_UNLINK  |  fs_server_unlink  |    移除文件    |
|  FS_REQ_WRITE   |  fs_server_write   |     写文件     |
|   FS_REQ_READ   |   fs_server_read   |     读文件     |
| FS_REQ_GET_SIZE | fs_server_get_size |  获取文件大小  |





## 练习5

> 实现在user/apps/init_main.c中定义的getchar()，以每次从标 准输入中获取字符，以及在user/apps/init.c中定义的readline。 该函数将按下回车键之前的输入内容存入内存缓冲区并将其标准输出。 可以使用在user/lib/syscall.{h,c}中的 I/O 函数。





## 练习6

> 实现在user/apps/init.c中定义的bultin_cmd()以支持 shell 中 的内置命令，例如ls，ls [dir]，cd [dir]，echo [string]，cat [filename]。





## 练习7

> 实现在user/apps/init.c中定义的run_cmd，以通过输入文件名来 运行可执行文件，并且支持按 tab 自动补全。可以使用在user/lib/目 录 下proc.h中 定 义 的launch_process_with_pmos_caps和 在liblauncher.c中定义的readelf_from_fs。



## 练习8

> 实现top以显示线程信息。top的格式如下：（还可以在每行的末尾添加 其他信息）：

```shell
$ top
Current CPU 0
===== CPU 0 =====
Thread ffffff0020600228 Type: ROOT    State: TS_RUNNING
CPU 0 AFF 0 Budget 2
Thread ffffff00206007a8 Type: USER    State: TS_READY
CPU 0 AFF 0 Budget 2
===== CPU 1 =====
Thread ffffff00001ecfa8 Type: IDLE    State: TS_RUNNING
CPU 1 AFF 1 Budget 2
===== CPU 2 ===== 
Thread ffffff00001ed000 Type: IDLE    State: TS_RUNNING
CPU 2 AFF 2 Budget 2
===== CPU 3 =====
Thread ffffff00001ed058 Type: IDLE    State: TS_RUNNING
CPU 3 AFF 3 Budget 2
```




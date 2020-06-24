如何查看obj，dll等文件是x86还是x64的架构，可以使用dumpbin

dumpbin /ALL /OUT:dump.txt xxx.obj

这个命令dumpbin会输出obj的所有信息，在“FILE HEADER VALUES”里面可以看到架构等summary信息

```
FILE HEADER VALUES
             14C machine (x86)
              4B number of sections
        5EF2EDBD time date stamp Wed Jun 24 14:07:57 2020
            2605 file pointer to symbol table
              FC number of symbols
               0 size of optional header
               0 characteristics
```




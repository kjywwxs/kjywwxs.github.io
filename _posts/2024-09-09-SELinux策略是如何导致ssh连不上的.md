## SElinux的工作模式

- 强制模式enforcing
- 宽松模式permissive
- 禁用disabled

查看当前模式
`getenforce`

**临时切换**
强制模式-->宽松模式
`setenforce 0`
宽松模式-->强制模式
`setenforce 1`

**永久切换**
修改配置文件`/etc/selinux/config`
然后重启

## SELinux安全上下文

mv不改变文件的安全上下文

## SELinux布尔值

`getsebool -a`
`setsebool`

## SELinux开放非默认端口

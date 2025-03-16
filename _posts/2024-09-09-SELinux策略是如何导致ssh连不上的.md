# 2024-09-09-SELinux策略是如何导致ssh连不上的.md

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

---

## SELinux安全上下文

mv不改变文件的安全上下文  
（若ssh密钥/配置文件被移动，可能导致上下文不匹配）  
使用`restorecon -Rv /path`恢复默认上下文  
查看上下文：`ls -Z /path/to/file`

---

## SELinux布尔值

`getsebool -a`  
`setsebool`  

关键布尔值：  
- `ssh_system_admin`（控制ssh管理权限）  
- `ssh_chroot_rw_homedirs`（控制chroot环境访问）  
修改示例：  
`setsebool -P ssh_system_admin on`

---

## SELinux开放非默认端口

当SSH使用非22端口时：  
1. 检查当前允许的SSH端口  
`semanage port -l | grep ssh`  
2. 添加新端口（如2222）  
`semanage port -a -t ssh_port_t -p tcp 2222`  
3. 验证配置  
`ss -tulnp | grep 2222`

---

## 故障排查流程（SSH无法连接）

### 1. 快速诊断
```bash
# 临时切换为宽松模式
setenforce 0
# 测试SSH连接是否恢复
```

### 2. 检查审计日志
```bash
grep "avc:.*sshd" /var/log/audit/audit.log
ausearch -m avc -c sshd | audit2why
```

### 3. 常见故障场景
#### 场景1：端口未授权
- 报错特征：`bind to port 2222 failed`
- 解决方案：使用`semanage port`命令添加端口

#### 场景2：密钥文件上下文错误
```bash
# 修复.ssh目录上下文
restorecon -R -v ~/.ssh
```

#### 场景3：非标准配置路径
```bash
# 若配置文件不在/etc/ssh，需添加新文件上下文
semanage fcontext -a -t sshd_config_t "/custom/path/sshd_config"
restorecon -v /custom/path/sshd_config
```

### 4. 生成自定义策略模块
```bash
grep sshd /var/log/audit/audit.log | audit2allow -M my_sshd
semodule -i my_sshd.pp
```

---

## 总结排查步骤
1. 通过`setenforce 0`快速定位是否SELinux导致
2. 使用`audit2why`分析具体拒绝事件
3. 根据错误类型选择：
   - 修正安全上下文
   - 开放对应端口
   - 调整布尔值
   - 创建自定义策略

> **注意事项**  
> - 生产环境不建议直接禁用SELinux  
> - 修改配置后需重启ssh服务：`systemctl restart sshd`  
> - 永久性修改务必使用`-P`参数（如`setsebool -P`）  


---

# 1. 高可用总体思路

```
             虚拟 IP (VIP)
            ┌─────────────┐
            │  Keepalived  │
            ├──────┬───────┤
      ┌─────▼──┐ ┌─▼───────┐
      │ Nginx 1 │ │ Nginx 2 │    (主备模式，Keepalived 管理 VIP 漂移)
      │ (MASTER)│ │(BACKUP) │
      └────┬────┘ └───┬─────┘
           │          │
    ┌──────▼──────────▼──────┐
    │      upstream 后端      │
    └────────────────────────┘
```

---

# 2. Keepalived + Nginx 主备方案

## 2.1 安装 Keepalived

```bash
yum install -y keepalived
```

## 2.2 Keepalived 配置（MASTER）

```bash
# /etc/keepalived/keepalived.conf
global_defs {
    router_id nginx_ha_master;
}

vrrp_script chk_nginx {
    script "/etc/keepalived/check_nginx.sh"    # Nginx 存活检测脚本
    interval 2                                  # 每 2 秒检测
    weight -20                                  # 检测失败时降低优先级
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100              # 优先级（MASTER > BACKUP）
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass 1111
    }

    virtual_ipaddress {
        192.168.1.100          # VIP
    }

    track_script {
        chk_nginx
    }
}
```

## 2.3 Keepalived 配置（BACKUP）

```bash
# /etc/keepalived/keepalived.conf
global_defs {
    router_id nginx_ha_backup;
}

vrrp_script chk_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    weight -20
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 90               # 优先级比 MASTER 低
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass 1111
    }

    virtual_ipaddress {
        192.168.1.100
    }

    track_script {
        chk_nginx
    }
}
```

## 2.4 Nginx 存活检测脚本

```bash
#!/bin/bash
# /etc/keepalived/check_nginx.sh
counter=$(ps -C nginx --no-header | wc -l)
if [ $counter -eq 0 ]; then
    systemctl start nginx
    sleep 2
    counter=$(ps -C nginx --no-header | wc -l)
    if [ $counter -eq 0 ]; then
        systemctl stop keepalived    # Nginx 启动失败，停止 keepalived，VIP 漂移
    fi
fi
```

```bash
chmod +x /etc/keepalived/check_nginx.sh
```

## 2.5 测试 VIP 漂移

```bash
# 查看 VIP 所在节点
ip addr show eth0 | grep 192.168.1.100

# 模拟 MASTER 故障
systemctl stop nginx        # MASTER 上停止 Nginx
# 观察 VIP 是否漂移到 BACKUP
```

---

# 3. 平滑升级（零停机版本更新）

```bash
# 1. 备份旧二进制
cp /usr/sbin/nginx /usr/sbin/nginx.old

# 2. 替换新二进制（源码编译的版本）
cp /usr/local/src/nginx-1.26.0/objs/nginx /usr/sbin/nginx

# 3. 启动新 Master 进程
kill -USR2 $(cat /var/run/nginx.pid)
# → 此时新旧 Master 并存

# 4. 优雅关闭旧 Worker
kill -WINCH $(cat /var/run/nginx.pid.oldbin)

# 5. 验证新版本正常
curl -I http://localhost

# 6. 关闭旧 Master
kill -QUIT $(cat /var/run/nginx.pid.oldbin)

# 回滚：
# kill -HUP $(cat /var/run/nginx.pid.oldbin)    # 唤醒旧 Master
# kill -QUIT $(cat /var/run/nginx.pid)           # 关闭新 Master
```

---

# 4. 多活架构（DNS 轮询）

```
              DNS 轮询 / 智能 DNS
         ┌─────────┴──────────┐
         ▼                    ▼
  ┌────────────┐      ┌────────────┐
  │  Nginx A   │      │  Nginx B   │ （多节点同时对外服务）
  │  北京机房   │      │  上海机房   │
  └─────┬──────┘      └─────┬──────┘
        │                    │
   ┌────▼────┐          ┌───▼─────┐
   │ 后端集群  │          │ 后端集群  │
   └─────────┘          └─────────┘
```

> DNS 轮询的局限：无法感知节点健康状态，故障切换慢。生产环境建议 **VIP + DNS** 组合或负载均衡设备（如 F5、SLB）。

---

# 5. 云环境高可用

| 云平台 | 方案 |
|--------|------|
| **阿里云** | SLB + Nginx（ECS）+ 多可用区 |
| **AWS** | ELB/ALB + Nginx（EC2）+ Multi-AZ |
| **腾讯云** | CLB + Nginx（CVM）+ 多可用区 |

云上推荐架构：

```
SLB/CLB/ALB → Nginx 集群（Auto Scaling）→ 后端（K8s / ECS)
```

---

# 6. Nginx 宕机自动恢复

```bash
# 创建 systemd 服务（yum 安装已自带）
# /usr/lib/systemd/system/nginx.service

[Service]
Restart=always
RestartSec=5
StartLimitInterval=0
```

```bash
# 或 crontab 定时检测
* * * * * /etc/keepalived/check_nginx.sh
```

---

# 7. 配置同步

多个 Nginx 节点间的配置同步方案：

| 方案 | 说明 |
|------|------|
| **rsync / scp** | 手动或脚本同步配置文件 |
| **Ansible / Salt** | 批量配置管理与分发 |
| **NFS / 共享存储** | 配置文件挂载共享目录 |
| **配置中心** | Apollo / Nacos / Consul 动态配置（需 OpenResty 支持） |

```bash
# Ansible 一键更新所有 Nginx 配置
ansible-playbook -i inventory nginx_deploy.yml
```

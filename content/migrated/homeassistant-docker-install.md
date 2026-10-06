---
title: 华为悦盒上 Docker 装 HomeAssistant + HACS
date: 2022-08-22
tags: [HomeAssistant, Docker, 智能家居, 折腾]
description: arm 版 ubuntu 上 Docker 跑 HomeAssistant：换源、装 ha、HACS 第三方商城与排坑。
origin: 博客园迁移
---

# 华为悦盒上 Docker 装 HomeAssistant + HACS

> 原载博客园（2022-08-22，阅读 2056）。基于华为悦盒刷的 arm_ubuntu_20.04。

## 换 Docker 源

```json
// /etc/docker/daemon.json
{"registry-mirrors": ["https://9cpn8tt6.mirror.aliyuncs.com", "..."]}
```

```bash
systemctl daemon-reload && systemctl restart docker
```

## 装 ha

跑官方 `install-homeassistant.sh` 一键脚本（按文档选对应设备类型）。


![hacs 插件](assets/homeassistant-docker-install/img2.png)

![hacs 插件](assets/homeassistant-docker-install/img3.png)

![hacs 插件](assets/homeassistant-docker-install/img4.png)

![hacs 插件](assets/homeassistant-docker-install/img5.png)

![hacs 插件](assets/homeassistant-docker-install/img6.png)

![hacs 插件](assets/homeassistant-docker-install/img7.png)

![hacs 插件](assets/homeassistant-docker-install/img8.png)

![hacs 插件](assets/homeassistant-docker-install/img9.png)

![hacs 插件](assets/homeassistant-docker-install/img10.png)

![hacs 插件](assets/homeassistant-docker-install/img11.png)

![hacs 插件](assets/homeassistant-docker-install/img12.png)
## 装 HACS（第三方插件商城）

```bash
docker exec -it homeassistant bash
mkdir -p /config/custom_components /config/www
cd /config/custom_components
wget -O - https://get.hacs.xyz | bash -
docker restart <ha容器>
```

重启后再登录即可见 HACS 入口，可装各类第三方集成。HACS 安装失败多为网络问题，挂代理或换源重试。

![安装](assets/homeassistant-docker-install/img1.png)

---
**相关**： [[00-索引-玩机NAS索引]] [[随身 WiFi 刷 Debian + Samba 共享]]

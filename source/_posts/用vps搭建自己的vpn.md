---
title: 用vps搭建自己的vpn
date: 2026-07-07 13:20:13
tags:
---

要求ubuntu22.04-LTS系统的服务器，需要公网IP，必须在国外，我使用的是recknerd,还挺好用的。

防火墙组开放443端口。

## 执行一键部署脚本

部署脚本是setup-hysteria.sh这个文件，名字无所谓，主要是内容如下：

PASSWORD="password123" 是示例，实际上可以设置复杂一些，在搭配fail2ban这样可以保证服务器安全不被爆破

~~~
#!/bin/bash
set -e

# 注意，要赋予此脚本执行权限：chmod +x setup-hysteria.sh
# 然后在执行：./setup-hysteria.sh

# ==================== 配置变量（按需修改） ====================
PASSWORD="password123"
LISTEN_PORT="443"
MASQUERADE_URL="https://www.bing.com"
CERT_DAYS="365"
HY_VERSION="v2.8.1"
# ============================================================

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

echo -e "${GREEN}========================================${NC}"
echo -e "${GREEN}   Hysteria ${HY_VERSION} VPN 服务器安装   ${NC}"
echo -e "${GREEN}          Ubuntu 22.04 专用           ${NC}"
echo -e "${GREEN}========================================${NC}"

if [[ $EUID -ne 0 ]]; then
    echo -e "${RED}错误：请使用 root 用户执行此脚本 (sudo ./script.sh)${NC}"
    exit 1
fi

echo -e "${YELLOW}[1/7] 更新系统并安装依赖...${NC}"
apt update -qq
apt install -y -qq wget curl openssl ufw

echo -e "${YELLOW}[2/7] 创建目录结构...${NC}"
mkdir -p /etc/hysteria /etc/ssl/hysteria

echo -e "${YELLOW}[3/7] 生成 SSL 证书（有效期${CERT_DAYS}天）...${NC}"
openssl req -x509 -newkey rsa:4096 -nodes \
    -keyout /etc/ssl/hysteria/key.pem \
    -out /etc/ssl/hysteria/cert.pem \
    -days ${CERT_DAYS} \
    -subj "/CN=www.bing.com"

chmod 644 /etc/ssl/hysteria/key.pem
chmod 644 /etc/ssl/hysteria/cert.pem
echo -e "${GREEN}✓ 证书权限已设置为 644${NC}"

echo -e "${YELLOW}[4/7] 创建配置文件 /etc/hysteria/config.yaml ...${NC}"
cat > /etc/hysteria/config.yaml << YAML
listen: :${LISTEN_PORT}

tls:
  cert: /etc/ssl/hysteria/cert.pem
  key: /etc/ssl/hysteria/key.pem

auth:
  type: password
  password: ${PASSWORD}

masquerade:
  type: proxy
  proxy:
    url: ${MASQUERADE_URL}
    rewriteHost: true

quic:
  initStreamReceiveWindow: 8388608
  maxStreamReceiveWindow: 8388608
  initConnReceiveWindow: 20971520
  maxConnReceiveWindow: 20971520
YAML

echo -e "${YELLOW}[5/7] 使用官方脚本安装 Hysteria ${HY_VERSION} ...${NC}"
bash <(curl -fsSL https://get.hy2.sh/) --version ${HY_VERSION}

echo -e "${YELLOW}[6/7] 配置防火墙 (ufw)...${NC}"
ufw allow ${LISTEN_PORT}/udp
echo -e "${GREEN}已允许 UDP ${LISTEN_PORT} 端口${NC}"

echo -e "${YELLOW}[7/7] 重启 Hysteria 服务并应用配置...${NC}"
systemctl stop hysteria-server || true
systemctl start hysteria-server
systemctl enable hysteria-server
sleep 3

if systemctl is-active --quiet hysteria-server; then
    SERVER_IP=$(curl -s ifconfig.me)
    echo -e "\n${GREEN}========================================${NC}"
    echo -e "${GREEN}✓ Hysteria 部署成功！${NC}"
    echo -e "${GREEN}========================================${NC}"
    echo -e "${YELLOW}服务状态：${NC}$(systemctl status hysteria-server --no-pager | grep "Active:")"
    echo -e "${YELLOW}端口监听：${NC}"
    ss -tulnp | grep ":${LISTEN_PORT}" | grep -v grep || echo "  等待端口监听..."
    echo ""
    echo -e "${GREEN}客户端连接信息：${NC}"
    echo -e "  服务器地址：${SERVER_IP}:${LISTEN_PORT}"
    echo -e "  密码：${PASSWORD}"
    echo -e "  协议：Hysteria ${HY_VERSION}"
    echo ""
    echo -e "${YELLOW}常用管理命令：${NC}"
    echo -e "  查看状态: systemctl status hysteria-server"
    echo -e "  查看日志: journalctl -u hysteria-server -f"
    echo -e "  重启服务: systemctl restart hysteria-server"
    echo -e "  停止服务: systemctl stop hysteria-server"
else
    echo -e "${RED}服务启动失败！查看错误日志：${NC}"
    journalctl -u hysteria-server -n 20 --no-pager
    exit 1
fi
~~~

然后，把这个`setup-hysteria.sh`脚本丢到服务器上（ssh链接）比如笔者是放在var目录下的

然后`chmod +x setup-hysteria.sh`给权限，再`./setup-hysteria.sh`就可以一键部署好vpn服务了

查看状态信息

~~~
root@racknerd-eca11d5:~# systemctl status hysteria-server
● hysteria-server.service - Hysteria Server Service (config.yaml)
     Loaded: loaded (/etc/systemd/system/hysteria-server.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2026-07-07 03:07:55 UTC; 3 weeks 5 days ago
   Main PID: 898026 (hysteria)
      Tasks: 8 (limit: 1611)
     Memory: 15.6M
        CPU: 46min 20.995s
     CGroup: /system.slice/hysteria-server.service
             └─898026 /usr/local/bin/hysteria server --config /etc/hysteria/config.yaml

Aug 01 02:49:33 racknerd-eca11d5 hysteria[898026]: 2026-08-01T02:49:33Z        INFO        client disconnected      >
Aug 01 02:49:37 racknerd-eca11d5 hysteria[898026]: 2026-08-01T02:49:37Z        INFO        client connected        {>
Aug 01 02:49:42 racknerd-eca11d5 hysteria[898026]: 2026-08-01T02:49:42Z        INFO        client connected        {>
Aug 01 02:49:43 racknerd-eca11d5 hysteria[898026]: 2026-08-01T02:49:43Z        INFO        client disconnected      >
Aug 01 02:51:45 racknerd-eca11d5 hysteria[898026]: 2026-08-01T02:51:45Z        INFO        client disconnected      >
Aug 01 03:07:55 racknerd-eca11d5 hysteria[898026]: 2026-08-01T03:07:55Z        INFO        update available        {>
Aug 02 03:07:55 racknerd-eca11d5 hysteria[898026]: 2026-08-02T03:07:55Z        INFO        update available        {>
Aug 02 06:46:46 racknerd-eca11d5 hysteria[898026]: 2026-08-02T06:46:46Z        INFO        client connected        {>
Aug 02 06:49:41 racknerd-eca11d5 hysteria[898026]: 2026-08-02T06:49:41Z        WARN        TCP error        {"addr":>
~~~

## 使用clash-verge-rev进行订阅VPN服务（通过配置文件的方式）

然后准备一个conf.yaml文件，内容如下：

注意：server: 64.176.80.218 就是 VPS服务器的ip
password: "password123" 也就是服务器的VPN的密码
***setup-hysteria.sh 这个文件里面配置信息要对上***
rule-providers也可以根据个人情况，适当修改

~~~
# ========== 代理节点配置 ==========
proxies:
  - name: "VPS-Hysteria2"
    type: hysteria2
    server: 64.176.80.218
    port: 443
    password: "password123"
    sni: www.bing.com
    skip-cert-verify: true
    # 以下为可选优化参数
    up: "100 Mbps"
    down: "500 Mbps"

# ========== 规则集配置（可选，用于增强分流）==========
rule-providers:
  reject:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/reject.txt"
    path: ./ruleset/reject.yaml
    interval: 86400

  icloud:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/icloud.txt"
    path: ./ruleset/icloud.yaml
    interval: 86400

  apple:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/apple.txt"
    path: ./ruleset/apple.yaml
    interval: 86400

  google:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/google.txt"
    path: ./ruleset/google.yaml
    interval: 86400

  proxy:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/proxy.txt"
    path: ./ruleset/proxy.yaml
    interval: 86400

  direct:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/direct.txt"
    path: ./ruleset/direct.yaml
    interval: 86400

  gfw:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/gfw.txt"
    path: ./ruleset/gfw.yaml
    interval: 86400

  tld-not-cn:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/tld-not-cn.txt"
    path: ./ruleset/tld-not-cn.yaml
    interval: 86400

  telegramcidr:
    type: http
    behavior: ipcidr
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/telegramcidr.txt"
    path: ./ruleset/telegramcidr.yaml
    interval: 86400

  cncidr:
    type: http
    behavior: ipcidr
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/cncidr.txt"
    path: ./ruleset/cncidr.yaml
    interval: 86400

  lancidr:
    type: http
    behavior: ipcidr
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/lancidr.txt"
    path: ./ruleset/lancidr.yaml
    interval: 86400

  applications:
    type: http
    behavior: classical
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/applications.txt"
    path: ./ruleset/applications.yaml
    interval: 86400

# ========== 代理组配置 ==========
proxy-groups:
  - name: "🚀 节点选择"
    type: select
    proxies:
      - "VPS-Hysteria2"
      - "DIRECT"

  - name: "🎬 流媒体"
    type: select
    proxies:
      - "VPS-Hysteria2"
      - "DIRECT"

  - name: "🤖 AI服务"
    type: select
    proxies:
      - "VPS-Hysteria2"
      - "DIRECT"

# ========== 规则配置 ==========
rules:
  # ===== 1. 规则集分流（如果不想用可以删除本块）=====
  - RULE-SET,applications,DIRECT
  - RULE-SET,icloud,DIRECT
  - RULE-SET,apple,DIRECT
  - RULE-SET,google,🚀 节点选择
  - RULE-SET,proxy,🚀 节点选择
  - RULE-SET,direct,DIRECT
  - RULE-SET,lancidr,DIRECT
  - RULE-SET,cncidr,DIRECT
  - RULE-SET,telegramcidr,🚀 节点选择

  # ===== 2. 国内网站强制直连 =====
  # 通用规则
  - DOMAIN-SUFFIX,cn,DIRECT
  - GEOIP,CN,DIRECT,no-resolve
  - GEOSITE,CN,DIRECT

  # 常见国内网站关键词
  - DOMAIN-KEYWORD,baidu,DIRECT
  - DOMAIN-KEYWORD,taobao,DIRECT
  - DOMAIN-KEYWORD,alipay,DIRECT
  - DOMAIN-KEYWORD,qq,DIRECT
  - DOMAIN-KEYWORD,weixin,DIRECT
  - DOMAIN-KEYWORD,bilibili,DIRECT
  - DOMAIN-KEYWORD,bytedance,DIRECT
  - DOMAIN-KEYWORD,zhihu,DIRECT
  - DOMAIN-KEYWORD,jd,DIRECT
  - DOMAIN-KEYWORD,meituan,DIRECT
  - DOMAIN-KEYWORD,douyin,DIRECT
  - DOMAIN-KEYWORD,pinduoduo,DIRECT

  # 局域网与保留地址
  #- IP-CIDR,192.168.0.0/16,DIRECT,no-resolve
  #- IP-CIDR,10.0.0.0/8,DIRECT,no-resolve
  #- IP-CIDR,172.16.0.0/12,DIRECT,no-resolve
  #- IP-CIDR,127.0.0.0/8,DIRECT,no-resolve
  #- IP-CIDR,100.64.0.0/10,DIRECT,no-resolve
  #- IP-CIDR,17.0.0.0/8,DIRECT,no-resolve

  # ===== 3. AI 服务走代理 =====
  # OpenAI
  - DOMAIN-KEYWORD,openai,🤖 AI服务
  - DOMAIN-SUFFIX,openai.com,🤖 AI服务
  - DOMAIN-SUFFIX,chatgpt.com,🤖 AI服务
  - DOMAIN-SUFFIX,ai.com,🤖 AI服务
  - DOMAIN-SUFFIX,oaistatic.com,🤖 AI服务
  - DOMAIN-SUFFIX,oaiusercontent.com,🤖 AI服务
  - DOMAIN-KEYWORD,chatgpt,🤖 AI服务

  # Anthropic (Claude)
  - DOMAIN-SUFFIX,anthropic.com,🤖 AI服务
  - DOMAIN-SUFFIX,claude.ai,🤖 AI服务

  # Google (Gemini/Bard/DeepMind)
  - DOMAIN-SUFFIX,gemini.google.com,🤖 AI服务
  - DOMAIN-SUFFIX,bard.google.com,🤖 AI服务
  - DOMAIN-SUFFIX,deepmind.google,🤖 AI服务
  - DOMAIN-SUFFIX,deepmind.com,🤖 AI服务
  - DOMAIN-SUFFIX,ai.google.dev,🤖 AI服务
  - DOMAIN-SUFFIX,generativeai.google,🤖 AI服务
  - DOMAIN-SUFFIX,proactivebackend-pa.googleapis.com,🤖 AI服务
  - DOMAIN-KEYWORD,generativelanguage,🤖 AI服务

  # Meta (Llama)
  - DOMAIN-SUFFIX,meta.ai,🤖 AI服务
  - DOMAIN-SUFFIX,llama.com,🤖 AI服务
  - DOMAIN-SUFFIX,llama.meta.com,🤖 AI服务

  # 其他海外AI服务
  - DOMAIN-SUFFIX,perplexity.ai,🤖 AI服务
  - DOMAIN-SUFFIX,pplx.ai,🤖 AI服务
  - DOMAIN-KEYWORD,perplexity,🤖 AI服务
  - DOMAIN-SUFFIX,x.ai,🤖 AI服务
  - DOMAIN-KEYWORD,grok,🤖 AI服务
  - DOMAIN-SUFFIX,poe.com,🤖 AI服务
  - DOMAIN-SUFFIX,you.com,🤖 AI服务

  # Hugging Face (AI模型社区)
  - DOMAIN-SUFFIX,huggingface.co,🤖 AI服务
  - DOMAIN-SUFFIX,hf.co,🤖 AI服务

  # 平台/聚合类AI服务
  - DOMAIN-SUFFIX,openrouter.ai,🤖 AI服务
  - DOMAIN-SUFFIX,together.ai,🤖 AI服务

  # Cursor AI 编辑器
  - DOMAIN-SUFFIX,cursor.com,🤖 AI服务
  - DOMAIN-SUFFIX,cursor.sh,🤖 AI服务
  - DOMAIN-SUFFIX,cursor-cdn.com,🤖 AI服务
  - DOMAIN-SUFFIX,workos.com,🤖 AI服务
  - DOMAIN-SUFFIX,challenges.cloudflare.com,🤖 AI服务

  # Amazon Kiro / Amazon AI 服务
  - DOMAIN-SUFFIX,kiro.dev,🤖 AI服务
  - DOMAIN-SUFFIX,amazonkiro.com,🤖 AI服务
  - DOMAIN-KEYWORD,kiro,🤖 AI服务
  - DOMAIN-SUFFIX,aws.amazon.com,🤖 AI服务
  - DOMAIN-SUFFIX,amazonaws.com,🤖 AI服务
  - DOMAIN-SUFFIX,bedrock.aws,🤖 AI服务
  - DOMAIN-KEYWORD,amazonbedrock,🤖 AI服务
  - DOMAIN-SUFFIX,q.aws.amazon.com,🤖 AI服务
  - DOMAIN-SUFFIX,codecatalyst.aws,🤖 AI服务
  - DOMAIN-SUFFIX,sagemaker.aws,🤖 AI服务

  # 国内AI服务 (默认直连，如需走代理请取消注释并修改策略)
  # - DOMAIN-SUFFIX,deepseek.com,DIRECT
  # - DOMAIN-SUFFIX,yiyan.baidu.com,DIRECT
  # - DOMAIN-SUFFIX,tongyi.aliyun.com,DIRECT
  # - DOMAIN-SUFFIX,doubao.com,DIRECT
  # - DOMAIN-SUFFIX,chatglm.cn,DIRECT
  # - DOMAIN-SUFFIX,xinghuo.xfyun.cn,DIRECT
  # - DOMAIN-SUFFIX,kimi.moonshot.cn,DIRECT
  # - DOMAIN-SUFFIX,yuanbao.tencent.com,DIRECT

  # ===== 4. 流媒体走代理 =====
  - DOMAIN-KEYWORD,youtube,🎬 流媒体
  - DOMAIN-KEYWORD,netflix,🎬 流媒体
  - DOMAIN-KEYWORD,disney,🎬 流媒体
  - DOMAIN-KEYWORD,hbo,🎬 流媒体
  - DOMAIN-KEYWORD,hulu,🎬 流媒体
  - DOMAIN-KEYWORD,spotify,🎬 流媒体
  - DOMAIN-KEYWORD,twitch,🎬 流媒体
  - DOMAIN-SUFFIX,googlevideo.com,🎬 流媒体
  - DOMAIN-SUFFIX,ytimg.com,🎬 流媒体
  - DOMAIN-SUFFIX,ggpht.com,🎬 流媒体
  - DOMAIN-SUFFIX,fastly.com,🎬 流媒体

  # ===== 5. 其他常用国外服务走代理 =====
  - DOMAIN-KEYWORD,github,🚀 节点选择
  - DOMAIN-SUFFIX,github.com,🚀 节点选择
  - DOMAIN-SUFFIX,github.io,🚀 节点选择
  - DOMAIN-SUFFIX,githubassets.com,🚀 节点选择
  - DOMAIN-SUFFIX,githubusercontent.com,🚀 节点选择
  - DOMAIN-KEYWORD,google,🚀 节点选择
  - DOMAIN-KEYWORD,twitter,🚀 节点选择
  - DOMAIN-KEYWORD,facebook,🚀 节点选择
  - DOMAIN-KEYWORD,instagram,🚀 节点选择
  - DOMAIN-KEYWORD,reddit,🚀 节点选择
  - DOMAIN-KEYWORD,telegram,🚀 节点选择
  - DOMAIN-KEYWORD,whatsapp,🚀 节点选择
  - DOMAIN-KEYWORD,zoom,🚀 节点选择
  - DOMAIN-KEYWORD,slack,🚀 节点选择
  - DOMAIN-KEYWORD,notion,🚀 节点选择

  # ===== 6. 最终兜底规则 =====
  # 所有未被上述规则匹配的流量，默认走代理节点
  - MATCH,🚀 节点选择
~~~

然后，在clash的订阅这里，新建、Local、随便起个名字，再上传刚刚准备好的订阅conf.yaml配置文件

然后，点击的保存按钮，再右键使用，就订阅好了。

至此，VPN搞定完毕，就可以正常访问github，用谷歌搜索学习代码知识啦
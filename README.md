# AI 工作站部署文档

> **主机名**: llama-225-195  
> **操作系统**: Debian 11  
> **GPU**: 2× NVIDIA RTX 3090 (24GB) + 1× NVIDIA RTX 5060 (规划中)  
> **文档生成日期**: 2026-09-07

---

## 一、系统概述

### 1.1 硬件清单

| 组件 | 型号/规格 | 用途 |
|:---|:---|:---|
| GPU ×2 | NVIDIA RTX 3090 (24GB GDDR6X) | Ollama LLM 推理（双卡分摊 27B 模型） |
| GPU ×1 | NVIDIA RTX 5060 | **规划中**（ComfyUI / MiniMax H3） |
| Docker Engine | 最新版本 | 容器化服务部署 |
| NVIDIA Driver + CUDA | 支持 RTX 3090 | GPU 计算 |

### 1.2 网络拓扑

```
外部访问
  │
  ▼
┌──────────────────────────────────────────────────────────┐
│              llama-225-195 (Debian 11)                    │
│                                                          │
│  ┌────────┐  ┌──────────┐  ┌────────┐  ┌──────────────┐ │
│  │Anything│  │  SearXNG │  │MCP-    │  │  MCP-        │ │
│  │LLM     │  │  :8080   │  │Chrome  │  │  Codebase    │ │
│  │  :3001 │  │(Docker)  │  │  :3002 │  │   :3003      │ │
│  └───┬────┘  └────┬─────┘  └───┬────┘  └──────┬───────┘ │
│      │            │            │               │         │
│      ▼            │            ▼               ▼         │
│  ┌────────────────┴────────────────────────────────────┐ │
│  │              Ollama (systemd) :11434                 │ │
│  │     Qwen3.8-27B-Uncensored | 双 3090 | ~44 t/s     │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                          │
│  ┌─────────────────────────────────────────────────────┐ │
│  │         HTTP Exec API :8931 (物理机命令代理)         │ │
│  │    容器通过 172.17.0.1:8931 执行宿主机命令           │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                          │
│  出网代理: 172.17.0.1:1081 (SearXNG 引擎访问外网)       │
└──────────────────────────────────────────────────────────┘
```

### 1.3 端口分配总表

| 端口 | 服务 | 绑定地址 | 说明 |
|:---|:---|:---|:---|
| **3001** | AnythingLLM | 0.0.0.0 | RAG + Agent 主入口 |
| **3002** | MCP-Chrome (Puppeteer) | 0.0.0.0 | 浏览器自动化 MCP Server |
| **3003** | MCP-Codebase (Filesystem) | 0.0.0.0 | 代码文件系统 MCP Server |
| **8080** | SearXNG | 0.0.0.0 | 元搜索引擎（88+ 引擎） |
| **8931** | HTTP Exec API | 0.0.0.0 | 容器 → 宿主机命令执行代理 |
| **11434** | Ollama | 0.0.0.0 | LLM 推理 API |
| **1081** | HTTP 代理 | 127.0.0.1 | SearXNG 出网代理（远程） |

---

## 二、架构总览与数据流

### 2.1 组件协同关系

```
用户浏览器 (http://<IP>:3001)
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│                     AnythingLLM (Docker)                     │
│                                                             │
│  ┌─────────┐    ┌──────────────┐    ┌─────────────────────┐ │
│  │  Agent  │    │    RAG       │    │   MCP Tools         │ │
│  │ 编排层  │    │ (LanceDB     │    │  · Chrome Puppeteer │ │
│  │         │    │  + nomic-    │    │  · Codebase FS      │ │
│  │         │    │  embed-text) │    │                     │ │
│  └────┬────┘    └──────┬───────┘    └──────────┬──────────┘ │
│       │                │                        │            │
└───────┼────────────────┼────────────────────────┼────────────┘
        │                │                        │
        ▼                ▼                        ▼
┌──────────────┐  ┌──────────────┐      ┌──────────────────┐
│   Ollama     │  │   Ollama     │      │  MCP-Chrome :3002 │
│  (LLM推理)   │  │ (Embedding)  │      │  (Puppeteer)      │
│  Qwen3.8-27B │  │ nomic-embed  │      │  网页浏览/截图     │
│  双3090      │  │  向量嵌入     │      └──────────────────┘
│  ~44 t/s     │  └──────────────┘      ┌──────────────────┐
└──────────────┘                        │  MCP-Codebase     │
                                        │  :3003            │
┌──────────────┐                        │  代码文件读写      │
│   SearXNG    │                        │  /root/anything-  │
│   :8080      │                        │   code            │
│  88+引擎     │                        └──────────────────┘
│  出网代理    │
└──────┬───────┘
       │
       ▼ (via proxy 172.17.0.1:1081)
   互联网搜索引擎
```

### 2.2 核心数据流路径

| 场景 | 数据流 |
|:---|:---|
| **Agent 联网搜索** | 用户提问 → AnythingLLM Agent → SearXNG(:8080) → 互联网(经代理) → 结果回传 → Ollama 总结 |
| **RAG 问答** | 用户提问 → AnythingLLM → nomic-embed-text(向量化) → LanceDB(检索) → Ollama(生成回答) |
| **代码分析** | Agent 指令 → MCP-Codebase(:3003) → 读取 /root/anything-code → 结果回传 Ollama |
| **网页自动化** | Agent 指令 → MCP-Chrome(:3002) → Puppeteer 操控浏览器 → 截图/DOM → 回传 Ollama |
| **容器执行宿主机命令** | AnythingLLM → HTTP Exec API(:8931) → 白名单校验 → subprocess 执行 → JSON 返回 |
| **编码(外部客户端)** | opencode/其他客户端 → Ollama(:11434) → Qwen3.8-27B → 代码输出 |

---

## 三、各模块部署细节

### 3.1 Ollama（LLM 推理引擎）

**部署方式**: systemd 服务（宿主机原生）  
**端口**: 11434  
**GPU 分配**: 双 RTX 3090（`CUDA_VISIBLE_DEVICES=0,1`）

#### 当前加载模型

| 模型 | 用途 | 说明 |
|:---|:---|:---|
| `orcarouter/Qwen3.8-27B-Uncensored` | 主推理 LLM | 27B 参数，双卡分摊，~44 t/s |
| `nomic-embed-text` | RAG 向量化 | 轻量 embedding 模型 |

#### 性能指标（实测日志）

```
# 单卡 3090 推理（27B 量化）
slot print_timing: n_gen=660, tg=13.13 t/s, tg_3s=13.12 t/s

# 双卡 3090 推理（27B）
输出速度: ~44 t/s
```

#### 关键配置

```bash
# systemd 服务
systemctl status ollama
systemctl restart ollama

# 查看当前加载模型
curl http://localhost:11434/api/ps

# 查看 GPU 分配
nvidia-smi
```

---

### 3.2 AnythingLLM（RAG + Agent 编排）

**部署方式**: Docker Compose  
**目录**: `/root/anything-setup/`  
**端口**: 3001  
**存储卷**: `/root/anythingllm` → 容器内 `/app/server/storage`  
**代码挂载**: `/root/anything-code` → 容器内 `/code`

#### 关键环境配置

| 变量 | 值 | 说明 |
|:---|:---|:---|
| `LLM_PROVIDER` | `ollama` | LLM 后端 |
| `OLLAMA_BASE_PATH` | `http://172.17.0.1:11434` | 宿主机 Ollama |
| `OLLAMA_MODEL_PREF` | `orcarouter/Qwen3.8-27B-Uncensored:latest` | 主模型 |
| `OLLAMA_MODEL_TOKEN_LIMIT` | `4096` | 最大 token |
| `EMBEDDING_ENGINE` | `ollama` | 向量引擎 |
| `EMBEDDING_MODEL_PREF` | `nomic-embed-text:latest` | 嵌入模型 |
| `EMBEDDING_MODEL_MAX_CHUNK_LENGTH` | `8192` | 分块长度 |
| `VECTOR_DB` | `lancedb` | 向量数据库 |
| `WHISPER_PROVIDER` | `local` | 本地语音转文字 |
| `TTS_PROVIDER` | `native` | 本地 TTS |
| `HTTP_TIMEOUT` | `300000` (5min) | HTTP 超时 |
| `OLLAMA_TIMEOUT` | `300000` (5min) | LLM 超时 |

#### Docker Compose 文件（原文）

> 文件路径: `/root/anything-setup/docker-compose.yml`
> 
> 目前searxng并未集成稍后补充再更新

```yaml
version: '3.3'
services:
  anythingllm:
    image: mintplexlabs/anythingllm
    container_name: anythingllm
    ports:
      - "3001:3001"
    cap_add:
      - SYS_ADMIN
    environment:
      - STORAGE_DIR=/app/server/storage
      - JWT_SECRET="make this a large list of random numbers and letters 20+"
      - LLM_PROVIDER=ollama
      - OLLAMA_BASE_PATH=http://172.17.0.1:11434
      - OLLAMA_MODEL_PREF=orcarouter/Qwen3.8-27B-Uncensored:latest
      - OLLAMA_MODEL_TOKEN_LIMIT=4096
      - EMBEDDING_ENGINE=ollama
      - EMBEDDING_BASE_PATH=http://172.17.0.1:11434
      - EMBEDDING_MODEL_PREF=nomic-embed-text:latest
      - EMBEDDING_MODEL_MAX_CHUNK_LENGTH=8192
      - VECTOR_DB=lancedb
      - WHISPER_PROVIDER=local
      - TTS_PROVIDER=native
      - PASSWORDMINCHAR=8
      - HTTP_TIMEOUT=300000
      - OLLAMA_TIMEOUT=300000
    volumes:
      - anythingllm_storage:/app/server/storage
      # 新增：把宿主机想让 AI 读取的代码路径挂载到容器内的 /code
      - /root/anything-code:/code
    restart: always


# Chrome DevTools / Puppeteer 自动化 MCP
  mcp-chrome:
    image: mcr.microsoft.com/playwright:v1.41.0-jammy
    container_name: mcp-chrome
    restart: always
    environment:
      - PUPPETEER_SKIP_CHROMIUM_DOWNLOAD=true
      - HTTP_PROXY=http://172.17.0.1:1081
      - HTTPS_PROXY=http://172.17.0.1:1081
      - NO_PROXY=127.0.0.1,172.17.0.0/16,172.19.0.0/16
    command: >
      bash -c "npm config set registry https://mirrors.cloud.tencent.com/npm/ &&
               npm install -g supergateway @modelcontextprotocol/server-puppeteer &&
               while true; do
                 supergateway --stdio 'mcp-server-puppeteer' --port 3001 --host 0.0.0.0 --stateful
                 echo 'Supergateway exited, restarting in 1 second...'
                 sleep 1
               done"
    ports:
      - "3002:3001"

  mcp-codebase:
    image: node:20-alpine
    container_name: mcp-codebase
    restart: always
    working_dir: /code
    volumes:
      - /root/anything-code:/code  # 替换为你宿主机真实的代码路径
    # 使用 node 直接执行 npx，不再多重嵌套
    command: >
         sh -c "npm config set registry https://mirrors.cloud.tencent.com/npm/ &&
             npm install -g supergateway @modelcontextprotocol/server-filesystem &&
             while true; do
               supergateway --stdio 'mcp-server-filesystem /code' --port 3002 --host 0.0.0.0 --stateful
               echo 'Supergateway exited, restarting in 1 second...'
               sleep 1
             done"
    ports:
      - "3003:3002"

volumes:
  anythingllm_storage:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /root/anythingllm
```

#### 启动命令

```bash
cd /root/anything-setup
docker compose up -d
docker compose logs -f
```

---

### 3.3 HTTP Exec API（容器 ↔ 宿主机通信桥梁）

**作用**: 让 Docker 容器内的 AnythingLLM/MCP 服务能够调用宿主机的命令（如 MySQL、文件操作、网络诊断等），同时通过白名单机制保证安全。  
**端口**: 8931  
**文件路径**: `/root/anything-setup/httpexec-api.py`  
**容器内访问地址**: `http://172.17.0.1:8931/`

#### 完整源码（原文）

```python
#!/usr/bin/env python3
"""
极简单的命令执行代理。
用法: python3 api.py --port 8931
容器内访问: http://172.17.0.1:8931/
"""
from http.server import HTTPServer, BaseHTTPRequestHandler
from urllib.parse import urlparse, parse_qs
import subprocess, json, sys, os

PORT = 8931

class Handler(BaseHTTPRequestHandler):
    def _respond(self, data: dict, code=200):
        self.send_response(code)
        self.send_header("Content-Type", "application/json")
        self.end_headers()
        self.wfile.write(json.dumps(data, ensure_ascii=False).encode("utf-8"))

    def do_GET(self):
        """
        GET /exec?cmd=<url-encoded command>
        例: /exec?cmd=echo%20hello
        例: /exec?cmd=mysql%20-h%2010.5.254.166%20-u%20root%20-e%20%22SELECT%201%22
        """
        qs = parse_qs(urlparse(self.path).query)
        cmd = qs.get("cmd", ["echo 'no cmd'"])[0]

        # 安全: 可选白名单前缀
        #allowed = ("mysql", "python", "python3", "echo", "ls", "cat", "grep", "which", "/bin/bash","find")
        allowed = (
            "mysql", "python", "python3", "echo", "ls", "cat", "grep", "which",
            "/bin/bash", "bash", "sh", "find",
            "head", "tail", "wc", "pwd", "df", "du", "stat", "file",
            "ps", "top", "free", "date", "whoami", "id", "hostname", "uname", "env",
            "mkdir", "touch", "cp", "mv", "ln", "realpath", "basename", "dirname",
            "chmod", "chown", "tar", "zip", "unzip", "gzip", "gunzip", "zcat", "zgrep",
            "awk", "sed", "cut", "tr", "sort", "uniq", "xargs", "tee",
            "curl", "wget", "ping", "nc", "ss", "netstat", "dig", "nslookup",
            "pip", "pip3", "npm", "npx", "node", "git", "docker",
            "dmesg", "journalctl", "pgrep", "lsof",
        )
        first_word = cmd.strip().split()[0] if cmd.strip() else ""
        if first_word not in allowed:
            self._respond({"ok": False, "error": f"command '{first_word}' not in whitelist"}, 403)
            return

        try:
            r = subprocess.run(
                cmd, shell=True, capture_output=True, text=True, timeout=60
            )
            self._respond({
                "ok": True,
                "stdout": r.stdout[:50000],
                "stderr": r.stderr[:50000],
                "code": r.returncode,
            })
        except subprocess.TimeoutExpired:
            self._respond({"ok": False, "error": "timeout (60s)"}, 504)
        except Exception as e:
            self._respond({"ok": False, "error": str(e)}, 500)

    def log_message(self, *a):
        pass  # 安静

if __name__ == "__main__":
    import argparse
    p = argparse.ArgumentParser()
    p.add_argument("--port", type=int, default=8931)
    p.add_argument("--bind", default="0.0.0.0")
    args = p.parse_args()
    print(f"Host Exec API listening on {args.bind}:{args.port}")
    HTTPServer((args.bind, args.port), Handler).serve_forever()
```

#### 使用方法

```bash
# 启动
python3 /root/anything-setup/httpexec-api.py --port 8931

# 测试（从容器内）
curl "http://172.17.0.1:8931/exec?cmd=echo%20hello"
# 返回: {"ok": true, "stdout": "hello\n", "stderr": "", "code": 0}

# 测试 MySQL 连接
curl "http://172.17.0.1:8931/exec?cmd=mysql%20-h%2010.5.254.166%20-u%20fudonghua_check_user%20-pFdh%23Check_2026!xQ%20csa_cmdb%20-e%20%22SELECT%201%22"
```

#### 白名单机制说明

| 类别 | 允许的命令 |
|:---|:---|
| 数据库 | `mysql` |
| 解释器 | `python`, `python3`, `node`, `npm`, `npx`, `pip`, `pip3` |
| 文件操作 | `ls`, `cat`, `find`, `head`, `tail`, `wc`, `mkdir`, `touch`, `cp`, `mv`, `ln`, `chmod`, `chown`, `tar`, `zip`, `unzip` |
| 文本处理 | `awk`, `sed`, `cut`, `tr`, `sort`, `uniq`, `xargs`, `tee`, `grep` |
| 网络 | `curl`, `wget`, `ping`, `nc`, `ss`, `netstat`, `dig`, `nslookup` |
| 系统 | `ps`, `top`, `free`, `date`, `whoami`, `id`, `hostname`, `uname`, `env`, `df`, `du`, `stat`, `file` |
| 运维 | `dmesg`, `journalctl`, `pgrep`, `lsof` |
| Shell | `/bin/bash`, `bash`, `sh` |
| 版本控制 | `git` |
| 容器 | `docker` |

> **设计经验**: 该白名单是长期调试中逐步完善的，覆盖了运维场景中 95% 以上的命令需求。超时 60s，输出截断 50000 字符。

---

### 3.4 SearXNG（元搜索引擎）

**部署方式**: Docker（独立容器）  
**镜像**: `searxng/searxng:latest`  
**端口**: 8080  
**配置目录**: `/root/searxng/`  
**出网代理**: `http://172.17.0.1:1081` / `https://172.17.0.1:1081`  
**当前状态**: **88 个引擎可用**

#### 目录结构

```
/root/searxng/
├── settings.yml              ← 主配置文件（88+ 引擎）
├── settings.yml.20260901     ← 历史备份
├── settings.yml.bak.1788250662 ← 时间戳备份
├── settings_check.py         ← 引擎检测 + 自动修复脚本
└── test.sh                   ← 基础连通性测试
```

#### settings.yml 关键配置（原文摘录）

> 完整文件较大（88+ 引擎定义），以下为关键配置段：

```yaml
general:
  debug: false
  instance_name: SearXNG
  enable_metrics: true

search:
  safe_search: 0
  default_lang: auto
  ban_time_on_fail: 5
  max_ban_time_on_fail: 120
  formats:
  - html
  - json

server:
  port: 8888
  bind_address: 127.0.0.1
  limiter: false
  public_instance: false
  secret_key: eN4MdepD226jMjmqiGDyyZuQKhkYj
  method: POST

outgoing:
  pool_connections: 100
  pool_maxsize: 20
  enable_http2: true
  request_timeout: 10.0
  max_request_timeout: 15.0
  proxies:
    all://:
    - http://172.17.0.1:1081
    - https://172.17.0.1:1081
```

#### 已启用的核心引擎（88个中部分示例）

| 引擎 | 类别 | 状态 |
|:---|:---|:---|
| baidu | general | ✅ 启用 |
| baidu images | images | ✅ 启用 |
| bing news | news | ✅ 启用 |
| bilibili | general | ✅ 启用 |
| github | it | ✅ 启用 |
| stackoverflow | it/q&a | ✅ 启用 |
| askubuntu | it/q&a | ✅ 启用 |
| arxiv | general | ✅ 启用 |
| reuters | general | ✅ 启用 |
| btdigg | general | ✅ 启用 |
| steam | general | ✅ 启用 |
| dailymotion | videos | ✅ 启用 |
| wikimedia 系列 (books/news/quote/source/species/dictionary/verse/voyage) | 多类 | ✅ 启用 |
| wikicommons.videos | videos | ✅ 启用 |
| mwmbl | general | ✅ 启用 |
| photon | images | ✅ 启用 |
| naver news | news | ✅ 启用 |
| ... 等共 88 个 | | |

#### settings_check.py（引擎检测 + 自动修复脚本 · 原文）

> 文件路径: `/root/searxng/settings_check.py`

```python
#!/usr/bin/env python3
import argparse
import os
import shutil
import sys
import time
import requests
import yaml

SETTINGS_FILE = 'settings.yml'
SEARXNG_URL = 'http://127.0.0.1:8080/search'


def load_config():
    """读取 YAML 配置文件"""
    if not os.path.exists(SETTINGS_FILE):
        print(
            f"[ERROR] 未找到 {SETTINGS_FILE}，请确保在 ~/searxng 目录下运行该脚本！"
        )
        sys.exit(1)
    with open(SETTINGS_FILE, 'r', encoding='utf-8') as f:
        return yaml.safe_load(f)


def backup_config():
    """备份当前配置文件"""
    timestamp = int(time.time())
    backup_path = f'{SETTINGS_FILE}.bak.{timestamp}'
    shutil.copyfile(SETTINGS_FILE, backup_path)
    print(f"[BACKUP] 已成功备份当前配置 -> {backup_path}")
    return backup_path


def check_engine(engine_name, query='test', timeout=5):
    """测试单个引擎连通性 (强制直连本地 127.0.0.1:8080)"""
    params = {'q': query, 'engines': engine_name, 'format': 'json'}
    # 强制禁用代理，确保请求直接打到本地容器
    proxies = {'http': None, 'https': None}

    try:
        resp = requests.get(
            SEARXNG_URL, params=params, timeout=timeout, proxies=proxies
        )
        if resp.status_code == 200:
            results = resp.json().get('results', [])
            return True, len(results)
        return False, 0
    except Exception:
        return False, 0


def cmd_check(args):
    """处理 check 命令"""
    config = load_config()
    engines = [item.get('name') for item in config.get('engines', [])]

    print(f"=== 开始测试 {len(engines)} 个搜索引擎 [直连本地 SearXNG 容器] ===")
    print(f"搜索词: '{args.query}' | 单超时: {args.timeout}s")
    print('-' * 65)

    working_engines = []

    for engine in engines:
        print(f"Testing [{engine:<18}] ... ", end='', flush=True)
        ok, count = check_engine(
            engine_name=engine, query=args.query, timeout=args.timeout
        )
        if ok and count > 0:
            print(f"✅ SUCCESS ({count} 条结果)")
            working_engines.append(engine)
        else:
            print("❌ FAILED (空结果/超时)")

    print('-' * 65)
    print(
        f"测试完成！可用引擎 ({len(working_engines)}/{len(engines)}): {working_engines}"
    )
    print('-' * 65)

    # 交互引导：询问用户是否将检测到的可用引擎写入配置
    if working_engines and not args.no_prompt:
        choice = (
            input(
                "\n是否立刻应用改动？(仅启用测试通过的可用引擎，将其余引擎设为 disable) [y/N]: "
            )
            .strip()
            .lower()
        )
        if choice == 'y':
            apply_change(working_engines, disable_all_others=True)
        else:
            print("已取消改动，未变更 settings.yml。")


def apply_change(
    target_engines, disable_mode=False, disable_all_others=False, no_restart=False
):
    """根据测试结果执行改动、备份与重启"""
    config = load_config()

    # 1. 自动执行备份
    backup_config()

    # 2. 修改结构
    engines_list = config.get('engines', [])
    updated = []

    for item in engines_list:
        name = item.get('name')
        if disable_all_others:
            if name in target_engines:
                item['disabled'] = False
                updated.append(name)
            else:
                item['disabled'] = True
        else:
            if name in target_engines:
                item['disabled'] = disable_mode
                updated.append(name)

    # 3. 写回文件
    with open(SETTINGS_FILE, 'w', encoding='utf-8') as f:
        yaml.dump(config, f, allow_unicode=True, sort_keys=False)

    print(f"[CHANGE] 已更新 {SETTINGS_FILE}！目前启用的引擎: {updated}")

    # 4. 重启容器
    if not no_restart:
        print("[DOCKER] 正在重启 SearXNG 容器以生效...")
        os.system('docker restart searxng')
        print("[DOCKER] 重启完成！")


def cmd_change(args):
    """处理手动执行 change 命令"""
    apply_change(
        args.engines,
        disable_mode=args.disable,
        disable_all_others=args.disable_all,
        no_restart=args.no_restart,
    )


def main():
    parser = argparse.ArgumentParser(
        description='SearXNG 引擎测试与管理工具 (干净版)'
    )
    subparsers = parser.add_subparsers(dest='command', required=True)

    # --- 命令 1: check ---
    check_parser = subparsers.add_parser('check', help='检查所有引擎可用性')
    check_parser.add_argument(
        '-q', '--query', default='test', help='测试搜索词 (默认: test)'
    )
    check_parser.add_argument(
        '-t', '--timeout', type=int, default=5, help='单请求超时时间 (默认: 5s)'
    )
    check_parser.add_argument(
        '--no-prompt', action='store_true', help='仅测试并打印，不主动提问修改'
    )
    check_parser.set_defaults(func=cmd_check)

    # --- 命令 2: change ---
    change_parser = subparsers.add_parser('change', help='手动应用修改并备份')
    change_parser.add_argument(
        'engines', nargs='+', help='需要修改的引擎名称列表'
    )
    change_parser.add_argument(
        '--disable', action='store_true', help='将指定引擎设为禁用'
    )
    change_parser.add_argument(
        '--disable-all',
        action='store_true',
        help='仅启用指定的引擎，将其余全部禁用',
    )
    change_parser.add_argument(
        '--no-restart', action='store_true', help='修改后不自动重启容器'
    )
    change_parser.set_defaults(func=cmd_change)

    args = parser.parse_args()
    args.func(args)


if __name__ == '__main__':
    main()
```

#### 使用方式

```bash
cd /root/searxng

# 检测所有引擎可用性
python3 settings_check.py check

# 检测并自动修复（仅保留可用引擎）
python3 settings_check.py check --no-prompt
# 然后手动:
python3 settings_check.py change <engine1> <engine2> --disable-all

# 手动禁用某个引擎
python3 settings_check.py change bing --disable

# 基础连通性测试（10次请求）
bash test.sh
```

#### test.sh（原文）

```bash
for i in $(seq 1 10); do
  echo -n "第${i}次: "
  curl -s -o /dev/null -w "%{http_code} (%{time_total}s)" "http://localhost:8080/search?q=test${i}&format=json"
  echo ""
  sleep 1
done
```

#### SearXNG 与 AnythingLLM 的协作

```
AnythingLLM Agent
    │
    │  HTTP GET /search?q=xxx&format=json
    ▼
SearXNG (:8080)
    │
    │  聚合 88 个引擎结果
    │  经代理 172.17.0.1:1081 出网
    ▼
互联网 (Baidu/Bing/Bilibili/GitHub/StackOverflow/...)
    │
    ▼
JSON 结果 → AnythingLLM → Ollama 总结 → 用户
```

---

### 3.5 MCP 服务（Model Context Protocol）

MCP 是让 LLM Agent 获得"工具能力"的标准化协议。本架构部署了两个 MCP Server：

#### 3.5.1 MCP-Chrome（浏览器自动化）

| 属性 | 值 |
|:---|:---|
| 镜像 | `mcr.microsoft.com/playwright:v1.41.0-jammy` |
| 端口 | 3002 (容器内 3001) |
| 协议 | MCP over HTTP (supergateway --stateful) |
| 能力 | 网页打开、截图、点击、填表、读取 DOM、执行 JS |
| 用途 | Agent 浏览网页、截图分析、自动化操作 |

#### 3.5.2 MCP-Codebase（代码文件系统）

| 属性 | 值 |
|:---|:---|
| 镜像 | `node:20-alpine` |
| 端口 | 3003 (容器内 3002) |
| 挂载 | `/root/anything-code` → `/code` |
| 协议 | MCP over HTTP (supergateway --stateful) |
| 能力 | 文件读写、目录遍历、代码搜索 |
| 用途 | Agent 直接读取/分析/修改代码仓库 |

#### MCP 与 AnythingLLM 的协作

```
AnythingLLM Agent 决定使用工具
    │
    ├──→ MCP-Chrome (:3002)
    │       例: "打开 https://github.com/xxx 并截图"
    │
    └──→ MCP-Codebase (:3003)
            例: "读取 /code/project/src/main.py 并分析函数结构"
```

> **设计经验**: supergateway 的 `--stateful` 模式保持会话状态，避免每次请求都重建浏览器/文件连接。`while true` 循环 + `sleep 1` 实现崩溃自动重启。

> 修改文件： /root/anythingllm/plugins/anythingllm_mcp_servers.json ，添加以下内容
```
{                                                                                                                                                                                            
  "mcpServers": {
    "chrome-devtools": {
      "url": "http://mcp-chrome:3001/sse"   // 访问 mcp-chrome 容器内部的 3001 端口
    },
    "codebase-memory": {
      "url": "http://mcp-codebase:3002/sse" // 访问 mcp-codebase 容器内部的 3002 端口
    }
  }
}
```

---

### 3.6 规划中的模块

| 模块 | 计划用途 | 当前状态 |
|:---|:---|:---|
| **ComfyUI** | 文生图（SDXL / Flux） | 规划中，等待 GPU 资源（5060 或释放一块 3090） |
| **MiniMax H3** | 文生视频（未审查模型） | 规划中，依赖 ComfyUI 或独立 API |
| **RTX 5060** | 为 ComfyUI / MiniMax 提供 GPU 算力 | 已安装，待分配 |

---

## 四、运维手册

### 4.1 服务启停

```bash
# === Ollama ===
systemctl start ollama
systemctl stop ollama
systemctl restart ollama
systemctl status ollama
journalctl -u ollama -f

# === AnythingLLM + MCP (Docker Compose) ===
cd /root/anything-setup
docker compose up -d        # 启动所有
docker compose down         # 停止所有
docker compose restart      # 重启
docker compose logs -f      # 查看日志

# === SearXNG (独立容器) ===
docker start searxng
docker stop searxng
docker restart searxng
docker logs -f searxng
docker exec -it searxng bash

# === docker 调试经验 ===
如果遇到容器无法访问与功能不健全，经常需要进入容器内部进行测试
docker exec -it (searxng|anythingllm|mcp-codebase|mcp-chrome)
通过curl等常用工具测试网络以及netstat -pln查看端口监听状态
docker mcp-chrome这个容器部署过程很久，要多等一等才能完成部署并正确启动
docker的外部接口访问受debian11系统的ufw影响，需要开启访问授权才可以访问。
11434/tcp                  ALLOW IN    172.16.0.0/12              # Allow Docker containers to access Ollama
8080                       ALLOW IN    172.16.0.0/12             
8931/tcp                   ALLOW IN    172.16.0.0/12              # AnythingLLM API


# === HTTP Exec API ===
# 前台运行
python3 /root/anything-setup/httpexec-api.py --port 8931

# 后台运行
nohup python3 /root/anything-setup/httpexec-api.py --port 8931 > /var/log/httpexec.log 2>&1 &
```

### 4.2 健康检查

```bash
# Ollama
curl http://localhost:11434/api/version
curl http://localhost:11434/api/ps

# AnythingLLM
curl -s http://localhost:3001/api/health

# SearXNG
curl -s "http://localhost:8080/search?q=test&format=json" | python3 -m json.tool

# MCP-Chrome
curl -s http://localhost:3002/

# MCP-Codebase
curl -s http://localhost:3003/

# HTTP Exec API
curl "http://localhost:8931/exec?cmd=echo%20ok"

# GPU 状态
nvidia-smi
```

### 4.3 常见问题排查

| 问题 | 排查步骤 |
|:---|:---|
| **Ollama 速度慢** | `nvidia-smi` 检查 GPU 利用率；`curl /api/ps` 确认模型加载；检查是否全层 offload |
| **容器访问宿主机服务超时** | 确认用 `172.17.0.1` 而非 `localhost`；检查 Exec API 是否存活 |
| **SearXNG 引擎批量失效** | `cd /root/searxng && python3 settings_check.py check`；检查代理 `172.17.0.1:1081` 是否存活 |
| **MCP 服务无响应** | `docker logs mcp-chrome`；确认 supergateway 进程存活；检查端口冲突 |
| **AnythingLLM 连不上 Ollama** | 容器内 `curl http://172.17.0.1:11434/api/version`；确认 Ollama 监听 0.0.0.0 |
| **显存不足 OOM** | 降低模型量化（Q6→Q4）；减小 `OLLAMA_MODEL_TOKEN_LIMIT`；检查是否有其他进程占显存 |

### 4.4 日志位置

| 服务 | 日志命令 |
|:---|:---|
| Ollama | `journalctl -u ollama --since "1 hour ago"` |
| AnythingLLM | `docker logs --tail 200 anythingllm` |
| MCP-Chrome | `docker logs --tail 200 mcp-chrome` |
| MCP-Codebase | `docker logs --tail 200 mcp-codebase` |
| SearXNG | `docker logs --tail 200 searxng` |
| HTTP Exec | `/var/log/httpexec.log`（若后台运行） |

---

## 五、关键经验总结

### 5.1 架构设计经验

1. **`172.17.0.1` 是容器访问宿主机的网关 IP**：所有容器内对宿主机的服务调用（Ollama:11434, Exec API:8931）都必须用这个地址，不能用 `localhost` 或 `127.0.0.1`。

2. **SearXNG 代理是关键瓶颈**：所有引擎出网都走 `172.17.0.1:1081`，代理挂了所有引擎都会超时。`settings_check.py` 检测时强制 `proxies=None` 直连本地容器，避免代理问题干扰判断。

3. **白名单 > 全开放**：Exec API 用命令白名单（而非 IP 白名单）更安全，因为容器网络是内网隔离的。白名单是逐步调试加出来的，覆盖了实际运维中 95% 的命令。

4. **supergateway --stateful**：MCP 服务必须用 stateful 模式，否则每次 HTTP 请求都会重建浏览器/文件连接，性能差 10 倍以上。

5. **双卡跑 27B 是性价比最优解**：单卡 3090 跑 27B 只有 ~13 t/s，双卡分摊后 ~44 t/s，体感从"能用"变成"流畅"。

### 5.2 文件备份策略

```
/root/searxng/
├── settings.yml              ← 当前生效
├── settings.yml.20260901     ← 日期备份
└── settings.yml.bak.<timestamp> ← 每次修改前自动备份

/root/anything-setup/
├── docker-compose.yml        ← 当前生效
└── docker-compose.yml.bak.20260904 ← 备份
```

---

## 六、未来扩展路线

| 优先级 | 模块 | 方案 | 依赖 |
|:---|:---|:---|:---|
| P0 | ComfyUI 文生图 | 5060 或释放 1 块 3090 部署 | GPU 资源 |
| P0 | MiniMax H3 文生视频 | ComfyUI 插件 / 独立 API | ComfyUI + GPU |
| P1 | Whisper STT | 5060 本地推理 | GPU 资源 |
| P1 | 语音对话闭环 | TTS + STT + Agent | 上述两者 |
| P2 | 监控告警 | Prometheus + Grafana | 新容器 |
| P2 | 多模型路由 | Ollama 多模型 + 按场景切换 | 显存规划 |

---

*文档结束*

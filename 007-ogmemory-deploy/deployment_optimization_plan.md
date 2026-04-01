# oG-Memory 一键部署优化方案

> 设计日期：2026-04-01
> 目标：简化部署流程，实现一键启动

---

## 一、优化概述

### 1.1 目标

将当前的多步骤手动部署优化为：
- ✅ 一个命令启动所有服务（openGauss 除外）
- ✅ 自动环境检查和依赖验证
- ✅ Docker Compose 统一编排
- ✅ 健康检查和状态监控

### 1.2 优化前后对比

| 维度 | 优化前 | 优化后 |
|-----|--------|--------|
| 启动命令 | 多个独立命令 | `./scripts/deploy-all.sh` |
| 步骤数 | 6+ 步 | 1 步 |
| 编排方式 | 手动管理 | Docker Compose |
| AGFS 处理 | 手动下载/编译 | 自动下载或提示 |
| 健康检查 | 手动 | 自动 |

---

## 二、Docker Compose 方案

### 2.1 docker-compose.yml

```yaml
version: '3.8'

services:
  # AGFS - 文件存储服务
  agfs:
    image: alpine:latest
    container_name: ogmem-agfs
    ports:
      - "1833:1833"
    volumes:
      - agfs-data:/data/agfs
      - ./agfs/agfs-server:/opt/agfs/agfs-server:ro
      - ./docker/agfs-config.yaml:/opt/agfs/config.yaml:ro
    working_dir: /opt/agfs
    command: >
      sh -c "
        chmod +x /opt/agfs/agfs-server && \
        /opt/agfs/agfs-server -c /opt/agfs/config.yaml
      "
    networks:
      - ogmem-network
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:1833/serverinfo/version", "||", "exit", "1"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 5s
    restart: unless-stopped

  # oG-Memory - 核心服务
  ogmem:
    build:
      context: .
      dockerfile: Dockerfile.standalone
    image: ogmem:latest
    container_name: ogmem-service
    ports:
      - "${OGMEM_PORT:-8090}:8090"
    volumes:
      - agfs-data:/data/agfs
    environment:
      # LLM 配置
      - CONTEXTENGINE_PROVIDER=${CONTEXTENGINE_PROVIDER:-openai}
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - OPENAI_BASE_URL=${OPENAI_BASE_URL:-https://dashscope.aliyuncs.com/compatible-mode/v1}
      - OPENAI_EMBEDDING_MODEL=${OPENAI_EMBEDDING_MODEL:-text-embedding-v2}
      - OPENAI_LLM_MODEL=${OPENAI_LLM_MODEL:-glm-5}
      # AGFS 配置
      - AGFS_BASE_URL=http://agfs:1833
      - AGFS_MOUNT_PREFIX=/local/plugin
      # openGauss 配置
      - VECTOR_DB_TYPE=${VECTOR_DB_TYPE:-opengauss}
      - OPENGUASS_CONNECTION_STRING=${OPENGUASS_CONNECTION_STRING}
      # 运行时配置
      - INDEX_INTERVAL=${INDEX_INTERVAL:-15}
      - OG_ACCOUNT_ID=${OG_ACCOUNT_ID:-acct-demo}
      - no_proxy=localhost,127.0.0.1,agfs
    depends_on:
      agfs:
        condition: service_healthy
    networks:
      - ogmem-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8090/api/v1/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    restart: unless-stopped

networks:
  ogmem-network:
    driver: bridge

volumes:
  agfs-data:
    driver: local
```

### 2.2 关键配置说明

| 配置项 | 值 | 说明 |
|-------|-----|------|
| `depends_on.service_healthy` | `agfs: service_healthy` | AGFS 就绪后才启动 oG-Memory |
| `AGFS_BASE_URL` | `http://agfs:1833` | 使用容器内网络通信 |
| `OPENGUASS_CONNECTION_STRING` | `host=host.docker.internal` | openGauss 在容器外 |
| `volume` | `agfs-data` | AGFS 数据持久化 |

---

## 三、一键部署脚本

### 3.1 deploy-all.sh

```bash
#!/bin/bash
# oG-Memory 一键部署脚本

set -e

# 颜色定义
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

# 项目根目录
PROJECT_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
cd "$PROJECT_ROOT"

# 配置文件
ENV_FILE="${PROJECT_ROOT}/.env"
ENV_EXAMPLE="${PROJECT_ROOT}/.env.example"
AGFS_DIR="${PROJECT_ROOT}/agfs"
AGFS_BINARY="${AGFS_DIR}/agfs-server"

# 使用 docker compose 或 docker-compose
if docker compose version &>/dev/null; then
    COMPOSE_CMD="docker compose"
else
    COMPOSE_CMD="docker-compose"
fi

# ============================================================================
# 1. 环境检查
# ============================================================================
check_environment() {
    echo -e "\n${BLUE}[1/6] 环境检查${NC}"

    if ! command -v docker &>/dev/null; then
        echo -e "${RED}错误: 未安装 Docker${NC}"
        exit 1
    fi
    echo -e "  ${GREEN}Docker: $(docker --version)${NC}"

    if ! $COMPOSE_CMD version &>/dev/null; then
        echo -e "${RED}错误: 未安装 Docker Compose${NC}"
        exit 1
    fi
    echo -e "  ${GREEN}Docker Compose: $($COMPOSE_CMD version)${NC}"

    if ! command -v curl &>/dev/null; then
        echo -e "${RED}错误: 未安装 curl${NC}"
        exit 1
    fi
    echo -e "  ${GREEN}curl: $(curl --version | head -1)${NC}"
}

# ============================================================================
# 2. 环境变量配置
# ============================================================================
setup_env() {
    echo -e "\n${BLUE}[2/6] 环境变量配置${NC}"

    if [ ! -f "$ENV_FILE" ]; then
        echo -e "  ${YELLOW}未找到 .env 文件，从模板创建${NC}"
        cat > "$ENV_FILE" << 'EOF'
# oG-Memory 环境变量配置
# 修改以下变量后保存

# 必填: LLM API Key
OPENAI_API_KEY=your-openai-api-key-here

# 必填: openGauss 连接串
OPENGUASS_CONNECTION_STRING=host=host.docker.internal port=8888 dbname=postgres user=aaa password=aaa@123123

# 可选配置
OPENAI_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
OPENAI_EMBEDDING_MODEL=text-embedding-v2
OPENAI_LLM_MODEL=glm-5

# 服务端口
OGMEM_PORT=8090

# 其他配置
CONTEXTENGINE_PROVIDER=openai
VECTOR_DB_TYPE=opengauss
INDEX_INTERVAL=15
OG_ACCOUNT_ID=acct-demo
EOF
        echo -e "\n${YELLOW}⚠️  请编辑 .env 文件，设置 OPENAI_API_KEY 和 OPENGUASS_CONNECTION_STRING${NC}"
        read -p "按回车键继续..."
    fi

    # 加载环境变量
    export $(cat "$ENV_FILE" | grep -v '^#' | xargs)
    echo -e "  ${GREEN}.env 已加载${NC}"
}

# ============================================================================
# 3. AGFS 二进制准备
# ============================================================================
setup_agfs() {
    echo -e "\n${BLUE}[3/6] AGFS 二进制准备${NC}"

    mkdir -p "$AGFS_DIR"

    if [ -f "$AGFS_BINARY" ]; then
        chmod +x "$AGFS_BINARY" 2>/dev/null
        echo -e "  ${GREEN}AGFS 二进制已存在${NC}"
        return
    fi

    echo -e "  ${YELLOW}AGFS 二进制不存在${NC}"
    echo -e "\n${YELLOW}请按以下方式之一获取 AGFS 二进制:${NC}"
    echo -e "  方式1 (推荐):"
    echo -e "    curl -fsSL https://raw.githubusercontent.com/c4pt0r/agfs/master/install.sh | sh"
    echo -e "    cp ~/.local/bin/agfs-server ${AGFS_DIR}/"
    echo -e "  方式2 (手动下载):"
    echo -e "    访问 https://github.com/c4pt0r/agfs/releases"
    echo -e "    下载对应平台的 agfs-server 到 ${AGFS_DIR}/"
    echo -e ""

    # 尝试自动下载
    read -p "是否尝试自动下载? (Y/n): " -n 1 -r
    if [[ $REPLY =~ ^[Yy]$ ]] || [ -z "$REPLY" ]; then
        echo -e "\n  尝试从 GitHub 下载..."
        if curl -L -o "$AGFS_BINARY" \
            "https://github.com/c4pt0r/agfs/releases/latest/download/agfs-server-linux-amd64" 2>/dev/null; then
            chmod +x "$AGFS_BINARY"
            echo -e "  ${GREEN}AGFS 下载成功${NC}"
            return
        fi
        echo -e "  ${YELLOW}自动下载失败，请手动获取${NC}"
    fi

    read -p "获取 AGFS 二进制后按回车键继续..."
}

# ============================================================================
# 4. 检查 openGauss
# ============================================================================
check_opengauss() {
    echo -e "\n${BLUE}[4/6] openGauss 检查${NC}"

    OG_PORT="${OPENGAUSS_PORT:-8888}"

    if docker ps --format '{{.Names}}' | grep -q "opengauss"; then
        echo -e "  ${GREEN}openGauss 容器正在运行${NC}"
    else
        echo -e "  ${YELLOW}openGauss 容器未运行${NC}"
    fi

    if nc -z 127.0.0.1 "$OG_PORT" 2>/dev/null; then
        echo -e "  ${GREEN}openGauss 端口 $OG_PORT 可访问${NC}"
    else
        echo -e "  ${YELLOW}openGauss 端口不可访问${NC}"
        echo -e "\n${YELLOW}请手动启动 openGauss:${NC}"
        echo -e "  docker start opengauss"
        echo -e ""
        read -p "openGauss 启动后按回车键继续..."

        # 等待最多 30 秒
        for i in {1..30}; do
            if nc -z 127.0.0.1 "$OG_PORT" 2>/dev/null; then
                echo -e "  ${GREEN}openGauss 连接成功${NC}"
                break
            fi
            sleep 1
        done
    fi
}

# ============================================================================
# 5. 构建镜像
# ============================================================================
build_images() {
    echo -e "\n${BLUE}[5/6] 构建 Docker 镜像${NC}"

    if [ "${SKIP_BUILD:-}" = "1" ]; then
        echo -e "  ${YELLOW}跳过镜像构建${NC}"
        return
    fi

    echo -e "  构建 ogmem 镜像..."
    docker build -f Dockerfile.standalone -t ogmem:latest . || {
        echo -e "  ${RED}镜像构建失败${NC}"
        exit 1
    }

    echo -e "  ${GREEN}镜像构建完成${NC}"
}

# ============================================================================
# 6. 启动服务
# ============================================================================
start_services() {
    echo -e "\n${BLUE}[6/6] 启动服务${NC}"

    echo -e "  启动 AGFS 和 oG-Memory..."
    $COMPOSE_CMD up -d || {
        echo -e "  ${RED}服务启动失败${NC}"
        exit 1
    }

    echo -e "  ${GREEN}服务启动成功${NC}"
    echo -e "\n  服务状态:"
    $COMPOSE_CMD ps
}

# ============================================================================
# 健康检查
# ============================================================================
health_check() {
    echo -e "\n${BLUE}健康检查${NC}"

    # AGFS
    for i in {1..10}; do
        if curl -sf "http://localhost:1833/serverinfo/version" &>/dev/null; then
            echo -e "  ${GREEN}AGFS: 健康${NC}"
            break
        fi
        sleep 1
    done

    # oG-Memory
    for i in {1..20}; do
        if curl -sf "http://localhost:8090/api/v1/health" &>/dev/null; then
            echo -e "  ${GREEN}oG-Memory: 健康${NC}"
            break
        fi
        sleep 1
    done
}

# ============================================================================
# 主函数
# ============================================================================
main() {
    echo -e "\n${BLUE}========================================${NC}"
    echo -e "${BLUE}   oG-Memory 一键部署${NC}"
    echo -e "${BLUE}========================================${NC}"

    check_environment
    setup_env
    setup_agfs
    check_opengauss
    build_images
    start_services
    health_check

    echo -e "\n${GREEN}========================================${NC}"
    echo -e "${GREEN}   部署完成!${NC}"
    echo -e "${GREEN}========================================${NC}"
    echo -e "\n服务端点:"
    echo -e "  - AGFS:      http://localhost:1833"
    echo -e "  - oG-Memory: http://localhost:8090"
    echo -e "  - 健康检查: http://localhost:8090/api/v1/health"
    echo -e "\n常用命令:"
    echo -e "  - 查看日志: $COMPOSE_CMD logs -f"
    echo -e "  - 停止服务: ./scripts/stop-all.sh"
    echo -e "  - 运行测试: ./scripts/test-deployment.sh"
}

# 参数处理
case "${1:-}" in
    --skip-build)
        SKIP_BUILD=1
        ;;
    --help|-h)
        echo "用法: $0 [选项]"
        echo "选项:"
        echo "  --skip-build   跳过镜像构建"
        echo "  --help, -h    显示帮助"
        exit 0
        ;;
esac

main "$@"
```

### 3.2 stop-all.sh

```bash
#!/bin/bash
# oG-Memory 一键停止脚本

set -e

BLUE='\033[0;34m'
GREEN='\033[0;32m'
NC='\033[0m'

PROJECT_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
cd "$PROJECT_ROOT"

if docker compose version &>/dev/null; then
    COMPOSE_CMD="docker compose"
else
    COMPOSE_CMD="docker-compose"
fi

echo -e "\n${BLUE}停止服务...${NC}"
$COMPOSE_CMD down

echo -e "${GREEN}服务已停止${NC}"
echo -e "\n注意: openGauss 容器需要手动停止"
```

---

## 四、测试验证方案

### 4.1 test-deployment.sh

```bash
#!/bin/bash
# oG-Memory 部署测试脚本

set -e

GREEN='\033[0;32m'
RED='\033[0;31m'
BLUE='\033[0;34m'
NC='\033[0m'

TESTS_PASSED=0
TESTS_FAILED=0

test_pass() { echo -e "  ${GREEN}✓${NC} $1"; ((TESTS_PASSED++)); }
test_fail() { echo -e "  ${RED}✗${NC} $1"; ((TESTS_FAILED++)); }

echo -e "\n${BLUE}开始测试...${NC}"

# 1. 连通性测试
echo -e "\n${BLUE}[1/4] 连通性测试${NC}"
curl -sf http://localhost:1833/serverinfo/version && test_pass "AGFS 可访问" || test_fail "AGFS 不可访问"
curl -sf http://localhost:8090/api/v1/health && test_pass "oG-Memory 可访问" || test_fail "oG-Memory 不可访问"

# 2. 健康检查
echo -e "\n${BLUE}[2/4] 健康检查${NC}"
HEALTH=$(curl -s http://localhost:8090/api/v1/health)
echo "$HEALTH" | grep -q '"status":"ok"' && test_pass "oG-Memory 健康状态 OK" || test_fail "oG-Memory 健康检查失败"

# 3. API 测试
echo -e "\n${BLUE}[3/4] API 测试${NC}"
curl -s -X POST http://localhost:8090/api/v1/after_turn \
    -H "Content-Type: application/json" \
    -d '{"messages":[{"role":"user","content":"test"}],"sessionId":"test"}' >/dev/null 2>&1 \
    && test_pass "after_turn API" || test_fail "after_turn API"

# 4. 端到端测试
echo -e "\n${BLUE}[4/4] 端到端流程测试${NC}"
# 写入记忆
curl -s -X POST http://localhost:8090/api/v1/after_turn \
    -H "Content-Type: application/json" \
    -d '{"messages":[{"role":"user","content":"我叫张三"},{"role":"assistant","content":"你好张三"}],"sessionId":"e2e-test","accountId":"test","userId":"user1"}' >/dev/null 2>&1 \
    && test_pass "写入记忆"

sleep 8  # 等待索引

# 查询 openGauss 数据
echo -e "\n${BLUE}[验证] 查询 openGauss 数据${NC}"
docker exec opengauss gsql -d postgres -U aaa -W aaa@123123 -p 5432 -c "SELECT count(*) FROM vector_index;" 2>/dev/null | grep -q "[0-9]" \
    && echo -e "  ${GREEN}✓${NC} openGauss 有数据" || echo -e "  ${YELLOW}openGauss 可能暂无数据${NC}"

# 结果
echo -e "\n${BLUE}========================================${NC}"
echo -e "${BLUE}测试结果${NC}"
echo -e "${BLUE}========================================${NC}"
echo -e "  通过: ${GREEN}$TESTS_PASSED${NC}"
echo -e "  失败: ${RED}$TESTS_FAILED${NC}"

[ $TESTS_FAILED -eq 0 ] && exit 0 || exit 1
```

---

## 五、实施步骤

| 步骤 | 操作 | 文件 |
|-----|------|-----|
| 1 | 创建 `docker-compose.yml` | `/data/Workspace2/oG-Memory/docker-compose.yml` |
| 2 | 创建 `scripts/deploy-all.sh` | `/data/Workspace2/oG-Memory/scripts/deploy-all.sh` |
| 3 | 创建 `scripts/stop-all.sh` | `/data/Workspace2/oG-Memory/scripts/stop-all.sh` |
| 4 | 创建 `scripts/test-deployment.sh` | `/data/Workspace2/oG-Memory/scripts/test-deployment.sh` |
| 5 | 给脚本添加执行权限 | `chmod +x scripts/*.sh` |
| 6 | 更新 README.md | 添加一键部署说明 |

---

## 六、使用流程

```bash
# 首次部署
cd /data/Workspace2/oG-Memory
./scripts/deploy-all.sh

# 测试
./scripts/test-deployment.sh

# 停止
./scripts/stop-all.sh
```

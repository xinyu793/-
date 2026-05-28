# 外贸报价单网站 · 技术架构方案

**日期**：2026-05-28
**版本**：v1.0
**关联文档**：`prd-trade-quotation-website-2026-05-28.md`

---

## 📌 TL;DR

- **架构风格**：前后端分离 + BFF层 + 微服务化模块（单体优先，逐步拆分）
- **前端**：Next.js 15（App Router）+ TypeScript + Tailwind CSS + shadcn/ui
- **后端**：NestJS + TypeScript + PostgreSQL + Redis
- **核心决策**：SKU采用EAV+JSON混合模型、报价单采用快照存储、商家独立站通过子域名+网关路由实现、PDF使用Puppeteer服务端渲染

---

## 1. 推荐技术栈

### 1.1 前端

| 层级 | 技术 | 理由 |
|------|------|------|
| 框架 | **Next.js 15（App Router）** | SSR/SSG混合渲染、i18n内置支持、Image优化、Edge Runtime支持全球加速 |
| 语言 | **TypeScript 5.x** | 全栈类型安全，与后端共享类型定义 |
| UI库 | **shadcn/ui + Radix UI** | 可定制、无样式锁定、Accessibility内置、适合外贸专业风格 |
| 样式 | **Tailwind CSS 4** | 原子化CSS、响应式设计友好、多语言RTL支持 |
| 状态管理 | **Zustand + TanStack Query** | 轻量状态管理 + 服务端缓存自动同步 |
| 图表 | **Recharts** | 报价单成本拆解可视化（饼图/堆叠柱状图） |
| 表单 | **React Hook Form + Zod** | 规格配置器复杂表单的高性能处理 + 运行时类型校验 |

### 1.2 后端

| 层级 | 技术 | 理由 |
|------|------|------|
| 框架 | **NestJS** | 模块化架构、依赖注入、TypeORM/Prisma集成、WebSocket支持、企业级可维护性 |
| 语言 | **TypeScript** | 与前端共享类型（Monorepo），减少接口契约维护成本 |
| ORM | **Prisma** | 类型安全的数据库访问、迁移管理、多数据库支持 |
| API风格 | **REST + GraphQL（查询场景）** | 写操作用REST（简洁），商品浏览/筛选等复杂查询用GraphQL（灵活） |
| 认证 | **JWT + Refresh Token** | 无状态认证，适合分布式部署 |
| 文件处理 | **Puppeteer（PDF生成）+ Sharp（图片处理）** | 服务端渲染专业报价单PDF；商品图片裁剪/压缩 |

### 1.3 数据层

| 组件 | 技术 | 理由 |
|------|------|------|
| 主数据库 | **PostgreSQL 16** | JSONB支持SKU灵活属性、全文搜索、CTE递归查询规格树、行级安全（RLS）支持多租户 |
| 缓存 | **Redis 7** | 汇率缓存、会话管理、报价单临时数据、限流计数器 |
| 搜索引擎 | **Meilisearch** | 商品全文搜索、多语言分词、实时索引更新、轻量易部署（Elasticsearch替代） |
| 对象存储 | **AWS S3 / 阿里云OSS** | 商品图片、检测证书PDF、报价单PDF存储 |
| 消息队列 | **BullMQ（Redis-backed）** | PDF生成异步任务、邮件通知、数据同步 |

### 1.4 DevOps

| 组件 | 技术 | 理由 |
|------|------|------|
| 容器化 | **Docker + Docker Compose** | 开发环境一致、部署标准化 |
| CI/CD | **GitHub Actions** | 自动化测试、构建、部署 |
| 监控 | **Grafana + Prometheus + Loki** | 全链路监控、日志聚合、告警 |
| APM | **Sentry** | 前后端错误追踪、性能监控 |

---

## 2. 核心架构设计

### 2.1 整体分层架构

```
┌─────────────────────────────────────────────────────────┐
│                     客户端接入层                          │
│  C端(Next.js SSR)  │  B端(Next.js SSR)  │  移动端(未来) │
└──────────┬─────────┴────────┬───────────┴───────┬───────┘
           │                  │                   │
┌──────────▼──────────────────▼───────────────────▼───────┐
│                    API 网关 / BFF 层                      │
│  路由分发 │ 限流 │ 认证 │ 商家域名识别 │ 响应聚合         │
└──────────┬──────────────────────────────────────────────┘
           │
┌──────────▼──────────────────────────────────────────────┐
│                    业务服务层                             │
│                                                          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│  │ 商品服务  │ │ 报价服务  │ │ 用户服务  │ │ 搜索服务  │  │
│  │ Product  │ │ Quote    │ │ Auth     │ │ Search   │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐               │
│  │ 计算引擎  │ │ PDF服务   │ │ 通知服务  │               │
│  │ Calculator│ │ PDF Gen  │ │ Notify   │               │
│  └──────────┘ └──────────┘ └──────────┘               │
└──────────┬──────────────────────────────────────────────┘
           │
┌──────────▼──────────────────────────────────────────────┐
│                    数据层                                 │
│  PostgreSQL │ Redis │ Meilisearch │ S3/OSS │ BullMQ     │
└─────────────────────────────────────────────────────────┘
           │
┌──────────▼──────────────────────────────────────────────┐
│                 第三方服务集成层                           │
│  汇率API │ 税率API(P1) │ 运费API(P1) │ 邮件 │ CDN      │
└─────────────────────────────────────────────────────────┘
```

### 2.2 商家独立站方案

**方案：子域名路由 + 网关识别**

```
商家注册时分配子域名：{shop-slug}.tradequote.com
自定义域名（Pro版）：quote.merchant-brand.com → CNAME指向 tradequote.com
```

**路由识别流程**：
1. 请求到达API网关 → 提取Host头中的子域名
2. 查询Redis缓存中的 子域名→商家ID 映射
3. 将商家ID注入请求上下文（x-merchant-id）
4. 后续服务基于商家ID进行数据隔离

**数据隔离策略**：共享数据库 + 商家ID字段过滤（Row-Level Security）
- Phase 1：所有商家共享数据库实例，通过merchant_id字段过滤
- Phase 3+：大客户可迁移至独立Schema（PostgreSQL Schema隔离）

### 2.3 SKU数据模型设计（核心难点）

**方案：EAV + JSONB 混合模型**

传统关系模型无法应对"5色×6码×2材质=60SKU"的组合爆炸。采用混合策略：

```sql
-- 规格维度定义（商家级）
CREATE TABLE spec_dimensions (
  id UUID PRIMARY KEY,
  merchant_id UUID NOT NULL REFERENCES merchants(id),
  name VARCHAR(50) NOT NULL,        -- e.g., "尺寸", "材质", "颜色"
  display_order INT,
  is_required BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 规格值定义
CREATE TABLE spec_values (
  id UUID PRIMARY KEY,
  dimension_id UUID NOT NULL REFERENCES spec_dimensions(id),
  value VARCHAR(100) NOT NULL,      -- e.g., "S", "M", "L"
  display_name JSONB,               -- 多语言显示名 {"en": "Red", "zh": "红色"}
  color_hex VARCHAR(7),             -- 颜色专用：色值
  pantone_code VARCHAR(20),         -- 颜色专用：Pantone编号
  extra_attrs JSONB DEFAULT '{}',   -- 扩展属性：如MOQ调整、加价规则
  display_order INT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 商品
CREATE TABLE products (
  id UUID PRIMARY KEY,
  merchant_id UUID NOT NULL REFERENCES merchants(id),
  name JSONB NOT NULL,              -- 多语言 {"en": "Steel Pipe", "zh": "钢管"}
  description JSONB,
  category_id UUID,
  status VARCHAR(20) DEFAULT 'draft', -- draft/active/archived
  base_price DECIMAL(12,2),         -- 基准价格
  currency VARCHAR(3) DEFAULT 'USD',
  moq INT DEFAULT 1,
  unit VARCHAR(20),                 -- 件/吨/米
  hs_code VARCHAR(20),              -- HS编码
  packaging_info JSONB,             -- 包装信息
  images JSONB DEFAULT '[]',        -- 图片列表
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 商品关联的规格维度
CREATE TABLE product_spec_dims (
  product_id UUID REFERENCES products(id),
  dimension_id UUID REFERENCES spec_dimensions(id),
  display_order INT,
  PRIMARY KEY (product_id, dimension_id)
);

-- SKU（具体可购买单元）
CREATE TABLE skus (
  id UUID PRIMARY KEY,
  product_id UUID NOT NULL REFERENCES products(id),
  sku_code VARCHAR(100) NOT NULL,   -- 商家自定义SKU编码
  spec_combo JSONB NOT NULL,        -- {"尺寸": "M", "颜色": "Red", "材质": "Cotton"}
  price_override DECIMAL(12,2),     -- 覆盖基准价（NULL则用product.base_price）
  stock INT,                        -- 库存（可选）
  weight DECIMAL(10,3),             -- 单件重量
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 阶梯定价
CREATE TABLE tier_pricing (
  id UUID PRIMARY KEY,
  sku_id UUID NOT NULL REFERENCES skus(id),
  min_qty INT NOT NULL,
  max_qty INT,
  price DECIMAL(12,2) NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 检测标准
CREATE TABLE certifications (
  id UUID PRIMARY KEY,
  product_id UUID NOT NULL REFERENCES products(id),
  standard VARCHAR(20) NOT NULL,    -- CE/FDA/ROHS/REACH/UL
  certificate_url TEXT,             -- 证书文件URL
  valid_until DATE,
  issuing_body VARCHAR(100),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 规格模板（复用）
CREATE TABLE spec_templates (
  id UUID PRIMARY KEY,
  merchant_id UUID NOT NULL REFERENCES merchants(id),
  name VARCHAR(100) NOT NULL,       -- e.g., "服装标准模板"
  dimensions JSONB NOT NULL,        -- 完整维度+值定义
  category_id UUID,                 -- 关联品类
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

**spec_combo JSONB 索引策略**：
```sql
-- GIN索引支持JSONB高效查询
CREATE INDEX idx_skus_spec_combo ON skus USING GIN (spec_combo);
-- 商品+SKU联合查询
CREATE INDEX idx_skus_product ON skus (product_id, is_active);
```

**SKU生成流程**：
1. 商家定义规格维度和值 → 保存到spec_dimensions/spec_values
2. 商家关联维度到商品 → product_spec_dims
3. 前端触发"生成SKU组合" → 后端计算笛卡尔积 → 批量INSERT到skus表
4. 商家批量定价 → UPDATE skus.price_override + INSERT tier_pricing

### 2.4 报价计算引擎设计

```
输入参数：
├── sku_id + quantity          → 基础价格（含阶梯价）
├── trade_term                 → 贸易术语（FOB/CIF/DDP）
├── origin_port                → 起运港
├── destination_country        → 目的国家
├── destination_port           → 目的港
├── shipping_method            → 运输方式（海运/空运/铁路）
├── tax_rate (manual)          → 手动输入税率
├── payment_method             → 付款方式
├── exchange_rate (realtime)   → 实时汇率
└── merchant_cost_config       → 商家成本配置（利润率/退税/包装费）

计算流程：
BasePrice = sku.price || tier_pricing.price
  → + PackagingCost (商家配置)
  → + ProfitMargin (商家配置)
  → = FOB_Price

  → + Freight (运费估算)
  → + Insurance (CIF时)
  → = CIF_Price

  → + ImportDuty (FOB/CIF × tax_rate)
  → + VAT ((CIF + Duty) × vat_rate)
  → + OtherFees (清关/内陆运输等)
  → = LandedCost (到岸成本)

  → + PaymentFee (T/T: 0.1%, L/C: 0.3%等)
  → = TotalCost
```

**关键设计**：
- 计算引擎为纯函数（无副作用），便于测试和缓存
- 每次计算生成完整成本拆解快照，与报价单一起存储
- 汇率取Redis缓存值（每30分钟从API更新），报价单标注汇率基准时间

---

## 3. 第三方服务集成方案

### 3.1 汇率API

| 方案 | 免费额度 | 延迟 | 支持币种 | 推荐度 |
|------|---------|------|---------|--------|
| **ExchangeRate-API** | 1,500次/月 | <100ms | 150+ | ⭐⭐⭐⭐⭐ 首选 |
| Open Exchange Rates | 1,000次/月 | <200ms | 170+ | ⭐⭐⭐⭐ 备选 |
| Fixer.io | 100次/月 | <200ms | 170+ | ⭐⭐⭐ 免费额度少 |
| European Central Bank | 无限制 | 每日更新 | 30+ | ⭐⭐⭐ 免费但币种少 |

**集成策略**：
- Redis缓存汇率数据，每30分钟从API更新
- 报价单标注汇率基准时间和来源
- 同时保存商家设置的"锚定汇率"选项（P2功能）

### 3.2 税率/关税数据API（P1）

| 方案 | 数据覆盖 | 价格 | 推荐度 |
|------|---------|------|--------|
| **TaxDataSolutions** | 180+国家，HS编码级 | ~$500/月起 | ⭐⭐⭐⭐⭐ 最全面 |
| Wise Business Rates | 50+国家 | 包含在Wise API中 | ⭐⭐⭐⭐ 覆盖主流 |
| SimplyDuty | 50+国家 | ~$99/月起 | ⭐⭐⭐ 性价比高 |
| 自建HS编码税率表 | 首期10-20国 | 人力成本 | ⭐⭐⭐ MVP可行 |

**MVP策略**：首期商家手动输入税率 + 内置10个主要贸易国的参考税率表（静态数据），P1接入TaxDataSolutions API。

### 3.3 运费估算API（P1）

| 方案 | 覆盖范围 | 特点 | 推荐度 |
|------|---------|------|--------|
| **FreightOS API** | 海运/空运 | 实时报价、多承运商 | ⭐⭐⭐⭐⭐ |
| SeaRates | 海运为主 | 免费、数据全面 | ⭐⭐⭐⭐ |
| 商家预设运费模板 | 自定义 | MVP首选 | ⭐⭐⭐⭐⭐ MVP |

**MVP策略**：商家预设运费模板（按目的地区域+重量/体积阶梯），P1接入FreightOS。

### 3.4 PDF生成方案

| 方案 | 优点 | 缺点 | 推荐度 |
|------|------|------|--------|
| **Puppeteer + HTML模板** | 高保真、支持复杂布局、复用前端组件 | 启动慢（~2s）、内存占用高 | ⭐⭐⭐⭐⭐ 首选 |
| React-PDF / @react-pdf/renderer | 纯JS、无浏览器依赖 | 布局能力有限、复杂表格难 | ⭐⭐⭐ |
| wkhtmltopdf | 成熟稳定 | 样式兼容性差 | ⭐⭐ |
| Prince XML | 专业级排版 | 商业付费 | ⭐⭐⭐ |

**推荐方案**：Puppeteer + HTML模板
- 使用Nunjucks/EJS模板引擎渲染报价单HTML
- Puppeteer headless Chrome生成PDF
- 通过BullMQ异步队列处理，避免阻塞API请求
- 模板支持多语言（i18n字符串注入）

### 3.5 短链接/分享方案

- 自建短链接服务（base62编码，存储在PostgreSQL）
- 报价单链接格式：`https://{shop}.tradequote.com/q/{quote-id}`
- 链接含有效期参数，过期后展示"报价已过期，请联系商家获取最新报价"
- Redis存储访问计数，用于报价单追踪分析

---

## 4. 部署与基础设施

### 4.1 推荐云服务商

**首选：阿里云（国际版）**

| 理由 | 说明 |
|------|------|
| 东南亚节点 | 新加坡/雅加达/吉隆坡节点，覆盖Tier1市场 |
| 中东节点 | 迪拜节点，覆盖Tier1市场 |
| 中国出海经验 | 12万+跨境电商主体在阿里云 |
| 成本 | 比AWS低20-30%（同等配置） |
| 合规 | 数据出境合规方案成熟 |

**备选：AWS（全球化需求增强时切换）**

### 4.2 部署架构

```
                    ┌─────────────────┐
                    │    CloudFlare    │  ← 全球CDN + DDoS防护 + WAF
                    │   (DNS + CDN)    │
                    └────────┬────────┘
                             │
              ┌──────────────▼──────────────┐
              │      阿里云 SLB / ALB        │  ← 负载均衡
              └──────────────┬──────────────┘
                             │
              ┌──────────────▼──────────────┐
              │     ECS / Container Service  │  ← 应用服务
              │  ┌───────┐  ┌───────┐       │
              │  │Next.js│  │NestJS │       │  ← 最小2实例
              │  │ SSR   │  │ API   │       │
              │  └───────┘  └───────┘       │
              │  ┌───────┐  ┌───────┐       │
              │  │Puppeteer│ │Worker │       │  ← PDF生成 + 异步任务
              │  │Service │  │ (BullMQ)│     │
              │  └───────┘  └───────┘       │
              └──────────────┬──────────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
    ┌────▼────┐        ┌────▼────┐        ┌────▼────┐
    │RDS PG   │        │Redis    │        │OSS      │
    │(主从)    │        │(集群)    │        │(图片/PDF)│
    └─────────┘        └─────────┘        └─────────┘
```

### 4.3 CDN策略

| 区域 | CDN节点 | 延迟目标 |
|------|---------|---------|
| 东南亚 | CloudFlare Singapore/Jakarta | <100ms |
| 中东 | CloudFlare Dubai | <150ms |
| 中国 | 阿里云CDN国内节点 | <50ms |
| 欧美 | CloudFlare 全球节点 | <200ms |

**CDN缓存策略**：
- 静态资源（JS/CSS/图片）：长期缓存 + 文件hash
- 商品列表页：SSG + ISR（60秒重新验证）
- 报价单页面：不缓存（含动态计算和有效期）

### 4.4 域名与SSL

- 主域名：`tradequote.com`（示例）
- 商家子域名：`{slug}.tradequote.com`（通配符SSL证书）
- 自定义域名（Pro版）：商家CNAME指向 `shops.tradequote.com`，Let's Encrypt自动签发

---

## 5. 关键技术决策

### 5.1 SSR vs CSR

**决策：SSR为主 + CSR交互增强**

| 页面类型 | 渲染策略 | 理由 |
|----------|---------|------|
| 商品列表/详情 | **SSR + ISR** | SEO关键页面、首屏速度、多语言元数据 |
| 报价计算器 | **CSR** | 重交互、实时计算、无需SEO |
| 报价单预览 | **CSR + SSR PDF** | 前端实时预览 + 服务端渲染PDF |
| 商家后台 | **CSR** | 认证后页面、无需SEO、重交互 |
| 报价单分享页 | **SSR** | 需要社交分享元数据（OG标签） |

### 5.2 Monorepo vs 多仓库

**决策：Monorepo（Turborepo）**

```
tradequote/
├── apps/
│   ├── web-c/          ← C端（客户端）
│   ├── web-b/          ← B端（商家后台）
│   └── api/            ← NestJS API
├── packages/
│   ├── shared-types/   ← 共享TypeScript类型
│   ├── shared-utils/   ← 共享工具函数
│   ├── ui-components/  ← 共享UI组件
│   ├── quote-engine/   ← 报价计算引擎（纯逻辑）
│   └── pdf-templates/  ← PDF模板
├── turbo.json
└── package.json
```

**理由**：
- 前后端共享类型定义（商品/SKU/报价单DTO）
- quote-engine可同时被API和PDF服务调用
- 统一版本管理和依赖更新

### 5.3 报价单：实时计算 vs 快照存储

**决策：实时计算 + 快照存储并存**

| 场景 | 策略 | 理由 |
|------|------|------|
| 客户配置报价时 | 实时计算 | 参数频繁变化，需即时反馈 |
| 报价单生成时 | 生成快照并存储 | 报价单具有法律效力，必须保存生成时刻的完整数据 |
| 报价单查看时 | 读取快照 | 确保数据一致性，不受后续价格/汇率变化影响 |
| 报价单过期重新报价 | 基于新参数重新计算 | 汇率/运费已变化，需基于最新数据 |

```sql
-- 报价单快照
CREATE TABLE quotes (
  id UUID PRIMARY KEY,
  merchant_id UUID NOT NULL,
  customer_id UUID,                 -- 可为空（游客注册后关联）
  quote_number VARCHAR(50) NOT NULL, -- QT-2026-00001
  status VARCHAR(20) DEFAULT 'draft', -- draft/sent/confirmed/expired/cancelled
  items JSONB NOT NULL,              -- 完整报价项快照
  cost_breakdown JSONB NOT NULL,     -- 成本拆解快照
  exchange_rate DECIMAL(12,6),       -- 生成时汇率
  exchange_rate_date TIMESTAMPTZ,    -- 汇率基准日期
  valid_until TIMESTAMPTZ,           -- 有效期
  total_amount DECIMAL(12,2),
  currency VARCHAR(3) DEFAULT 'USD',
  pdf_url TEXT,                      -- PDF文件URL
  share_token VARCHAR(32) UNIQUE,    -- 分享令牌
  view_count INT DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### 5.4 数据库分库分表

**决策：Phase 1-2不分库分表，Phase 3按需Schema隔离**

- Phase 1-2：单PostgreSQL实例，merchant_id字段过滤 + 索引优化
- Phase 3：大客户独立Schema，中小客户共享
- 触发条件：单表超5000万行 或 单商家数据量超总量的30%

---

## 6. 性能目标

### 6.1 用户体验指标

| 指标 | 目标 | 测量方式 |
|------|------|---------|
| 首屏加载时间（LCP） | <2.5s（东南亚/中东） | Lighthouse + Real User Monitoring |
| 首次输入延迟（FID） | <100ms | Lighthouse |
| 累积布局偏移（CLS） | <0.1 | Lighthouse |
| 商品列表页加载 | <1.5s | SSR + ISR |
| 报价计算响应 | <500ms | API P95 |

### 6.2 API性能指标

| 接口 | 目标响应时间 | 策略 |
|------|------------|------|
| 商品列表/搜索 | <200ms | Meilisearch + CDN缓存 |
| 商品详情+SKU | <150ms | PostgreSQL索引优化 + Redis缓存 |
| 报价计算 | <300ms | 纯函数计算 + Redis汇率缓存 |
| PDF生成 | <5s（异步） | BullMQ队列 + Puppeteer池 |
| 用户注册/登录 | <500ms | JWT签发 |

### 6.3 并发与容量

| 指标 | Phase 1 目标 | Phase 2 目标 |
|------|------------|------------|
| 同时在线用户 | 500 | 5,000 |
| API QPS | 100 | 1,000 |
| PDF生成吞吐 | 10份/分钟 | 50份/分钟 |
| 数据库连接池 | 20 | 50 |
| Redis内存 | 512MB | 2GB |

### 6.4 可用性目标

| 指标 | 目标 |
|------|------|
| 系统可用性 | ≥99.5%（月度） |
| 计划外停机 | <4小时/月 |
| 数据备份 | 每日全量 + 实时WAL归档 |
| RPO（恢复点目标） | <1小时 |
| RTO（恢复时间目标） | <4小时 |

---

## 7. 安全设计

| 领域 | 方案 |
|------|------|
| 认证 | JWT + Refresh Token + OAuth2（Google/LinkedIn社交登录） |
| 授权 | RBAC角色权限（商家：管理员/业务员/只读；平台：超管/运营/客服） |
| 数据加密 | 传输层TLS 1.3；存储层AES-256（敏感字段如银行账号） |
| API安全 | Rate Limiting（Redis计数器）、CORS白名单、请求签名 |
| PDF防篡改 | 报价单PDF添加水印+数字签名哈希 |
| 多租户隔离 | merchant_id字段级过滤 + API层强制注入 |
| 内容安全 | XSS防护（DOMPurify）、SQL注入防护（Prisma参数化）、CSRF Token |

---

## 8. 关键依赖与风险

| 依赖 | 风险 | 缓解措施 |
|------|------|---------|
| 汇率API可用性 | API宕机导致汇率数据过期 | Redis缓存30min + 降级为最近可用汇率 + 人工修正入口 |
| Puppeteer内存 | PDF生成内存占用高 | 独立Worker进程 + 实例池 + 请求队列限流 |
| 商家子域名路由 | DNS解析延迟 | CloudFlare通配符解析 + Redis映射缓存 |
| Meilisearch数据同步 | 数据不一致 | 双写（PostgreSQL写入 → BullMQ事件 → Meilisearch索引） |
| 税率数据准确性 | 手动输入错误 | 报价单标注"税率由客户提供"免责 + P1接入权威数据源 |
| 图片/文件存储成本 | S3/OSS费用增长 | 图片压缩（Sharp）+ CDN缓存 + 冷热分层 |

---

> 本文档由产品战略团队技术架构调研产出，关键决策需技术负责人审定。

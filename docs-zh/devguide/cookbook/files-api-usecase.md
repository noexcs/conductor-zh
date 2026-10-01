# Conductor OSS — 文件管理用例

五个真实场景，展示 Conductor 如何在多个工作流阶段中编排文件的创建、处理与交付。

---

## 1. 退货与退款单据处理

客户发起产品退货。Conductor 编排退货照片/文档的接收、资格审核、RMA（Return Merchandise Authorization，退货授权）表单的生成，以及最终退款收据的产出——全部作为单个可追踪的工作流。

### 工作流

```mermaid
flowchart TD
    A["客户提交<br/>退货请求"] --> B["HTTP 任务：<br/>获取订单详情"]
    B --> C["INLINE 任务：<br/>校验退货窗口"]
    C --> D{"SWITCH：<br/>符合条件？"}
    D -- 否 --> E["生成拒绝<br/>函 PDF"]
    E --> E1["向客户<br/>发送拒绝邮件"]
    D -- 是 --> F["HUMAN 任务：<br/>客服审核照片"]
    F --> G{"SWITCH：<br/>状况检查"}
    G -- 损坏 --> H["生成 RMA 表单<br/>+ 预付运费标签"]
    G -- 商品不符 --> H
    G -- 其他 --> I["HUMAN 任务：<br/>升级至主管"]
    I --> H
    H --> J["FORK"]
    J --> K["分支 1：<br/>通过支付网关<br/>处理退款"]
    J --> L["分支 2：<br/>生成退款<br/>收据 PDF"]
    J --> M["分支 3：<br/>更新库存<br/>系统"]
    K --> N["JOIN"]
    L --> N
    M --> N
    N --> O["向客户发送 RMA + 收据<br/>+ 运费标签"]
    O --> P["将全部文档<br/>归档到 S3"]

    style A fill:#4CAF50,color:#fff
    style D fill:#FF9800,color:#fff
    style G fill:#FF9800,color:#fff
    style J fill:#2196F3,color:#fff
    style N fill:#2196F3,color:#fff
    style P fill:#9C27B0,color:#fff
```

### 产出文件

| 阶段 | 文件 | 格式 |
|-------|------|--------|
| RMA 生成 | `rma_RET-9001.pdf` | PDF |
| 运费标签 | `label_RET-9001.png` | 4×6 ZPL/PNG |
| 退款收据 | `receipt_RET-9001.pdf` | PDF |
| 拒绝函 | `denial_RET-9001.pdf` | PDF（如不符合条件） |

### Conductor 原语

SWITCH, HUMAN, FORK/JOIN, HTTP, INLINE, SUB_WORKFLOW

---

## 2. AI 驱动的知识库构建器（RAG 流水线）

组织将文档（PDF、Word 文件、网页）摄取到 AI 就绪的知识库中。Conductor 编排爬取、抽取、分块、嵌入生成和向量库索引——为聊天机器人和搜索提供检索增强生成（RAG）能力。

### 工作流

```mermaid
flowchart TD
    A["触发器：<br/>新文档上传到<br/>S3 存储桶"] --> B["DO_WHILE：<br/>逐个处理文档"]
    B --> C{"SWITCH：<br/>文件类型？"}
    C -- PDF --> D["通过 Apache Tika<br/>抽取文本"]
    C -- DOCX --> E["通过 python-docx<br/>抽取文本"]
    C -- HTML --> F["通过 BeautifulSoup<br/>抓取并清洗"]
    C -- 其他 --> G["通过 Tesseract<br/>进行 OCR"]
    D --> H["INLINE 任务：<br/>文本分块<br/>（512 tokens，重叠 50）"]
    E --> H
    F --> H
    G --> H
    H --> I["FORK_JOIN_DYNAMIC：<br/>生成嵌入<br/>（每块 1 个）"]
    I --> J["LLM_TEXT_COMPLETE：<br/>创建嵌入向量"]
    J --> K["JOIN：<br/>收集全部向量"]
    K --> L["HTTP 任务：<br/>写入向量数据库<br/>（Pinecone / Weaviate）"]
    L --> M["生成元数据<br/>索引 JSON"]
    M --> N{"还有文档？"}
    N -- 是 --> B
    N -- 否 --> O["写入总索引<br/>清单"]
    O --> P["上传清单<br/>+ 日志到 S3"]

    style A fill:#4CAF50,color:#fff
    style C fill:#FF9800,color:#fff
    style I fill:#2196F3,color:#fff
    style K fill:#2196F3,color:#fff
    style N fill:#FF9800,color:#fff
    style P fill:#9C27B0,color:#fff
```

### 产出文件

| 阶段 | 文件 | 格式 |
|-------|------|--------|
| 抽取文本 | `extracted_{doc_id}.txt` | 纯文本 |
| 分块清单 | `chunks_{doc_id}.jsonl` | JSONL |
| 嵌入向量 | `embeddings_{doc_id}.npy` | NumPy 二进制 |
| 元数据索引 | `index_{doc_id}.json` | JSON |
| 总清单 | `kb_manifest_{run_id}.json` | JSON |
| 流水线日志 | `pipeline_log_{run_id}.txt` | 文本 |

### Conductor 原语

DO_WHILE, SWITCH, FORK_JOIN_DYNAMIC, LLM_TEXT_COMPLETE, HTTP, INLINE

---

## 3. 多格式媒体转码与发布

媒体公司上传一个主视频文件。Conductor 扇出转码任务以产出多种分辨率和格式，生成缩略图，通过语音转文字提取字幕，并将所有内容发布到 CDN——尽可能全部并行进行。

### 工作流

```mermaid
flowchart TD
    A["主视频<br/>已上传（4K ProRes）"] --> B["INLINE 任务：<br/>校验并抽取<br/>媒体元数据"]
    B --> C["FORK（3 个分支）"]
    
    C --> D["分支 1：<br/>FORK_JOIN_DYNAMIC<br/>转码变体"]
    D --> D1["1080p H.264 MP4"]
    D --> D2["720p H.264 MP4"]
    D --> D3["480p H.264 MP4"]
    D --> D4["1080p WebM VP9"]
    D --> D5["HLS 自适应<br/>播放列表（.m3u8）"]
    
    C --> E["分支 2：<br/>生成缩略图"]
    E --> E1["抽取关键帧<br/>（每 30 秒）"]
    E1 --> E2["缩放到<br/>320×180 JPG"]
    E2 --> E3["生成海报<br/>图 1920×1080"]
    
    C --> F["分支 3：<br/>语音转文字"]
    F --> F1["LLM_TEXT_COMPLETE：<br/>转录音频"]
    F1 --> F2["生成 SRT<br/>字幕文件"]
    F2 --> F3["生成 VTT<br/>字幕文件"]

    D1 --> G["JOIN"]
    D2 --> G
    D3 --> G
    D4 --> G
    D5 --> G
    E3 --> G
    F3 --> G
    
    G --> H["生成<br/>清单 JSON"]
    H --> I["HTTP 任务：<br/>将全部素材<br/>上传到 CDN"]
    I --> J["HTTP 任务：<br/>用 URL 更新<br/>CMS"]
    J --> K["通过 Slack 通知<br/>编辑团队"]

    style A fill:#4CAF50,color:#fff
    style C fill:#2196F3,color:#fff
    style D fill:#FF5722,color:#fff
    style G fill:#2196F3,color:#fff
    style K fill:#9C27B0,color:#fff
```

### 产出文件

| 阶段 | 文件 | 格式 |
|-------|------|--------|
| 转码视频 | `video_{res}.mp4`、`video_1080p.webm` | MP4、WebM |
| HLS 播放列表 | `stream.m3u8` + 分片 `.ts` 文件 | HLS |
| 缩略图 | `thumb_{timestamp}.jpg` | JPEG |
| 海报图 | `poster.jpg` | JPEG 1920×1080 |
| 字幕 | `subs_en.srt`、`subs_en.vtt` | SRT、VTT |
| 清单 | `publish_manifest.json` | JSON |

### Conductor 原语

FORK/JOIN, FORK_JOIN_DYNAMIC, LLM_TEXT_COMPLETE, HTTP, INLINE

---

## 4. 订单发票、装箱单与运费标签生成

一个电商订单触发 Conductor 获取订单数据，然后并行扇出生成三份文档——面向客户的发票、仓库装箱单（不含价格）和承运商运费标签——最后打包并分发。

### 工作流

```mermaid
flowchart TD
    A["订单创建<br/>（Webhook）"] --> B["HTTP 任务：<br/>获取订单 +<br/>客户档案"]
    B --> C["INLINE 任务：<br/>计算总额<br/>（税、折扣、运费）"]
    C --> D["FORK（3 个分支）"]
    
    D --> E["分支 1：<br/>生成发票 PDF"]
    E --> E1["应用品牌样式<br/>（Logo、颜色、页脚）"]
    E1 --> E2["格式化明细行<br/>+ 税费明细"]
    E2 --> E3["渲染 PDF<br/>invoice_ORD-12345.pdf"]
    
    D --> F["分支 2：<br/>生成装箱单"]
    F --> F1["去除价格信息"]
    F1 --> F2["添加拣货位置<br/>+ 货位编号"]
    F2 --> F3["添加仓库<br/>条码"]
    F3 --> F4["渲染 PDF<br/>packslip_ORD-12345.pdf"]
    
    D --> G["分支 3：<br/>生成运费标签"]
    G --> G1{"SWITCH：<br/>承运商？"}
    G1 -- FedEx --> G2["调用 FedEx API"]
    G1 -- UPS --> G3["调用 UPS API"]
    G1 -- USPS --> G4["调用 USPS API"]
    G2 --> G5["获取运单号<br/>+ 标签图像"]
    G3 --> G5
    G4 --> G5
    G5 --> G6["渲染标签<br/>label_ORD-12345.png"]
    
    E3 --> H["JOIN"]
    F4 --> H
    G6 --> H
    
    H --> I["将 3 个文件<br/>打包为订单包"]
    I --> J["上传到 S3<br/>orders/ORD-12345/"]
    J --> K["FORK（2 个分支）"]
    K --> L["向客户<br/>发送发票邮件"]
    K --> M["将装箱单 + 标签<br/>发送到仓库打印机"]
    L --> N["JOIN"]
    M --> N
    N --> O["更新订单状态：<br/>待发货"]

    style A fill:#4CAF50,color:#fff
    style D fill:#2196F3,color:#fff
    style G1 fill:#FF9800,color:#fff
    style H fill:#2196F3,color:#fff
    style K fill:#2196F3,color:#fff
    style N fill:#2196F3,color:#fff
    style O fill:#9C27B0,color:#fff
```

### 产出文件

| 阶段 | 文件 | 格式 |
|-------|------|--------|
| 发票 | `invoice_ORD-12345.pdf` | PDF |
| 装箱单 | `packslip_ORD-12345.pdf` | PDF |
| 运费标签 | `label_ORD-12345.png` | 4×6 ZPL/PNG |

### Conductor 原语

FORK/JOIN, SWITCH, HTTP, INLINE, SUB_WORKFLOW

---

## 5. 企业视频监控归档与告警流水线

一组安防摄像机将视频流推送到边缘服务器。Conductor 编排整条流水线：摄取视频片段、运行基于 AI 的异常检测、生成带标注的告警片段、按保留策略归档原始录像，并产出每日汇总报告。

### 工作流

```mermaid
flowchart TD
    A["摄像机视频流：<br/>60 秒片段到达<br/>边缘服务器"] --> B["INLINE 任务：<br/>抽取元数据<br/>（摄像机 ID、时间戳、<br/>分辨率）"]
    B --> C["上传原始片段<br/>到冷存储<br/>（S3 Glacier）"]
    C --> D["HTTP 任务：<br/>AI 异常检测<br/>模型推理"]
    D --> E{"SWITCH：<br/>检测到异常？"}
    
    E -- 否 --> F["记录：正常<br/>更新每日计数器"]
    
    E -- 是 --> G["FORK（3 个分支）"]
    G --> H["分支 1：<br/>截取异常时间戳<br/>前后 30 秒片段"]
    H --> H1["叠加边界框<br/>+ 标签"]
    H1 --> H2["渲染告警片段<br/>alert_CAM04_1712345678.mp4"]
    
    G --> I["分支 2：<br/>生成告警<br/>快照"]
    I --> I1["抽取最佳帧"]
    I1 --> I2["添加检测元数据<br/>标注"]
    I2 --> I3["保存快照<br/>alert_CAM04_1712345678.jpg"]
    
    G --> J["分支 3：<br/>创建事件<br/>报告"]
    J --> J1["LLM_TEXT_COMPLETE：<br/>总结事件"]
    J1 --> J2["生成 PDF<br/>incident_1712345678.pdf"]
    
    H2 --> K["JOIN"]
    I3 --> K
    J2 --> K
    
    K --> L["上传告警包<br/>到热存储（S3）"]
    L --> M["HTTP 任务：<br/>向安全团队<br/>推送通知"]
    M --> N["将事件<br/>记录到 SIEM"]

    F --> O["TIMER：<br/>一天结束？"]
    N --> O
    O --> P["DO_WHILE：<br/>汇总所有<br/>摄像机日志"]
    P --> Q["生成每日<br/>汇总报告 PDF"]
    Q --> R["应用保留策略<br/>（90 天<br/>热 → 冷 → 删除）"]
    R --> S["向设施经理<br/>发送每日报告邮件"]

    style A fill:#4CAF50,color:#fff
    style E fill:#FF9800,color:#fff
    style G fill:#2196F3,color:#fff
    style K fill:#2196F3,color:#fff
    style O fill:#FF5722,color:#fff
    style S fill:#9C27B0,color:#fff
```

### 产出文件

| 阶段 | 文件 | 格式 |
|-------|------|--------|
| 原始片段 | `raw_CAM04_1712345678.mp4` | MP4（60 秒） |
| 告警片段 | `alert_CAM04_1712345678.mp4` | MP4（30 秒，带标注） |
| 告警快照 | `alert_CAM04_1712345678.jpg` | JPEG（带标注） |
| 事件报告 | `incident_1712345678.pdf` | PDF |
| 每日汇总 | `daily_report_2026-04-08.pdf` | PDF |

### Conductor 原语

FORK/JOIN, SWITCH, DO_WHILE, TIMER, LLM_TEXT_COMPLETE, HTTP, INLINE

---

*为 Conductor OSS 文件管理用例探索而生成。*

---
layout: page
title: "项目"
---

<div class="projects-page">
  <p style="margin-bottom: 32px; color: var(--color-text-secondary);">
    参与开发的开源项目与个人作品。
  </p>

  <div class="projects-grid">
    <div class="project-card">
      <h3 class="project-title">
        <i class="fas fa-microchip" style="color: var(--color-accent);"></i>
        物联网数据采集网关
      </h3>
      <p class="project-desc">
        高并发传感器数据接收与处理网关，支持海量设备并发上报。集成 Redis 缓存与消息队列实现流量削峰，
        配合 MySQL 批量写入优化确保数据高效稳定入库。
      </p>
      <div class="project-tech">
        <span class="tag">Netty</span>
        <span class="tag">Redis</span>
        <span class="tag">RocketMQ</span>
        <span class="tag">MySQL</span>
      </div>
    </div>

    <div class="project-card">
      <h3 class="project-title">
        <i class="fas fa-globe" style="color: var(--color-accent);"></i>
        外部数据采集 Pipeline
      </h3>
      <p class="project-desc">
        外部平台数据与市场行情的定向采集管道。运用 Headless 浏览器技术突破动态渲染及反爬策略限制，
        构建稳定的领域语料抓取系统。
      </p>
      <div class="project-tech">
        <span class="tag">Headless Browser</span>
        <span class="tag">Python</span>
        <span class="tag">Playwright</span>
      </div>
    </div>

    <div class="project-card">
      <h3 class="project-title">
        <i class="fas fa-robot" style="color: var(--color-accent);"></i>
        AI 智能问答系统
      </h3>
      <p class="project-desc">
        将大语言模型（LLM）能力引入业务平台，配合 RAG 知识库检索技术实现智能化问答交互服务。
        采用 Spring AI + pgvector 技术栈。
      </p>
      <div class="project-tech">
        <span class="tag">LLM</span>
        <span class="tag">RAG</span>
        <span class="tag">Spring AI</span>
        <span class="tag">pgvector</span>
      </div>
    </div>

    <div class="project-card">
      <h3 class="project-title">
        <i class="fas fa-chart-line" style="color: var(--color-accent);"></i>
        高可用数据可视化平台
      </h3>
      <p class="project-desc">
        面向大规模实时数据的高性能 API 查询服务。通过 Redis 分布式缓存体系与 SQL 索引深度优化，
        将核心查询接口 P99 延迟降低 40% 以上。
      </p>
      <div class="project-tech">
        <span class="tag">Redis</span>
        <span class="tag">MySQL</span>
        <span class="tag">Spring Boot</span>
      </div>
    </div>
  </div>
</div>

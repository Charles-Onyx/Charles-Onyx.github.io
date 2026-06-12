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
        <i class="fas fa-leaf" style="color: var(--color-accent);"></i>
        智慧农业大数据管理平台
      </h3>
      <p class="project-desc">
        面向现代大型农场提供设备集控、环境监测及智能决策支持。系统处理海量物联网传感器并发上报数据，
        整合全网农业行情以支撑 AI 辅助分析。核心技术栈：Java、Spring Boot、MySQL、Redis、RocketMQ、LLM。
      </p>
      <div class="project-tech">
        <span class="tag">Java</span>
        <span class="tag">Spring Boot</span>
        <span class="tag">Redis</span>
        <span class="tag">RocketMQ</span>
        <span class="tag">MySQL</span>
        <span class="tag">LLM</span>
      </div>
    </div>

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
        农业数据采集 Pipeline
      </h3>
      <p class="project-desc">
        外部气象数据与农产品市场行情的定向采集管道。运用 Headless 浏览器技术突破动态渲染及反爬策略限制，
        构建稳定的农业领域语料抓取系统。
      </p>
      <div class="project-tech">
        <span class="tag">Headless Browser</span>
        <span class="tag">Python</span>
        <span class="tag">Playwright</span>
        <span class="tag">RustFS</span>
      </div>
    </div>

    <div class="project-card">
      <h3 class="project-title">
        <i class="fas fa-robot" style="color: var(--color-accent);"></i>
        AI 智能问答系统
      </h3>
      <p class="project-desc">
        将大语言模型（LLM）能力引入农业平台，配合 RAG 知识库检索技术实现病虫害防治、作物生长周期预测等
        智能化问答交互服务。采用 Spring AI + pgvector 技术栈。
      </p>
      <div class="project-tech">
        <span class="tag">LLM</span>
        <span class="tag">RAG</span>
        <span class="tag">Spring AI</span>
        <span class="tag">pgvector</span>
      </div>
    </div>
  </div>
</div>

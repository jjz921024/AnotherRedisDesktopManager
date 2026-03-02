# TairHash UI Display Support Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 在 UI 中只读展示 TairHash 数据，包含 Field、值、过期时间和、版本号

**Architecture:** 创建独立的 `KeyContentTairHash.vue` 组件，复用现有 Hash 组件结构但增加 Expire和 Version列，使用vxe-table 展示， FormatViewer 显示值

**Tech Stack:** Vue 2.x, Element UI, vxe-table, ioredis

## Files to Modify
- `src/components/KeyDetail.vue` - 添加类型映射和导入
- `src/components/contents/KeyContentTairHash.vue` - 创建新组件
- `src/i18n/langs/en.js` - 添加英文翻译
- `src/i18n/langs/cn.js` - 添加中文翻译
---


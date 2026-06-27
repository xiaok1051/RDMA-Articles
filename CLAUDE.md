# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

RDMA 技术文章系列。所有文章位于 `articles/` 目录，按编号顺序排列，从基础到进阶递进。

## Project Structure

```
articles/           # 全部 md 文章 (16 篇，编号 01-15)
images/             # 配图 (png/svg)
html/               # 微信公众号等排版后的 HTML 版本
docs/superpowers/   # 文章设计和实施计划
specs/              # 设计文档
plans/              # 实施计划
```

## Article Writing Conventions

- **语言**: 中文技术文章
- **格式**: Markdown，中英文混排时中英文之间加空格
- **配图**: 使用 drawio 制作，导出为 PNG，存放在 `images/` 目录
- **图片引用**: 使用相对路径 `../images/xxx.png`
- **编号**: 新文章按学习路径编号，插入到对应 phase，后续编号顺延

## Article Numbering (Learning Path)

Phase 1 (环境搭建): 01-02
Phase 2 (编程入门): 03-05
Phase 3 (协议网络): 06-07
Phase 4 (硬件接口): 08
Phase 5 (内核调用栈): 09-14
Phase 6 (分布式系统): 15+

## Writing New Articles

1. 先在 `docs/superpowers/specs/` 写设计文档
2. 用 drawio 绘配图并导出
3. 写入 `articles/` 目录，按编号命名
4. 图片引用路径使用 `../images/`

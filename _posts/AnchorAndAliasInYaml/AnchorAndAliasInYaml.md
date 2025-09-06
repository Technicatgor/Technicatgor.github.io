---
layout: post
title: Anchor And Alias in Yaml
date: 2025-09-06 20:00:00 +800
categories: [Docker]
tags: [docker, yaml]
image:
  path: /assets/img/headers/anchor_alias.png
  lqip: data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAA8gAAAH4CAIAAAAZ1VPRAALJIklEQVR4Aeyah5IbuxVE0aQ3h/eUv8L//1MOm/NumxaKp9TVZInO4e1d1QgDXNyMHgyG+vj59+Od
---

## YAML Anchor 與 Alias 範例說明

如何用  **anchor**（錨點）與  **alias**（別名）以及  `<<:`  合併鍵來重用設定：

```yaml
# 定義一個 anchor，設定 GPU 設置
x-gpu-enabled: &gpu-enabled
  devices:
    - driver: nvidia
      count: all
      capabilities:
        - gpu

x-gpu-disabled: &gpu-disabled
  devices: [] # 沒有 GPU

services:
  service_with_gpu:
    image: example/image
    deploy:
      resources:
        reservations:
          <<: *gpu-enabled # 將 gpu-enabled 設定合併過來

  service_without_gpu:
    image: example/image
    deploy:
      resources:
        reservations:
          <<: *gpu-disabled # 將 gpu-disabled 設定合併過來
```

> [!note]
>
> - `&gpu-enabled` 宣告一個錨點名為 gpu-enabled，包含使用 GPU 的設定
> - `&gpu-disabled` 宣告一個錨點名為 gpu-disabled，禁用 GPU 的設定
> - `<<: *gpu-enabled` 表示合併 gpu-enabled 錨點內的內容到此處 - 這樣寫可以讓多個地方重複使用同樣設定，避免 YAML 文件重複冗長
>
>   **這是一種讓 YAML 配置更加乾淨且易於維護的方式。**

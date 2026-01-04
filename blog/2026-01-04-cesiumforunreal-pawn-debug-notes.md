---
title: CesiumForUnreal Pawn：一次从“动不了”到“可控”的调试心路
authors: [Yiyong]
tags: [unreal-engine, ue5, simulation]
slug: cesiumforunreal-pawn-debug-notes
---

这篇是“按时间线”的记录：我在做 CesiumForUnreal Pawn 时遇到的现象、定位路径、调试手段，以及最后沉淀下来的检查清单。

如果你想看更体系化、可长期维护的版本（结构/输入/关键概念/排错清单），对应的 Docs 在这里：[/docs/unreal-engine/cesium-for-unreal-pawn](/docs/unreal-engine/cesium-for-unreal-pawn)

<!-- truncate -->

## 背景

目标很简单：做一个可以在 Cesium 全球场景里稳定漫游的 Pawn。

我预期它至少要满足：

- WASD 能移动、鼠标能转向
- 在离原点很远的位置也不至于明显抖动
- 后续能接入“按经纬度/高度飞行到目标点”的功能

## 现象（Symptoms）

最常见的三类“第一天就会撞上”的问题：

1. Pawn 看起来被生成了，但输入完全不生效
2. 能转向但不能移动（或只能在某个轴动）
3. 在某些高度/距离开始出现明显抖动、穿模或视角跳变

## 我当时的排查顺序

### 1) 先证明：输入有没有进来

- 先不急着怀疑 Cesium：把 Pawn 简化成「不接任何 Cesium 逻辑」的最小骨架
- 在输入绑定处打日志/断点，确认 Axis/Action 数值在变化

如果输入没进来：

- 检查是否被 `PlayerController` possess
- 检查 Project Settings / Enhanced Input 的 Mapping 是否真的挂到本地玩家

### 2) 再证明：移动是不是被“别的逻辑”覆盖

当你看到 Pawn “像是动了但又瞬间被拉回去”，通常意味着：

- 你在某个 Tick/回调里又把 Transform 重置了
- 或者地理对齐/原点漂移相关逻辑在每帧强行改位置

做法：临时注释掉所有“设置 Actor Transform”的代码，只保留 `AddMovementInput` 看行为是否恢复。

### 3) 最后处理：全球尺度下的抖动/精度

当移动/转向都正常了，再把“抖动”当作一个独立问题来解决：

- 记录出现抖动时的距离/高度（越远越明显通常是精度）
- 检查是否存在原点漂移/地理锚定的重复设置
- 尽量用“增量移动”，避免直接给一个巨大的绝对 UE 世界坐标

## 这次调试留下的 5 个要点

- 把问题切片：输入 → possess → movement → transform 是否被覆盖 → 精度
- 先做最小 Pawn 骨架，再逐步接 Cesium 逻辑（一次只加一层复杂度）
- 任何“每帧设置 Transform”的代码都要特别谨慎
- 先保证控制路径唯一：Controller 旋转 or Pawn 旋转 or Camera 旋转，不要叠加
- 写下你的检查清单，并固化到 Docs（方便以后复用/给队友）

## 下一步我打算补充的内容

- 一个“飞行速度曲线”（高度越高速度越快）的简单实现
- 一个“按经纬度/高度飞行到目标点”的接口封装
- 抖动问题的具体复现场景与修复前后对比

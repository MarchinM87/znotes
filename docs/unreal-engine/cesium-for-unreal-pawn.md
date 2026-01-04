---
title: CesiumForUnreal 的 Pawn：结构、输入与调试要点
tags: [unreal-engine, ue5, simulation]
---

这篇是“长期可维护”的手册页：把 CesiumForUnreal 场景里常见的 Pawn 设计选择、坐标/精度问题、输入与调试重点整理成一个可复用的参考。

如果你想看一篇按时间线写的调试心路（“我当时遇到什么问题、怎么定位、怎么验证假设”），看这篇 Blog：[/blog/cesiumforunreal-pawn-debug-notes](/blog/cesiumforunreal-pawn-debug-notes)

## 1. 这类 Pawn 的目标是什么

在 CesiumForUnreal 里，Pawn 往往承担的是“全球尺度场景中的观察/交互载体”，常见目标：

- 可以稳定地在高空/近地表切换观察，不因浮点精度导致抖动
- 支持飞行/漫游/轨道相机等控制模式
- 在需要时能把“地理坐标（经纬度/高度）”和“UE 世界坐标”互相转换

## 2. Pawn 选型：Pawn + Movement vs Character

多数“飞行/漫游相机”场景下，优先考虑：

- `APawn` + `UFloatingPawnMovement`（或自定义 MovementComponent）

只有当你明确要用 UE 的“角色地面行走/胶囊碰撞/CharacterMovement”整套能力时，才选择 `ACharacter`。

原因（经验向，不同项目会有例外）：

- 全球尺度/大范围移动时，`CharacterMovement` 牵涉到地面检测、步进、网络预测等，调参/排错成本更高
- 许多 Cesium 漫游体验更像“相机 Rig”，用轻量 Movement 更容易控制与定位问题

## 3. 必须提前想清楚的 3 件事

### 3.1 世界原点漂移与浮点精度

全球尺度下，UE 世界坐标是单精度浮点（float）为主：离原点越远，抖动/穿模/旋转不稳定越明显。

你的 Pawn 设计要明确：

- 谁负责“把关注区域保持在原点附近”？（通常是 Cesium 的地理参考/原点管理机制 + 你的 Pawn/相机更新逻辑配合）
- 你的移动逻辑是不是“增量移动”（AddMovementInput）而不是直接设置一个很大的绝对坐标？

### 3.2 坐标系：地理坐标 vs UE 坐标

建议在文档里固定术语：

- “地理坐标”：经纬度（度）+ 高度（米）或 ECEF
- “UE 坐标”：Unreal 世界坐标（cm）

并明确：

- 哪些系统使用地理坐标作为“权威坐标”（比如保存书签、相机飞行目标）
- 哪些系统只在本地用 UE 坐标做物理/碰撞/动画

### 3.3 相机控制：Yaw/Pitch/Roll 的来源

全球场景里“向上”不一定等于 UE 的 +Z（取决于你如何定义局部切平面、是否让相机始终对齐地表法线等）。

写代码前先决定：

- 你要“相机始终保持地平线”（基于局部 ENU）还是完全自由滚转
- 旋转是基于 Pawn 还是基于 SpringArm/Camera

## 4. 推荐的组件结构（最小可用）

一个简单、可调试的结构通常是：

- Root（SceneComponent）
- (可选) SpringArm
- Camera
- MovementComponent（`UFloatingPawnMovement` 或自定义）

如果你需要把 Pawn 绑定到地理参考/进行地理定位，通常会额外挂一个“地理锚定/地理对齐”类组件（具体名称随 CesiumForUnreal 版本而不同）。

> 这里我刻意不把具体类名写死成某个版本的 Cesium API；建议你在补充项目细节时，把你实际使用的 Cesium 版本号和组件名写在本节。

## 5. 输入（Enhanced Input）要点

建议把输入拆分成 3 组动作，方便调试和后续拓展：

- `Move`：WASD/手柄左摇杆 → 平移
- `Look`：鼠标/手柄右摇杆 → 视角
- `Boost` / `SpeedUp`：Shift/滚轮 → 速度倍率

调试技巧：先在“没有 Cesium 任何额外逻辑”的情况下，把 Pawn 的移动/旋转跑通；再逐步接入地理锚定、原点漂移等机制。

## 6. 一段最小 C++ Pawn 框架（可直接当模板）

> 这段代码不依赖 Cesium 头文件，目的是把「可控、可调试」的 Pawn 骨架先立住；你再在此基础上接 Cesium 的地理对齐/飞行目标等逻辑。

```cpp
// MyCesiumPawn.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Pawn.h"
#include "MyCesiumPawn.generated.h"

class UCameraComponent;
class UFloatingPawnMovement;

UCLASS()
class AMyCesiumPawn : public APawn
{
  GENERATED_BODY()

public:
  AMyCesiumPawn();

  virtual void SetupPlayerInputComponent(UInputComponent* PlayerInputComponent) override;

protected:
  virtual void BeginPlay() override;

private:
  UPROPERTY(VisibleAnywhere)
  USceneComponent* Root;

  UPROPERTY(VisibleAnywhere)
  UCameraComponent* Camera;

  UPROPERTY(VisibleAnywhere)
  UFloatingPawnMovement* Movement;

  void MoveForward(float Value);
  void MoveRight(float Value);
  void LookYaw(float Value);
  void LookPitch(float Value);
};
```

```cpp
// MyCesiumPawn.cpp
#include "MyCesiumPawn.h"

#include "Camera/CameraComponent.h"
#include "GameFramework/FloatingPawnMovement.h"
#include "Components/InputComponent.h"

AMyCesiumPawn::AMyCesiumPawn()
{
  PrimaryActorTick.bCanEverTick = false;

  Root = CreateDefaultSubobject<USceneComponent>(TEXT("Root"));
  SetRootComponent(Root);

  Camera = CreateDefaultSubobject<UCameraComponent>(TEXT("Camera"));
  Camera->SetupAttachment(Root);

  Movement = CreateDefaultSubobject<UFloatingPawnMovement>(TEXT("Movement"));

  AutoPossessPlayer = EAutoReceiveInput::Player0;
}

void AMyCesiumPawn::BeginPlay()
{
  Super::BeginPlay();
}

void AMyCesiumPawn::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
  Super::SetupPlayerInputComponent(PlayerInputComponent);

  PlayerInputComponent->BindAxis(TEXT("MoveForward"), this, &AMyCesiumPawn::MoveForward);
  PlayerInputComponent->BindAxis(TEXT("MoveRight"), this, &AMyCesiumPawn::MoveRight);
  PlayerInputComponent->BindAxis(TEXT("LookYaw"), this, &AMyCesiumPawn::LookYaw);
  PlayerInputComponent->BindAxis(TEXT("LookPitch"), this, &AMyCesiumPawn::LookPitch);
}

void AMyCesiumPawn::MoveForward(float Value)
{
  if (FMath::IsNearlyZero(Value)) return;
  AddMovementInput(GetActorForwardVector(), Value);
}

void AMyCesiumPawn::MoveRight(float Value)
{
  if (FMath::IsNearlyZero(Value)) return;
  AddMovementInput(GetActorRightVector(), Value);
}

void AMyCesiumPawn::LookYaw(float Value)
{
  AddControllerYawInput(Value);
}

void AMyCesiumPawn::LookPitch(float Value)
{
  AddControllerPitchInput(Value);
}
```

## 7. 调试清单（排“动不了/抖动/方向不对”最快）

- Pawn 是否真的被 Possess（PIE 时看 `PlayerController` 的 `GetPawn()`）
- 输入映射是否生效（先用 Axis Mapping / Enhanced Input 的 Debug 看数值有没有变化）
- 运动是“增量”还是“设绝对位置”（全球场景更推荐增量）
- 是否被碰撞体/地形阻挡（先临时关碰撞或把 Capsule/Root 提高验证）
- 相机旋转是加在 Controller、Pawn 还是 Camera 上（先固定一种路径，避免多处叠加）
- 若出现“越飞越抖”：重点检查原点漂移/地理对齐是否在 Tick 中反复重置 Transform

## 8. 这页建议你补充的项目特定信息

- 你使用的 CesiumForUnreal 版本号
- 你实际选择的地理锚定/对齐组件与原因
- 你的 Pawn 控制模式（第一人称/自由飞行/轨道）
- 你遇到过的 2–3 个典型 bug（每个 bug：现象 → 假设 → 验证方法 → 修复）

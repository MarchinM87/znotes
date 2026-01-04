---
sidebar_position: 1
tags: [unreal, ue5, gas, gameplay, architecture]
---

# Gameplay Ability System (GAS) 基础

> UE5的模块化技能系统框架，用于构建复杂的游戏能力和属性系统。

## 📋 概述

**用途：** 提供可扩展的游戏技能、属性、效果系统。

**核心组件：**
- **Ability System Component (ASC)** - 核心组件
- **Gameplay Abilities** - 技能/能力
- **Gameplay Effects** - 效果（buff/debuff）
- **Gameplay Attributes** - 属性（HP, Mana等）
- **Gameplay Tags** - 标签系统

---

## 💡 核心架构

### 1. Ability System Component 设置

```cpp
// Character.h
#include "AbilitySystemInterface.h"
#include "AbilitySystemComponent.h"
#include "AttributeSet.h"

UCLASS()
class AMyCharacter : public ACharacter, public IAbilitySystemInterface
{
    GENERATED_BODY()

protected:
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Abilities")
    UAbilitySystemComponent* AbilitySystemComponent;

    UPROPERTY()
    const UAttributeSet* AttributeSet;

public:
    AMyCharacter();

    // IAbilitySystemInterface
    virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;

    // 初始化默认技能
    void GiveDefaultAbilities();
};

// Character.cpp
AMyCharacter::AMyCharacter()
{
    AbilitySystemComponent = CreateDefaultSubobject<UAbilitySystemComponent>(TEXT("AbilitySystemComp"));
    AbilitySystemComponent->SetIsReplicated(true);
    AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Mixed);
    
    AttributeSet = CreateDefaultSubobject<UMyAttributeSet>(TEXT("AttributeSet"));
}

UAbilitySystemComponent* AMyCharacter::GetAbilitySystemComponent() const
{
    return AbilitySystemComponent;
}
```

### 2. 定义属性集 (Attribute Set)

```cpp
// MyAttributeSet.h
#include "AttributeSet.h"
#include "AbilitySystemComponent.h"

// 使用宏简化属性访问器
#define ATTRIBUTE_ACCESSORS(ClassName, PropertyName) \
    GAMEPLAYATTRIBUTE_PROPERTY_GETTER(ClassName, PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_GETTER(PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_SETTER(PropertyName) \
    GAMEPLAYATTRIBUTE_VALUE_INITTER(PropertyName)

UCLASS()
class UMyAttributeSet : public UAttributeSet
{
    GENERATED_BODY()

public:
    UMyAttributeSet();

    // 属性定义
    UPROPERTY(BlueprintReadOnly, Category = "Attributes", ReplicatedUsing = OnRep_Health)
    FGameplayAttributeData Health;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, Health)

    UPROPERTY(BlueprintReadOnly, Category = "Attributes", ReplicatedUsing = OnRep_MaxHealth)
    FGameplayAttributeData MaxHealth;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, MaxHealth)

    UPROPERTY(BlueprintReadOnly, Category = "Attributes", ReplicatedUsing = OnRep_Mana)
    FGameplayAttributeData Mana;
    ATTRIBUTE_ACCESSORS(UMyAttributeSet, Mana)

protected:
    // 属性变化回调
    virtual void PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data) override;

    // 网络复制
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

    UFUNCTION()
    virtual void OnRep_Health(const FGameplayAttributeData& OldHealth);
    
    UFUNCTION()
    virtual void OnRep_MaxHealth(const FGameplayAttributeData& OldMaxHealth);
    
    UFUNCTION()
    virtual void OnRep_Mana(const FGameplayAttributeData& OldMana);
};

// MyAttributeSet.cpp
#include "Net/UnrealNetwork.h"

UMyAttributeSet::UMyAttributeSet()
{
    // 初始化默认值
    InitHealth(100.f);
    InitMaxHealth(100.f);
    InitMana(50.f);
}

void UMyAttributeSet::PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data)
{
    Super::PostGameplayEffectExecute(Data);

    // 处理属性变化
    if (Data.EvaluatedData.Attribute == GetHealthAttribute())
    {
        // 限制血量范围
        SetHealth(FMath::Clamp(GetHealth(), 0.f, GetMaxHealth()));
        
        // 死亡检测
        if (GetHealth() <= 0.f)
        {
            // 触发死亡逻辑
        }
    }
}

void UMyAttributeSet::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, Health, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, MaxHealth, COND_None, REPNOTIFY_Always);
    DOREPLIFETIME_CONDITION_NOTIFY(UMyAttributeSet, Mana, COND_None, REPNOTIFY_Always);
}

void UMyAttributeSet::OnRep_Health(const FGameplayAttributeData& OldHealth)
{
    GAMEPLAYATTRIBUTE_REPNOTIFY(UMyAttributeSet, Health, OldHealth);
}
```

### 3. 创建 Gameplay Ability

```cpp
// MyGameplayAbility.h
#include "Abilities/GameplayAbility.h"

UCLASS()
class UMyGameplayAbility : public UGameplayAbility
{
    GENERATED_BODY()

public:
    UMyGameplayAbility();

    // 技能执行入口
    virtual void ActivateAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        const FGameplayEventData* TriggerEventData
    ) override;

    // 技能结束
    virtual void EndAbility(
        const FGameplayAbilitySpecHandle Handle,
        const FGameplayAbilityActorInfo* ActorInfo,
        const FGameplayAbilityActivationInfo ActivationInfo,
        bool bReplicateEndAbility,
        bool bWasCancelled
    ) override;

protected:
    UPROPERTY(EditDefaultsOnly, Category = "Costs")
    TSubclassOf<UGameplayEffect> CostGameplayEffect;

    UPROPERTY(EditDefaultsOnly, Category = "Cooldowns")
    TSubclassOf<UGameplayEffect> CooldownGameplayEffect;
};

// MyGameplayAbility.cpp
UMyGameplayAbility::UMyGameplayAbility()
{
    // 设置默认策略
    InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
    NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalPredicted;
}

void UMyGameplayAbility::ActivateAbility(
    const FGameplayAbilitySpecHandle Handle,
    const FGameplayAbilityActorInfo* ActorInfo,
    const FGameplayAbilityActivationInfo ActivationInfo,
    const FGameplayEventData* TriggerEventData)
{
    if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
        return;
    }

    // 技能逻辑
    // ...

    // 可以使用AbilityTask来处理异步操作
    // 例如：等待输入、播放动画、等待延迟等
}
```

---

## 🎯 实际应用：火球术技能

```cpp
// FireballAbility.h
UCLASS()
class UFireballAbility : public UMyGameplayAbility
{
    GENERATED_BODY()

public:
    UFireballAbility();

protected:
    virtual void ActivateAbility(...) override;

    UPROPERTY(EditDefaultsOnly, Category = "Projectile")
    TSubclassOf<class AProjectile> ProjectileClass;

    UPROPERTY(EditDefaultsOnly, Category = "Effects")
    TSubclassOf<UGameplayEffect> DamageEffect;

    UFUNCTION()
    void OnProjectileHit(AActor* HitActor);
};

// FireballAbility.cpp
UFireballAbility::UFireballAbility()
{
    AbilityTags.AddTag(FGameplayTag::RequestGameplayTag(FName("Ability.Skill.Fireball")));
    ActivationOwnedTags.AddTag(FGameplayTag::RequestGameplayTag(FName("State.Casting")));
    BlockAbilitiesWithTag.AddTag(FGameplayTag::RequestGameplayTag(FName("State.Dead")));
}

void UFireballAbility::ActivateAbility(...)
{
    if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
    {
        EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
        return;
    }

    // 生成火球
    FVector SpawnLocation = ActorInfo->AvatarActor->GetActorLocation();
    FRotator SpawnRotation = ActorInfo->AvatarActor->GetActorRotation();
    
    AProjectile* Projectile = GetWorld()->SpawnActor<AProjectile>(
        ProjectileClass, SpawnLocation, SpawnRotation);
    
    if (Projectile)
    {
        Projectile->OnHit.AddDynamic(this, &UFireballAbility::OnProjectileHit);
    }

    EndAbility(Handle, ActorInfo, ActivationInfo, true, false);
}

void UFireballAbility::OnProjectileHit(AActor* HitActor)
{
    if (IAbilitySystemInterface* ASI = Cast<IAbilitySystemInterface>(HitActor))
    {
        UAbilitySystemComponent* TargetASC = ASI->GetAbilitySystemComponent();
        if (TargetASC)
        {
            // 应用伤害效果
            FGameplayEffectContextHandle EffectContext = TargetASC->MakeEffectContext();
            EffectContext.AddSourceObject(this);
            
            FGameplayEffectSpecHandle SpecHandle = TargetASC->MakeOutgoingSpec(
                DamageEffect, 1.f, EffectContext);
            
            if (SpecHandle.IsValid())
            {
                TargetASC->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get());
            }
        }
    }
}
```

---

## ⚠️ 重要注意事项

### 网络复制

1. **ASC复制模式：**
   - `Full` - 完全复制（单人游戏）
   - `Mixed` - 混合模式（多人推荐）
   - `Minimal` - 最小复制（AI）

2. **客户端预测：**
   - 技能激活支持预测
   - GE应用可以预测
   - 需要正确设置NetExecutionPolicy

### 性能优化

- 使用Gameplay Tag容器而非字符串
- 合理使用GE的Duration类型（Instant vs Duration vs Infinite）
- AttributeSet中避免频繁的网络复制

---

## 📚 面试要点

**常见问题：**

1. **GAS的核心组件有哪些？**
   - ASC, Abilities, Effects, Attributes, Tags

2. **Gameplay Effect的三种持续时间类型？**
   - Instant（瞬时）、Duration（持续）、Infinite（永久）

3. **如何处理网络同步？**
   - ASC自带复制支持
   - 使用ReplicationMode配置
   - AbilityTask支持客户端预测

4. **GAS相比传统实现的优势？**
   - 模块化、可扩展
   - 内置网络复制
   - Tag系统便于管理
   - 支持客户端预测

**实战建议：**
- 理解GE的Modifier和Execution
- 熟悉AbilityTask的使用
- 掌握Tag系统的查询和管理
- 了解客户端预测的原理

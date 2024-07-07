# GamePlay Ability System源码学习

## AbilitySystemComponent

### 什么是AbilitySystemComponent

AbilitySystemComponent(ASC，能力系统组件)是继承于UActorComponent的一个组件，它可以和AbilitySystem(能力系统)中的3个方面交互：GameplayAbilities(GA，能力/技能)、GameplayEffects(GE，效果/buff)、GameplayAttributes(属性)。挂载能力系统组件的Actor都可以使用技能，施加buff以及拥有自己的属性。要想Actor之间通过GA,GE来交互就必须挂载能力系统组件。

### 重要的成员变量/方法

#### **TArray<TObjectPtr\<UAttributeSet> SpawnedAttributes**

存放了这个ASC组件所拥有的AttributeSet，看几组函数

```c++
/** Access the spawned attributes list when you don't intend to modify the list. */
const TArray<UAttributeSet*>& UAbilitySystemComponent::GetSpawnedAttributes() const
{
	return SpawnedAttributes;
}
/** Add a new attribute set */
void UAbilitySystemComponent::AddSpawnedAttribute(UAttributeSet* Attribute)
{
	if (!IsValid(Attribute))
	{
		return;
	}

	if (SpawnedAttributes.Find(Attribute) == INDEX_NONE)
	{
		if (IsUsingRegisteredSubObjectList() && IsReadyForReplication())
		{
			AddReplicatedSubObject(Attribute);
		}

		SpawnedAttributes.Add(Attribute);
		SetSpawnedAttributesListDirty();
	}
}
/** Remove an existing attribute set */
void RemoveSpawnedAttribute(UAttributeSet* Attribute);
```

对AttributeSet的增删改查都是通过SpawnedAttributes。

#### ApplyGameplayEffectSpecToTarget/ApplyGameplayEffectSpecToSelf

```c++
virtual FActiveGameplayEffectHandle ApplyGameplayEffectSpecToTarget(const FGameplayEffectSpec& GameplayEffect, UAbilitySystemComponent *Target, FPredictionKey PredictionKey=FPredictionKey());

/** Applies a previously created gameplay effect spec to a target */
UFUNCTION(BlueprintCallable, Category = GameplayEffects, meta = (DisplayName = "ApplyGameplayEffectSpecToTarget", ScriptName = "ApplyGameplayEffectSpecToTarget"))
FActiveGameplayEffectHandle BP_ApplyGameplayEffectSpecToTarget(const FGameplayEffectSpecHandle& SpecHandle, UAbilitySystemComponent* Target);

virtual FActiveGameplayEffectHandle ApplyGameplayEffectSpecToSelf(const FGameplayEffectSpec& GameplayEffect, FPredictionKey PredictionKey = FPredictionKey());
```

向目标施加GameplayEffect的行为都是通过ApplyGameplayEffectSpecToSelf函数。例如ApplyGameplayEffectSpecToTarget函数

```c++
FActiveGameplayEffectHandle UAbilitySystemComponent::ApplyGameplayEffectSpecToTarget(const FGameplayEffectSpec &Spec, UAbilitySystemComponent *Target, FPredictionKey PredictionKey)
{
	SCOPE_CYCLE_COUNTER(STAT_AbilitySystemComp_ApplyGameplayEffectSpecToTarget);
	UAbilitySystemGlobals& AbilitySystemGlobals = UAbilitySystemGlobals::Get();

	if (!AbilitySystemGlobals.ShouldPredictTargetGameplayEffects())
	{
		// If we don't want to predict target effects, clear prediction key
		PredictionKey = FPredictionKey();
	}

	FActiveGameplayEffectHandle ReturnHandle;
	//最终都是通过自身的ASC调用ApplyGameplayEffectSpecToSelf
	if (Target)
	{
		ReturnHandle = Target->ApplyGameplayEffectSpecToSelf(Spec, PredictionKey);
	}

	return ReturnHandle;
}
```

函数名带有BP_的都是对应方法的蓝图方法。

#### MakeOutgoingSpec

创建一个即将应用的GE

```c++
/** Get an outgoing GameplayEffectSpec that is ready to be applied to other things. */
UFUNCTION(BlueprintCallable, Category = GameplayEffects)
virtual FGameplayEffectSpecHandle MakeOutgoingSpec(TSubclassOf<UGameplayEffect> GameplayEffectClass, float Level, FGameplayEffectContextHandle Context) const;
```

主要是通过根据传入的GE和level创建一个GE实例

```c++
FGameplayEffectSpecHandle UAbilitySystemComponent::MakeOutgoingSpec(TSubclassOf<UGameplayEffect> GameplayEffectClass, float Level, FGameplayEffectContextHandle Context) const
{
	SCOPE_CYCLE_COUNTER(STAT_GetOutgoingSpec);
	if (Context.IsValid() == false)
	{
		Context = MakeEffectContext();
	}

	if (GameplayEffectClass)
	{
		UGameplayEffect* GameplayEffect = GameplayEffectClass->GetDefaultObject<UGameplayEffect>();

		FGameplayEffectSpec* NewSpec = new FGameplayEffectSpec(GameplayEffect, Context, Level);
		return FGameplayEffectSpecHandle(NewSpec);
	}

	return FGameplayEffectSpecHandle(nullptr);
}
```

#### MakeEffectContext

```c++
/** Create an EffectContext for the owner of this AbilitySystemComponent */
UFUNCTION(BlueprintCallable, Category = GameplayEffects)
virtual FGameplayEffectContextHandle MakeEffectContext() const;
```

#### GameplayTags operations

ASC中包含一个成员叫做：GameplayTagCountContainer，存放了一堆GameplayTags

可以通过以下函数查看是否存在Tag

```c++
bool HasMatchingGameplayTag(FGameplayTag TagToCheck)
```

#### GiveAbility

学习一个技能，技能在被释放之前需要先学习

```c++
FGameplayAbilitySpecHandle GiveAbility(const FGameplayAbilitySpec& AbilitySpec);
```

#### TryActivateAbilitiesByTag

尝试激活所有和Tag匹配的Ability

```c++
/** 
 * Attempts to activate every gameplay ability that matches the given tag and DoesAbilitySatisfyTagRequirements().
 * Returns true if anything attempts to activate. Can activate more than one ability and the ability may fail later.
 * If bAllowRemoteActivation is true, it will remotely activate local/server abilities, if false it will only try to locally activate abilities.
 */
UFUNCTION(BlueprintCallable, Category = "Abilities")
bool TryActivateAbilitiesByTag(const FGameplayTagContainer& GameplayTagContainer, bool bAllowRemoteActivation = true);
```



## 常见错误

### 添加AbilitySystemComponent.h时vs报错

报错信息：无法打开源文件xxxx.h

解决办法：项目==\>属性==>VC++目录==>包含目录

![image-20240707135645271](C:/Users/octanezhong/AppData/Roaming/Typora/typora-user-images/image-20240707135645271.png)

添加好目录后重新打开项目


# GamePlay Ability System源码学习

## AbilitySystemComponent

### 什么是AbilitySystemComponent

AbilitySystemComponent(ASC，能力系统组件)是继承于UActorComponent的一个组件，它可以和AbilitySystem(能力系统)中的3个方面交互：GameplayAbilities(GA，能力/技能)、GameplayEffects(GE，效果/buff)、GameplayAttributes(属性)。挂载能力系统组件的Actor都可以使用技能，施加buff以及拥有自己的属性。要想Actor之间通过GA,GE来交互就必须挂载能力系统组件。

### InitAbilityActorInfo( AActor* InOwnerActor, AActor* InAvatarActor)

在挂载ASC组件到Actor上时最好在Actor的BeginPlay函数中调用ASC的这个函数.

```
void Actor::BeginPlay()
{
	ASC->InitAbilityActorInfo(this,this);
}
```

InitAbilityActorInfo有两个参数，OwnerActor和AvatarActor。一般OwnerActor是我们的Controller/PlayerState，AvatarActor是我们扮演的Character / Pawn / Building。区别在于，你是否想让你的Character所拥有的技能或者某些状态具有持久化。最经典的就是MOBA游戏，当你的英雄死亡后，ASC的AvatarActor‘被销毁，如果OwnerActor和AvatarActor是同一个，那么你的英雄在复活后就会失去以前所学会的技能。为了避免这种情况，可以把OwnerActor设置为PlayerState / Controller。

### GetAbilitySystemComponent

在有些情况下，你会想要部分Actor（例如玩家的Pawn或角色）使用其它Actor（例如玩家状态或玩家控制器）拥有的技能系统组件。这种情况一般涉及到玩家分数或者长期存在的技能冷却计时器，因为即便是玩家的Pawn或角色被销毁并重生，或者玩家拥有一个新的Pawn或角色，这些内容也不会重置。游戏玩法技能系统便可以支持此行为；如果你需要使用它，就要编写Actor的`GetAbilitySystemComponent`函数，从而让它返回到你想要使用的技能系统组件。
此函数定义在IAbilitySystemInterface中(AbilitySystemInterface.h)

```c++
//MyCharacter.h
virtual UAbilitySystemComponent* GetAbilitySystemComponent()const override;
//MyCharacter.cpp
UAbilitySystemComponent* ATestCharacter::GetAbilitySystemComponent() const
{
	return ASC;
}
//ASC定义
UPROPERTY(BlueprintReadOnly)
TObjectPtr<UAbilitySystemComponent> ASC;
```



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

向目标施加GameplayEffect的行为都是通过ApplyGameplayEffectSpecToSelf函数。例如ApplyGameplayEffectSpecToTarget函数。在使用ApplyGameplayEffectSpecToTarget时需要获得Target的ASC组件。所以挂载ASC组件的Actor要继承IAbilitySystemInterface并重写函数。

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

## GameplayAbility

UGameplayAbility:技能类，包含了技能的逻辑，技能的标签(Tag)，技能cd

```c++
class GAMEPLAYABILITIES_API UGameplayAbility : public UObject, public IGameplayTaskOwnerInterface
{
	/** This ability has these tags */
	UPROPERTY(EditDefaultsOnly, Category = Tags, meta=(Categories="AbilityTagCategory"))
	FGameplayTagContainer AbilityTags;
	/** This GameplayEffect represents the cooldown. It will be applied when the ability is committed and the ability cannot be used again until it is expired. */
	UPROPERTY(EditDefaultsOnly, Category = Cooldowns)
	TSubclassOf<class UGameplayEffect> CooldownGameplayEffectClass;
}
```

### Ability exclusion/canceling

在技能蓝图中的标签一栏

![image-20240713150304396](./../../../Typora_save/GAS/image-20240713150304396.png)

在代码中被定义如下

```c++
/** Abilities with these tags are cancelled when this ability is executed */
UPROPERTY(EditDefaultsOnly, Category = Tags, meta=(Categories="AbilityTagCategory"))
FGameplayTagContainer CancelAbilitiesWithTag;

/** Abilities with these tags are blocked while this ability is active */
UPROPERTY(EditDefaultsOnly, Category = Tags, meta=(Categories="AbilityTagCategory"))
FGameplayTagContainer BlockAbilitiesWithTag;

/** Tags to apply to activating owner while this ability is active. These are replicated if ReplicateActivationOwnedTags is enabled in AbilitySystemGlobals. */
UPROPERTY(EditDefaultsOnly, Category = Tags, meta=(Categories="OwnedTagsCategory"))
FGameplayTagContainer ActivationOwnedTags;

/** This ability can only be activated if the activating actor/component has all of these tags */
UPROPERTY(EditDefaultsOnly, Category = Tags, meta=(Categories="OwnedTagsCategory"))
FGameplayTagContainer ActivationRequiredTags;

/** This ability is blocked if the activating actor/component has any of these tags */
UPROPERTY(EditDefaultsOnly, Category = Tags, meta=(Categories="OwnedTagsCategory"))
FGameplayTagContainer ActivationBlockedTags;

/** This ability can only be activated if the source actor/component has all of these tags */
UPROPERTY(EditDefaultsOnly, Category = Tags, meta=(Categories="SourceTagsCategory"))
FGameplayTagContainer SourceRequiredTags;

/** This ability is blocked if the source actor/component has any of these tags */
UPROPERTY(EditDefaultsOnly, Category = Tags, meta=(Categories="SourceTagsCategory"))
FGameplayTagContainer SourceBlockedTags;

/** This ability can only be activated if the target actor/component has all of these tags */
UPROPERTY(EditDefaultsOnly, Category = Tags, meta=(Categories="TargetTagsCategory"))
FGameplayTagContainer TargetRequiredTags;

/** This ability is blocked if the target actor/component has any of these tags */
UPROPERTY(EditDefaultsOnly, Category = Tags, meta=(Categories="TargetTagsCategory"))
FGameplayTagContainer TargetBlockedTags;
```

以CancelAbilitiesWithTag为例，当为技能的CancelAbilitiesWithTag添加一个名为“posioned”的tag时，此技能会取消能力标签为“posioned”的能力。

### CancelAbility

```c++
/** Destroys instanced-per-execution abilities. Instance-per-actor abilities should 'reset'. Any active ability state tasks receive the 'OnAbilityStateInterrupted' event. Non instance abilities - what can we do? */
virtual void CancelAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, bool bReplicateCancelAbility);

```

用于取消一个游戏能力，包括是否可以取消能力，是否需要在网络上同步取消操作，以及在取消能力时需要执行的自定义逻辑。如果需要执行自定义逻辑，需要为OnGameplayAbilityCancelled绑定函数。

### CommitAbility

提交/确认技能释放，同时会ApplyCoolDown,ApplyCost。如果在编写技能逻辑的时候定义了技能的cd与花费，在这里就会被执行。

```
bool UGameplayAbility::CommitAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, OUT FGameplayTagContainer* OptionalRelevantTags)
{
	// Last chance to fail (maybe we no longer have resources to commit since we after we started this ability activation)
	if (!CommitCheck(Handle, ActorInfo, ActivationInfo, OptionalRelevantTags))
	{
		return false;
	}
	// commitExecute will call function ApplyCost & ApplyCoolDown
	CommitExecute(Handle, ActorInfo, ActivationInfo);

	// Fixme: Should we always call this or only if it is implemented? A noop may not hurt but could be bad for perf (storing a HasBlueprintCommit per instance isn't good either)
	K2_CommitExecute();

	// Broadcast this commitment
	ActorInfo->AbilitySystemComponent->NotifyAbilityCommit(this);

	return true;
}
```

### CommitAbilityCoolDown

底层也是调用ApplyCoolDown函数

### CommitAbilityCost

底层是调用ApplyCost

### ApplyCoolDown & ApplyCost

是通过调用ApplyGameplayEffectToOwner函数

### ApplyGameplayEffectToOwner & ApplyGameplayEffectToTarget

二者都是拿到目标的ASC组件，然后在ASC组件上调用ApplyGameplayEffectSpecToSelf函数，但是ApplyGameplayEffectToTarget处理略微麻烦

### CoolDown

技能冷却，可在蓝图中设置

```c++
/** This GameplayEffect represents the cooldown. It will be applied when the ability is committed and the ability cannot be used again until it is expired. */
UPROPERTY(EditDefaultsOnly, Category = Cooldowns)
TSubclassOf<class UGameplayEffect> CooldownGameplayEffectClass;
```

![image-20240713160120664](./../../../Typora_save/GAS/image-20240713160120664.png)



### FGameplayAbilitySpec

可激活的技能实例，注册在ASC组件上。它定义了什么样的技能(技能的类别/等级/绑定的输入…)同时拥有运行时的状态。

```c++
struct GAMEPLAYABILITIES_API FGameplayAbilitySpec : public FFastArraySerializerItem
{
    
    /** Handle for outside sources to refer to this spec by */
	UPROPERTY()
	FGameplayAbilitySpecHandle Handle;

	/** Ability of the spec (Always the CDO. This should be const but too many things modify it currently) */
	UPROPERTY()
	TObjectPtr<UGameplayAbility> Ability;

	/** Level of Ability */
	UPROPERTY()
	int32	Level;

	/** InputID, if bound */
	UPROPERTY()
	int32	InputID;
}
```

## AttributeSet

属性集，里面包含了一个角色所需要的属性。

定义一个自己的属性集
```c++
UClass()
class TEST_API UMyAttributeSet : public UAttributeSet
{
	GENERATED_BODY()
	
	public:
	UPROPERTY()
	FGameplayAttributeData Health;

}
```

### FGameplayAttributeData

各自的属性就是FGameplayAttributeData。上面的代码中在UMyAttributeSet中定义了一个生命值(Health)属性。属性结构体定义如下：

```c++
USTRUCT(BlueprintType)
struct GAMEPLAYABILITIES_API FGameplayAttributeData
{
	GENERATED_BODY()
	FGameplayAttributeData()
		: BaseValue(0.f)
		, CurrentValue(0.f)
	{}

	FGameplayAttributeData(float DefaultValue)
		: BaseValue(DefaultValue)
		, CurrentValue(DefaultValue)
	{}

	virtual ~FGameplayAttributeData()
	{}

	/** Returns the current value, which includes temporary buffs */
	float GetCurrentValue() const;

	/** Modifies current value, normally only called by ability system or during initialization */
	virtual void SetCurrentValue(float NewValue);

	/** Returns the base value which only includes permanent changes */
	float GetBaseValue() const;

	/** Modifies the permanent base value, normally only called by ability system or during initialization */
	virtual void SetBaseValue(float NewValue);

protected:
	UPROPERTY(BlueprintReadOnly, Category = "Attribute")
	float BaseValue;

	UPROPERTY(BlueprintReadOnly, Category = "Attribute")
	float CurrentValue;
};
```

可以看到，FGameplayAttributeData包含两个浮点数，一个是BaseValue，另一个是CurrentValue。

#### BaseValue & CurrentValue

BaseValue可以理解为一个属性的起始值，CurrentValue是属性的当前值。当你被敌人砍了一刀，扣掉

### UAttributeSet提供的一些虚函数

UAttributeSet提供了一些虚函数，重写这些虚函数以达到自己想要的效果。

#### PreGameplayEffectExecute

在即将修改属性值之前，此函数可以拒绝或更改拟定修改。

#### PostGameplayEffectExecute

在修改属性值后，此函数可立即对更改做出响应。这通常包括限制属性的最终值或触发对新值的游戏内响应，例如当"生命值"属性降至零时死亡。

#### PreAttributeChange/PreAttributeBaseChange

这个函数执行完之后，CurrentValue被修改

PreAttributeBaseChange这个函数结束后baseValue被修改，CurrentValue不修改

PreGameplayEffectExecute—>PreAttributeBaseChange—>PreAttributeChange—>PostAttributeChange—>PostAttributeBaseChange—>PosGameplayEffectExecute

### 帮助宏

ATTRIBUTE_ACCESSORS(ClassName, PropertyName)，会自动生成对Health属性的访问器(GetPropertyName()，SetPropertyName())。在使用这个宏之前确保UMyAttributeSet.h头文件包含了AbilitySystemComponent.h

```c++
UClass()
class TEST_API UMyAttributeSet : public UAttributeSet
{
	GENERATED_BODY()
	
	public:
    //定义生命值属性
	UPROPERTY()
	FGameplayAttributeData Health;
	//帮助宏使用方法
	ATTRIBUTE_ACCESSORS(UMyAttributeSet,Health)
	//定义速度属性
	UPROPERTY()
	FGameplayAttributeData MaxHealth;
	ATTRIBUTE_ACCESSORS(UMyAttributeSet,MaxHealth)
    UPROPERTY()
	FGameplayAttributeData Mana;
	ATTRIBUTE_ACCESSORS(UMyAttributeSet,Mana)
    UPROPERTY()
	FGameplayAttributeData MaxMana;
	ATTRIBUTE_ACCESSORS(UMyAttributeSet,MaxMana)
}
```

上一段代码中定义了Health,MaxHealth,Mana,MaxMana四种属性的访问器。现在假如有一个UMyAttributeSet的实例对象的指针my_Set，使用方法如下。

```c++
if(my_Set)
{
	float Health = my_set->GetHealth();
	float MaxHealth = my_set->GetMaxHealth();
	float Mana = my_set->GetMana();
	float MaxMana = my_set->GetMaxMana();
	UE_LOG(LogTemp,Log,TEXT("Health:%f"),Health);
	UE_LOG(LogTemp,Log,TEXT("MaxHealth:%f"),MaxHealth);
	UE_LOG(LogTemp,Log,TEXT("Mana:%f"),Mana);
	UE_LOG(LogTemp,Log,TEXT("Health:%f"),MaxMana);
}
```

![image-20240723153204936](./../../../Typora_save/GAS/image-20240723153204936.png)

### AttributeSet初始化

#### 手动初始化

如果你想在AttributeSet构造时对属性进行初始化，这种做法是不推荐的。为什么？一方面方便了数值策划填表，另一方面使用GE减少了代码量，可重用性高。

#### 使用GameplayEffect初始化

这种做法涉及到创建一个GameplayEffect句柄，放到后面展开。

#### 使用数据表格初始化



## GameplayTag

标签，形如：AA.BB.CC、Damage.Poision.Continue，通过打标签的方式可以发动技能/区分敌友/施加GE

### 结构定义

展示GameplayTag结构体的基本信息

````c++
USTRUCT()
struct FGameplayTag
{
	FName TagName; //存放Tag
    //拿到和TagName匹配的FGameplayTag
    static GAMEPLAYTAGS_API FGameplayTag RequestGameplayTag(const FName& TagName, bool ErrorIfNotFound=true);
    /**
    *Tag匹配，规则是向上匹配。例如"A.B".MatchesTag("A")/"A.B".MatchesTag("A.B") return true
    *"A".MatchesTag("A.B")	return false;
    *不是A.B.C这种形式的return false
    *没找到return false
    */
    GAMEPLAYTAGS_API bool MatchesTag(const FGameplayTag& TagToCheck) const;
    /**
    *准确匹配，只有当TagToCheck有效并且"A.1".MatchesTagExact("A.1")return true，其他返回false
    */
    FORCEINLINE bool MatchesTagExact(const FGameplayTag& TagToCheck) const;
    /**
    *"A.B".MatchesAny({"A","B"}) return true,这个就是MatchesTag的多匹配版本
    */
    GAMEPLAYTAGS_API bool MatchesAny(const FGameplayTagContainer& ContainerToCheck) const;

};
````



### FGameplayTagContainer

FGameplayTag的容器。

````c++
USTRUCT()
struct FGameplayTagContainer
{
	TArray<FGameplayTag> GameplayTags;
    
	GAMEPLAYTAGS_API void AddTag(const FGameplayTag& TagToAdd);
    
    FORCEINLINE_DEBUGGABLE bool HasTag(const FGameplayTag& TagToCheck) const
	
};
````



## GameplayEffect

效果类，可以用来制作任何伤害/buff

### 定义

```c++
UCLASS()
class GAMEPLAYABILITIES_API UGameplayEffect : public UObject, public IGameplayTagAssetInterface
{
	/** Policy for the duration of this effect */
	UPROPERTY(EditDefaultsOnly, Category=Duration)
	EGameplayEffectDurationType DurationPolicy;

/** Duration in seconds. 0.0 for instantaneous effects; -1.0 for infinite duration. */
	UPROPERTY(EditDefaultsOnly, Category=Duration, meta=(EditCondition="DurationPolicy == 		EGameplayEffectDurationType::HasDuration", EditConditionHides))
	FGameplayEffectModifierMagnitude DurationMagnitude;

/** Period in seconds. 0.0 for non-periodic effects */
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category="Duration|Period", meta=(EditCondition="DurationPolicy != EGameplayEffectDurationType::Instant", EditConditionHides))
	FScalableFloat	Period;

/** If true, the effect executes on application and then at every period interval. If false, no execution occurs until the first period elapses. */
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category="Duration|Period", meta=(EditCondition="true", EditConditionHides)) // EditCondition in FGameplayEffectDetails
	bool bExecutePeriodicEffectOnApplication;

/** How we should respond when a periodic gameplay effect is no longer inhibited */
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category="Duration|Period", meta=(EditCondition="true", EditConditionHides)) // EditCondition in FGameplayEffectDetails
	EGameplayEffectPeriodInhibitionRemovedPolicy PeriodicInhibitionPolicy;

/** Array of modifiers that will affect the target of this effect */
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category=GameplayEffect, meta=(TitleProperty=Attribute))
	TArray<FGameplayModifierInfo> Modifiers;

/** Array of executions that will affect the target of this effect */
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = GameplayEffect)
	TArray<FGameplayEffectExecutionDefinition> Executions;
    
    /** Cues to trigger non-simulated reactions in response to this GameplayEffect such as sounds, particle effects, etc */
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "GameplayCues")
	TArray<FGameplayEffectCue>	GameplayCues;
};
```

这是一个GE蓝图编辑器样式

![image-20240723165602252](./../../../Typora_save/GAS/image-20240723165602252.png)

下面详细展开

### 持续时间策略 DurationPolicy

EGameplayEffectDurationType枚举类型值，分为三种：Instant，Infinite，HasDuration

```c++
UENUM()
enum class EGameplayEffectDurationType : uint8
{
	/** This effect applies instantly */
	Instant,
	/** This effect lasts forever */
	Infinite,
	/** The duration of this effect will be specified by a magnitude */
	HasDuration
};
```

Instant：立即生效的GE，会修改属性的BaseValue和CurrentValue，砍一刀减去20血，调用的是FActiveGameplayEffectsContainer::ExecuteActiveEffectsFrom

Infinite：永久生效的GE，类似装备的属性加成，一直穿着一件衣服，那么这个衣服的加成就会一直生效

HasDuration：有持续时间的GE，可以用来制作buff/debuff或者各种有持续时间的状态，比如宫崎英高家的粪池。

Infinite和HasDuration可以制作成周期生效的GE，比如在宫崎英高家的粪池游了一会后会中毒然后周期扣血。周期性的GE会被FActiveGameplayEffectsContainer::ExecutePeriodicGameplayEffect所调用，然后执行ExecuteActiveEffectsFrom。

#### Infinite

![image-20240723171230661](../../../Typora_save/GAS/image-20240723171230661.png)

由于Period是一个FScalableFloat，所以可以使用一个CurveTable来初始化Period

#### HasDuration

![image-20240723172527411](../../../Typora_save/GAS/image-20240723172527411.png)

HasDuration类型的GE，多了一个DM，可以对HasDuration类型的GE的持续时间根据“幅度计算类型”来自定义

##### 幅度计算类型

分为：可扩展浮点、Attribute Based、Custom Calculation Class
1.可扩展浮点使用的是一个FScalebleFloat来确定GE持续的时间
2.Attribute Based

基于属性值计算时间。计算公式：( 系数 × ( 预乘加值 + [ 根据策略计算的属性值 ])) + 后乘加值
要捕获的属性：选择一个Attribute
属性源：(要捕获的属性)是来自施法者，还是接收者
属性曲线：如果属性曲线不为空，那么在计算的时候会根据属性曲线的值来计算，否则根据捕获的属性值来计算。
属性计算类型：分为属性幅度，属性基值，属性幅度加值。

例如根据Health属性来确定GE生效的时间，在游戏开始时Health是100，Mana是50。周期10，那么这个GE会生效11次，为什么会是11次？首先Health/10=10，然后GE在应用的时候也会生效一次，故11次。

![image-20240724092923842](./../../../Typora_save/GAS/image-20240724092923842.png)

3.Custom Calculation Class

![image-20240723173752411](./../../../Typora_save/GAS/image-20240723173752411.png)

### Gameplay效果

#### 组件

这会让你在应用GE时添加一些额外操作
![image-20240724110221871](./../../../Typora_save/GAS/image-20240724110221871.png)

#### 修饰符

修饰符提供了这个GE会对目标属性做何种运算，有＋× ÷ 重载 无效五种
＋× ÷顾名思义，重载就是“初始化”，比如现在Health：100，重载给的值为20，那么GE生效后的值为20。
无效：对一个Attribute使用无效的GE后，这个属性就不可以被访问。访问会异常

### 堆叠

堆叠描述了多个相同的GE是如何被处理的。

#### 堆叠样式

按源聚合：每个施法者/施加GE的ASC都一个自己的堆栈
按目标聚合：被施加GE的对象对这种GE只有一个堆栈。

#### Stack Limit Count

描述了这个堆栈的容量。最多支持这类GE存放多少个。

TODO：实践

#### 堆栈持续时间刷新策略

TODO

#### 堆栈周期重设策略

TODO

#### Satck Expiration Policy

TODO

#### 溢出

施加的GE超出了stack count时的操作，例如5次毒伤害变成一个持续时间的中毒debuff

### 创建一个GameplayEffect

#### 在蓝图中创建

首先，你需要创建一个蓝图类让它继承自GameplayEffect，然后配置好这个GE。在角色蓝图中应用
![image-20240725094725048](./../../../Typora_save/GAS/image-20240725094725048.png)
ASC是AbilitySystemComponent实例对象

#### 在c++中创建

定义一个创建GE的函数，在C++中创建GE优点折磨人。

```c++
UGameplayEffect* CreateMyGameplayEffect()
{
    UGameplayEffect* NewEffect = NewObject<UMyGameplayEffect>();

    // 设置持续时间策略为 HasDuration
    NewEffect->DurationPolicy = EGameplayEffectDurationType::HasDuration;
    NewEffect->DurationMagnitude = FScalableFloat(10.0f); // 持续时间为 10 秒

    // 添加一个修饰符
    FGameplayModifierInfo ModifierInfo;
    ModifierInfo.Attribute = UMyAttributeSet::GetHealthAttribute();
    ModifierInfo.ModifierOp = EGameplayModOp::Additive;
    ModifierInfo.Magnitude = FScalableFloat(50.0f); // 增加 50 点健康值

    NewEffect->Modifiers.Add(ModifierInfo);

    return NewEffect;
}
```

应用GE

````c++
void ApplyMyGameplayEffect(UAbilitySystemComponent* AbilitySystemComponent)
{
    UMyGameplayEffect* MyEffect = CreateMyGameplayEffect();

    FGameplayEffectContextHandle EffectContext = AbilitySystemComponent->MakeEffectContext();
    FGameplayEffectSpecHandle SpecHandle = AbilitySystemComponent->MakeOutgoingSpec(MyEffect, 1.0f, EffectContext);

    if (SpecHandle.IsValid())
    {
        FActiveGameplayEffectHandle ActiveGEHandle = AbilitySystemComponent->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get());
    }
}
````

基本流程就是先调用ASC–>MakeEffectContext()创建一个FGameplayEffectContextHandle，然后把它传入ASC–>MakeOutgoingSpec，最后在ASC–>ApplyGameplayEffectSpecToSelf / ApplyGameplayEffectSpecToTarget

#### 蓝图和C++

你可以在蓝图中把GE配置好，然后在c++中调用。基本的思路就是使用UE的静态加载资源的方法，当然也可以动态加载
```c++
//查找蓝图，在MyCharactor的构造函数中使用
static ConstructorHelpers::FClassFinder<UGameplayEffect> BP_GE(TEXT("/Game/BluePrints/BP_Init.BP_Init_C"));
if (BP_GE.Succeeded())
{
	my_BPClass = BP_GE.Class;	
}

//my_BPClass定义在MyCharactor.h文件中
TSubclassOf<UGameplayEffect> my_BPClass;

//在MyCharactor.cpp的BeginPlay中应用GE
if (my_BPClass)
{	
	FGameplayEffectContextHandle effectHandle = ASC->MakeEffectContext();
	FGameplayEffectSpecHandle specHandle = ASC->MakeOutgoingSpec(my_BPClass, 0, effectHandle);
	if (specHandle.IsValid())
	{
		ASC->ApplyGameplayEffectSpecToSelf(*specHandle.Data.Get());

	}
}
```



## 常见问题

### 添加AbilitySystemComponent.h时vs报错

报错信息：无法打开源文件xxxx.h

解决办法：项目==\>属性==>VC++目录==>包含目录

![image-20240707135645271](./../../../Typora_save/GAS/image-20240707135645271.png)

添加好目录后重新打开项目

### 如何Debug

运行游戏后键入\`，就是ESC下面的那个。然后输入showdebug abilitysystem。屏幕就会出现
![image-20240726112458515](./../../../Typora_save/GAS/image-20240726112458515.png)


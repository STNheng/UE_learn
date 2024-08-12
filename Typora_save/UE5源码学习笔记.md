# UE5源码学习笔记

自2024.6.14起

## Character

ACharacter(角色)继承自APawn(棋子)，它区别于APawn的是：ACharacter自带网格体，碰撞以及“内嵌”运动逻辑。这些组件负责和玩家/AI/世界进行物理交互，同时也实现基本的联网和输入模块。通过CharacterMovementComponent玩家可以实现walk(行走),jump(跳跃),swim(游泳)等功能。

一个角色最基本的属性包括：Mesh(网格体)，CharacterMovement(角色移动)，CapsuleComponent(胶囊体组件)，ArrowComponent(箭头组件)。
``` c++
	/** The main skeletal mesh associated with this Character (optional sub-object). */
	UPROPERTY(Category=Character, VisibleAnywhere, BlueprintReadOnly, meta=(AllowPrivateAccess = "true"))
	TObjectPtr<USkeletalMeshComponent> Mesh;

	/** Movement component used for movement logic in various movement modes (walking, falling, etc), containing relevant settings and functions to control movement. */
	UPROPERTY(Category=Character, VisibleAnywhere, BlueprintReadOnly, meta=(AllowPrivateAccess = "true"))
	TObjectPtr<UCharacterMovementComponent> CharacterMovement;

	/** The CapsuleComponent being used for movement collision (by CharacterMovement). Always treated as being vertically aligned in simple collision check functions. */
	UPROPERTY(Category=Character, VisibleAnywhere, BlueprintReadOnly, meta=(AllowPrivateAccess = "true"))
	TObjectPtr<UCapsuleComponent> CapsuleComponent;

	/** Component shown in the editor only to indicate character facing */
	UPROPERTY()
	TObjectPtr<UArrowComponent> ArrowComponent;
```


# ACharacter

 * Characters are Pawns that have a mesh, collision, and built-in movement logic.
 * They are responsible for all physical interaction between the player or AI and the world, and also implement basic networking and input models.
 * They are designed for a vertically-oriented player representation that can walk, jump, fly, and swim through the world using CharacterMovementComponent.

Character是Pawn的子类，拥有了网格体，碰撞以及内嵌的移动组件。
````c++
UCLASS(config=Game, BlueprintType, meta=(ShortTooltip="A character is a type of Pawn that includes the ability to walk around."), MinimalAPI)
class ACharacter : public APawn
{
	//角色的骨骼网格
	UPROPERTY(Category=Character, VisibleAnywhere, BlueprintReadOnly, meta=(AllowPrivateAccess = "true"))
	TObjectPtr<USkeletalMeshComponent> Mesh;

	//移动组件
	UPROPERTY(Category=Character, VisibleAnywhere, BlueprintReadOnly, meta=(AllowPrivateAccess = "true"))
	TObjectPtr<UCharacterMovementComponent> CharacterMovement;

	//碰撞胶囊
	UPROPERTY(Category=Character, VisibleAnywhere, BlueprintReadOnly, meta=(AllowPrivateAccess = "true"))
	TObjectPtr<UCapsuleComponent> CapsuleComponent;
};
````

对于这三个成员，可以使用以下函数来获得(c++)
```c++
FORCEINLINE class USkeletalMeshComponent* GetMesh() const { return Mesh; }
FORCEINLINE UCharacterMovementComponent* GetCharacterMovement() const { return CharacterMovement; }
FORCEINLINE class UCapsuleComponent* GetCapsuleComponent() const { return CapsuleComponent; }
```

玩家如何控制Character呢？Character提供了函数
```c++
ENGINE_API virtual void PossessedBy(AController* NewController) override;
ENGINE_API virtual void UnPossessed() override;
```

使用`PossessedBy`来控制Character，`UnPossessed`来解除控制


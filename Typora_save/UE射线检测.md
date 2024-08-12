# UE射线检测

有关射线检测的函数/数据结构定义在WorldCollision.h文件中。

## LineTraceSingleByChannel

```c++
bool UWorld::LineTraceSingleByChannel(
				struct FHitResult& OutHit,
				const FVector& Start,
				const FVector& End,
				ECollisionChannel TraceChannel,
				const FCollisionQueryParams& Params, 
				const FCollisionResponseParams& ResponseParam ) const
{
	return FPhysicsInterface::RaycastSingle(this, OutHit, Start, End, TraceChannel, Params, ResponseParam, FCollisionObjectQueryParams::DefaultObjectQueryParam);
}
```

单条射线检测，蓝图中的LineTraceByChannel就是在这个函数基础上封装了一层。

![image-20240720211249828](C:/Users/25377/AppData/Roaming/Typora/typora-user-images/image-20240720211249828.png)

```c++
//LineTraceByChannel函数原型
bool UKismetSystemLibrary::LineTraceSingle(
    const UObject* WorldContextObject, 
    const FVector Start, 
    const FVector End, 
    ETraceTypeQuery TraceChannel, 
    bool bTraceComplex, 
    const TArray<AActor*>& ActorsToIgnore, 
    EDrawDebugTrace::Type DrawDebugType, 
    FHitResult& OutHit, 
    bool bIgnoreSelf, 
    FLinearColor TraceColor, 
    FLinearColor TraceHitColor, 
    float DrawTime)
```

LineTraceSingleByChannel使用的是FPhysicsInterface提供的函数接口。FPhysicsInterface是ue中用于抽象物理引擎接口的类，他提供一组函数，用于与底层的物理引擎交互。在物理引擎(Chaos引擎)方面做射线检测主要步骤

1. **空间分区**：Chaos 引擎使用空间分区技术（如 BVH，Bounding Volume Hierarchy）来快速缩小可能与射线相交的物体范围。
2. **包围盒测试**：对每个可能的物体，首先进行包围盒（Bounding Box）测试，检查射线是否与物体的包围盒相交。
3. **精确碰撞检测**：如果包围盒测试通过，则进行更精确的碰撞检测，检查射线是否与物体的几何形状（如多边形网格、胶囊体）相交。

在cpp中调用射线检测
```c++
GetWorld()->LineTraceSingleByChannel(...);
```




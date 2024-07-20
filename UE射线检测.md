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


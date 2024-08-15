# UE反射



## 反射包裹

在ObjectMacros.h文件中定义了UPROPERTY,UFUNCTION等反射包裹体。

在710行可以看到这些宏的定义

### namespace UP

定义了UPROPERTY的关键字。例如：Category，EditAnywhere，EditInstanceOnly。
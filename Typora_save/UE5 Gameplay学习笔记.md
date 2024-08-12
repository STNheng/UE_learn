# UE5 Gameplay学习笔记

## 程序的入口

Windows平台的程序入口地址在LaunchWindows.cpp文件中。
``` c++
int32 WINAPI WinMain(_In_ HINSTANCE hInInstance, _In_opt_ HINSTANCE hPrevInstance, _In_ char* pCmdLine, _In_ int32 nCmdShow)
{
	int32 Result = LaunchWindowsStartup(hInInstance, hPrevInstance, pCmdLine, nCmdShow, nullptr);
	LaunchWindowsShutdown();
	return Result;
}
```

* hInInstance:当前实例的句柄，即程序的唯一标识符。
* hPreInstance:前一个实例的句柄，已不再使用
* pCmdLine:命令行参数，表示程序启动时的输入参数
* nCmdShow:窗口显示的方式，可以指定为最大化，最小化，隐藏等等。

接下来进入到LaunchWindowsStartup()函数中，这个函数会做一些Windows环境设置，命令行参数的处理以及异常的判断。处理完成后进入到
``` c++
ErrorLevel=GuardedMain(cmdLine);
```

在GuardedMain()函数中，找到EngineTick()。
``` c++
if (!GUELibraryOverrideSettings.bIsEmbedded)
{
	while( !IsEngineExitRequested() )
	{
		EngineTick();
	}
}
```

EngineTick()函数中直接调用了GEngineLoop.Tick()。
``` c++
LAUNCH_API void EngineTick( void )
{
	GEngineLoop.Tick();
}
```

在GEngineLoo.Tick()函数中找到引擎的Tick函数调用。
``` c++
// main game engine tick (world, game objects, etc.)
GEngine->Tick(FApp::GetDeltaTime(), bIdleMode);
```

在这个Tick函数中，会调用父类的Tick函数。

## 世界的BeginPlay

经过一系列的处理，引擎会创建一个游戏实例(UGameInstance)，在PIE(Play In Editor)模式下。会调用UGameInstance的StartPlayInEditorGameInstance()函数。在这个函数中调用了这个游戏实例包含的PlayWorld(世界)的BeginPlay()函数。
``` c++
void UWorld::BeginPlay()
{
    //获取所有的UWorldSubsystem实例
	const TArray<UWorldSubsystem*>& WorldSubsystems = SubsystemCollection.GetSubsystemArray<UWorldSubsystem>(UWorldSubsystem::StaticClass());
	//这个条件检查是否支持创建可见的事务请求，并且网络模式是专用服务器模式或监听服务器模式。如果是，那么就生成一个服务器流级别可见性的实例。
	if (SupportsMakingVisibleTransactionRequests() && (IsNetMode(NM_DedicatedServer) || IsNetMode(NM_ListenServer)))
	{
		ServerStreamingLevelsVisibility = AServerStreamingLevelsVisibility::SpawnServerActor(this);
	}

#if WITH_EDITOR
	// Gives a chance to any assets being used for PIE/game to complete
	FAssetCompilingManager::Get().ProcessAsyncTasks();
#endif
	//遍历所有的UWorldSubsystem实例，并调用它们的OnWorldBeginPlay()函数
	for (UWorldSubsystem* WorldSubsystem : WorldSubsystems)
	{
		WorldSubsystem->OnWorldBeginPlay(*this);
	}
	//获取这个世界的游戏模式(GameMode)，并调用它的StartPlay方法，如果存在AI系统那么就调用AI系统的StartPlay()
	AGameModeBase* const GameMode = GetAuthGameMode();
	if (GameMode)
	{
		GameMode->StartPlay();
		if (GetAISystem())
		{
			GetAISystem()->StartPlay();
		}
	}
	//通知所有监听这个事件的对象：世界已经开始播放。
	OnWorldBeginPlay.Broadcast();
	//检查是否存在物理场景，如果存在就调用场景的OnWorldGeinPlay()函数
	if(PhysicsScene)
	{
		PhysicsScene->OnWorldBeginPlay();
	}
}
```

## 游戏模式的StartPlay

游戏模式(GameMode)的StartPlay()只做了一件事：调用游戏状态(GameState)的HandleBeginPlay()。
``` c++
void AGameModeBase::StartPlay()
{
	GameState->HandleBeginPlay();
}
```

## 游戏状态的HandleBeginPlay

主要调用世界设置(WorldSettings)的NotifyBeginPlay()

``` c++
void AGameStateBase::HandleBeginPlay()
{	
    //表示游戏已经开始播放，通常用于网络同步
	bReplicatedHasBegunPlay = true;
    //获取世界的设置，通知游戏已经开始
	GetWorldSettings()->NotifyBeginPlay();
	GetWorldSettings()->NotifyMatchStarted();
}
```

## 世界设置的NotifyBeginPlay

在这里会调用世界里所有演员(Actor)的BeginPlay函数。
``` c++
void AWorldSettings::NotifyBeginPlay()
{
    //获取当前的世界实例
	UWorld* World = GetWorld();
    //检查世界是否开始播放，如果没有则执行一下操作。
	if (!World->GetBegunPlay())
	{
        //遍历所有的演员，对于每一个演员通知它进行BeginPlay()处理。游戏开始了，在开始前你有没有想做的？初始化血量，加载属性等等。
		for (FActorIterator It(World); It; ++It)
		{
			SCOPE_CYCLE_COUNTER(STAT_ActorBeginPlay);
			const bool bFromLevelLoad = true;
			It->DispatchBeginPlay(bFromLevelLoad);
		}
		//设置世界开始播放，这就意味着BeginPlay()只会执行一次
		World->SetBegunPlay(true);
	}
}
```

这里的It是演员Actor的“实际类型”，比如在UE5自带的第三人称模板游戏中(例如创建一个第三人称模板游戏叫Test)，那么就会有一个TestCharacter类继承自AActor(非直接继承)。这在下面一步中很关键。

## AActor的DispatchBeginPlay

这个函数中显式调用了BeginPlay()函数。而BeginPlay()是在AActor中被声明为虚函数。这就会发生多态，根据对象实际类型调用相应的函数。所以调用的是TestCharacter中的BeginPlay()函数。
AActor中的BeginPlay()函数声明：

``` c++
/** Overridable native event for when play begins for this actor. */
ENGINE_API virtual void BeginPlay();
```

TestCharacter中的BeginPlay()函数声明：
``` c++
virtual void BeginPlay();
```

至此，从启动编辑器到一个具体Actor的BeginPlay()流程就结束了。


# GameCore

## HeroControler

### Born(const PooledHandled\<SGCore::ActorRoot>& owner)

HeroContoler:英雄的控制器，在加载对局的时候创建，有几位玩家创建几个。英雄死亡后不销毁。

### OnDead()

在英雄死亡时调用。先判断是否能复活(复活甲/)，不能复活的话清除自己的连杀记录，连死记录+1，然后清除一些缓存，接着设置最大复活时间

```c++
 void HeroControler::UpdateMaxLiveTime()
 {
     UInt32 CurTime = SGameFrameSynchr->GetCurFrameTick();
     UInt32 CurDelta = CurTime >= LastReviveFrameTick ? CurTime - LastReviveFrameTick : 0;

     MaxLiveTime = SMax(CurDelta, MaxLiveTime);
 }
```

MaxLiveTime在HeroControler的构造函数中被初始化为0。考虑实际场景：当英雄在第5s第一次死亡时，CurTime = 5，CurDelta = 5，那么MaxLiveTime = 5，也就是5秒复活。当第20s又一次死亡时，MaxLiveTime = 10

### OnRevive()

在英雄复活时调用，复活甲复活时也调用

### CmdCommonLearnSkill()

升级/学习技能时调用

### CmdMoveDirection()

移动时调用，位移技能不算



## MonsterControler

小兵，野怪的控制器

### MonsterControler()

时机：开局第9s-10s，生成小兵前；当小兵在线上遇见然后野怪生成，也调用这个函数。弩车生成时也会构造，米莱迪召唤机器人也会构造(只会触发一次)，这个函数调用Construct()函数完成构造

当小兵/野怪阵亡时也会调用Construct()
UObject $\rightarrow$ Actor $\rightarrow$ Component

Level $\rightarrow$ World $\rightarrow$ WorldContext $\rightarrow$ GameInstance $\rightarrow$ Engine

Pawn $\rightarrow$ Controller $\rightarrow$ PlayerState $\rightarrow$ Player

World $\rightarrow$ GameMode $\rightarrow$ GameState

Object是基础提供了GC反射序列化等功能，Actor继承自Object具有基础的生命周期和网络复制，Component继承自Actor通过组合的方式实现各类功能且带有transform

多个Level组成了World，WorldContext指向了当前的World，并在GameInstance存储了全局信息，最后由Engine控制

MVC模式(M-PlayerState V-Pawn C-Controller)，Pawn是玩家控制的表现部分(常见的Character类衍射于此包含移动组件、碰撞胶囊体、骨骼网格体)，Controller是玩家控制逻辑部分通常用于实现一些可替换逻辑或者智能决策

GameMode作为WorldController控制游戏层面逻辑上的处理，用GameState存储GaeMode运行过程中产生的数据



#### UObject 

一切的根基包含了元数据、反射生成、GC垃圾回收、序列化、编辑器可见



#### Actor

具有网络复制(Replication)、生死(Spawn)、心跳(Tick)，对应Unity Prefab

（1）Actor不带有Transform

- 它并不只是3D中的表示一些不在世界里展示的不可见对象也是actor，例如AInfo(相关各种派生类)。但其实大部分Actor都是有Transform的但内部实现一般都转发至RootComponent，而Actor接收处理Input事件的能力也是转发到内部的UInputComponent



#### Component

（1）Actor与Component关系

- TSet<UActorComponent*> OwnedComponents保存着这个Actor所拥有的所有Component，一般会有个SceneComponent作为RootComponent
- TArray<UActorComponent*> InstanceComponents保存着实例化的Components，就是Actor被实例化之后附属的Component也会被实例化，相关预分类还有ReplicatedComponents
- Actor若想被放进场景里需要实例化 USceneComponent* RootComponent

（2）Component家族

- 为什么只提供USceneComponent一级嵌套不提供Actor嵌套：UE更偏向于编写功能单一的Component而不是整合了其他Component的大管家Component，而且一般游戏逻辑也不写在Component里面不会实现一个很复杂的Component

```
UActorComponent
	USceneComponent(Transform + 互相嵌套)
		UPrimitiveComponent
			UMeshComponent
				UStaticMeshComponent
		UChildActorComponent
```

（3）Actor间父子关系确定：Actor之间父子关系确定：通过Component确定，Child::AttachToActor或Child::AttachToComponent，从而attach到具有transform的SceneComponent上



#### Level

```
AActor
ULevel->ALevelScript
	AInfo
	AWorldSettings
```

（1）基本类别：

- ALevelScript 允许在关卡中编辑脚本获取所有actor

- AInfo 记录本level的各种规则，AWorldSettings
  - AWorldSettings通常放在Actors[0]的位置，非网络Actor排在前面网络可复制的Actor排在后面然后加一个起始索引标记iFirstNetRelevantActor



#### World

一个worl里面有多个level，每个World支持一个PersistenLevel和多个其他Level，运行时CurrentLevel指向PersistentLevel



#### WorldContext

（1）World不是只有一种类型

```c++
namespace EWorldType
{
	enum Type
	{
		None,		// An untyped world, in most cases this will be the vestigial worlds of streamed in sub-levels
		Game,		// The game world
		Editor,		// A world being edited in the editor
		PIE,		// A Play In Editor world
		Preview,	// A preview world for an editor tool
		Inactive	// An editor world that was loaded but not currently being edited in the level editor
	};
}
```

（2）FWorldContext保存着ThisCurrentWold来指向当前的World，而在World切换时就会保存上下文信息(World和Level的信息)。对于独立运行的游戏WorldContext只有一个，而编辑器模式中一个WorldContext给编辑器一个给PIE

- Level切换信息为什么不放在world李，因为world只有一个persistentLevel，打开它时会先释放掉当前World然后创建一个新的World所以如果存在World中就需要拷贝回来一遍



#### GameInstance

保存着当前WorldContext和其他整个游戏的信息，不管怎么切换都一直存在的那个变量



#### Engine

（1）UE不支持同时运行多个World所以GameInstance也是唯一的

```c++
UEngine
    UEditorEngine
    UGameEngine
    	UGameInstance
```



#### GamePlayStatic

这个类相当于C++的静态类，只为蓝图提供了一些静态方法，在想借鉴或者查询某个功能实现时会是一个入口



#### Pawn

（1）会响应玩家(或者AI)交互的Actor我们称为Pawn，他包含了以下三个部分：

- 可被Controller控制
- PhysicsCollision表示
- MovementInput基本响应接口

（2）Pawn的注重点在于更清楚地去表示，而不是作为逻辑的载体，提线木偶提线的是Controller木偶是Pawn

- Actor可以响应WASD的按键事件，但是按键之后呢该如何移动Pawn定义了一个基本的MovementInput套路相当于把高层的输入响应往前封装了一步

（3）类别

- DefaultPawn：默认带了DefaultPawnMovementComponent、SphericalCollisionComponent、StaticMeshComponent
- SpectatorPawn：提供了基本的USpectatorPawnMovement不带重力漫游
- Character：像人一样行走的CharacterMovementComponent，尽量贴合的CapsuleComponent，骨骼蒙皮

```
APawn
    ADefaultPawn
    	ASpectatorPawn
    ACharacter
```



#### AController

（1）基本功能：

- 在Actor中增加关联Pawn的能力Posse/UnPossess，PawnPendingDestory
- 继承于Actor中也就有了EnableInput和Tick
- 带一个SceneComponent可以摆放在世界中
- 自身FName StateName可以切换自身状态

（2）哪些逻辑需要写在Controller里面(因为Pawn也可以接收用户输入事件所以只要愿意甚至可以脱离Controller做一个特例独行的Pawn但这样不符合设计时想的规范)

- Pawn本身表现是固有的能力逻辑，而一些可替换逻辑或者智能角色就应该归Controller管辖
- 如果一个逻辑只属于一类pawn那么可以放入pawn，如果一个逻辑属于多个pawn可以放入controller
- controller生命周期更长可以存放在pawn之外需要处理的逻辑



#### APlayerController

（1）大致模块：

- Camera管理：控制玩家的视角，PlayerCameraManager这一个关联很紧密的摄像机管理类
- Input系统：构建InputStack用来路由输入事件
- UPlayer关联，一个UPlayer可以是本地的LocalPlayer也可以是一个网络控制的UNetConnection
- Level切换，在进行Level切换时也都是先通过PlayerController来进行RPC调用然后由PlayerController来转发到自己World中来实际进行
- Voice，方便网络中语音聊天的一些控制函数



#### AAIController

（1）基本AI组件：

- Navigation：用于智能导航寻路，常用的MoveTo接口
- AI组件：运行启动行为树使用黑板数据探索周围环境
- Task系统：让AI去完成一些任务



#### APlayerState

（1）继承自AInfo，纯数据部分，记录了玩家的一些游戏状态。还可以在网络波动断线的时候缓存玩家状态

（2）存储的实际上是当前关卡玩家相关数据



#### GameMode

（1）继承自AInfo可以理解为WorldController

（2）功能模块：

- Class登记：需要的时候通过UClass的反射自动Spawn出相应的对象添加进关卡
- 游戏内实体Spawn：玩家进入游戏Login后生成在什么位置的实体管理工作，也控制着游戏支持的玩家、旁观者和AI实体的数目
- 游戏进度：SetPause、RestartPlayer等函数
- Level/World切换
- 多人游戏的步调同步：使用一个状态机来标记开始和结束的状态

（3）哪些逻辑应该写在GameMode，哪些应该写在LevelBlueprint

- GameMode应该专注于逻辑的视线，LevelScriptActor应该专注于本Level的表示逻辑
- GameMode可以应用于不同的Level搜易通用的玩家应该放在GameMode里面
- GameMode只在Server存在，所以不要在其中实现Client逻辑(放在LevelScript中)
- GameMode的逻辑属于游戏，PlayerController的逻辑属于玩家



#### GameState

类似于APlayerState，用于保存当前游戏的状态数据(比如任务数据等)



#### UPlayer

（1）继承自UObject，因为Actor必须在World中才能存在Player不需要被摆放在Level中也不需要各种Component组装，和一个PlayerController关联起来

（2）LocalPlayer类：GameInstance里保存有LocalPlayer列表

（3）UNetConnection类：一个网络连接也是一个Player



#### GameInstance

（1）游戏引擎的管理类，通常包括：

- 引擎的初始化加载，Init和ShutDown
- Player创建，如CreateLocalPlayer, GetLocalPlayers
- GameMode重载修改
- OnlineSession管理：一个网络会话的管理辅助控制类

（2）Engine站在更高的层次上管理协调多个Game



#### SaveGame

（1）UE存档功能，GameInstance里面存放的是临时数据，SaveGame里面是持久数据


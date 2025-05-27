# Actor同步机制

---

> 以下内容基于UE4

"`bReplicates = true` 是一切网络同步的**前提条件**"

如果我们要生成一个可同步的`Actor`，唯一的方法就是在服务端处构建这个`Actor`对象，并把这个`Actor`对象设置为`bReplicates = true`，它表示：

**“这个 Actor 允许被网络系统复制到客户端”。**

- 如果你没设置 `bReplicates = true`，那这个 `Actor` 完全不会出现在客户端。
- **即使你写了属性同步代码，也不会生效**，因为根本没被复制过去。
- 在客户端生成的`Actor`，无论是否写了`bReplicates`，这个对象都不会被网络复制，只会在客户端本地出现

---



## 一、Server To Client

>  🛤 1.同步方式

在服务端生成 `Actor`对象，并令其 `bReplicates = true` ，会让这个对象在服务端处被构造的时候，在与服务端连接的**所有客户端**处生成一份相同的对象，但是如果后续还需要对这个 `Actor`对象进行更多的同步（如位置的变更同步、某个成员属性的变更同步等等），则还需要明确告诉引擎要**同步什么内容**：

| 你想同步的内容               | 所需设置                                                   |
| ---------------------------- | ---------------------------------------------------------- |
| **位置、旋转、缩放**         | `SetReplicateMovement(true)`                               |
| **变量值（如血量、速度等）** | `UPROPERTY(Replicated)` + `DOREPLIFETIME()`                |
| **变量值 + 回调函数**        | `UPROPERTY(ReplicatedUsing=OnRep_XXX)` + `DOREPLIFETIME()` |
| **指定客户端调用 RPC**       | `UFUNCTION(Client, Reliable)`                              |
| **指定服务端调用 RPC**       | `UFUNCTION(Server, Reliable)`                              |

以上这些方法都是建立在`bReplicates = true` 的基础上

<br>

> 📌 2.举个小例子

```clike
// 1. 在构造函数中设置复制开关
AMyActor::AMyActor()
{
    bReplicates = true;
    SetReplicateMovement(true);
    health = 100.f;
}

// 2. 声明一个可同步的变量
UPROPERTY(Replicated)
float health;

// 3. 注册同步变量
void AMyActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME(AMyActor, health);
}
```

这样服务器修改 `health`，客户端会自动跟着同步。

<br>

**总结：**

> `bReplicates = true` 是开启网络同步的门票，但你得“明确告诉它”要同步什么，才会真正发生同步。

后续的同步行为则完全取决于你是否启用了 `SetReplicateMovement()`、声明 `Replicated` 变量等机制。

---



## 二、Client To Server

在 UE4 网络模型中，**客户端并不能直接修改服务器上的状态**，它必须通过某些明确的“通道”来实现。而这些通道主要是通过 **RPC（远程过程调用）** 实现的。

> ✔1.前提条件

- **Actor 必须是服务器产生且 `bReplicates = true`**。否则不会在客户端生成副本，也就谈不上同步。
- 客户端想要“控制”某个 Actor，必须要**拥有这个 Actor 的控制权**（ownership）。即客户端能**调用 RPC 的前提是它拥有这个 Actor**（或者拥有控制它的 PlayerController）。
  - 控制权由服务器通过 `SetOwner(PlayerController)` 或 `Possess()` 显式分配。
- RPC 方法必须加上 UFUNCTION 宏并声明为某种调用类型（ Multicast/Client/Server ）。

<br>

> 🎲 2.RPC调用的方式

**通过 UFUNCTION(Server, Reliable）声明一个RPC函数和调用类型**

```clike
//在actor的类中编写一个函数，加上UFUNCTION宏
UFUNCTION(Server, Reliable)
void ServerChangeHealth(float NewHealth);
```

**调用时仍然在客户端调用：**

```clike
ServerChangeHealth(NewHealth); // 客户端调用
```

但此时实际是由客户端向服务器发起一个“请求”，要求服务器执行这个函数。这个请求是否生效取决于“控制权”。

<br>

> 🎈 3.权限与控制权（重点）

权限与控制权是RPC同步的核心问题：

因为一个可同步的 `Actor` 对象是由服务器生成的，然后复制到多个客户端中，也就是说此时假设有N个客户端，1个服务器，这时候一共有 **N+1** 个该 `Actor` 对象（1个真实对象和N个本地对象）

- **即使一个客户端上存在某个 Actor（本地对象），它也不一定能向服务器发 RPC 请求更改它的状态。**
- 服务器只接受来自**拥有该 Actor 的客户端的 RPC 请求**。
- 判断标准就是 `Actor->GetOwner()` 最终是否指向了当前客户端的 `PlayerController`。

举个例子：

|       | Server             | Client 1      | Client 2 |
| ----- | ------------------ | ------------- | -------- |
| Actor | ✅ 实体（真实对象） | ✅ 副本+控制权 | ✅ 副本   |

只有 **Client 1** 可以向服务器发起 `Server_ChangeState()` 类型的 RPC 并被服务器接受。

**Client 2 即使调用了该函数，服务器也会拒绝处理（验证失败）。**

<br>

**总结：**

客户端只有拥有某个 `Actor` 的控制权，才能通过 `RPC` 向服务器请求修改其状态，而最终状态是否改变由服务器决定。

---

## 三、Actor Relevancy

想象一下，如果游戏中的关卡/地图足够大，那么地图远处的某些Actor对于玩家来说可能被视为“不重要”。这时候就不需要同步这些Actor。

>  Actor Relevancy会决定Server是否应该把某些bReplicates的Actor同步给Client（同步的所有基础都是bReplicates==true）

1.AActor类有一个函数叫`IsNetRelevantFor()`，当Server要同步一个Actor给Client，它会先调用这个Actor的`IsNetRelevantFor()`函数，如果返回true那么才会进行同步

2.AActor::IsNetRelevantFor()有默认实现，即对于一个最普通的Actor，它的Actor Relevancy机制为：

```
1.如果 
1）Actor 被标记为 bAlwaysRelevant，或者：
2）Actor 归属于该玩家的 Pawn 或 PlayerController，
3）Actor 就是该玩家的 Pawn，
4）该玩家的 Pawn 是某些行为（比如声音或伤害）的发起者（Instigator），
那么这个 Actor 就是“Relevant”的。

2.如果 Actor 被标记为 bNetUserOwnerRelevancy 并且有 Owner，
就根据 Owner 的相关性来判断该 Actor 是否 Relevant。

3.如果 Actor 被标记为 bOnlyRelevantToOwner，但是没有通过第 1 条检查，
则该 Actor 对当前玩家是非Relevant的。

4.如果 Actor 附着在另一个 Actor 的骨骼（Skeleton）上，
则根据其被附着的那个基础 Actor 的Relevant来判断。

5.如果 Actor 被隐藏（bHidden == true）且根组件没有碰撞，
则该 Actor 对当前玩家是非Relevant的。

6.如果 Actor 没有根组件，
AActor::IsNetRelevantFor() 会发出警告，并提示是否应该把 Actor 设置成 bAlwaysRelevant = true。

7.如果游戏中启用了基于距离的网络相关性管理（由 AGameNetworkManager 负责），
则当 Actor 距离玩家足够近（在网络剔除距离内）时，才算是Relevant。
```

`Pawn` 和 `PlayerController` 重写了 `IsNetRelevantFor()` 函数，所以它们判断相关性的条件会有所不同。

3.如果我们想自定义一个Actor的Relevant判断，只需要重写`IsNetRelevantFor()`函数即可

4.**Actor复制（Replication）机制** 的核心流程

```
[Server 每帧]
|
|-- 对每个 Actor 检查 bReplicates
|     |如果 bReplicates==true
|     |检查实
|     |-- 对每个 Client:
|           |-- 是否到达下次可同步的时间点（由AActor::NetUpdateFrequency决定一个Actor的同步频率）
|           |  (没有到下一次同步时间点或者到了但是Actor距离上次值无更新，则会跳过该Actor的同步)
|           |-- 调用 IsNetRelevantFor(该 Client 的 ViewPawn)
|           |-- 如果返回 true:
|                → 将该 Actor 进入 Replication 队列
|                → 执行变量复制（根据条件，如 COND_OwnerOnly）
|                → 客户端更新、调用 OnRep
|           |-- 否则：不做任何事，客户端也不知道这个 Actor 存在；
|                    如果Actor前一帧已存在于该客户端，客户端会删除该Actor；
```

---

## 四、Actor Priority

在 UE4 的网络同步中，服务器为每个客户端维护一个待同步的 **Replicator 列表**。当列表太长、带宽不够时，需要决定“谁先同步”，这就靠 `Priority`。

> 所以 Priority 只针对要进行同步的Actor（bRep==true && IsNetRelevantFor()==true）

1.什么时候会计算 Priority？

**只有在以下前提都满足时才会计算 Priority：**

1. 该 Actor 的 `bReplicates = true`
2. `IsNetRelevantFor()` 返回 `true` → 是「Relevant」的 Actor
3. Actor 已加入当前客户端的 Replication 队列
4. 此时系统需要决定：**哪些 Actor 本帧应该发送数据（带宽允许），哪些暂时跳过**

2.可以重写 `AActor::GetNetPriority()` 来自定义某些 Actor 的同步优先级
# Controller 与 Pawn 的绑定与生成

如果我们创建一个空世界，并运行游戏，此时**GameMode（仅在服务器生成）、GameState、PlayerState和PlayerController**都是会生成的。在游戏里面，**PlayerController**一般会控制我们的角色，这个过程需要我们手动的将一个**Pawn**和**PlayerController**进行关联后才可以实现。

---

## 1.空世界（没有指定Pawn）时会发生什么？

- **没有指定 Pawn**，游戏启动时：
  - PlayerController 会生成（客户端和服务器都有）。
  - 但**不会自动生成任何 Pawn**。
- **UE 默认生成的 GameModeBase**，如果没有手动指定 `DefaultPawnClass`，它实际上默认是 `DefaultPawn`（或项目配置的默认Pawn）这个类的一个空壳实例。
  - 这个默认Pawn通常是没有绑定任何可见的 Mesh 或 SkeletalMesh（模型）的，所以**你看不到角色模型**，但Pawn本体确实存在于世界中。
  - 这个默认Pawn自带基础的移动组件和输入绑定，所以：
    - 你可以用鼠标和键盘控制它移动和旋转。
    - 视角通常绑定在Pawn的摄像机上，所以你能控制视角。

---

## 2. 如何指定一个 Pawn 给 Controller？

### 方式 A：在 `GameMode` 设置默认Pawn类

- 在 GameMode 里设置：

  ```clike
  DefaultPawnClass = AMyPawn::StaticClass();
  ```

- 游戏启动时，PlayerController 会自动调用 `SpawnDefaultPawnFor`：

  - 根据 DefaultPawnClass 生成 Pawn。
  - Controller 调用 `Possess` 拥有该 Pawn。

### 方式 B：在地图（关卡）里手动放置Pawn实例，并设置它自动被控制

- 在关卡里拖拽一个 Pawn 蓝图实例到地图上（相当于同时指定了Pawn的出生点）。
- 在该 Pawn 的细节面板设置：
  - `Auto Possess Player = Player0`（自动绑定第一个玩家控制器）
- 运行游戏时，PlayerController 会自动 Possess 这个手动放置的 Pawn。

### 方式 C：运行时手动调用 Controller 的 Possess 绑定一个Pawn实例

- 通过代码获取 Controller，然后调用：

  ```clike
  Controller->Possess(SomePawnInstance);
  ```

- 适合运行时动态切换或生成 Pawn。

---

## 3. 如果指定的是默认Pawn或者某个Pawn类（方式A和C），没有指定出生点，Pawn会生成在哪里？

- 默认情况下，`SpawnDefaultPawnFor` 会尝试在 **PlayerStart**（玩家出生点）上生成 Pawn。
- 如果地图中没有 PlayerStart 或者没有有效的出生点，Pawn 会生成在坐标 `(0,0,0)` 或者其他默认位置（可能导致Pawn卡地面下）。
- 因此，**地图中至少放置一个 PlayerStart（玩家出生点）是推荐做法**（地图上的 PlayerStart 会被扫描到，然后在这个地方生成 DefaultPawn）。

---

## 4. 如何指定Pawn的出生点？

- **方法 1：放置 PlayerStart 组件（蓝图/关卡中）**

  - 地图中放置一个或多个 PlayerStart（蓝图Actor），
  - GameMode 的 `SpawnDefaultPawnFor` 会优先选取对应玩家的 PlayerStart 作为出生点。

- **方法 2：重写 GameMode 的 SpawnDefaultPawnFor**

  - 可以重写函数，自定义生成Pawn的位置：

    ```clike
    APawn* AMyGameMode::SpawnDefaultPawnFor(AController* NewPlayer, AActor* StartSpot)
    {
        // 使用 StartSpot 的位置生成Pawn，也可以自定义坐标
        return Super::SpawnDefaultPawnFor(NewPlayer, StartSpot);
    }
    ```

- **方法 3：手动生成Pawn时，调用SpawnActor时传入指定坐标**

  - 例如在代码中：

    ```clike
    FVector SpawnLocation = FVector(100, 100, 300);
    FRotator SpawnRotation = FRotator::ZeroRotator;
    APawn* MyPawn = GetWorld()->SpawnActor<APawn>(MyPawnClass, SpawnLocation, SpawnRotation);
    Controller->Possess(MyPawn);
    ```
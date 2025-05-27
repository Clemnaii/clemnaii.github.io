# Rotation的关联

在 Unreal Engine（UE）中，**Rotation** 代表的是一个物体的**旋转状态**，通常用于描述一个对象的朝向或视角。

```clike
struct FRotator
{
    float Pitch; // 上下俯仰 (绕X轴)
    float Yaw;   // 左右转头 (绕Z轴)
    float Roll;  // 翻滚 (绕Y轴)
};
```

**Rotation** 可以转换为一个 **向量**，这个**向量**是一个“指向 **Rotation** 朝向的方向的**向量**”

> 在一个游戏中，常常会有几个 Rotation 容易混淆，它们分别是
>
> 1.Controller Rotation
>
> 2.Camera Rotation
>
> 3.Pawn Rotation

---

## 1.Controller Rotation

Controller（即游戏的 PlayerController ）管理着玩家的输入和视角旋转。

**“它的 Rotation 代表玩家“看向”的方向，也就是玩家视角的方向。”**

> **PlayerController 的 Rotation 并不会自动随鼠标动**

想让鼠标驱动 Controller 的 Rotation，需要：

1. 让  Controller 绑定一个 Pawn 或 Character
2. 在 Pawn 或 Character 的 `SetupPlayerInputComponent` 中调用 `AddControllerYawInput()` 和 `AddControllerPitchInput()`。
3. 你在编辑器绑定了鼠标X和Y的输入
   - 游戏运行后：PlayerController 会把鼠标X和Y输入分发给它当前控制（Possess）的 Pawn 或 Character

---

## 2.Camera Rotation

Camera 通常是 Pawn 的子组件，比如 `UCameraComponent`，或者附着在 `USpringArmComponent`（控制臂）上。在游戏里，我们通常会让 Camera Rotation 直接跟随 Controller Rotation，使得摄像机Rotation的方向就是Controller的Rotation方向，也就是鼠标移动的方向。

> 通过调用xxxComp->bUsePawnControlRotation=true，来使得这个xxxComp的Rotation跟随其附着的Pawn的Controller的Rotation

这种让Camera Rotation 直接跟随 Controller Rotation 的方式在**第一/三人称游戏**里面通过不同的方式去实现：

**第一人称**：

- 摄像机 **直接绑定在 Pawn 根节点上**。
- `CameraComp->bUsePawnControlRotation = true`（让摄像机和视角一致）
- 没有 SpringArm。

**第三人称**：

- 摄像机 **绑定在 Pawn 上的 SpringArm 上**。
- `SpringArmComp->bUsePawnControlRotation = true`
- `CameraComp->SetupAttachment(SpringArmComp);`
- `CameraComp->bUsePawnControlRotation = false`

---

## 3.Pawn Rotation

Pawn 是游戏中被控制的角色或对象。

所以它的 Rotation 从表明上看就很好理解，就是就是 Pawn角色模型 的朝向，即身体的转向。

> 所以Pawn Rotation 并不一定和 Controller Rotation 一致

---

## 4.第三人称游戏视角控制

在很多第三人称游戏里面，一般的原则是：

- 键盘控制人物的Movement和Rotation，
- 鼠标控制摄像机的Rotaion（即也是Controller的Rotation）

**1）一种常见于第三人称游戏的视角和控制逻辑是：**

1. Camera附着在SpringArm上，SpringArm的Rotaion关联于Controller的Rotation
2. 鼠标控制Controller的Rotation，因此最终“鼠标控制Camera的Rotation”
3. 键盘控制人物的Movement，即前进后退左移右移
4. 但是人物前进的方向指向“Camera的Rotation”方向（如按W进行前进时候不朝Pawn的Rotation而是朝Camera的Rotation
5. 人物的Rotation跟随人物的Movement方向，即人物会自动向Movement的方向转身，始终保证脸朝着Movement的方向去移动（而不是像螃蟹一样横向移动

**2）实现方法：**

1.Camera 附着在 SpringArm 上

```clike
SpringArmComp = CreateDefaultSubobject<USpringArmComponent>(TEXT("SpringArm"));
SpringArmComp->SetupAttachment(RootComponent);
SpringArmComp->bUsePawnControlRotation = true; // 让控制臂旋转跟随 Controller

CameraComp = CreateDefaultSubobject<UCameraComponent>(TEXT("Camera"));
CameraComp->SetupAttachment(SpringArmComp);
CameraComp->bUsePawnControlRotation = false; // 相机不单独旋转，由 SpringArm 带动
```

> 📌 所以：**鼠标控制的是 Controller Rotation → 带动 SpringArm 旋转 → Camera 旋转**

2.按键控制角色移动

在 `SetupPlayerInputComponent()` 中绑定：

```clike
PlayerInputComponent->BindAxis("MoveForward", this, &AMyCharacter::MoveForward);
PlayerInputComponent->BindAxis("MoveRight", this, &AMyCharacter::MoveRight);

PlayerInputComponent->BindAxis("Turn", this, &APawn::AddControllerYawInput);
PlayerInputComponent->BindAxis("LookUp", this, &APawn::AddControllerPitchInput);
```

3.MoveForward/MoveRight 实现

```clike
void AMyCharacter::MoveForward(float Value)
{
    if (Controller && Value != 0.0f)
    {
        // 获取控制器的朝向（忽略 Pitch 和 Roll）
        FRotator YawRotation(0.f, Controller->GetControlRotation().Yaw, 0.f);
        FVector Direction = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::X); // 前方向
        AddMovementInput(Direction, Value);
    }
}

void AMyCharacter::MoveRight(float Value)
{
    if (Controller && Value != 0.0f)
    {
        FRotator YawRotation(0.f, Controller->GetControlRotation().Yaw, 0.f);
        FVector Direction = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::Y); // 右方向
        AddMovementInput(Direction, Value);
    }
}
```

> 🎯 **这一步是关键：角色不再朝自己朝向移动，而是朝 Camera 方向移动！**

4.角色自动面向移动方向

使用 CharacterMovementComponent 的这个设置：

```clike
GetCharacterMovement()->bOrientRotationToMovement = true;  // Pawn自动朝向Movement移动方向
bUseControllerRotationYaw = false;       // 不使用控制器 Rotation 控制Pawn转向
```

---

## 5.第一人称游戏视角控制

**1）一种常见于第一人称游戏的视角和控制逻辑是：**

1. **Camera** 放在角色内部、代表“眼睛位置”
2. **角色朝向 = 相机朝向 = Controller Rotation**
3. 鼠标直接控制 Controller Rotation
4. 移动方向依赖 Controller Rotation，视角即方向

**2）实现方法：**

1.设置 Camera 在角色内部

```clike
// 在角色构造函数中（假设是 AMyCharacter）

CameraComp = CreateDefaultSubobject<UCameraComponent>(TEXT("FirstPersonCamera"));
CameraComp->SetupAttachment(GetCapsuleComponent()); // 直接附着在胶囊体或 Mesh 上
CameraComp->SetRelativeLocation(FVector(0.f, 0.f, 64.f)); // 高度可调
CameraComp->bUsePawnControlRotation = true; // 相机直接跟随 Controller 旋转
```

> ✅ 相机就是“眼睛”，直接跟随 Controller 旋转（bUsePawnControlRotation = true）

2.控制器设置与输入绑定

```clike
PlayerInputComponent->BindAxis("MoveForward", this, &AMyCharacter::MoveForward);
PlayerInputComponent->BindAxis("MoveRight", this, &AMyCharacter::MoveRight);
PlayerInputComponent->BindAxis("Turn", this, &APawn::AddControllerYawInput);
PlayerInputComponent->BindAxis("LookUp", this, &APawn::AddControllerPitchInput);
```

3.移动函数实现（以视角为准）

```clike
void AMyCharacter::MoveForward(float Value)
{
    if (Controller && Value != 0.0f)
    {
        const FRotator Rotation = Controller->GetControlRotation();
        const FVector Direction = FRotationMatrix(Rotation).GetUnitAxis(EAxis::X);
        AddMovementInput(Direction, Value);
    }
}

void AMyCharacter::MoveRight(float Value)
{
    if (Controller && Value != 0.0f)
    {
        const FRotator Rotation = Controller->GetControlRotation();
        const FVector Direction = FRotationMatrix(Rotation).GetUnitAxis(EAxis::Y);
        AddMovementInput(Direction, Value);
    }
}
```

4.控制角色旋转方式

```clike
bUseControllerRotationYaw = true;                          // Pawn方向由控制器决定
GetCharacterMovement()->bOrientRotationToMovement = false; // Pawn不自动朝移动方向转身
```

> ✅ 第一人称通常让角色始终**跟随相机朝向**，不会自动根据移动方向旋转（所以枪战游戏中会出现类似螃蟹般横移的情况
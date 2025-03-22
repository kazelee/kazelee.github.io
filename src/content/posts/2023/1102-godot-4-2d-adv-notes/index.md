---
title: Godot 4 教程笔记：勇者传说
published: 2023-11-02
description: "基于 timothyqiu 的 Godot 4《勇者传说》横版卷轴动作游戏教程，所整理的笔记（截至第 12 节）"
image: "./cover.jpg"
tags: ["Godot", "游戏"]
category: 游戏开发
draft: false
---

教程视频链接：[合集·《勇者传说》Godot 4教程](https://space.bilibili.com/7092/lists/1304862?type=season)

考虑到跟着教程边看边做，容易忽略技术细节和设计思想，故重新浏览一遍教程，并将重要知识点整理成笔记。

注：部分笔记参考了[瓦格良](https://space.bilibili.com/3863194/article)等其他网友的评论和总结

## 00 基础项目

### 0.1 修改窗口大小和拉伸模式

1. 设置「视口大小」为 **「窗口覆盖大小」的 1/3**（把游戏放大 3 倍显示，像素风游戏常用）
2. 修改拉伸模式为 `canvas_items`（拉伸窗口后，画面会跟着放大）
3. 项目 - 项目设置 - 渲染 - 纹理 - 默认纹理过滤：Nearest（保留纹理像素风格，不做线性滤波）

### 0.2 TileMap 的设置

1. 新建 `tilemap` 节点 - 新建 `tileset` - 拖入图片素材（取消自动创建图块弹窗）
2. 设置物理层 0（碰撞）

【快捷键】按住 <kbd>Shift</kbd> 拖出一条直线

### 0.3 玩家场景

【步骤】`sprite2d` - `collision_shape2d` - `animation_player`（关键帧包括：`region_rect`、`hframes`、`frame`）

补充说明：

1. 素材导入的时候需要提前设置「栅格吸附」和「步长」，方便框选所需的部分
2. **只选用素材的一部分，所以需要将 `region_rect` 加入关键帧（因为要获取选中的区域）**
3. 水平框选动画素材（系统不知道实际的帧数），所以需要将 `hframe` 加入关键帧
4. 如使用不同的素材文件，还需要将素材也加入关键帧

### 0.4 玩家脚本

【步骤】编写脚本、设置输入映射、实例化子场景

代码补充说明：

1. `Input.get_axis` 根据玩家的输入方向，返回 `(-1, 0, 1)`
2. `is_zero_approx` 表示与 0 的距离小于内置的判定区间，用于浮点数等于 0 的检测

```gdscript
extends CharacterBody2D
@onready var animation_player: AnimationPlayer = $AnimationPlayer
@onready var sprite_2d: Sprite2D = $Sprite2D

const RUN_SPEED := 200.0
# 是负数的原因是因为在2D空间中y轴向上为负
const JUMP_VELOCITY := -300.0
# 获取引擎给的重力加速度
var gravty := ProjectSettings.get("physics/2d/default_gravity") as float

# 每个物理帧调用一次
func _physics_process(delta: float) -> void:
	# 获取按键输入
	var direction :=  Input.get_axis("move_left","move_right")
	# 修改速度向量
	velocity.x = direction * RUN_SPEED
	velocity.y += gravty * delta

	# 如果在地板上并且按下了jump键，那么就修改角色y坐标变为跳跃值
	if is_on_floor() and Input.is_action_just_pressed("jump"):
		velocity.y = JUMP_VELOCITY

	# 如果在地板上，没有移动则播放idle动画，有移动则播放running动画，如果不在地板上则播放jump动画
	if is_on_floor():
		if is_zero_approx(direction):
			animation_player.play("idle")
		else:
			animation_player.play("running")
	else:
		animation_player.play("jump")

	# 如果在移动，并且是向左移动，那么将角色水平翻转
	if not is_zero_approx(direction):
		sprite_2d.flip_h = direction < 0

	move_and_slide()
```

## 01 相机

### 1.0 编辑器设置

将脚本编辑器中的「补全」，设置为「添加类型提示」：可以提高编辑器性能，使编写更流畅

### 1.1 TileMap 快捷键补充

1. <kbd>Ctrl</kbd> + <kbd>鼠标左键</kbd>：吸取单个图块
2. <kbd>Ctrl</kbd> + 按住 <kbd>鼠标左键</kbd> 拖动：吸取多个图块
3. <kbd>Ctrl</kbd> + <kbd>Shift</kbd> + 按住 <kbd>鼠标左键</kbd> 拖动：绘制矩形区域
4. <kbd>鼠标右键</kbd>：删除图块

### 1.2 相机位置

【步骤】在 `player` 节点下新建相机节点（这一步不在 `player` 场景下设置，**而在 `world` 场景下的 `player` 中设置**）

补充说明：

1. 拖动相机时，按住 <kbd>Ctrl</kbd> 键可以更好的定位（十字辅助线提示）
2. 实际上可以将相机定位在 `player` 场景中，但这样就很难方便的通过 `world` 场景下的 `tilemap` 来控制相机位置；同时也不能很好地预览相机视角下，角色在 `world` 场景下的画面

### 1.3 相机跟随效果

游戏中相机并不总是跟随玩家，玩家在屏幕中心附近有一定的自由活动空间（即玩家走动一段距离后再移动相机）

1. 在 Camera2D 节点的 Drag 属性勾选 Horizontal Enable 和 Vertical Enable （水平和垂直方向上的相机拖动功能）
2. 在 Camera2D 节点的 Editor 属性勾选 Draw Drag Margin，可以观察到可自由活动的范围，通过调整 Drag 属性的 Left Margin 等，可以控制其大小，值是 0 至 1 的比例；
3. 实现相机平滑移动：勾选 Camera2D 节点的 Position Smoothing 下的 Enabled（其中 Speed 可调整相机的平滑移动速度）

### 1.4 限制相机的拍摄范围

1. 可以利用标尺确定位置，然后在相机节点的 Limit 中设置（比较麻烦）
2. 或者使用脚本进行修改（借助 TileMap 的 `size`；需要使用 `reset_smoothing` 结束“出界过渡”的动画）

```gdscript
extends Node2D
@onready var tile_map: TileMap = $TileMap
@onready var camera_2d: Camera2D = $Player/Camera2D

func _ready() -> void:
	# 获取瓦片地图的范围
	var used := tile_map.get_used_rect()
	# 获取单个图块的尺寸
	var tile_size := tile_map.tile_set.tile_size
	# 为相机的上下左右添加限制
	camera_2d.limit_top = used.position.y * tile_size.y
	camera_2d.limit_right = used.end.x * tile_size.x
	camera_2d.limit_bottom = used.end.y * tile_size.y
	camera_2d.limit_left = used.position.x * tile_size.x
	# 将相机的位置立即设置为其当前平滑的目标位置
	camera_2d.reset_smoothing()
```

## 02 TileMap

- TileSet
  - 选择需要的纹理块
  - 对每个纹理块微调（拉伸长宽，更改纹理原点）
  - 绘制：包括地形、生成概率、物理层（碰撞箱）
    - 可以利用绘制功能，批量更改纹理原点等属性
    - 🎲散布：类似绘制的概率，将选定的图案按照 `n:1:1:...` 的概率绘制，n 表示空白
- Terrain 地形
  - 模式：`match corners`（根据角落匹配中心和角落邻接点）
  - 直接涂会出现渲染错误（再涂一遍就好了），建议使用 <kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>鼠标左键</kbd> / <kbd>鼠标右键</kbd>
- TileMap
  - 图层功能（图层的顺序是从后向前，越往后的图层越近）

参考笔记：[如何使用 TileMap｜Godot 4 教程《勇者传说》#2 - 哔哩哔哩 (bilibili.com)](https://www.bilibili.com/read/cv23978148/?jump_opus=1)

补充：对 1.4 中代码的修改（将 `grow()` 更改为 `grow(-1)`）

```gdscript
...
func _ready() -> void:
#	var used := tile_map.get_used_rect()
	# grow(-1) 为什么要向内缩小一格？ 隐藏最外层的边界，给人一种「地图很大」的感觉。
	var used := tile_map.get_used_rect().grow(-1)
    ...
```

## 03 视差背景

提升画面质感的技巧（背景移动速度不同）

- ParallaxBackground/ParallaxLayer
  - 直接拖动素材会放到根节点，按住 <kbd>Ctrl</kbd> 键后拖动会移动到「当前选中节点的子节点」
  - 将素材移动到原点：取消 `offset` 的 `centered`，将 `transform` 的 `position` 重置
  - 原则：`scale` 越小，背景越远；`scale` 越大，背景越近；`1` 是标准的距离
  - `mirroring`：镜像，输入图片的长宽，复制一遍相当于无数遍（自动重复）
- 边运行边修改，可以设置项目-项目设置-显示-窗口-置顶打开
- Bug：画面会出现竖线缝隙（Godot 4.1 已经修复了这个 Bug）
  - 解决 1：更改设置：项目 - 常规 - 渲染 - 2D - 吸附启用（但会像素抖动）
  - 解决 2：将纹理单独切出来保存

【建议】将前后景都选中，作为新 `node2d` 节点的子节点，编组加锁（平常不需要选中）

参考笔记：[如何实现视差背景｜Godot 4 教程《勇者传说》#3 - 哔哩哔哩 (bilibili.com)](https://www.bilibili.com/read/cv24333389/?jump_opus=1)

## 04 运动控制

### 4.1 加速度

对 0.4 中代码的修改：使用 `move_toward` 函数实现加速运动效果

```gdscript
...
const ACCELERATION := RUN_SPEED / 0.2

func _physics_process(delta: float) -> void:
	...
#	velocity.x = direction * RUN_SPEED
	# 加速度，从A到B步进C：从v到vmax步进a*dt
	velocity.x = move_toward(velocity.x, direction * RUN_SPEED, ACCELERATION * delta)
	...
```

问题：玩家停止 input 后还会“漂移”一段距离（原因：没有同步更改动画播放的逻辑）

解决：调整动画播放的逻辑（输入 `direction` 为 0，且速度也为 0）

```gdscript
...
func _physics_process(delta: float) -> void:
    ...
    if is_on_floor():
#		if is_zero_approx(direction):
        # 输入方向和当前速度均为0的时候，才播放站立动画
        if is_zero_approx(direction) and is_zero_approx(velocity.x):
        	animation_player.play("idle")
        else:
        	animation_player.play("running")
    ...
```

### 4.2 区分空中/地面加速度

玩家直觉：空中很灵活，地上很迟缓（地面上逆转方向需一定时间，空中反跳可以很快反应）

```gdscript
...
#const ACCELERATION := RUN_SPEED / 0.2
# 空中转身更加容易，所以加速度更大
const FLOOR_ACCELERATION := RUN_SPEED / 0.2
const AIR_ACCELERATION := RUN_SPEED / 0.02
...

func _physics_process(delta: float) -> void:
	...
    # 在地面上：地面加速度；否则：空中加速度
	var acceleration := FLOOR_ACCELERATION if is_on_floor() else AIR_ACCELERATION
#	velocity.x = move_toward(velocity.x, direction * RUN_SPEED, ACCELERATION * delta)
	velocity.x = move_toward(velocity.x, direction * RUN_SPEED, acceleration * delta)
	...
```

### 4.3 郊狼时间（CoyoteTimer）

> 前面的优化是把游戏“往真实了做”，从而提升手感；接下来的优化是把游戏“往不真实了做”，从而提升手感。

Timer 节点设置时间为 0.1s、OneShot（一次性）

【注意】计时器如果要实现 timeout 后就停止，**必须设置 `one_shot`**，否则停止后就会立即重新开始！

【条件】必须离开地面，而且**不是「因为跳跃」离开的地面**（必须是「走出地面」的一瞬间）

|   实际条件   |                            代码逻辑                            |   操作   |
| :----------: | :------------------------------------------------------------: | :------: |
| 玩家走出地面 | `is_on_floor = 0` `was_on_floor = 1` `should_jump = 0` | 开始计时 |
| 玩家跳离地面 | `is_on_floor = 0` `was_on_floor = 1` `should_jump = 1` | 停止计时 |

```gdscript
@onready var coyote_timer: Timer = $CoyoteTimer
...
func _physics_process(delta: float) -> void:
    ...
    # 可以跳跃的条件：在地面上，或者倒计时未结束
    var can_jump = is_on_floor() or coyote_timer.time_left > 0
    # 跳跃动作的触发条件：可以跳跃，且按下了跳跃键
	var should_jump = can_jump and Input.is_action_just_pressed("jump")
#	if is_on_floor() and Input.is_action_just_pressed("jump"):
	if should_jump:
		velocity.y = JUMP_VELOCITY
        # 需要跳跃的时候，必须关掉计时器，否则就可以反复起跳了
		coyote_timer.stop()
    ...
    var was_on_floor := is_on_floor()
    move_and_slide()

	if is_on_floor() != was_on_floor:
        # 仅当不是跳跃导致的“离开地面”时，启动计时器
		if was_on_floor and not should_jump:
			coyote_timer.start()
		else:
			coyote_timer.stop()
```

### 4.4 跳跃缓冲（提前跳和长短跳）

角色快要着陆，但还没有着陆的瞬间，按下跳跃，角色也能够起跳（预判）

根据按键时长控制跳跃高度：如果刚跳跃没多久就松开跳跃键，则快速下落，实现“小跳”的效果

- 实现：跳跃键松开后，判断**向上的速度**是否还很大；如果还很大，立刻将其设置成一个较小的值，使其快速下落
- 补充：刚跳跃没多久 = 向上的速度还很大 = 速度的 y 分量还很小（负值）

参考：一个上抛运动的各时间点速度和高度的值（注：Godot 中 y 分量均为负值）

|   time   |    $0$    |   $t_{\frac{1}{4}}$   | $t_{\frac{1}{2}}$ |   $t_{\frac{3}{4}}$   |   $t_1$   |
| :------: | :---------: | :---------------------: | :-----------------: | :----------------------: | :----------: |
| velocity | $v_{max}$ | $\frac{1}{2} v_{max}$ |        $0$        | $-\frac{1}{2} v_{max}$ | $-v_{max}$ |
|  height  |    $0$    | $\frac{3}{4}h_{max}$ |     $h_{max}$     |  $\frac{3}{4}h_{max}$  |    $0$    |

代码补充：`_unhandled_input` 事件回调函数，用于处理未处理的跳跃指令

```gdscript
@onready var jump_request_timer: Timer = $JumpRequestTimer
...
func _unhandled_input(event: InputEvent) -> void:
	if event.is_action_pressed("jump"):
        # 把跳跃请求倒计时作为跳跃触发的依据
		jump_request_timer.start()
    # 松开跳跃键后，当向上的速度还很大时，更改成一个较小的值
    # 注：仅当速度向上且较大时；若速度已经很小甚至向下时，不做处理
	if event.is_action_released("jump") and velocity.y < JUMP_VELOCITY / 2:
		velocity.y = JUMP_VELOCITY / 2

func _physics_process(delta: float) -> void:
    ...
#	var should_jump = can_jump and Input.is_action_just_pressed("jump")
    # 跳跃动作的触发条件更改为：可以跳跃，且倒计时未结束
	var should_jump = can_jump and and jump_request_timer.time_left > 0
    ...
```

问题：如果在落地前 0.1 秒内（`jump_request_timer` 的 `wait_time`）按下跳跃并在落地前放开，在落地瞬间应该满足跳的条件然后跳起，但由于按键已处于 release 状态，所以不触发 `is_action_just_released("jump")`，导致因为一次短按进行一个大跳

解决：松开 jump 的同时，把 `jump_request_timer` 停掉

```gdscript
func _unhandled_input(event: InputEvent) -> void:
	if event.is_action_pressed("jump"):
		jump_request_timer.start()
#	if event.is_action_released("jump") and velocity.y < JUMP_VELOCITY / 2:
#		velocity.y = JUMP_VELOCITY / 2
	if event.is_action_released("jump"):
		jump_request_timer.stop()
		if velocity.y < JUMP_VELOCITY / 2:
			velocity.y = JUMP_VELOCITY / 2
```

## 05 状态机

### 5.0 下落动画

播放下落动画：不使用状态机的写法（麻烦，后面会用状态机重写）

```gdscript
func _unhandled_input(event: InputEvent) -> void:
    ...
    if is_on_floor():
		if is_zero_approx(direction):
			animation_player.play("idle")
		else:
			animation_player.play("running")
#	else:
	elif velocity.y < 0:
		animation_player.play("jump")
    else:
        animation_player.play("fall")
    ...
```

### 5.1 可复用的状态机脚本

新建脚本，并在对应的角色场景下，**直接找到对应的节点**并添加

要求：引用时，必须为父节点实现函数 `get_next_state`（获取下一个状态）、`transition_state`（实现状态转换后的操作）和 `tick_physics`（作为 `_physics_process` 函数的替代）

```gdscript
extends Node
class_name StateMachine

# 枚举变量的值本质上是一个int值
# 避免枚举默认为0，导致“从0变到0”
var current_state: int = -1:
	# 当前状态修改时，即调用transition_state函数
	set(v):
		owner.transition_state(current_state, v)
		current_state = v

func _ready() -> void:
	# 确保父节点ready，避免初始化后调用父节点函数而父节点unready
	await owner.ready
	current_state = 0

func _physics_process(delta: float) -> void:
	while true:
		var next := owner.get_next_state(current_state) as int
		if current_state == next:
			break
		current_state = next

	# 使用方无需定义_physics_process，只需定义此函数
	owner.tick_physics(current_state, delta)
```

### 5.2 重构当前代码

0. 声明状态枚举 State

```gdscript
enum State {
	IDLE,
	RUNNING,
	JUMP,
	FALL,
}
```

1. `get_next_state` 函数的实现

```gdscript
func get_next_state(state: State) -> State:
	var can_jump := is_on_floor() or coyote_timer.time_left > 0
	var should_jump := can_jump and jump_request_timer.time_left > 0
	if should_jump:
		return State.JUMP

    var direction := Input.get_axis("move_left", "move_right")
	var is_still := is_zero_approx(direction) and is_zero_approx(velocity.x)

    match state:
		State.IDLE:
			if not is_on_floor():
				return State.FALL
			if not is_still:
				return State.RUNNING

		State.RUNNING:
			if not is_on_floor():
				return State.FALL
			if is_still:
				return State.IDLE

		State.JUMP:
			if velocity.y >= 0:
				return State.FALL

		State.FALL:
			if is_on_floor():
				# 如果角色横向移动，直接切换成running状态
                # 补充：if后面的语句即便不写，StateMachine的死循环也可以让状态最终变成RUNNING
				return State.IDLE if is_still else State.RUNNING

    return state
```

2. `transition_state` 函数的实现

```gdscript
# 在地面上的状态：站立和跑动
const GROUND_STATES := [State.IDLE, State.RUNNING]
...

func transition_state(from: State, to: State) -> void:
    # 当上一状态不在地面上，下一状态在地面上，则关闭郊狼时间计时器
	if from not in GROUND_STATES and to in GROUND_STATES:
		coyote_timer.stop()

	match to:
		State.IDLE:
			animation_player.play("idle")

		State.RUNNING:
			animation_player.play("running")

		State.JUMP:
			animation_player.play("jump")
			velocity.y = JUMP_VELOCITY
			coyote_timer.stop()
			jump_request_timer.stop()

		State.FALL:
			animation_player.play("fall")
            # 进入fall状态，且上一状态在地面上（说明不是跳跃），则启动郊狼时间计时器
            # 使用状态机就避免了使用长串bool表达式判断上一状态是否是跳跃状态的麻烦
			if from in GROUND_STATES:
				coyote_timer.start()
```

3. `tick_physics` 函数（在 `_physics_process` 的基础上，更名，删去多余的代码，加入状态机）

为了方便起见，将原 `_physics_process` 函数的内容封装进 `move` 函数

```gdscript
func tick_physics(state: State, delta: float) -> void:
	match state:
		State.IDLE:
			move(delta)

		State.RUNNING:
			move(delta)

		State.JUMP:
			move(delta)

		State.FALL:
			move(delta)


func move(delta: float) -> void:
	var direction := Input.get_axis("move_left", "move_right")
	var acceleration := FLOOR_ACCELERATION if is_on_floor() else AIR_ACCELERATION
	velocity.x = move_toward(velocity.x, direction * RUN_SPEED, acceleration * delta)
	velocity.y += gravity * delta

	if not is_zero_approx(direction):
		sprite_2d.flip_h = direction < 0

	move_and_slide()
```

### 5.3 解决“跳不动”的问题

“跳不动”的原因：原先设置完跳跃速度后会直接 `move_and_slide`，但改写后的代码通过 `move` 函数，会先被重力减速，再调用 `move_and_slide`，导致跳跃高度变小

- 解决 1：直接更改重力（不合理，对 `delta` 有依赖）
- 解决 2：**在跳跃状态的第一帧关掉重力**（更合理的做法）

构造 `is_first_tick`，使跳跃的第一帧没有重力（避免“跳跃困难”）

这里配合修改 `move` 函数，添加参数 `gravity`，便于跳跃状态调用时更改参数（需要将原先的全局变量 `gravity` 更名，函数体内部的语句由于使用的名称是 `gravity`，无需更改）

```gdscript
#var gravity := ProjectSettings.get("physics/2d/default_gravity") as float
var default_gravity := ProjectSettings.get("physics/2d/default_gravity") as float
var is_first_tick = false
...

func tick_physics(state: State, delta: float) -> void:
	match state:
		State.IDLE:
#			move(delta)
			move(default_gravity, delta)

		State.RUNNING:
#			move(delta)
			move(default_gravity, delta)

		State.JUMP:
#			move(delta)
			# 确保第一帧时重力为0，避免出现“跳不起来”的情况
			move(0.0 if is_first_tick else default_gravity, delta)

		State.FALL:
#			move(delta)
			move(default_gravity, delta)
    # 每帧运行完后，再标识不是第一帧
    # 这样当第一帧进入tick_physics时，调用move函数的is_first_tick就为true，运行完后恢复false
    is_first_tick = false

#func move(delta: float) -> void:
func move(gravity: float, delta: float) -> void:
    ...

func transition_state(from: State, to: State) -> void:
	...
	# 标识改变状态的第一帧
	is_first_tick = true
```

### 5.4 着陆状态

添加 `LANDING` 状态（需要为 landing 状态特制一个 `stand` 函数并在 `tick_physics` 中调用）

注意：着陆动画是一次性的，要在动画节点中**取消循环动画**

对画面进行优化：

1. 对着陆动画的微调（更改吸附间隔为 0.05s，将后两帧往前移动 0.05s，缩短动画时长为 0.25s）
2. 着陆后奔跑，动画不会立刻停止，而是“边着陆边移动”：需要在 fall 状态转换时先判断是否静止

```gdscript
enum State {
	...
	LANDING,
}

#const GROUND_STATES := [State.IDLE, State.RUNNING]
const GROUND_STATES := [State.IDLE, State.RUNNING, State.LANDING]
...

func tick_physics(state: State, delta: float) -> void:
	match state:
        ...
        State.LANDING:
            # 着陆时需要保证玩家站立不动，不需要接受玩家水平移动输入
			stand(default_gravity, delta)

# 为landing状态特制的函数
func stand(gravity: float, delta: float) -> void:
	var acceleration := FLOOR_ACCELERATION if is_on_floor() else AIR_ACCELERATION
	velocity.x = move_toward(velocity.x, 0.0, acceleration * delta)
	velocity.y += gravity * delta

	move_and_slide()

func get_next_state(state: State) -> State:
    ...
    match state:
        ...
        State.FALL:
            if is_on_floor():
                # 当前状态不是静止不动时，立即转换成跑动状态
                return State.LANDING if is_still else State.RUNNING
  
        State.LANDING:
            # 着陆动画结束后，再转换到站立状态
            if not animation_player.is_playing():
                return State.IDLE
	...

func transition_state(from: State, to: State) -> void:
	...
    match to:
        ...
        State.LANDING:
			animation_player.play("landing")
```

### 5.5 补充：状态机相关

1. 枚举状态机太过于传统，为什么不使用返回节点的方式？

虽然把状态做成节点既符合 Godot 的哲学，也易于复用。但实际这样做太繁琐了，并且要花很大的力气才能真正做到状态的自由复用；状态机用枚举更易于理解，也更加适于复用要求不高的场景，如果复用要求高的话，可以使用行为树。

补充：节点状态树，使用 `@export` 进入到哪个状态，一个节点写一个状态处理状态逻辑脚本

状态机可视化插件：[imjp94/gd-YAFSM: Yet Another Finite State Machine for godot](https://github.com/imjp94/gd-YAFSM)（作者：[imjp94](https://space.bilibili.com/5519364)）

高级状态机实现教程：[Building a more advanced state machine in Godot – The Shaggy Dev](https://shaggydev.com/2022/02/13/advanced-state-machines-godot/)

2. 为什么要在函数 `_physics_process` 中设置一个 while 死循环，函数本身不是不断执行的“循环”吗？

可以节省一些状态判断的逻辑：比如从 A 状态出来的时候我要求进入 B 状态，而此时又满足从 B 进入 C 的条件，就会 A -> B -> C，当前帧最终执行的是 C 的逻辑；没有 while 的话，就会在 B 里面停留一帧。

（当然也可以在确定 A 状态进入哪个状态的时候把 B 可能进入 C 考虑进去，但是写起来就会比较麻烦）

3. 如果把 `player` 节点放到和 `tilemap` 一个场景里，和 `tilemap` 一个层级，`owner` 还能运行吗？

`owner` 只看和谁一起保存（属于哪个场景）；StateMachine 保存在 Player 场景里，那么 `owner` 就是 Player，即便这个 Player 保存在别的场景里也一样。

## 06 滑墙

### 6.1 滑墙动画

1. 素材翻转、位置改动、将属性设置加入动画轨道
2. 修复位置改动后的翻转错位问题：重设父节点为新节点，修改翻转代码（`graphics` 的 `scale.x` 设为 `-1`）

```gdscript
#@onready var sprite_2d: Sprite2D = $Sprite2D
@onready var graphics: Node2D = $Graphics

func move(gravity: float, delta: float) -> void:
    ...
	if not is_zero_approx(direction):
#		sprite_2d.flip_h = direction < 0
		graphics.scale.x = -1 if direction < 0 else +1
```

3. 使用不同素材，`texture` 等关键帧必须在其他动画中重复设置（使用插件解决）

补充：RESET 动画（一帧，0.001s，存放默认值）

### 6.2 编写逻辑部分

0. 新建滑墙状态（设置状态间的转换）

```gdscript
enum State {
	...
	WALL_SLIDING,
}

func get_next_state(state: State) -> State:
    ...
    match state:
        ...
        State.FALL:
            ...
            # Godot 内置的判断角色是否靠墙的函数
            if is_on_wall():
                return State.WALL_SLIDING

        State.WALL_SLIDING:
            # 着陆时，转换到站立状态
            if is_on_floor():
				return State.IDLE
            # 在空中离开墙面，回到下落状态
			if not is_on_wall():
				return State.FALL

func transition_state(from: State, to: State) -> void:
	...
    match to:
        ...
		State.WALL_SLIDING:
			animation_player.play("wall_sliding")
```

1. 滑墙时，`move` 参数设为 1/3 的重力
2. 滑墙动画以墙面方向而非玩家输入为准，使用 `get_wall_normal` 实现

```gdscript
func tick_physics(state: State, delta: float) -> void:
	match state:
		State.WALL_SLIDING:
			# 滑墙状态下重力会变小
			move(default_gravity / 3, delta)
            # get_wall_normal返回最近一次碰撞的墙面法线
            graphics.scale.x = get_wall_normal().x
```

3. 画面优化：滑墙条件的限制（手不能悬空，身体必须靠墙）-> 使用 `RayCast` 进行碰撞检测
4. `RayCast` 检测手和脚是否碰墙（改变父节点 scale 可以改变箭头方向）

```gdscript
@onready var hand_checker: RayCast2D = $Graphics/HandChecker
@onready var foot_checker: RayCast2D = $Graphics/FootChecker

func get_next_state(state: State) -> State:
	...
    match state:
        ...
        State.FALL:
            ...
            # 靠墙的同时，手的脚在角色朝向上都应该靠墙，否则不进入滑墙状态
            if is_on_wall() and hand_checker.is_colliding() and foot_checker.is_colliding():
                return State.WALL_SLIDING
```

### 6.3 微调动画素材

1. 保证各状态动画的位置匹配（主要是 fall 状态，jump 状态不必考虑）
2. 再次修改 landing 动画（删去第一帧，因为 fall 的位置和第一帧雷同）
3. 手感优化：landing 后有移动输入，直接进入 running 状态（否则会出现“硬直”效果）

```gdscript
func get_next_state(state: State) -> State:
    ...
    match state:
        ...
        State.LANDING:
            # 避免落地后有一小段时间无法移动
			if not is_still:
				return State.RUNNING
            ...
```

### 6.4 补充：方向的另一解

原方案的问题：用 `graphics` 将图像的节点和碰撞检测节点包起来，但由于没有包含碰撞 shape，如果遇到不对称的碰撞形状，那么在反转的时候也需要跟着反转，而 `collision_shape` 没法作为 `graphics` 的子节点

建议方案：通过控制 `character2d` 节点的 `scale` 做反转，更加直接

可能的问题：Godot 里的各种物理 Body 在做**非统一缩放**（X 和 Y 上的缩放值不一致）的时候经常会遇到各种问题，比如有时候会不停上下左右翻转、移动的时候卡住等

一种解决方案：另外设计一个变量，赋值时间接控制 `scale.x`

```gdscript
@export var move_direction := 1.0:
	set(v):
		if not is_node_ready():
			await ready
        # 这里通过set前后相乘是否小于零来确定是否要转向
		if move_direction * v < 0:
			scale.x *= -1
		move_direction = v
```

## 07 蹬墙跳

### 7.1 新建状态

基本与跳跃部分的逻辑一致，但无需处理郊狼时间（这里为蹬墙跳设置了不同的起跳速度，包含水平分量）

```gdscript
enum State {
	...
	WALL_JUMP,
}

# 蹬墙跳速度包含水平分量
const WALL_JUMP_VELOCITY := Vector2(1000, -320)

func tick_physics(state: State, delta: float) -> void:
	match state:
		...
		State.WALL_JUMP:
			move(0.0 if is_first_tick else default_gravity, delta)
    ...

func get_next_state(state: State) -> State:
    ...
    match state:
        ...
        State.WALL_SLIDING:
            # 滑墙时起跳，进入蹬墙跳状态
			if jump_request_timer.time_left > 0:
				return State.WALL_JUMP
            ...
        State.WALL_JUMP:
            # 和跳跃状态一样，速度向下时进入fall状态
			if velocity.y >= 0:
				return State.FALL

func transition_state(from: State, to: State) -> void:
    ...
    match to:
        ...
        State.WALL_JUMP:
            # 和跳跃状态一样，但更改了起跳速度，且不处理郊狼时间
			animation_player.play("jump")
			velocity = WALL_JUMP_VELOCITY
            # 默认速度向右，乘墙面法向量即可改变方向，确保水平方向与墙面方向一致
            velocity.x *= get_wall_normal().x
			jump_request_timer.stop()
```

### 7.2 设置“慢动作”方便观察调整

1. 蹬墙跳“慢动作”：`Engine.time_scale` 为游戏的时钟快慢

```gdscript
func transition_state(from: State, to: State) -> void:
    ...
    # 进入蹬墙跳状态后时钟变慢，离开后恢复
    if to == State.WALL_JUMP:
		Engine.time_scale = 0.3
	if from == State.WALL_JUMP:
		Engine.time_scale = 1.0
```

2. 优化：蹬墙跳开始的一小段时间内，角色应该始终背对墙面（但玩家输入会导致离开墙面的一瞬间方向朝向墙面，所以需要在蹬墙跳状态刚开始的一小段时间内，不接受玩家的输入）

在状态机脚本中引入 `state_time += delta` 实现 `Timer` 倒计时的效果：

```gdscript
class_name StateMachine
extends Node

var current_state: int = -1:
	set(v):
		owner.transition_state(current_state, v)
		current_state = v
        # 进入新状态时重置时间
		state_time = 0

# 在状态机中实现Timer的效果
var state_time: float

func _physics_process(delta: float) -> void:
	...
    # 进入状态后，每过一帧就增加对应的时间
	state_time += delta
```

然后，更改 `tick_physics`，确保在进入状态的一小段时间内，执行 `stand` 函数并使得 graphics 的方向为墙面法线方向：

```gdscript
func tick_physics(state: State, delta: float) -> void:
	match state:
        ...
		State.WALL_JUMP:
            # 进入状态开始的一小段时间内
			if state_machine.state_time < 0.1:
                # stand函数不接受玩家输入，参考landing状态的处理
				stand(0.0 if is_first_tick else default_gravity, delta)
				# 蹬墙跳开始的方向以墙面法线为准
				graphics.scale.x = get_wall_normal().x
			else:
                # move函数肯定不是第一帧，无需考虑is_first_tick
				move(default_gravity, delta)
```

3. 修复跳跃的 “S” 形运动：松开向左、按下向右导致的，从向左减速变成向右加速，需要微调空中加速度和蹬墙跳速度水平值

```gdscript
#const AIR_ACCELERATION := RUN_SPEED / 0.02
const AIR_ACCELERATION := RUN_SPEED / 0.1
#const WALL_JUMP_VELOCITY := Vector2(1000, -320)
const WALL_JUMP_VELOCITY := Vector2(500, -320)
```

### 7.3 优化“左右蹬墙跳”的体验

1. 蹬墙跳状态下，如果碰到墙，直接进入滑墙状态（“取消前摇”，无需等到下落才滑墙）

```gdscript
func get_next_state(state: State) -> State:
	match state:
        ...
        State.WALL_JUMP:
        	if is_on_wall():
                return State.WALL_SLIDING
    ...
```

2. 修复“慢动作”消失的问题（可以在 `transition_state` 中**打印 Debug 信息**方便定位问题所在）

```gdscript
func transition_state(from: State, to: State) -> void:
    # 打印“什么时候”，从“什么状态”转换到“什么状态”的信息
    # 如："[0] <START> ==> IDLE"，"[10] FALL ==> LANDING "
    # 在transition_state中调用，所以只有在状态转换时才会打印信息
    print("[%s] %s => %s" % [
        # 当前物理帧
        Engine.get_physics_frames(),
		# Godot 中的枚举可以调用字典的一些方法，如keys()，返回状态名称数组
        # 用State值做索引，本质上是用int值当作数组的下标索引（如果索引-1会返回最后一个值）
        # 上一个状态，如果是没有（说明现在是第一个状态）则返回"<START>"
        State.keys()[from] if from != -1 else "<START>",
        # 下一个状态
        State.keys()[to],
    ])
    ...
```

这里运行会打印形如下面的 Debug 信息（滑墙 -> 蹬墙跳 -> 滑墙）

```text
[1990] FALL ==> WALL_SLIDING
[2000] WALL_SLIDING ==> WALL_JUMP
[2000] WALL_JUMP ==> WALL_SLIDING
[2001] WALL_SLIDING ==> FALL
```

根据 Debug 信息，进入 WALL_JUMP 状态的瞬间又会回到 WALL_SLIDING 状态，原因是从 WALL_SLIDING 状态进入 WALL_JUMP `状态后，is_on_wall` 仍然为 true，触发了转换回 WALL_SLIDING 状态的逻辑。

解决方案就是用 `is_first_tick` 限定条件，刚进入蹬墙跳状态时的第一帧不改变状态：

```gdscript
func get_next_state(state: State) -> State:
	match state:
		...
		State.WALL_SLIDING:
			# 触发了跳跃的条件
			if jump_request_timer.time_left > 0:
				return State.WALL_JUMP
            ...
        State.WALL_JUMP:
#			if is_on_wall():
			# is_first_tick确保刚进入WALL_JUMP状态时，不会立即回到滑墙状态
    		# 由于滑墙跳开始时水平方向必定背对墙面，所以第二帧的时候就不会满足is_on_wall了
			if is_on_wall() and not is_first_tick:
        		return State.WALL_SLIDING
        	...
```

出现了之前“手脚悬空”也能滑墙的问题，需要把头脚的碰撞检测也考虑进来，可以封装成一个函数：

```gdscript
func can_wall_silde() -> bool:
	return is_on_wall() and hand_checker.is_colliding() and foot_checker.is_colliding()

func get_next_state(state: State) -> State:
	match state:
		...
        State.FALL:
            ...
#			if is_on_wall() and hand_checker.is_colliding() and foot_checker.is_colliding():
			if can_wall_silde():
				return State.WALL_SLIDING
        ...
        State.WALL_JUMP:
#			if is_on_wall() and not is_first_tick:
			if can_wall_silde() and not is_first_tick:
				return State.WALL_SLIDING
```

一个可选的修复建议：快速地按下跳跃键，蹬墙跳时会直接省略“滑墙”动画，应该保留过渡动画

这个时候也会打印类似下方的 Debug 信息（蹬墙跳 -> 滑墙 -> 蹬墙跳）

```text
[1980] WALL_SLIDING ==> WALL_JUMP
[2000] WALL_JUMP ==> WALL_SLIDING
[2000] WALL_SLIDING ==> WALL_JUMP
[2040] WALL_JUMP ==> FALL
```

可能的解决方案：使用 is_first_tick 或 state_time 加以限制

```gdscript
func get_next_state(state: State) -> State:
	match state:
		...
		State.WALL_SLIDING:
			if jump_request_timer.time_left > 0 and not is_first_tick:
            #或者：
#			if jump_request_timer.time_left > 0 and state_machine.state_time < 0.1:
				return State.WALL_JUMP
```

### 7.4 删除调试代码与数值优化

1. 删除“慢动作”的逻辑和 Debug 代码（或者注释掉）
2. 自行测试，调整 `WALL_JUMP_VELOCITY` 及其他变量的数值（根据实际需要，不必照抄案例）

参考数值（目前的案例）

```gdscript
const RUN_SPEED := 160.0
const FLOOR_ACCELERATION := RUN_SPEED / 0.2
const AIR_ACCELERATION := RUN_SPEED / 0.1
const JUMP_VELOCITY := -320.0
const WALL_JUMP_VELOCITY := Vector2(380, -280)
```

## 08 野猪

### 8.1 制作敌人场景

为敌人设计一个模板场景，各节点的设计与 Player 类似

- Enemy（Character2D 节点）
  - Graphics（Node2D 节点）
    - Sprite2D
  - CollisionShape2D（形状留空）
  - AnimationPlayer
  - StateMachine（脚本）

编写模板场景的脚本（`@export` 声明导出变量，可以在编辑器中赋值，类似 Unity 的 `[SerializeField]`）

```gdscript
class_name Enemy
extends CharacterBody2D

enum Direction {
	LEFT = -1,
	RIGHT = +1,
}

@export var direction := Direction.LEFT:
	set(v):
		direction = v
        # 素材图片默认面朝左边
		graphics.scale.x = -direction
# 在父场景中设置最大速度和加速度，声明为导出变量，子场景可以修改
@export var max_speed: float = 180
@export var acceleration: float = 2000

var default_gravity := ProjectSettings.get("physics/2d/default_gravity") as float

@onready var graphics: Node2D = $Graphics
@onready var animation_player: AnimationPlayer = $AnimationPlayer
@onready var state_machine: Node = $StateMachine
```

### 8.2 制作野猪场景

新建空场景，选择继承自 Enemy 场景（**黄色的节点**表示继承自其他场景）

参照 Player，在 `graphics` 下新建碰撞检测（分别检测墙壁和地面，确保野猪不会撞墙和走出悬崖）

【注意】检测地面的 `RayCast` 指向地面，原点应该在**地面上方**，如果在 x 轴上可能会导致无法检测到地面

检测玩家：需要设置**碰撞层**（Collision Layer）和**碰撞遮罩/掩码**（Collision Mask）

【区别】Layer 表示在“哪一层”，Mask 表示“只会和哪一层相碰撞”

（补充：玩家和敌人不在同一层，因为玩家可以穿过敌人）

### 8.3 编写野猪逻辑

实现基本的状态和状态机需要的函数

补充：新建 `calm_down_timer` 设置为 2.5s，OneShot

```gdscript
extends Enemy

enum State {
	IDLE,
	WALK,
	RUN,
}

@onready var wall_checker: RayCast2D = $Graphics/WallChecker
@onready var floor_checker: RayCast2D = $Graphics/FloorChecker
@onready var player_checker: RayCast2D = $Graphics/PlayerChecker
@onready var calm_down_timer: Timer = $CalmDownTimer

func tick_physics(state: State, delta: float) -> void:
	match state:
		State.IDLE:
            # 站立状态静止不动，move_toward到0
			move(0.0, delta)

		State.WALK:
            # 走动是最大速度的1/3
			move(max_speed / 3, delta)

		State.RUN:
            # 跑动时时刻检测墙壁和悬崖，并立即转向
			if wall_checker.is_colliding() or not floor_checker.is_colliding():
				direction *= -1
			move(max_speed, delta)
			# 看到玩家时，开始计时（如若一直看到玩家，则时刻刷新；直到玩家从视野中消失，才会慢慢减少）
            if player_checker.is_colliding():
				calm_down_timer.start()

func get_next_state(state: State) -> State:
    # 看到玩家，进入暴走状态
	if player_checker.is_colliding():
		return State.RUN

	match state:
		State.IDLE:
            # 保持站立2s后会转换到走动状态
			if state_machine.state_time > 2:
				return State.WALK

		State.WALK:
            # 当前面是墙壁或悬崖时，转换到站立状态
            # 注意：墙壁是碰撞检测到的情况，悬崖是碰撞检测不到的情况
			if wall_checker.is_colliding() or not floor_checker.is_colliding():
				return State.IDLE

		State.RUN:
            # 等到“冷静”计时器结束再恢复到walk状态
			if calm_down_timer.is_stopped():
				return State.WALK

	return state

func transition_state(from: State, to: State) -> void:
	match to:
		State.IDLE:
			animation_player.play("idle")
            # 碰到墙面，立即转身
            # 如果是悬崖，不会立即转身，会等到进入walk状态时再转身
			if wall_checker.is_colliding():
				direction *= -1

		State.WALK:
			animation_player.play("walk")
            # 碰到悬崖，立即转身
			if not floor_checker.is_colliding():
				direction *= -1
			floor_checker.force_raycast_update()

		State.RUN:
			animation_player.play("run")
```

由于 `tick_physics` 函数调用了 `move` 函数，可以在父场景中定义一个基本的 `move` 函数，子场景只需要传入目标速度的参数 `speed` 即可）

```gdscript
# Enemy.gd
# speed为目标速度
func move(speed: float, delta: float) -> void:
	velocity.x = move_toward(velocity.x, speed * direction, acceleration * delta)
	velocity.y += default_gravity * delta

	move_and_slide()
```

### 8.4 调试常见错误

1. `export` 先于 `onready` 初始化，所以 `export` 的 `set` 方法修改值 `onready` 的值时，应该等待 ready 完成

```gdscript
@export var direction := Direction.LEFT:
	set(v):
		direction = v
        # 等待当前节点ready，在修改其变量
		if not is_node_ready():
			await ready
		graphics.scale.x = -direction
```

2. Godot 的 raycast 碰撞检测会缓存旧值（这会导致野猪转身的时候，仍然沿用之前的碰撞检测值，认为前方是悬崖，所以会先停止一会儿，然后再走动）

解决：在转身的逻辑后，强制更新 raycast 再进行碰撞检测

```gdscript
func transition_state(from: State, to: State) -> void:
	match to:
		...
		State.WALK:
			animation_player.play("walk")
			if not floor_checker.is_colliding():
				direction *= -1
            # 转身之后，强制再进行碰撞检测
			floor_checker.force_raycast_update()
```

## 09 三段攻击

### 9.1 设置场景

- 设置 `can_combo` 变量，在动画轨道上添加 true 和 false 的帧
- 添加 `attack` 输入映射（如果想要降低难度，可以添加一个 `attack_request_timer`）

### 9.2 编写代码

0. 在代码中添加攻击状态

【快捷键】<kbd>Ctrl</kbd> + <kbd>Shift</kbd> + <kbd>D</kbd>：复制上一行

```gdscript
enum State {
	...
	ATTACK_1,
	ATTACK_2,
	ATTACK_3,
}

# 攻击状态也属于地面状态
const GROUND_STATES := [
	State.IDLE, State.RUNNING, State.LANDING,
	State.ATTACK_1, State.ATTACK_2, State.ATTACK_3,
]
```

1. 在 `_unhandled_input` 函数中添加连击状态的判定条件

```gdscript
@export var can_combo: bool = false
var is_combo_requested := false

func _unhandled_input(event: InputEvent) -> void:
	...
    # 仅当可以连击，且按下攻击键后，触发连击条件
	if Input.is_action_just_pressed("attack") and can_combo:
		is_combo_requested = true
```

2. 攻击状态转换的逻辑

```gdscript
func get_next_state(state: State) -> State:
	...
	match state:
        State.IDLE:
            if not is_on_floor():
                return State.FALL
            # 在下落之后，判断是否按下攻击键，以决定是否进入攻击第一状态
            if Input.is_action_just_pressed("attack"):
				return State.ATTACK_1
			...
        State.RUNNING:
            if not is_on_floor():
                return State.FALL
            # 在下落之后，判断是否按下攻击键，以决定是否进入攻击第一状态（与IDLE部分逻辑一致）
            if Input.is_action_just_pressed("attack"):
				return State.ATTACK_1
		...
		State.ATTACK_1:
            # 动画播放完，如果触发连击条件，则进入下一状态；否则回到IDLE状态
			if not animation_player.is_playing():
				return State.ATTACK_2 if is_combo_requested else State.IDLE

		State.ATTACK_2:
            # 动画播放完，如果触发连击条件，则进入下一状态；否则回到IDLE状态（同上）
			if not animation_player.is_playing():
				return State.ATTACK_3 if is_combo_requested else State.IDLE

		State.ATTACK_3:
            # 动画播放完，回到IDLE状态（因为没有后续）
			if not animation_player.is_playing():
				return State.IDLE
```

3. 处理地面消失的情况（应该优先处理，所以可以把 `is_on_floor` 的判断放在开头）

补充：这样也解决了 LANDING 状态下如果离开地面不会立刻进入 FALL 状态的隐藏 Bug

```gdscript
func get_next_state(state: State) -> State:
	...
    # 如果上一状态属于“地面上状态”，但现在不满足is_on_floor，则进入fall状态
	if state in GROUND_STATES and not is_on_floor():
		return State.FALL
    ...
	match state:
        State.IDLE:
#			if not is_on_floor():
#				return State.FALL
            if Input.is_action_just_pressed("attack"):
				return State.ATTACK_1
			...
        State.RUNNING:
#			if not is_on_floor():
#				return State.FALL
            if Input.is_action_just_pressed("attack"):
				return State.ATTACK_1
```

4. 补完剩下的状态机逻辑

```gdscript
func tick_physics(state: State, delta: float) -> void:
	match state:
        ...
        # 三段攻击的物理逻辑相同：要求攻击时停在原地，动画播完后再移动
		State.ATTACK_1, State.ATTACK_2, State.ATTACK_3:
			stand(default_gravity, delta)

func transition_state(from: State, to: State) -> void:
	...
    match to:
		...
        # 播放动画，并将连击触发条件恢复到false（下同）
		State.ATTACK_1:
			animation_player.play("attack_1")
			is_combo_requested = false

		State.ATTACK_2:
			animation_player.play("attack_2")
			is_combo_requested = false

		State.ATTACK_3:
			animation_player.play("attack_3")
			is_combo_requested = false
```

### 9.3 野猪 Bug 修复

Bug：野猪可以透过墙面看到玩家

- `player_checker` 添加环境 mask，同时使用 `can_see_player` 做判断
- 需要在 `player` 脚本前添加 `class_name Player`

```gdscript
# boar.gd
func can_see_player() -> bool:
    # 没检测到，返回false
	if not player_checker.is_colliding():
		return false
	else:
        # 仅当检测到的对象是Player是，返回true
		return player_checker.get_collider() is Player

func tick_physics(state: State, delta: float) -> void:
	match state:
		...
		State.RUN:
#			if player_checker.is_colliding():
			if can_see_player():
				calm_down_timer.start()

func get_next_state(state: State) -> State:
#	if player_checker.is_colliding():
	if can_see_player():
		return State.RUN
    ...
```

### 9.4 补充：项目参考

一个暂时做到三段攻击的项目，有部分改进：[xingmot/2d\_ARPG](https://github.com/xingmot/2d_ARPG)

- 修改了跳跃、着陆和蹬墙跳的手感，然后加了往上看、往下看（以及左上、右上、左下、右下）
- 着陆状态现在只有从比较高的地方掉下来才会进入，而且此状态下玩家仍然可以左右移动，但是速度会变慢
- 为蹬墙跳的 x 轴方向也做了小跳

## 10 攻击框

### 10.1 攻击框和受击框

要点：将攻击双方抽象成 hitbox 和 hurtbox

- 两个 box 的重叠表示“攻击”
- 通过信号传递 hit 和 hurt 信息，一般只由其中一方发出（案例中是 hitbox）

hitbox.gd

```gdscript
extends Area2D
class_name Hitbox

signal hit(hurtbox)

func _init() -> void:
	area_entered.connect(_on_area_entered)

func _on_area_entered(hurtbox: Hurtbox) -> void:
	# 调试内容：谁打了谁
	print("[Hit] %s => %s" % [owner.name, hurtbox.owner.name])
	hit.emit(hurtbox)
	hurtbox.hurt.emit(self)
```

hurtbox.gd

```gdscript
extends Area2D
class_name Hurtbox

signal hurt(hitbox)
```

### 10.2 场景处理

1. 野猪攻击玩家

设置：需要为 hurtbox 专门设置物理层和碰撞形状

- Hurtbox 应该呆在自己的层上（layer），不主动寻找别人（mask）
- Hitbox 不应该呆在任何层上（layer），但需要寻找别的 Hurtbox（mask）

Area2D 的**碰撞区域可以设置多个**，组成更复杂的形状（如十字）

2. 玩家攻击野猪

注意：玩家三段攻击的攻击区域各不相同（通过动画帧设置）

Godot 复制节点，资源是共享的，所以复制节点更改属性，原节点也会更改（这时需要在 Rectangle2D 中的矩形选择「唯一化」）

可以在运行时，通过左侧节点树的「远程」选项，将野猪的 PlayerChecker 禁用（野猪不会“暴走”），便于测试三段攻击的命中效果

### 10.3 传递信号

脚本中自定义的信号，可以在节点面板找到并添加（这里也体现了信号参数的作用）

如：在野猪脚本中（使用信号）添加如下函数，玩家攻击野猪后，打印“Ouch!”

```gdscript
func _on_hurtbox_hurt(hitbox: Hitbox) -> void:
	print("Ouch!")
```

### 10.4 补充建议

1. 这种解决方法在大部分情况都有效果，但是在处理隔墙或者隔盾攻击等场景时无法满足需求。比如隔盾攻击时如果攻击框同时覆盖盾和敌人，希望是盾收到攻击判定，但是如果是从敌人后方同时覆盖，则希望是敌人收到判断。一种解决方案是加上 `raycast`，碰到墙壁或者盾后停止，根据 RayCast 长度修正攻击框形状，但是这个解决方案有点复杂。是否有更简明一些的解决方案?

这种设计不可避免地会涉及到 RayCast；盾牌的情况，因为有时候可能会希望隔着盾牌只是减少若干百分比的攻击，或者也能产生一定的击退，所以不在攻击/受击框的层面解决这个问题会灵活一点。

2. 攻击判定还是用代码控制好一些。以坐标进行判定，攻击伤害、击飞、出现时间等数据可以存在数据表内，不用依赖于动画。

使用动画来控制一些行为理论上肯定是没有问题的，对于小项目而言完全够用；当然，根据评论区给出的建议，通过别的方式控制或许会更好。

<!--个人补充：参考之前个人复刻油管的 ARPG 游戏教程，将其移植到 Godot 4 版本的时候，就在动画处理中出了问题。当时的问题是，要实现攻击状态动画播放完毕后进入站立/跑动状态，如果直接在动画帧上新建信号函数会出 Bug，当时的解决方案就是在代码中使用计时器，设置为动画时长，进入攻击状态后开始计时，`timeout` 后进入下一状态。

现在回头再看，其实直接在脚本中判断当前的 `animation_player` 是否在播放动画即可（因为攻击动画必然是没有循环的）；不过当时的项目是使用动画树的，会有很多问题。在这种情况下，使用计时器这种方法虽然麻烦，但肯定是最保险的做法。

此外，这种方式也很符合把逻辑和画面拆分的思想。逻辑上，攻击状态是按下攻击键引发的，动画是进入攻击状态后开始播放的；退出攻击状态，也应该与攻击键绑定，而不是与动画绑定。-->

## 11 受伤和死亡

### 11.1 基础逻辑

如果简单地实现“被打后消失”，可以直接调用 `queue_free` 函数：

```gdscript
func _on_hurtbox_hurt(hitbox: Hitbox) -> void:
	queue_free()
```

显然，我们需要野猪血更厚，这就需要为野猪设置血量。我们可以写一个 stats.gd 脚本，用于存储和处理对象（玩家和敌人）的血量等统计数值。

```gdscript
extends Node
class_name Stats

@export var max_health: int = 3

# 节点ready后再初始化，避免了export值更改无法同步变更的问题
@onready var health: int = max_health:
	set(v):
		# 限制health的范围在0-max之间
		v = clampi(v, 0, max_health)
		if health == v:
			return
		health = v
```

在 Enemy 场景中导入 Stats 节点（并在脚本中引用），在野猪场景中编写代码：

```gdscript
# enemy.gd
@onready var stats: Node = $Stats

# boar.gd
func _on_hurtbox_hurt(hitbox: Hitbox) -> void:
	stats.health -= 1
	if stats.health == 0:
		queue_free()
```

这是最简单的实现，默认攻击 1 次减少 1 点血，可以引入玩家的攻击力，或者针对玩家的几段攻击予以不同的扣血量等等。这里是教程就不多做延申了。

### 11.2 野猪动画

淡出效果：添加 `modulate` 关键帧，使得开始的 `alpha` 为 1，结束的 `alpha` 为 0（这个值会乘上颜色，乘 0 就表示透明）

野猪受击或死亡时，进入“硬直”状态，不会再对玩家攻击，不会再受到攻击：通过动画帧实现

补充：Area2D 的 `monitoring` 是能否检测别的区域，`monitorable` 是能否被别的区域检测到

### 11.3 野猪代码

添加受击和濒死状态：

```gdscript
enum State {
	...
	HURT,
	DYING,
}
```

一般的教程会在 `_on_hurtbox_hurt` 函数中处理受击逻辑，本教程的做法是只传递信息，然后交给状态机的处理函数处理。这里使用一个继承自 RefCounted 的脚本来完成。

注：ReferCounted 是最基础的计数类，会**在不使用的时候自动释放**

```gdscript
# damage.gd
extends RefCounted
class_name Damage

var amount: int
var source: Node2D
```

在脚本中新建变量 `pending_damage` 表示待处理的伤害：

```gdscript
var pending_damage: Damage

func _on_hurtbox_hurt(hitbox: Hitbox) -> void:
	pending_damage = Damage.new()
	pending_damage.amount = 1
	# 这意味着永远只记录最后一次攻击的攻击方
	# 如果需要记录多个攻击者的话，可以改成数组或者混合处理
	pending_damage.source = hitbox.owner
```

改写状态机相关的函数代码，这里将 `can_see_player` 的逻辑写进原先的各状态里面，因为引入的新状态会导致旧逻辑不成立（看到玩家，野猪不一定会跑动）

```gdscript
# 击退值
const KNOCKBACK_AMOUNT := 512.0

func tick_physics(state: State, delta: float) -> void:
	match state:
		# 受击和濒死状态是不动的
		State.IDLE, State.HURT, State.DYING:
			move(0.0, delta)
		...

func get_next_state(state: State) -> State:
#	if can_see_player():
#		return State.RUN

	if stats.health == 0:
		return State.DYING

	if pending_damage:
		return State.HURT

	match state:
		State.IDLE:
			if can_see_player():
				return State.RUN
			...

		State.WALK:
			if can_see_player():
				return State.RUN
			...

		State.RUN:
			if not can_see_player() and calm_down_timer.is_stopped():
				return State.WALK
	
		State.HURT:
			if not animation_player.is_playing():
				return State.RUN

	return state

func transition_state(from: State, to: State) -> void:
	match to:
		...
		State.HURT:
			animation_player.play("hit")
			stats.health -= pending_damage.amount
			# 方向由攻击来源指向自己
			var dir := pending_damage.source.global_position.direction_to(global_position)
			velocity = dir * KNOCKBACK_AMOUNT
		
			if dir.x > 0:
				direction = Direction.LEFT
			else:
				direction = Direction.RIGHT
		
			pending_damage = null
		
		State.DYING:
			animation_player.play("die")
```

目前野猪血量为 0 后会变透明，但不会真的消失，还是需要调用 queue_free 函数。我们可以在 enemy 脚本中编写函数专门处理死亡的情况。

```gdscript
func die() -> void:
	queue_free()
```

一个可行的调用方法是，在 `get_next_state` 函数中编写 `State.DYING` 逻辑，但作者人物这样会使得 `get_next_state` 函数“不再纯粹”。作者更建议直接在动画轨道中调用函数。

### 11.4 问题修复

1. 玩家攻击很迟钝（“前摇太长”）：仿照 landing 动画缩短前几帧的时长
2. 野猪实际上是不需要设置十字形状的碰撞区域的（教程只是为了演示多碰撞区域）
3. 可能会出现玩家背对野猪攻击依然能够攻击到的情况
   1. 解决方案 1：加入方向判断（比较麻烦）
   2. 解决方案 2：将 hitbox 缩小（向玩家面朝方向移动）
4. 快速攻击可能会卡在受击状态（可以对状态机脚本进行修改）

```gdscript
const KEEP_CURRENT := -1

func _physics_process(delta: float) -> void:
	while true:
		var next := owner.get_next_state(current_state) as int
#		if current_state == next:
		# 用-1代替状态不变的情况
		if next == KEEP_CURRENT:
			break
		current_state = next
```

需要对 Player 和 Boar 的状态机部分的 `get_next_state` 函数进行修改：将函数返回类型改成 int（因为会返回 -1），将兜底返回值改成 KEEP_CURRENT（表示状态不变）

```gdscript
# 对 player.gd 和 boar.gd
#func get_next_state(state: State) -> State:
func get_next_state(state: State) -> int:
	...
#	return state
	# 直接引用常量，所以使用静态类的属性即可
	return StateMachine.KEEP_CURRENT
```

<!-- 不必彻底理解内在的机理，只需要了解有这个问题，以后写代码的时候注意即可；不要在这种问题上浪费太多的时间！ -->

这段代码就可以保证进入受击状态后，不会再卡在受击状态中：

1. `get_next_state` **每帧都会调用**，而 `transition_state` 只有改变状态了才会调用
2. 在当前状态为 hurt，清空 pending_damage 且动画未播完后，又受到了一次攻击，这个时候 hurt -> hurt 不会调用 `transition_state` 中清空 pending_damage 的逻辑，所以会卡死在 hurt 状态中
3. 现在引入了 KEEP_CURRENT 后，保持在 hurt 状态下会返回 -1，在这期间再次受到攻击，就会再次返回 hurt 状态，调用 `transition_state` 函数，清空 pending_damage，避免卡死

当然，这样就会导致死亡后会重复进入死亡状态（血量为 0 -> 死了 -> 返回 -1 -> 血量为 0 -> 死了 -> 返回 -1 -> ……），需要稍作修改：

```gdscript
func get_next_state(state: State) -> int:
	if stats.health == 0:
#		return State.DYING
		# 保证第一帧进入的时候返回dying，以后都返回-1
		return StateMachine.KEEP_CURRENT if state == State.DYING else State.DYING
```

### 11.5 玩家动画

- 为玩家添加 hurt 和 die 动画（无需考虑 hitbox，因为 hitbox 只会在攻击阶段出现）
- 框选素材范围超过一行，需要额外设置 VFrames 为 2
- 死亡动画播放时，不应该再有 hurtbox 了，设置 hurtbox 的 `monitorable` 为 false
- 调整 RESET 动画的默认属性（`VFrames = 1`，`monitorable = true`）

### 11.6 玩家代码

效仿野猪代码的实现，添加状态、pending_damage、处理函数等：

```gdscript
enum State {
	...
	HURT,
	DYING,
}

const KNOCKBACK_AMOUNT := 512.0
var pending_damage: Damage
@onready var stats: Node = $Stats

func tick_physics(state: State, delta: float) -> void:
	match state:
		...
		State.HURT, State.DYING:
			stand(default_gravity, delta)
	...

func get_next_state(state: State) -> int:
	if stats.health == 0:
		return StateMachine.KEEP_CURRENT if state == State.DYING else State.DYING
	if pending_damage:
		return State.HURT
	...
	match state:
		...
		State.HURT:
			if not animation_player.is_playing():
				return State.RUN
	...

func transition_state(from: State, to: State) -> void:
	match to:
		...
		State.HURT:
			animation_player.play("hurt")
			stats.health -= pending_damage.amount
			var dir := pending_damage.source.global_position.direction_to(global_position)
			velocity = dir * KNOCKBACK_AMOUNT
			# 玩家代码不需要考虑受击后的方向
			pending_damage = null

		State.DYING:
			animation_player.play("die")
	...
```

为玩家编写死亡处理函数（游戏场景重新加载）并在动画轨道中调用

```gdscript
func die() -> void:
	# 重新加载当前场景
	get_tree().reload_current_scene()
```

如果需要死亡动画播完后等一会再重新加载，可以延长死亡动画播放的时间

### 11.7 玩家无敌

问题：野猪可能会连续对玩家造成多次伤害（需要在被攻击时设置“无敌时间”）

```gdscript
@onready var invincible_timer: Timer = $InvincibleTimer

func transition_state(from: State, to: State) -> void:
	match to:
		...
		State.HURT:
			...
			# 受击后，开启无敌时间
			invincible_timer.start()
	...

func _on_hurtbox_hurt(hitbox: Hitbox) -> void:
	# 被攻击时，如果处于无敌时间，则不作处理
	if invincible_timer.time_left > 0:
		return
	...
```

我们可以给玩家受击设置“一闪一闪”的效果，这里使用 `sin` 函数设置透明度 `alpha` 来实现：

```gdscript
func tick_physics(state: State, delta: float) -> void:
	if invincible_timer.time_left > 0:
		# sin(t)*0.5+0.5 确保取值在[0,1]之间
		# Time.get_ticks_msec() 返回从游戏开始到现在经过了多少毫秒
		graphics.modulate.a = sin(Time.get_ticks_msec() / 20) * 0.5 + 0.5
	else:
		graphics.modulate.a = 1
	...
```

当然，死亡的时候就不要再闪烁了，需要关闭计时器

```gdscript
func transition_state(from: State, to: State) -> void:
	match to:
		...
		State.DYING:
			animation_player.play("die")
			# 死亡后，无敌时间关闭
			invincible_timer.stop()
	...
```

## 12 血条

### 12.1 头像框

- 新建场景并加入 HBoxContainer 节点（默认大小是 40×40，教程作者习惯把大小清零，然后让内容将 Container “撑起来”）
- 使用 AtlasTexture 可以像 Sprite2D 一样对所选素材图集进行切割框选
- PanelContainer：专门为控件提供背景的容器

问题：头像是 11×11 的，背景是 26×26 的，但容器会跟着头像缩小，而不是头像跟着容器放大

解决：在 PanelContainer 的 Layout 属性中，将 Custom Minimum Size 设置成 26×26

- PanelContainer 做单一方向的拉伸，子节点也会跟着拉伸
- 我们希望无论如何头像都保持长宽比且填充背景，所以可以将头像的 `scratch mode` 设置成 `keep aspect centered`
- 在 PanelContainer 素材的 Content Margins 中设置背景和头像的间距

### 12.2 血条

- 血条的本质是“进度条”，这里使用 TextureProgressBar 来实现
- TextureProgressBar 理论上可以设置三种素材：Under（背景板），Over（顶层，如进度框），Progress（进度）
- 使用 ProgressOffset 调整进度条与进度框之间的素材偏移
- 作者习惯上将 MaxValue 归一化，设置成 1，步长设置成 0，Value 为 0 ~ 1 之间的浮点数（注意：不要勾选 Exp Edit）

<!-- 这个操作很类似 Unity 中的操作，但我一直不太理解 Unity 中的导出变量和事件是如何运行的，而且 Godot 这里也还是在一个更大的场景树下，引用这个场景树的节点——如果不是同一个场景呢？当然这种方法足够应对很多我过去没能很好处理的情况了。 -->

为 StatusPanel 编写脚本，注意到 StatusPanel 作为独立场景没有 stats 节点，所以需要使用导出变量，新建一个待导入的 stats 变量，然后继续编写代码（这里需要在 stats 中设置信号并在 health 改变的时候发出，然后 StatusPanel 接受信号）

```gdscript
extends HBoxContainer

@export var stats: Stats
@onready var health_bar: TextureProgressBar = $HealthBar

func _ready() -> void:
	# 场景初始化的时候，连接信号
	stats.health_changed.connect(update_health)
	update_health()

func update_health() -> void:
	# 根据stats的health值，设置进度条的百分比
	var percentage := stats.health / float(stats.max_health)
	health_bar.value = percentage
```

stats.gd 新增信号 health_changed：

```gdscript
signal health_changed

@onready var health: int = max_health:
	set(v):
		...
		health = v
		health_changed.emit()
```

代码编写完后，需要在 Player 场景下实例化 StatusPanel，然后在导出变量 `stats` 中指定节点。

我们希望血条面板固定在屏幕的左上角，而不是跟随玩家。可以用一个 CanvasLayer 包裹住面板，这样面板的位置就会相对屏幕，而非相对玩家。

### 12.3 血条动画

血条的“缓冲”效果：使用一绿一红的两个血条，绿条在前，受伤后直接变短；红条在后，受伤后慢慢变短即可

注意：复制进度条，设置不同素材，唯一化的时候针对的是 Progress 而不是 Atlas！

- 清除红色血条的边框（用不到，直接使用绿色血条的即可）
- 设置 CanvasItem - Visibility - Show Behind Parent 为 true

编写代码，为红色血条创建“缓冲”效果的补间动画：

```gdscript
@onready var eased_health_bar: TextureProgressBar = $HealthBar/EasedHealthBar

func update_health() -> void:
	var percentage := stats.health / float(stats.max_health)
	health_bar.value = percentage
	# 为红色血条创建补间动画：value -> percentage 持续0.3s
	create_tween().tween_property(eased_health_bar, "value", percentage, 0.3)
```

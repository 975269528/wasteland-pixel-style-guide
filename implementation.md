# 像素伪3D制作与运行实现

本文记录 `adbd259` 的可复用做法。文中路径均相对项目根目录；下文用“小样目录”指
`research/pseudo3d-prototype/`。阅读和复现不需要原始参考视频。
离线分享包以文档和精选画面为主；源码定位用于说明实现，不表示包内包含全部源码、资产或可执行程序。
执行下文命令需另有配套小样源码与完整资产。

## 1. 技术边界

当前方案是固定斜俯视的 2.5D 画面：离线建立有体积的低模母版，生成多方向动作 PNG，
运行时以 Lua、NanoVG 和 2D 图集合成。游戏不加载 `.blend`，也不实时计算角色骨骼。
真实体积、关节和自遮挡在烘焙阶段确定，运行时只选择帧、排列深度、移动脚点和绘制反馈。

小样直接使用已有官方 UrhoX Runtime，未绑定 Maker 项目，无平台身份和远端构建。
已验证的是 Windows 本地交互及 GPU 呈现；触控、手机显存、性能、完整战斗和联网均不在验证范围。

## 2. 资源与代码分工

| 工作 | 小样目录内入口 | 产物/职责 |
| --- | --- | --- |
| 基础人马母版 | `assets/source/bake_sprites.py`、`model_parts.py` | 多方向原生帧，透明背景，不烘地面影 |
| 武器与整身动作 | `assets/source/bake_actions.py`、`action_pose.py`、`action_weapons.py` | 握持、蓄力、接触和收招的母版 |
| 移动战斗分层 | `assets/source/bake_combat_layers.py`、`combat_geometry.py` | face/relative 下身、持械上身、马与坐腿底层 |
| 普通走跑 | `assets/source/bake_locomotion.py`、`normal_pose.py` | 独立直腿走路、跑步和自然 idle |
| 骑乘近战 | `assets/source/bake_mounted_melee.py`、`mounted_melee_pose.py` | 座高下劈/下刺和真实切刃点 |
| 细分骑枪 | `assets/source/bake_mounted_gun16.py` | 真正 16 方向的身体/手姿 |
| 环境 | `assets/source/bake_environment.py` | Pillow/NumPy 制作的地表、树、岩壁和小景物 |
| 图集与数据 | `assets/source/pack_*.py`、`write_combat_metadata.py` | PNG、manifest、手锚、步幅、座位偏移 |
| 运行时 | `scripts/main.lua`、`actions.lua`、`combat.lua` | 输入、状态推进、攻击时序与命中 |
| 呈现 | `scripts/pixel.lua`、`renderer.lua`、`presentation.lua` | 像素世界、清晰 HUD 和颜色适配 |

母版通过独立后台 Blender 创建，不依赖用户活动场景。脚本、源模型和报告应一起归档；
只有 PNG 不足以恢复关节、相机、武器刃向或分层接缝。生成目录和具体命令见 `assets/source/README.md`。
该说明包含演进阶段记录；当前播放参数以 `scripts/config.lua` 为准。

## 3. 相机、像素与坐标契约

离线相机采用固定 45° 俯角、正交投影。所有同场资产共用相机、尺度、灯光和脚锚。
修改相机角度应整组重烘，不能旋转某张 PNG 来假装新的 3D 视角。

运行时逻辑场地为 `1280×720`，世界缓冲为 `640×360`。逻辑纵向地面压缩系数为 `0.66`。
方向编号从 0 开始：下、右下、右、右上、上、左上、左、左下。
图集行列从 0 开始；Lua 元数据用 `direction+1`、`frame+1` 访问。

| 资产 | 单格/图集 | 脚锚 | 使用约束 |
| --- | --- | --- | --- |
| 旧完整主体 | 256 格，6 列×8 行 | `(128,220)` | 原生 128 格最近邻放回 256；当前独立马仍使用 |
| 分层上身 | 原生 128 格 | `(64,110)` | 近战攻击 6 列；步行 hold 两列为站/蹲 |
| 战斗下身 | 原生 64 格 | `(32,46)` | walk/run 6 列，crouch 2 列；每模式 8 张 relative 图 |
| 普通下身 | `448×512`，64 格、7 列×8 行 | `(32,46)` | 列 0–5 循环，列 6 是独立 idle |
| 骑乘底层 | 128 格，6 列×8 行 | `(64,110)` | 马、pelvis、坐腿一起烘焙；行对应 move |
| 骑枪上身 | `256×1024`，128 格、2 列×8 行 | `(64,110)` | pose 0–15：列 `pose%2`，行 `floor(pose/2)` |
| 骑近战上身 | 128 格，hold 1 列/attack 6 列×8 行 | `(64,110)` | 三武器独立图，行对应锁定的攻击方向 |

原生分层素材显示倍率为 `2×humanScale` 或 `2×actorScale`；当前分别是 `1.68`、`1.72`。
上身与下身共用同一个地面脚点，不把锚点迁到某一帧最前脚趾。
下身裁切来自原生 128 图的 `(32,64,96,128)` 区域，因此局部锚为 `(32,46)`。

## 4. 深度、阴影与分层

`world.lua` 将人物、未骑乘马、树、小景物和训练袋按脚点 y 从小到大绘制。
人物上下身必须作为同一个深度对象相邻绘制，不能分别插入排序队列。
地表先画，阴影独立画；环境和主体不带烘焙地面影，以免双影或换姿态时影跳。

`environment.lua` 使用树轮廓范围判断角色或马是否被挡，淡出的是整棵树。
当前树冠检测高度 208、半宽 48，再乘实例与全局树缩放；这是一种近似区域判断，不是像素级遮挡。
透明岩壁作为后景：逻辑图 `1280×260` 的实际实体底缘为 y202，早于最小活动脚点 y220。
移植为可穿行前景墙时，必须新增脚点排序或碰撞，不能沿用后景规则。
整场地表图不是可平铺纹理；扩大场景需重新制作地块系统。

下马后人物与马为两个实体：马留在原地，人物偏移到旁边；走近至 85 逻辑单位内可重骑。
当前下马偏移为 `(72,12)`，位置受地面边界约束。马有独立阴影与遮挡，不能在下马时删除马。

## 5. 面向、移动与攻击的独立状态

`faceDirection` 是人物/近战上身的面向，`moveDirection` 是原输入移动方向。
步行上下身都按 face 绘制，下身文件使用 `relative=(move-face)%8`：0 前进、4 后退、2/6 侧移。
这些是实际烘焙的后退/侧步轨迹，不能将腿转到 move 方向或倒播整身动画代替。

近战每招开招锁定 attack.direction，脚点仍按输入移动；contact 使用该帧移动后的脚点。
枪持续按住期间，包括射击节流空档，保持准星朝向。近战持续按住、收招和冷却也保持战斗步态。
上身 attack.elapsed 与下身 locomotionTime 分开，开始下一招不把腿重置到第 0 帧。
同一移动模式切普通/战斗时，按旧 rate/新 rate 重标时间以保留循环相位。

每帧核心顺序为：推进特效 → 输入/演示移动与瞄准 → 接触/冷却判定 → 动作节拍。
若把 contact 放在输入移动之前，会使用前一帧脚点，造成“已经进圈却没命中”。
攻击期间拒绝换装和上下马；蹲姿近战先起身，骑乘拒绝蹲伏。

## 6. 普通走跑与战斗步态

普通步态只用于步行、非蹲伏且不处于战斗的状态。
战斗判据覆盖 `attackRequested`、`combatFacing`、attack 对象及尚未结束的近战冷却。
普通走路使用较低抬脚、动态髋高和较直的支撑膝；跑步使用更大摆幅和抬脚。
普通触地相位为 `(-1,-1/3,1/3,1,1/3,-1/3)`，支撑回扫三段等距，避免匀速角色配不匀脚步。

| 当前播放参数 | 值 | 备注 |
| --- | --- | --- |
| 普通 walk/run rate | `16.2 / 21.6 FPS` | 当前 `adbd259`；不使用源说明的历史 18/24 |
| 普通横向 walk/run 速度 | `143.88192 / 235.605888` | 逻辑单位/秒；由实际步幅派生 |
| 战斗 walk/run/crouch rate | `14 / 16 / 6 FPS` | 保留原战斗节拍 |
| 战斗横向前走/后退速度 | `87.2592 / 68.06688` | 后退短步幅自然更慢 |
| 战斗横向跑速度 | `164.55936` | 普通提速不改变攻击移动 |
| 骑乘 walk/gallop 速度 | `250 / 390` | 独立于人物步幅计算 |

`Actions.Speed` 根据元数据计算速度，而不是直接使用配置中的旧 fallback 速度：

```text
projection = sqrt(sin(moveAngle)^2 + (0.66*cos(moveAngle))^2)
speed = nativeSpan * displayScale * frameRate / groundFrames / projection
```

`locomotion-meta.lua` 提供普通 stride/upperOffsets；`combat-handmeta.lua` 提供旧战斗 stride。
普通纵向/斜向速度由其各方向数据决定，表中的精确值仅代表横向。
动态髋偏移要同时作用于上身和真实主掌/枪原点；只移动图片会让枪脱手。

## 7. 关节与装备限制

`action_pose.py` 先把真实腕/脚端点约束到双骨关节内角 25–170° 的可达壳，再构造肘/膝。
只夹求解器的距离而不移动实际端点，会生成不真实的骨长。
双握装备必须整体平移主握与支撑握保持同轴；无法满足可达范围则明确报错。
相关报告测量实际渲染端点和骨长，不只检查参数。

砍刀、斧、矛与近战手臂一体烘焙。切刃检查使用真正长刃/斧外缘/矛尖，不能用刀背或任意最远点。
刀背暗、磨刃亮是辨识辅助；真正刃向仍由局部武器平面与接触轨迹确定。
枪管由运行时从真实主掌叠加。主握与支撑握距 `.12m`，掌半径 `.05m`，相对身体方向余角限 24°。
其支撑偏移约 `.0499m`，不是允许无限旋枪；枪元数据不再使用双腕平均中心。

## 8. 骑乘攻击与 16 向骑枪

马与坐腿底层按 move 绘制，上身按合法目标面向绘制；马保持原输入方向，不为了瞄准自动倒走。
骑手身体相对马最多左右 90°。近战每招锁八向，急转超过限制时取消挥击、提示并保留冷却，禁止取消后补命中。
骑近战为座高 `1.53m` 的专用下劈/下刺，不能简单把站立攻击贴高。

当前刀/斧/矛骑乘 reach 为 `50/47/76`，步行仍为 `55/50/84`。
命中先检查当前脚点地面扇区，再将 `mounted-melee-meta.lua` 的真实 contact2 切刃点
加实时 seatOffset，检查是否落入训练袋椭圆。袋中心高 37、半径 `(24,35)`；其他目标形状需重新定义。
当前 timing 为刀 `.60/.24/.74`、斧 `.82/.34/1.00`、矛 `.64/.26/.80` 秒，分别代表 duration/contact/cooldown。

骑枪私有 `mountedGunPose` 采用 16 个真实 22.5° 身体姿态，原 face/move 八向合同不变。
选择器扫描相对马不超过 90° 的候选，以每个候选真实主掌和当前座位升降求最小枪管余角。
绘制、枪原点和角度检查使用同一细分姿态。合法侧向身体可再使用 24° 的枪管余角，
因此射界不是按旧肩部参考点强行截断的总 90°；真正不可达的背后目标提示并拒射。
释放攻击、换装或上下马必须清理该私有姿态。

## 9. 世界与 HUD 的呈现链

世界 PNG 以 `NVG_IMAGE_NEAREST` 加载，NanoVG 世界 context 用 `nvgCreate(0)` 关闭边缘 AA。
`Texture2D` 创建 RGBA render target、单级 mip、`FILTER_NEAREST`，绘入 640×360 世界。
原生 `BorderImage` 最近邻放大；HUD 使用窗口尺寸透明 RT、独立 context 和上层面板。
两面板采用预乘 alpha 混合。标准 1280×720 是严格 2×2 像素；1024×768 适配为 1024×576，
上下各留边 96，非整数采样会交替分配 1/2 个屏幕像素。
`viewport.lua` 结果同时用于面板与鼠标映射；`ui:GetScale()` 是 Vector2，要分别处理 x/y。

当前 Runtime 的默认 UI shader 会对已经为显示 RGB 的 NanoVG RT 再编码一次 Gamma。
`presentation.lua` 注册项目局部 `scripts/rendering/Shaders/BLGL/UI.glsl`，保留 SDK 分支，
省去不需要的末级编码，并显式 reload 资源及 shaders。PNG 和 palette 不做反向补偿。
这适用于该独立进程的两个 RT 面板；不能无审查推广给同时显示普通线性资源的 UI。
退出恢复资源搜索优先级与缓存路径；已加入目录和已载 shader 留到该进程退出才释放。

## 10. 复现与迁移顺序

1. 保存小样的 scripts、assets、source、manifest 和 shader；使用相同固定相机合同。
2. 先用已有 PNG 启动新项目的最小静态场景，验证锚点、颜色、nearest、HUD 和留边。
3. 接 depth/独立马/阴影，再接 face/move、独立腿相位和真实手锚；不要一次改所有约定。
4. 逐项移植普通 stride、座位偏移、骑乘 cut 点与 16 向枪；调整相机或比例后重新生成元数据。
5. 最后重建业务状态、目标形状和碰撞，保留行为测试并用新项目 GPU 连帧验收。

进入小样目录，安装/提供兼容的官方 Runtime 后运行：

```powershell
.\launch-prototype.ps1 -Mode Test
.\launch-prototype.ps1 -Mode Capture
.\launch-prototype.ps1 -Mode Interactive
```

可用 `-RuntimePath` 指定自行安装的 Runtime 可执行文件；不借用其他项目的身份或配置。
当前验收版本为 `20260923-1717-bf3316f`。本分享包不包含官方 Runtime 安装库。

需要重烘时，先做 samples，再做完整图集；下面以小样目录为当前目录，`blender` 指已安装的 Blender：
母版脚本需要 Blender 自带 Python/bpy；普通打包和环境脚本使用具备 Pillow、NumPy 的 Python。
这些工具属于制作依赖，游戏启动无需 Blender。

```powershell
blender --background --factory-startup --python assets/source/bake_locomotion.py -- --samples
blender --background --factory-startup --python assets/source/bake_locomotion.py
python assets/source/pack_locomotion.py
```

combat、mounted melee 和 gun16 使用各自 bake/pack 入口，命令见 `assets/source/README.md`。
不要把 samples 当完整运行资产。固定母版、相机、PNG 和相应 metadata 必须来自同一生成版本。
当前日志实录有 31/31 Runtime 行为检查；GPU 截图与连续序列验证动作、握持、颜色和命中。
截图 CPU marker 存在呈现延迟，不能宣称它就是画面准确相位。诊断方法见 [troubleshooting.md](troubleshooting.md)。

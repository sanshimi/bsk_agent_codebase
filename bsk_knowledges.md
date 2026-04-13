# 基于 basilisk 代码生成的仿真场景配置知识库

## 脚本基础骨架
Basilisk（简称 bsk）遵循“进程-任务-模块”即“Process-Task-Module”的层级来管理航天仿真场景。一个最初级的航天仿真 bsk 脚本应该包含创建进程，创建任务，创建模块以及执行仿真四部分内容。针对仿真场景生成任务，特别规定了一个仿真场景脚本的基础模板函数，该模板配置了一个仿真场景的最基础构成，包括创建进程、创建任务、创建星历模块、仿真设置以及卸载星历内核五部分内容。

函数模板下所示，依赖三个核心参数变量，包括仿真步长 `simTask`, 仿真开始时间 `startTimeUTC` 和仿真时长 `simTime`。统一约定用户设置的时间参数的单位为秒，因此需要调用 `macros.sec2nano(...)` 函数进行单位转换。该模板确定的基础仿真场景为 仿真步长 10 秒，仿真开始时间为 2026 年 1 月 4 日 15 时整，仅为举例所用，实际需要修改为用户需求。

```python
from Basilisk.utilities import SimulationBaseClass, macros, simIncludeGravBody

def run():
    # 1. 创建进程
	scSim = SimulationBaseClass.SimBaseClass()
	scSim.SetProgressBar(False) # 关闭在命令行打印进度条
	simProcessName = "simProcess" # 用一个字符串变量存储进程名
	dynProcess = scSim.CreateNewProcess(simProcessName)

    # 2. 创建任务
	simTaskName = "simTask"
    timeStep = 10 # 存储仿真步长
    simTask = scSim.CreateNewTask(simTaskName, macros.sec2nano(timeStep))# 由于bsk 底层用纳秒作为基本单位，因此需要将秒转换为纳秒 
	dynProcess.addTask(simTask) # 任务注册到进程中

    # 3. 创建星历模块，利用星历模块可以设置仿真开始时间
	gravFactory = simIncludeGravBody.gravBodyFactory()
    startTimeUTC = "2026 January 04 15:00:00.0" # 存储仿真开始时间
	spiceObject = gravFactory.createSpiceInterface(time=startTimeUTC, epochInMsg=True)
	scSim.AddModelToTask(simTaskName, spiceObject) # 将模块注册到任务中

    # 4. 仿真设置
	scSim.InitializeSimulation() # 初始化仿真
    simTime = 60 # 存储仿真时长
	scSim.ConfigureStopTime(macros.sec2nano(simTime)) # 设置仿真时长
	scSim.ExecuteSimulation() # 执行仿真

    # 5. 卸载加载的星历内核
	gravFactory.unloadSpiceKernels()
```

## 向仿真场景脚本中添加天体
向仿真场景中添加天体依赖 `gravBodyFactory()` 类以及星历模块。一个航天仿真场景中常见包含的天体有太阳、地球、月球等。在创建仿真场景时，十分重要的是**要确定该场景基准系的中心天体是哪一个**。

以一个地月日系统仿真场景为例，该场景设定参考系的中心天体为地球。该功能被封装为以下函数，它支持链式或多次调用来构建复杂引力场。

```python
def add_celestial_body(gravFactory, body_name, is_central=False):
    """
    向仿真工厂中添加特定的天体，并设置其是否为中心天体。
    
    参数:
        gravFactory: 由 simIncludeGravBody.gravBodyFactory() 创建的工厂对象
        body_name (str): 天体名称，支持 'earth', 'moon', 'sun'等
        is_central (bool): 是否将该天体设置为场景的中心天体（基准系原点）
        
    返回:
        body: 创建的天体对象（可用于获取其 mu 等属性）
    """
    # 动态调用工厂方法，例如 gravFactory.createEarth()
    create_method = getattr(gravFactory, f"create{body_name.capitalize()}")
    body = create_method()
    
    # 设置中心天体属性
    body.isCentralBody = is_central
    
    return body
```

此外，示例中创建的 `earth` , `moon` 等类还有其他的用途，比如其中的 `mu` 属性就存储了该天体的引力常数值。这些其他用途将在别的小节介绍。本小节**只关注如何将天体添加到仿真场景中**。

## 向仿真场景中添加一颗围绕中心天体运动并由开普勒轨道六根数确定初始状态的卫星
向仿真场景中添加卫星依赖 `spacecraft` 模块与 `orbitalMotion` 工具库。创建一颗卫星对象后，还需要将其注册到仿真任务、绑定引力天体，才能让仿真引擎驱动该卫星运动。卫星的初始轨道状态由开普勒轨道六根数给出后，需通过 `orbitalMotion.elem2rv()` 转换为位置向量 $\mathbf{r}$ 和速度向量 $\mathbf{v}$，再写入卫星的 `hub` 属性完成初始化。

轨道六根数中，倾角 $i$、升交点赤经 $\Omega$、近地点辐角 $\omega$、真近点角 $f$ 均以角度为单位输入，但 bsk 内部使用弧度，因此必须乘以 `macros.D2R` 进行转换；半长轴 $a$ 的单位为米。将六根数转换为位置速度时，需要传入所围绕天体的引力常数 `mu`，该值由 `gravFactory.createEarth()`（或对应天体）返回的天体对象的 `.mu` 属性提供，因此**天体对象必须在卫星初始化之前创建**。

以向场景中添加一颗地球同步轨道（GEO）卫星为例，包含所有必要组成的代码片段如下：

```python
from Basilisk.utilities import SimulationBaseClass, macros, simIncludeGravBody, orbitalMotion
from Basilisk.simulation import spacecraft

# 前置：gravFactory 与 earth 已在"添加天体"步骤中创建
gravFactory = simIncludeGravBody.gravBodyFactory()
earth = gravFactory.createEarth()
earth.isCentralBody = True
spiceObject = gravFactory.createSpiceInterface(time=startTimeUTC, epochInMsg=True)
scSim.AddModelToTask(simTaskName, spiceObject)

# 1. 创建卫星对象并注册到仿真任务
geo = spacecraft.Spacecraft()
geo.ModelTag = "GEO"
gravFactory.addBodiesTo(geo)        # 绑定引力天体到卫星，卫星才会受这些天体的引力影响
scSim.AddModelToTask(simTaskName, geo)

# 2. 设置开普勒轨道六根数
oe_geo = orbitalMotion.ClassicElements()
oe_geo.a = 42164e3          # 半长轴 [m]
oe_geo.e = 5e-5             # 离心率 [-]
oe_geo.i = 0.05 * macros.D2R     # 轨道倾角 [rad]
oe_geo.Omega = 60.0 * macros.D2R # 升交点赤经 [rad]
oe_geo.omega = 0.0 * macros.D2R  # 近地点辐角 [rad]
oe_geo.f = 0.0 * macros.D2R      # 真近点角 [rad]

# 3. 由六根数转换为位置速度，并写入卫星初始状态
muEarth = earth.mu   # 从天体对象获取引力常数
r_geo, v_geo = orbitalMotion.elem2rv(muEarth, oe_geo)
geo.hub.r_CN_NInit = r_geo
geo.hub.v_CN_NInit = v_geo
```

此外，`geo.hub` 还有姿态相关的初始化属性，即 `sigma_BNInit`（MRP 初始姿态）和 `omega_BN_BInit`（初始角速度），在纯轨道运动场景中若不关心姿态可以不设置，bsk 默认为零值；若需要精确姿态则需显式赋值，例如：

```python
geo.hub.sigma_BNInit = [[0], [0], [0]]   # 初始姿态 MRP，零值表示与惯性系对齐
geo.hub.omega_BN_BInit = [[0], [0], [0]] # 初始本体角速度 [rad/s]
```

本小节**只关注如何添加一颗围绕中心天体运动的卫星**。若卫星围绕的是场景中的非中心天体（如地心系下的月心轨道卫星），这部分内容请去查看另一小节的介绍。

## 向仿真场景中添加一颗围绕非中心天体运动并由开普勒轨道六根数确定初始状态的卫星
向仿真场景中添加卫星依赖 `spacecraft` 模块与 `orbitalMotion` 工具库。创建一颗卫星对象后，还需要将其注册到仿真任务、绑定引力天体，才能让仿真引擎驱动该卫星运动。卫星的初始轨道状态由开普勒轨道六根数给出后，需通过 `orbitalMotion.elem2rv()` 转换为位置向量 $\mathbf{r}$ 和速度向量 $\mathbf{v}$，再写入卫星的 `hub` 属性完成初始化。

与围绕中心天体的卫星相比，核心差异在于 **bsk 的惯性系原点固定在中心天体上，所有卫星的 `hub.r_CN_NInit` 和 `hub.v_CN_NInit` 都必须是相对于中心天体、在中心惯性系下表达的**。开普勒六根数通过 `orbitalMotion.elem2rv()` 只能给出卫星相对于其所围绕天体的位置速度，因此还需要额外查询该非中心天体在仿真开始时刻相对于中心天体的位置速度，两者叠加后才能得到正确的初始状态。

非中心天体的位置速度通过 `spkRead()` 函数从 SPICE 内核中查询。`spkRead` 返回的单位是千米，必须乘以 1000 转换为米，否则初始轨道会出现量级错误。调用 `spkRead` 前还需要通过 `pyswice.furnsh_c()` 手动加载必要的历书内核（`de430.bsp`、`naif0012.tls`、`pck00010.tpc`），否则查询会报内核未找到的错误。

`spkRead` 的四个参数依次为：目标天体名称、查询时刻（与 `startTimeUTC` 格式一致）、参考坐标系（固定使用 `'J2000'`）、观察者天体名称（即中心天体）。返回值是长度为 6 的数组，前三位为位置，后三位为速度。

以在地心系场景下添加一颗月球低轨道（LLO）卫星为例，包含所有必要组成的代码片段如下：

```python
from Basilisk.utilities import SimulationBaseClass, macros, simIncludeGravBody, orbitalMotion
from Basilisk.simulation import spacecraft
from Basilisk.topLevelModules import pyswice
from Basilisk.utilities.pyswice_spk_utilities import spkRead

# 前置：gravFactory、earth、moon、spiceObject 已在"添加天体"步骤中创建
gravFactory = simIncludeGravBody.gravBodyFactory()
earth = gravFactory.createEarth()
earth.isCentralBody = True
moon = gravFactory.createMoon()
spiceObject = gravFactory.createSpiceInterface(time=startTimeUTC, epochInMsg=True)

# 加载 spkRead 所需的 SPICE 历书内核（必须在调用 spkRead 前完成）
pyswice.furnsh_c(spiceObject.SPICEDataPath + "de430.bsp")
pyswice.furnsh_c(spiceObject.SPICEDataPath + "naif0012.tls")
pyswice.furnsh_c(spiceObject.SPICEDataPath + "pck00010.tpc")
scSim.AddModelToTask(simTaskName, spiceObject)

# 1. 创建卫星对象并注册到仿真任务
llo = spacecraft.Spacecraft()
llo.ModelTag = "LLO"
gravFactory.addBodiesTo(llo)        # 绑定引力天体到卫星
scSim.AddModelToTask(simTaskName, llo)

# 2. 设置卫星相对于月球的开普勒轨道六根数
oe_llo = orbitalMotion.ClassicElements()
oe_llo.a = 1838e3           # 半长轴 [m]，月球半径 1737.4km + 轨道高度
oe_llo.e = 0.005            # 离心率 [-]
oe_llo.i = 10.0 * macros.D2R     # 轨道倾角 [rad]
oe_llo.Omega = 30.0 * macros.D2R # 升交点赤经 [rad]
oe_llo.omega = 0.0               # 近地点辐角 [rad]
oe_llo.f = 180.0 * macros.D2R   # 真近点角 [rad]

# 3. 在月心系下将六根数转换为位置速度
muMoon = moon.mu    # 从天体对象获取月球引力常数
r_llo_M, v_llo_M = orbitalMotion.elem2rv(muMoon, oe_llo)

# 4. 查询月球在仿真开始时刻相对于地球的位置速度（单位需从 km 转换为 m）
moonState = 1000 * spkRead('moon', startTimeUTC, 'J2000', 'earth')
r_moon = moonState[0:3]   # 月球位置 [m]，地心惯性系
v_moon = moonState[3:6]   # 月球速度 [m/s]，地心惯性系

# 5. 叠加得到卫星在地心惯性系下的初始状态并写入
llo.hub.r_CN_NInit = r_moon + r_llo_M
llo.hub.v_CN_NInit = v_moon + v_llo_M
```

本小节**只关注如何将非中心天体轨道卫星的初始状态转换到中心惯性系**。如果场景中有多颗围绕同一非中心天体运动的卫星，`spkRead` 只需调用一次，将结果复用即可。

## 向仿真场景中添加一颗由 SPICE 星历确定的卫星
向仿真场景中添加一颗由 SPICE 星历确定的卫星，依赖 `spacecraft` 模块、`spiceObject.loadSpiceKernel(...)` 以及 `spiceObject.addSpacecraftNames(...)`。与前两节“由轨道六根数初始化状态”不同，本节卫星的位置速度不再由 `orbitalMotion.elem2rv()` 计算，而是由 SPICE 星历模块在仿真过程中持续输出参考轨道状态。

SPICE 路径下最关键的步骤有四个：

1. 使用 `loadSpiceKernel` 加载目标卫星对应的 `.bsp` 文件。
2. 使用 `zeroBase` 设置加载星历内核的参考系中心天体名称（如 "Earth"、"Moon"），以确保输出的轨道状态与仿真场景的基准系一致。
3. 使用 `addSpacecraftNames(messaging.StringVector([...]))` 声明该 `.bsp` 中要读取的卫星 ID。
4. 让卫星通过 `transRefInMsg.subscribeTo(...)` 订阅星历输出消息。

其中，`addSpacecraftNames` 的顺序会决定 `transRefStateOutMsgs` 的索引顺序。若只添加一颗卫星，通常订阅 `transRefStateOutMsgs[0]`；若添加多颗卫星，必须保证“ID 列表顺序”和“订阅索引”一一对应。

此外，代码中 `satelliteId` 必须与 `.bsp` 文件内的目标 ID 一致，否则不会得到对应轨道状态。若 ID 不匹配，常见现象是卫星没有按预期轨道运动或出现消息绑定失败。

以向场景中添加一颗由 SPICE 星历定义的 DRO 卫星为例，包含所有必要组成的代码片段如下：

```python
from Basilisk.utilities import SimulationBaseClass, macros, simIncludeGravBody
from Basilisk.simulation import spacecraft
from Basilisk.architecture import messaging

# 前置：场景基本骨架与天体创建已完成
startTimeUTC = "2029 March 12 01:00:00.0"
bspFile = "DRO_2029.bsp"
satelliteId = "-2000"

spiceObject = gravFactory.createSpiceInterface(
	time=startTimeUTC,
	epochInMsg=True
)

# 1. 加载目标卫星的 SPICE 星历内核
spiceObject.loadSpiceKernel(bspFile, "")
spiceObject.zeroBase = "Earth" 

# 2. 声明要从该内核中读取的卫星 ID
spiceObject.addSpacecraftNames(messaging.StringVector([satelliteId]))
scSim.AddModelToTask(simTaskName, spiceObject)

# 3. 创建卫星并订阅 SPICE 输出状态
dro = spacecraft.Spacecraft()
dro.ModelTag = "DRO_2029"
gravFactory.addBodiesTo(dro)
dro.transRefInMsg.subscribeTo(spiceObject.transRefStateOutMsgs[0])
scSim.AddModelToTask(simTaskName, dro)
```

本小节**只关注如何将由 SPICE 星历定义的轨道状态注入卫星**。若需要在同一场景中同时存在“SPICE 星历卫星”和“六根数卫星”，可分别按本节与前两节方法创建，并统一注册到同一仿真任务中。

## 输出基础场景配置文件
输出基础场景配置文件依赖 `vizSupport` 模块。该模块用于将当前仿真中的卫星、轨道与环境信息导出为 Vizard 可加载的可视化配置数据，便于后续回放和分析。

在 bsk 场景中，推荐将可视化输出放在“卫星与环境创建完成之后、仿真执行之前”，并用 `if vizSupport.vizFound:` 做条件判断。这样即使本地未安装可视化组件，也不会影响仿真主流程执行。

本节代码模式核心由两部分组成：

1. `vizSupport.vizFound`：检查当前环境是否可用可视化能力。
2. `vizSupport.enableUnityVisualization(...)`：注册可视化输出并设置输出文件名。

其中，`enableUnityVisualization` 常用参数含义如下：

- 第 1 个参数 `scSim`：当前仿真对象。
- 第 2 个参数 `simTaskName`：可视化绑定的任务名。
- 第 3 个参数 `[satellite_list]`：需要输出到可视化文件的卫星列表。
- 关键字参数 `saveFile=vizFileName`：输出配置文件名（不带路径时默认在当前工作目录生成）。

以输出一个包含单星场景可视化配置文件为例，代码片段如下：

```python
from Basilisk.utilities import vizSupport

# 前置：场景、任务、卫星对象已创建
vizFileName = "DRO_2029_Visualization"

if vizSupport.vizFound:
	vizSupport.enableUnityVisualization(
		scSim,
		simTaskName,
		[dro],
		saveFile=vizFileName
	)
```

`saveFile` 建议使用能反映场景语义的名称（如任务名、轨道类型、历元时间），便于后续批量管理与回放检索。

本小节**只关注如何配置可视化输出文件**。可视化配置完成后，仿真执行流程仍保持不变，即按 `InitializeSimulation -> ConfigureStopTime -> ExecuteSimulation` 顺序运行。
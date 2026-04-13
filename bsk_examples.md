# Basilisk 场景示例（IR 与 Code 对照）

## 示例1

### IR

```json
{
  "version": "0.1.3",
  "object": [
    {
      "name": "DRO_2029",
      "type": "spacecraft",
      "ref": ["DRO_spice"]
    }
  ],
  "environment": {
    "reference_frame": "J2000",
    "central_body": "earth",
    "non_central_bodies": ["moon", "sun"]
  },
  "time": {
    "start_time_utc": "2029 March 12 01:00:00.0",
    "duration": 518400.0,
    "step": 60.0
  },
  "mission": [
    {
      "name": "DRO_spice",
      "type": "spice",
      "description": "DRO卫星星历任务，通过BSP文件定义轨道参数",
      "elements": {
        "bsp_file_path": "DRO_2029.bsp",
        "satellite_id": "-2000"
      }
    }
  ]
}
```

### Code

```python
from Basilisk.utilities import SimulationBaseClass, macros, simIncludeGravBody, vizSupport
from Basilisk.simulation import spacecraft
from Basilisk.architecture import messaging


def run(
        timeStep=60.0,
        simTime=518400.0,
        startTimeUTC="2029 March 12 01:00:00.0",
        bspFile="DRO_2029.bsp",
        satelliteId="-2000",
        vizFileName="DRO_2029_Visualization"
    ):
    # 1. 创建仿真场景基础
    scSim = SimulationBaseClass.SimBaseClass()
    scSim.SetProgressBar(True) # 命令行打印进度条
    simProcessName = "simProcess"
    dynProcess = scSim.CreateNewProcess(simProcessName) # 创建仿真进程
    simTaskName = "simTask"
    dynProcess.addTask(scSim.CreateNewTask(simTaskName, macros.sec2nano(timeStep))) # 创建仿真任务并设置步长
    startTimeUTC = "2029 March 12 01:00:00.0" # 设置仿真开始时间

    # 2. 环境与星历
    gravFactory = simIncludeGravBody.gravBodyFactory()
    earth = gravFactory.createEarth() # 创建地球，并设置为中心体
    earth.isCentralBody = True
    gravFactory.createMoon()
    gravFactory.createSun()
    spiceObject = gravFactory.createSpiceInterface(
        time=startTimeUTC,
        epochInMsg=True
    ) # 创建星历模块
    spiceObject.loadSpiceKernel(bspFile, "") # 加载DRO卫星的星历文件
    spiceObject.zeroBase = "Earth" # 设置参考系中心天体名称
    spiceObject.addSpacecraftNames(messaging.StringVector([satelliteId])) # 绑定卫星id到SPICE
    scSim.AddModelToTask(simTaskName, spiceObject) # 注册星历模块到仿真任务

    # 3. 创建卫星模块
    dro = spacecraft.Spacecraft()
    dro.ModelTag = "DRO_2029"
    gravFactory.addBodiesTo(dro) # 将环境天体添加到卫星模型中
    dro.transRefInMsg.subscribeTo(spiceObject.transRefStateOutMsgs[0]) # 卫星订阅SPICE提供的状态信息
    scSim.AddModelToTask(simTaskName, dro) # 注册卫星模块到仿真任务

    # 4. 可视化设置
    if vizSupport.vizFound:
        vizSupport.enableUnityVisualization(
            scSim,
            simTaskName,
            [dro],
            saveFile=vizFileName
        )

    # 5. 运行仿真
    scSim.InitializeSimulation()
    scSim.ConfigureStopTime(macros.sec2nano(simTime))
    scSim.ExecuteSimulation()

    # 6. 卸载SPICE内核
    gravFactory.unloadSpiceKernels()

if __name__ == "__main__":
    run()
```

## 示例2

### IR

```json
{
  "version": "0.1.3",
  "object": [
    {"name": "GEO", "type": "spacecraft", "ref": ["GEO_orbit"]},
    {"name": "LLO", "type": "spacecraft", "ref": ["LLO_orbit"]}
  ],
  "environment": {
    "reference_frame": "J2000",
    "central_body": "earth",
    "non_central_bodies": ["moon"]
  },
  "time": {
    "start_time_utc": "2026 January 04 15:00:00.0",
    "duration": 1728000.0,
    "step": 120.0
  },
  "mission": [
    {
      "name": "GEO_orbit",
      "type": "orbit",
      "description": "GEO satellite orbit design task, orbiting Earth, defined by classic orbital elements",
      "elements": {
        "a": 42164000.0,
        "e": 5.0,
        "i": 2.865,
        "Omega": 60.0,
        "omega": 0.0,
        "f": 0.0
      }
    },
    {
      "name": "LLO_orbit",
      "type": "orbit",
      "description": "LLO satellite orbit design task, orbiting Moon, defined by classic orbital elements with Moon-centered state",
      "elements": {
        "a": 1838000.0,
        "e": 0.005,
        "i": 10.0,
        "Omega": 30.0,
        "omega": 0.0,
        "f": 180.0
      }
    }
  ]
}
```

### Code

```python
from Basilisk.utilities import (
    SimulationBaseClass,
    macros,
    orbitalMotion,
    simIncludeGravBody,
    vizSupport
)
from Basilisk.simulation import spacecraft
from Basilisk.topLevelModules import pyswice
from Basilisk.utilities.pyswice_spk_utilities import spkRead


def run(
        timeStep=120.0,
        simTime=1728000.0,
        startTimeUTC="2026 January 04 15:00:00.0",
        vizFileName="GEO_LLO_Visualization"
    ):
    # 1. 创建仿真场景基础
    scSim = SimulationBaseClass.SimBaseClass()
    scSim.SetProgressBar(True) # 命令行打印进度条
    simProcessName = "simProcess"
    dynProcess = scSim.CreateNewProcess(simProcessName) # 创建仿真进程
    simTaskName = "simTask"
    dynProcess.addTask(scSim.CreateNewTask(simTaskName, macros.sec2nano(timeStep))) # 创建仿真任务并设置步长
    startTimeUTC = "2026 January 04 15:00:00.0"

    # 2. 环境与星历
    gravFactory = simIncludeGravBody.gravBodyFactory()
    earth = gravFactory.createEarth() # 创建地球，并设置为中心体
    earth.isCentralBody = True
    moon = gravFactory.createMoon()
    moon.isCentralBody = False

    spiceObject = gravFactory.createSpiceInterface(
        time=startTimeUTC,
        epochInMsg=True
    ) # 创建星历模块
    pyswice.furnsh_c(spiceObject.SPICEDataPath + "de430.bsp")
    pyswice.furnsh_c(spiceObject.SPICEDataPath + "naif0012.tls")
    pyswice.furnsh_c(spiceObject.SPICEDataPath + "pck00010.tpc")

    scSim.AddModelToTask(simTaskName, spiceObject) # 注册星历模块到仿真任务

    # 3. 创建卫星模块
    # 由轨道六根数定义的卫星，并且绕场景中心天体运动
    geo = spacecraft.Spacecraft()
    geo.ModelTag = "GEO"
    gravFactory.addBodiesTo(geo)
    scSim.AddModelToTask(simTaskName, geo)

    oe_geo = orbitalMotion.ClassicElements()
    oe_geo.a = 42164e3
    oe_geo.e = 5e-5
    oe_geo.i = 0.05 * macros.D2R
    oe_geo.Omega = 60 * macros.D2R
    oe_geo.omega = 0.0
    oe_geo.f = 0.0

    muEarth = earth.mu
    r_geo, v_geo = orbitalMotion.elem2rv(muEarth, oe_geo)

    geo.hub.r_CN_NInit = r_geo
    geo.hub.v_CN_NInit = v_geo
    geo.hub.sigma_BNInit = [[0], [0], [0]]
    geo.hub.omega_BN_BInit = [[0], [0], [0]]

    # 由轨道六根数定义的卫星，并且绕场景非中心天体运动
    llo = spacecraft.Spacecraft()
    llo.ModelTag = "LLO"
    gravFactory.addBodiesTo(llo)
    scSim.AddModelToTask(simTaskName, llo)

    oe_llo = orbitalMotion.ClassicElements()
    oe_llo.a = 1838e3
    oe_llo.e = 0.005
    oe_llo.i = 10 * macros.D2R
    oe_llo.Omega = 30 * macros.D2R
    oe_llo.omega = 0.0
    oe_llo.f = 180 * macros.D2R

    muMoon = moon.mu

    rLLO_M, vLLO_M = orbitalMotion.elem2rv(muMoon, oe_llo)

    # 获取初始时刻月球在地心系下的位置速度 (基于 SPICE)
    moonState = 1000 * spkRead(
        'moon',
        startTimeUTC,
        'J2000',
        'earth'
    )

    rMoon = moonState[0:3]
    vMoon = moonState[3:6]

    # LLO 的初始状态是相对于月球的，因此需要叠加月球在地心系下的状态
    llo.hub.r_CN_NInit = rMoon + rLLO_M
    llo.hub.v_CN_NInit = vMoon + vLLO_M
    llo.hub.sigma_BNInit = [[0], [0], [0]]
    llo.hub.omega_BN_BInit = [[0], [0], [0]]

    # 4. 可视化设置
    if vizSupport.vizFound:
        vizSupport.enableUnityVisualization(
            scSim,
            simTaskName,
            [geo, llo],
            saveFile=vizFileName
        )

    # 5. 运行仿真
    scSim.InitializeSimulation()
    scSim.ConfigureStopTime(macros.sec2nano(simTime))  # ✅ 转换为纳秒，20天
    scSim.ExecuteSimulation()

    # 6. 卸载SPICE内核
    gravFactory.unloadSpiceKernels()

if __name__ == "__main__":
    run()
```

## 示例3

### IR

```json
{
  "version": "0.1.3",
  "object": [
    {"name": "LEO1", "type": "spacecraft", "ref": ["LEO1_spice"]},
    {"name": "MoonSat_200", "type": "spacecraft", "ref": ["MoonSat_200_orbit"]},
    {"name": "MoonSat_850", "type": "spacecraft", "ref": ["MoonSat_850_orbit"]},
    {"name": "MoonSat_1500", "type": "spacecraft", "ref": ["MoonSat_1500_orbit"]}
  ],
  "environment": {
    "reference_frame": "J2000",
    "central_body": "earth",
    "non_central_bodies": ["moon"]
  },
  "time": {
    "start_time_utc": "2028 January 01 00:00:00.0",
    "duration": 86400,
    "step": 60
  },
  "mission": [
    {
      "name": "LEO1_spice",
      "type": "spice",
      "description": "LEO1 satellite ephemeris task, defined by BSP file",
      "elements": {
        "bsp_file_path": "LEO1.bsp",
        "satellite_id": "1",
        "zero_base": "earth"
      }
    },
    {
      "name": "MoonSat_200_orbit",
      "type": "orbit",
      "description": "200 km altitude lunar orbiter design task, classic orbital elements with Moon-centered state superposition",
      "elements": {
        "a": 1937400,
        "e": 0,
        "i": 1.5,
        "Omega": 0,
        "omega": 0,
        "f": 0
      }
    },
    {
      "name": "MoonSat_850_orbit",
      "type": "orbit",
      "description": "850 km altitude lunar orbiter design task, classic orbital elements with Moon-centered state superposition",
      "elements": {
        "a": 2587400,
        "e": 0,
        "i": 1.5,
        "Omega": 0,
        "omega": 0,
        "f": 120
      }
    },
    {
      "name": "MoonSat_1500_orbit",
      "type": "orbit",
      "description": "1500 km altitude lunar orbiter design task, classic orbital elements with Moon-centered state superposition",
      "elements": {
        "a": 3237400,
        "e": 0,
        "i": 1.5,
        "Omega": 0,
        "omega": 0,
        "f": 240
      }
    }
  ]
}
```

### Code

```python
from Basilisk.utilities import (
   SimulationBaseClass,
   macros,
   simIncludeGravBody,
   vizSupport,
   orbitalMotion
)
from Basilisk.simulation import spacecraft
from Basilisk.architecture import messaging
from Basilisk.topLevelModules import pyswice
from Basilisk.utilities.pyswice_spk_utilities import spkRead


def setupOriginalLEO(scSim, simTaskName, gravFactory, spiceObject):
   """
   配置原始LEO卫星（通过SPICE星历定义）
   """
   leo1 = spacecraft.Spacecraft()
   leo1.ModelTag = "LEO1"
   gravFactory.addBodiesTo(leo1)
   leo1.transRefInMsg.subscribeTo(spiceObject.transRefStateOutMsgs[0])
   scSim.AddModelToTask(simTaskName, leo1)

   return [leo1]


def setupMoonSatellites(scSim, simTaskName, gravFactory, moon, startTimeUTC):
   """
   配置三个绕月航天器（由轨道六根数定义，并转换到地心惯性系）
   """
   muMoon = moon.mu
   moonRad = 1737.4e3  # 月球半径 [m]

   # 轨道设计参数
   altitudes = [200e3, 850e3, 1500e3]   # 轨道高度 [m]
   inclinations = [1.5, 1.5, 1.5]       # 轨道倾角 [deg]
   omegas = [0.0, 0.0, 0.0]             # 升交点赤经 [deg]
   fai = [0.0, 120.0, 240.0]            # 真近点角 [deg]

   moonSats = []

   # 获取初始时刻月球在地心系下的位置速度 (基于 SPICE)
   moonState = 1000 * spkRead('moon', startTimeUTC, 'J2000', 'earth')
   rMoon_N = moonState[0:3]
   vMoon_N = moonState[3:6]

   for i in range(3):
       tag = f"MoonSat_{int(altitudes[i]/1000)}"
       sc = spacecraft.Spacecraft()
       sc.ModelTag = tag
       gravFactory.addBodiesTo(sc)

       # 设计轨道六根数
       oe = orbitalMotion.ClassicElements()
       oe.a = moonRad + altitudes[i]
       oe.e = 0.0
       oe.i = inclinations[i] * macros.D2R
       oe.Omega = omegas[i] * macros.D2R
       oe.omega = 0.0
       oe.f = fai[i] * macros.D2R

       # 转换为月心惯性系下的状态，并叠加月球在地心系下的状态
       r_sc_M, v_sc_M = orbitalMotion.elem2rv(muMoon, oe)
       sc.hub.r_CN_NInit = rMoon_N + r_sc_M
       sc.hub.v_CN_NInit = vMoon_N + v_sc_M
       scSim.AddModelToTask(simTaskName, sc)
       moonSats.append(sc)

   return moonSats


def run(
       timeStep=60.0,
       simTime=86400.0,
       startTimeUTC="2028 January 01 00:00:00.0",
       bspFile="LEO1.bsp",
       satelliteId="1",
       vizFileName="Earth_Moon_Combined_Visualization"
   ):
   # 1. 创建仿真场景基础
   scSim = SimulationBaseClass.SimBaseClass()
   scSim.SetProgressBar(True)  # 命令行打印进度条
   simProcessName = "simProcess"
   dynProcess = scSim.CreateNewProcess(simProcessName)  # 创建仿真进程
   simTaskName = "simTask"
   dynProcess.addTask(scSim.CreateNewTask(simTaskName, macros.sec2nano(timeStep)))  # 创建仿真任务并设置步长

   # 2. 环境与星历
   gravFactory = simIncludeGravBody.gravBodyFactory()
   earth = gravFactory.createEarth()  # 创建地球，并设置为中心体
   earth.isCentralBody = True
   moon = gravFactory.createMoon()  # 创建月球
   moon.isCentralBody = False

   spiceObject = gravFactory.createSpiceInterface(
       time=startTimeUTC,
       epochInMsg=True
   )  # 创建星历模块
   spiceObject.loadSpiceKernel(bspFile, "")  # 加载LEO卫星的星历文件
   spiceObject.addSpacecraftNames(messaging.StringVector([satelliteId]))  # 绑定卫星id到SPICE

   # 加载月球与历元内核
   pyswice.furnsh_c(spiceObject.SPICEDataPath + "de430.bsp")
   pyswice.furnsh_c(spiceObject.SPICEDataPath + "naif0012.tls")
   pyswice.furnsh_c(spiceObject.SPICEDataPath + "pck00010.tpc")

   scSim.AddModelToTask(simTaskName, spiceObject)  # 注册星历模块到仿真任务

   # 3. 创建卫星模块
   # LEO卫星 - 通过SPICE星历定义
   leoSats = setupOriginalLEO(scSim, simTaskName, gravFactory, spiceObject)

   # 绕月卫星 - 由轨道六根数定义，并转换到地心惯性系
   moonSats = setupMoonSatellites(scSim, simTaskName, gravFactory, moon, startTimeUTC)

   # 4. 可视化设置
   if vizSupport.vizFound:
       allSats = leoSats + moonSats
       vizSupport.enableUnityVisualization(
           scSim,
           simTaskName,
           allSats,
           saveFile=vizFileName
       )

   # 5. 运行仿真
   scSim.InitializeSimulation()
   scSim.ConfigureStopTime(macros.sec2nano(simTime))
   scSim.ExecuteSimulation()

   # 6. 卸载SPICE内核
   gravFactory.unloadSpiceKernels()


if __name__ == "__main__":
   run()
```

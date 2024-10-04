# 1. LodestoneLib注册粒子
## in startup_scripts

```js
//priority: 9999
//导入需要java包
const $LodestoneParticleRegistry = Java.loadClass('team.lodestar.lodestone.registry.common.particle.LodestoneParticleRegistry')
const $LodestoneParticleType = Java.loadClass('team.lodestar.lodestone.systems.particle.type.LodestoneParticleType')
const $LodestoneParticleType$Factory = Java.loadClass('team.lodestar.lodestone.systems.particle.type.LodestoneParticleType$Factory')
const $LodestoneScreenParticleRegistry = Java.loadClass('team.lodestar.lodestone.registry.common.particle.LodestoneScreenParticleRegistry')
const $LodestoneScreenParticleType = Java.loadClass('team.lodestar.lodestone.systems.particle.type.LodestoneScreenParticleType')
const $LodestoneLib = Java.loadClass('team.lodestar.lodestone.LodestoneLib')
const $LodestoneScreenParticleType$Factory = Java.loadClass('team.lodestar.lodestone.systems.particle.type.LodestoneScreenParticleType$Factory')

//在注册表添加ParticleType
let LAVA_PARTICLE = $LodestoneParticleRegistry.PARTICLES.register("lava", () => new $LodestoneParticleType())

let NAUTILUS_PARTICLE = $LodestoneParticleRegistry.PARTICLES.register("nautilus", () => new $LodestoneParticleType())

let CLOUD_PARTICLE = $LodestoneParticleRegistry.PARTICLES.register("cloud", () => new $LodestoneParticleType())

//ParticleType列表和ScreenParticleType列表方便后续批量注册
const PARTICLE_LIST = [LAVA_PARTICLE,NAUTILUS_PARTICLE,CLOUD_PARTICLE]
const PARTICLE_LIST$SCREEN = [
    "lava",
    "nautilus",
    "cloud"
]

//监听RegisterParticleProvidersEvent来注册注册表里面的粒子
//不使用ForgeEvents来监听是因为优先级不够，ForgeModEvents可以监听游戏初始化时的注册事件
ForgeModEvents.onEvent("net.minecraftforge.client.event.RegisterParticleProvidersEvent", (event) => {
    registerParticleFactory$Screen(event,PARTICLE_LIST$SCREEN)
    registerParticleFactory(event,PARTICLE_LIST)
})

//global变量方便跨脚本调用粒子
global.LAVA_PARTICLE = LAVA_PARTICLE
global.NAUTILUS_PARTICLE = NAUTILUS_PARTICLE
global.CLOUD_PARTICLE = CLOUD_PARTICLE


//批量注册ScreenParticle方法
/**
 * 
 * @param {Internal.RegisterParticleProvidersEvent} event 
 * @param {String[]} List 
 */
let registerParticleFactory$Screen = (event,List) => {//TODO maybe use event?
    List.forEach(e => {
        $LodestoneScreenParticleRegistry.registerProvider(
            $LodestoneScreenParticleRegistry.registerType(new $LodestoneScreenParticleType()), 
            new $LodestoneScreenParticleType$Factory(
                $LodestoneScreenParticleRegistry.getSpriteSet(
                    $LodestoneLib.lodestonePath(e)
                )
            )
        );
    });
}

//批量注册ParticleType方法
/**
 * 
 * @param {Internal.RegisterParticleProvidersEvent} event 
 * @param {Internal.RegistryObject<Internal.LodestoneParticleType>[]} List
 */
let registerParticleFactory = (event,List) => {
    List.forEach(e => {
        event.registerSpriteSet(
            e.get(), 
            () => new $LodestoneParticleType$Factory(
                $LodestoneScreenParticleRegistry.getSpriteSet(
                    $LodestoneLib.lodestonePath(
                        e.id.getPath()
                    )
                )
            )
        );
    })
}

```


# 2. 让玩家发送粒子数据到Client的粒子类
以下代码来自[kjspkg](https://kjspkglookup.modernmodpacks.site/p/?id=lodestone-js),因为没有k6版本，所以便写下
## in server_scripts
```js
//priority: 9999


//function + this 实现kjs的类
/**
 * 
 * @param {Internal.ExplosionEventJS$After} event 
 */
function Particle(event) {
    const { level } = event
    let particleType
    let lifetime = 100
    let transparencyData = [0, 0]
    let scaleData = [1, 1]
    let colorStart = [0, 0, 0]
    let colorEnd = [0, 0, 0]
    let position = [0, 0, 0]
    let motion = [0, 0, 0]
    let randomMotion = [0, 0, 0]
    let randomOffset = 0
    let count = 1
    this.type = function (type) {
        return particleType = type
    }
    this.lifetime = function (time) {
        return lifetime = time
    }
    this.transparencyData = function (start, end) {
        return transparencyData = [start, end]
    }
    this.scaleData = function (start, end) {
        return scaleData = [start, end]
    }
    this.colorData = function (start, end) {
        colorStart = start
        colorEnd = end
        return colorStart, colorEnd
    }
    this.position = function (x, y, z) {
        return position = [x, y, z]
    }
    this.motion = function (x, y, z) {
        return motion = [x, y, z]
    }
    this.randomMotion = function (x, y, z) {
        return randomMotion = [x, y, z]
    }
    this.randomOffset = function (amount) {
        return randomOffset = amount
    }
    this.spawn = function (amount) {
        count = amount
        level.getPlayers().forEach(player => {
            /**@type {Internal.Player} */(player).sendData('particle', {
                type: particleType,
                x: position[0],
                y: position[1],
                z: position[2],
                motionX: motion[0],
                motionY: motion[1],
                motionZ: motion[2],
                randomMotionX: randomMotion[0],
                randomMotionY: randomMotion[1],
                randomMotionZ: randomMotion[2],
                randomOffset: randomOffset,
                count: count,
                lifetime: lifetime,
                scaleStart: scaleData[0],
                scaleEnd: scaleData[1],
                colorStart: colorStart,
                colorEnd: colorEnd,
                transparencyStart: transparencyData[0],
                transparencyEnd: transparencyData[1]
            })
        })
    }
}
```

# 3. Client接受粒子数据
同上，以下代码来自[kjspkg](https://kjspkglookup.modernmodpacks.site/p/?id=lodestone-js),因为没有k6版本，所以便写下
## in client_scripts
```js
const $WorldParticleBuilder = Java.loadClass('team.lodestar.lodestone.systems.particle.builder.WorldParticleBuilder')
const $LodestoneParticleRegistry = Java.loadClass('team.lodestar.lodestone.registry.common.particle.LodestoneParticleRegistry')
const $GenericParticleData = Java.loadClass('team.lodestar.lodestone.systems.particle.data.GenericParticleData')
const $ColorParticleData = /**@type {Internal.ColorParticleData} */(Java.loadClass('team.lodestar.lodestone.systems.particle.data.color.ColorParticleData'))
const $SimpleParticleOptions = Java.loadClass('team.lodestar.lodestone.systems.particle.SimpleParticleOptions')
const $LodestoneWorldParticleRenderType = Java.loadClass('team.lodestar.lodestone.systems.particle.render_types.LodestoneWorldParticleRenderType')
const $Color = Java.loadClass('java.awt.Color')
NetworkEvents.dataReceived("particle",event=>{
    const { level, data } = event
    let type
    switch (data.type) {
        case 'WISP': {
            type = $LodestoneParticleRegistry.WISP_PARTICLE //这里是Lodestone自带的的粒子
            break
        };
        case 'SMOKE': {
            type = $LodestoneParticleRegistry.SMOKE_PARTICLE
            break
        };
        case 'SPARKLE': {
            type = $LodestoneParticleRegistry.SPARKLE_PARTICLE
            break
        };
        case 'TWINKLE': {
            type = $LodestoneParticleRegistry.TWINKLE_PARTICLE
            break
        };
        case 'STAR': {
            type = $LodestoneParticleRegistry.STAR_PARTICLE
            break
        }
        case 'LAVA': {
            type = global.LAVA_PARTICLE //这里是最开始我说的“global变量方便跨脚本调用粒子”
            break
        }
        case 'NAUTILUS': {
            type = global.NAUTILUS_PARTICLE
            break
        }
        case 'CLOUD': {
            type = global.CLOUD_PARTICLE
            break
        }
    }
    const colorStart = data.colorStart
    const colorEnd = data.colorEnd
    console.log(global.NAUTILUS_PARTICLE.get())
    $WorldParticleBuilder.create(type)
        .setTransparencyData($GenericParticleData.create(data.transparencyStart, data.transparencyEnd).build())
        .setScaleData($GenericParticleData.create(data.scaleStart, data.scaleEnd).build())
        .setColorData($ColorParticleData.create(new $Color(colorStart[0] / 255, colorStart[1] / 255, colorStart[2] / 255), new $Color(colorEnd[0] / 255, colorEnd[1] / 255, colorEnd[2] / 255)).build())
        .setLifetime(data.lifetime)
        .setRandomOffset(data.randomOffset)
        .addMotion(data.motionX, data.motionY, data.motionZ)
        .setRandomMotion(data.randomMotionX, data.randomMotionY, data.randomMotionZ)
        .setDiscardFunction($SimpleParticleOptions.ParticleDiscardFunctionType.INVISIBLE)
        .setRenderType($LodestoneWorldParticleRenderType.LUMITRANSPARENT)
        .repeat(level, data.x, data.y, data.z, data.count)
})
```

# 4. 使用它 (以爆炸后事件为例)
## in server_scripts
```js
let $ParticleTypes = Java.loadClass('net.minecraft.core.particles.ParticleTypes') 
LevelEvents.afterExplosion(event => {
    const { x, y, z } = event
    let count = 200
    event.affectedBlocks.forEach(block => {
        if (block.id == 'minecraft:air') { return }
        count++
    })
    let SmokeParticle = new Particle(event)
    // console.log($ParticleTypes.LAVA.class)
    SmokeParticle.type('CLOUD')
    SmokeParticle.colorData([160,80,40], [215,115,51].map(e => e * Client.partialTick))
    SmokeParticle.lifetime(count / 100 * 50)
    SmokeParticle.motion(0, 0.05, 0)
    SmokeParticle.position(x, y, z)
    SmokeParticle.randomMotion(0.05, 0.05, 0.05)
    SmokeParticle.randomOffset(count / 60)
    SmokeParticle.scaleData(0.5, 0)
    SmokeParticle.transparencyData(0.5, 0)
    SmokeParticle.spawn(count * 20)
})
```


## 从1.4.3.1更新到1.6.2.1
### in startup_scripts
对原来的注册,我分为了两个文件,都在startup_scripts中
```js
// startup_scripts/particlere.js

const $RegisterParticleProvidersEvent = Java.loadClass('net.minecraftforge.client.event.RegisterParticleProvidersEvent');
const $RegistryObject = Java.loadClass('net.minecraftforge.registries.RegistryObject');
const $LodestoneWorldParticleType = Java.loadClass('team.lodestar.lodestone.systems.particle.world.type.LodestoneWorldParticleType');
const $LodestoneParticleType = $LodestoneWorldParticleType;
const $LodestoneScreenParticleRegistry = Java.loadClass('team.lodestar.lodestone.registry.common.particle.LodestoneScreenParticleRegistry');
const $LodestoneParticleType$Factory = Java.loadClass('team.lodestar.lodestone.systems.particle.world.type.LodestoneWorldParticleType$Factory');
const $LodestoneScreenParticleType = Java.loadClass('team.lodestar.lodestone.systems.particle.screen.LodestoneScreenParticleType');
const $LodestoneLib = Java.loadClass('team.lodestar.lodestone.LodestoneLib')
const $LodestoneScreenParticleType$Factory = Java.loadClass('team.lodestar.lodestone.systems.particle.screen.LodestoneScreenParticleType$Factory');

/**
 * 
 * @param {$RegisterParticleProvidersEvent} event 
 * @param {String[]} List 
 */
export function registerParticleFactory$Screen(event,List){//TODO maybe use event?
    List.forEach(e => {
        $LodestoneScreenParticleRegistry.registerProvider($LodestoneScreenParticleRegistry.registerType(new $LodestoneScreenParticleType()), new $LodestoneScreenParticleType$Factory($LodestoneScreenParticleRegistry.getSpriteSet($LodestoneLib.lodestonePath(e))));
    });
}
/**
 * 
 * @param {$RegisterParticleProvidersEvent} event 
 * @param {$RegistryObject<$LodestoneParticleType>[]} List
 */
export function registerParticleFactory(event,List){
    List.forEach(e => {
        event.registerSpriteSet(e.get(), () => new $LodestoneParticleType$Factory($LodestoneScreenParticleRegistry.getSpriteSet($LodestoneLib.lodestonePath(e.id.getPath()))));
    })
}
```

```js
// startup_scripts/particleregistry.js

//priority: 9999

const $LodestoneParticleRegistry = Java.loadClass('team/lodestar/lodestone/registry/common/particle/LodestoneParticleRegistry')

const $LodestoneParticleType = Java.loadClass('team.lodestar.lodestone.systems.particle.world.type.LodestoneWorldParticleType');

const $RegisterParticleProvidersEvent = Java.loadClass('net.minecraftforge.client.event.RegisterParticleProvidersEvent')

const { registerParticleFactory, registerParticleFactory$Screen } = require('./particlere') // 这里用probejs链接另一个脚本, 如果没有probejs可以不需要这一行,但要把particlere.js的export去掉,因为kjs默认会把方法暴露出去,而pjs不会

let LAVA_PARTICLE = $LodestoneParticleRegistry.PARTICLES.register("lava", () => new $LodestoneParticleType())

let NAUTILUS_PARTICLE = $LodestoneParticleRegistry.PARTICLES.register("nautilus", () => new $LodestoneParticleType())

let CLOUD_PARTICLE = $LodestoneParticleRegistry.PARTICLES.register("cloud", () => new $LodestoneParticleType())

let SONIC_BOOM_PARTICLE = $LodestoneParticleRegistry.PARTICLES.register("sonic_boom", () => new $LodestoneParticleType())

let SOUL_PARTICLE = $LodestoneParticleRegistry.PARTICLES.register("soul", () => new $LodestoneParticleType())

let VIBRATION_PARTICLE = $LodestoneParticleRegistry.PARTICLES.register("vibration", () => new $LodestoneParticleType())

let ELECTRIC_SPARK_PARTICLE = $LodestoneParticleRegistry.PARTICLES.register("electric_spark", () => new $LodestoneParticleType())

let NOTE_PARTICLE = $LodestoneParticleRegistry.PARTICLES.register("note", () => new $LodestoneParticleType())

const PARTICLE_LIST = [
    LAVA_PARTICLE,
    NAUTILUS_PARTICLE,
    CLOUD_PARTICLE,
    SONIC_BOOM_PARTICLE,
    SOUL_PARTICLE,
    VIBRATION_PARTICLE,
    ELECTRIC_SPARK_PARTICLE,
    NOTE_PARTICLE
]
const PARTICLE_LIST$SCREEN = [
    "lava",
    "nautilus",
    "cloud",
    "sonic_boom",
    "soul",
    "vibration",
    "electric_spark",
    "note"
]


ForgeModEvents.onEvent($RegisterParticleProvidersEvent, (event) => {
    registerParticleFactory$Screen(event,PARTICLE_LIST$SCREEN)
    registerParticleFactory(event,PARTICLE_LIST)
})


global.LAVA_PARTICLE = LAVA_PARTICLE
global.NAUTILUS_PARTICLE = NAUTILUS_PARTICLE
global.CLOUD_PARTICLE = CLOUD_PARTICLE
global.SONIC_BOOM_PARTICLE = SONIC_BOOM_PARTICLE
global.SOUL_PARTICLE = SOUL_PARTICLE
global.VIBRATION_PARTICLE = VIBRATION_PARTICLE
global.ELECTRIC_SPARK_PARTICLE = ELECTRIC_SPARK_PARTICLE
global.NOTE_PARTICLE = NOTE_PARTICLE
```

### in client_scripts

```js
// 所需类不变

NetworkEvents.dataReceived("particle",event=>{
    const { level, data } = event
    let mapper = {
        "WISP": $LodestoneParticleRegistry.WISP_PARTICLE,
        "SMOKE": $LodestoneParticleRegistry.SMOKE_PARTICLE,
        "SPARKLE": $LodestoneParticleRegistry.SPARKLE_PARTICLE,
        "TWINKLE": $LodestoneParticleRegistry.TWINKLE_PARTICLE,
        "STAR": $LodestoneParticleRegistry.STAR_PARTICLE,
        "LAVA": global.LAVA_PARTICLE,
        "NAUTILUS": global.NAUTILUS_PARTICLE,
        "CLOUD": global.CLOUD_PARTICLE,
        "SONIC_BOOM": global.SONIC_BOOM_PARTICLE,
        "SOUL": global.SOUL_PARTICLE,
        "VIBRATION": global.VIBRATION_PARTICLE,
        "ELECTRIC_SPARK": global.ELECTRIC_SPARK_PARTICLE,
        "NOTE": global.NOTE_PARTICLE
    }
    let type = mapper[data.type.toUpperCase()] || null
    if(type == null){return}
    const colorStart = data.colorStart
    const colorEnd = data.colorEnd
    $WorldParticleBuilder.create(type)
        .setTransparencyData($GenericParticleData.create(data.transparencyStart, data.transparencyEnd).build())
        .setScaleData($GenericParticleData.create(data.scaleStart, data.scaleEnd).build())
        .setColorData($ColorParticleData.create(new $Color(colorStart[0] / 255, colorStart[1] / 255, colorStart[2] / 255), new $Color(colorEnd[0] / 255, colorEnd[1] / 255, colorEnd[2] / 255)).build())
        .setLifetime(data.lifetime)
        .setRandomOffset(data.randomOffset)
        .addMotion(data.motionX, data.motionY, data.motionZ)
        .setRandomMotion(data.randomMotionX, data.randomMotionY, data.randomMotionZ)
        .setDiscardFunction($SimpleParticleOptions.ParticleDiscardFunctionType.INVISIBLE)
        .setRenderType($LodestoneWorldParticleRenderType.LUMITRANSPARENT)
        .repeat(level, data.x, data.y, data.z, data.count)
})
```

### in server_scripts

```js
// 所需类不变
// 无变化
```


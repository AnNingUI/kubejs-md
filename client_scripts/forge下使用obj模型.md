# 如何在forge下使用Obj模型

## 建立Obj模型
首先，您需要建立一个Obj模型。您可以使用任何建模软件来创建一个Obj模型，比如3ds Max、Blender、BlockBench等。
这里发一个实例
```obj
# Made in Blockbench 4.8.3
mtllib qqqqoo.mtl

o pyramid
v 1 -2.220446049250313e-16 1
v 1 0.9999999999999998 1.0000000000000002
v 0 0.9999999999999998 1.0000000000000002
v 0 -2.220446049250313e-16 1
v 0.5 0.4999999999999999 0.5000000000000001
vt 0.25 0.75
vt 0.25 1
vt 0 1
vt 0 0.75
vt 0 0.479471875
vt 0.25 0.479471875
vt 0.125 0.65625
vt 0.1875 0.620096875
vt 0.4375 0.620096875
vt 0.3125 0.796875
vt 0.265625 0.823221875
vt 0.515625 0.823221875
vt 0.390625 1
vt 0.375 0.479471875
vt 0.625 0.479471875
vt 0.5 0.65625
vn 0 -2.220446049250313e-16 1
vn 0.7071067811865476 1.5700924586837752e-16 -0.7071067811865476
vn 0 0.7071067811865477 -0.7071067811865475
vn 0 -0.7071067811865475 -0.7071067811865477
vn -0.7071067811865476 1.5700924586837752e-16 -0.7071067811865476
usemtl m_c5b90dae-3a75-90c1-2280-8a6b9dc0fba2
f 1/1/1 2/2/1 3/3/1 4/4/1
f 2/5/2 1/6/2 5/7/2
f 3/8/3 2/9/3 5/10/3
f 1/11/4 4/12/4 5/13/4
f 4/14/5 3/15/5 5/16/5
```
同时导出时会自动生成一个mtl文件和一个png文件
```mtl
# Made in Blockbench 4.8.3
newmtl m_c5b90dae-3a75-90c1-2280-8a6b9dc0fba2
map_Kd kubejs:block/qqqqoo
newmtl none
```
注意obj里面的usemtl m_c5b90dae-3a75-90c1-2280-8a6b9dc0fba2要与mtl文件里面的newmtl m_c5b90dae-3a75-90c1-2280-8a6b9dc0fba2对应上
obj的mtllib qqqqoo.mtl是确定mtl文件的，这里默认同级就行
mtl里面的map_Kd则是要指定贴图的路径
贴图放在-**{ modid }**/assets/textures/block下
## 指向Obj文件
因为mc无法直接读取obj文件，所以需要一个json模型指向obj文件
```json
{
  "loader": "forge:obj",
  "model": "kubejs:models/block/qqqqoo.obj",
  "flip_v": true,
  "textures": {
    "particle": "kubejs:block/qqqqoo"
  }
}
```
loader表示要加载的类型,这里指定obj模型
model表示要加载的Obj模型路径
flip_v表示是否翻转贴图,因为obj模型贴图相对于MC来说是倒着的，所以要true
textures表示要加载的贴图，这里指定particle贴图来确定粒子

## 确定blockstate
在加载obj模型时，需要确定blockstate，因为blockstate在加载Obj时不会自动生成的
导致无法让方块确定模型
```json
{
  "variants": {
    "facing=down": {
      "model": "kubejs:block/qqqqoo",
      "x": 90
    },
    "facing=east": {
      "model": "kubejs:block/qqqqoo",
      "y": 90
    },
    "facing=north": {
      "model": "kubejs:block/qqqqoo"
    },
    "facing=south": {
      "model": "kubejs:block/qqqqoo",
      "y": 180
    },
    "facing=up": {
      "model": "kubejs:block/qqqqoo",
      "x": 270
    },
    "facing=west": {
      "model": "kubejs:block/qqqqoo",
      "y": 270
    }
  }
}
```

## 检验路径
mtl，obj，指向obj模型的json文件放在-**{ modid }**/assets/models/block下
png放在-**{ modid }**/assets/textures/block下
确定blockstate的json文件需要放在-**{ modid }**/assets/blockstate下
同时要该文件命名都为方块注册名

## 参考自
> [【我的世界 1.20.4 NeoForge 最新模组教程】18 加载OBJ模型](https://www.bilibili.com/video/BV1jm421J7UR/?spm_id_from=333.999.0.0&vd_source=a6e9e72f334103d28476ce3f30850f61)
> [【Boson 1.16 Modding Tutorial】 - Obj](https://boson.v2mcdev.com/specialmodel/obj.html)


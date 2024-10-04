# 基于Renderjs的基座方块

首先你需要去下载[renderjs](https://www.curseforge.com/minecraft/mc-mods/renderjs)
然后按照这个案例修改就可以了

startup_scripts/src/block.js
```js
const { $Player } = require("packages/net/minecraft/world/entity/player/$Player");
const { $ItemStack } = require("packages/net/minecraft/world/item/$ItemStack");

// 一个常量，值为0
const OMMMMO = 0;

// 注册方块和相关功能
StartupEvents.registry("block", event => {
    event.create("pedestal")
    .blockEntity(info => {
        info.inventory(1, 1); // 创建一个有一个槽的库存
        info.attachCapability(
            CapabilityBuilder.ITEM.blockEntity()
              .extractItem((blockEntity, slot, amount, simulate) => blockEntity.inventory.extractItem(slot, amount, simulate)) // 提取物品
              .insertItem((blockEntity, slot, stack, simulate) => blockEntity.inventory.insertItem(slot, stack, simulate)) // 插入物品
              .getSlotLimit((blockEntity, slot) => blockEntity.inventory.getSlotLimit(slot)) // 获取槽位限制
              .getSlots((blockEntity) => blockEntity.inventory.slots) // 获取所有槽位
              .getStackInSlot((blockEntity, slot) => blockEntity.inventory.getStackInSlot(slot)) // 获取槽位中的物品
              .isItemValid((blockEntity, slot, stack) => blockEntity.inventory.isItemValid(slot, stack)) // 检查物品是否有效
              .availableOn((blockEntity, direction) => direction != Direction.UP) // 方块的有效方向
          );
        info.serverTick(OMMMMO, 0, be => {
            be.load(be.block.entityData); // 加载方块实体数据
            const level = be.level;
            if (!level) return;
            const players = be.level.getPlayers();
            if (!players || players.length === 0) return;
            const player = players[0];
            if (!player) return;

            // 发送数据到客户端
            player.sendData("pedestal", {
                entityData: be.block.entityData,
                pos: {
                    x: be.block.pos.x,
                    y: be.block.pos.y,
                    z: be.block.pos.z
                }
            });
        });
    })
    .rightClick(e => {
        /**
         * 更新物品
         * @param {*} block 方块
         * @param {*} item 物品
         * @param {*} isNew 是否是新物品
         * @param {$Player} player 玩家
         */
        const updateItems = (block, item, isNew) => {
            const items = block.getEntityData().attachments[0].items || [];
            if (isNew) {
                items.push(item); // 添加新物品
            } else {
                Object.assign(items[0], item); // 更新现有物品
            }
            const newAttachment = {
                id: item.id,
                Count: item.Count
            };
            if (item.tag) {
                newAttachment.tag = item.tag; // 添加NBT标签
            }
            block.mergeEntityData({
                attachments: [{ items: [newAttachment] }]
            });
        };

        let { id, nbt, count, setCount } = e.player.getHeldItem("main_hand");
        if (id === "minecraft:air") return; // 如果手持物品为空气，直接返回
        const bitem = { id: id, Count: count };
        if (nbt) {
            bitem.tag = nbt; // 如果有NBT标签，添加到物品中
        }
        const items = e.block.getEntityData().attachments[0].items || [];
        updateItems(e.block, { Count: bitem.Count, Slot: 0, id: bitem.id, tag: bitem.tag }, items.length === 0); // 更新物品
    });    
});
```

---

client_scripts/src/render.js
```js
const { $Vec3 } = require("packages/net/minecraft/world/phys/$Vec3");
const { $Axis } = require("packages/com/mojang/math/$Axis");
const { $ItemStack } = require("packages/net/minecraft/world/item/$ItemStack");
const { $BlockPos } = require("packages/net/minecraft/core/$BlockPos");

// 存储项目的位置和物品信息
let _Items = [{
    pos: "0_-71_0",
    item: Item.of("minecraft:air", 1) // 默认初始物品为空气
}];

// 添加或更新物品
function _addItem(pos, item) {
    _removeItem("0_-71_0"); // 移除默认位置的物品
    const index = _Items.findIndex(i => i.pos === pos);
    if (index !== -1) {
        _Items[index].item = item; // 更新现有物品
    } else {
        _Items.push({ pos: pos, item: item }); // 添加新物品
    }
}

// 获取指定位置的物品
function _getItem(pos) {
    const item = _Items.find(i => i.pos === pos);
    return item ? item.item : null; // 找到返回物品，否则返回null
}

// 设置物品
function _setItem(pos, item) {
    _removeItem("0_-71_0"); // 移除默认位置的物品
    const index = _Items.findIndex(i => i.pos === pos);
    if (index !== -1) {
        _Items[index].item = item; // 更新现有物品
    } else {}
}

// 移除指定位置的物品
function _removeItem(pos) {
    const index = _Items.findIndex(i => i.pos === pos);
    if (index !== -1) {
        _Items.splice(index, 1); // 移除物品
    } else {}
}

// 处理接收到的数据
NetworkEvents.dataReceived("pedestal", e => {
    const { level, data } = e;
    const pos2str = `${data.pos.x}_${data.pos.y}_${data.pos.z}`; // 将位置转换为字符串
    const itemData = data.entityData.attachments[0].items[0];
    if (!itemData || itemData.id === "minecraft:air") return; // 如果没有物品或物品为空气，直接返回
    const itemStack = new $ItemStack(itemData.id, itemData.Count, itemData.tag);
    const itembool = _getItem(pos2str) == itemStack;
    itembool ? _setItem(pos2str, itemStack) : _addItem(pos2str, itemStack); // 根据物品是否存在决定添加或更新
    _Items.forEach((z) => {
        const pos = str2pos(z.pos);
        level.getBlock(pos).id != "kubejs:pedestal" && _removeItem(z.pos); // 如果位置的方块不是pedestal，则移除物品
    });
});

// 渲染物品
RenderJSEvents.AddWorldRender((e) => {
    console.log([_Items]);
    e.addWorldRender((r) => {
        _Items.forEach((_i) => {
            const itemStack = _i.item;
            const _pos = str2pos(_i.pos);
            const camera = r.camera;
            const cameraX = camera.position.get("x");
            const cameraY = camera.position.get("y");
            const cameraZ = camera.position.get("z");
            const block = Client.level.getBlock(_pos);
            if (block.id == "kubejs:pedestal") {
                r.poseStack.pushPose();
                const pos = _n(_pos);
                const renderX = pos.x + 0.5;
                const renderY = pos.y + 1.25;
                const renderZ = pos.z + 0.5;
                r.poseStack.translate(renderX - cameraX, renderY - cameraY, renderZ - cameraZ);
                const time = Date.now();
                const angle = (time / 10) % 360; // 计算旋转角度
                r.poseStack.mulPose($Axis.YP.rotationDegrees(angle)); // 在Y轴旋转物品
                r.poseStack.scale(0.95, 0.95, 0.95); // 缩放物品
                RenderJSWorldRender.renderItem(r.poseStack, itemStack, 15, 15, Client.level); // 渲染物品
                r.poseStack.popPose();
            } else {
                _removeItem(_i.pos); // 如果方块不是pedestal，移除物品
            }
        });
    });
});

// OTMapper 配置
export const OTMapper = {
    NO_OVERLAY: 655360.0,
    NO_WHITE_U: 0.0,
    RED_OVERLAY_V: 3.0,
    WHITE_OVERLAY_V: 10.0
};

// 将 $Vec3 对象转换为普通对象
function _n(_pos) {
    return { x: _pos.get("x"), y: _pos.get("y"), z: _pos.get("z") };
}

// 将字符串位置转换为 $BlockPos 对象
const str2pos = (str) => {
    let _pos = str.split("_").map((e) => parseInt(e));
    return new $BlockPos(_pos[0], _pos[1], _pos[2]);
};
```
























































Discord

# Creating a Pedestal Block with RenderJS

First you need to download [RenderJS](https://www.curseforge.com/minecraft/mc-mods/renderjs) and [ProbeJS](https://www.curseforge.com/minecraft/mc-mods/probejs), and then modify the following as needed

File: `startup_scripts/src/block.js`
```js
const { $Player } = require("packages/net/minecraft/world/entity/player/$Player");
const { $ItemStack } = require("packages/net/minecraft/world/item/$ItemStack");

// Constant value 0
const OMMMMO = 0;

// Register the block and related functionality
StartupEvents.registry("block", event => {
    event.create("pedestal")
    .blockEntity(info => {
        info.inventory(1, 1); // Create an inventory with one slot
        info.attachCapability(
            CapabilityBuilder.ITEM.blockEntity()
              .extractItem((blockEntity, slot, amount, simulate) => blockEntity.inventory.extractItem(slot, amount, simulate)) // Extract items
              .insertItem((blockEntity, slot, stack, simulate) => blockEntity.inventory.insertItem(slot, stack, simulate)) // Insert items
              .getSlotLimit((blockEntity, slot) => blockEntity.inventory.getSlotLimit(slot)) // Get slot limit
              .getSlots((blockEntity) => blockEntity.inventory.slots) // Get all slots
              .getStackInSlot((blockEntity, slot) => blockEntity.inventory.getStackInSlot(slot)) // Get item in slot
              .isItemValid((blockEntity, slot, stack) => blockEntity.inventory.isItemValid(slot, stack)) // Check if item is valid
              .availableOn((blockEntity, direction) => direction != Direction.UP) // Valid directions for the block
          );
        info.serverTick(OMMMMO, 0, be => {
            be.load(be.block.entityData); // Load block entity data
            const level = be.level;
            if (!level) return;
            const players = be.level.getPlayers();
            if (!players || players.length === 0) return;
            const player = players[0];
            if (!player) return;

            // Send data to the client
            player.sendData("pedestal", {
                entityData: be.block.entityData,
                pos: {
                    x: be.block.pos.x,
                    y: be.block.pos.y,
                    z: be.block.pos.z
                }
            });
        });
    })
    .rightClick(e => {
        /**
         * Update item
         * @param {*} block Block
         * @param {*} item Item
         * @param {*} isNew Whether it's a new item
         * @param {$Player} player Player
         */
        const updateItems = (block, item, isNew) => {
            const items = block.getEntityData().attachments[0].items || [];
            if (isNew) {
                items.push(item); // Add new item
            } else {
                Object.assign(items[0], item); // Update existing item
            }
            const newAttachment = {
                id: item.id,
                Count: item.Count
            };
            if (item.tag) {
                newAttachment.tag = item.tag; // Add NBT tag
            }
            block.mergeEntityData({
                attachments: [{ items: [newAttachment] }]
            });
        };

        let { id, nbt, count, setCount } = e.player.getHeldItem("main_hand");
        if (id === "minecraft:air") return; // If held item is air, return
        const bitem = { id: id, Count: count };
        if (nbt) {
            bitem.tag = nbt; // Add NBT tag if present
        }
        const items = e.block.getEntityData().attachments[0].items || [];
        updateItems(e.block, { Count: bitem.Count, Slot: 0, id: bitem.id, tag: bitem.tag }, items.length === 0); // Update item
    });    
});
```

---

File: `client_scripts/src/render.js`
```js
const { $Vec3 } = require("packages/net/minecraft/world/phys/$Vec3");
const { $Axis } = require("packages/com/mojang/math/$Axis");
const { $ItemStack } = require("packages/net/minecraft/world/item/$ItemStack");
const { $BlockPos } = require("packages/net/minecraft/core/$BlockPos");

// Store item positions and information
let _Items = [{
    pos: "0_-71_0",
    item: Item.of("minecraft:air", 1) // Default initial item is air
}];

// Add or update item
function _addItem(pos, item) {
    _removeItem("0_-71_0"); // Remove item from default position
    const index = _Items.findIndex(i => i.pos === pos);
    if (index !== -1) {
        _Items[index].item = item; // Update existing item
    } else {
        _Items.push({ pos: pos, item: item }); // Add new item
    }
}

// Get item at a specified position
function _getItem(pos) {
    const item = _Items.find(i => i.pos === pos);
    return item ? item.item : null; // Return item if found, otherwise null
}

// Set item at a specified position
function _setItem(pos, item) {
    _removeItem("0_-71_0"); // Remove item from default position
    const index = _Items.findIndex(i => i.pos === pos);
    if (index !== -1) {
        _Items[index].item = item; // Update existing item
    } else {}
}

// Remove item from a specified position
function _removeItem(pos) {
    const index = _Items.findIndex(i => i.pos === pos);
    if (index !== -1) {
        _Items.splice(index, 1); // Remove item
    } else {}
}

// Handle received data
NetworkEvents.dataReceived("pedestal", e => {
    const { level, data } = e;
    const pos2str = `${data.pos.x}_${data.pos.y}_${data.pos.z}`; // Convert position to string
    const itemData = data.entityData.attachments[0].items[0];
    if (!itemData || itemData.id === "minecraft:air") return; // Return if item is air or not present
    const itemStack = new $ItemStack(itemData.id, itemData.Count, itemData.tag);
    const itembool = _getItem(pos2str) == itemStack;
    itembool ? _setItem(pos2str, itemStack) : _addItem(pos2str, itemStack); // Add or update item based on existence
    _Items.forEach((z) => {
        const pos = str2pos(z.pos);
        level.getBlock(pos).id != "kubejs:pedestal" && _removeItem(z.pos); // Remove item if block is not pedestal
    });
});

// Render items
RenderJSEvents.AddWorldRender((e) => {
    console.log([_Items]);
    e.addWorldRender((r) => {
        _Items.forEach((_i) => {
            const itemStack = _i.item;
            const _pos = str2pos(_i.pos);
            const camera = r.camera;
            const cameraX = camera.position.get("x");
            const cameraY = camera.position.get("y");
            const cameraZ = camera.position.get("z");
            const block = Client.level.getBlock(_pos);
            if (block.id == "kubejs:pedestal") {
                r.poseStack.pushPose();
                const pos = _n(_pos);
                const renderX = pos.x + 0.5;
                const renderY = pos.y + 1.25;
                const renderZ = pos.z + 0.5;
                r.poseStack.translate(renderX - cameraX, renderY - cameraY, renderZ - cameraZ);
                const time = Date.now();
                const angle = (time / 10) % 360; // Calculate rotation angle
                r.poseStack.mulPose($Axis.YP.rotationDegrees(angle)); // Rotate item around Y-axis
                r.poseStack.scale(0.95, 0.95, 0.95); // Scale item
                RenderJSWorldRender.renderItem(r.poseStack, itemStack, 15, 15, Client.level); // Render item
                r.poseStack.popPose();
            } else {
                _removeItem(_i.pos); // Remove item if block is not pedestal
            }
        });
    });
});

// OTMapper Configuration
export const OTMapper = {
    NO_OVERLAY: 655360.0,
    NO_WHITE_U: 0.0,
    RED_OVERLAY_V: 3.0,
    WHITE_OVERLAY_V: 10.0
};

// Convert $Vec3 object to a plain object
function _n(_pos) {
    return { x: _pos.get("x"), y: _pos.get("y"), z: _pos.get("z") };
}

// Convert string position to $BlockPos object
const str2pos = (str) => {
    let _pos = str.split("_").map((e) => parseInt(e));
    return new $BlockPos(_pos[0], _pos[1], _pos[2]);
};
```





## New RenderJS -1.1.9

```js
// startup_scripts

const { $Axis } = require("packages/com/mojang/math/$Axis");
const { $BlockEntityJS } = require("packages/dev/latvian/mods/kubejs/block/entity/$BlockEntityJS");
const { $Player } = require("packages/net/minecraft/world/entity/player/$Player");

const OMMMMO = 0;

StartupEvents.registry("block", event => {
    event.create("pedestal")
    .blockEntity(info => {
        info.enableSync()
        info.inventory(1, 1);
        info.attachCapability(
            CapabilityBuilder.ITEM.blockEntity()
              .extractItem((blockEntity, slot, amount, simulate) => blockEntity.inventory.extractItem(slot, amount, simulate))
              .insertItem((blockEntity, slot, stack, simulate) => blockEntity.inventory.insertItem(slot, stack, simulate))
              .getSlotLimit((blockEntity, slot) => blockEntity.inventory.getSlotLimit(slot))
              .getSlots((blockEntity) => blockEntity.inventory.slots)
              .getStackInSlot((blockEntity, slot) => blockEntity.inventory.getStackInSlot(slot))
              .isItemValid((blockEntity, slot, stack) => blockEntity.inventory.isItemValid(slot, stack))
              .availableOn((blockEntity, direction) => direction != "*")
          );
        info.serverTick(OMMMMO, 0, be => {
            const entityData = be.getBlock().entityData;
            const item = entityData.attachments[0].items[0] || undefined
            const entity = be.getBlock().entity;
            if (entity instanceof $BlockEntityJS) {
                const itemObj = item ? (item?.tag ? { id : item.id, count : item.Count, tag : item.tag } : { id : item.id, count : item.Count }) : {}
                entity.data.put("item",itemObj)
                entity.sync();
                be.sync();
            }
        });
    })
    .rightClick(e => {
        /**
         * 
         * @param {*} block 
         * @param {*} item 
         * @param {*} isNew 
         * @param {$Player} player 
         */
        const updateItems = (block, item, isNew) => {
            const items = block.getEntityData().attachments[0].items || [];
            if (isNew) {
                items.push(item);
            } else {
                Object.assign(items[0], item);
            }
            const newAttachment = {
                id: item.id,
                Count: item.Count
            };
            if (item.tag) {
                newAttachment.tag = item.tag;
            }
            block.mergeEntityData({
                attachments: [{ items: [newAttachment] }]
            });
        };

        let { id, nbt, count, setCount } = e.player.getHeldItem("main_hand");
        if (id === "minecraft:air") return;
        const bitem = { id: id, Count: count };
        if (nbt) {
            bitem.tag = nbt;
        }
        const items = e.block.getEntityData().attachments[0].items || [];
        updateItems(e.block, { Count: bitem.Count, Slot: 0, id: bitem.id, tag: bitem.tag }, items.length === 0);
    });    
});





global.inited = false
ClientEvents.init(event => {
    global.inited=true
    event.registerBlockEntityRenderer("kubejs:pedestal", (context) => 
        RenderJSBlockEntityRenderer
            .create(context)
            .setCustomRender((renderer, context) => {
                let poseStack = context.poseStack
                let light = LevelRenderer.getLightColor(context.blockEntityJS.level, context.blockEntityJS.blockPos.above())//获取亮度
                let item = Item.getEmpty()
                let data = context.blockEntityJS.data//读取kjs方块实体的data("同步的信息")
                let itemData = data.get("item")//读取kjs方块实体的data("同步的信息")中的item
                if (itemData) {
                    item = itemData.tag ? Item.of(itemData.id, itemData.count, itemData.tag) : Item.of(itemData.id, itemData.count)
                }
                if (item != Item.getEmpty()) {
                    poseStack.pushPose();
                    poseStack.translate(0.5, 1.25, 0.5)//平移(0.5, 1.5, 0.5)
                    const time = Date.now();
                    const angle = (time / 10) % 360;
                    poseStack.mulPose($Axis.YP.rotationDegrees(angle));
                    poseStack.scale(0.95, 0.95, 0.95);
                    renderer.itemRenderer.renderStatic(item, "ground", light, context.packedOverlay, context.poseStack, context.bufferSource, Client.level, Client.player.getId())
                    poseStack.popPose();
                }
            })
    )
})
```


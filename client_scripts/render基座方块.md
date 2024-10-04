# 基于Renderjs的基座方块

首先你需要去下载[renderjs](https://www.curseforge.com/minecraft/mc-mods/renderjs)
然后按照这个案例修改就可以了

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


## 显示多方块
```js

JEIAddedEvents.registerCategories(event => {
	event.custom('kubejs:MultiBlock', category => {
	    let { jeiHelpers } = category;
		let { guiHelper } = jeiHelpers;

		global.multiBlockRecipeType = category
			.title(Text.translatable('category.kubejs.multiblock.test'))
			.background(guiHelper.createBlankDrawable(120, 120))
			.icon(guiHelper.createDrawableItemStack('minecraft:structure_block'))
			.isRecipeHandled(r => r.data != undefined && r.data.description != undefined && r.data.offset != undefined && r.data.renderScale != undefined)
			.handleLookup((builder, r, focuses) => global["handlemultiBlockLookup"](builder, r, focuses))
			.setDrawHandler((r, recipeSlotsView, guiGraphics, mouseX, mouseY) => global.draw(r,guiGraphics))
	})
})

JEIAddedEvents.registerRecipes(event => {
	event.custom('kubejs:MultiBlock')
		.add({
			multiBlock: (level) => {
				let mutblocks = [
					{get:Block.getBlock("minecraft:obsidian"), pos:new BlockPos(0,3,0)},
					{get:Block.getBlock("minecraft:obsidian"), pos:new BlockPos(0,3,1)},
					{get:Block.getBlock("minecraft:obsidian"), pos:new BlockPos(0,3,-1)},
					{get:Block.getBlock("minecraft:obsidian"), pos:new BlockPos(1,3,0)},
					{get:Block.getBlock("minecraft:obsidian"), pos:new BlockPos(-1,3,0)},
					{get:Block.getBlock("minecraft:obsidian"), pos:new BlockPos(1,3,-1)},
					{get:Block.getBlock("minecraft:obsidian"), pos:new BlockPos(-1,3,1)},
					{get:Block.getBlock("minecraft:obsidian"), pos:new BlockPos(1,3,1)},
					{get:Block.getBlock("minecraft:obsidian"), pos:new BlockPos(-1,3,-1)},
					{get:Block.getBlock('minecraft:sculk_shrieker'), pos:new BlockPos(0,4,0)},
				];
				return mutblocks;
			},
			recipes: {
				"input": [
					Item.of("minecraft:diamond_sword"),
					Item.of("morecategory:diamond_sickle"),
				],
				"Amount": 1000,
				"output": [
					Item.of("3x minecraft:diamond")
				]
			},
		    description: 'jei.description.ignis',
			offset: {x:7,y:23,z:7}, // Change to specification
			renderScale: 4 // Change to specification
		})
})

//渲染方法
let $AnimatedKinetics = Java.loadClass("com.simibubi.create.compat.jei.category.animations.AnimatedKinetics")
//使用$AnimatedKinetics.defaultBlockElement方法渲染方块
//参考自https://github.com/Creators-of-Create/Create/blob/mc1.20.1/dev/src/main/java/com/simibubi/create/compat/jei/category/animations/AnimatedKinetics.java
/* 
let $GuiGameElement = Java.loadClass("com.simibubi.create.foundation.gui.element.GuiGameElement")
let $CustomLightingSettings = Java.loadClass("com.simibubi.create.foundation.gui.CustomLightingSettings")
以下 defaultBlockElement(block.get) 可替换为 $GuiGameElement.of(block.get).lighting($CustomLightingSettings.builder()
			.firstLightRotation(12.5, 45.0)
			.secondLightRotation(-20.0, 50.0)
			.build())
使用of也可以渲染物品，流体，以下不做演示
参考自
https://github.com/Creators-of-Create/Create/blob/mc1.20.1/dev/src/main/java/com/simibubi/create/foundation/gui/element/GuiGameElement.java
https://github.com/Creators-of-Create/Create/blob/mc1.20.1/dev/src/main/java/com/simibubi/create/foundation/gui/CustomLightingSettings.java
*/
let $Axis = Java.loadClass("com.mojang.math.Axis")
/**
 * 
 * @param {Internal.CustomJSRecipe} r 
 * @param {Internal.GuiGraphics} graphics 
 * @param {number} xOffset 
 * @param {number} yOffset 
 */
global.draw = ( r, graphics) => {
	graphics.drawWordWrap(Client.font, Text.of("=>"), global.j_go+10, 85, 140, 0);
    let matrixStack = graphics.pose();
	let {defaultBlockElement} = $AnimatedKinetics
    matrixStack.pushPose();
    // matrixStack.translate(xOffset, yOffset, 200);
    matrixStack.mulPose($Axis.XP.rotationDegrees(-15.5));
    matrixStack.mulPose($Axis.YP.rotationDegrees(22.5));
    let scale = r.data.renderScale;
	matrixStack.scale(scale, scale, scale)
	matrixStack.translate(r.data.offset.x, r.data.offset.y, r.data.offset.z);
	let mutblocks = /**@type {Internal.ArrowFunction} */(r.data.multiBlock)
	let mub = /**@type {Internal.ArrayList} */(mutblocks(Client.level))
	mub.forEach(block => {
	    defaultBlockElement(block.get)
				.atLocal(block.pos.x,-block.pos.y,block.pos.z)
				.scale(scale)
				.render(graphics);
	})
	matrixStack.centre();
    // console.log(mutblocks)
    matrixStack.scale(scale, -scale, scale);
    matrixStack.translate(0, -1.8, 0);
    matrixStack.popPose();
}

//添加插槽
/**
 * 
 * @param {Internal.IRecipeLayoutBuilder} builder 
 * @param {Internal.CustomJSRecipe} r 
 */
global["handlemultiBlockLookup"] = (builder, r) => {
	builder.setShapeless()
	let i = 0
	builder.addSlot("INPUT", 10, 80).addFluidStack(Fluid.getType("cai:mana"), r.data.recipes.Amount).setSlotName("input")
	r.data.recipes.input.forEach((input,index)=>{
		builder.addSlot("INPUT", 30+index*23, 80).addItemStack(Item.of(input)).setSlotName("input");
		i = Math.max(i,20+index*23)
	})
	global.j_go = i+20
	r.data.recipes.output.forEach((output,index)=>{
		builder.addSlot("OUTPUT", i+40+index*25, 80).addItemStack(Item.of(output)).setSlotName("output");
	})
}

```
> 本文思路来自于dc，准确来说来自于[dc老哥](https://discord.com/channels/303440391124942858/1229657610698100806/1229657610698100806 )与[create源码](https://github.com/Creators-of-Create/Create/tree/mc1.20.1/dev )
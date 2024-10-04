
# 客户端脚本 in clint_scripts
## 显示实体
```js
// In client scripts

// Some reflection to prevent the entity from being recreated every frame

//注 ：我也不知道每个方法的作用，只能做一些由自己的经验得出的猜测
const Entity = Java.loadClass('net.minecraft.world.entity.Entity');
const Level = Java.loadClass('net.minecraft.world.level.Level');
const setLevelMethod = Entity.__javaObject__.getDeclaredMethod('m_284535_', Level);
setLevelMethod.setAccessible(true);

//注册主类型
JEIAddedEvents.registerCategories(event => {
	event.custom('kubejs:entity', category => {
		let { jeiHelpers } = category;
		let { guiHelper } = jeiHelpers;

		global.entityRecipeType = category
			.title(Text.translatable('category.kubejs.entity.test'))
			.background(guiHelper.createBlankDrawable(100, 100))
			.icon(guiHelper.createDrawableItemStack('cataclysm:ignis_spawn_egg'))
      //显示条件判断
			.isRecipeHandled(r => global.verifyEntityRecipe(r))
      //添加插槽 物品 流体
			.handleLookup((builder, r, focuses) => global.handleEntityLookup(builder, r, focuses))
			//显示方块 实体
      .setDrawHandler((r, recipeSlotsView, guiGraphics, mouseX, mouseY) => global.renderEntityRecipe(r, guiGraphics))
			.recipeType;
	})
})

//注册配方类型
JEIAddedEvents.registerRecipes(event => {
	let entity = Client.level.createEntity('cataclysm:ignis');
	entity.noCulling = true;

	event.custom('kubejs:entity')
		.add({
			entity: (level) => {
				if (entity.level != level) {
					setLevelMethod.invoke(entity, level);
				}
				return entity;
			},
			description: 'jei.description.ignis',
			offset: 1, // Change to specification
			renderScale: 15 // Change to specification
		});
})

//注册侧边标签页
JEIAddedEvents.registerRecipeCatalysts(event => {
	event.data.addRecipeCatalyst('cataclysm:ignis_spawn_egg', global.entityRecipeType)
})

//验证配方
/**
 * 
 * @param {Internal.CustomJSRecipe} r 
 */
global.verifyEntityRecipe = (r) => {
	return r.data != undefined && r.data.entity != undefined && r.data.description != undefined && r.data.offset != undefined && r.data.renderScale != undefined;
}

//添加插槽
/**
 * 
 * @param {Internal.IRecipeLayoutBuilder} builder 
 */
global.handleEntityLookup = (builder) => {
	// Required because JEI doesn't seem to build a category if it has no slots
	builder.addSlot('catalyst', 0, 0);
}

//设置渲染方法
/**
 * 
 * @param {Internal.CustomJSRecipe} r 
 * @param {Internal.GuiGraphics} guiGraphics 
 */
global.renderEntityRecipe = (r, guiGraphics) => {
	guiGraphics.drawWordWrap(Client.font, Text.translatable(r.data.description), 0, 5, 100, 0);
	let poseStack = guiGraphics.pose();
	poseStack.pushPose();
	let entity = r.data.entity(Client.level);

	// This part mostly comes from looking at how patchouli does it
	poseStack.translate(58, 60, 50);
	let scale = r.data.renderScale;
	poseStack.scale(scale, scale, scale)
	poseStack.translate(0, r.data.offset, 0);
	poseStack.mulPose(new Quaternionf().rotationZ(KMath.PI)); // Whoever decided to bind Quaternionf thank you so much
	poseStack.mulPose(new Quaternionf().rotationY(KMath.PI / 3)); // Experiment with these values, find a value you like

	let entityRenderDispatcher = Client.entityRenderDispatcher;
	entityRenderDispatcher.setRenderShadow(false);
	entityRenderDispatcher.render(entity, 0, 0, 0, 0, 1, poseStack, guiGraphics.bufferSource(), 0xF000F0);
	entityRenderDispatcher.setRenderShadow(true);

	guiGraphics.bufferSource().endBatch();
	poseStack.popPose();
}
```
> 本文思路来自于dc，准确来说来自于[dc老哥](https://discord.com/channels/303440391124942858/1229657610698100806/1229657610698100806 )与[create源码](https://github.com/Creators-of-Create/Create/tree/mc1.20.1/dev )
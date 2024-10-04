# PatchouliAPI实战 -- 写一个简单的类凝聚板效果
[示例视频](https://www.bilibili.com/video/BV1aM4m1Q7TM/?spm_id_from=333.788&vd_source=a6e9e72f334103d28476ce3f30850f61)
## in startup_script
### 注册多方快
```javascript
// 导包
const $PatchouliAPI = Java.loadClass('vazkii.patchouli.api.$PatchouliAPI');
const $Character = Java.loadClass('java.lang.Character');

// 选择你需要的方块
global.Test_MultiBlock = {
    G: Block.getBlock('minecraft:obsidian'), // 基座底部
    O: Block.getBlock('minecraft:sculk_shrieker') // 凝聚板本体
}

// 注册多方快形状
global.Test_MultiBlock_Machine = () =>
    $PatchouliAPI.get().makeMultiblock(
        [
            ["___", "___", "___"],
            ["___", "_0_", "___"],
            ["ggg", "ggg", "ggg"],
        ],
        new $Character('_'),
        $PatchouliAPI.get().anyMatcher(),
        new $Character('g'),
        global.Test_MultiBlock.G,
        new $Character('0'),
        global.Test_MultiBlock.O
    )

// 多方快形状放入到同一管理的全局变量
global.MULTIBLOCK = {
    // 其他多方快形状...
    Test_MultiBlock_Machine: global.Test_MultiBlock_Machine,
    // 其他多方快形状...
}

// 注册多方快到游戏中
StartupEvents.postInit((event) => {
    // 其他多方快注册...
    $PatchouliAPI.get().registerMultiblock(
        ResourceLocation("kubejs:test_multiblock_machine"),
        global.Test_MultiBlock_Machine()
    );
    // 其他多方快注册...
})
```

## in server_script
### 辅助方法
```javascript
/**
 * 获取凝聚板上的掉落物物品列表
 * @param {$BlockPos$MutableBlockPos} worldPosition 
 * @param {$Level} level*/
global.getItemListToBe = (worldPosition, level) => {
	let gp = worldPosition.offset(1, 1, 1)
	let aabb = new $AABB(worldPosition.x, worldPosition.y, worldPosition.z, gp.x, gp.y, gp.z);
	let itemEntities = level.getEntitiesOfClass($ItemEntity, aabb, $EntitySelector.ENTITY_STILL_ALIVE)
	let stacks = [];
	itemEntities.forEach(itemEntity => {
		if (!itemEntity.getItem().isEmpty()) {
			stacks.push(itemEntity.getItem());
		}
	})
	return stacks;
}

/**
 * 移除配方的输入物品并弹出配方的输出物品
 * @param {$BlockContainerJS} block
 * @param {$BlockPos$MutableBlockPos} pos 
 * @param {$Level} level
 * @param {[{}]} recipes*/
global.RemoveAndPopItemListToBe = (block, pos, level, recipes) => {
	let gp = pos.offset(1, 1, 1)
	let aabb = new $AABB(pos.x, pos.y, pos.z, gp.x, gp.y, gp.z);
	let itemEntities = level.getEntitiesOfClass($ItemEntity, aabb, $EntitySelector.ENTITY_STILL_ALIVE)
	/**@param {$List<$ItemEntity>} itemEntities*/
	const isSubset = new $ArrayList(recipes.input).every(item => itemEntities.map(e => e.getItem().id).indexOf(item.id) != -1)
	//id检测^^^
	// new ArrayList(recipes.input).forEach(item => {console.log(item.count,itemEntities.find(e => e.getItem().id == item.id).getItem().count)})
	// console.log(itemEntities.map(e=>e.getItem()))
	// console.log(isSubset)
	if (!isSubset) {
		// console.log(isSubset)
		return
	}
	const countok = new $ArrayList(recipes.input).every(item => item.count <= itemEntities.find(e => e.getItem().id == item.id).getItem().count)
	//数量匹配^^^
	if (!countok) {
		// console.log(countok)
		return
	}
	// 定义递归函数来添加粒子
	let addParticles = (i, poss) => {
		if (i > 4) {
			return; // 当i大于4时结束递归
		}

		block.level.server.scheduleInTicks(2 + i, () => {
			global.AddParticle(
				global.particle.sonic_boom,
				poss.x + Math.cos(i),
				poss.y + i / 2,
				poss.z + Math.sin(i),
				0,
				0,
				0
			);
		});

		// 递归调用addParticles函数，每次都将i加1
		addParticles(i + 1, pos);
	}
	let i = 0;
	addParticles(i, pos);
	let boolfilterScreen = {
		boolers: {},
	};
	//^^^布尔过滤器
	recipes.input.forEach(input => {
		let items = recipes.input.filter(e => e.maxStackSize == 1);
		items.forEach(item => {
			boolfilterScreen.boolers[item.id] = false;
		});
	});

	itemEntities.forEach(itemEntity => {
		if (!itemEntity.getItem().isEmpty()) {
			recipes.input.forEach(input => {
				let item = recipes.input.find(e => e.maxStackSize == 1 && e.id === itemEntity.getItem().id);
				if (item && itemEntity.getItem().count >= input.count) {
					if (boolfilterScreen.boolers[item.id]) {
						return;
					} else {
						boolfilterScreen.boolers[item.id] = true;
					}
					itemEntity.getItem().count -= input.count;
				} else if (!item && itemEntity.getItem().count >= input.count && input.id === itemEntity.getItem().id) {
					itemEntity.getItem().count -= input.count;
				}
			});
			global.PlaySound("minecraft:block.sculk_shrieker.shriek", 200, 1)
		}
	});
	block.level.server.scheduleInTicks(20, () => {

		recipes.output.forEach(E => {

			block.up.popItem(E)
		})
	})
	return false;
}


/**
 * 判断arr1是否是arr2的子集
 * @param {[]} arr1 
 * @param {[]} arr2 
 * @returns 
 */
const isSubset = (arr1, arr2) => {
    return arr1.every(elem => arr2.includes(elem));
}


/**
 * 配方选择器
 * 假设ItemList有[Item.of("minecraft:iron_sword"),Item.of("minecraft:iron_sword"),Item.of("morecategory:diamond_sickle"),Item.of("morecategory:diamond_sickle"),Item.of("minecraft:diamond_sword")]
 * 那么就选recipes中input为[Item.of("minecraft:iron_sword"),Item.of("morecategory:diamond_sickle"),Item.of("minecraft:diamond_sword")]物品最齐全的配方
 * 如果有多个符合则选择第一个
 * 注意recipe的input里面可以有相同元素如[Item.of("minecraft:iron_sword"),Item.of("minecraft:iron_sword")],则表示为有两个铁剑
 * 
 * 根据可用的物品选择最合适的配方。
 * @param {$ItemStack[]} ItemList - 可用物品的列表。
 * @param {{ input: $ItemStack[], Amount: number, output: $ItemStack[] }[]} recipes - 配方列表。
 * @returns {{ input: $ItemStack[], Amount: number, output: $ItemStack[] } | null} - 选择的配方或如果没有合适的配方则返回 null。
 */
const selectRecipe = (ItemList, recipes) => {
    // 将 ItemList 转换为物品计数映射
    let itemCountMap = {};
    for (let item of ItemList) {
        let itemId = item.id;
        if (!itemCountMap[itemId]) {
            itemCountMap[itemId] = 0;
        }
        itemCountMap[itemId] += item.count;
    }

    // 获取配方能满足的物品数
    const countFulfilledItems = (requiredItems) => {
        let itemCountMapCopy = Object.assign({}, itemCountMap);
        let fulfilledCount = 0;

        for (let item of requiredItems) {
            let requiredCount = item.count;
            let availableCount = itemCountMapCopy[item.id] || 0;

            if (availableCount >= requiredCount) {
                fulfilledCount += requiredCount;
                itemCountMapCopy[item.id] -= requiredCount;
            }
        }

        return fulfilledCount;
    };

    // 选择物品最齐全的配方
    let bestRecipe = null;
    let maxFulfilledCount = -1;

    for (let recipe of recipes) {
        let fulfilledCount = countFulfilledItems(recipe.input);

        if (fulfilledCount > maxFulfilledCount) {
            maxFulfilledCount = fulfilledCount;
            bestRecipe = recipe;
        }
    }

    // 如果找到最齐全的配方，则返回
    if (bestRecipe) {
        return bestRecipe;
    }

    // 如果没有找到合适的配方，则返回 null
    return null;
};
```

### 本体逻辑
```javascript
//PatchouliAPI多方块检测 actualCombat
BlockEvents.rightClicked((event) => {
    const { item, level, block, player, } = event

    const { pos } = block
    /*
    // 调试获取data
    if(item == Item.of('kubejs:bone_meal_battle')){
        event.player.runCommand(`data get block ${block.x} ${block.y} ${block.z}`)
    }
    */
    if (item != "minecraft:stick" || level.isClientSide()) {
        return
    }
    if (block.id == 'minecraft:sculk_shrieker') {
        let rotation = global.MULTIBLOCK.Test_MultiBlock_Machine().validate(level, pos)
        if (rotation === null) {
            return
        }
        let ItemList = global.getItemListToBe(pos, level)
        if (ItemList === undefined || ItemList.length == 0) {
            return
        }

        let recipe = selectRecipe(ItemList,recipes)
        // 判断有无储罐
        let ftBool = false
        // 判断有无魔力
        let ftBool2 = false

        // 通过ForgeCapabilities获取储罐的流体并作判断与处理
        let Amountreduce = () => {
            let offsetCoordinateSet = [
                [-2, 0, 2], [-2, 0, 1], [-2, 0, 0], [-2, 0, -1], [-2, 0, -2],
                [-1, 0, -2], [0, 0, -2], [1, 0, -2], [2, 0, -2],
                [2, 0, -1], [2, 0, 0], [2, 0, 1], [2, 0, 2],
                [1, 0, 2], [0, 0, 2], [-1, 0, 2]
                , [-2, -1, 2], [-2, -1, 1], [-2, -1, 0], [-2, -1, -1], [-2, -1, -2],
                [-1, -1, -2], [0, -1, -2], [1, -1, -2], [2, -1, -2],
                [2, -1, -1], [2, -1, 0], [2, -1, 1], [2, -1, 2],
                [1, -1, 2], [0, -1, 2], [-1, -1, 2]
            ]
            let ftPos = offsetCoordinateSet
                .filter(e => event.block.offset(e[0], e[1], e[2]).id == 'create:fluid_tank')[0]

            if (!ftPos) {
                player.statusMessage = Text.translate("tell.anningui.interesting.mana.need.0").darkRed()
                ftBool = true
                return
            }
            let ftBlock = event.block.offset(ftPos[0], ftPos[1], ftPos[2])
            let ftBlockBe = ftBlock.getEntity()
            let ftBlockCap = /**@type { $IFluidHandler }*/ (ftBlockBe.getCapability(ForgeCapabilities.FLUID_HANDLER).orElse(null))
            let ftFluid = ftBlockCap.getFluidInTank(0).fluid.arch$registryName()
            let ftAmount = ftBlockCap.getFluidInTank(0).amount
            if (ftFluid != 'cai:mana') {
                player.statusMessage = Text.translate("tell.anningui.interesting.mana.need.1").darkRed()
                ftBool2 = true
                return
            }
            if (ftAmount < recipe.Amount) {
                player.statusMessage = Text.translate("tell.anningui.interesting.mana.need.2").darkRed()
                return
            }
            if (isSubset(
                recipe.input.map(e => e.id), ItemList.map(i => i.id)
            ) == false) {
                return
            }
            ftBlockCap.drain(Fluid.of('cai:mana', recipe.Amount), "execute");
            ftBool = false
            ftBool2 = false
        }

        recipe.Amount && Amountreduce()
        if (ftBool) { return }
        if (ftBool2) { return }
        if (global.RemoveAndPopItemListToBe(block, pos, level, recipe)) { }
        else {
            return
        }
    }

})
```

### 测试配方(与本体逻辑同文件)
```javascript
let recipes = [{
    "input": [
        Item.of("minecraft:diamond_sword"),
        Item.of("morecategory:diamond_sickle"),
    ],
    "Amount": 1000,
    "output": [
        Item.of("3x minecraft:diamond")
    ]
}, {
    "input": [
        Item.of("minecraft:iron_sword"),
        Item.of("morecategory:diamond_sickle"),
    ],
    "Amount": 500,
    "output": [
        Item.of("3x minecraft:iron_ingot")
    ]
}, {
    "input": [
        Item.of("minecraft:iron_sword"),
        Item.of("morecategory:diamond_sickle"),
        Item.of("minecraft:diamond_sword")
    ],
    "Amount": 500,
    "output": [
        Item.of("3x minecraft:apple")
    ]
}];
```

> 语言文件请自行添加

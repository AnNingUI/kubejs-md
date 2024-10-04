## 关于curios的一些魔改想法
# in server_scripts
```js
/**
 * 返回对应槽位物品列表
 * @param {Internal.Player} player 
 * @param {string} slot
 * @returns {ItemList: [{}]}
 */
function getCuriosItemList(player,slot){
    let curio = player.nbt.ForgeCaps['curios:inventory']["Curios"].find(function(curio) {
		return curio["Identifier"] === slot;
	})
    return curio ? curio.StacksHandler.Stacks.Items : [];
}


/**
 * 返回是否有此物品在player的slot上，及物品数量，对应槽位号(该对应槽位的第几个)，对应槽位数量
 * @param {Internal.Player} player 
 * @param {string} slot
 * @param {string} id 
 * @returns {{hasItem: boolean, count: number, SlotNum: number, SlotSize: number}}
 */
function CuriosPlayer(player,slot,id){
	let result = { 
        hasItem: false, 
        count: 0, 
        SlotNum: 0, 
        SlotSize: 0
    };
	
    let ItemList = getCuriosItemList(player,slot)
	result.SlotSize = player.nbt.ForgeCaps['curios:inventory']["Curios"].find(function(curio) {
		return curio["Identifier"] === slot;
	}).StacksHandler.Cosmetics.Size
	ItemList.forEach(item => {if (item.id === id) { 
		result.hasItem = true;
        result.count += item.Count;
        result.SlotNum = item.Slot;
	}})
    return result;
}

//以下思路来源于群友落秋与curios源码

let $CuriosApi = Java.loadClass("top.theillusivec4.curios.api.CuriosApi")
/**
 * 
 * @param {"add"|"shrink"|"grow"|"getfor"|"setfor"|"unlock"|"lock"} method 
 * @param {string} slot 
 * @param {Internal.Player} player 
 * @param {Number} amount 
 * @returns 
 */
function CuriosSlotMethod(method,slot,player,amount){
    switch(method)
    {
        case "add":
            $CuriosApi.getSlotHelper().addSlotType(slot, amount, player)
            break;
        case "shrink":
            $CuriosApi.getSlotHelper().shrinkSlotType(slot, amount, player)
            break;
        case "grow":
            $CuriosApi.getSlotHelper().growSlotType(slot, amount, player)
            break;
        case "getfor":
            console.log($CuriosApi.getSlotHelper().getSlotsForType(player, slot))
            return $CuriosApi.getSlotHelper().getSlotsForType(player, slot)
        case "setfor":
            $CuriosApi.getSlotHelper().setSlotsForType(slot, player, amount)
            break;
        case "unlock":
            $CuriosApi.getSlotHelper().unlockSlotType(slot, player)
            break;
        case "lock":
            $CuriosApi.getSlotHelper().lockSlotType(slot, player)
            break;
    }
}

```
其实关于我自己写的那几个读nbt的方法可以替换为导包来解决且跟好，但我难的改了（）

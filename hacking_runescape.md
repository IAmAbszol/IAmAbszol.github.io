# RuneScape AI (Current Version: 0.1.0)

This notebook details the testing performed within the SmartieCollect plugin which deals with collecting data from the RuneLite client every game tick. A goal to complete within 0.1.0 is to have a workable Alching and Nightmare Zone AFK AI.

Alching:
- Proves clicking, noticing animation, and inventory items are noticed.
- Able to be consistent.

NMZ (AFK):
- Proves it notices more widgets such as health, absorption, etc.

## Feature Space

Currently the feature space provided by SmartieCollect consists of the following:

**Actor**
- x
- y
- z
- cx
- cy
- cz
- pitch
- yaw
- scale
- energy
- animation
- health

**Inventory**
- [...] [BLANK|(id, stack_size)]

**Mouse**
- mx
- my
- midle_ms (Time since last click, milliseconds)
- mclick (Left -> 1, Right -> 2)

## Elaborating Feature & Action Spaces

As of right now the feature space does contain bountiful information though the lack of insight appears to plague it, lacking information as to why or what we're clicking. Not only that, where we could memorize mouse cartesian coordinates at a specific state and use that mouse action but this would be large, quite large for an action space. Can we instead focus on what ID to select, if it were an NPC, GameObject, or Inventory Item? Yes.

To be able to transcibe this type of decision from a prediction to peripheral that interacts with RuneLite a knowledge set from the client must be used to properly gather the correct object to choose from.

![Prediction-Translate-Peripheral](./docs/hacking/diagrams/translation_peripheral.png)

### Action Space Decisions

The action space must vary between three states within this release.

* Inventory:
    - Mouse Click <-- 1 for Left, 2 for Right
    - Enum.INVENTORY <-- Identifies the option
    - Item ID <-- Id of item to click
    - In stack depth <-- If three items exist, which item should it click [0-2]

* GameObject
    - Mouse Click <-- 1 for Left, 2 for Right
    - Enum.GAME_OBJECT <-- Identifies the option
    - Object ID <-- Id of object to click
    - In stack depth <-- If three objects exist, which object should it click [0-2]

NPC interaction will be available in the next release, this information here is sufficient for Alching and NMZ AFK.

### Feature Space Updates

Because the action space now contains information suchn as stack depth, something inventory takes care of but not objects or clicking, we must update that. Another will be clicking, what is it that we just clicked and was it an Inventory and GameObject interface.

**Mouse**
- mx
- my
- midle_ms
- mclick
- **Interaction**
    - Type.Enum [INVENTORY, GAMEOBJECT]
    - interaction_id <-- Id of object clicked
    - stack_depth <-- Computed by distancing player and comparing id clicked.
-

**Objects**
- objects
    - [object_id]

### Stack Depth

This subsection is an important discussion to be had about how one can describe what item, out of a set of the same items, that was clicked.

Imagine if the AI was only spitting out an ID, it might become quite obvious that this AI isn't a human by the sheer nature of selecting the closest ID.
***Example***: A player has an inventory of [Fish, Fish, Fish], if the AI was to constantly pick from index 0 (0th indexed array) then a detection program would find this suspicious as no true thought is put into it.
***Solution***: Monitor the size of the stack of items held in that interface, this case inventory with (3) Fish. If the player averages to click Fish 2 and 3 around 50/50, then 1 would be the last choice said player would do.

How to compute stack depth comes down to evaluating all the objects or inventory items present in the scene, fortunately this only happens when a player interacts with an item and not every gametick (Which can become quite expensive). We should also begin to think about how we can organize the objects in such a way that an an intense object right environment would impact the RuneLite clients performance negatively.

## Translator

Translation between found game objects can be performed all within the SmartieCollect variant that's built which intercepts the messages and performs in-game actions. Later SmartieCollect will be renamed to match this feature.

Examples of translating the object click to mouse x, y coordinates, in turn we can take these and translate them using the viewport to offset our general mouses coordinates.

```java
// NPC ID to Mouse
for(NPC npc : npcs) {
    System.out.println(npc.getId() + "\t " + npc.getName());
    if(npc.getId() == menuOpt.getMenuEntry().getNpc().getId()) {
        Rectangle2D npcBounds = npc.getConvexHull().getBounds2D();
        System.out.println("Guessed on client mouse (x,y): " + npcBounds.getCenterX() + ", " + npcBounds.getCenterY());
    }
}

// GameObject ID to Mouse
try {
    System.out.println(menuOpt.getMenuTarget() + ", " + menuOpt.getId());
    for(TileObject object : objects.values()) {
        if(object.getId() == menuOpt.getId()) {
            GameObject go = (GameObject) object;
            Shape shape = go.getConvexHull();
            Rectangle2D rect = shape.getBounds2D();
            System.out.println("Guessed mouse (x,y): " + rect.getCenterX() + ", " + rect.getCenterY());
        }
    }
} catch (NullPointerException npe) {
    npe.printStackTrace();
}

// Item click to Mouse
if(menuOpt.isItemOp()) {
    Widget itemWidget = menuOpt.getWidget();
    Rectangle2D itemRect = itemWidget.getBounds().getBounds2D();
    if(itemRect != null) {
        System.out.println("Guessed mouse (x,y): " + itemRect.getCenterX() + ", " + itemRect.getCenterY());
    }
}

// Comparing Ids in Inventory
Widget[] widgets = client.getWidget(WidgetInfo.INVENTORY).getChildren();
for(Widget widget : widgets) {
    System.out.println(widget.getId());
}
```

> Codeblock 1 - Object clicked, converted to cartesian coordinate plane (x,y).


## Computing Stack Depth

Computing the stack depth of an object that was clicked, GameObjects and Inventory items alike.

### Inventory Items

This should be relatively straight forward as to how to compute which item was clicked in the stack of items in a players inventory.

```java
@Subscribe
public void onMenuOptionClicked(MenuOptionClicked menuOpt) {
    if(menuOpt.isItemOp()) {
        Widget[] widgets = client.getWidget(WidgetInfo.INVENTORY).getChildren();
        for(Widget widget : widgets) {
            if (menuOpt.getMenuEntry().getWidget().getBounds().contains(widget.getBounds())) {
                System.out.println(widget.getName()); // Print slot number of stack depth
            }
        }
    }
}
```
> Codeblock 2 - Performs intersection between entire inventory and item clicked, when hit, prints name.

### Objects

Objects of RuneScape, more particularly GameObjects aren't as easy but there's a commonality between how we determined Inventory and how it can be applied to GameObjects, NPCs, etc. That is GameObjects are able to be determined by distance from the player. Simply put, like Inventory where we start at the first slot and work our way to slot 28 (27 0th indexed), the distance from the start to i increases as it moves apart; GameObjects are the same idea.

We'll need a specialized data structure that holds the IDs of TileObjects (GameObjects inherit from) and can be drawn within constant time with the TileObject portion being sorted by distance at all times. Because of the odds in favor for us, there is very little chance a Scene will contain all the same IDs and derive a cost high enough to lag the RuneLite client downwards. A Priority Queue (Min-Heap) solution will be introduced where the distances between player and objects are drawn and sorted ONLY when interacted with.

```java
// Called on any object not Actor spawning/despawning from Scene.
private void onTileObject(Tile tile, TileObject oldObject, TileObject newObject)
{
    if(oldObject != null) {
        if(objects.containsKey(oldObject.getId())) {
            if(objects.get(oldObject.getId()).contains(oldObject)) {
                objects.get(oldObject.getId()).remove(oldObject);
            }
        }

        if(objects.get(oldObject.getId()).isEmpty()) {
            objects.remove(oldObject.getId());
        }
    }

    if (newObject == null)
    {
        return;
    }

    // Allow for quick lookup when Menu is clicked.
    if(!objects.containsKey(newObject.getId())) {
        objects.put(newObject.getId(), new HashSet<>());
    }
    objects.get(newObject.getId()).add(newObject);
}

@Subscribe
public void onMenuOptionClicked(MenuOptionClicked menuOpt) {
    // Craft custom comparator for PQ of TileObjects using distance
    Comparator<TileObject> tileObjectComparator = new Comparator<TileObject>() {
        @Override
        public int compare(TileObject t0, TileObject t1) {
            LocalPoint playerPoint = client.getLocalPlayer().getLocalLocation();
            int t0Distance = playerPoint.distanceTo(t0.getLocalLocation());
            int t1Distance = playerPoint.distanceTo(t1.getLocalLocation());
            return t0Distance - t1Distance;
        }
    };
    PriorityQueue<TileObject> distancePq = new PriorityQueue<>(tileObjectComparator);
    if(objects.containsKey(menuOpt.getId())) {
        for (TileObject object : objects.get(menuOpt.getId())) {
            if(object != null && ((GameObject) object).getConvexHull() != null) {
                distancePq.add(object);
            }
        }
        // Retrieve the intersection between the two, if found, print
        for(int index = 0; index < distancePq.size(); index++) {
            TileObject object = distancePq.poll();
            LocalPoint playerPoint = client.getLocalPlayer().getLocalLocation();
            System.out.println(playerPoint.distanceTo(object.getLocalLocation()));
            if (((GameObject) object).getConvexHull().contains(clientPoint.getX(), clientPoint.getY())) {
                System.out.println("Stack Depth " + index + "\t " + menuOpt.getMenuTarget());
            }
        }
    }
}
```
> Codeblock 3 - Finding the object/slot is bidirectional for finding both ways.

---
- [Home](./index.md)
- Blog by Kyle Darling

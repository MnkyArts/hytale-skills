---
name: hytale-ui-windows
description: Create custom UI windows, containers, and interactive interfaces for Hytale plugins. Use when asked to "create inventory UI", "make custom window", "add container interface", "build crafting UI", or "custom GUI".
metadata:
  author: hytale-modding
  version: "1.1.0"
---

# Hytale UI Windows

Complete guide for creating custom UI windows, container interfaces, and interactive menus in Hytale server plugins.

## When to use this skill

Use this skill when:
- Creating custom inventory windows
- Building container interfaces (chests, benches)
- Implementing crafting UI systems
- Making interactive menus
- Handling window actions and clicks
- Syncing window state between server and client

## Window Architecture Overview

Hytale uses a window system for server-controlled UI. Windows are opened server-side and rendered client-side, with actions sent back to the server for processing. Window data is transmitted as JSON and inventory contents are synced separately.

### Window Class Hierarchy

```
Window (abstract)
├── ContainerWindow                  # Simple item container (implements ItemContainerWindow)
├── ItemStackContainerWindow         # Container tied to an ItemStack (implements ItemContainerWindow)
├── FieldCraftingWindow              # Pocket/inventory crafting (WindowType.PocketCrafting)
├── MemoriesWindow                   # Memories/achievements display (WindowType.Memories)
└── BlockWindow (abstract)           # Tied to a block in the world (implements ValidatedWindow)
    ├── ContainerBlockWindow         # Container tied to a block (implements ItemContainerWindow)
    └── BenchWindow (abstract)       # Crafting bench base (implements MaterialContainerWindow)
        ├── ProcessingBenchWindow    # Furnace-like processing (implements ItemContainerWindow)
        └── CraftingWindow (abstract)
            ├── SimpleCraftingWindow       # Basic workbench crafting (implements MaterialContainerWindow)
            ├── DiagramCraftingWindow      # Blueprint/anvil crafting (implements ItemContainerWindow)
            └── StructuralCraftingWindow   # Block transformation crafting (implements ItemContainerWindow)
```

### Key Interfaces

| Interface | Purpose |
|-----------|---------|
| `ItemContainerWindow` | Windows with item inventory slots |
| `MaterialContainerWindow` | Windows with extra resource materials |
| `ValidatedWindow` | Windows that validate state (e.g., player distance) |

### Window Types (WindowType Enum)

| WindowType | Value | Description | Use Case |
|------------|-------|-------------|----------|
| `Container` | 0 | Item storage | Chests, backpacks |
| `PocketCrafting` | 1 | Field crafting | Player inventory crafting |
| `BasicCrafting` | 2 | Standard crafting | Crafting tables |
| `DiagramCrafting` | 3 | Blueprint-based | Advanced workbenches, anvils |
| `StructuralCrafting` | 4 | Block transformation | Stonecutters, construction benches |
| `Processing` | 5 | Time-based conversion | Furnaces, smelters |
| `Memories` | 6 | Special display | Memory/achievement UI |

### Window Flow

```
Server: openWindow(window) -> OpenWindow packet (ID 200) -> Client: Render UI
Client: User Action -> SendWindowAction packet (ID 203) -> Server: handleAction()
Server: invalidate() -> updateWindows() -> UpdateWindow packet (ID 201) -> Client: Refresh UI
Server: closeWindow() -> CloseWindow packet (ID 202) -> Client: Close UI
```

### Window Data Pattern

Windows use `getData()` to return a `JsonObject` that is serialized and sent to the client. This data controls client-side rendering:

```java
@Override
public JsonObject getData() {
    JsonObject data = new JsonObject();
    data.addProperty("type", windowType.ordinal());
    data.addProperty("title", "My Window");
    data.addProperty("customProperty", someValue);
    return data;
}
```

## Basic Window Implementation

### Abstract Window Base

All windows extend from `Window` and must implement these abstract methods:

```java
package com.example.myplugin.windows;

import com.google.gson.JsonObject;
import com.hypixel.hytale.server.core.entity.entities.player.windows.Window;
import com.hypixel.hytale.protocol.packets.window.WindowType;
import com.hypixel.hytale.protocol.packets.window.WindowAction;
import com.hypixel.hytale.component.Ref;
import com.hypixel.hytale.component.Store;
import com.hypixel.hytale.server.core.universe.world.storage.EntityStore;

public class CustomWindow extends Window {
    
    private final JsonObject windowData = new JsonObject();
    
    public CustomWindow() {
        super(WindowType.Container);
        // Initialize window data
        windowData.addProperty("title", "Custom Window");
    }
    
    @Override
    public JsonObject getData() {
        // Return data to send to client (serialized as JSON)
        return windowData;
    }
    
    @Override
    protected boolean onOpen0() {
        // Called when window opens
        // Return false to cancel opening
        return true;
    }
    
    @Override
    protected void onClose0() {
        // Called when window closes - cleanup here
    }
    
    @Override
    public void handleAction(Ref<EntityStore> ref, Store<EntityStore> store, WindowAction action) {
        // Handle window actions from client
        // Default implementation is no-op
    }
}
```

### Opening Windows

Windows are opened through the `WindowManager`:

```java
public class StorageCommand extends AbstractPlayerCommand {
    
    public StorageCommand() {
        super("storage", "Open storage window");
    }
    
    @Override
    protected void execute(CommandContext ctx, Player player) {
        StorageWindow window = new StorageWindow();
        
        // Open via WindowManager
        WindowManager windowManager = player.getWindowManager();
        OpenWindow packet = windowManager.openWindow(window);
        
        if (packet != null) {
            // Window opened successfully - packet is sent automatically
            player.sendMessage("Window opened!");
        } else {
            // Opening was cancelled (onOpen0() returned false)
            player.sendMessage("Failed to open window");
        }
    }
}
```

### Updating Windows

Mark a window as needing update with `invalidate()`:

```java
public void updateData(String newValue) {
    windowData.addProperty("value", newValue);
    invalidate(); // Mark for update
}

// For full rebuild (client re-renders entire window)
public void requireRebuild() {
    setNeedRebuild();
    invalidate();
}
```

Updates are batched and sent via `WindowManager.updateWindows()` which checks `isDirty` flag.

## Window Manager

The `WindowManager` handles window lifecycle for each player:

```java
// Get player's window manager
WindowManager windowManager = player.getWindowManager();

// Open a window (returns OpenWindow packet or null if cancelled)
OpenWindow packet = windowManager.openWindow(new MyWindow());

// Open multiple windows atomically (all or none)
List<OpenWindow> packets = windowManager.openWindows(window1, window2);

// Get window by ID
Window window = windowManager.getWindow(windowId);

// Get all open windows
List<Window> windows = windowManager.getWindows();

// Update a specific window (sends UpdateWindow packet)
windowManager.updateWindow(window);

// Update all dirty windows
windowManager.updateWindows();

// Validate all ValidatedWindow instances (closes invalid ones)
windowManager.validateWindows();

// Close a specific window
windowManager.closeWindow(windowId);

// Close all windows
windowManager.closeAllWindows();

// Mark a window as changed
windowManager.markWindowChanged(windowId);
```

### Window IDs

- ID `0` is reserved for client-requested windows
- ID `-1` is invalid
- Server-assigned IDs start at 1 and increment

## Block Windows

Windows tied to blocks in the world (chests, crafting tables). Extends `BlockWindow` which implements `ValidatedWindow`:

```java
public class CustomChestWindow extends BlockWindow implements ItemContainerWindow {
    
    private final SimpleItemContainer itemContainer;
    private final JsonObject windowData = new JsonObject();
    
    public CustomChestWindow(int x, int y, int z, int rotationIndex, BlockType blockType) {
        super(WindowType.Container, x, y, z, rotationIndex, blockType);
        this.itemContainer = new SimpleItemContainer(27); // 3 rows
        
        // Set max interaction distance (default: 7.0)
        setMaxDistance(7.0);
        
        // Initialize window data
        Item item = blockType.getItem();
        windowData.addProperty("blockItemId", item != null ? item.getId() : "");
    }
    
    @Override
    public JsonObject getData() {
        return windowData;
    }
    
    @Override
    public ItemContainer getItemContainer() {
        return itemContainer;
    }
    
    @Override
    protected boolean onOpen0() {
        // Load chest contents from block entity
        PlayerRef playerRef = getPlayerRef();
        Ref<EntityStore> ref = playerRef.getReference();
        Store<EntityStore> store = ref.getStore();
        World world = store.getExternalData().getWorld();
        
        // Load items from persistent storage
        loadItemsFromWorld(world);
        return true;
    }
    
    @Override
    protected void onClose0() {
        // Save chest contents
        saveItemsToWorld();
    }
}
```

### Block Validation

`BlockWindow` automatically validates that:
1. Player is within `maxDistance` of the block (default 7.0 blocks)
2. The block still exists in the world
3. The block type matches (via item comparison)

When validation fails, the window is automatically closed.

### Block Interaction Handler

```java
@EventHandler
public void onBlockInteract(BlockInteractEvent event) {
    Player player = event.getPlayer();
    BlockPos pos = event.getBlockPos();
    Block block = event.getBlock();
    
    if (block.getType().getId().equals("my_mod:custom_chest")) {
        CustomChestWindow window = new CustomChestWindow(
            pos.x(), pos.y(), pos.z(),
            block.getRotationIndex(),
            block.getType()
        );
        player.getWindowManager().openWindow(window);
        event.setCancelled(true);
    }
}
```

## Crafting Windows

### BenchWindow Base

All crafting bench windows extend `BenchWindow`:

```java
public abstract class BenchWindow extends BlockWindow implements MaterialContainerWindow {
    protected final Bench bench;
    protected final BenchState benchState;
    protected final JsonObject windowData = new JsonObject();
    private MaterialExtraResourcesSection extraResourcesSection;
    
    // Window data includes:
    // - type: bench type ordinal
    // - id: bench ID string
    // - name: translation key
    // - blockItemId: item ID
    // - tierLevel: current tier level
    // - worldMemoriesLevel: world memories level
    // - progress: crafting progress (0.0 - 1.0)
    // - tierUpgradeProgress: tier upgrade progress
}
```

### SimpleCraftingWindow (Basic Workbench)

```java
public class WorkbenchWindow extends SimpleCraftingWindow {
    
    public WorkbenchWindow(BenchState benchState) {
        super(benchState);
    }
    
    @Override
    public void handleAction(Ref<EntityStore> ref, Store<EntityStore> store, WindowAction action) {
        if (action instanceof CraftRecipeAction craftAction) {
            String recipeId = craftAction.recipeId;
            int quantity = craftAction.quantity;
            // Handle crafting
            CraftingManager craftingManager = store.getComponent(ref, CraftingManager.getComponentType());
            craftSimpleItem(store, ref, craftingManager, craftAction);
        } else if (action instanceof TierUpgradeAction) {
            // Handle bench tier upgrade
            handleTierUpgrade(ref, store);
        }
    }
}
```

### ProcessingBenchWindow (Furnace-like)

```java
public class SmelterWindow extends ProcessingBenchWindow {
    
    public SmelterWindow(BenchState benchState) {
        super(benchState);
    }
    
    // ProcessingBenchWindow provides:
    // - setActive(boolean): toggle processing
    // - setProgress(float): update progress (0.0 - 1.0)
    // - setFuelTime(float): current fuel remaining
    // - setMaxFuel(int): maximum fuel capacity
    // - setProcessingSlots(Set<Short>): slots currently processing
    // - setProcessingFuelSlots(Set<Short>): fuel slots in use
    
    @Override
    public void handleAction(Ref<EntityStore> ref, Store<EntityStore> store, WindowAction action) {
        if (action instanceof SetActiveAction activeAction) {
            setActive(activeAction.state);
            invalidate();
        } else if (action instanceof TierUpgradeAction) {
            handleTierUpgrade(ref, store);
        }
    }
}
```

### Updating Crafting Progress

```java
// Update progress with throttling (min 5% change or 500ms interval)
public void updateCraftingJob(float percent) {
    windowData.addProperty("progress", percent);
    checkProgressInvalidate(percent);
}

public void updateBenchUpgradeJob(float percent) {
    windowData.addProperty("tierUpgradeProgress", percent);
    checkProgressInvalidate(percent);
}

// On tier level change (requires full rebuild)
public void updateBenchTierLevel(int newValue) {
    windowData.addProperty("tierLevel", newValue);
    updateBenchUpgradeJob(0.0f);
    setNeedRebuild();
    invalidate();
}
```

## Item Container Windows

Windows with inventory slots implement `ItemContainerWindow`:

```java
public interface ItemContainerWindow {
    @Nonnull ItemContainer getItemContainer();
}
```

### ItemContainer Integration

```java
public class InventoryWindow extends Window implements ItemContainerWindow {
    
    private final SimpleItemContainer itemContainer;
    private final JsonObject windowData = new JsonObject();
    
    public InventoryWindow(int size) {
        super(WindowType.Container);
        this.itemContainer = new SimpleItemContainer(size);
        
        // Register change listener for automatic updates
        itemContainer.registerChangeEvent(EventPriority.NORMAL, event -> {
            invalidate();
        });
    }
    
    @Override
    public ItemContainer getItemContainer() {
        return itemContainer;
    }
    
    @Override
    public JsonObject getData() {
        return windowData;
    }
    
    @Override
    protected boolean onOpen0() {
        return true;
    }
    
    @Override
    protected void onClose0() {
        // Cleanup
    }
}
```

**Note:** When a window implements `ItemContainerWindow`, the `WindowManager` automatically:
1. Registers a change listener to mark the window dirty when inventory changes
2. Includes `InventorySection` in `OpenWindow` and `UpdateWindow` packets
3. Unregisters the listener when the window closes

## Window Actions

Handle user interactions with `handleAction()`:

```java
@Override
public void handleAction(Ref<EntityStore> ref, Store<EntityStore> store, WindowAction action) {
    if (action instanceof CraftRecipeAction craft) {
        handleCraft(craft.recipeId, craft.quantity);
    } else if (action instanceof SelectSlotAction select) {
        handleSlotSelect(select.slot);
    } else if (action instanceof SetActiveAction active) {
        handleActiveToggle(active.state);
    } else if (action instanceof SortItemsAction sort) {
        handleSort(sort.sortType);
    }
}
```

### WindowAction Types

| Type ID | Class | Fields | Description |
|---------|-------|--------|-------------|
| 0 | `CraftRecipeAction` | `recipeId: String`, `quantity: int` | Craft a recipe |
| 1 | `TierUpgradeAction` | (none) | Upgrade bench tier |
| 2 | `SelectSlotAction` | `slot: int` | Select a slot |
| 3 | `ChangeBlockAction` | `down: boolean` | Cycle block type direction |
| 4 | `SetActiveAction` | `state: boolean` | Toggle processing on/off |
| 5 | `CraftItemAction` | (none) | Confirm diagram crafting |
| 6 | `UpdateCategoryAction` | `category: String`, `itemCategory: String` | Change recipe category |
| 7 | `CancelCraftingAction` | (none) | Cancel current crafting |
| 8 | `SortItemsAction` | `sortType: SortType` | Sort inventory items |

### SortType Enum

```java
public enum SortType {
    Name(0),   // Sort by item translation key
    Type(1),   // Sort by item type (Weapon, Armor, Tool, Item, Special)
    Rarity(2); // Sort by quality value (reversed)
}
```

## Window Packets

Network communication for windows:

### Server to Client

| Packet | ID | Fields | Purpose |
|--------|----|--------|---------|
| `OpenWindow` | 200 | `id`, `windowType`, `windowData`, `inventory`, `extraResources` | Open window on client |
| `UpdateWindow` | 201 | `id`, `windowData`, `inventory`, `extraResources` | Update window contents |
| `CloseWindow` | 202 | `id` | Close window on client |

### Client to Server

| Packet | ID | Fields | Purpose |
|--------|----|--------|---------|
| `SendWindowAction` | 203 | `id`, `action: WindowAction` | User interaction |
| `ClientOpenWindow` | 204 | `type: WindowType` | Request client-initiated window |

### Packet Structure

The `OpenWindow` packet includes:
- `windowData`: JSON string with window-specific data
- `inventory`: `InventorySection` (nullable) - only for `ItemContainerWindow`
- `extraResources`: `ExtraResources` (nullable) - only for `MaterialContainerWindow`

```java
// Creating OpenWindow packet (done automatically by WindowManager)
OpenWindow packet = new OpenWindow(
    windowId,
    window.getType(),
    window.getData().toString(),  // JSON string
    itemContainerWindow != null ? itemContainerWindow.getItemContainer().toPacket() : null,
    materialContainerWindow != null ? materialContainerWindow.getExtraResourcesSection().toPacket() : null
);
```

## Client-Requestable Windows

Some windows can be opened by client request (e.g., pressing a key). Register these in `Window.CLIENT_REQUESTABLE_WINDOW_TYPES`:

```java
public class MyPlugin extends JavaPlugin {
    
    @Override
    protected void setup() {
        // Register client-requestable window
        Window.CLIENT_REQUESTABLE_WINDOW_TYPES.put(
            WindowType.Memories,
            MemoriesWindow::new
        );
    }
}
```

When client sends `ClientOpenWindow` packet, the server:
1. Looks up the `WindowType` in `CLIENT_REQUESTABLE_WINDOW_TYPES`
2. Creates a new window instance using the supplier
3. Opens it with ID 0 via `windowManager.clientOpenWindow(window)`

```java
// Handle client-requested window
@PacketHandler
public void onClientOpenWindow(ClientOpenWindow packet) {
    Supplier<? extends Window> supplier = Window.CLIENT_REQUESTABLE_WINDOW_TYPES.get(packet.type);
    if (supplier != null) {
        Window window = supplier.get();
        UpdateWindow updatePacket = windowManager.clientOpenWindow(window);
        if (updatePacket != null) {
            player.sendPacket(updatePacket);
        }
    }
}
```

## Custom Window Rendering

Define window appearance through `getData()`:

```java
public class CustomMenuWindow extends Window {
    
    private final JsonObject windowData = new JsonObject();
    
    public CustomMenuWindow() {
        super(WindowType.Container);
        setupLayout();
    }
    
    @Override
    public JsonObject getData() {
        return windowData;
    }
    
    private void setupLayout() {
        windowData.addProperty("title", "Main Menu");
        windowData.addProperty("rows", 6);
        
        // Add custom properties for client rendering
        JsonArray menuItems = new JsonArray();
        menuItems.add(createMenuItem("pvp", "PvP Arena", "diamond_sword", 20));
        menuItems.add(createMenuItem("survival", "Survival", "grass_block", 22));
        menuItems.add(createMenuItem("lobby", "Lobby", "ender_pearl", 24));
        windowData.add("menuItems", menuItems);
    }
    
    private JsonObject createMenuItem(String id, String name, String icon, int slot) {
        JsonObject item = new JsonObject();
        item.addProperty("id", id);
        item.addProperty("name", name);
        item.addProperty("icon", icon);
        item.addProperty("slot", slot);
        return item;
    }
    
    @Override
    public void handleAction(Ref<EntityStore> ref, Store<EntityStore> store, WindowAction action) {
        if (action instanceof SelectSlotAction select) {
            switch (select.slot) {
                case 20 -> joinPvP(ref, store);
                case 22 -> joinSurvival(ref, store);
                case 24 -> teleportToLobby(ref, store);
            }
        }
    }
    
    @Override
    protected boolean onOpen0() { return true; }
    
    @Override
    protected void onClose0() { }
}
```

## Material Container Windows

Windows with extra resource materials implement `MaterialContainerWindow`:

```java
public interface MaterialContainerWindow {
    @Nonnull MaterialExtraResourcesSection getExtraResourcesSection();
    void invalidateExtraResources();
    boolean isValid();
}
```

### MaterialExtraResourcesSection

```java
public class MaterialExtraResourcesSection {
    private boolean valid;
    private ItemContainer itemContainer;
    private ItemQuantity[] extraMaterials;
    
    // Methods
    public void setExtraMaterials(ItemQuantity[] materials);
    public ExtraResources toPacket();
    public boolean isValid();
    public void setValid(boolean valid);
}
```

Usage in crafting windows:

```java
@Override
public MaterialExtraResourcesSection getExtraResourcesSection() {
    if (!extraResourcesSection.isValid()) {
        // Recompute extra materials from bench state
        CraftingManager.feedExtraResourcesSection(benchState, extraResourcesSection);
    }
    return extraResourcesSection;
}

@Override
public void invalidateExtraResources() {
    extraResourcesSection.setValid(false);
    invalidate();
}
```

## Close Event Registration

Register handlers for when a window closes:

```java
public class MyWindow extends Window {
    
    @Override
    protected boolean onOpen0() {
        // Register close event handler
        registerCloseEvent(event -> {
            // Called when window closes
            saveData();
            cleanupResources();
        });
        
        // With priority
        registerCloseEvent(EventPriority.FIRST, event -> {
            // Called first
        });
        
        return true;
    }
}
```

## Complete Example: Container Block Window

```java
package com.example.storage;

import com.google.gson.JsonObject;
import com.hypixel.hytale.component.Ref;
import com.hypixel.hytale.component.Store;
import com.hypixel.hytale.protocol.packets.window.WindowAction;
import com.hypixel.hytale.protocol.packets.window.WindowType;
import com.hypixel.hytale.protocol.packets.window.SortItemsAction;
import com.hypixel.hytale.server.core.asset.type.blocktype.config.BlockType;
import com.hypixel.hytale.server.core.entity.entities.player.windows.BlockWindow;
import com.hypixel.hytale.server.core.entity.entities.player.windows.ItemContainerWindow;
import com.hypixel.hytale.server.core.inventory.container.ItemContainer;
import com.hypixel.hytale.server.core.inventory.container.SimpleItemContainer;
import com.hypixel.hytale.server.core.inventory.container.SortType;
import com.hypixel.hytale.server.core.universe.world.storage.EntityStore;

public class StorageBlockWindow extends BlockWindow implements ItemContainerWindow {
    
    private final SimpleItemContainer itemContainer;
    private final JsonObject windowData = new JsonObject();
    
    public StorageBlockWindow(int x, int y, int z, int rotationIndex, BlockType blockType, int rows) {
        super(WindowType.Container, x, y, z, rotationIndex, blockType);
        this.itemContainer = new SimpleItemContainer(rows * 9);
        
        // Initialize window data
        windowData.addProperty("title", "Storage");
        windowData.addProperty("rows", rows);
        windowData.addProperty("blockItemId", blockType.getItem().getId());
    }
    
    @Override
    public JsonObject getData() {
        return windowData;
    }
    
    @Override
    public ItemContainer getItemContainer() {
        return itemContainer;
    }
    
    @Override
    protected boolean onOpen0() {
        // Load items from persistent storage
        loadFromStorage();
        return true;
    }
    
    @Override
    protected void onClose0() {
        // Save items to persistent storage
        saveToStorage();
    }
    
    @Override
    public void handleAction(Ref<EntityStore> ref, Store<EntityStore> store, WindowAction action) {
        if (action instanceof SortItemsAction sort) {
            SortType serverSortType = SortType.fromPacket(sort.sortType);
            itemContainer.sort(serverSortType);
            invalidate();
        }
    }
    
    private void loadFromStorage() {
        // Load from block entity or database
    }
    
    private void saveToStorage() {
        // Save to block entity or database
    }
}
```

### Usage

```java
@EventHandler
public void onBlockInteract(BlockInteractEvent event) {
    Block block = event.getBlock();
    
    if (block.getType().getId().equals("my_mod:storage_block")) {
        StorageBlockWindow window = new StorageBlockWindow(
            event.getX(), event.getY(), event.getZ(),
            block.getRotationIndex(),
            block.getType(),
            3 // 3 rows
        );
        
        Player player = event.getPlayer();
        OpenWindow packet = player.getWindowManager().openWindow(window);
        
        if (packet != null) {
            event.setCancelled(true);
        }
    }
}
```

## Best Practices

### State Management

```java
// Always invalidate after modifications
public void updateValue(String key, Object value) {
    windowData.addProperty(key, value.toString());
    invalidate(); // Mark for next update cycle
}

// For structural changes, use setNeedRebuild
public void rebuildCategories() {
    recalculateCategories();
    setNeedRebuild(); // Client will re-render entire window
    invalidate();
}
```

### Resource Cleanup

```java
@Override
protected void onClose0() {
    // Cancel scheduled tasks
    if (updateTask != null) {
        updateTask.cancel(false);
    }
    
    // Save state
    saveToDatabase();
    
    // Return items to player if needed
    returnItemsToPlayer();
    
    // Unregister event listeners (if manually registered)
}
```

### Thread Safety

Window operations should be on the main server thread:

```java
public void updateFromAsync(Data data) {
    server.getScheduler().runTask(() -> {
        applyData(data);
        invalidate();
    });
}
```

### Progress Update Throttling

For windows with progress bars (like crafting), throttle updates:

```java
private static final float MIN_PROGRESS_CHANGE = 0.05f;
private static final long MIN_UPDATE_INTERVAL_MS = 500L;
private float lastUpdatePercent;
private long lastUpdateTimeMs;

private void checkProgressInvalidate(float percent) {
    if (lastUpdatePercent != percent) {
        long time = System.currentTimeMillis();
        if (percent >= 1.0f ||
            percent < lastUpdatePercent ||
            percent - lastUpdatePercent > MIN_PROGRESS_CHANGE ||
            time - lastUpdateTimeMs > MIN_UPDATE_INTERVAL_MS ||
            lastUpdateTimeMs == 0L) {
            
            lastUpdatePercent = percent;
            lastUpdateTimeMs = time;
            invalidate();
        }
    }
}
```

## Troubleshooting

### Window Not Opening

1. Check `onOpen0()` returns `true`
2. Verify WindowType is valid
3. Check for exceptions in initialization
4. Ensure WindowManager.openWindow() is called on correct thread

### Items Not Updating

1. Call `invalidate()` after modifications
2. Verify window implements `ItemContainerWindow` correctly
3. Check `WindowManager.updateWindows()` is being called (usually automatic)
4. Verify `getItemContainer()` returns the correct container

### Actions Not Received

1. Ensure `handleAction()` is implemented
2. Check action type casting (use `instanceof` pattern matching)
3. Verify window ID matches in client packets

### Window Closing Unexpectedly

For `BlockWindow` subclasses:
1. Check player is within `maxDistance` (default 7.0)
2. Verify block still exists at position
3. Ensure block type hasn't changed

## Detailed References

For comprehensive documentation:

- `references/window-types.md` - All window types with configuration options
- `references/slot-handling.md` - Item containers, sorting, and inventory handling

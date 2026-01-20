# Slot Handling Reference

Complete reference for window slot behaviors, item filters, and interaction handling.

## Slot Basics

Slots are indexed positions in a window where items can be placed or displayed.

```java
public class MyWindow extends Window {
    
    public MyWindow(Player player) {
        super(WindowType.Container, 27); // 27 slots (3 rows)
    }
    
    // Get item in slot
    ItemStack item = getItem(0);
    
    // Set item in slot
    setItem(0, new ItemStack(Items.DIAMOND));
    
    // Clear slot
    setItem(0, ItemStack.EMPTY);
    
    // Clear all slots
    clearAll();
}
```

## Slot Layout Patterns

### Grid Layout

```java
// 9-slot rows
// Row 0: slots 0-8
// Row 1: slots 9-17
// Row 2: slots 18-26

public int getSlotIndex(int row, int col) {
    return row * 9 + col;
}

public int getRow(int slot) {
    return slot / 9;
}

public int getColumn(int slot) {
    return slot % 9;
}
```

### Border Pattern

```java
public void fillBorder(ItemStack borderItem) {
    int rows = getSize() / 9;
    
    for (int i = 0; i < 9; i++) {
        setItem(i, borderItem);                    // Top row
        setItem((rows - 1) * 9 + i, borderItem);   // Bottom row
    }
    
    for (int row = 1; row < rows - 1; row++) {
        setItem(row * 9, borderItem);              // Left column
        setItem(row * 9 + 8, borderItem);          // Right column
    }
}
```

### Centered Content

```java
public void setCenteredItem(int row, ItemStack item) {
    setItem(row * 9 + 4, item); // Center column
}

public void setCenteredRow(ItemStack... items) {
    int start = 4 - (items.length / 2);
    for (int i = 0; i < items.length; i++) {
        setItem(start + i, items[i]);
    }
}
```

## Slot Handlers

Define custom behavior for specific slots:

```java
public interface SlotHandler {
    boolean canPlace(ItemStack item);
    boolean canTake();
    void onPlace(Player player, ItemStack item);
    void onTake(Player player, ItemStack item);
}

// Usage
public class FilteredSlotHandler implements SlotHandler {
    
    private final Predicate<ItemStack> filter;
    
    public FilteredSlotHandler(Predicate<ItemStack> filter) {
        this.filter = filter;
    }
    
    @Override
    public boolean canPlace(ItemStack item) {
        return filter.test(item);
    }
    
    @Override
    public boolean canTake() {
        return true;
    }
    
    @Override
    public void onPlace(Player player, ItemStack item) {
        // Custom logic when item placed
    }
    
    @Override
    public void onTake(Player player, ItemStack item) {
        // Custom logic when item taken
    }
}

// Applying to window
public class MyWindow extends Window {
    
    @Override
    protected void setupSlots() {
        // Only allow weapons in slot 0
        setSlotHandler(0, new FilteredSlotHandler(
            item -> item.hasTag(ItemTags.WEAPON)
        ));
        
        // Output-only slot
        setSlotHandler(9, new OutputOnlySlot());
        
        // Read-only display slots
        for (int i = 18; i < 27; i++) {
            setSlotHandler(i, new ReadOnlySlot());
        }
    }
}
```

## Preset Slot Handlers

### OutputOnlySlot

Items can only be taken, not placed:

```java
public class OutputOnlySlot implements SlotHandler {
    
    @Override
    public boolean canPlace(ItemStack item) {
        return false;
    }
    
    @Override
    public boolean canTake() {
        return true;
    }
}
```

### ReadOnlySlot

Items cannot be taken or placed (display only):

```java
public class ReadOnlySlot implements SlotHandler {
    
    @Override
    public boolean canPlace(ItemStack item) {
        return false;
    }
    
    @Override
    public boolean canTake() {
        return false;
    }
}
```

### InputOnlySlot

Items can only be placed, not taken:

```java
public class InputOnlySlot implements SlotHandler {
    
    @Override
    public boolean canPlace(ItemStack item) {
        return true;
    }
    
    @Override
    public boolean canTake() {
        return false;
    }
}
```

## Item Filters

### By Item Type

```java
// Single item type
Predicate<ItemStack> diamondOnly = item -> item.is(Items.DIAMOND);

// Multiple item types
Set<Item> allowedItems = Set.of(Items.DIAMOND, Items.EMERALD, Items.GOLD_INGOT);
Predicate<ItemStack> gemFilter = item -> allowedItems.contains(item.getItem());
```

### By Item Tag

```java
// Using item tags
Predicate<ItemStack> weaponFilter = item -> item.hasTag(ItemTags.WEAPON);
Predicate<ItemStack> armorFilter = item -> item.hasTag(ItemTags.ARMOR);
Predicate<ItemStack> fuelFilter = item -> item.hasTag(ItemTags.FUEL);

// Multiple tags
Predicate<ItemStack> combatGear = item -> 
    item.hasTag(ItemTags.WEAPON) || item.hasTag(ItemTags.ARMOR);
```

### By Item Properties

```java
// By durability
Predicate<ItemStack> undamaged = item -> 
    !item.isDamageable() || item.getDamage() == 0;

// By enchantment
Predicate<ItemStack> enchanted = item -> 
    !item.getEnchantments().isEmpty();

// By stack size
Predicate<ItemStack> fullStack = item -> 
    item.getCount() >= item.getMaxStackSize();

// By custom NBT
Predicate<ItemStack> hasSoulbound = item -> 
    item.hasTag() && item.getTag().getBoolean("Soulbound");
```

### Combining Filters

```java
Predicate<ItemStack> filter = weaponFilter
    .and(undamaged)
    .and(enchanted);

// Or use composite
Predicate<ItemStack> anyGear = weaponFilter.or(armorFilter);
```

## Slot Interaction Methods

### canPlaceItem

Called when player tries to place item:

```java
@Override
public boolean canPlaceItem(int slot, ItemStack item) {
    // Check slot-specific rules
    SlotHandler handler = getSlotHandler(slot);
    if (handler != null && !handler.canPlace(item)) {
        return false;
    }
    
    // Check window-wide rules
    if (isDisplaySlot(slot)) {
        return false;
    }
    
    // Check item-specific rules
    if (slot == FUEL_SLOT && !isFuel(item)) {
        return false;
    }
    
    return true;
}
```

### canTakeItem

Called when player tries to take item:

```java
@Override
public boolean canTakeItem(int slot) {
    // Output slot requires valid recipe
    if (slot == OUTPUT_SLOT) {
        return hasValidRecipe();
    }
    
    // Display slots are read-only
    if (isDisplaySlot(slot)) {
        return false;
    }
    
    return true;
}
```

### onSlotChange

Called after slot contents change:

```java
@Override
protected void onSlotChange(int slot, ItemStack oldItem, ItemStack newItem) {
    // Update crafting output when input changes
    if (isInputSlot(slot)) {
        updateCraftingResult();
    }
    
    // Trigger effects when item placed
    if (slot == ALTAR_SLOT && !newItem.isEmpty()) {
        triggerAltarEffect(newItem);
    }
    
    // Notify listeners
    notifySlotListeners(slot, oldItem, newItem);
}
```

## Click Handling

### Click Types

| Button | Shift | Action |
|--------|-------|--------|
| 0 (Left) | No | Pick up / Place stack |
| 0 (Left) | Yes | Quick move to other inventory |
| 1 (Right) | No | Pick up half / Place one |
| 1 (Right) | Yes | Quick move one item |
| Middle | No | Clone item (creative) |

### Processing Clicks

```java
@Override
protected void onAction(Player player, WindowAction action) {
    if (action instanceof ClickSlotAction click) {
        int slot = click.getSlot();
        int button = click.getButton();
        boolean shift = click.isShift();
        
        if (button == 0 && !shift) {
            handleLeftClick(player, slot);
        } else if (button == 0 && shift) {
            handleShiftClick(player, slot);
        } else if (button == 1) {
            handleRightClick(player, slot);
        }
    }
}
```

### Custom Click Actions

```java
private void handleLeftClick(Player player, int slot) {
    ItemStack cursor = player.getCursorItem();
    ItemStack slotItem = getItem(slot);
    
    if (isActionSlot(slot)) {
        // Trigger action instead of item manipulation
        performAction(player, slot);
        return;
    }
    
    if (cursor.isEmpty()) {
        // Pick up item
        if (canTakeItem(slot)) {
            player.setCursorItem(slotItem);
            setItem(slot, ItemStack.EMPTY);
        }
    } else {
        // Place item
        if (canPlaceItem(slot, cursor)) {
            if (slotItem.isEmpty()) {
                setItem(slot, cursor);
                player.setCursorItem(ItemStack.EMPTY);
            } else if (canStack(slotItem, cursor)) {
                // Merge stacks
                int space = slotItem.getMaxStackSize() - slotItem.getCount();
                int toAdd = Math.min(space, cursor.getCount());
                slotItem.grow(toAdd);
                cursor.shrink(toAdd);
            } else {
                // Swap items
                player.setCursorItem(slotItem);
                setItem(slot, cursor);
            }
        }
    }
    
    sendSlotUpdate(player, slot);
}
```

## Quick Move (Shift-Click)

Handle shift-click transfers:

```java
@Override
protected void onQuickMove(Player player, int slot) {
    ItemStack item = getItem(slot);
    if (item.isEmpty()) return;
    
    // Determine destination
    int destStart, destEnd;
    
    if (isPlayerInventorySlot(slot)) {
        // Move to window slots
        destStart = 0;
        destEnd = PLAYER_INV_START;
    } else {
        // Move to player inventory
        destStart = PLAYER_INV_START;
        destEnd = getSize();
    }
    
    // Try to move item
    ItemStack remaining = tryMoveItem(item, destStart, destEnd);
    setItem(slot, remaining);
    
    sendFullUpdate(player);
}

private ItemStack tryMoveItem(ItemStack item, int start, int end) {
    // First pass: try to stack with existing
    for (int i = start; i < end && !item.isEmpty(); i++) {
        ItemStack existing = getItem(i);
        if (canStack(existing, item)) {
            int space = existing.getMaxStackSize() - existing.getCount();
            int toAdd = Math.min(space, item.getCount());
            existing.grow(toAdd);
            item.shrink(toAdd);
        }
    }
    
    // Second pass: place in empty slots
    for (int i = start; i < end && !item.isEmpty(); i++) {
        if (getItem(i).isEmpty() && canPlaceItem(i, item)) {
            setItem(i, item.copy());
            return ItemStack.EMPTY;
        }
    }
    
    return item;
}
```

## Drag Operations

Handle item dragging across slots:

```java
@Override
protected void onDrag(Player player, DragAction drag) {
    List<Integer> slots = drag.getSlots();
    ItemStack item = drag.getItem();
    DragMode mode = drag.getMode();
    
    // Calculate distribution
    List<Integer> validSlots = slots.stream()
        .filter(s -> canPlaceItem(s, item))
        .filter(s -> getItem(s).isEmpty() || canStack(getItem(s), item))
        .toList();
    
    if (validSlots.isEmpty()) return;
    
    int totalItems = item.getCount();
    
    switch (mode) {
        case EVEN -> {
            // Distribute evenly
            int perSlot = totalItems / validSlots.size();
            for (int slot : validSlots) {
                addToSlot(slot, item, perSlot);
            }
        }
        case SINGLE -> {
            // One per slot
            for (int slot : validSlots) {
                if (totalItems > 0) {
                    addToSlot(slot, item, 1);
                    totalItems--;
                }
            }
        }
    }
    
    sendFullUpdate(player);
}
```

## Slot Groups

Organize slots into logical groups:

```java
public class OrganizedWindow extends Window {
    
    private final SlotGroup inputGroup;
    private final SlotGroup outputGroup;
    private final SlotGroup storageGroup;
    
    public OrganizedWindow(Player player) {
        super(WindowType.Container, 54);
        
        inputGroup = new SlotGroup(0, 8);       // Top row
        outputGroup = new SlotGroup(9, 17);     // Second row
        storageGroup = new SlotGroup(18, 53);   // Rest
    }
    
    public void clearInputs() {
        inputGroup.clear(this);
    }
    
    public List<ItemStack> getInputItems() {
        return inputGroup.getItems(this);
    }
    
    public boolean addToStorage(ItemStack item) {
        return storageGroup.tryAdd(this, item);
    }
}

public class SlotGroup {
    
    private final int start;
    private final int end;
    
    public SlotGroup(int start, int end) {
        this.start = start;
        this.end = end;
    }
    
    public void clear(Window window) {
        for (int i = start; i <= end; i++) {
            window.setItem(i, ItemStack.EMPTY);
        }
    }
    
    public List<ItemStack> getItems(Window window) {
        List<ItemStack> items = new ArrayList<>();
        for (int i = start; i <= end; i++) {
            ItemStack item = window.getItem(i);
            if (!item.isEmpty()) {
                items.add(item);
            }
        }
        return items;
    }
    
    public boolean tryAdd(Window window, ItemStack item) {
        for (int i = start; i <= end; i++) {
            if (window.getItem(i).isEmpty()) {
                window.setItem(i, item);
                return true;
            }
        }
        return false;
    }
}
```

## Slot Update Optimization

Only send updates for changed slots:

```java
public class OptimizedWindow extends Window {
    
    private final Map<Integer, ItemStack> previousState = new HashMap<>();
    
    public void sendChangedSlots(Player player) {
        for (int i = 0; i < getSize(); i++) {
            ItemStack current = getItem(i);
            ItemStack previous = previousState.get(i);
            
            if (!ItemStack.matches(current, previous)) {
                sendSlotUpdate(player, i);
                previousState.put(i, current.copy());
            }
        }
    }
    
    @Override
    protected void onOpen(Player player) {
        // Initialize state tracking
        for (int i = 0; i < getSize(); i++) {
            previousState.put(i, getItem(i).copy());
        }
    }
}
```

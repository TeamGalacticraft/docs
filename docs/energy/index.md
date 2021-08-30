# Galacticraft Energy
An energy API that utilizes [LibBlockAttributes](https://github.com/AlexIIL/LibBlockAttributes).
It is highly recommended to read through [LibBlockAttributes' Readme](https://github.com/AlexIIL/LibBlockAttributes/blob/0.9.x-1.17.x/README.md) for basic attribute information.

## Installation
![Maven metadata URL](https://img.shields.io/maven-metadata/v?logo=Apache%20Maven&metadataUrl=https%3A%2F%2Fmaven.galacticraft.dev%2Fdev%2Fgalacticraft%2FGalacticraftEnergy%2Fmaven-metadata.xml&style=flat-square&logoColor=white)

=== "Groovy (`build.gradle`)"
    ```groovy
    repositories {
        maven {
            url "https://maven.galacticraft.dev/"
            content {
                includeGroup("dev.galacticraft")
            }
        }
    }

    dependencies {
        include modApi "dev.galacticraft:GalacticraftEnergy:${version}"
    }
    ```

=== "Kotlin DSL (`build.gradle.kts`)"
    ```kotlin
    repositories {
        maven("https://maven.galacticraft.dev/") {
            content {
                includeGroup("dev.galacticraft")
            }
        }
    }

    dependencies {
        include(modApi("dev.galacticraft:GalacticraftEnergy:${version}") {})
    }
    ```



## Energy Types

Galacticraft-Energy is written to allow for multiple different energy types, if you wish to have energy measured in different units. To achieve compatibility between multiple energy types, you are required to implement a conversion mechanism between your new unit and the defaut `EnergyType` (see `DefaultEnergyType#INSTANCE`).

## Types of Energy Attributes

There are five different types of energy attributes:

 * `EnergyInsertable`: handles insertion of fluids
 * `EnergyExtractable`: handles extraction of fluids
 * `EnergyTransferrable`: handles insertion and extraction of fluids, extension of `EnergyInsertable` and `EnergyExtractable`
 * `Capacitor`: energy storage, extension of `CapacitorView` and can be converted into `EnergyInsertable`, `EnergyExtractable` and `EnergyTransferrable`
 * `CapacitorView`: read-only view of a `Capacitor`

 You might be wondering, why can't we just have `Capacitor` and call it a day? Well, the other attribute types exist so that you only expose as much of your energy storage as you want to; you probably woundn't want machines meant for processing have energy extracted for them, or generators have energy inserted into them, so, you should only expose what you need.

### EnergyInsertable

Allows for the insertion of energy of any `EnergyType`. If `Simulation` is `SIMULATE`, the backing storage will not be changed. Returns the amount of energy that failed to be inserted into it.
Also provides a convienience method `getPureExtractable` that may be used to guarentee that you won't expose other peer attributes.
```java
    int attemptInsertion(EnergyType type, int amount, Simulation simulation);
    
    EnergyExtractable getPureExtractable();
```

### EnergyExtractable

Allows for the extraction of energy of any `EnergyType`. If `Simulation` is `SIMULATE`, the backing storage will not be changed. Returns the amount of energy that was extracted from it.
Also provides a convienience method `getPureInsertable` that may be used to guarentee that you won't expose other peer attributes.
```java
    int attemptExtraction(EnergyType type, int amount, Simulation simulation);
        
    EnergyInsertable getPureInsertable();
```

### EnergyTransferrable

Extends `EnergyInsertable` and `EnergyInsertable`. Provides all parent methods. Useful for when you want peers to insert/extract energy from your storage, but not directly set it.

### CapacitorView

Provides the amount of energy stored in its own arbitrary `EnergyType`.
```java
    EnergyType getEnergyType();

    int getEnergy();
```

### Capacitor

Extends `EnergyTransferrable` and `CapacitorView`. Allows for the direct setting of energy values. You should prefer to expose a `EnergyTransferrable` rather than a `Capacitor` to other blocks/items. Provides helper methods for insertion and extration of energy too. This should used as your energy storage instance in your attribute providers.

```java
    void setEnergy(int amount);
    
    int insert(EnergyType type, int amount, Simulation simulation);

    int extract(EnergyType type, int amount, Simulation simulation);
```

### SimpleCapacitor

GalacticraftEnergy provides a default simple capacitor implementation which should cover most needs. It allows for arbitrary energy types and (de)serialization. See `dev.galacticraft.energy.impl.SimpleCapacitor` for more info.

## Adding an Energy Attribute Instance to a Block

A `Block` can provide a simple, immutable energy attribute instance by implementing `AttributeProviderBlock` and providing it in `addAllAttributes`

```java
public class EnergyVoidBlock extends Block implements AttributeProviderBlock {
    // -- snip --

    @Override
    public void addAllAttributes(World world, BlockPos pos, BlockState state, AttributeList<?> to) {
        to.offer(EmptyCapacitor.INSTANCE);
    }
}
```

In this example, `EmptyCapacitor.INSTANCE` can be replaced by any other attribute, however since this is a block, it is more complex to save/load the state of an attribute, so it is recommended to use a `BlockEntity` for that.

## Adding an Energy Attribute Instance to a BlockEntity

A `BlockEntity` can provide a more complex, mutable energy attribute instance by implementing `AttributeProviderBlockEntity` and providing it in `addAllAttributes`.

```java
public class EnergyStorageBlockEntity extends BlockEntity implements AttributeProviderBlockEntity {
    private final Capacitor capacitor = new SimpleCapacitor(DefaultEnergyType.INSTANCE, 5000);

    // -- snip --

    @Override
    public NbtCompound writeNbt(NbtCompound nbt) {
        super.writeNbt(nbt);
        this.capacitor.toTag(nbt);
        return tag;
    }
    
    @Override
    public void readNbt(NbtCompound nbt) {
        super.readNbt(nbt);
        this.capacitor.fromTag(nbt);
    }

    @Override
    public void addAllAttributes(AttributeList<?> to) {
        Direction direction = to.getSearchDirection();

        if (direction == Direction.UP) {
            to.offer(this.capacitor.getPureInsertable());
        } else if (direction == Direction.DOWN) {
            to.offer(this.capacitor.getPureExtractable());
        }
    }
}
```

This `BlockEntity` allows for energy to be inputted from the top of the block, and outputted on the bottom. To allow for the contingency of energy levels beyond a single operation, the attribute is stored as a field in the `BlockEntity`. Since attributes are not automatically saved to disk and loaded from disk, so you must use `readNbt` and `writeNbt` respectively.

## Obtaining an Energy Attribute Instance

All attributes are stored in `dev.galacticraft.energy.GalacticraftEnergy`. They are all `DefaultedAttributes` so calling `getFirst` is guarenteed to be not `null`. To verify whether an attribute instance is actually attached, prefer `getFirstOrNull` and use a `null` check.

### Blocks
Example for obtaining an `EnergyExtractable` instance from a `Block` or `BlockEntity`:

```java
public static EnergyExtractable exampleExtractable(World world, BlockPos pos, Direction direction) {
    return GalacticraftEnergy.EXTRACTABLE.getFirst(world, pos, SearchOptions.inDirection(direction));
}
```
The `SearchOption` parameter may be omitted or `null`, however it is recommended to be as specific as you possible can with your searches.

### Items

Example for obtaining an `Capacitor` instance from a `ItemStack`:
A `Reference` is LBA's wrapper/container class. Refrences are useful as it allows for the attribute to mutate itself while keeping the no-`ItemStack`-mutation rule of some inventories.
*Note: If you use LBA's `FixedItemInv` (or other related ones), you can probably pass an inventory slot (as it is a `Reference`).*

```java
public static Capacitor exampleCapacitorItemRef(Reference<ItemStack> ref) {
   return GalacticraftEnergy.CAPACITOR.getFirst(ref);
}
```

In the senario when you don't need to worry about syncing/inventories/mutation you can just pass an `ItemStack`. However, is highly recommended to utilize a `Reference` rather than just passing the stack.
```java
public static boolean doesItemStackHaveCapacitor(ItemStack stack) {
   return GalacticraftEnergy.CAPACITOR.getFirstOrNull(stack) != null;
}
```

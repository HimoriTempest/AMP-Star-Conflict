# Item list

Every item the server knows, with its ID for the [account commands](CONSOLE.md#account-commands): `give <uid> item|ship <ID> [amount]`, and the title IDs for `give <uid> title <ID>`. In the console, `find <text>` searches the same data.

| List | Items | Rows |
|---|---:|---:|
| [Ships](items/ships.md) | 659 | 659 |
| [Weapons](items/weapons.md) | 1,885 | 572 |
| [Modules](items/modules.md) | 7,853 | 2,491 |
| [Missiles and ammunition](items/ammo.md) | 1,015 | 243 |
| [Consumables](items/consumables.md) | 15 | 15 |
| [Bundles](items/bundles.md) | 806 | 806 |
| [Resources and ship parts](items/resources.md) | 328 | 328 |
| [Blueprints](items/blueprints.md) | 1,018 | 1,018 |
| [Portraits](items/portraits.md) | 349 | 349 |
| [Taunts](items/taunts.md) | 349 | 349 |
| [Loot](items/loot.md) | 155 | 155 |
| [Titles](items/titles.md) | 343 | 343 |

Weapons, modules and ammunition have one row per item with a column for each grade (Mk1, Mk2, Mk3, Rare, Epic, Relic, Premium), so they need fewer rows than they have items.

- **Name** is the English name from the server's shop data. Many modules, missiles, portraits and taunts only have an internal name there (the game's own texts are packed inside the client); those names are written out from the internal name and shown in *italics*.
- **Description** is built from the server's data: faction, ship class, ranks, prices, bundle contents and so on. The game's own description texts aren't available on the server.
- **"Not sold in the shop"** describes the unmodified shop. The template's settings make some of these buyable (DLC packs, former destroyer pack items); see the [README](README.md).
- Items with "not sold" and no price, or with test names, are often leftovers from development or events. Giving them works, but they may not show up or work properly in the client.

Generated from the game's server files (`server/cloud/data/store.xml`, `titles.xml`) and the client's definition files: 14,432 items, 343 titles.

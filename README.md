# gravvy_items

DB-backed dynamic items with **QB**, **OX**, and **Qbox** support, image sync, consumables, and a small open **bridge**.

- ✅ Add/update items in DB
- ✅ **QB mode**: live add/update (no restart) + auto-wired consumables
- ✅ **OX / Qbox**: generate `ox_inventory` items file + copy icons, then restart `ox_inventory`
- ✅ **Mixed stacks (qb-core + ox_inventory)**: keep `QBCore.Shared.Items` and QB useables updated for legacy scripts, while ox_inventory drives the UI/consumption
- ✅ Copy-if-missing icons from `gravvy_items/images/` to the correct inventory path
- ✅ Escrowed core with open `config.lua`, `bridge/bridge.lua`, images, and docs

## Requirements

- `oxmysql`
- One of:
  - **QB stack**: `qb-core` + `qb-inventory`
  - **OX stack**: `ox_inventory`
  - **Qbox stack**: `qbx-core` (uses `ox_inventory`)
- (Mixed stack supported) `qb-core` + `ox_inventory`

## Installation

Example order (adjust to your stack):

```
ensure qb-core          # for QB or mixed stacks
ensure qb-inventory     # for QB stacks only
ensure ox_inventory     # for OX / Qbox / mixed stacks
ensure gravvy_items
```

Run SQL in your DB (see `schema.sql` too).

## Configuration

Edit `config.lua` (open file). Key options:

```lua
Config.InventorySystem = 'auto'          -- 'auto' | 'qb' | 'ox' | 'qbox'
Config.UpdateQBCoreSharedItems = true    -- keep QB Shared.Items & useables updated even in mixed stacks
Config.QBInventoryResource  = 'qb-inventory'
Config.QBInventoryImagesDir = 'html/images'
Config.OXInventoryResource  = 'ox_inventory'
Config.OXInventoryImagesDir = 'web/images'
Config.OXGeneratedItemsFile = 'ox_items.generated.lua'
Config.QboxCoreResource     = 'qbx-core'
Config.LocalImagesDir       = 'images'
Config.DeleteAfterCopy      = false
Config.AdminPermission      = 'god'          -- QB command perms
Config.AdminAce             = 'command.gravvy' -- ACE when QB Commands not present
```

ACE example (qbox/standalone perms) in `server.cfg`:
```
add_ace group.admin command.gravvy allow
# add_principal identifier.steam:XXXX group.admin
```

## Images

Place icons in: `gravvy_items/images/`

On start (or `/gi-imgsync`), they copy **if missing** to:
- **QB**: `qb-inventory/html/images/`
- **OX / Qbox**: `ox_inventory/web/images/`

We **never overwrite** existing files in the inventory.

Use `image = "wrench.png"` in your item. (No URLs.)

## Workflows

### QB (qb-core + qb-inventory)
Upsert into DB, then ensure live:
```lua
exports['gravvy_items']:SaveItemToDB({
  name = 'wrench',
  label = 'Wrench',
  weight = 200,
  type = 'item',
  image = 'wrench.png',
  unique = false,
  useable = true,
  shouldClose = true,
  description = 'Hand tool',
})
exports['gravvy_items']:EnsureItemFromDB('wrench', { overwrite = true })
```

### OX (ox_inventory) or Qbox (qbx-core + ox_inventory)
Upsert into DB, then run:
```
/gi-ox-export
```
This writes `gravvy_items/ox_items.generated.lua`. Merge it into `ox_inventory/data/items.lua` (or `dofile` it), then **restart `ox_inventory`**.

### Mixed stack (qb-core + ox_inventory)
Set `Config.UpdateQBCoreSharedItems = true`. We will:
- Populate `QBCore.Shared.Items` and auto-wire QB useables (for legacy QB scripts),
- Copy icons into **ox_inventory**,
- Let **ox_inventory** drive item consumption/UI via the generated file.

## Consumables (QB, OX, Qbox)

Put consumable info into `meta` when you upsert.

```lua
exports['gravvy_items']:SaveItemToDB({
  name = 'cola',
  label = 'Cola',
  weight = 200,
  type = 'item',
  image = 'cola.png',
  unique = false,
  useable = true,
  shouldClose = true,
  description = 'Fizzy drink',
  meta = {
    -- QB framework (CreateUseableItem)
    consumable = { amount = 1 },  -- remove 1 on use in QB
    qb = {
      clientEvent = 'my_resource:drinkCola',  -- or serverExport = 'my_res:applyDrink'
      clientArgs  = { thirst = 25 }
    },

    -- OX/Qbox (ox_inventory item fields)
    ox = {
      consume = 1,                 -- ox removes 1 on use
      close   = true,
      client  = { event = 'my_resource:drinkCola', args = { thirst = 25 } },
      -- server = { export = 'my_resource:applyDrink' },
    }
  }
})
```

## Commands

- `/gi-imgsync` – copy missing icons into the inventory images folder (OX preferred in mixed stacks).
- `/gi-add name "Label" weight type image unique useable shouldClose "desc"` – upsert + (QB) refresh.
- `/gi-disable name` – soft-disable + (QB) refresh.
- `/gi-harddel name` – hard delete (refuses if any player holds it) + (QB) refresh.
- `/gi-refresh` – rebuild QB `Shared.Items` from DB (QB/mixed).
- `/gi-sync` – ensure enabled DB items present (QB, no rebuild).
- `/gi-ox-export` – generate `ox_items.generated.lua` for OX/Qbox.
- `/gi-useables` – (re)register QB consumables from DB.

## Bridge (custom backend adapter)

We ship an **open** `bridge/bridge.lua` so owners can adapt detection, icon paths, and item mapping without touching escrow.

- `Bridge.detect()` → `'qb' | 'ox' | 'qbox'`
- `Bridge.targetInventory()` → `(resourceName, imagesDir)`
- `Bridge.toQB(def)` → QB item mapping
- `Bridge.toOX(def)` → OX item mapping

If you run a fork, edit `bridge/bridge.lua`. The core automatically uses it, or falls back to built-ins if absent.

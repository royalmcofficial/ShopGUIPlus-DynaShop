# DynaShop: configuration price reconciliation (RoyalMC)

## Problem

Changing `sellDynamic.min/max` or `buyDynamic.min/max` in ShopGUIPlus
does not update the existing FlatFile `data/prices.json` entry. The old
bounds are copied by `FlatFileStorageManager.savePrice()`, which can
silently undo market adjustments on the next transaction or price update.

## Behavior in this PR

- For **existing** dynamic items, synchronize only explicitly configured
  `buyDynamic` / `sellDynamic` `min`, `max`, `growth`, `decay`.
- Preserve current buy/sell market price and its stock. The `DynamicPrice`
  constructor clamps only if an existing price is outside the **new**
  allowed range.
- Do not reset prices to YAML `buyPrice` / `sellPrice`.
- Preserve Vihara's missing-type repair and stored `linked` flag.
- On startup/reload only changed records are written; invalidate only the
  affected item's runtime cache.
- Existing MySQL storage persists current values rather than price metadata;
  its interface implementation remains a no-op.

## Operations / important limitation

Setting YAML `sellPrice` (base price) alone **does not reset** a stored
dynamic price. For an immediate, targeted nerf, change that item's current
price separately (e.g. the safe, offline `prices.json` patch procedure)
or set a lower maximum bound and allow the reconciliation to clamp it.

Do **not** edit `prices.json` while Minecraft is running; DynaShop can
flush its in-memory cache and overwrite changes. Take a current backup first.

## Staging test plan

1. Snapshot current `data/prices.json` and start from a restored production
   clone; do not run this unverified in production.
2. Change only `drops:20` configuration from min=2/max=8 to
   min=0.05/max=4. With persisted sellPrice=2, restart (or reload both
   ShopGUI+ and DynaShop). Verify min/max update and sellPrice stays 2.
3. Sell one Nether Wart stack. Confirm price moves below 2 according to
   the configured decay, persists to `prices.json` and appears in GUI.
4. Set new max below the current price; confirm the current price clamps
   down to that max. Increase max again; confirm it **does not reset**
   upwards to the YAML base price.
5. Verify all unrelated entries, including the heavily decayed market prices,
   preserve their current buy/sell values.
6. Validate `linked: false`, buy-disabled (`buyPrice: -1`), RECIPE/LINK,
   STOCK/STATIC_STOCK cases, and a second restart after the first update.
7. Check no concurrent `prices.json` write or malformed JSON occurred.

## Verification status

Code review + simulated reconciliation of the supplied 78-item snapshot:
5 entries have explicit bounds differing from YAML; all 78 current
`sellPrice` values remain identical in that snapshot after reconciliation.
This is a **data-level simulation**, not a Java build or staging test.

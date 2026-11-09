# Vehicles CE Overhaul — modding guide

How to tune your vehicle mod for Vehicles CE Overhaul: armor, turrets and ammo, active protection, raids.
Short version: **most vehicle mods need nothing.** The mod adapts every Vehicle Framework vehicle by itself; you only
write data when you want your own numbers.

## 0. What happens automatically

| What | Without any data from you |
|---|---|
| Armor | Every `VehicleDef` without a table gets an armor class and its plate armor, hull armor, part health and mass (see 1.4). |
| Turret ammo | A turret without CE data gets a CE ammo set by weapon kind (rifle, MG, autocannon, cannon, rocket, mortar). |
| Active protection | Land APCs, medium and battle tanks and super-heavies get an APS upgrade node (player vehicles only). |
| Raids | Ground vehicles with crew and turrets can be brought by raiders (players choose who in the settings). |

**Check the result:** Dev mode → Debug actions → *Vehicles CE* → *Armor report* and *Ammo report* (written to the log).
The setting *Log a report for every hit* prints how each round went through a vehicle.

## 1. Armor

### 1.1 Units
The stock CE scale. **Sharp** armor is in mm, on the same scale as CE ammo penetration:
7.62×51 AP 14 · .50 AP 28 · 14.5 AP 36 · 20 mm AP 30 · 30 mm AP 70 · 90 mm AP 170 · RPG-7 HEAT 500.
**Blunt** armor of plates is about 14 × sharp (thin plates ~10 ×), roofs about 3.5 ×.

### 1.2 Which parts are plates
A component is an armor plate when (in this order):
1. our markup says so (`plates` / `notPlates`, see 1.3 and 1.5);
2. it has its own `<armor>` and no `<categories>`;
3. its key or label says armor (`Armor`, `Armour`, `Plating`, `Plate`, `Hull`, `Roof`) and it is not running gear, engine,
   fuel tank, turret or other system.

The plate's side comes from its hitbox side (`Front`, `Back`, `Left`, `Right`). A plate with an `Empty`/`Body` side is a
**roof** (hit by explosions above the vehicle). A round that gets through a plate goes on through the internal components
in its path and can hit crew whose role has a hitbox cell there, so give roles hitbox cells.

### 1.3 Your own armor table
A `VehicleArmorDef` is applied by the core at startup, after all XML patches, so load order cannot undo it. The vehicle is
referenced by defName: a table for a vehicle that is not loaded is ignored.

```xml
<Defs>
  <VehiclesCEOverhaul.VehicleArmorDef>
    <defName>MyMod_Armor_MyTank</defName>
    <vehicle>MyMod_MyTank</vehicle>
    <hullSharp>25</hullSharp>      <!-- armor where no plate covers the hull -->
    <hullBlunt>120</hullBlunt>
    <!-- <mass>36000</mass> -->     <!-- kg; optional -->
    <components>
      <li><key>FrontArmor</key><sharp>200</sharp><blunt>2800</blunt><health>750</health></li>
      <li><key>LeftArmor</key><sharp>80</sharp><blunt>1120</blunt><health>700</health></li>
      <li><key>RightArmor</key><sharp>80</sharp><blunt>1120</blunt><health>700</health></li>
      <li><key>RearArmor</key><sharp>40</sharp><blunt>560</blunt><health>600</health></li>
      <li><key>Roof</key><sharp>30</sharp><blunt>105</blunt><health>500</health></li>
    </components>
    <!-- optional: fix what the auto rules get wrong -->
    <plates><li>TurretArmor</li></plates>
    <notPlates><li>ArmoredTreads</li></notPlates>
  </VehiclesCEOverhaul.VehicleArmorDef>
</Defs>
```
Any value below 0 (or left out) means "keep what is there".

### 1.4 The class ladder (automatic armor)
The class comes from the name (*tank*, *APC*, *armored car*, *super-heavy*…), plates, tracks, guns and size.

| Class | Front | Side | Rear | Roof | Front stops |
|---|---|---|---|---|---|
| Civilian | 2 | 2 | 2 | 2 | fragments |
| Armored car | 28 | 20 | 14 | 8 | .50 AP |
| APC / IFV | 45 | 28 | 20 | 14 | 14.5 AP |
| Light tank | 60 | 36 | 28 | 20 | .50 Sabot, 20 mm AP |
| Medium tank | 200 | 80 | 40 | 30 | everything but anti-tank weapons |
| Battle tank | 700 | 250 | 60 | 50 | even an RPG-7 |
| Super-heavy | 1200 | 450 | 150 | 100 | — |
| Aircraft / armed | 7 / 13 | | | | rifle FMJ / 7.62 AP |
| Wooden ship | 12 | 12 | 10 | 6 | a musket ball |

Use these as a guide for hand-made tables, so your vehicles sit on the same ladder as everyone else's.

### 1.5 Plate markup without a table
```xml
<Operation Class="PatchOperationAddModExtension">
  <xpath>Defs/Vehicles.VehicleDef[defName="MyMod_MyTank"]</xpath>
  <value>
    <li Class="VehiclesCEOverhaul.VehicleArmorLayout">
      <plates><li>TurretArmor</li></plates>
      <notPlates><li>ArmoredTreads</li></notPlates>
    </li>
  </value>
</Operation>
```

## 2. Turrets and ammo

- **CE data** — the usual CE way, `CETurretDataDefModExtension` on the `VehicleTurretDef` (`ammoSet`, `speed`, `sway`,
  `spread`, `shotHeight`, `recoil`). If your turret has it, the core uses it as is.
- **No CE data** — the core picks a set by weapon kind and writes the extension itself (*Ammo report* shows the choice).
- **Sights and recoil** for the CE-style aiming (optional, on the `VehicleTurretDef`):
```xml
<li Class="VehiclesCEOverhaul.TurretAimExtension">
  <sightsEfficiency>1.5</sightsEfficiency>   <!-- like the weapon stat -->
  <recoilAmount>1.2</recoilAmount>           <!-- drift during a burst -->
  <circularError>1</circularError>           <!-- mortars -->
  <indirectFirePenalty>0.15</indirectFirePenalty>
</li>
```

## 3. Active protection (APS)

Automatic for land APCs, medium and battle tanks and super-heavies (high-tech vehicles get stronger numbers). To set your
own, give the `VehicleDef` this comp; the automation then leaves the vehicle alone. `<maxCharges>0</maxCharges>` = no APS.
```xml
<li Class="VehiclesCEOverhaul.CompProperties_ActiveProtection">
  <maxCharges>4</maxCharges>
  <cooldownTicks>90</cooldownTicks>   <!-- pause after a salvo -->
  <salvo>1</salvo>                    <!-- intercepts before the pause -->
  <range>5</range>                    <!-- cells from the hull -->
  <minRange>1</minRange>
  <chance>0.85</chance>
</li>
```
APS is an upgrade: the node is added to the vehicle's upgrade tree automatically (a single-node tree if it has none).
It catches rockets and missiles only; shells and bullets are the armor's job.

## 4. Raids

Raiders bring ground vehicles with crew roles and turrets. Artillery (turret range over 120 cells) is never taken.
Players have the last word in *Mod settings → Vehicles and factions in raids*; your XML sets the defaults.

On the `VehicleDef`:
```xml
<li Class="VehiclesCEOverhaul.VehicleRaidExtension">
  <mode>Factions</mode>              <!-- Auto, TechLevel, Factions or Off -->
  <factions><li>MyMod_Faction</li></factions>
  <!-- <techLevel>Spacer</techLevel>    for mode TechLevel: factions of this tech level and up -->
  <costFactor>1.5</costFactor>       <!-- price in raid points x -->
</li>
<li Class="VehiclesCEOverhaul.VehicleRaidRoleExtension">
  <role>Breacher</role>              <!-- Auto, Breacher (breaks walls in a siege), Support, None -->
</li>
```
On a `VehicleTurretDef` (siege): `<li Class="VehiclesCEOverhaul.VehicleRaidTurretExtension"><breaches>false</breaches></li>`
(or `<breachAmmo>` — the ammo it shells walls with).

**Factions "by tech level":** add yours to the shared list.
```xml
<Operation Class="PatchOperationAdd">
  <xpath>Defs/VehiclesCEOverhaul.VehicleRaidTuningDef[defName="VCEO_VehicleRaidTuning"]/factions</xpath>
  <value><li>MyMod_Faction</li></value>
</Operation>
```
Every raid number (vehicle share of raid points, price per point, AI distances, capture time…) lives in
`Defs/VehicleRaidTuningDefs/VCEO_VehicleRaidTuning.xml`, commented; patch it like any def.

## 5. Cargo
Armored vehicles get a cargo capacity floor by class, enough for a few reloads of real-weight shells:
`Defs/VehicleCargoTuningDefs/VCEO_CargoFloor.xml`.

## 6. Loading your files only when the core is there
Our def types and extensions exist only when Vehicles CE Overhaul is loaded. Put your data in a folder that loads only
then (`LoadFolders.xml` in your mod's root folder):
```xml
<loadFolders>
  <v1.6>
    <li>/</li>
    <li>1.6</li>
    <li IfModActive="blakil.vehiclesceoverhaul">Mods/VCEO</li>
  </v1.6>
</loadFolders>
```
Then `Mods/VCEO/Defs/...` (tables) and `Mods/VCEO/Patches/...` (extensions) load with the core and are skipped without it.
No dependency on the core is needed.

## 7. Questions
Ask in the mod's Workshop discussions. If something adapts wrong, post the vehicle defName and the *Armor report* /
*Ammo report* lines for it.

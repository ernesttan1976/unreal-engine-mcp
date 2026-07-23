# Shoot The Monster (UE 5.6 / FirstPersonProject)

Target level:

- `D:\UEProjects\FirstPersonProject\FirstPersonProject\Content\Monster.umap`

Existing project assets found on disk:

- First person template: `/Game/FirstPerson/Blueprints/BP_FirstPersonCharacter`, `BP_FirstPersonGameMode`, `BP_FirstPersonPlayerController`
- Monster: `/Game/BP_Monster`
- Soldier mesh: `/Game/Characters/Soldier/Ch15_nonPBR` (plus skeleton/anim assets)
- Chaos assets (likely for breakables): `/Game/Chaos/GC_BP_Monster` and `/Game/Chaos/GC_BP_Monster2`
- Enhanced Input assets: `/Game/Input/IMC_Default`, `IA_Move`, `IA_Look`, `IA_Jump` (no Fire action yet)

Rifle mesh:

- `ar_15_style_rifle` was added (verify the asset reference path in Content Browser: right-click it, `Copy Reference`).
- If it is not actually present in the project yet (not saved/imported), you can still proceed using a placeholder mesh and swap it later.

## Goal

1. Add a first person character to `Monster.umap`
2. Apply the soldier mesh (`Ch15_nonPBR`) to the character
3. Add a rifle to the character
4. Let the character shoot and hit the monster. 1000 rounds.
5. Monster has 100 health points. Each 1 health lost breaks off exactly 1 item. Remaining items stay intact.

## 1) Add a first person character to `Monster.umap`

1. Open the project in Unreal Engine 5.6.
2. Open the level `Monster` (`Content/Monster.umap`).
3. In **World Settings**:
   - Set **GameMode Override** to `/Game/FirstPerson/Blueprints/BP_FirstPersonGameMode`.
4. Open `/Game/FirstPerson/Blueprints/BP_FirstPersonGameMode` and confirm:
   - **Default Pawn Class** = `/Game/FirstPerson/Blueprints/BP_FirstPersonCharacter`.
   - (Optional) **Player Controller Class** = `/Game/FirstPerson/Blueprints/BP_FirstPersonPlayerController`.
5. In `Monster.umap`, place a **PlayerStart** where the player should spawn.

Alternative (no GameMode changes):

- Drag `BP_FirstPersonCharacter` into the level and set **Auto Possess Player = Player 0** on that placed instance.

## 2) Apply the soldier mesh to the first person character

Asset:

- `/Game/Characters/Soldier/Ch15_nonPBR`

Recommended setup (keeps first-person camera/control clean even if soldier animations are not perfect):

1. Open `BP_FirstPersonCharacter`.
2. Add a new **SkeletalMeshComponent** named `SoldierMesh`.
3. Attach it to the character root (usually the Capsule).
4. Set `SoldierMesh`:
   - **Skeletal Mesh** = `Ch15_nonPBR`.
   - **Animation Mode**:
     - If you have/need animations: **Use Animation Blueprint** (create/retarget as needed).
     - If you just need it visible: use a simple idle / default.
5. If the body blocks the first-person camera:
   - Set `SoldierMesh` to **Owner No See** and keep any existing first-person arms for the owning player view.

## 3) Add a rifle to the character

Use the rifle mesh you added (`ar_15_style_rifle`). If you need to import it again later, place it under something like `Content/Weapons/Rifle/`.

Attach it:

1. In `BP_FirstPersonCharacter`, add a **StaticMeshComponent** named `Rifle`.
2. Attach `Rifle` to `SoldierMesh`.
3. Set `Rifle` **Static Mesh** = `ar_15_style_rifle` (or the correct `SM_*` mesh asset inside that pack).
3. Create a hand socket on the soldier skeleton:
   - Open `Ch15_nonPBR_Skeleton`.
   - Find the right-hand bone (often `hand_r` / `RightHand`).
   - Add a socket named `Socket_Rifle`.
4. Back in `BP_FirstPersonCharacter`:
   - Set `Rifle` **Parent Socket** = `Socket_Rifle`.
   - Adjust relative transform until it sits correctly in the hand.

## 4) Shooting: hit the monster, 1000 rounds

### A) Add an Enhanced Input action for firing

1. Create `IA_Fire` (Input Action).
   - Value Type: **Digital (bool)**.
2. Open `IMC_Default`.
   - Add mapping: `IA_Fire` -> **Left Mouse Button**.
   - (Optional) Also map: `IA_Fire` -> **Gamepad Right Trigger**.

### B) Implement hitscan shooting in `BP_FirstPersonCharacter`

Add variables:

- `Ammo` (int) default `1000`
- `FireRange` (float) default e.g. `100000` (tune as desired)

Implement (simple + reliable):

1. Ensure `IMC_Default` is added to the player’s Enhanced Input subsystem on BeginPlay (template usually already does this).
2. Add Enhanced Input event for `IA_Fire`.
3. On `IA_Fire Triggered`:
   - If `Ammo <= 0`: return.
   - `Ammo = Ammo - 1`.
   - Get camera location + forward vector.
   - `LineTraceByChannel` from camera to `Camera + Forward * FireRange`.
   - If hit:
     - If hit actor is `BP_Monster`, call a monster event (example: `ApplyBulletHit`) with `Damage = 1`.

Optional: full-auto while holding:

- Use `IA_Fire Started` to start a timer (loop) and `IA_Fire Completed` to clear it.

## 5) Monster health (100) with deterministic break-off per HP (recommended)

This approach guarantees exactly 1 part breaks per 1 HP lost.

### A) Monster variables and parts list

In `BP_Monster` add:

- `Health` (int) default `100`
- `BreakParts` (Array of `PrimitiveComponent` or `StaticMeshComponent`)
- `NextBreakIndex` (int) default `0`

### B) Build the monster out of 100 parts

1. In `BP_Monster`, add (or generate) 100 visible mesh components that make up the monster.
2. Keep them attached and not simulating physics initially.
3. Populate `BreakParts` in a stable order:
   - Manual ordering is fine.
   - Or, on BeginPlay, gather components by a naming convention like `Part_001`..`Part_100` and sort.

### C) Apply damage and break exactly one part per point

1. Create a custom event/function on `BP_Monster`: `ApplyBulletHit(Damage, HitLocation, ShotDirection)`.
2. When called:
   - Clamp and reduce `Health`.
   - For each point of damage (usually 1):
     - `Part = BreakParts[NextBreakIndex]`.
     - `NextBreakIndex++`.
     - `Part -> DetachFromComponent(KeepWorld)`.
     - `Part -> SetSimulatePhysics(true)`.
     - `Part -> SetCollisionEnabled(QueryAndPhysics)`.
     - (Optional) `Part -> AddImpulse(ShotDirection * ImpulseStrength)`.

This ensures:

- 100 HP -> 100 parts can break, one per hit.
- The rest remains intact until its turn.

## Integration: connect shooting to monster breaking

In `BP_FirstPersonCharacter` after the line trace hit:

1. If `Hit Actor` is `BP_Monster`:
   - Call `ApplyBulletHit(1, HitLocation, ShotDirection)`.

## Validation checklist

1. Playing `Monster.umap` spawns you as the first person character.
2. Soldier mesh is applied (and first-person camera is not obstructed).
3. Rifle is attached to `Socket_Rifle` and follows the hand.
4. Shooting reduces ammo from 1000 to 0.
5. Each monster hit reduces health by 1 and breaks off exactly 1 part; remaining parts stay intact.

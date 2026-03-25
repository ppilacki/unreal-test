# Survive Together — Dev Log

## Project Overview
- **Engine:** Unreal Engine 5.7
- **Type:** Local co-op survival game, same screen (not split screen)
- **Template:** Third Person

## Input Setup
- Player 1: WASD + Controller 1
- Player 2: Arrow keys + Controller 2
- Using Enhanced Input system

## Progress

### 2026-03-24 — Controller Setup
- Confirmed WASD keyboard input works for Player 1
- PS5 DualSense controller does NOT work natively with UE 5.7 on Windows
  - Windows detects it fine (Game Controllers / joy.cpl shows it working)
  - Unreal does not receive input from DualSense (neither Bluetooth nor USB)
  - Workaround: use DS4Windows to emulate Xbox controller
- Xbox controllers work out of the box
- Enabled RawInput plugin (may want to disable if causing issues)
- Fixed Xbox controller disconnecting after sleep: disable "Allow the computer to turn off this device to save power" in Device Manager → USB/Xbox Peripherals → Power Management
- Current working setup: 2x Xbox controllers, Controller 1 → Player 1, Controller 2 → Player 2
- Device Mapping Policy: "Primary User Shares Keyboard and First Gamepad"

### 2026-03-24 — Co-op Camera Setup
- Created BP_CoopCamera actor with Spring Arm + Camera components
  - Spring Arm: Target Arm Length 1500, Pitch -50, Do Collision Test off, Inherit Pitch/Yaw/Roll off
  - Event Tick calculates midpoint between both players and sets actor location
- Key fix: **"Auto Manage Active Camera Target"** must be **unchecked** in BP_ThirdPersonPlayerController Class Defaults — this was overriding Set View Target every frame
- Set View Target with Blend is called from Level Blueprint after Create Local Player + Delay
  - Execution chain: Delay → Get All Actors of Class (BP_CoopCamera) → Set View Target for Player 0 and Player 1
- Removed CameraBoom + FollowCamera + Aim nodes from BP_ThirdPersonCharacter (character no longer has its own camera)
- Camera component on BP_CoopCamera needs Set Active (New Active = true) on BeginPlay

### 2026-03-24 — Leash System (Phase 1)
- Created BP_Leash actor with a Cable Component connecting both players visually
  - Cable Component settings: Cable Length 500, Num Sides 8, Cable Width 10, Solve Iterations 8, Enable Stiffness off
  - Actor positions itself at Player 1's location every tick; cable end stretches to Player 2's location
  - End Location is calculated in local space using Inverse Transform Location (world → local conversion)
- BeginPlay: gets Player 1 and Player 2 references via Get Player Character (index 0 and 1) with a 0.5s Delay before the calls, because Player 2 is spawned by Create Local Player in the Level Blueprint slightly after BP_Leash's BeginPlay fires
- Event Tick: guarded with Is Valid check on Player 2 — skips the frame silently if Player 2 isn't ready yet, preventing "Accessed None" crashes
- BP_Leash placed in level — no manual positioning needed as it overrides its own location on tick

### 2026-03-25 — Leash System (Phase 2) — Dynamic Length + Constraint
- Cable length now adjusts dynamically each tick: Cable Length = distance between players + CableSlack variable (default 50)
  - Gives the rope a natural sag when players are close, goes taut as they approach max distance
- Added MaxLeashLength variable (float) — configurable in editor per instance
- Added leash constraint in Event Tick exec chain: SET End Location → SET Cable Length → Branch
  - Branch condition: raw Vector Length (player distance) > MaxLeashLength
  - True branch: symmetric pull — both players are repositioned toward the midpoint
  - Midpoint = (P1 + P2) / 2; HalfStep = Normalize(P2 - P1) × (MaxLeashLength / 2)
  - P1 new position = Midpoint - HalfStep; P2 new position = Midpoint + HalfStep
  - False branch: unconnected — players are within range, nothing happens

### Current Known Issue — Asymmetric Constraint
The symmetric pull logic is correctly wired in BP_Leash, but in practice only Player 2 gets constrained. Player 1 moves freely past the limit because **SetActorLocation is overridden by the CharacterMovementComponent** — the movement component recalculates and reasserts the character's position every frame, overwriting the teleport on Player 1 while Player 2's teleport happens to stick.

The result: Player 1 drags Player 2 when the leash is taut. Functional but not symmetric.

**To fix this properly (when ready):**
Replace SetActorLocation with **Launch Character** on both players. When distance > MaxLeashLength, calculate a velocity vector pointing from each player toward the midpoint and call Launch Character with that velocity (with bXYOverride and bZOverride set appropriately). This works with the movement component rather than against it and produces smooth, physically believable resistance instead of a hard teleport. The magnitude of the launch velocity can be tuned to feel like elastic resistance or a hard stop.

### Next Steps
- [ ] Fix symmetric leash constraint using Launch Character (see known issue above)
- [ ] Finish zoom logic in BP_CoopCamera (dynamic Spring Arm length based on player distance)
- [ ] Add IsValid check for Player 2 in BP_CoopCamera tick to prevent errors
- [ ] Attach VFX and 3D models along the cable
- [ ] Add damage to enemies that intersect the leash

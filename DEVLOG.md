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

### Next Steps
- [ ] Finish zoom logic in BP_CoopCamera (dynamic Spring Arm length based on player distance)
- [ ] Add IsValid check for Player 2 in BP_CoopCamera tick to prevent errors
- [ ] Set up arrow keys input for Player 2

# RootMovement

Exposes Unreal's `FRootMotionSource_ConstantForce` root-motion system to Blueprints — a properly engine-integrated way to push a character around (thrust, dashes, knockback, launches) without fighting the `CharacterMovementComponent`, replication, or animation root motion.

Two Blueprint nodes:

- **Apply Root Motion Constant Force** — fire-and-forget, re-apply on a tick/timer for continuous forces (e.g. jetpack thrust).
- **Apply Root Motion Constant Force with Callbacks** — async, cancellable, single-shot, with `OnComplete` / `OnFail` delegates (e.g. a dash or launch you may need to interrupt).

No project-specific dependencies. Copy `RootMovement_5.5` into any UE5 project's `Plugins/` folder and it just works.

---

## Where this came from

Unreal's Gameplay Ability System (GAS) — the framework Epic's own **Lyra** sample project is built on for essentially all of its gameplay — ships a built-in ability task called `UAbilityTask_ApplyRootMotionConstantForce`:

```
Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/Tasks/AbilityTask_ApplyRootMotionConstantForce.h
Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Abilities/Tasks/AbilityTask_ApplyRootMotionConstantForce.cpp
```

Diffing that task's `SharedInitAndApply()` against this plugin's `ApplyRootMotionConstantForce()` / `UAsyncRootMovement::Activate()`, the two build the `FRootMotionSource_ConstantForce` **field-by-field, in the same order, with the same values and nearly identical variable names** (`InstanceName`, `AccumulateMode` from a `bIsAdditive` bool, `Priority = 5`, `Force = WorldDirection * Strength`, `Duration`, `StrengthOverTime`, `FinishVelocityParams.Mode/SetVelocity/ClampVelocity`, the same `IgnoreZAccumulate` gravity-flag gate). The public factory function signature is parameter-for-parameter identical too.

That's direct, verifiable evidence this plugin re-implements Epic's own GAS ability task as plain Blueprint-callable functions/async actions — **without the GAS dependency** (no `UAbilitySystemComponent`, no `UGameplayAbility`, no replication setup). In other words: it's GAS's battle-tested root-motion application technique, made available to any project, GAS or not.

(One honest caveat: Epic's docs confirm Lyra runs on GAS broadly and that this exact ability task is a standard GAS feature, but I did not independently verify that a specific Lyra Blueprint calls this task by name — that would require Lyra's own project source, not just the engine plugin it depends on.)

---

## Use cases

- **Continuous propulsion** — jetpack/thruster hold-to-move, wind zones, conveyor-style forced movement (repeating `ApplyRootMotionConstantForce`, re-issued every tick/timer interval).
- **One-shot bursts** — dashes, dodges, ledge-slide pops, grapple pulls, knockback from an explosion or melee hit (`AsyncRootMovement`, with `OnComplete`/`OnFail` to chain follow-up logic).
- **Scripted forced movement** — a cutscene or gameplay beat that needs to move the character along a fixed direction for a fixed duration regardless of input, with a clean, controllable exit velocity.
- **Any case where you need the force to compose correctly** with other movement — animation root motion, other simultaneous abilities — instead of clobbering `Velocity` directly.

## Why use this instead of `LaunchCharacter` / `AddForce` / a Timeline?

The obvious DIY approach is a Blueprint `Timeline` node driving `LaunchCharacter` or `AddForce`/`AddImpulse` every tick. It works, but it has real problems this plugin avoids:

| Problem with Timeline + LaunchCharacter/AddForce | How `FRootMotionSource_ConstantForce` solves it |
|---|---|
| `LaunchCharacter`/`AddForce` **directly stomp or add to `Velocity`** outside the movement component's own simulation step — if two systems touch velocity the same frame, it's last-write-wins, not composition. | Root motion sources are accumulated **inside** `UCharacterMovementComponent`'s own update, with an explicit `AccumulateMode` (`Additive` vs `Override`) and `Priority`, so multiple forces (including animation root motion) combine predictably instead of fighting. |
| A Timeline needs a running Blueprint tick, manual per-frame force math, and manual cleanup if the effect is interrupted (character dies, gets stunned, etc.) — miss the cleanup and the force silently keeps re-applying or leaves the timeline orphaned. | `Duration` and `StrengthOverTime` (a curve) are handled **natively** by the root motion source itself — no Tick, no Timeline, no per-frame Blueprint math. `AsyncRootMovement::Cancel()` explicitly tears the source down by ID on interruption. |
| Timeline-driven velocity edits aren't part of the movement component's prediction/replication path, so they're awkward and error-prone to get right in a networked game. | Root motion sources are exactly what GAS itself uses for network-predicted movement abilities — this plugin exposes that same, already-solved replication-friendly mechanism. |
| No standard way to control what velocity you're left with when the force ends — you have to manually snap/clamp velocity yourself in the Timeline's "Finished" event. | `FinishVelocityParams` (`VelocityOnFinishMode` + `SetVelocityOnFinish` / `ClampVelocityOnFinish`) gives you an explicit, built-in handoff back to normal movement. |
| Gravity interaction during the force is another manual Timeline branch. | `bEnableGravity` toggles `ERootMotionSourceSettingsFlags::IgnoreZAccumulate` for you. |

Short version: a Timeline + `LaunchCharacter` is a hand-rolled, per-project reimplementation of something Unreal already solved properly at the engine level for exactly this purpose — this plugin just makes that existing, correct mechanism reachable from Blueprint without requiring the rest of GAS.

## Extraction process

1. Ran a full dependency audit: every `#include` across all `.h`/`.cpp` files checked by hand, and every class name / asset path scanned for anything project-specific (Blueprint classes like a character or component Blueprint, hardcoded `/Game/...` asset paths, project-specific enums/structs). Result: **zero** project-specific coupling — every dependency is a stock Unreal `Core`/`CoreUObject`/`Engine`/`Kismet` type.
2. Verified compilability in isolation: since nothing depends on project-specific code, the plugin compiles unmodified in a brand-new, empty UE5 project.
3. Traced the actual origin of the logic against the engine's own `GameplayAbilities` plugin source (see "Where this came from" above).
4. Repackaged as a standalone, source-only distribution: `Source/`, `Resources/`, and `RootMovement.uplugin` only — `Binaries/` and `Intermediate/` (machine-specific compiled output) intentionally excluded, since they're regenerated automatically and shouldn't ship.

## Adapting this for a different Unreal Engine version

Checked directly against the engine source for UE **5.5, 5.6, 5.7, and 5.8** (all installed locally): `Engine/Source/Runtime/Engine/Classes/Engine/CancellableAsyncAction.h` is **byte-for-byte identical** across all four versions, and `FRootMotionSource_ConstantForce` / `GameFramework/RootMotionSource.h` has been stable, core `Engine`-module API since it was introduced back in UE4.20 — it isn't a GAS-only type. So:

- **UE 5.0 – 5.8+: no source changes should be needed.** This is source-only (no prebuilt binaries), so the engine just recompiles it against whatever version opens it.
- **The one thing to actually change per target:** the `"EngineVersion"` field in `RootMovement.uplugin`. It currently reads `"5.7.0"` (despite the folder being named `_5.5` — a pre-existing inconsistency worth fixing). Unreal uses this field as an advisory compatibility gate; set it to your target version, or remove the field entirely to skip the gate.
- **UE4 or very old UE5 preview builds:** not verified. Before assuming compatibility, manually confirm `UCancellableAsyncAction`, `FRootMotionSource_ConstantForce`, and the `ERootMotionFinishVelocityMode`/`FinishVelocityParams` API exist and have the same shape in that version (Epic's API reference or that engine's own source).
- No de-coupling work is required for a version port — the dependency audit already confirmed there's no project-specific code to strip out. It's a recompile, not a rewrite.

## Known issue

`UAsyncRootMovement::Cancel()` dereferences `CharacterMovement` without a null check. `Activate()`'s `OnFail` path can be reached with an invalid `CharacterMovement`, so a `Cancel()` call after that failure path is a potential null-pointer dereference. Guard this with a validity check before calling `RemoveRootMotionSourceByID` if you touch this code.

## API reference

### `URootMovementLibrary::ApplyRootMotionConstantForce`

**Blueprint node:** *Apply Root Motion Constant Force*

```cpp
static void ApplyRootMotionConstantForce(
    const UObject* WorldContext,
    UCharacterMovementComponent* CharacterMovement,
    FVector WorldDirection,
    float Strength,
    float Duration,
    bool bIsAdditive,
    UCurveFloat* StrengthOverTime,
    ERootMotionFinishVelocityMode VelocityOnFinishMode,
    FVector SetVelocityOnFinish,
    float ClampVelocityOnFinish,
    bool bEnableGravity
);
```

Builds an `FRootMotionSource_ConstantForce` (`Force = WorldDirection * Strength`, `Priority = 5`, `AccumulateMode` = Additive or Override per `bIsAdditive`) and applies it via `CharacterMovement->ApplyRootMotionSource(...)`. **The returned root-motion source ID is discarded** — fire-and-forget, no completion signal, no explicit cancel. Intended for short, self-expiring, repeatedly-reapplied forces (e.g. a thrust tick reapplied every frame/timer interval), not a single long-duration force you might need to interrupt.

`bEnableGravity = false` sets `ERootMotionSourceSettingsFlags::IgnoreZAccumulate` (force overrides vertical accumulation, no gravity fighting it); `true` lets normal gravity keep accumulating alongside the force.

If `CharacterMovement` is null, logs a warning (`LogTemp`, Warning) and returns without applying anything.

### `UAsyncRootMovement`

**Blueprint node:** *Apply Root Motion Constant Force with Callbacks*

```cpp
static UAsyncRootMovement* AsyncRootMovement(
    const UObject* WorldContext,
    UCharacterMovementComponent* CharacterMovement,
    FVector WorldDirection,
    float Strength,
    float Duration,
    bool bIsAdditive,
    UCurveFloat* StrengthOverTime,
    ERootMotionFinishVelocityMode VelocityOnFinishMode,
    FVector SetVelocityOnFinish,
    float ClampVelocityOnFinish,
    bool bEnableGravity
);
```

Same parameters, but returns a `UCancellableAsyncAction` with two Blueprint-assignable delegates:

- `OnComplete` — fired when `Duration` elapses naturally.
- `OnFail` — fired if the world or `CharacterMovement` is invalid when the action tries to activate.

`Activate()` applies the force and stores the returned `RootMotionSourceID`, then starts a one-shot timer for `Duration`; on expiry it broadcasts `OnComplete` and calls `Cancel()`. `Cancel()` calls `Super::Cancel()` and, if the completion timer is still valid, calls `CharacterMovement->RemoveRootMotionSourceByID(RootMotionSourceID)` for explicit teardown — use this node over the plain library call whenever you need a guaranteed cleanup callback or the ability to cancel the force before it naturally expires.

## Installing in your own project

1. Download or copy the `RootMovement_5.5` folder into your project's `Plugins/` directory.
2. Open your project — Unreal will prompt to build the new plugin (or generate/build project files first on a source build).
3. Confirm it's enabled via **Edit → Plugins** (search "RootMovement").
4. The two nodes above become available under the "RootMovement" category on any `UCharacterMovementComponent`.

## Requirements

- Unreal Engine 5.x (see "Adapting this for a different Unreal Engine version" above)
- No dependencies beyond stock engine modules: `Core`, `CoreUObject`, `Engine`, `Slate`, `SlateCore`

## License

[MIT](LICENSE) — use it, modify it, ship it in your own commercial or non-commercial project. Just keep the copyright notice.

# OUTLAND — offline extraction shooter blueprint

A small, playable browser prototype is included in this repository. It is an original tactical extraction concept, using familiar genre conventions rather than Arena Breakout assets or branding. The current playable build is a top-down slice; this document describes a production-ready Unity URP mobile direction.

## Product and core loop

**Pillars:** deliberate tactical combat, meaningful risk/reward, readable realism, and a complete solo/offline loop. Target Android/iOS mid-range devices (Snapdragon 7-series / Dimensity 8000-series, 6–8 GB RAM), 60 FPS target with a 30 FPS thermal fallback. Establish a 16+ content rating and make injury feedback non-graphic.

1. **Safehouse:** stash grid with item footprint, weight, stack size, condition, insurance flag and category filters. Equip one primary, sidearm, armor, rig, backpack, meds and ammunition. Weapon workbench previews compatible optics, muzzle devices, magazines and grips; show changed handling, dispersion, ergonomics and cost before committing. Local traders have rotating stock, reputation, buy/sell prices and short offline refresh timers. Save profile, stash and trader seed locally.
2. **Raid:** choose a compact authored map, kit, difficulty and insertion point. Scavenge buildings and AI, monitor weight/ammo/injury, and choose one of several extraction conditions (open, timed, or item/lever dependent). Gunfire propagates as a sound event with range and occlusion; it attracts nearby AI. Combat is lethal enough to reward cover, controlled fire, and disengagement.
3. **Outcome:** extraction commits carried loot and mission rewards to the stash. KIA drops uninsured raid gear and carried loot; insured gear returns after a configurable recovery delay, except items looted from the map. Show an itemized debrief, injuries, XP and trader standing. Save atomically before returning to the safehouse.

**Combat/health rules:** projectile or hitscan-with-travel-time ballistics; deterministic weapon spread, recoil impulse, velocity-based drop and material/armor penetration. Armor uses plate zones, durability, and penetration class; avoid a single binary armor check. Body zones track head, thorax, arms and legs, with damage multipliers, bleeding timers, pain effects, and movement/aim penalties. Bandages stop light bleeds; a surgical kit restores a disabled limb; painkillers temporarily suppress pain. All treatment has an animation/action time and can be interrupted.

**Content scope for a vertical slice:** one 500–700 m play space, 3 extraction points, 4 container types, 6 AI archetypes (scav, guard, scout, flanker, marksman, boss), 8 weapons, 15 attachments, 25 loot items, one trader and two raid modifiers. Prefer hand-authored encounter spaces over procedural geometry; randomize loot tables and patrol seeds.

## Unity project structure

```text
Assets/
  Art/{Characters,Environment,Props,UI}/
  Audio/{Ambience,Weapons,Footsteps,UI}/
  Data/{Items,Weapons,Armor,LootTables,AI}/       # ScriptableObjects
  Prefabs/{Player,Weapons,AI,World,UI}/
  Scenes/{Boot,Safehouse,OldMill}/
  Scripts/
    Bootstrap/          # save, scene flow, service wiring
    Gameplay/{Player,Combat,Health,Inventory,Extraction}/
    AI/{Perception,Decision,Navigation}/
    Economy/{Trader,Insurance}/
    UI/{HUD,Safehouse,Touch}/
    Platform/{Save,Quality,Telemetry}/
  Settings/{URP,Quality}/
  Tests/{EditMode,PlayMode}/
```

### Runtime architecture

- **Unity 2022 LTS or Unity 6 + URP**, single-player deterministic simulation, no dedicated server or network dependency. Use a scene bootstrap and explicit interfaces (`IInventoryService`, `ISaveService`, `IAudioPerception`) rather than global mutable singletons.
- Author item, weapon, armor, loot and AI configuration as immutable `ScriptableObject` data. Runtime inventory uses stable item IDs plus mutable stack/condition/insurance state. A transactional `RaidRunState` is the only object that commits extraction/KIA results to profile storage.
- Split input from action: Input System action maps (`Safehouse`, `OnFoot`, `UI`), a `PlayerMotor`, `WeaponController`, `HealthController`, and HUD view-model. Use events for inventory, health, ammo and raid state. Persist versioned JSON/binary profile to `Application.persistentDataPath` using temp-write + replace and a backup; validate data on load.
- Weapon loop: input → fire gate → recoil/spread → shot simulation → hit/material/armor resolution → damage event → effects/audio. Use pooled projectiles, decals, shell casings and impacts; use hitscan for short-range rifles unless travel-time/drop gameplay is required. A shared weapon definition drives both UI stats and runtime.
- AI: small hierarchical state machine/Behavior Tree: Patrol → InvestigateSound/LastKnownPosition → SeekCover → Engage/Suppress → Flank → Search → Return. Perception emits lightweight sound stimuli with source, radius, timestamp and obstruction attenuation. Bosses use authored phases, not just inflated HP. Sample NavMesh and sensors at staggered intervals; reserve full-rate perception/aim only for nearby threats.

## Mobile graphics/performance plan

- Start with URP Forward rendering; bake static environment GI and mixed/baked lights. Use Light Probes for moving characters, Reflection Probes for authored interiors and a subtle sky/ambient contribution. Keep real-time shadowing to the player/nearby AI only, with short distance, low resolution and cached shadow settings; prefer blob/contact decals for small props.
- PBR authoring with ORM packed as R=Occlusion, G=Roughness, B=Metallic. Share trim sheets/materials, use texture atlases for props, ASTC 6×6 or 8×8 (validate detail at device resolution), mipmaps and platform-specific texture limits. Use tiling detail normals sparingly; triplanar is for selected large terrain/rock materials, not every prop. Avoid expensive transparent layers and overdraw.
- Bake occlusion data for indoor/urban clusters, add GPU occlusion where measured, and validate visibility cells against moving doors. Use 3–5 LODs for characters/large props and impostors/billboards for distant foliage. GPU Instancing for foliage, debris and repeated props; static batching for non-instanced architecture. Use pooled VFX and cap particles/decals.
- Set a GPU/frame-time budget by device tier: target ≤16.6 ms sustained, profile CPU and GPU separately. Dynamic resolution (e.g. 0.75–1.0), URP Render Scale and FSR 2 where supported can recover GPU time; keep a native-resolution UI. Upscaling is not a substitute for reducing overdraw, material passes or shadow cost. Reduce resolution/effects on thermal pressure and expose a 30 FPS power mode.
- Profile on real devices, not only Editor: Unity Profiler, Frame Debugger, Memory Profiler and Android GPU Inspector/Xcode Instruments. Budget memory for textures, audio, meshes, and peak raid loading. Stream additive map sections and unload safehouse-only assets. Check cold start, 20-minute thermal soak, battery use, and 6 GB memory pressure.

## Offline AI and difficulty

Distance throttling should be deterministic and staggered: nearby (<25 m) combat/senses at 10–15 Hz; mid-range (25–80 m) at 3–5 Hz; distant at 0.5–1 Hz or event-driven. Keep animation and navigation update rates separately configurable. Run expensive line-of-sight raycasts on a rotating budget and cache results briefly. AI reacts to the last known player position rather than omnisciently tracking through walls. Cover selection scores concealment, incoming-fire direction, flank routes and distance; suppressive fire consumes ammo and is intentionally imprecise. Boss encounter placement and loot are seeded per raid.

## Touch controls and accessibility

Use a left floating movement stick and a right drag-to-look/aim region. Keep contextual fire/ADS, crouch/prone, lean, reload, interact and med buttons on the right, but show lean/prone/weapon-check only in an expandable tactical strip. Make extraction/loot prompts contextual. Use a hold/tap setting for ADS, optional gyro aiming, left-handed layout, button-size/opacity sliders, sensitivity curves, haptics toggle, color-safe injury indicators, subtitles and aim-assist tuning for PvE. Looting is a two-pane inventory with tap-to-transfer and hold-to-compare; pause world time only in safehouse, never during a raid.

## Starter Unity recoil component

Place in `Assets/Scripts/Gameplay/Combat/WeaponRecoil.cs`. Drive it from a weapon's successful-fire event; call `AddShot()` only after the fire gate accepts a round. A camera rig can consume `ViewKick`, while weapon animation consumes `WeaponKick`. Tune values per weapon ScriptableObject and test with device touch sensitivity.

```csharp
using UnityEngine;

namespace Outland.Gameplay.Combat
{
    /// <summary>Frame-rate independent, recoverable recoil impulses for a mobile weapon.</summary>
    public sealed class WeaponRecoil : MonoBehaviour
    {
        [Header("Per-shot impulse, degrees / local units")]
        [SerializeField] private Vector2 viewImpulse = new(0.65f, 0.18f);
        [SerializeField] private Vector3 weaponImpulse = new(0.025f, 0.01f, -0.035f);
        [Header("Recovery")]
        [SerializeField, Min(0.1f)] private float returnSpeed = 8f;
        [SerializeField, Min(0.1f)] private float snappiness = 14f;
        [SerializeField, Range(0f, 1f)] private float adsMultiplier = 0.58f;

        private Vector2 currentView, targetView;
        private Vector3 currentWeapon, targetWeapon;
        private Quaternion weaponRest;
        private bool aiming;

        public Vector2 ViewKick => currentView;

        private void Awake() => weaponRest = transform.localRotation;

        public void SetAiming(bool value) => aiming = value;

        // Call once for each shot actually fired, not for each input frame.
        public void AddShot(float stability = 1f)
        {
            float scale = Mathf.Max(0.1f, stability) * (aiming ? adsMultiplier : 1f);
            targetView += new Vector2(viewImpulse.x, Random.Range(-viewImpulse.y, viewImpulse.y)) * scale;
            targetView.x = Mathf.Clamp(targetView.x, -2f, 8f);
            targetView.y = Mathf.Clamp(targetView.y, -3f, 3f);
            targetWeapon += weaponImpulse * scale;
        }

        // Consume ViewKick in the player look rig as a local pitch/yaw offset.
        private void LateUpdate()
        {
            float dt = Time.deltaTime;
            currentView = Vector2.Lerp(currentView, targetView, snappiness * dt);
            targetView = Vector2.Lerp(targetView, Vector2.zero, returnSpeed * dt);
            currentWeapon = Vector3.Lerp(currentWeapon, targetWeapon, snappiness * dt);
            targetWeapon = Vector3.Lerp(targetWeapon, Vector3.zero, returnSpeed * dt);
            transform.localRotation = weaponRest * Quaternion.Euler(currentWeapon);
            transform.localPosition = Vector3.Lerp(transform.localPosition, -currentWeapon, snappiness * dt);
        }
    }
}
```

**Implementation order:** movement/aim and one weapon → container interaction/inventory → extraction + transactional outcome → one scav archetype → touch UX → baked map art/LOD → device profiling and tuning. Add automated tests for loot transfer, insurance, KIA rollback, extraction commit, armor durability, and save migration before content scale-up.

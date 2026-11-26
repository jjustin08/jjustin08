# Gods Gather — Modular attack abilities

Selected original C# source from Gods Gather, a Unity third-person action roguelite prototype. The excerpts below retain the original code; surrounding members and formatting have been omitted where indicated. The project history records my work on the ability refactor, attack execution and the Flurry ability.

## Design

A common attack base handles activation, direction and handing attack data to an executor. Individual attacks describe their own timings, damage and hit volumes instead of implementing separate physics and telegraph logic.

```text
AttackAbility
    └── FlurryAbility → AttackData → HitPhase[] → HitVolume[]
                             ↓
                      AttackExecutor
```

### 1. Attack abstraction

Source: `AttackAbility.cs`. The abstract base exposes timing properties, receives progression and execution dependencies through its constructor, and delegates attack-specific data creation to subclasses. It also supports camera-relative and non-camera-relative direction selection.

```csharp
using UnityEngine;

[System.Serializable]
public abstract class AttackAbility : IAbility
{
    public abstract float cooldown { get; }
    public abstract float activeDuration { get; }

    public abstract float windUp { get; }
    public abstract float countdown { get; }
    public abstract float unWind { get; }

    public bool IsActive { get; private set; }

    protected readonly Character_Progression stats;
    protected readonly AttackExecutor executor;

    protected AttackAbility(Character_Progression stats, AttackExecutor executor)
    {
        this.stats = stats;
        this.executor = executor;
    }

    public virtual bool CanUse(Character_Controller controller) => true;

    public virtual void Use(Character_Controller controller)
    {
        IsActive = true;
        Vector3 forward = GetAttackDirection(controller);
        controller.transform.rotation = Quaternion.LookRotation(forward);
        AttackData data = CreateData();
        executor.StartAttack(data, activeDuration);
    }

    public virtual void Tick(float deltaTime) { }

    public virtual void Cancel()
    {
        IsActive = false;
    }

    protected abstract AttackData CreateData();  // Subclasses must implement this for their hit phases

    protected Vector3 GetAttackDirection(Character_Controller controller)
    {
        var input = controller.CurrentInput.Move;
        var cam = controller.useCameraForMovement ? Camera.main : null;
        if (input.sqrMagnitude > 0.1f) return GetDirection(input, cam);
        return cam != null ? Vector3.ProjectOnPlane(cam.transform.forward, Vector3.up).normalized : controller.transform.forward;
    }

    protected Vector3 GetDirection(Vector2 input, Camera cam)
    {
        if (cam == null) return new Vector3(input.x, 0, input.y).normalized;
        var forward = Vector3.ProjectOnPlane(cam.transform.forward, Vector3.up).normalized;
        var right = Vector3.ProjectOnPlane(cam.transform.right, Vector3.up).normalized;
        return (right * input.x + forward * input.y).normalized;
    }

    public virtual float GetUtilityScore(Character_Controller controller,Transform target,float distance,float healthPct)
    {
        return 1;
    }
}
```

### 2. Concrete multi-hit ability

Source: `FlurryAbility.cs`. Three hit phases describe a sequence of spherical hits at different wind-up times. Each uses the same executor without implementing its own collision loop. The numeric phase arguments are the actual timings; the original inline timing comments are stale.

```csharp
using UnityEngine;

[System.Serializable]
public class FlurryAbility : AttackAbility
{
    public override float cooldown => 5f;
    public override float activeDuration => 2.0f; // Total anim time: windUp + countdown

    public override float windUp => 0.2f; // not used here
    public override float countdown => 0.1f; // not used here
    public override float unWind => 0.1f; // not used here
    public FlurryAbility(Character_Progression stats, AttackExecutor exec) : base(stats, exec) { }

    protected override AttackData CreateData()
    {
        AttackData data = new AttackData();
        // Phase 1: Quick first hit, invisible
        HitPhase hit1 = new HitPhase(0.4f, 0.1f, false); // Appear at 0.2s, damage after 0.1s countdown (at 0.3s)
        hit1.volumes.Add(new HitVolume(
            Vector3.forward * 1.5f,
            Quaternion.Euler(0, 0, 90f),
            new Vector3(1.5f, 1.0f, 0f),
            HitShape.Sphere,
            1 + stats.damage,
            Vector3.forward * 0f
        ));
        data.hitPhases.Add(hit1);
        // Phase 2
        HitPhase hit2 = new HitPhase(0.8f, 0.1f, false); // Appear at 0.6s, damage at 0.7s
        hit2.volumes.Add(new HitVolume(
            Vector3.forward * 1.5f,
            Quaternion.Euler(0, 0, 90f),
            new Vector3(1.5f, 1.0f, 0f),
            HitShape.Sphere,
            1 + stats.damage,
            Vector3.forward * 0f
        ));
        data.hitPhases.Add(hit2);
        // Phase 3
        HitPhase hit3 = new HitPhase(1.4f, 0.1f, false); // Appear at 1.0s, damage at 1.1s
        hit3.volumes.Add(new HitVolume(
            Vector3.forward * 1.5f,
            Quaternion.Euler(0, 0, 90f),
            new Vector3(1.5f, 1.0f, 0f),
            HitShape.Sphere,
            1 + stats.damage,
            Vector3.forward * 0f
        ));
        data.hitPhases.Add(hit3);
        return data;
    }
}
```

### 3. Shared execution and hit detection

Source: `AttackExecutor.cs` (selected methods). The executor initializes phase timers, advances phases, displays optional warning volumes and resolves either a projectile spawn or an overlap query. All attacks share this path. The overlap queries use a reusable collider buffer instead of allocating a new result array for each query.

```csharp
private Collider[] overlapResults = new Collider[64];

public void StartAttack(AttackData data, float duration)
{
    if (data == null || data.hitPhases.Count == 0)
    {
        Debug.LogWarning("AttackExecutor: Started with null or empty AttackData");
        return;
    }
    if (isAttacking) EndAttack();
    currentAttack = data;
    isAttacking = true;
    hitIdsThisAttack.Clear();
    attackTimer = duration;
    originalDuration = duration;
    phaseTimers.Clear();
    for (int i = 0; i < data.hitPhases.Count; i++)
    {
        phaseTimers.Add(new PhaseTimer(data.hitPhases[i]));
    }
}

private void Update()
{
    if (!isAttacking || currentAttack == null) return;
    attackTimer -= Time.deltaTime;
    if (attackTimer <= 0f)
    {
        EndAttack();
        return;
    }
    float timeElapsed = originalDuration - attackTimer;
    for (int i = 0; i < phaseTimers.Count; i++)
    {
        var pt = phaseTimers[i];
        if (!pt.started && timeElapsed >= pt.phase.windUpTime)
        {
            pt.started = true;
            pt.activeRemaining = pt.phase.countdownTime;
            pt.originalCountdown = pt.phase.countdownTime;
            pt.resolved = false;
            hitIdsThisAttack.Clear(); // Clear for new phase
            if (pt.phase.visible)
            {
                foreach (var vol in pt.phase.volumes)
                {
                    Vector3 worldPos = transform.position + transform.rotation * vol.offset;
                    Quaternion worldRot = transform.rotation * vol.localRot;
                    visualizer.ShowVisual(worldPos, worldRot, vol.size, vol.shape, pt.phase.countdownTime);
                }
            }
        }
        if (pt.started)
        {
            pt.activeRemaining -= Time.deltaTime;
            if (pt.activeRemaining <= 0f && !pt.resolved)
            {
                pt.resolved = true;

                foreach (var vol in pt.phase.volumes)
                {
                    Vector3 worldPos = transform.position + transform.rotation * vol.offset;
                    worldPos.y += 0.6f;
                    Quaternion worldRot = transform.rotation * vol.localRot;

                    if (vol.shape == HitShape.Projectile && projectilePrefab != null)
                    {
                        // Spawn projectile instead of overlap check
                        GameObject projObj = Instantiate(projectilePrefab, worldPos, worldRot);
                        Projectile proj = projObj.GetComponent<Projectile>();

                        if (proj != null)
                        {
                            proj.damage = vol.damage;
                            proj.knockbackForce = vol.knockback;
                            proj.attacker = gameObject;
                            proj.attackerId = characterId;
                            proj.speed = vol.size.y;           // Using size.y for speed
                            proj.lifetime = vol.size.z;        // Using size.z for lifetime
                                                               // Optional: scale the visual if desired
                            proj.transform.localScale = new Vector3(vol.size.x, vol.size.x, vol.size.x); // uniform scale from radius
                        }
                        else
                        {
                            Debug.LogWarning("Projectile prefab missing Projectile component!");
                            Destroy(projObj);
                        }
                    }
                    else
                    {
                        // Original area/overlap behavior for Box, Sphere, Capsule, Cone
                        int hits = GetOverlaps(vol, worldPos, worldRot, overlapResults, hitLayerMask);
                        ProcessHits(hits, worldPos, worldRot, vol);
                    }
                }

                if (pt.phase.visible)
                {
                    visualizer.HideVisual();
                }
            }
        }
    }
    // End early if all phases done
    bool allDone = true;
    for (int i = 0; i < phaseTimers.Count; i++)
    {
        if (!phaseTimers[i].resolved) allDone = false;
    }
    if (allDone) EndAttack();
}

private int GetOverlaps(HitVolume vol, Vector3 worldPos, Quaternion worldRot, Collider[] results, LayerMask mask)
{
    if (vol.shape == HitShape.Box)
    {
        return Physics.OverlapBoxNonAlloc(worldPos, vol.size * 0.5f, results, worldRot, mask);
    }
    else if (vol.shape == HitShape.Capsule)
    {
        Vector3 p1 = worldPos;
        Vector3 p2 = worldPos + worldRot * Vector3.up * vol.size.y;
        float radius = vol.size.x;
        return Physics.OverlapCapsuleNonAlloc(p1, p2, radius, results, mask);
    }
    else if (vol.shape == HitShape.Sphere)
    {
        return Physics.OverlapSphereNonAlloc(worldPos, vol.size.x, results, mask);
    }
    else if (vol.shape == HitShape.Cone)
    {
        // Use sphere for candidates
        float coneLength = vol.size.x;
        float coneAngleDeg = vol.size.y;
        float maxRadius = coneLength * Mathf.Tan(coneAngleDeg * Mathf.Deg2Rad / 2f);
        return Physics.OverlapSphereNonAlloc(worldPos, maxRadius, results, mask);
    }
    return 0;
}
```

## Scope and limitations

This is original prototype code, not a standalone library. It depends on Unity and project-specific types, including the character controller, attack data, visualizer and projectile components. Cooldown properties describe ability configuration; cooldown enforcement is not shown here. The fixed-size overlap buffer and handling of overlapping hit phases are areas to revisit as the combat system matures. No runtime testing is claimed for these excerpts.

The complete game repository remains private.

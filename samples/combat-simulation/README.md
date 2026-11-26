# Kami — Shared client/server combat simulation

Selected source-code excerpts from **Kami**, a Unity/C# multiplayer auto-battler developed at Finite Games Studio. These are excerpts of the original project implementation, not a recreated demo. Only surrounding class members and indentation/whitespace have been omitted or adjusted.

## Problem and contribution

The server must resolve battles immediately, while the client must display the combat over time. I refactored the client to use the same combat simulation as the server, rather than maintaining separate gameplay logic.

## 1. Client-side fixed-step execution

Source: `C_StaticCombat.cs`. The client accumulates Unity's fixed-update delta and advances the shared simulator in 0.1-second steps.

```csharp
private static float accumulatedTime = 0f;
private static readonly float timeStep = 0.1f;

public static void FixedCombatUpdate()
{
    if (!isCombatActive)
        return;

    accumulatedTime += Time.fixedDeltaTime;

    while (accumulatedTime >= timeStep)
    {

        if (SH_CombatSimulator.IsCombatOver())
        {
            isCombatActive = false;

            break;
        }
        SH_CombatSimulator.CombatUpdate();

        accumulatedTime -= timeStep;
    }
}
```

## 2. Instant battle resolution

Source: `SH_CombatSimulator.cs`. The server runs the same simulation loop without waiting for elapsed real time, records the outcome and returns the number of iterations. The method also clears temporary combat effects and companions.

```csharp
public static int InstantCombatSimulation()
{

    while(!IsCombatOver())
    {
        CombatUpdate();
    }

    DetermineCombatOutcome();

    SH_AbilityEffects.ResetEffects();

    for (int e = pawns.Count - 1; e >= 0; e--)
    {
        if (pawns[e].GetStats().IsCompanion())
        {
            SH_Pawn pawn = pawns[e];
            GameObject.Destroy(pawn.gameObject);
            pawns.RemoveAt(e);
        }
    }

    int tempIterations = iterations;
    Reset();
    return tempIterations; // Return iteration count
}
```

## 3. Shared combat update

Source: `SH_CombatSimulator.cs`. Each iteration updates living units' stats and AI, advances ability effects and adds newly spawned units after iteration over the existing collection.

```csharp
public static void CombatUpdate()
{
    iterations++;
    List<SH_Pawn> newPawns = new List<SH_Pawn>();

    foreach (SH_Pawn pawn in pawns)
    {
        if (pawn.GetStats().GetCurrentHealth() > 0)
        {
            pawn.GetStats().StatsUpdate(timeStep); // Update buffs
            pawn.GetAI().AiUpdate(timeStep, pawns, newPawns); // Pass currentTime for logging
        }

    }
    SH_AbilityEffects.UpdateEffects(timeStep, pawns, newPawns); // Update ability effects


    pawns.AddRange(newPawns);
}
```

## 4. Combat termination

Source: `SH_CombatSimulator.cs`. Combat ends when one side has no surviving pawns or when the simulation reaches the iteration limit (500 in the reviewed source).

```csharp
public static bool IsCombatOver()
{
    if (iterations >= maxIterations) // Enforce max duration
    {
        return true;
    }
    List<SH_Pawn> aliveOne = new List<SH_Pawn>();
    List<SH_Pawn> aliveTwo = new List<SH_Pawn>();
    int playerIdOne = match.GetPlayers()[0].Id;
    int playerIdTwo = match.GetPlayers()[1].Id;

    foreach (SH_Pawn pawn in pawns)
    {
        if (pawn.GetStats().GetCurrentHealth() > 0)
        {
            if (pawn.OwnerId == playerIdOne) aliveOne.Add(pawn);
            else if (pawn.OwnerId == playerIdTwo) aliveTwo.Add(pawn);
        }
    }

    bool over = aliveOne.Count == 0 || aliveTwo.Count == 0;
    return over;
}
```

## Scope

These excerpts depend on the original Unity project, including `SH_Pawn`, `SH_Match` and the ability system. They are presented for code review rather than as a standalone runnable project. Shared logic reduces duplicate implementations but does not by itself guarantee identical outcomes or network synchronization. No build or runtime verification is claimed.

The complete game repositories remain private.

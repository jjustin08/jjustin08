# Ascending Flags — Server-side card placement

Selected original Unity/C# code from Ascending Flags, a multiplayer tactical card and board game. The project history records my contributions to networked card placement and validation. Excerpts preserve the original implementation, with surrounding class members omitted and formatting and comments cleaned up for readability.

## Design

A client sends a card index and a network reference to a target tile. The server identifies the sender from the RPC metadata, retrieves the card from the player's server-side hand, validates the game rules and applies the action. A client RPC communicates the resulting placement.

```text
Client request
    ↓
Server RPC → sender ID → server-side hand
    ↓
Visibility / ownership / occupancy / mana checks
    ↓
Spend mana → spawn network object → remove card
    ↓
Client RPC
```

### 1. Process a placement request

Source: `ServerPlaceCards.cs`. The client does not supply the card's mana cost or card definition to the server RPC; those come from the server-side hand. After the rule checks, the server chooses the appropriate spawn path and updates the player's hand.

```csharp
[ServerRpc(RequireOwnership = false)]
private void PlaceCardServerRpc(int cardIndex, NetworkObjectReference tileSlotNetworkObjectReference, ServerRpcParams serverRpcParams = default)
{
    ulong playerID = serverRpcParams.Receive.SenderClientId;

    ExtraPlaceCard(cardIndex, tileSlotNetworkObjectReference, playerID);
}

public void ExtraPlaceCard(int cardIndex, NetworkObjectReference tileSlotNetworkObjectReference, ulong playerID)
{
    tileSlotNetworkObjectReference.TryGet(out NetworkObject tileNetworkObject);
    TileSlot tileSlot = tileNetworkObject.GetComponent<TileSlot>();

    _CardBaseSO cardSO = null;

    cardSO = cardManager.GetHand(playerID)[cardIndex];

    if (!PlacingCardChecks(playerID, cardSO, tileSlot))
    {
        return;
    }

    manaManager.UseMana(playerID, cardSO.manaCost);

    GameObject placedObject = null;
    if (cardSO is UnitCardSO)
    {
        UnitCardSO unitCardSO = cardSO as UnitCardSO;
        placedObject = SpawnUnit(unitCardSO, tileSlot, playerID);
        GameMode.Instance.AddUnitToList(placedObject.GetComponent<UnitInfo>());
    }
    else if (cardSO is TileCardSO)
    {
        TileCardSO tileCardSO = cardSO as TileCardSO;
        placedObject = SpawnTile(tileCardSO, tileSlot, playerID);
    }
    else
    {
        return;
    }

    cardManager.GetHand(playerID).RemoveAt(cardIndex);

    PlaceCardClientRpc(placedObject.GetComponent<NetworkObject>(), tileSlotNetworkObjectReference, cardIndex, playerID);
}
```

### 2. Validate game rules

Source: `ServerPlaceCards.cs`. Placement is restricted by fog-of-war visibility, tile ownership and occupancy for units, empty slots for tile cards and available mana.

```csharp
public bool PlacingCardChecks(ulong playerID, _CardBaseSO cardSO, TileSlot tileSlot)
{
    if (!tileSlot.GetComponent<FogOfWarController>().checkIfVisible(playerID))
    {
        return false;
    }
    if (cardSO is UnitCardSO)
    {
        if (tileSlot.GetTileInfo() == null)
        {
            return false;
        }
        if (tileSlot.GetTileInfo().GetPlayerOwnerShip().GetOwnerShip() != playerID)
        {
            return false;
        }
        if (tileSlot.GetUnit() != null)
        {
            return false;
        }
    }
    else if (cardSO is TileCardSO)
    {
        if (tileSlot.GetTileInfo() != null)
        {
            return false;
        }
    }
    else
    {
        return false;
    }

    if (manaManager.GetMana(playerID) < cardSO.manaCost)
    {
        return false;
    }

    return true;
}
```

### 3. Spawn a networked unit

Source: `ServerPlaceCards.cs`. The server creates the network object, attaches it to the target tile and sets its game-state references.

```csharp
private GameObject SpawnUnit(UnitCardSO unitCardSO, TileSlot tileSlot, ulong playerID)
{
    GameObject newUnit = Instantiate(unitCardSO.UnitPrefab.gameObject, tileSlot.GetTileInfo().GetUnitPlaceTransform());
    newUnit.GetComponent<NetworkObject>().Spawn();
    newUnit.GetComponent<NetworkObject>().TrySetParent(tileSlot.transform);

    newUnit.GetComponent<UnitInfo>().SetTile(tileSlot);
    newUnit.GetComponent<UnitInfo>().SetPlayerOwnerShip(playerID);
    newUnit.GetComponent<UnitInfo>().SetCardBaseSO(unitCardSO);
    tileSlot.SetUnit(newUnit.GetComponent<UnitInfo>());

    if (playerID == PLAYER_ONE)
    {
        newUnit.GetComponent<UnitVisualController>().ChangeMaterialOnBase(player1MAT);
        newUnit.transform.localRotation = Quaternion.Euler(0, 150, 0);
    }
    else if (playerID == PLAYER_TWO)
    {
        newUnit.GetComponent<UnitVisualController>().ChangeMaterialOnBase(player2MAT);
        newUnit.transform.localRotation = Quaternion.Euler(0, -30, 0);
    }
    return newUnit;
}
```

## Scope and limitations

This is original student-project prototype code, not a hardened multiplayer security implementation. The excerpts show server-side game-rule checks, but do not validate the card index or whether the incoming network reference resolves successfully before dereferencing it. Those checks would be important before shipping a public competitive game. The example is intentionally limited to card placement; it does not claim complete authorization for every multiplayer action.

The complete project repository remains private.

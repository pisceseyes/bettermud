# BetterMUD Logic System

This note explains how BetterMUD attaches logic modules to entities, how those logic modules are invoked, and how they fit into the broader action system.

## Big picture

The engine supports a reusable, data-driven behavior layer for entities:

- Characters
- Items
- Rooms
- Portals
- Regions

These can all carry one or more logic modules. A logic module is essentially a behavior plugin that responds to actions dispatched by the game engine.

This is a strong design pattern for a MUD because it lets you:
- attach behavior to an entity at runtime
- keep gameplay rules out of hardcoded C++ behaviour
- add or remove logic without recompiling the engine
- script or configure interactions by entity and world state

---

## Where logic exists in the code

The entity-level logic API is exposed in `LogicEntity.h`.

Key methods include:

```cpp
bool AddLogic( const std::string& p_logic );
bool AddExistingLogic( Logic* p_logic );
bool DelLogic( const std::string& p_logic );
Logic* GetLogic( const std::string& p_logic );
bool HasLogic( const std::string& p_logic );
int DoAction( const Action& p_action );
int GetLogicAttribute( const std::string& p_logic, const std::string& p_attr );
```

These methods are composed into the actual entity types via the `HASLOGICHEADERS` macro in `BetterMUD/accessors/AccessorMacros.h`.

That macro defines the interface shared by relevant entity types:

```cpp
#define HASLOGICHEADERS                                                     \
    bool AddLogic( const std::string& p_logic );                            \
    bool AddExistingLogic( Logic* p_logic );                                \
    bool DelLogic( const std::string& p_logic );                            \
    Logic* GetLogic( const std::string& p_logic );                          \
    bool HasLogic( const std::string& p_logic );                            \
    int DoAction( const Action& p_action );                                 \
    int DoAction( const std::string& p_act, entityid p_data1 = 0, entityid p_data2 = 0, entityid p_data3 = 0, entityid p_data4 = 0, const std::string& p_data = "" ); \
    int GetLogicAttribute( const std::string& p_logic, const std::string& p_attr ); \
    void AddHook( TimedAction* p_hook );                                    \
    void DelHook( TimedAction* p_hook );                                    \
    size_t Hooks();                                                         \
    void KillHook( const std::string& p_act, const std::string& p_stringdata ); \
    void ClearHooks();                                                      \
    void ClearLogicHooks( const std::string& p_logic );
```

This is a strong sign that logic is meant to be a standard behavior layer across the game world, not just a special case for one class.

---

## How logic is stored

The entity stores a `LogicCollection`:

```cpp
protected:
    LogicCollection m_logic;
```

The collection manages the actual logic modules associated with the entity.

This is important because it means logic is not hardcoded into entity classes; instead, it is attached compositionally from the outside.

---

## How logic is loaded and invoked

The main runtime entry point is `Game::LogicAction()` in `BetterMUD/Game.cpp`.

```cpp
void Game::LogicAction( const Action& p_act )
{
    std::string modname = BasicLib::ParseWord( p_act.stringdata, 0 );
    Logic* l = 0;
    switch( p_act.data1 )
    {
    case CHARACTER:
        l = character( p_act.data2 ).GetLogic( modname );
        break;
    case ITEM:
        l = item( p_act.data2 ).GetLogic( modname );
        break;
    case ROOM:
        l = room( p_act.data2 ).GetLogic( modname );
        break;
    case PORTAL:
        l = portal( p_act.data2 ).GetLogic( modname );
        break;
    case REGION:
        l = region( p_act.data2 ).GetLogic( modname );
        break;
    }
    if( !l ) throw Exception( "Game::LogicAction: Cannot load logic " + modname );
    l->DoAction( p_act );
}
```

This tells us several key things:

1. Logic is keyed by a module name string.
2. The target entity type is encoded in `Action.data1`.
3. The target entity id is encoded in `Action.data2`.
4. The engine resolves a `Logic*` and calls `DoAction()` on it.

This is the central event dispatch path for logic.

---

## How logic gets attached

The `Game` class also provides generic Add/Del logic operations:

```cpp
void Game::AddLogic( const Action& p_act )
{
    switch( p_act.data1 )
    {
    case CHARACTER:
        character( p_act.data2 ).AddLogic( p_act.stringdata );
        break;
    case ITEM:
        item( p_act.data2 ).AddLogic( p_act.stringdata );
        break;
    case ROOM:
        room( p_act.data2 ).AddLogic( p_act.stringdata );
        break;
    case PORTAL:
        portal( p_act.data2 ).AddLogic( p_act.stringdata );
        break;
    case REGION:
        region( p_act.data2 ).AddLogic( p_act.stringdata );
        break;
    }
}
```

And removal:

```cpp
void Game::DelLogic( const Action& p_act  )
{
    switch( p_act.data1 )
    {
    case CHARACTER:
        character( p_act.data2 ).DelLogic( p_act.stringdata );
        break;
    case ITEM:
        item( p_act.data2 ).DelLogic( p_act.stringdata );
        break;
    case ROOM:
        room( p_act.data2 ).DelLogic( p_act.stringdata );
        break;
    case PORTAL:
        portal( p_act.data2 ).DelLogic( p_act.stringdata );
        break;
    case REGION:
        region( p_act.data2 ).DelLogic( p_act.stringdata );
        break;
    }
}
```

This means the logic system can be manipulated dynamically at runtime using actions, not only when the server boots.

---

## What this means in gameplay

From a gameplay perspective, this is the MUD’s behavior system.

Examples from the code:

```python
me = BetterMUD.character( self.me )
if not me.HasLogic( "cantreceiveitems" ):
    me.AddLogic( "cantreceiveitems" )
```

This is not a direct C++ behavior branch. It is a runtime module attachment that changes how the character behaves.

The same pattern is used for:
- player status effects
- room traps
- item effects
- portal behaviors
- region rules
- faction states
- temporary buffs/debuffs

---

## Logic and command interaction

The command system and logic system are complementary.

Commands create actions, and logic responds to those actions.

For example:

- A player types `receive off`
- The player command updates state
- It calls `AddLogic( "cantreceiveitems" )`
- That logic module triggers new behavior the next time an item is received or an item action is processed

This is elegant because the command layer does not need to know every rule implementation detail. It simply triggers the correct action and the logic system handles the behavior.

---

## Logic hooks and timed effects

Logic modules are not only triggered by direct actions; they can also be connected to time-based events via the timed action system.

In `LogicEntity.h`, the logic system also supports hooks:

```cpp
typedef std::set<TimedAction*> actionhooks;
void AddHook( TimedAction* p_hook );
void DelHook( TimedAction* p_hook );
size_t Hooks();
void KillHook( const std::string& p_act, const std::string& p_stringdata );
void ClearHooks();
```

This allows logic modules to attach event listeners to timers, which is how things like:
- poison ticks
- regenerating health
- temporary status effects
- delayed spells
- event chains

are implemented.

So the model is:

- logic module = behavior rules
- timer = scheduled future action
- hook = association between entity and timer

Together, they produce stateful gameplay behavior over time.

---

## Why this matters architecturally

This system is one of the strongest parts of the project, because it separates:

- **entity state** (character/item/room identity)
- **behavior** (logic modules)
- **event dispatch** (Game::DoAction / Game::LogicAction)
- **time scheduling** (TimedAction)

That is a clean separation of concerns.

It also means if you refactor the engine, the logic layer should stay distinct from raw entity structure. It should be treated as a first-class subsystem, not hidden inside C++ logic or script-specific hacks.

---

## Recommended refactoring boundary

For maintainability, I would organize logic into a dedicated layer like this:

```text
BetterMUD/
├── core/
│   ├── Game.h
│   ├── Game.cpp
│   └── Action.h
├── entities/
│   ├── Character.h
│   ├── Item.h
│   ├── Room.h
│   ├── Region.h
│   └── Portal.h
├── logic/
│   ├── LogicCollection.h
│   ├── LogicEntity.h
│   ├── Logic.h
│   └── LogicRegistry.h
├── scripts/
│   └── Python bridge / script loader
├── data/
│   └── logics/
└── ...
```

This makes it much easier to reason about:
- entity model
- behavior model
- runtime dispatch
- script integration

---

## Example mental model

Think of the engine like this:

- Entity = the thing in the world
- Logic = the behavior attached to that thing
- Action = an event or command
- TimedAction = a future event to happen
- Hook = the binding between an entity and a future event

This gives you a modular system that can model:
- buffs and debuffs
- AI behavior
- environmental states
- timed traps
- faction effects
- custom magical outcomes

---

## Bottom line

The `data/logics` system is the MUD’s runtime behavior layer.

It is not just a set of scripts; it is the game’s dynamic rules engine.

The architecture is already conceptually strong:
- entities can carry multiple logic modules
- actions dispatch into logic modules
- timed actions can be scheduled and hooked to entities
- the engine can load behavior at runtime

The main improvement opportunity is organizational clarity, not conceptual redesign.

A refactor should preserve the logic architecture but make its responsibilities explicit and separated from the engine core.

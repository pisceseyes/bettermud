# Hooks and Timed Actions in BetterMUD

A complete explanation of how delayed actions and event subscriptions work in the engine.

## Overview

BetterMUD uses two interrelated systems to handle time-delayed actions and event notification:

1. **Timed Actions** — Actions that execute at a specific time in the future
2. **Hooks** — Event listeners that entities attach to timed actions so they're notified when the action occurs

Together, they enable:
- Delayed effects (poison damage ticking over time)
- Scheduled events (respawning, timers)
- Event cascades (when action X completes, trigger action Y)
- Persistent timers (saved/loaded across server restarts)

---

## Timed Actions

### What is a TimedAction?

A `TimedAction` is a wrapper around an `Action` plus an execution time:

```cpp
struct TimedAction
{
    BasicLib::sint64 executiontime;  // when to execute (game time in ms)
    Action actionevent;               // what to do
    bool valid;                       // is this timer still valid?
};
```

### The Action struct

Actions are the fundamental event type in BetterMUD:

```cpp
struct Action
{
    std::string actiontype;    // what happened (e.g., "attemptsay", "command")
    entityid data1;            // entity parameter 1
    entityid data2;            // entity parameter 2
    entityid data3;            // entity parameter 3
    entityid data4;            // entity parameter 4
    std::string stringdata;    // string parameter (e.g., text player said)
};
```

Example actions:
- `Action( "attemptsay", player_id, 0, 0, 0, "Hello world" )`
- `Action( "destroyitem", 0, item_id, 0, 0, "" )`
- `Action( "command", player_id, 0, 0, 0, "look" )`
- `Action( "modifyattribute", CHARACTER, player_id, -5, 0, "health" )`

### Creating and scheduling timed actions

You create a timed action and add it to the game loop:

```cpp
// Option 1: Schedule relative to now (5 seconds from now)
BasicLib::sint64 when = g_game.GetTime() + 5000;  // 5000 ms = 5 seconds
Action poison_tick( "modifyattribute", CHARACTER, player_id, -5, 0, "health" );
g_game.AddActionAbsolute( when, poison_tick );

// Option 2: Schedule relative to current time
// Execute in 10 seconds
g_game.AddActionRelative( 10000, poison_tick );

// Option 3: More concise syntax
g_game.AddActionRelative( 
    10000,                          // 10 seconds from now
    "modifyattribute",              // action type
    CHARACTER,                       // data1 (entity type)
    player_id,                       // data2 (entity id)
    -5,                              // data3 (attribute amount)
    0,                               // data4 (unused)
    "health"                         // stringdata (attribute name)
);
```

### The timer queue

The game maintains a **priority queue** of all pending timed actions:

```cpp
// In Game.h
typedef std::priority_queue< 
    TimedAction*, 
    std::vector<TimedAction*>, 
    TimedActionComp 
> timerqueue;

timerqueue m_timerregistry;  // sorted by executiontime (earliest first)
```

The priority queue is ordered by `executiontime` — the earliest timers are always at the front.

### ExecuteLoop — processing timed actions

The game's main loop continuously checks if any timed actions are ready:

```cpp
void Game::ExecuteLoop()
{
    while( m_running )
    {
        // While there are timed actions and the first one is ready
        while( !m_timerregistry.empty() && 
               m_timerregistry.top()->executiontime <= m_gametime.GetMS() )
        {
            // Pop the earliest ready action
            TimedAction* action = m_timerregistry.top();
            m_timerregistry.pop();
            
            // Execute it
            if( action->valid )
            {
                DoAction( action->actionevent );
            }
            
            delete action;
        }
        
        // ... handle player commands, etc ...
    }
}
```

This is the heartbeat of the game — every tick, it:
1. Checks if any timers are ready to fire
2. Executes them immediately
3. Returns to handling player input and network traffic

---

## Hooks

### What is a Hook?

A **hook** is a reference from an entity back to a `TimedAction`. It's the mechanism that lets entities "know about" upcoming actions that affect them.

When you schedule a timed action, you register the action on the entities it will affect via `Hook()`:

```cpp
void TimedAction::Hook()
{
    if( actionevent.actiontype == "attemptsay" ||
        actionevent.actiontype == "command" ||
        actionevent.actiontype == "attemptenterportal" ||
        actionevent.actiontype == "attempttransport" ||
        actionevent.actiontype == "transport" ||
        actionevent.actiontype == "destroycharacter" )
    {
        character( actionevent.data1 ).AddHook( this );
    }
    else if( actionevent.actiontype == "attemptgetitem" ||
             actionevent.actiontype == "attemptdropitem" )
    {
        character( actionevent.data1 ).AddHook( this );
        item( actionevent.data2 ).AddHook( this );
    }
    else if( actionevent.actiontype == "attemptgiveitem" )
    {
        character( actionevent.data1 ).AddHook( this );
        character( actionevent.data2 ).AddHook( this );
        item( actionevent.data3 ).AddHook( this );
    }
    else if( actionevent.actiontype == "destroyitem" )
    {
        item( actionevent.data1 ).AddHook( this );
    }
    else if( actionevent.actiontype == "messagelogic" ||
             actionevent.actiontype == "dellogic" ||
             actionevent.actiontype == "do" ||
             actionevent.actiontype == "modifyattribute" )
    {
        switch( actionevent.data1 )
        {
        case CHARACTER:
            character( actionevent.data2 ).AddHook( this );
            break;
        case ITEM:
            item( actionevent.data2 ).AddHook( this );
            break;
        case ROOM:
            room( actionevent.data2 ).AddHook( this );
            break;
        case PORTAL:
            portal( actionevent.data2 ).AddHook( this );
            break;
        case REGION:
            region( actionevent.data2 ).AddHook( this );
            break;
        }
    }
}
```

### Why do hooks exist?

Hooks serve several critical purposes:

#### 1. Event notification
Entities maintain a set of hooks so they know about all pending actions involving them:

```cpp
class LogicEntity
{
    typedef std::set<TimedAction*> actionhooks;
    actionhooks m_hooks;
};
```

#### 2. Cleanup when deleting entities
If you delete an entity (e.g., a character dies), you need to unhook all timed actions involving that entity:

```cpp
void Game::DestroyCharacter( entityid p_character )
{
    character c( p_character );
    c.ClearHooks();  // Unhook all pending actions
    // ... cleanup ...
}
```

If you didn't do this, a timer might fire later and try to interact with a dead character, causing a crash.

#### 3. Dynamic cancellation
You can kill specific hooks matching certain criteria:

```cpp
void KillHook( const std::string& p_act, const std::string& p_stringdata )
{
    // Find all hooks matching the action type and string data
    for( auto hook : m_hooks )
    {
        if( hook->actionevent.actiontype == p_act &&
            ParseWord( hook->actionevent.stringdata, 0 ) == p_stringdata )
        {
            hook->Unhook();  // Cancel the action
        }
    }
}
```

Example: Cancel all "poison damage" timers on a character:

```cpp
character( player_id ).KillHook( "modifyattribute", "health" );
```

---

## Complete Example: Poison Damage

Here's a realistic example showing how hooks and timed actions work together.

### Scenario: A player gets poisoned

**Step 1: Logic module applies poison**

A logic module (running in Python) decides to poison a player:

```python
# In data/logics/poisoned.py
class poisoned( Logic ):
    def __init__( self, owner ):
        self.owner = owner
        self.damage = 5
        self.ticks = 10
        self.current_tick = 0
    
    def OnApply( self ):
        # Schedule the first poison tick in 2 seconds
        action = BetterMUD.Action(
            "modifyattribute",
            BetterMUD.CHARACTER,
            self.owner.ID(),
            -self.damage,
            0,
            "health"
        )
        self.mud.AddActionRelative( 2000, action )
```

**Step 2: Action is scheduled and hooked**

The game receives the `AddActionRelative` call:

```cpp
void Game::AddActionRelative( BasicLib::sint64 p_time, const Action& p_action )
{
    // Create the timed action
    TimedAction* timed = new TimedAction( 
        GetTime() + p_time,  // executiontime = now + 2000ms
        p_action
    );
    
    // Register hooks on involved entities
    timed->Hook();  // <-- This calls AddHook on the target character
    
    // Add to priority queue
    m_timerregistry.push( timed );
}
```

The `Hook()` call adds this `TimedAction*` to the character's `m_hooks` set:

```cpp
void TimedAction::Hook()
{
    if( actionevent.actiontype == "modifyattribute" )
    {
        // Register on the character being modified
        character( actionevent.data2 ).AddHook( this );
    }
}

// In LogicEntity
void AddHook( TimedAction* p_hook )
{
    m_hooks.insert( p_hook );  // <-- Character now knows about this timer
}
```

Now the character's `m_hooks` contains a pointer to the pending poison tick action.

**Step 3: Player logs out**

The player decides to leave the game. The game needs to clean up:

```cpp
void Game::Logout( entityid p_id )
{
    character c( p_id );
    
    // Clear all hooks (cancel all pending actions on this character)
    c.ClearHooks();  // <-- Poison ticks are cancelled
    
    // ... save player ...
}
```

The `ClearHooks()` call iterates through all hooks and unhooks them:

```cpp
void ClearHooks()
{
    for( auto hook : m_hooks )
    {
        hook->Unhook();  // Mark as invalid
    }
}

void TimedAction::Unhook()
{
    valid = false;  // <-- This timer will be skipped in ExecuteLoop
}
```

**Step 4: Later, poison timer fires (if player is still online)**

The game loop reaches the execution time:

```cpp
void Game::ExecuteLoop()
{
    while( m_running )
    {
        // Current game time is 2000ms later
        if( m_timerregistry.top()->executiontime <= GetTime() )
        {
            TimedAction* poison_tick = m_timerregistry.top();
            m_timerregistry.pop();
            
            if( poison_tick->valid )  // <-- Check if it was unhook'd
            {
                // Execute: "modifyattribute CHARACTER player_id -5 0 health"
                DoAction( poison_tick->actionevent );
                
                // This calls the character's logic modules
                character( player_id ).DoAction( poison_tick->actionevent );
                
                // The "poisoned" logic module responds:
                // - Reduce health by 5
                // - If still poisoned, schedule next tick in 2 seconds
                // - If health < 0, remove poisoned logic
            }
            
            delete poison_tick;
        }
    }
}
```

---

## Practical Use Cases

### 1. Regeneration (healer logic module)

```python
# data/logics/regenerating.py
class regenerating( Logic ):
    def __init__( self, owner ):
        self.owner = owner
        self.hp_per_tick = 10
        self.tick_rate = 1000  # 1 second
    
    def Start( self ):
        # Schedule regeneration ticks
        for i in range(20):  # 20 ticks = 20 seconds
            when = self.tick_rate * (i + 1)
            self.mud.AddActionRelative(
                when,
                "modifyattribute",
                BetterMUD.CHARACTER,
                self.owner.ID(),
                self.hp_per_tick,
                0,
                "health"
            )
```

### 2. Spell cooldown (spellcasting logic)

```python
# data/logics/infirefallcooldown.py
class infirefallcooldown( Logic ):
    def __init__( self, owner ):
        self.owner = owner
    
    def OnCooldown( self ):
        # Schedule removal of cooldown in 30 seconds
        self.mud.AddActionRelative(
            30000,
            "dellogic",
            BetterMUD.CHARACTER,
            self.owner.ID(),
            0,
            0,
            "infirefallcooldown"
        )
```

### 3. NPC respawn (room logic)

```python
# data/logics/monsterrespawner.py
class monsterrespawner( Logic ):
    def __init__( self, owner ):
        self.owner = owner  # the room
        self.monster_template = 42
    
    def OnMonsterKilled( self ):
        # Respawn the monster in 5 minutes
        self.mud.AddActionRelative(
            300000,  # 5 minutes
            "spawncharacter",
            self.monster_template,
            self.owner.ID(),
            0,
            0,
            ""
        )
```

### 4. Drowning (character logic)

```python
# data/logics/drowning.py
class drowning( Logic ):
    def __init__( self, owner ):
        self.owner = owner
        self.breath_remaining = 30  # 30 seconds
    
    def OnDrown( self ):
        # Schedule suffocation damage every second
        for tick in range(self.breath_remaining):
            when = (tick + 1) * 1000
            self.mud.AddActionRelative(
                when,
                "modifyattribute",
                BetterMUD.CHARACTER,
                self.owner.ID(),
                -20,  # damage
                0,
                "health"
            )
```

---

## Persistence: Saving and Loading Timers

Timed actions can be saved to disk and restored when the server restarts:

```cpp
void TimedAction::Save( std::ofstream& p_stream )
{
    p_stream << "[TIMER]\n";
    p_stream << "    [TIME]    " << executiontime << "\n";
    p_stream << "    [NAME]    " << actionevent.actiontype << "\n";
    p_stream << "    [DATA1]   " << actionevent.data1 << "\n";
    p_stream << "    [DATA2]   " << actionevent.data2 << "\n";
    p_stream << "    [DATA3]   " << actionevent.data3 << "\n";
    p_stream << "    [DATA4]   " << actionevent.data4 << "\n";
    p_stream << "    [STRING]  " << actionevent.stringdata << "\n";
    p_stream << "[/TIMER]\n";
}

void Game::SaveTimers()
{
    std::ofstream file( "data/timers.dat" );
    for( auto timer : m_timerregistry )
    {
        timer->Save( file );
    }
}

void Game::LoadTimers()
{
    std::ifstream file( "data/timers.dat" );
    while( file.good() )
    {
        TimedAction* timer = new TimedAction();
        timer->Load( file );
        timer->Hook();
        m_timerregistry.push( timer );
    }
}
```

This means:
- If a player is mid-poison when the server crashes, the poison tick timers are saved
- When the server restarts, those timers are restored
- If the player logs back in, they're still poisoned

---

## For Your "Veil of Dusk" World

You could use hooks and timed actions for:

1. **Veil Corruption** — corruption meter ticks up, causes status effects
2. **Faction Events** — spawn faction patrols, trigger world events
3. **Day/Night Cycle** — change room descriptions, spawn nocturnal threats
4. **Fracture Instability** — portals unstable, cause damage/teleportation
5. **Curses** — temporary debuffs with countdown timers
6. **Environmental Hazards** — rising tide, spreading blight, storms
7. **Boss Timers** — boss attacks on a schedule
8. **Respawning** — defeated enemies, loot

---

## Summary

| Concept | Purpose | Example |
|---------|---------|---------|
| **Action** | An event that happened or will happen | `poison_tick`, `command`, `destroy_item` |
| **TimedAction** | Action + future execution time | "Execute poison_tick at game time 50000ms" |
| **Priority Queue** | Holds all pending timers, sorted by time | First-in-first-out execution |
| **Hook** | Reference from entity back to timer | Character knows about pending damage ticks |
| **ExecuteLoop** | Main game loop that fires timers | Checks queue every frame, fires ready timers |
| **ClearHooks** | Cleanup when entity is deleted | Cancel poison when character dies |

The system is elegant: **timers are first-class citizens**, with full persistence support, and entities stay aware of their own destiny.

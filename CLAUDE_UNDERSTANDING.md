# Claude's Understanding of AbletonMCP

## Overview

AbletonMCP is a Model Context Protocol (MCP) server that enables Claude AI to directly interact with and control Ableton Live for music production. It creates a bridge between AI and the DAW through a socket-based communication protocol.

## Architecture

The system consists of two main components that communicate via TCP sockets on localhost:9877:

### 1. Ableton Remote Script (`AbletonMCP_Remote_Script/__init__.py`)
- **Location**: Installed in Ableton's MIDI Remote Scripts directory
- **Technology**: Python (compatible with both Python 2 and 3 for Ableton compatibility)
- **Function**:
  - Runs inside Ableton Live as a Control Surface
  - Creates a socket server listening on port 9877
  - Executes commands that modify Ableton's state on the main thread
  - Provides access to Ableton's browser and session data
- **Key Features**:
  - Thread-safe command processing using queues
  - Direct access to Ableton's Live API (`_Framework.ControlSurface`)
  - Browser navigation for instruments, effects, and sounds

### 2. MCP Server (`MCP_Server/server.py`)
- **Technology**: Python 3.10+ using FastMCP framework
- **Function**:
  - Implements MCP protocol to expose Ableton capabilities as tools
  - Connects to the Remote Script via socket client
  - Translates Claude's tool calls into Ableton commands
- **Connection Management**:
  - Persistent connection with automatic reconnection (3 attempts)
  - 15-second timeout for state-modifying operations
  - Graceful error handling and connection validation

## Available Tools (MCP Functions)

### Session Management
- `get_session_info()` - Get tempo, time signature, track count, master track info
- `set_tempo(tempo)` - Change session tempo in BPM
- `start_playback()` / `stop_playback()` - Control transport

### Track Management
- `get_track_info(track_index)` - Get detailed track info including devices, clips, mixer settings
- `create_midi_track(index)` - Create new MIDI track at specified position
- `set_track_name(track_index, name)` - Rename a track

### Clip Operations
- `create_clip(track_index, clip_index, length)` - Create MIDI clip with specified length in beats
- `add_notes_to_clip(track_index, clip_index, notes)` - Add MIDI notes (pitch, start_time, duration, velocity, mute)
- `set_clip_name(track_index, clip_index, name)` - Rename a clip
- `fire_clip(track_index, clip_index)` - Start playing a clip
- `stop_clip(track_index, clip_index)` - Stop a clip

### Browser & Instrument Loading
- `get_browser_tree(category_type)` - Get hierarchical tree of browser categories
  - Categories: 'all', 'instruments', 'sounds', 'drums', 'audio_effects', 'midi_effects'
- `get_browser_items_at_path(path)` - Navigate to specific path and get available items
  - Path format: "category/folder/subfolder" (e.g., "drums/acoustic/kit1")
- `load_instrument_or_effect(track_index, uri)` - Load instrument/effect using its URI
- `load_drum_kit(track_index, rack_uri, kit_path)` - Load drum rack and populate with kit

## Communication Protocol

### Command Format (Client → Ableton)
```json
{
  "type": "command_name",
  "params": {
    "param1": "value1",
    "param2": "value2"
  }
}
```

### Response Format (Ableton → Client)
```json
{
  "status": "success" | "error",
  "result": { /* command-specific data */ },
  "message": "error message if status is error"
}
```

### Threading Model
- **Read operations**: Execute directly on socket thread (safe for querying)
- **Write operations**: Scheduled on Ableton's main thread using `schedule_message()`
- **Response handling**: Uses thread-safe Queue to return results

## Browser Integration Deep Dive

The browser system is critical for understanding how to work with Ableton's instruments and effects:

### Browser Hierarchy
1. **Root Categories**: instruments, sounds, drums, audio_effects, midi_effects
2. **Navigation**: Each item has:
   - `name`: Display name
   - `uri`: Unique identifier for loading
   - `is_folder`: Whether it contains children
   - `is_device`: Whether it's a device/instrument
   - `is_loadable`: Whether it can be loaded onto a track
   - `children`: Sub-items (if folder)

### URI System
- URIs are the primary way to identify and load browser items
- Example: `'query:Synths#Instrument%20Rack:Bass:FileId_5116'`
- The `_find_browser_item_by_uri()` method recursively searches up to 10 levels deep

### Current Browser Capabilities
- Can access Ableton's default library (instruments, sounds, drums, effects)
- Can navigate folder hierarchies by path
- Can load items by URI onto tracks

## How Custom Ableton Components Work Currently

### What Works Now
1. **Discovery**: The browser can access any category visible in Ableton's browser
   - Standard categories: instruments, sounds, drums, audio_effects, midi_effects
   - The system dynamically checks `dir(app.browser)` for other available attributes

2. **Navigation**: Custom packs/libraries appear in the browser hierarchy if they're:
   - Properly installed in Ableton's User Library
   - Indexed by Ableton (visible in Ableton's browser)

3. **Loading**: Any browser item with a valid URI can be loaded, including:
   - Third-party instruments (if they appear in the browser)
   - User-created instrument racks
   - Custom sample libraries
   - Saved presets

### Limitations for Custom Components

#### 1. **Discovery Challenge**
The MCP server doesn't currently have a way to:
- Know what custom libraries are installed
- Understand the semantic meaning of custom instruments
- Map musical styles/genres to appropriate custom instruments

#### 2. **Context Gap**
Claude currently lacks context about:
- What custom packs the user has installed
- The sonic characteristics of custom instruments
- When to prefer custom components over stock Ableton devices

#### 3. **Path Navigation**
- Paths must be known in advance
- No semantic search (e.g., "find me a dark ambient pad")
- No fuzzy matching on instrument names

## How to Improve Custom Component Understanding

### Approach 1: Enhanced Tool Descriptions (Immediate)
**Implementation**: Add rich descriptions to MCP tool definitions
```python
@mcp.tool()
def load_instrument_or_effect(ctx: Context, track_index: int, uri: str) -> str:
    """
    Load an instrument or effect onto a track using its URI.

    CUSTOM COMPONENTS:
    - Users may have custom libraries installed (e.g., Serum, Omnisphere, custom racks)
    - Always check available browser items first using get_browser_tree()
    - Ask the user about their custom instruments if unsure
    - Common custom locations: instruments/user library/[pack name]

    Parameters:
    - track_index: The index of the track to load the instrument on
    - uri: The URI of the instrument or effect to load
    """
```

**Benefit**: Claude gets better context directly in the tool definition

### Approach 2: User Library Inventory (Recommended)
**Implementation**: Add a new tool to catalog custom components
```python
@mcp.tool()
def get_user_library_inventory(ctx: Context) -> str:
    """
    Get a complete inventory of user-installed instruments, packs, and custom devices.

    Returns:
    - List of third-party instruments
    - Custom instrument racks
    - User sample libraries
    - Metadata about each item (if available)
    """
```

**How it works**:
1. On first call, scan the browser hierarchy deeply
2. Identify non-stock Ableton items (in "User Library" or third-party sections)
3. Cache this inventory
4. Return structured data about available custom components

**Benefit**: Claude can see what's available before making decisions

### Approach 3: Semantic Tagging System
**Implementation**: Create a configuration file for custom components
```json
{
  "custom_instruments": {
    "serum": {
      "uri": "query:Instruments#User%20Library:Serum:FileId_123",
      "type": "synthesizer",
      "styles": ["edm", "dubstep", "future bass"],
      "characteristics": ["wavetable", "modern", "aggressive"],
      "use_cases": ["leads", "basses", "pads"]
    },
    "my_lofi_rack": {
      "uri": "query:Instruments#User%20Library:Custom%20Racks:LoFi%20Keys:FileId_456",
      "type": "instrument_rack",
      "styles": ["lofi", "hip-hop", "chillhop"],
      "characteristics": ["vintage", "warm", "vinyl"],
      "use_cases": ["chords", "melodies"]
    }
  }
}
```

**How it works**:
1. User creates a config file describing their custom instruments
2. MCP server loads this on startup
3. New tool: `get_custom_component_recommendations(style, use_case)`
4. Claude can query: "what instruments do I have for lofi hip-hop beats?"

**Benefit**: Provides semantic understanding of when to use custom components

### Approach 4: Prompt Engineering via MCP Resources
**Implementation**: Use MCP resources to provide context
```python
@mcp.resource("user-library://inventory")
def get_library_resource():
    """Returns user's custom library inventory as a resource"""
    return scan_user_library()

@mcp.resource("user-library://guide")
def get_usage_guide():
    """Returns user's guide for when to use their custom instruments"""
    return load_user_guide()
```

**Benefit**: Claude automatically receives this context in every conversation

### Approach 5: Interactive Discovery Flow
**Implementation**: Multi-step workflow for new users
1. Claude asks: "Would you like me to scan your Ableton library?"
2. Run deep browser scan to discover all available items
3. Claude asks follow-up questions:
   - "I found Serum. What styles do you use it for?"
   - "I see a 'My Epic Bass' rack. Tell me about it?"
4. Build a knowledge base from user responses
5. Save for future sessions

**Benefit**: Personalized understanding without manual config files

## Recommended Implementation Strategy

### Phase 1: Quick Wins (No Code Changes)
1. **Better prompting**: When users ask for music creation, Claude should:
   - First call `get_browser_tree()` to see what's available
   - Ask users about their preferred custom instruments
   - Take notes in conversation about user preferences

2. **Documentation**: Add examples in README showing:
   - How to find custom instrument URIs
   - Examples with popular third-party plugins
   - Workflow for discovering and using custom components

### Phase 2: Enhanced Discovery (New Tools)
1. Implement `get_user_library_inventory()` tool
2. Add deep scanning with caching
3. Return structured data about custom vs. stock items

### Phase 3: Semantic Layer (Configuration)
1. Add config file support for custom component metadata
2. Implement recommendation tools
3. Create MCP resources for library context

### Phase 4: Intelligence (ML/Heuristics)
1. Learn from user corrections (when Claude picks wrong instrument)
2. Build usage patterns (user always uses Serum for bass)
3. Suggest instruments based on:
   - Musical style mentioned
   - Previous successful sessions
   - Instrument characteristics needed

## Example Workflow with Custom Components

### Current Workflow (Limited)
```
User: "Create a dubstep track with my Serum bass"
Claude:
1. create_midi_track()
2. ??? (doesn't know where Serum is)
3. Asks user for help
```

### Improved Workflow (With Inventory)
```
User: "Create a dubstep track"
Claude:
1. get_user_library_inventory()
   → Sees: Serum, Massive, custom dubstep bass rack
2. Recognizes dubstep → needs heavy bass
3. Knows Serum is good for dubstep (from tool description or config)
4. create_midi_track(index=0)
5. set_track_name(0, "Dubstep Bass")
6. load_instrument_or_effect(0, "uri:...serum...")
7. create_clip(0, 0, 8)
8. add_notes_to_clip(0, 0, [bass_notes])
```

## Technical Considerations

### Performance
- Deep browser scans can be slow (recursive traversal)
- Should cache browser inventory
- Invalidate cache when Ableton restarts or user adds new packs

### Reliability
- Browser access requires Ableton to be fully loaded
- Some items may not have stable URIs across Ableton versions
- Third-party plugins need to be loaded in Ableton first

### User Experience
- Don't assume users know URIs (provide discovery tools)
- Guide users through setup (scan library on first use)
- Allow override (user can always specify exact instrument)

## Conclusion

The MCP server has solid foundations for browser integration and can technically access custom components if they appear in Ableton's browser. The main gap is **semantic understanding** - knowing what custom instruments exist and when to use them.

The recommended path forward is:
1. **Immediate**: Better prompting and documentation
2. **Short-term**: Add inventory/discovery tools
3. **Long-term**: Implement semantic layer with configuration or learning

This would transform the MCP from "can load instruments by URI if told" to "understands the user's custom library and makes intelligent suggestions."

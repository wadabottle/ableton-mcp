# Library Inventory Tool - Usage Guide

## Overview

The `get_user_library_inventory()` tool provides deep insight into your Ableton Live library, enabling Claude to make intelligent decisions about which instruments and effects to use based on what you actually have installed.

## How It Works

### Scanning Process

1. **Browser Traversal**: The tool recursively scans all browser categories:
   - Instruments
   - Sounds
   - Drums
   - Audio Effects
   - MIDI Effects

2. **Depth & Limits**:
   - Maximum recursion depth: 8 levels
   - Maximum children per folder: 50 items
   - These limits prevent timeouts while scanning large libraries

3. **Custom Detection**: Items are flagged as "custom" if they:
   - Exist in "User Library" paths
   - Contain known third-party plugin names (Serum, Massive, Vital, etc.)
   - Have custom naming patterns ("My...", "Custom...")

4. **File-Based Caching**: Results are cached to disk for performance
   - **Cache Location**: `~/.ableton-mcp/cache/library_inventory.json`
   - **First scan**: 10-30 seconds (scans Ableton's browser)
   - **Subsequent scans**: Instant (loads from file)
   - **Cache expiration**: 7 days
   - **Persistence**: Survives restarts of Claude, Ableton, and system reboots
   - **Two-tier cache**: In-memory (Ableton Remote Script) + File (MCP Server)

## MCP Tool Interface

### get_user_library_inventory()

**Function Signature:**
```python
get_user_library_inventory(force_refresh: bool = False) -> str
```

**Parameters:**
- `force_refresh` (bool, optional): If True, invalidates cache and rescans library. Default: False

**Returns:**
Formatted string containing:
- Summary statistics (total items, custom items, stock items)
- Custom items organized by category and folder
- URIs for each loadable instrument
- Timestamp of last scan
- Cache file location and age

**Caching Behavior:**
- If `force_refresh=False` (default):
  1. Try to load from file cache (`~/.ableton-mcp/cache/library_inventory.json`)
  2. If cache exists and is < 7 days old, return cached data (instant)
  3. If cache doesn't exist or is too old, scan from Ableton and save to file
- If `force_refresh=True`:
  1. Skip file cache check
  2. Scan from Ableton directly
  3. Update file cache with new data

### clear_library_inventory_cache()

**Function Signature:**
```python
clear_library_inventory_cache() -> str
```

**Parameters:**
None

**Returns:**
Success message or error

**Use Cases:**
- Just installed new instruments/packs
- Want to force a fresh scan immediately
- Troubleshooting cache issues
- Manual cache management

## Data Structure

### File Cache Format

The cache file (`~/.ableton-mcp/cache/library_inventory.json`) uses this structure:

```json
{
  "cached_at": 1234567890.123,
  "cached_at_readable": "2024-01-15T10:30:00",
  "version": "1.0",
  "inventory": {
    "total_items": 1250,
    "custom_items": 75,
    "stock_items": 1175,
    "scan_timestamp": 1234567890.123,
    "categories": {
      ...
    }
  }
}
```

**Top-level metadata:**
- `cached_at`: Unix timestamp when file was saved
- `cached_at_readable`: Human-readable ISO timestamp
- `version`: Cache format version (for future compatibility)
- `inventory`: The actual inventory data

### Inventory Data Format (JSON)
```json
{
  "total_items": 1250,
  "custom_items": 75,
  "stock_items": 1175,
  "scan_timestamp": 1234567890.123,
  "categories": {
    "instruments": {
      "total": 500,
      "custom": 35,
      "stock": 465,
      "custom_items": [
        {
          "name": "Serum Bass",
          "path": "instruments/User Library/Serum/Bass",
          "uri": "query:Instruments#User%20Library:Serum:Bass:FileId_12345",
          "is_folder": false,
          "is_device": true,
          "is_loadable": true,
          "is_custom": true,
          "category": "instruments",
          "depth": 4
        }
      ],
      "stock_items": [...]
    },
    "drums": {...},
    "audio_effects": {...},
    "midi_effects": {...}
  }
}
```

## Usage Patterns

### Pattern 1: Discovery
Ask Claude to scan and report what's available:
```
"Scan my Ableton library and tell me what custom instruments I have"
```

### Pattern 2: Contextual Creation
Scan first, then create music with that context:
```
"Scan my library, then create a synthwave track using my custom instruments"
```

### Pattern 3: Specific Instrument Search
Find specific instruments:
```
"Do I have any custom bass instruments? Show me what's available."
```

### Pattern 4: Force Refresh
After installing new packs:
```
"Rescan my library to pick up the new instruments I just installed"
```

### Pattern 5: Clear Cache
Manually clear the cache file:
```
"Clear my library cache"
```

### Pattern 6: Check Cache Status
View cache information:
```
"Show me my library inventory"
```
(Output includes cache file location and age)

## Custom Detection Logic

### Path-Based Detection
Items are marked custom if their path contains:
- "user library"
- "user"
- "packs"
- "downloaded"
- "samples"
- "my "
- "custom"

### Name-Based Detection (Third-Party Plugins)
Items are marked custom if their name contains:
- serum, massive, omnisphere, sylenth, spire
- nexus, kontakt, reaktor, pigments, vital
- phase plant, diva, repro, u-he, arturia
- native instruments, spectrasonics, lennar
- vst, au, rack, preset

### Adding Custom Keywords
To extend detection, edit the `is_custom_item()` function in:
```
AbletonMCP_Remote_Script/__init__.py
```

Add keywords to either:
- `custom_indicators` list (path-based)
- `third_party_keywords` list (name-based)

## Performance Considerations

### Initial Scan Time (First Time Only)
- Small library (< 500 items): 5-10 seconds
- Medium library (500-2000 items): 10-20 seconds
- Large library (2000+ items): 20-30 seconds

### Subsequent Access (File Cache)
- **Load time**: < 100ms (instant)
- **No Ableton connection needed** (can work even if Ableton isn't running)
- **Cross-session persistence**: Cache survives all restarts

### Optimization Strategies
1. **Depth limiting**: Max depth of 8 prevents excessive recursion
2. **Child limiting**: Max 50 children per folder prevents timeout
3. **Two-tier caching**:
   - In-memory (Ableton Remote Script): Fast, volatile
   - File-based (MCP Server): Persistent, cross-session
4. **Selective scanning**: Only scans main browser categories
5. **Cache expiration**: 7-day limit prevents stale data

### Storage Requirements
- **Memory**: 100KB - 2MB (in-memory cache in Ableton)
- **Disk**: 100KB - 2MB (file cache at `~/.ableton-mcp/cache/`)
- **Location**: User home directory (cross-platform)

### Cache Performance
| Scenario | Time | Source |
|----------|------|--------|
| First scan (no cache) | 10-30s | Ableton browser |
| Cached (same session) | < 10ms | Ableton memory |
| Cached (new session) | < 100ms | File system |
| Cache expired (> 7 days) | 10-30s | Ableton browser |
| force_refresh=True | 10-30s | Ableton browser |

## Troubleshooting

### "Browser is not available"
**Cause**: Ableton not fully loaded or browser isn't initialized
**Solution**: Wait for Ableton to fully load, then try again

### Scan Times Out
**Cause**: Extremely large library or slow system
**Solution**:
- Close other applications
- Reduce library size
- Edit depth/child limits in code if needed

### Custom Items Not Detected
**Cause**: Items don't match detection patterns
**Solution**:
- Check item paths (should be in User Library)
- Add custom keywords to detection logic
- Verify items are visible in Ableton's browser

### Cache Not Invalidating
**Cause**: File cache persists until expired (7 days) or manually cleared
**Solution**:
- Use `force_refresh=True` parameter
- Call `clear_library_inventory_cache()`
- Manually delete `~/.ableton-mcp/cache/library_inventory.json`

### File Cache Won't Save
**Cause**: Permission issues or disk full
**Solution**:
- Check write permissions for `~/.ableton-mcp/cache/`
- Ensure disk has space (cache is typically < 2MB)
- Check logs for file system errors

### Cache Loading But Data is Stale
**Cause**: Library changed but cache hasn't expired
**Solution**: Use `force_refresh=True` to update cache immediately

## Integration with Other Tools

### Loading Instruments
Use URIs from inventory with:
```python
load_instrument_or_effect(track_index, uri)
```

Example:
```
User: "Load my Serum bass on track 1"

Claude:
1. get_user_library_inventory()
2. Find Serum bass URI in inventory
3. load_instrument_or_effect(0, "query:Instruments#User%20Library:Serum:Bass:FileId_12345")
```

### Browser Navigation
URIs can also be used to navigate to parent folders:
```python
get_browser_items_at_path("instruments/User Library/Serum")
```

## Future Enhancements

Potential improvements to consider:

1. **Semantic Tags**: Allow users to tag instruments with styles/moods
2. **Usage Analytics**: Track which instruments are used most often
3. **Smart Recommendations**: ML-based suggestions for musical styles
4. **Disk Caching**: Persist inventory between sessions
5. **Delta Updates**: Only scan changed items instead of full rescan
6. **Parallel Scanning**: Use threading for faster scans
7. **Audio Preview**: Include audio characteristics (if available)

## API Reference

### Remote Script Method
```python
def _get_user_library_inventory(self, force_refresh=False):
    """
    Scan and return an inventory of the user's library.

    Args:
        force_refresh (bool): If True, invalidate cache and rescan

    Returns:
        dict: Inventory data structure

    Raises:
        RuntimeError: If browser is unavailable
        Exception: For other scanning errors
    """
```

### MCP Tool
```python
@mcp.tool()
def get_user_library_inventory(ctx: Context, force_refresh: bool = False) -> str:
    """
    Get comprehensive inventory of user's Ableton library.

    Parameters:
    - force_refresh: Invalidate cache and rescan (default: False)

    Returns:
    - Formatted string with inventory data
    """
```

## Examples

### Example 1: First-Time Scan
```
User: "What custom instruments do I have?"

Claude calls: get_user_library_inventory(force_refresh=False)

Output:
User Library Inventory (Total: 1250, Custom: 75, Stock: 1175)

INSTRUMENTS - 35 custom items:

  Serum:
    - Serum Bass 1
      URI: query:Instruments#User%20Library:Serum:Bass1:FileId_123
    - Serum Lead Pluck
      URI: query:Instruments#User%20Library:Serum:LeadPluck:FileId_124

  User Library:
    - My Lofi Keys Rack
      URI: query:Instruments#User%20Library:MyLofiKeysRack:FileId_125
...
```

### Example 2: Using Inventory for Track Creation
```
User: "Create a dubstep track using my custom instruments"

Claude:
1. get_user_library_inventory()
   → Finds: Serum (bass), Massive (leads), Custom Wobble Rack
2. create_midi_track()
3. set_track_name(0, "Dubstep Bass")
4. load_instrument_or_effect(0, "query:...Serum:Bass...")
5. create_clip(0, 0, 8)
6. add_notes_to_clip(0, 0, [heavy_bass_notes])
```

## Contributing

To improve the inventory system:
1. Add more third-party plugin keywords to detection
2. Optimize scanning performance
3. Add metadata extraction (if available from browser API)
4. Improve custom vs. stock classification accuracy

See `CLAUDE_UNDERSTANDING.md` for architectural details.

# PlayerVisibility Plugin - Copilot Instructions

## Repository Overview
This repository contains a SourcePawn plugin for SourceMod that implements a player visibility system. The plugin dynamically adjusts player transparency based on proximity to other players, making players fade when they get too close to each other. This is particularly useful in crowded servers to improve visibility and gameplay experience.

**Plugin Details:**
- **Name**: PlayerVisibility
- **Version**: 1.4.4
- **Authors**: BotoX, maxime1907
- **Purpose**: Fades players away when you get close to them using dynamic transparency
- **Integration**: Works with Zombie Reloaded plugin (zombies are excluded from fading)

## Technical Environment

### Build System
- **Primary Tool**: SourceKnight (Python-based build system)
- **Config File**: `sourceknight.yaml` in repository root
- **Dependencies**: Automatically downloaded by SourceKnight
  - SourceMod 1.11.0-git6934 (minimum version)
  - Zombie Reloaded plugin includes
- **Output**: Compiled `.smx` files in `/addons/sourcemod/plugins`

### Build Commands
```bash
# Primary method: Use GitHub Actions CI (recommended)
# The repository is configured to build automatically via CI

# Alternative: Manual SourceKnight installation (if needed locally)
pip install sourceknight
sourceknight build

# Output will be in .sourceknight/package/addons/sourcemod/plugins/

# Note: Manual builds may require dependency setup
# CI environment handles all dependencies automatically
```

### CI/CD Pipeline
- **Platform**: GitHub Actions (`.github/workflows/ci.yml`)
- **Triggers**: Push, pull request, manual dispatch
- **Workflow**: Build → Package → Release (automatic for main/master branch)
- **Artifacts**: Automatically creates releases with `.tar.gz` packages

## Repository Structure
```
/
├── .github/
│   ├── workflows/ci.yml          # CI/CD pipeline
│   └── copilot-instructions.md   # This file
├── addons/sourcemod/scripting/
│   └── PlayerVisibility.sp       # Main plugin source
├── sourceknight.yaml            # Build configuration
└── .gitignore                   # Excludes build artifacts, .smx files
```

## Code Architecture & Patterns

### Core Components
1. **PlayerData Struct**: Manages per-client state (enabled, bot status, alpha value)
2. **DHooks Integration**: Hooks entity AcceptInput for transparency control
3. **ConVar System**: Runtime configuration through server console variables
4. **Batch Processing**: Performance-optimized frame updates with configurable rate
5. **Event Integration**: Hooks player spawn and zombie infection events

### Key Variables & Configuration
```sourcepawn
// ConVars (runtime configurable)
sm_pvis_updaterate        // Players to update per frame (default: 3)
sm_pvis_maxdistance       // Distance threshold for fading (default: 100.0)
sm_pvis_minfactor         // Minimum alpha factor (default: 0.75)
sm_pvis_minalpha          // Minimum alpha value (default: 75.0)
sm_pvis_minplayers        // Min players in range to enable fading (default: 3)
sm_pvis_minplayers_enable // Min total players to enable plugin (default: 40)
```

### Performance Optimizations
- **Batched Updates**: Only processes `g_iUpdateRate` players per frame
- **Early Skipping**: Bypasses bots and disabled players immediately
- **Static Variables**: Reuses position vectors to reduce allocations
- **Configurable Thresholds**: Allows fine-tuning based on server capacity

## Code Style & Standards

### Naming Conventions (Existing Pattern)
- **Global Variables**: `g_` prefix (e.g., `g_fMaxDistance`, `g_playerData`)
- **ConVars**: `g_CVar_` prefix (e.g., `g_CVar_UpdateRate`)
- **Function Names**: PascalCase (e.g., `OnPluginStart`, `CheckClientCount`)
- **Local Variables**: camelCase (e.g., `client`, `fDistance`)
- **Constants/Enums**: PascalCase

### Code Structure Requirements
```sourcepawn
#pragma semicolon 1
#pragma newdecls required

// Standard includes pattern
#include <sourcemod>
#include <sdkhooks>
#include <sdktools>
#include <dhooks>
#undef REQUIRE_PLUGIN
#include <zombiereloaded>  // Optional dependency
#define REQUIRE_PLUGIN
```

### Memory Management Patterns
- Use `CloseHandle()` immediately after GameConfig usage
- DHooks are automatically cleaned up on plugin unload
- Reset player data structures in `OnClientDisconnect`
- Call `ResetTransparency()` in `OnPluginEnd()` for cleanup

### Error Handling & Validation
- Always validate client indices: `if (client < 1 || client > MAXPLAYERS)`
- Check `IsClientInGame()` before client operations
- Use `DHookIsNullParam()` for null parameter validation
- Handle gameconfig loading failures with `SetFailState()`

## Plugin-Specific Implementation Details

### Transparency Control System
```sourcepawn
// Custom transparency function (preferred over SetEntityRenderColor)
ToolsSetEntityAlpha(client, alpha)  // Handles RENDER_NORMAL vs RENDER_TRANSCOLOR

// State management per client
PlayerData.enabled   // Whether client can be faded
PlayerData.bot      // Skip processing for bots
PlayerData.alpha    // Current alpha value (avoid redundant updates)
```

### Integration Points
1. **Zombie Reloaded Events**:
   - `ZR_OnClientInfected`: Disable fading for zombies
   - `ZR_OnClientHumanPost`: Re-enable fading for humans

2. **Entity Input Hooking**:
   - Monitors `addoutput` commands for rendermode/renderfx changes
   - Handles `alpha` input commands from other plugins
   - Prevents conflicts with external transparency systems

### Distance Calculation Logic
```sourcepawn
// Multi-player proximity algorithm
for each player in range:
    factor = distance / maxDistance
    if factor < minFactor: factor = minFactor
    alpha *= factor
    
if players_in_range < minPlayers: alpha = 255 (no fading)
if alpha < minAlpha: alpha = minAlpha
```

### Key Algorithmic Considerations
1. **Multiplicative Alpha Reduction**: Each nearby player reduces alpha multiplicatively
2. **Distance Normalization**: Uses configurable maximum distance for factor calculation
3. **Minimum Thresholds**: Prevents complete invisibility and excessive fading
4. **Player Count Gating**: Requires minimum nearby players before fading activates
5. **Performance Batching**: Processes limited players per frame to maintain server performance

### Frame Processing Logic
```sourcepawn
// Cyclic processing pattern
static int client = 0;
for (int i = 0; i < updateRate; i++) {
    if (client == MAXPLAYERS) client = 0;  // Reset cycle
    client++;
    // Process client...
}
```

## Testing & Validation

## Testing & Validation

### Build Validation
```bash
# Primary: Use CI for automated building and validation
# Push changes to trigger GitHub Actions workflow

# Alternative: Local development validation
# Method 1: Use SourceKnight (if successfully installed)
sourceknight build

# Method 2: Manual SourceMod compiler (if available)
# Requires SourceMod development environment
spcomp -o output.smx PlayerVisibility.sp

# Verify compilation success
# Check for warnings/errors in output
# Ensure .smx file generation
```

### Runtime Testing Scenarios
1. **Basic Functionality**: Multiple players in close proximity should fade
2. **Performance**: Monitor server performance with high player counts
3. **Integration**: Test with Zombie Reloaded (zombies shouldn't fade)
4. **Edge Cases**: 
   - Plugin disable/enable with `sm_pvis_minplayers_enable -1/0`
   - External plugins setting player transparency
   - Player spawning/respawning behavior

### Configuration Testing
- Test all ConVar changes during runtime
- Verify immediate effect of `sm_pvis_minplayers_enable` changes
- Test extreme values (very low/high distances, factors)

## Common Modification Patterns

### Adding New ConVars
1. Declare ConVar handle: `ConVar g_CVar_NewSetting;`
2. Create in `OnPluginStart()`: `CreateConVar("sm_pvis_newsetting", ...)`
3. Cache value in global variable
4. Add change hook: `g_CVar_NewSetting.AddChangeHook(OnConVarChanged)`
5. Handle in `OnConVarChanged()` function

### Performance Optimizations
- Modify `g_iUpdateRate` for batch size adjustment
- Add distance-based early termination in proximity checks
- Consider using squared distance for performance: `GetVectorDistance(..., false)`
- Cache frequently accessed client data

### Integration Extensions
- Add new game event hooks in `OnPluginStart()`
- Extend `PlayerData` struct for additional state tracking
- Add new DHook parameters for additional entity inputs
- Consider methodmap patterns for complex data structures

## Dependencies & Compatibility

### Required Dependencies
- SourceMod 1.12+ (development), 1.11+ (runtime compatibility)
- DHooks extension (for entity input hooking)
- SDKTools (for entity manipulation)
- SDKHooks (for entity events)

### Optional Dependencies
- Zombie Reloaded plugin (graceful degradation if not present)

### Game Compatibility
- Designed for Source engine games
- Tested with Counter-Strike: Source / Counter-Strike: Global Offensive
- Should work with any Source game supported by SourceMod

## Troubleshooting Common Issues

### Compilation Issues
- **SourceKnight installation fails**: Use CI pipeline for building instead of local compilation
- **Missing include files**: Ensure `zombiereloaded.inc` is available in include path
- **SourceMod version mismatch**: Verify compatibility with target SourceMod version
- **DHooks dependency**: Ensure DHooks extension is loaded on target server

### Runtime Issues
- **Players not fading**: 
  - Check `sm_pvis_minplayers_enable` setting (default: 40 players required)
  - Set to 0 for always enabled, -1 to disable plugin
  - Verify sufficient players within `sm_pvis_maxdistance`
- **Performance problems**: 
  - Reduce `sm_pvis_updaterate` value (lower = fewer players processed per frame)
  - Monitor server tick rate and adjust accordingly
- **Conflicts with other plugins**: 
  - Check AcceptInput hook conflicts
  - Verify no external alpha/rendermode overrides
  - Review plugin load order

### Build System Issues
- **SourceKnight installation issues**: 
  - Use GitHub Actions CI instead of local builds
  - Dependency conflicts may occur with newer Python versions
- **Missing dependencies in CI**: 
  - Check sourceknight.yaml configuration
  - Verify dependency URLs are accessible
  - Review GitHub Actions workflow logs
- **CI build failures**: 
  - Check Actions tab for detailed error messages
  - Verify branch permissions and secrets configuration

### Development Workflow
```bash
# Recommended development cycle:
1. Edit PlayerVisibility.sp locally
2. Commit and push changes
3. Monitor GitHub Actions for build status
4. Download artifacts from successful builds
5. Test on development server
```
# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**SillyTavern-MessageSummarize** is a SillyTavern extension that provides an alternative memory system by summarizing individual messages rather than entire conversations. It injects summaries into the LLM context as "short-term" and "long-term" memory.

**Key Value Proposition:**
- Individual message summarization (vs. bulk conversation summarization)
- Prevents summary degradation over time (summaries don't depend on LLM decisions)
- Automatic memory rotation with configurable limits
- User control over long-term memory preservation

## Extension Architecture

### Core Structure
This is a **single-file extension** (4587 lines in `index.js`) that integrates with SillyTavern's extension API. The main components are:

1. **Settings Management** - Profile-based configuration system
2. **Memory System** - Short-term and long-term memory injection
3. **Summarization Engine** - AI-powered message summarization
4. **UI Components** - Configuration interface, message buttons, and editing interfaces
5. **Event Handlers** - ST event listeners for chat changes

### Key Files
- `index.js` - Main extension logic (~4600 lines, all functionality)
- `settings.html` - Configuration UI HTML template
- `style.css` - Extension styling
- `manifest.json` - Extension metadata and entry point
- `locales/` - i18n translation files

### Data Storage

**Extension Settings** (stored in ST's `extension_settings[MODULE_NAME]`):
- Configuration profiles (default settings + user-created profiles)
- Character/chat profile mappings
- Global toggle states

**Chat Metadata** (stored in ST's `chat_metadata[MODULE_NAME]`):
- Per-chat enabled state
- Profile locked to specific chat

**Message Data** (stored on message object at `message.extra[MODULE_NAME]`):
- `memory` - The summary text (without prefill)
- `remember` - Marked for long-term memory
- `exclude` - Force-excluded from memory
- `include` - Current inclusion state ('short', 'long', or null)
- `lagging` - Behind injection threshold
- `edited` - Manually edited
- `error` - Error message from failed summarization
- `reasoning` - Reasoning from reasoning models
- `prefill` - Response prefill used
- `hash` - Hash of message when summarized

## Development Commands

### Testing
No automated tests - manual testing in SillyTavern required.

### Debugging
- Enable "Debug Mode" in extension settings to see console logs
- Use `/qm-debug` slash command to dump context and settings
- Check browser console (F12) for errors
- ST terminal shows backend errors

### Common Operations
```bash
# View extension in SillyTavern
# Extensions are loaded from: [ST]/data/default-user/extensions/
# Or manually installed from: Extensions panel -> Install Extension

# Clear all settings (use with caution)
# In browser console or via slash command:
/qm-hard-reset
```

## Architecture Patterns

### Settings System
The extension uses a **profile-based configuration** where:
- `default_settings` defines the schema
- `global_settings` contains non-profile data (profiles list, mappings)
- Profiles can be locked to characters or specific chats
- Settings are stored in ST's `extension_settings` and auto-saved with debouncing

### Memory Injection Flow
```
1. Message created → Event handler triggered
2. Inclusion criteria checked (check_message_exclusion)
3. Summarization (if needed) via summarize_message()
4. Memory flags updated (update_message_inclusion_flags)
5. Context built (get_short_memory / get_long_memory)
6. Injected via ctx.setExtensionPrompt()
```

### UI Architecture
Three main interfaces implemented as classes:
- `MemoryEditInterface` - Bulk memory management with pagination
- `SummaryPromptEditInterface` - Prompt/macro editing
- Popout system for floating config window

### Event Handling
Heavily relies on SillyTavern's event system:
- `CHARACTER_MESSAGE_RENDERED` - Auto-summarize on new messages
- `MESSAGE_EDITED` - Re-summarize edited messages
- `MESSAGE_SWIPED` - Handle swipe memory
- `CHAT_CHANGED` - Load profiles, refresh state
- Custom `generate_interceptor` to remove summarized messages from context

## Important Implementation Details

### Preset/Profile Switching
When summarizing, the extension temporarily switches completion presets and connection profiles, which **discards unsaved changes**. This is a ST limitation.

### Memory Macros
The extension provides custom macros for ST:
- `{{qm-short-term-memory}}` - Short-term summaries
- `{{qm-long-term-memory}}` - Long-term summaries
- Summary prompt supports custom macros via STScript

### Prompt Construction
Summary prompts use Handlebars templating and support:
- Range macros (e.g., previous 3-10 messages)
- STScript commands
- Regex scripts
- Instruct template wrapping (TC) or separate messages (CC)

### Text Completion vs Chat Completion
The extension handles both ST modes (main_api):
- TC: Uses instruct templates, wraps in system prompts
- CC: Uses message arrays with roles

### Group Chat Support
Special handling for group chats:
- Per-character enable/disable toggles
- Character identification via `original_avatar`
- Profile locking to group ID

## Common Pitfalls

1. **Settings not persisting** - Profiles must be manually saved (click save icon)
2. **Memory not injecting** - Check injection position settings and thresholds
3. **Empty summaries** - Prompt role/format issues with API (read Troubleshooting in README)
4. **Preset switching** - Save presets before summarizing if using custom preset
5. **Context limits** - Token counting uses ST's tokenizer (may vary by API)

## ST Version Requirements

Minimum ST version: **1.13.2**

The extension relies on specific ST features:
- Extension prompt injection system
- `parseReasoningFromString()` for reasoning models
- Event system (various PRs referenced in CHANGELOG)
- Moving UI support for popout window

## Key Dependencies

Imported from SillyTavern core:
- `utils.js` - Utility functions (debounce, token counting, hashing)
- `script.js` - Core ST functions (generateRaw, context, etc.)
- `extensions.js` - Extension settings management
- `preset-manager.js` - Completion preset management
- `instruct-mode.js` - Text completion formatting
- `macros.js` - Macro registration
- `i18n.js` - Translation support

External libraries (provided by ST):
- jQuery - DOM manipulation and event handling
- Handlebars - Template compilation for prompt macros
- select2 - Enhanced select dropdowns
- pagination.js - Table pagination in edit interface

## Slash Commands

All commands use `qm-` prefix (with `qvink-memory-` as alias):
- `/qm-toggle` - Enable/disable memory
- `/qm-summarize [index]` - Summarize message
- `/qm-toggle-remember [index]` - Mark for long-term
- `/qm-get [index|range]` - Get memory text
- `/qm-set <index> [text]` - Set memory text
- See README for complete list

## CSS Customization

Memory display colors controlled by CSS variables:
- `--qm-short` - Short-term memory (green)
- `--qm-long` - Long-term memory (blue)
- `--qm-old` - Out of context (red)
- `--qm-default` - Not included (light grey)
- `--qm-excluded` - Force-excluded (dark grey)
- `--qm-message-removed` - Removed from context (light grey)

## Code Style Notes

- Uses jQuery conventions (`$variable` for jQuery objects)
- Heavy use of closures and inline function definitions
- Settings use string keys with get/set wrappers
- Debug logging gated behind `debug_mode` setting
- Async/await for API calls and delays
- Debouncing for performance (settings saves, UI updates)

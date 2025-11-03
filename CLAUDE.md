# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Cargo workspace containing two binaries:

1. **gia** - Command-line tool that sends prompts to Google's Gemini API and returns AI responses. Supports multiple input sources (command line, clipboard, stdin, files) and output destinations (stdout, clipboard). Supports multimodal interactions with automatic detection of media files (JPEG, PNG, WebP, HEIC, PDF, MP3, MP4, etc.). Also supports local Ollama models.

2. **giagui** - GUI wrapper for gia using the egui framework. Must have gia installed and available in PATH.

## Development Commands

### Build and Test

```bash
# Build both binaries
cargo build --release      # Production build
cargo build                # Development build

# Build specific binaries
cargo build --release -p gia      # CLI only
cargo build --release -p giagui   # GUI only

# Test
cargo test                 # Run tests
cargo test -- --nocapture  # Run tests with output
```

### Code Quality

```bash
cargo clippy --fix --allow-dirty  # Fix linting issues
cargo fmt                         # Format code
cargo check                      # Check compilation without building
```

### Running

```bash
# CLI Development
cargo run -p gia -- "your prompt here"

# GUI Development
cargo run -p giagui                    # Normal GUI mode
cargo run -p giagui -- --spinner       # Spinner-only mode (runs until killed)
cargo run -p giagui -- -s              # Spinner-only mode (short flag)

# After building
./target/release/gia "your prompt here"
./target/release/giagui                # Normal GUI
./target/release/giagui --spinner      # Spinner-only mode

# Using Ollama (local, requires Ollama running on localhost:11434)
cargo run -- -m "ollama::llama3.2" "your prompt here"

# Resume conversations
cargo run -- --resume "continue previous conversation"
cargo run -- --resume abc123 "continue specific conversation"
cargo run -- --list-conversations  # List all saved conversations

# Image analysis (auto-detected as media files)
cargo run -- "What do you see in this image?" -f photo.jpg
cargo run -- "Compare these images" -f img1.jpg -f img2.png

# Text file input
cargo run -- "Summarize these documents" -f document1.txt -f document2.txt
cargo run -- "What are the differences between these files?" -f old.txt -f new.txt

# Combining multiple input sources (auto-detection of media vs text)
cargo run -- "Analyze this code and documentation" -f README.md -f main.rs -f diagram.png

# Clipboard image analysis (copy an image to clipboard first)
cargo run -- "What do you see in this image?" -c

# Text-to-speech output
cargo run -- "Tell me a short story" --tts en-US
cargo run -- "What is the weather today?" -T en-US
cargo run -- "Erzähl mir eine Geschichte" --tts de-DE
cargo run -- "Erzähl mir eine Geschichte" --tts          # Uses default: de-DE
cargo run -- "Tell me a joke" -T                         # Uses default: de-DE

# Transcribe-only mode (no conversation history saved)
cargo run -- --record-audio --role EN --no-save         # English transcription only
cargo run -- --record-audio --role DE --no-save         # German transcription only
cargo run -- "Transcribe this" --record-audio --no-save # Custom prompt transcription
```

### Environment Setup

Set API key(s):

```bash
# Single key
export GEMINI_API_KEY="your_api_key_here"

# Multiple keys for fallback (pipe-separated)
export GEMINI_API_KEY="key1|key2|key3"
```

Windows:

```cmd
set GEMINI_API_KEY=your_api_key_here
set GEMINI_API_KEY=key1|key2|key3
```

For Ollama: Install from <https://ollama.ai> and run `ollama serve`

### Default Model Configuration

Set default AI model:

```bash
# Set default model globally
export GIA_DEFAULT_MODEL="gemini-2.5-pro"

# Use Ollama model as default
export GIA_DEFAULT_MODEL="ollama::llama3.2"

# Windows
set GIA_DEFAULT_MODEL=gemini-2.5-pro
```

### Logging

```bash
# Console logging (requires RUST_LOG to be set)
RUST_LOG=debug cargo run -- "test"  # Debug logging to stderr
RUST_LOG=info cargo run -- "test"   # Info logging to stderr
RUST_LOG=error cargo run -- "test"  # Error logging only to stderr

# File logging (per-conversation log files in ~/.gia/conversations/)
GIA_LOG_TO_FILE=1 cargo run -- "test"  # Enable file logging only
GIA_LOG_TO_FILE=1 RUST_LOG=info cargo run -- "test"  # Enable both file and console logging

# Without either variable, no logs are produced (clean stdout/stderr)
cargo run -- "test"  # No logging output
```

**Logging Behavior:**

- **Console logging**: Only enabled when `RUST_LOG` is set (outputs to stderr)
- **File logging**: Only enabled when `GIA_LOG_TO_FILE` is set (creates `<conversation-id>.log` in `~/.gia/conversations/`)
- **Both can be enabled simultaneously**: Set both environment variables
- **Clean output by default**: Without these variables, stdout remains clean for piping

## Architecture

### Workspace Structure

This is a Cargo workspace with shared dependencies and build configuration:

- **Workspace root**: Contains workspace `Cargo.toml` and shared resources (`icons/`)
- **gia/**: CLI binary crate
- **giagui/**: GUI binary crate
- Both crates share the same `build.rs` for git-based versioning
- Shared resources: `~/.gia/roles` and `~/.gia/tasks` (runtime, not in repo)

### Module Structure (gia CLI)

- `gia/src/main.rs` - CLI argument parsing, main application flow
- `gia/src/gemini.rs` - Gemini API client with rate limit fallback logic
- `gia/src/ollama.rs` - Ollama API client for local LLM support
- `gia/src/provider.rs` - Provider abstraction and factory
- `gia/src/api_key.rs` - API key management (multiple keys, validation, fallback)
- `gia/src/clipboard.rs` - Clipboard operations using arboard
- `gia/src/conversation.rs` - Conversation management with persistent storage
- `gia/src/logging.rs` - Structured logging to stderr

### Module Structure (giagui GUI)

- `giagui/src/main.rs` - Single-file egui application
- **Args struct**: Command-line argument parsing with clap
- **SpinnerApp struct**: Minimal spinner-only display mode
- **GiaApp struct**: Full GUI application state
- **Command execution**: Spawns `gia` CLI process
- **Icons**: References `../icons/gia.png` from workspace root
- **Spinner mode**: Launched with `--spinner` or `-s` flag, displays only animated spinner until process is killed

### Key Design Patterns

**Multi-API Key Support**: The application supports pipe-separated API keys (`key1|key2|key3`) and automatically falls back to alternative keys when encountering 429 rate limit errors. This is handled in `gemini.rs` with cooperation from `api_key.rs`.

**Input Sources**: Seven input sources can be combined:

1. Command line arguments (primary prompt)
2. Audio recording (with `-a, --record-audio` flag) - Native recording using cpal + ogg-opus
3. Clipboard content (with `-c` flag) - supports both text and images
4. Stdin content (automatically detected when available)
5. File content (with `-f` flag) - automatically detects media files vs text files, can be used multiple times, supports recursive directory processing
6. Roles & Tasks (with `-t, --role` flag) - Load from `~/.gia/roles/` or `~/.gia/tasks/`
7. Resume conversations (with `-r, --resume` flag) - Continue with conversation history

**Output Destinations**: Four output options:

1. Stdout (default)
2. Clipboard (with `-o` flag)
3. Browser preview (with `-b` flag)
4. Text-to-speech (with `-T` or `--tts` flag)

**Conversation Management**: Persistent conversation storage allowing users to resume previous conversations:

- Local JSON-based storage in `~/.gia/conversations/`
- Automatic conversation history inclusion in prompts
- Context window management with automatic truncation
- Support for resuming latest or specific conversations

**Error Handling**: Comprehensive error handling with user-friendly messages for common issues like missing API keys, authentication failures, and rate limits.

### API Key Management

The `api_key.rs` module handles:

- Loading keys from `GEMINI_API_KEY` environment variable
- Supporting both single keys and pipe-separated multiple keys
- Validation of Google API key format (39 chars, starts with "AIza")
- Random key selection for initial requests
- Alternative key selection for fallback scenarios
- User guidance when keys are missing or invalid

**API Key Fallback Algorithm** (implemented in `gemini.rs`):

1. **Initialization**: Read all API keys from `GEMINI_API_KEY` environment variable (pipe-separated: `key1|key2|key3`)
2. **Random Start**: Randomly select one key as the starting key
3. **API Request**: Make request using current key
4. **On 429 Rate Limit Error**:
   - Log the rate limit hit
   - Show user message: `⚠️  Rate limit hit on API key. Trying next key... (X/Y)`
   - Move to next key using round-robin (modulo wrap-around)
   - Retry the request with the new key
5. **Cycle Detection**: Track the starting key index; if we cycle back to it, all keys have been tried
6. **All Keys Failed**:
   - Log error with total attempts
   - Show user message: `❌ All X API keys exhausted. All keys have hit rate limits.`
   - Return error and stop processing
7. **Important**: The `GEMINI_API_KEY` environment variable is **never modified** at runtime; keys are passed directly to the API client via `AuthResolver`

### Conversation Management

The `conversation.rs` module handles:

- Persistent conversation storage in JSON format
- Automatic conversation history management
- Context window optimization through intelligent truncation
- UUID-based conversation identification with human-readable slugs
- Conversation listing and resumption functionality
- Token usage tracking per message
- API key preference caching for better continuity

**Conversation ID Format**: Conversations are identified by a slug-hash format: `<slug>-<4char-hash>`

- **Slug**: 3-5 significant words from first prompt (max 40 chars), with stopword filtering
- **Hash**: First 4 characters of UUID for uniqueness
- **Example**: `analyze-code-documentation-ab12`
- **Resumption**: Can resume by full ID, 4-char hash, or conversation index

### Testing

Tests use the `serial_test` crate to prevent environment variable conflicts when running in parallel. Tests cover:

- API key parsing and validation
- Multi-key fallback logic
- Input/output handling
- Conversation creation, serialization, and history management

## Important Notes

### CLI (gia)

- All logging goes to stderr, leaving stdout clean for piping
- The tool validates API key format but continues with warnings if invalid
- Rate limit handling automatically tries alternative keys if available
- Windows-specific registry support was removed in favor of environment variables only
- Clipboard input automatically detects and handles both text and images
- When using `-c` flag, if an image is in the clipboard, it will be treated as an image input rather than text

### GUI (giagui)

- The `show_conversation()` method (Ctrl+O) spawns the GIA command without clearing or modifying the response box - do not change this behavior
- Focus management: Prompt input automatically receives focus on application start
- Icon handling: Application icon and logo are embedded from `../../icons/gia.png` (workspace root)
- Requires `gia` binary in PATH

### Build System

- Both `gia/build.rs` and `giagui/build.rs` are identical copies that generate version info from git commit count
- Git commands traverse up to find `.git` directory at workspace root
**Environment variables set: `GIA_VERSION`, `GIA_COMMIT_COUNT`, `GIA_IS_DIRTY`**

**Default Model Configuration**: Both CLI and GUI respect the `GIA_DEFAULT_MODEL` environment variable:

- **Default**: `gemini-2.5-flash-lite`
- **CLI**: Uses environment variable for `--model` flag default value
- **GUI**: Uses environment variable for model dropdown default selection
- **Supported formats**: Standard model names (`gemini-2.5-pro`) or provider format (`ollama::llama3.2`)

### GitHub Actions

- Workflow builds both `gia` and `giagui` for three platforms (Windows x64, macOS Intel, macOS ARM)
- Releases 10 artifacts per release:
  - 6 bare binaries: `gia-{platform}` and `giagui-{platform}` for each platform
  - 2 macOS packages: `.pkg` installers for Intel and ARM
  - 2 macOS tarballs: `.tar.gz` archives for Intel and ARM
- Triggers Homebrew formula update via separate workflow
- Generates changelog from commits since last release
- Requires `GH_PERSONAL_TOKEN` secret for Homebrew tap updates

## Additional Features

### Shell Completions

Generate shell completion scripts for various shells:

```bash
# Bash
gia --completions bash > /usr/local/etc/bash_completion.d/gia

# Zsh
gia --completions zsh > ~/.zsh/completions/_gia

# Fish
gia --completions fish > ~/.config/fish/completions/gia.fish

# PowerShell
gia --completions powershell > gia.ps1

# Nushell
gia --completions nushell > gia-completions.nu
```

Supported shells: bash, zsh, fish, powershell, nushell

### Audio Recording

Native audio recording with no external dependencies:

```bash
# List available audio devices
gia --list-audio-devices

# Record with specific device
gia --record-audio --audio-device "Built-in Microphone"

# Use environment variable for default device
export GIA_AUDIO_DEVICE="USB Microphone"
gia --record-audio "Transcribe and summarize this"

# Transcribe-only mode (no conversation saved)
gia --record-audio --no-save

# Audio formats supported: OGG, Opus, MP3, M4A, MP4
# Automatic resampling to supported rates: 8k, 12k, 16k, 24k, 48k Hz
```

**Device Priority**: CLI argument > `GIA_AUDIO_DEVICE` environment variable > system default

### System Notifications

Automatic system notifications for certain operations:

- **Audio Recording Completion**: Notifies when audio recording + clipboard output completes (only if not using spinner mode)
- **Platform-specific**: Uses osascript on macOS, notify-rust on Windows/Linux

### Binary File Handling

Sophisticated file type detection:

- **Text Files**: Automatically decoded (supports non-UTF-8 encodings)
- **Media Files**: Auto-detected by extension and sent as multimodal content
- **Binary Files**: Skipped with warning messages to avoid corrupting prompts
- **Detection Heuristics**: BOM checking, null byte detection, character ratio analysis

### Browser Preview

When using `-b, --browser-output` flag:

- Saves formatted markdown to `~/.gia/outputs/<slug-hash>_<timestamp>.md`
- Opens in default browser automatically
- Includes metadata footer with:
  - Model name
  - Conversation ID (clickable link)
  - Token usage (prompt/completion/total)
  - Timestamp

### Verbose Help

Access detailed documentation:

```bash
gia --verbose-help  # Opens GitHub README in browser
```

### Context Window Management

Configure context window limits:

```bash
# Set custom context window limit (default: 8000)
export CONTEXT_WINDOW_LIMIT=16000

# Automatic truncation keeps last 20 messages when limit exceeded
# Token usage tracked per message for accurate counting
```

### Environment Variables Reference

Complete list of supported environment variables:

| Variable | Purpose | Default | Example |
|----------|---------|---------|---------|
| `GEMINI_API_KEY` | API key(s), pipe-separated | *required* | `key1\|key2\|key3` |
| `GIA_DEFAULT_MODEL` | Default AI model | `gemini-2.5-flash-lite` | `gemini-2.5-pro` |
| `GIA_AUDIO_DEVICE` | Default audio input device | system default | `Built-in Microphone` |
| `CONTEXT_WINDOW_LIMIT` | Context window size limit | `8000` | `16000` |
| `RUST_LOG` | Logging level (stderr) | none | `debug`, `info`, `error` |
| `GIA_LOG_TO_FILE` | Enable file logging | disabled | `1` |
| `OLLAMA_API_BASE` | Ollama server URL | `http://localhost:11434` | `http://custom:11434` |

## Installation

### Homebrew (macOS/Linux)

```bash
brew tap mschnecke/gia
brew install gia
```

### macOS Package Installer

Download `.pkg` installer from [GitHub Releases](https://github.com/mschnecke/gia/releases):

- Installs to `~/bin/` (user-specific, no admin required)
- Automatically updates shell profiles (.zshrc, .bash_profile, .profile)
- Removes quarantine attributes
- **Note**: Currently unsigned; may require "Open Anyway" in System Preferences

### From Source

```bash
git clone https://github.com/mschnecke/gia.git
cd gia
cargo build --release
cp target/release/gia ~/bin/  # or any directory in PATH
cp target/release/giagui ~/bin/
```

### Pre-built Binaries

Download from [GitHub Releases](https://github.com/mschnecke/gia/releases):

- `gia-macos-aarch64` - macOS ARM (M1/M2/M3)
- `gia-macos-x86_64` - macOS Intel
- `gia-windows-x86_64.exe` - Windows
- Corresponding `giagui-*` binaries for GUI

## Provider Architecture

### Adding New AI Providers

The codebase uses a provider abstraction pattern that makes it easy to add new AI backends:

1. **Implement the `AiProvider` trait** in a new module (e.g., `src/openai.rs`):

   ```rust
   pub struct OpenAiProvider { /* ... */ }

   impl AiProvider for OpenAiProvider {
       fn send_message(&mut self, content: Vec<ContentPartWrapper>) -> Result<String>;
       fn get_conversation_history(&self) -> &[ChatMessageWrapper];
   }
   ```

2. **Update `ProviderFactory`** in `src/provider.rs`:

   ```rust
   pub fn create_provider(model: &str, api_key: Option<String>) -> Result<Box<dyn AiProvider>> {
       if model.starts_with("openai::") {
           // Create OpenAI provider
       } else if model.starts_with("ollama::") {
           // Existing Ollama provider
       } else {
           // Default Gemini provider
       }
   }
   ```

3. **Model string format**: Use `provider::model` notation (e.g., `openai::gpt-4`, `anthropic::claude-3`)

4. **Conversation compatibility**: Use the wrapper types (`ChatMessageWrapper`, `ContentPartWrapper`) for cross-provider compatibility

### Keyboard Shortcuts (giagui)

Complete list of GUI keyboard shortcuts:

| Shortcut | Action |
|----------|--------|
| `Ctrl+Enter` | Send prompt |
| `Ctrl+R` | Send with audio recording |
| `Ctrl+L` | Clear form |
| `Ctrl+Shift+C` | Copy response to clipboard |
| `Ctrl+O` | Show conversation in browser |
| `F1` | Show help |
| `Ctrl+1` | Toggle "Resume" checkbox |
| `Ctrl+2` | Toggle "Clipboard Input" checkbox |
| `Ctrl+3` | Toggle "Browser Output" checkbox |
| `Ctrl+4` | Toggle "TTS Output" checkbox |
| `Ctrl+E` | Record audio (English transcription) |
| `Ctrl+D` | Record audio (German transcription) |

### giagui Features

Full GUI capabilities:

- **Auto-resume**: Automatically resumes previous conversation after first prompt
- **Ollama Model Discovery**: Auto-fetches available models from local Ollama instance
- **Task/Role Selection**: Load from `~/.gia/roles/` and `~/.gia/tasks/` directories
- **Spinner Mode**: Transparent, borderless, always-on-top 150x150 window with animated spinner
  - Launched with `--spinner` or `-s` flag
  - 8 rotating dots animation
  - Remains visible until process is killed
- **Response Display**: Markdown rendering with syntax highlighting
- **Environment Management**: Passes environment variables to spawned `gia` process
- **Focus Management**: Auto-focus on prompt input at startup

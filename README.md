# playerctl

Flutter plugin for Linux — wraps the `playerctl` CLI to give your app real-time media control over any MPRIS-compatible player. Pure Dart, no FFI.

## Requirements

- Linux only
- `playerctl` installed on the system

```bash
# Debian/Ubuntu
sudo apt install playerctl

# Arch Linux
sudo pacman -S playerctl

# Fedora
sudo dnf install playerctl

# openSUSE
sudo zypper install playerctl
```

## Installation

```yaml
dependencies:
  playerctl: ^1.2.0
```

Or from source:

```yaml
dependencies:
  playerctl:
    path: ../playerctl
```

## Usage

```dart
import 'package:playerctl/playerctl.dart';

final manager = MediaPlayerManager();

manager.stateStream.listen((state) {
  print('${state.currentMedia.title} — ${state.currentMedia.artist}');
  print('Status: ${state.playbackStatus}');
  print('Volume: ${state.volume}');
});

await manager.initialize();

await manager.play();
await manager.pause();
await manager.next();
await manager.previous();
await manager.setVolume(75);
await manager.toggleShuffle();
await manager.cycleLoop();
await manager.switchPlayer('spotify');

manager.dispose();
```

### Seeking

Position values follow the MPRIS standard — microseconds.

```dart
final position = await manager.getPosition();

await manager.seekTo(30000000);   // jump to 30s
await manager.seek(10000000);     // forward 10s
await manager.seek(-10000000);    // backward 10s
await manager.seekForward(10);    // forward 10s (seconds shorthand)
await manager.seekBackward(5);    // backward 5s

// target a specific player
await manager.seekForward(10, 'spotify');
```

### Album art

The plugin runs a small HTTP server on port `8765` to serve local album art files. Online URLs (Spotify, etc.) pass through unchanged.

```dart
final artUrl = state.currentMedia.artUrl;
// artUrl looks like: http://0.0.0.0:8765/art/abc123.jpg

// swap 0.0.0.0 with your machine's IP to access from other devices
final networkUrl = artUrl.replaceAll('0.0.0.0', '192.168.1.100');
```

The server starts when metadata is first fetched and stops when the manager is disposed. Health check at `http://0.0.0.0:8765/`.

### Error handling

```dart
final installed = await service.isPlayerctlInstalled();
if (!installed) {
  // playerctl not found on PATH
  return;
}

final players = await service.getAvailablePlayers();
if (players.isEmpty) {
  // no MPRIS players running
  return;
}
```

## Multi-player support

The plugin detects all active MPRIS players and lets you switch between them at runtime. If the active player closes, it switches automatically.

```dart
final players = await manager.getAvailablePlayers();
await manager.switchPlayer('vlc');
```

## Architecture

Three layers:

**Core** — `MediaPlayerManager` coordinates everything: player lifecycle, reconnection, stream merging, and debounced switching. `PlayerState` is an immutable snapshot of everything the plugin knows at a given moment.

**Services** — each service owns one concern:

- `MetadataProvider` — streams metadata; auto-restarts up to 5 times on crash
- `PlayerDetector` — lists and monitors available MPRIS players
- `PlaybackController` — sends play/pause/next/previous/shuffle/loop commands
- `VolumeController` — gets and sets volume; syncs periodically to catch external changes
- `CommandExecutor` — low-level process runner

**Models** — `MediaInfo` holds the current track's title, artist, album, status, player name, position, and length.

## Direct service access

```dart
final service = PlayerctlService();

bool installed = await service.isPlayerctlInstalled();
List<String> players = await service.getAvailablePlayers();

service.listenToMetadata().listen((metadata) {
  print('${metadata['title']} — ${metadata['artist']}');
});

// target a specific player
service.listenToMetadata('spotify');
await service.play('vlc');
await service.next('spotify');

await service.setVolume(75);
service.dispose();
```

## Logging

Log level can be set globally, per-instance, or at runtime.

```dart
// before startup
PlayerctlLogger.level = LogLevel.info;

// or in the constructor
final manager = MediaPlayerManager(logLevel: LogLevel.info);

// or while running
manager.setLogLevel(LogLevel.error);
```

Available levels: `none`, `error`, `warning`, `info`, `debug`.

Defaults: `debug` in debug mode, `error` in release mode.

## Example app

```bash
cd example
flutter run -d linux
```

## Troubleshooting

**"playerctl is not installed"** — install it via your distro's package manager (see Requirements).

**"No active media players found"** — open a player that supports MPRIS (Spotify, VLC, Firefox, etc.) and start playing something.

**Commands not working** — not all players implement the full MPRIS spec. Check what the player supports.

**Stream stopped updating** — there's a periodic fallback that refreshes every 3 seconds even if the real-time stream stalls.

**Glitching on player switch** — switching is debounced. If it's still rough, enable debug logging to check the timer lifecycle.

## Supported players

Anything MPRIS-compatible: Spotify, VLC, Firefox, Chromium, MPV, Audacious, Rhythmbox, and more.

## License

MIT

## Contributing

PRs welcome.

## Credits

- [playerctl](https://github.com/altdesktop/playerctl) — the CLI this plugin wraps
- Flutter

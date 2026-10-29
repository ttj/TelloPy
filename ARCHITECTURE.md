# Architecture

Source map checked at `2e3ff77f87448307d6d2656c91ac80e2fb352193` on `develop-0.7.0`.
This TTJ fork retains the upstream TelloPy driver by Hanyazou, whose README attributes much of the protocol implementation to Gobot. It speaks the Tello's raw binary command/telemetry protocol and exposes video, file-transfer and log events. It is a driver library with hardware examples, not the lab's navigation stack or SDK text-protocol controller.

## Read first

- [README.md](README.md) describes upstream installation and examples.
- [setup.py](setup.py) declares package `tellopy`, version `0.7.0.dev0`, upstream attribution and package discovery. Its broad historical Python requirement is not a current compatibility test matrix.
- [tellopy/__init__.py](tellopy/__init__.py) exports `Tello`; [its implementation](tellopy/_internal/tello.py) contains the API docstrings and transport lifecycle.
- The lab's [external-project declaration](https://github.com/ttj/vu-autonomy/blob/main/config/external_projects.json) and [external-project guide](https://github.com/ttj/vu-autonomy/blob/main/docs/EXTERNAL-PROJECTS.md) define this optional fork's role. They distinguish it from `ttj/DJITelloPy` and the flight path's SDK controller.

## Module map

| Area | Responsibility and useful seam |
| --- | --- |
| [tello.py](tellopy/_internal/tello.py) | `Tello` owns sockets, connection state, RC stick state, command transmission, packet processing, video reception and file/log dispatch. Public APIs include `connect()`, `wait_for_connection()`, `subscribe()`, `get_video_stream()`, `take_picture()`, `record_log_data()` and `quit()`. |
| [protocol.py](tellopy/_internal/protocol.py) | Command IDs and wire models. `Packet` builds packets; `FlightData` and `LogData` parse telemetry; `DownloadedFile` assembles file fragments; `VideoData` tracks video packet gaps. |
| [crc.py](tellopy/_internal/crc.py), [utils.py](tellopy/_internal/utils.py) | Packet checksum and byte-conversion helpers used by the protocol/driver. |
| [event.py](tellopy/_internal/event.py), [dispatcher.py](tellopy/_internal/dispatcher.py), [state.py](tellopy/_internal/state.py) | Event identities, synchronous subscriber dispatch and connection state values. The dispatcher registry is module-global. |
| [video_stream.py](tellopy/_internal/video_stream.py) | `VideoStream` subscribes to raw video events, drops incomplete post-gap frame starts, queues payload bytes and exposes a file-like `read()` interface. It does not decode H.264 into images. |
| [examples/](tellopy/examples) | Physical-device demonstrations: takeoff, keyboard/joystick control, video display and log recording. Additional video/UI packages are example dependencies, not core `install_requires`. |
| [tests/](tests) | Historical hardware/UI scripts and shell launchers. `tests/test.py` constructs a controller and sends movement/landing commands; the directory name does not make it an offline regression suite. |

## Data and control flow

Constructing `Tello` binds the command socket (default local port `9000`) and starts receive/video threads. `connect()` publishes a connection-request event; the state machine sends the raw handshake to `192.168.10.1:8889`, handles acknowledgements/timeouts and signals connection state. The receive loop sends current stick values when connected, receives packets and routes their command IDs to parsers or events. Movement setters update stick state; command APIs build and send raw packets.

Consumers subscribe to events such as `EVENT_FLIGHT_DATA`, `EVENT_LOG_RAWDATA`, `EVENT_LOG_DATA` and `EVENT_FILE_RECEIVED`. The dispatcher calls handlers synchronously in the publishing thread, so handlers should avoid blocking receive work. A still-image request sends `TAKE_PICTURE_COMMAND`; file-size and file-data messages allocate/assemble `DownloadedFile` buffers, acknowledge chunks/completion and publish the completed file bytes.

The separate video thread binds UDP `6038`. It publishes video payload events; `get_video_stream()` lazily creates `VideoStream` and starts video. Example applications pass that byte stream to a decoder such as PyAV before displaying frames. Video bytes, decoded images, raw log bytes and parsed flight/log objects are distinct interfaces.

## Setup and entrypoints

Use an isolated development environment and the upstream README's source installation instructions when evaluating this fork. `setup.py` has no core third-party `install_requires`; video/joystick examples additionally need their imported packages. There is no installed console-script entrypoint. Read API docstrings through `import tellopy; help(tellopy.Tello)` without constructing a controller.

The lab already has [vu-autonomy/scripts/tello_stills.py](https://github.com/ttj/vu-autonomy/blob/main/scripts/tello_stills.py) as its optional stills integration. Its `--protocol` mode prints the documented exchange without constructing `Tello`; capture mode is a separate live-device session. Reuse this entrypoint rather than adding another stills or transfer implementation. Its source explicitly says successful JPEG capture on the lab's airframes remains unverified.

The README's `python -m tellopy.examples.simple_takeoff` is a hardware flight demonstration: it connects, takes off, sends a descent command and lands. It is not an installation smoke test. `quit()` requests driver shutdown; examples use a `finally` block for cleanup. This guide does not execute those commands or establish firmware/airframe support.

## Traps and change boundaries

- Importing the library and constructing `Tello()` have different effects: construction immediately binds sockets and starts threads, before `connect()`. Avoid controller construction in source-discovery or documentation checks.
- `tellopy` and `djitellopy` use different protocols/APIs. The lab's external-project contract requires separate raw-stills and SDK flight sessions; do not run them as concurrent controllers for one aircraft or replace the flight environment's dependency.
- File receive code explicitly warns against taking a second picture while the first transfer is active. It also records a short second file-size payload seen in its upstream observations; callers must wait for actual file completion.
- Event handlers receive `event`, `sender`, `data` and optional keyword arguments. Raw video events and completed file events are not decoded image callbacks.
- Legacy setup classifiers and example names do not prove modern dependency compatibility, control quality or lab hardware readiness. No GitHub Actions workflow is checked in at this source snapshot.

Index searched: lab CODE_MAP/CODE_INDEX, `config/external_projects.json`, `docs/EXTERNAL-PROJECTS.md`, safe sibling README/docs and integration references, target README/setup/driver/protocol/video/events/examples/tests and all-state architecture PR/issue search.

Reusing: `tellopy.Tello`, its existing protocol/event/video/file-transfer seams and `vu-autonomy/scripts/tello_stills.py`. New because: this safe TTJ fork lacked a root architecture onboarding guide; upstream implementation and existing lab integration remain unchanged.

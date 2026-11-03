# Multithreaded Server

A Python TCP server that shares one integer among concurrent clients and coordinates readers and writers with a writer-preferring lock.

## Features

- Accepts TCP connections and handles each one on its own thread.
- Keeps a single shared integer, starting at `0`.
- Answers `READ` with `READ:<value>`. Readers may be in the critical section together while no writer is active or waiting.
- Answers `WRITE:<integer>` with `WRITE:OK` and stores that integer. A writer waits until every reader and writer has left the critical section, and a waiting writer blocks new readers.
- When a writer finishes, another waiting writer is woken if one exists; otherwise every waiting reader is woken.
- Runs a 2,000,000-iteration busy loop inside the critical section before sending the reply, so overlapping access shows up in the server log.

## Tech stack

- Python 3
- Standard library only: `socket` and `threading` on the server; the clients also use `time`
- No third-party packages

## Project layout

| File | Role |
| --- | --- |
| `multithreaded_server.py` | Listening server and readers-writers lock |
| `read_client.py` | Demo reader clients |
| `write_client.py` | Demo writer clients |

## Setup

Nothing to install. Use the Python 3 interpreter already on the machine.

The server and both clients resolve the address with `socket.gethostbyname(socket.gethostname())` and use port `5566`. Run them on the same machine so that lookup reaches the process that bound the port.

## Usage

Start the server:

```bash
python3 multithreaded_server.py
```

It prints `Server is starting...` and then `Server is listening on <ip>:5566`, and it keeps accepting connections.

In another terminal, run the readers:

```bash
python3 read_client.py
```

`read_client.py` opens three reader connections, one after another, in a single process. Each connection sends `READ` ten times, prints the reply (`READ:<value>`), and waits one second after each reply. It then sends `!DISCONNECT`.

In a third terminal, run the writers. Starting this while the readers are still running is what overlaps the two sides on the server:

```bash
python3 write_client.py
```

`write_client.py` opens three writer connections, one after another, in a single process. Each connection sends `WRITE:1` through `WRITE:10`, prints `WRITE:OK`, and waits one second after each reply. It then sends `!DISCONNECT`.

The server finishes a connection when `recv` returns no data, which happens once the client socket closes. After `write_client.py` finishes, the shared integer is `10`, the last value that script writes.

## Message format

Messages are UTF-8. Each receive reads up to 1024 bytes.

| Client sends | Server replies | Effect |
| --- | --- | --- |
| `READ` | `READ:<value>` | Returns the shared integer |
| `WRITE:<integer>` | `WRITE:OK` | Sets the shared integer to the integer after the first `:` |

A write uses `int()` on the text after the first colon, so the payload is a base-10 integer such as `WRITE:10`.

## License

MIT License. Copyright (c) 2026 Farras Inas. See [LICENSE](LICENSE).

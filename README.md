# IPSec ESP Hijack Demo

A C++ demonstration of how a TCP connection protected by IPSec ESP in transport mode can
be taken over by an on-path attacker when the security association is weak. The project
ships a toy client and server that talk over an ESP-protected channel, plus a hijacking
tool that observes the channel and injects its own ESP-encapsulated TCP segments. The ESP
packet handling (SPI, sequence numbers, AES encryption and HMAC-SHA1 authentication) is
implemented from scratch using [Crypto++](https://www.cryptopp.com/), and the whole
scenario runs inside a self-contained Docker network so nothing leaves the lab.

> For education and authorized testing only. Run it exclusively against the bundled Docker
> environment or networks and systems you own or have explicit permission to test.

## How it works

- `source/encoder.cpp` implements the ESP cipher and authentication primitives (an AES
  encryptor/decryptor and an HMAC generator) over the Crypto++ library.
- `source/sadb.cpp` reads the kernel's IPSec security associations through the PF_KEY
  (`PF_KEY_V2`) interface so the tools use the same keys and parameters that the Linux
  `xfrm` stack was configured with.
- `source/session.cpp` tracks the ESP and TCP sequence state of an observed connection.
- `source/util.cpp` provides packet-header structures and socket helpers.
- `standalone/source/` contains the three executables: `server`, `client` and `hijack`.
- `scripts/config.sh` configures the Linux `xfrm` IPSec state and policy (transport-mode
  ESP with `hmac(sha1)` authentication and a null cipher) that the demo relies on.

## Repository structure

```
esp-hijack/
  include/            ESP, SADB, session and utility headers
  source/             library implementation (encoder, sadb, session, util)
  standalone/source/  server.cpp, client.cpp, main.cpp (hijack)
  scripts/config.sh   xfrm IPSec state/policy setup
  cmake/              CPM.cmake and tool helpers (starter framework)
  CMakeLists.txt      library build
  Makefile            convenience wrapper that builds the hijack tool
Dockerfile            build environment image
docker-compose.yml    client/server containers and network
```

## Requirements

- Docker and Docker Compose.
- A C++20 toolchain and CMake 3.24+ if building outside the container. Crypto++ and
  cxxopts are fetched automatically through CPM at configure time.
- Linux with `xfrm`/IPSec support; the tools use raw and PF_KEY sockets and therefore
  need root (or `NET_ADMIN`), which the Compose file already grants to the containers.

## Build and run

Build the image:

```bash
sudo docker build -t esp-hijack -f Dockerfile .
```

Start the client and server containers (the Compose network uses `172.18.0.0/16`; remove
any existing network on that range first):

```bash
sudo docker compose -f docker-compose.yml up -d
```

Run the server (listening on port `1111`):

```bash
docker exec -it server bash
./server 1111
```

Run the client (server IP, server port, client port):

```bash
docker exec -it client bash
./client 172.18.100.254 1111 2222
```

Build and run the hijacking tool on a chosen interface:

```bash
docker exec -it client bash
make
./hijack eth0
```

Tear everything down:

```bash
sudo docker compose -f docker-compose.yml down
sudo docker rmi esp-hijack
```

## Environment

With the default Compose settings:

- server: `172.18.100.254:1111`, SSH exposed on host port `3333`
- client: `172.18.1.1:2222`, SSH exposed on host port `4444`

`scripts/config.sh` installs matching `ip xfrm` state and policy on each side and takes a
single argument, `server` or `client`, to select the direction of each security
association.

## Notes

- The CMake project layout (CPM-based dependency fetching and the `standalone`/`all`
  split) comes from a C++ starter framework and is distributed under the MIT license in
  `esp-hijack/LICENSE`.
- `scripts/config.sh` contains fixed demonstration keys for the lab's security
  associations; they are placeholders for the controlled environment and should never be
  reused elsewhere.

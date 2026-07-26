# device-control-ios — WebDriverAgent pin for iOS device control

Support repo that pins and vendors the [WebDriverAgent](https://github.com/llm-actuators/WebDriverAgent) build [`idb`](https://github.com/llm-actuators/idb) drives real iPhones/iPads with. It carries no code of its own — its job is to fix *which* WDA revision the iOS control surface is built and run against, so device automation is reproducible across machines.

Part of the [llm-actuators](https://github.com/llm-actuators) toolchain — single-purpose actuators an LLM uses to act on real systems (devices, processes, files, other agents). Each binary is sterile: primitives only, domain knowledge lives in the model.

## What it does

- **Pins WebDriverAgent.** WDA is the on-device HTTP automation server (an Appium-lineage XCUITest app) that exposes tap/swipe/type/screenshot/element-query over a REST API. Which WDA revision you build against materially changes behavior; this repo is the single place that revision is declared, so every machine running `idb` builds the same server.
- **Vendors it as a submodule.** The pinned source lives at `WebDriverAgent/`, tracked as a git submodule against [`llm-actuators/WebDriverAgent`](https://github.com/llm-actuators/WebDriverAgent).
- **Nothing else.** No Swift sources, no wrapper binary, no build system of its own. The control logic — session lifecycle, port forwarding, request shaping, retry — lives in [`idb`](https://github.com/llm-actuators/idb). This repo is the integration point, not the driver.

## Layout

```
device-control-ios/
├── WebDriverAgent/          # pinned WDA source (git submodule -> llm-actuators/WebDriverAgent)
├── .github/workflows/ci.yml # docs link-check only (offline); no build/test job
└── README.md
```

There is no `Package.swift` or `.xcodeproj` at this level — those belong to WDA itself, under `WebDriverAgent/`.

## Build

You do not build this repo directly; you build the pinned WDA and run it on-device. That is normally done *for* you by `idb`, but the underlying steps are:

```sh
# clone with the pinned WDA
git clone --recurse-submodules git@github.com:llm-actuators/device-control-ios.git
# or, in an existing checkout:
git submodule update --init --recursive

# build + install the WDA runner onto a connected device (signing required)
cd WebDriverAgent
xcodebuild \
  -project WebDriverAgent.xcodeproj \
  -scheme WebDriverAgentRunner \
  -destination 'id=<UDID>' \
  test
```

Signing (a valid provisioning profile / team) is required to install WDA on a physical device — the same constraint any XCUITest runner carries. Consult the [WebDriverAgent](https://github.com/llm-actuators/WebDriverAgent) repo for the authoritative build matrix.

## Usage

You do not invoke this repo. The iOS control surface is driven through [`idb`](https://github.com/llm-actuators/idb), which starts the pinned WDA on a claimed device, holds the session, and translates high-level commands (`tap`, `swipe`, `type`, `screenshot`, `launch`, `screen`) into WDA REST calls. Callers talk to `idb`; `idb` talks to WDA; WDA talks to the device.

```
agent / caller
      │
      ▼
    idb  ──starts & drives──▶  WebDriverAgent (on device)  ──▶  iPhone / iPad
      │                              ▲
      └── revision pinned here ──────┘
```

Do not issue raw WDA HTTP calls or hand-roll `curl` to the WDA port — the session lifecycle, forwarding, and device-mutex handling in `idb` exist to keep that correct and contended-safe.

## Where it fits

The toolchain exposes one device-control surface per platform, kept behaviorally parallel:

| Platform | Driver | On-device layer |
|----------|--------|-----------------|
| iOS      | `idb`  | WebDriverAgent (pinned by **this repo**) |
| Android  | `ddb`  | ADB / UIAutomator |
| Web      | `wdb`  | Chrome DevTools Protocol |

All three present the same verb vocabulary (tap, swipe, type, screenshot, launch, inspect screen) so an agent drives a phone, an emulator, or a browser through one mental model. This repo's sole contribution to that surface is reproducibility: it guarantees every `idb` install builds against the same WebDriverAgent revision, so iOS device behavior does not drift machine to machine.

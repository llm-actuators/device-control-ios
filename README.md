# device-control-ios

> iOS device-control support repo (WebDriverAgent integration for \`idb\`).

Part of the [llm-actuators](https://github.com/llm-actuators) toolchain — single-purpose tools an LLM uses to act on real systems (devices, processes, files, other agents). Each binary is sterile: primitives only, domain knowledge lives in the model.

## What it does

Support repo for iOS physical-device control: pins/wraps the WebDriverAgent used by `idb` to drive real iPhones/iPads. See [idb](https://github.com/llm-actuators/idb) for the device-control surface.

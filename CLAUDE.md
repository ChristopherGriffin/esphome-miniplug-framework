# ESPHome Framework Instructions

This repo contains reusable ESPHome packages, hardware definitions, add-ons, and function templates for building local-first ESPHome devices.

## Project Layout

- `common/`: shared base ESPHome configuration used across devices.
- `addon/`: optional feature packages such as BMS, Modbus, notify, and battery analysis.
- `functions/`: reusable device behavior templates such as miniplug appliance, heater, lamp, monitor, battery backup, and timed light logic.
- `hardware/`: board and hardware-specific definitions such as Sonoff S31, SwitchBot MiniPlug, NodeMCU, and Wemos D1 Mini variants.
- `examples/`: complete example device configurations.
- `docs/`: supporting documentation and design notes.
- `README*.md`: user-facing documentation for framework features.

## General Rules

- Prefer small, targeted edits.
- Do not rewrite entire files unless explicitly asked.
- Preserve existing entity IDs unless explicitly asked to change them.
- Preserve existing file structure and naming conventions.
- Prefer ESPHome `packages`, `substitutions`, `!include`, and reusable YAML blocks over duplicated configuration.
- Keep YAML readable and maintainable.
- Do not introduce cloud dependencies.
- Do not assume Home Assistant is always available for core device behavior.
- Home Assistant may enhance behavior, but the ESPHome device should retain its basic local function whenever practical.
- Never expose, print, summarize, or copy values from `secrets.yaml`.
- Use `secrets.yaml.example` only as a template reference.

## ESPHome Design Philosophy

Devices in this repo should follow a local-first model:

1. The ESPHome device should handle its core function locally.
2. Home Assistant may provide enhanced control, visibility, scheduling, or automation.
3. External data from Home Assistant may be used as an enhancement, not as a hard requirement for basic safety or operation.

Avoid designs where a device becomes useless, unsafe, or stuck simply because Home Assistant, Wi-Fi, or internet access is unavailable.

## Editing Rules

When modifying YAML:

- Make the smallest change that solves the problem.
- Read only the files needed for the requested task.
- Do not scan the entire repo unless specifically asked.
- Check directly included packages before assuming behavior.
- Prefer modifying the most specific file rather than a shared framework file.
- Only modify shared files when the change is clearly useful to multiple devices.
- Avoid changing hardware definitions unless the task is specifically hardware-related.
- Avoid changing examples unless the task is about documentation or example correctness.

## ESPHome Validation

After editing an ESPHome YAML file, validate it when possible with:

```bash
esphome config <file>
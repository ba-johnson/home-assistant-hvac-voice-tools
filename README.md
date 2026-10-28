# Home Assistant HVAC voice tools

Set HVAC operating modes through an LLM conversation agent and confirm the device's reported mode before returning success.

Home Assistant already provides `climate.set_hvac_mode`. These native HA scripts make mode discovery and verified mode changes available as Assist LLM tools. They need no custom integration, HACS installation, external server or API token.

## Why this exists

In a tested Home Assistant 2026.10.4 installation, the default Assist climate tool exposed temperature control but no HVAC-mode setter. An Ollama agent interpreted “set the aircon to heat” as a temperature request, then claimed the mode had changed. These tools supply the missing operation and an explicit result.

## Tools

| Tool | Behaviour |
|---|---|
| `script__hvac_available_modes` | Lists climate entities, names, areas, reported modes and each unit's supported modes. Optional name/entity ID filters the list. |
| `script__hvac_set_mode_verified` | Resolves a name/entity ID, validates the requested mode, calls `climate.set_hvac_mode`, and waits up to 40 seconds for the matching reported mode. |

The setter accepts an exact entity ID, friendly name (case-insensitive), or entity object ID with underscores replaced by spaces. Unknown or ambiguous names are rejected. It does not resolve aliases or fuzzy matches itself: the conversation agent should choose a unit from discovery or ask for clarification.

The tool returns `success: true` only after the reported mode matches. Invalid targets, unavailable entities, unsupported modes and confirmation timeouts return `success: false` with a reason. A service-call exception may also propagate as a tool error. Target temperature is never set by these scripts; an integration may independently adjust attributes when changing mode.

## Supported modes

All standard HA HVAC modes are accepted **only when listed in the chosen unit's `hvac_modes` attribute**:

| Stored mode | Meaning |
|---|---|
| `off` | Off |
| `heat` | Heating |
| `cool` | Cooling |
| `heat_cool` | Automatic heating/cooling |
| `auto` | Device schedule, learned behaviour or AI mode |
| `dry` | Dry/dehumidification |
| `fan_only` | Fan only |

`heat_cool` and `auto` are distinct. A heat-only thermostat cannot be switched to cooling. Fan speeds, presets, swing and humidity targets are separate controls and are outside this package.

## Install

1. Back up your HA scripts and the conversation agent's existing instructions. If editing files through a shell:

   ```sh
   # Run in your HA configuration directory; verify pwd first.
   pwd
   stamp=$(date +%Y%m%d-%H%M%S)
   cp -a scripts.yaml "scripts.yaml.bak.$stamp"
   ```

2. Merge the **two top-level script entries** from [scripts.yaml](scripts.yaml) into your existing HA `scripts.yaml`. Keep all existing entries. Do not overwrite the whole file or add a surrounding `script:` key if your `configuration.yaml` already uses `script: !include scripts.yaml`.

   Alternatively, create two scripts in the HA script editor. Use **Edit in YAML** and paste each entry's value, without its outer key. Ensure their entity IDs are exactly `script.hvac_available_modes` and `script.hvac_set_mode_verified`; the prompt uses those names.

3. Check configuration in **Developer tools → YAML**, then reload **Scripts**. A full HA restart should not be necessary when scripts are already configured.
4. In **Settings → Voice assistants → Expose**, expose both scripts to Assist. UI labels can vary by HA version. In your LLM conversation agent's settings, enable Home Assistant control through the Assist API.
5. Append [agent-prompt.md](agent-prompt.md)'s text to that agent's existing instructions. Do not replace its other instructions. Apply it separately to each agent you intend to use; the provided wording is English.
6. Remove competing one-device HVAC workaround scripts from Assist exposure, if any.

**Scope:** when exposed, these scripts can discover and control any loaded `climate` entity. They do not enforce each entity's separate Assist exposure setting. Expose the scripts only to an assistant you intend to give that scope. They also discover newly added climate entities automatically.

## Examples

- “What modes does the study aircon support?”
- “Set the study aircon to dry mode.”
- “Set the workshop aircon to fan only.”
- “Set the aircon to automatic heating and cooling.”

Use your own unit names. An ambiguous name or “automatic” with multiple possible meanings should produce a clarification.

To test the setter without voice, use **Developer tools → Actions**:

```yaml
action: script.hvac_set_mode_verified
data:
  entity_id: climate.your_aircon
  hvac_mode: heat
```

Calling the script directly waits for completion and exposes its response to callers supporting response data. Use the direct script action, not the fire-and-forget `script.turn_on` action, when you need the verified result.

## Validate

First ask which modes a unit supports. Then test a suitable real mode transition and inspect the Assist debug trace. Expect `script__hvac_set_mode_verified`, followed by a result containing `success: true` and the requested `hvac_mode`, **before** a spoken success response. Confirm the entity state independently in HA.

For a negative test, request a mode absent from a chosen unit's supported list. Expect `success: false`, no mode command from the script, and no invented substitute temperature action. The prompt is necessary: tool-side validation cannot guarantee an LLM will choose the correct tool or faithfully describe errors.

## Tested status and limits

Tested on HA Core **2026.10.4**, with the Assist API and **Ollama qwen3:14b**:

- Discovery across 22 climate entities.
- Real Heat → Fan only → Heat transitions through the configured voice pipeline, with confirmation after the verified result.
- Setter on a second unit by exact entity ID.
- Unsupported cooling on a heat-only thermostat, nonexistent entity, wrong entity domain and invented mode rejection.
- A complete spoken list of the six modes advertised by a Panasonic aircon.

These tests used the pipeline from the intent stage. They do not constitute microphone/TTS validation or physical testing of every device/mode combination. Different HA versions, agents and integrations may behave differently.

The returned mode is the **integration-reported state**, not independent physical telemetry. `hvac_action` may lag and is returned as context; it is not the success condition. A compressor need not start immediately when a mode is selected. This is a script extension, not a patch to HA Core's built-in climate tool provider.

## Uninstall / rollback

1. Remove both scripts from Assist exposure.
2. Remove only `hvac_available_modes` and `hvac_set_mode_verified` from your scripts configuration (or delete the two scripts in the editor).
3. Reload Scripts and restore your previous agent instructions. Re-expose a previous workaround only if you want it back.

If restoring an entire backup, preserve later unrelated edits first:

```sh
# Replace the filename with the exact backup created during installation.
cp -a scripts.yaml.bak.YYYYMMDD-HHMMSS scripts.yaml
```

Check configuration and repeat a read-only Assist question to confirm the system remains operational. Uninstalling these tools does not restore physical HVAC modes changed while they were installed.

## References

- [HA climate actions and mode meanings](https://www.home-assistant.io/integrations/climate/)
- [HA scripts](https://www.home-assistant.io/integrations/script/)
- [Assist script tools in HA 2026.10.4](https://github.com/home-assistant/core/blob/2026.10.4/homeassistant/components/script/llm.py)
- [Climate tools in HA 2026.10.4](https://github.com/home-assistant/core/blob/2026.10.4/homeassistant/components/climate/llm.py)

MIT licensed. Provided as-is; this repository is not a managed installation or support service.

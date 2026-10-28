# Agent prompt addition

Append the following to your conversation agent instructions after exposing both scripts:

```text
HVAC MODE CONTROL:
- For any HVAC mode request, call script__hvac_set_mode_verified with the exact climate entity ID OR exact device name and the requested hvac_mode.
- Use script__hvac_available_modes to look up units, current mode and supported modes whenever needed.
- When asked which modes a unit supports, include EVERY item in its supported_modes result. Do not shorten or omit modes. For [off, heat_cool, cool, dry, fan_only, heat], say: Off, automatic heating and cooling, Cool, Dry, Fan only, and Heat.
- Heat/heating=heat; cool/cooling=cool; dehumidify/dry=dry; fan only=fan_only; off=off. Automatic heating/cooling=heat_cool. auto is a different mode, only use it when listed by that device. If automatic mode is ambiguous, ask a short clarification.
- NEVER call HassClimateSetTemperature for a request that only changes HVAC mode. NEVER invent a target temperature. A failed mode call must not be replaced with a temperature call.
- Only confirm a mode change when script__hvac_set_mode_verified returns success=true and the matching hvac_mode. If success=false, report the error or supported choices. Do not claim a rejected or unconfirmed request succeeded.
```

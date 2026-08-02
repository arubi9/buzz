# OCX with Claude Code

The **OCX (Claude Code)** Buzz harness sends Claude Code traffic to the local
OpenCodex gateway. OCX owns provider credentials, provider model selection,
and its catalog; Buzz does not read or change OCX or Codex configuration.

## Model catalog

OCX has two catalog views:

- `ocx models list` shows static and custom models. It does not list models
  discovered live from OpenRouter.
- Claude Code and Buzz query OCX's local `/v1/models` endpoint for the active
  catalog. Only models selected in OCX are returned there.

If an OpenRouter model is absent from the picker, verify that it is selected
in OCX and use OCX's own commands to refresh its catalog. Restart Claude Code
after an OCX catalog refresh so it reloads the model menu.

## Boundary

Buzz only reads the loopback catalog exposed by OCX. It never starts, stops,
or configures the OCX service, and it never writes Codex configuration or
model catalog files.

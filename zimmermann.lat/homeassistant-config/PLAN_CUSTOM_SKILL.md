# Plan: Replace Boolean-Based Voice Queries with Alexa Custom Skill

## Goal
Replace the current `input_boolean` → automation → `notify.send_message` pattern with a proper **Alexa Custom Skill** using `intent_script`. The response goes back to the Alexa device that asked — no booleans, no room mapping, no notify calls needed.

## Why This Is Better
| Current (boolean) approach | Custom Skill approach |
|---|---|
| 4 input_booleans per query type per room (12 total for 3 queries) | 0 booleans |
| customize.yaml entries for room-to-speaker mapping | Not needed — Alexa responds on the device that was asked |
| alexa.yaml entity exposure for each boolean | Not needed |
| 1 automation per query type + shared script | 1 intent_script entry per query type |
| User configures Alexa routines manually per room/device | User just speaks: "Alexa, frage [name] ..." from any device |

## Architecture

```
Current:  User speaks → Alexa Routine → turns on input_boolean → HA automation
          → looks up speaker → notify.send_message → Alexa speaks

New:      User speaks → Alexa Custom Skill → Lambda → HA /api/alexa
          → intent_script returns speech → Lambda → Alexa speaks on same device
```

## Implementation Steps

### Step 1: Manual — Create Custom Skill in Alexa Developer Console

This is done in https://developer.amazon.com/alexa/console/ask (not in code).

1. **Create a new skill** (separate from "homeassistant2" Smart Home skill)
   - Type: **Custom**
   - Name: e.g. "Zuhause Abfrage" (display name)
   - Language: **German (DE)**
   - Hosting: **Provision your own** (Lambda)
   - Choose an invocation name, e.g. **"mein zuhause"**
     → User says: "Alexa, frage mein Zuhause wie warm es draußen ist"

2. **Define the Interaction Model** (JSON or via console):

   **Intents:**

   | Intent | Sample Utterances (German) |
   |--------|---------------------------|
   | `GetOutsideTemperatureIntent` | "wie ist die außentemperatur", "wie warm ist es draußen", "wie kalt ist es draußen", "sag mir die temperatur draußen" |
   | `GetHumidityIntent` | "wie ist die luftfeuchtigkeit", "wie feucht ist es", "sag mir die luftfeuchtigkeit" |
   | `GetWindowStatusIntent` | "sind fenster offen", "welche fenster sind offen", "ist ein fenster offen", "fenster status" |

   No custom slots needed — all three queries return global data, not per-room.

3. **Configure Endpoint** → point to the Lambda function (Step 2)

4. **Set up Account Linking:**
   - Authorization URI: `https://homeassistant.zimmermann.lat/auth/authorize`
   - Access Token URI: `https://homeassistant.zimmermann.lat/auth/token`
   - Client ID: `https://layla.amazon.com/` (for EU region)
   - Client Secret: any non-empty string (HA doesn't validate it)
   - Scheme: **Credentials in request body**
   - Scope: `intent`

5. **Complete account linking** in the Alexa app after deploying

### Step 2: Manual — Create AWS Lambda Function

1. **Create a new Lambda** in **eu-west-1** (Ireland) — required for DE locale Alexa skills
   - Runtime: Python 3.12
   - Create IAM role with `AWSLambdaBasicExecutionRole`
   - Add **Alexa Skills Kit** trigger (paste Skill ID from Step 1)

2. **Lambda code** — use the standard HA proxy from the docs (Python):
   ```python
   # Standard HA Alexa Custom Skill proxy
   # Forwards all requests to HA's /api/alexa endpoint
   ```
   Environment variables:
   - `BASE_URL`: `https://homeassistant.zimmermann.lat`

3. **Copy the Lambda ARN** → paste as endpoint in the Alexa skill (Step 1)

### Step 3: Code — Add `intent_script.yaml` to HA config

**New file:** `zimmermann.lat/homeassistant-config/intent_script.yaml`

```yaml
GetOutsideTemperatureIntent:
  speech:
    text: >
      {%- set temp = states('sensor.aussenthermometer_temperature') %}
      {%- if temp in ['unknown', 'unavailable'] %}
      Die Außentemperatur ist momentan nicht verfügbar.
      {%- else %}
      Die Außentemperatur beträgt {{ temp | float | round(1) }} Grad.
      {%- endif %}

GetHumidityIntent:
  speech:
    text: >
      {%- set max_humidity = states('sensor.feuchtigkeitssensoren') | float(0) | round(0) | int %}
      {%- set group = 'sensor.feuchtigkeitssensoren' %}
      {%- set sensors = state_attr(group, 'entity_id') %}
      {%- set threshold = states('input_number.humidity_threshold_percent') | float(60) %}
      {%- set ns = namespace(room_data=[]) %}
      {%- if sensors %}
        {%- for sensor_id in sensors %}
          {%- set sensor_state = states(sensor_id) %}
          {%- if sensor_state not in ['unknown', 'unavailable'] and sensor_state | float(0) > threshold %}
            {%- set room_name = sensor_id | area_name %}
            {%- set humidity_val = sensor_state | float | round(0) | int %}
            {%- set ns.room_data = ns.room_data + [{'name': room_name, 'value': humidity_val}] %}
          {%- endif %}
        {%- endfor %}
      {%- endif %}
      {%- set sorted_rooms = ns.room_data | sort(attribute='value', reverse=true) %}
      {%- set ns2 = namespace(room_strings=[]) %}
      {%- for room in sorted_rooms %}
        {%- set ns2.room_strings = ns2.room_strings + [room.name ~ ' ' ~ room.value ~ ' Prozent'] %}
      {%- endfor %}
      Die Luftfeuchtigkeit beträgt {{ max_humidity }} Prozent.
      {%- if ns2.room_strings | length > 0 %} {{ ns2.room_strings | join(', ') }}.{% endif %}

GetWindowStatusIntent:
  speech:
    text: >
      {%- set windows = expand(states.binary_sensor)
         | selectattr('attributes.device_class', 'eq', 'window')
         | selectattr('state', 'eq', 'on')
         | list %}
      {%- if windows | length > 0 %}
        {%- set names = windows | map(attribute='name') | list %}
      Ja, {{ windows | length }} {{ 'Fenster ist' if windows | length == 1 else 'Fenster sind' }} geöffnet: {{ names | join(', ') }}.
      {%- else %}
      Nein, alle Fenster sind geschlossen.
      {%- endif %}
```

### Step 4: Code — Wire up configuration.yaml and ConfigMap

**`configuration.yaml`** — add:
```yaml
intent_script: !include intent_script.yaml
```

**`templates/configmap.yaml`** — add `intent_script.yaml` entry so it gets deployed.

### Step 5: Code — Clean up the boolean-based approach

Once the Custom Skill is tested and working, remove:

| File | What to remove |
|------|---------------|
| `input_boolean.yaml` | All 12 `*_request_*` entries (humidity_request_*, fenster_request_*, temperatur_request_*) |
| `customize.yaml` | All 12 `notify_target` entries (file becomes empty or removable) |
| `alexa.yaml` | All 12 `input_boolean.*_request_*` from `include_entities` and `entity_config` |
| `automations.yaml` | Remove all 3 "Alexa Ansage" automations (Luftfeuchtigkeit, Fenster offen, Außentemperatur) |
| `scripts.yaml` | Remove `alexa_room_announce` script (back to `{}`) |
| `configuration.yaml` | Remove `customize: !include customize.yaml` under `homeassistant:` |
| `templates/configmap.yaml` | Remove `customize.yaml` entry |

**Keep untouched:**
- The `fenster_alexa_*` toggle booleans — these control push announcements in "Fenster zu lange offen", which is a separate HA-initiated flow that still needs `notify.send_message`

## Files to Modify (code changes only)

| File | Change |
|------|--------|
| `zimmermann.lat/homeassistant-config/intent_script.yaml` | **New file** — 3 intent definitions |
| `zimmermann.lat/homeassistant-config/configuration.yaml` | Add `intent_script: !include intent_script.yaml` |
| `zimmermann.lat/homeassistant-config/templates/configmap.yaml` | Add `intent_script.yaml` to ConfigMap |
| `zimmermann.lat/homeassistant-config/input_boolean.yaml` | Remove 12 `*_request_*` entries |
| `zimmermann.lat/homeassistant-config/customize.yaml` | Remove file or empty it |
| `zimmermann.lat/homeassistant-config/alexa.yaml` | Remove 12 boolean entity entries |
| `zimmermann.lat/homeassistant-config/automations.yaml` | Remove 3 "Alexa Ansage" automations |
| `zimmermann.lat/homeassistant-config/scripts.yaml` | Remove `alexa_room_announce` script → `{}` |

## Suggested Rollout Order

1. **Phase 1** (deploy alongside existing): Steps 1-4 — add Custom Skill + intent_script without removing anything
2. **Phase 2** (test): Verify all 3 intents work from each Alexa device
3. **Phase 3** (cleanup): Step 5 — remove the boolean-based approach

## Verification
- Say "Alexa, frage mein Zuhause wie warm es draußen ist" → should announce temperature
- Say "Alexa, frage mein Zuhause wie die Luftfeuchtigkeit ist" → should announce humidity
- Say "Alexa, frage mein Zuhause ob Fenster offen sind" → should announce window status
- Test from each Alexa device to confirm response comes back on the correct device
- Verify the "Fenster zu lange offen" push announcements still work (separate system)

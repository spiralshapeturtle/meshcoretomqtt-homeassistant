# Meshcore SNR Monitoring in Home Assistant via MQTT

Automatically collect and visualize SNR (Signal-to-Noise Ratio) data from Meshcore repeater packets in Home Assistant using MQTT.

<img width="1057" height="535" alt="image" src="https://github.com/user-attachments/assets/0736076f-32e2-4297-8541-3d9f16c0cf35" />


## Overview

This setup listens to Meshcore packets arriving via an MQTT broker (e.g. [letsmesh.net](https://letsmesh.net) MQTT broker 3), filters relevant packet types, and creates individual SNR sensors per contact node in Home Assistant. It includes automatic sensor creation, cleanup of stale sensors, and dashboard cards for visualization.

## Prerequisites

- Home Assistant with the **MQTT integration** configured and a working MQTT broker (e.g. Mosquitto)
- Meshcore packets being published to your broker via MQTT.
- For the dashboard: [custom:auto-entities](https://github.com/thomasloven/lovelace-auto-entities) and [custom:apexcharts-card](https://github.com/RomRider/apexcharts-card) installed via HACS

## Components

| Component | Type | Purpose |
|---|---|---|
| Meshcore - SNR Info Repeater | Automation | Listens to MQTT, filters packets, publishes parsed SNR data |
| Meshcore - SNR | Script | Creates an MQTT auto-discovery sensor for a new contact |
| Meshcore - Cleanup oude SNR sensoren | Automation | Removes sensors with no update in 24 hours |
| Meshcore Cleanup SNR Sensors (Smart) | Script | Manual one-shot cleanup of all meshcore SNR sensors |
| ApexCharts card | Dashboard | Visualizes SNR trends over time |
| Markdown card | Dashboard | Shows a live overview of all received nodes |

---
## 0. MQTT config

This is the feed to fill the MQTT broker with MeshCore data

```
user@server:~/ nano .meshcoretomqtt/.env.local

MCTOMQTT_MQTT3_ENABLED=true
MCTOMQTT_MQTT3_SERVER=192.168.1.x
MCTOMQTT_MQTT3_PORT=1883
MCTOMQTT_MQTT3_TRANSPORT=tcp
MCTOMQTT_MQTT3_USE_TLS=false
MCTOMQTT_MQTT3_USERNAME=username
MCTOMQTT_MQTT3_PASSWORD=password
MCTOMQTT_MQTT3_TOPIC_PREFIX=meshcore
MCTOMQTT_MQTT3_CLIENT_ID=YOURID
```


## 1. Automation: Meshcore - SNR Info Repeater (MQTT)

This is the core automation. It subscribes to `meshcore/UTC/+/packets`, parses the raw hex payload, and republishes structured JSON per contact node.

**Packet filtering logic:**
- The first byte (hex positions 0–1) must be one of `5, 11, 15, 21` (specific Meshcore packet types)
- Hops (hex positions 2–3) must be greater than 0

```yaml
alias: Meshcore - SNR Info Repeater (MQTT)
description: Collect SNR data from MQTT packets (filtered on packet types)
triggers:
  - trigger: mqtt
    topic: meshcore/UTC/+/packets
conditions:
  - condition: template
    value_template: >
      {% set raw = trigger.payload_json.raw | default('') %}
      {% if raw | length < 4 %}
        false
      {% else %}
        {% set first_byte = raw[0:2] | int(base=16) %}
        {% set hops = raw[2:4] | int(base=16) %}
        {{ hops > 0 and first_byte in [5, 11, 15, 21] }}
      {% endif %}
actions:
  - variables:
      raw_hex: "{{ trigger.payload_json.raw }}"
      first_byte: "{{ raw_hex[0:2] | int(base=16) }}"
      hops: "{{ raw_hex[2:4] | int(base=16) }}"
      path: >
        {% set start = (raw_hex[2:4] | int(base=16)) * 2 + 4 %}
        {{ raw_hex[4:start] }}
      contact: >
        {% set start = (raw_hex[2:4] | int(base=16)) * 2 + 4 %}
        {% set p = raw_hex[4:start] %}
        {{ ('x' ~ p[-2:]).lower()[1:] }}
      snr: "{{ trigger.payload_json.SNR | default(0) | float(0) }}"
      rssi: "{{ trigger.payload_json.RSSI | default(0) | int(0) }}"
      packet_type: "{{ trigger.payload_json.packet_type | default('unknown') }}"
      timestamp: "{{ as_timestamp(trigger.payload_json.timestamp) | int }}"
      origin: "{{ trigger.payload_json.origin | default('unknown') }}"
  - action: system_log.write
    data:
      message: >
        Meshcore REPEATER: type={{ first_byte }}, hops={{ hops }},
        contact={{ contact }}, SNR={{ snr }}
      level: info
    enabled: false
  - if:
      - condition: template
        value_template: "{{ states['sensor.meshcore_snr_' ~ contact] is none }}"
    then:
      - action: script.meshcore_snr
        data:
          contact: "{{ contact }}"
  - action: mqtt.publish
    data:
      topic: meshcore/snr/{{ contact }}
      payload: |
        {
          "snr": {{ snr }},
          "rssi": {{ rssi }},
          "path": "{{ path }}",
          "hops": {{ hops }},
          "origin": "{{ origin }}",
          "first_byte": {{ first_byte }},
          "packet_type": "{{ packet_type }}",
          "timestamp": {{ timestamp }}
        }
      qos: 0
      retain: true
mode: queued
max: 20
```

**How it works:**
1. Triggers on any MQTT message on `meshcore/UTC/+/packets`
2. Filters for specific packet types with at least 1 hop
3. Extracts the contact ID from the routing path in the raw hex data
4. If no sensor exists yet for this contact, calls the SNR script to create one via MQTT discovery
5. Publishes a structured JSON payload to `meshcore/snr/<contact>`

---

## 2. Script: Meshcore - SNR (Sensor Auto-Creation)

Creates a Home Assistant sensor via MQTT auto-discovery for each new contact node.

```yaml
alias: Meshcore - SNR
description: Create SNR sensor for each contact
fields:
  contact:
    selector:
      text: null
    name: contact
    required: true
sequence:
  - action: mqtt.publish
    data:
      topic: homeassistant/sensor/mc-{{ contact }}/snr/config
      retain: true
      payload: >-
        {
          "name": "Meshcore SNR {{ contact }}",
          "unique_id": "meshcore_snr_{{ contact }}",
          "state_topic": "meshcore/snr/{{ contact }}",
          "state_class": "measurement",
          "unit_of_measurement": "dB",
          "device_class": "signal_strength",
          "value_template": "{% raw %}{{ value_json.snr }}{% endraw %}",
          "icon": "mdi:signal",
          "json_attributes_topic": "meshcore/snr/{{ contact }}"
        }
      qos: 0
mode: single
```

---

## 3. Automation: Cleanup Stale SNR Sensors

Runs every 2 hours and removes sensors that haven't been updated in the last 24 hours.

```yaml
alias: Meshcore - Cleanup oude SNR sensoren
description: Remove SNR sensors that haven't been updated in 24 hours
triggers:
  - trigger: time_pattern
    hours: /2
    minutes: "0"
actions:
  - variables:
      max_age_hours: 24
      cutoff_ts: "{{ (now().timestamp() - (max_age_hours * 3600)) }}"
      sensors_to_remove: |
        {% set ns = namespace(sensors=[]) %}
        {% for sensor in states.sensor
           | selectattr('entity_id', 'match', 'sensor.meshcore_snr_[a-f0-9]+$') %}
          {% set last_update_ts = sensor.last_updated.timestamp() %}
          {% if last_update_ts < (cutoff_ts | float) %}
            {% set ns.sensors = ns.sensors + [sensor.entity_id] %}
          {% endif %}
        {% endfor %}
        {{ ns.sensors }}
  - condition: template
    value_template: "{{ sensors_to_remove | length > 0 }}"
  - action: system_log.write
    data:
      message: >
        Meshcore Cleanup: Removing {{ sensors_to_remove | length }} sensors
        older than {{ max_age_hours }}h
      level: warning
  - repeat:
      for_each: "{{ sensors_to_remove }}"
      sequence:
        - variables:
            node_id: "{{ repeat.item.split('_')[-1] }}"
        - action: mqtt.publish
          data:
            topic: homeassistant/sensor/mc-{{ node_id }}/snr/config
            payload: ""
            retain: true
        - action: mqtt.publish
          data:
            topic: meshcore/snr/{{ node_id }}
            payload: ""
            retain: true
        - delay:
            milliseconds: 50
  - action: persistent_notification.create
    data:
      title: Meshcore Cleanup
      message: >
        ✓ Removed: {{ sensors_to_remove | length }} stale SNR sensor(s)
        Nodes: {{ sensors_to_remove | map('replace', 'sensor.meshcore_snr_', '') | join(', ') }}
mode: single
max_exceeded: silent
```

---

## 4. Script: Manual Full Cleanup

Use this script to manually remove **all** meshcore SNR sensors at once (useful for testing or resetting).

```yaml
alias: Meshcore Cleanup SNR Sensors (Smart)
description: Remove all mc- sensors without a manual list
sequence:
  - repeat:
      for_each: |
        {{ states.sensor
          | selectattr('entity_id', 'search', 'meshcore_snr_')
          | rejectattr('entity_id', 'search', 'repeater')
          | map(attribute='entity_id') | list }}
      sequence:
        - action: mqtt.publish
          data:
            topic: >
              {% set id = repeat.item.split('meshcore_snr_')[-1] %}
              homeassistant/sensor/mc-{{ id }}/snr/config
            payload: ""
            retain: true
mode: single
```

---

## 5. Dashboard Cards

### ApexCharts – SNR Trend (48h, 60min average)

Requires [apexcharts-card](https://github.com/RomRider/apexcharts-card) and [auto-entities](https://github.com/thomasloven/lovelace-auto-entities).

> **Note:** Change the `origin` filter value to match your own repeater/node name.

```yaml
type: custom:auto-entities
filter:
  include:
    - entity_id: sensor.meshcore_snr_*
      attributes:
        origin: <<<<YOUR REPEATER NAME>>>>
  exclude:
    - entity_id: "*repeater*"
card:
  type: custom:apexcharts-card
  header:
    show: true
    title: SNR Trend (60 min avg.)
    show_states: false
  graph_span: 48h
  update_interval: 5min
  yaxis:
    - min: -10
      max: 15
      decimals: 0
  all_series_config:
    stroke_width: 2
    curve: smooth
    group_by:
      duration: 60min
      func: avg
  apex_config:
    chart:
      height: 450px
    legend:
      show: true
      position: bottom
      floating: false
      fontSize: 10px
  series_in_legend: order_by_value_desc
```

### Markdown – Live Node Overview

```yaml
type: markdown
title: RX SNR Received from Repeaters
content: >
  {% set DEVICES = states.sensor
    | selectattr('entity_id', 'match', 'sensor.meshcore_snr_')
    | selectattr('attributes.timestamp', 'defined')
    | sort(attribute='attributes.timestamp', reverse=true)
  %}
  {% for sensor in DEVICES %}
  **{{ sensor.attributes.friendly_name | default(sensor.name) }}**
  Via: {{ sensor.attributes.origin }} | Hops: {{ sensor.attributes.hops }}
  SNR: {{ sensor.attributes.snr }} dB | RSSI: {{ sensor.attributes.rssi }} dBm
  Last: {{ sensor.attributes.timestamp | int | timestamp_custom('%H:%M:%S') }}
  Route: {% set path = sensor.attributes.path %}{% for i in range(0, path|length, 2) %}{{ path[i:i+2] }}{% if not loop.last %} → {% endif %}{% endfor %}
  {% endfor %}
```

---

## Notes

- Sensors are created dynamically — no manual configuration needed per node.
- Stale sensors are automatically cleaned up after 24 hours of inactivity.
- The `contact` ID is derived from the last byte of the routing path in the raw hex payload.
- All data is retained on MQTT so sensors restore after a Home Assistant restart.
- Adjust the `origin` filter in the ApexCharts card to match your repeater name(s).

## License

Feel free to use, modify, and share.


# HA-CF-Relay-Client

Client integrations and web monitoring dashboard for the [HA-CF-Relay](https://github.com/maxrhys/HA-CF-Relay) Cloudflare Worker edge broker.

This repository contains:
1. **Home Assistant Integrations**: Configurations and automations to securely batch and push arbitrary local sensor telemetry out to the edge broker via `POST /update`.
2. **Web Client Dashboard**: A responsive, zero-dependency HTML/CSS/JS telemetry monitor that queries the edge broker via `GET /data`.

---

## Directory Structure

```
.
├── home-assistant/
│   ├── configuration.yaml       # rest_command.push_telemetry service definition
│   └── automations.yaml         # 5-minute batched push automation
├── web/
│   └── index.html               # Responsive web telemetry dashboard
├── .github/
│   └── workflows/
│       └── deploy-pages.yml     # Automated GitHub Pages hosting for the web dashboard
├── .gitignore
└── README.md
```

---

## 1. Home Assistant Setup

### Step A: Add REST Command
Add the contents of [home-assistant/configuration.yaml](home-assistant/configuration.yaml) into your Home Assistant `configuration.yaml`:

```yaml
rest_command:
  push_telemetry:
    url: "https://ha-cf-relay.<YOUR_SUBDOMAIN>.workers.dev/update"
    method: POST
    headers:
      Authorization: "Bearer YOUR_RANDOM_SECRET_TOKEN"
      Content-Type: "application/json"
    payload: "{{ devices | to_json }}"
```
* Replace `<YOUR_SUBDOMAIN>` with your Cloudflare Workers subdomain.
* Replace `YOUR_RANDOM_SECRET_TOKEN` with the `SECRET_TOKEN` secret set on your Worker.

### Step B: Add Telemetry Automation
Add the contents of [home-assistant/automations.yaml](home-assistant/automations.yaml) to your `automations.yaml`:

```yaml
alias: "Push Telemetry to HA-CF-Relay (5m Interval)"
description: "Bundles environment & unit telemetry and writes a single update to HA-CF-Relay"
trigger:
  - platform: time_pattern
    minutes: "/5"
action:
  - service: rest_command.push_telemetry
    data:
      devices:
        unit_alpha:
          name: "Unit Alpha"
          updated_at: "{{ now().isoformat() }}"
          sensors:
            temperature:
              value: "{{ states('sensor.unit_alpha_temperature') | float(0) }}"
              unit: "{{ state_attr('sensor.unit_alpha_temperature', 'unit_of_measurement') | default('°C') }}"
            humidity:
              value: "{{ states('sensor.unit_alpha_humidity') | float(0) }}"
              unit: "{{ state_attr('sensor.unit_alpha_humidity', 'unit_of_measurement') | default('%') }}"
mode: single
```

Customize device names and entity IDs (`sensor.unit_alpha_temperature`, etc.) to match your real devices.

### Step C: Reload Home Assistant
Navigate to **Developer Tools > YAML** in Home Assistant and click **Quick Reload**.

---

## 2. Web Client Dashboard

The dashboard in [web/index.html](web/index.html) is a responsive, dark-mode monitor that renders real-time device cards and flags stale telemetry (> 10 minutes old).

### Configuration
Edit line 43 in [web/index.html](web/index.html):
```javascript
const API_ENDPOINT = "https://ha-cf-relay.<YOUR_SUBDOMAIN>.workers.dev/data";
```

### Hosting on GitHub Pages
This repository includes a preconfigured GitHub Actions workflow ([deploy-pages.yml](.github/workflows/deploy-pages.yml)):
1. In this GitHub repository, go to **Settings > Pages**.
2. Under **Build and deployment > Source**, select **GitHub Actions**.
3. Pushing any changes to `web/` will automatically build and publish your dashboard at:
   `https://maxrhys.github.io/HA-CF-Relay-Client/`

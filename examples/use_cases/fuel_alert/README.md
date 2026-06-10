# Use Case: Fuel Level Alerts

This use case demonstrates how to use the spoken alerts solution to notify the driver about fuel level and recommend the best fuel station.

## Overview

Two versions are provided:

1. **Simple version** ([`automation_simple.yaml`](automation_simple.yaml)): Just announces fuel level thresholds via TTS
2. **Advanced version** ([`automation_with_station.yaml`](automation_with_station.yaml)): Searches for the best station and sends route to car navigation

## Files

| File | Description |
|------|-------------|
| [`automation_simple.yaml`](automation_simple.yaml) | Multi-threshold fuel alerts (30%, 20%, 10%) with simple TTS announcements |
| [`automation_with_station.yaml`](automation_with_station.yaml) | Advanced: fuel alerts + automatic station search + route sending |
| [`find_best_fuel_station.yaml`](find_best_fuel_station.yaml) | Script to find best fuel station and send route to car (called by `automation_with_station.yaml`) |

## Requirements

### For both versions:
- Core TTS solution configured ([`../../core/`](../../core/))
- A fuel level sensor in Home Assistant (replace `YOUR_FUEL_LEVEL_SENSOR` in the files)

### Additional for the advanced version:
- **France only**: [prix_carburant](https://github.com/Aohzan/hass-prixcarburant) custom integration by Aohzan (HACS)
- **Mercedes-Benz**: `mbapi2020` integration to send routes to car navigation
- A person entity with location tracking (replace `person.DRIVER` in the files)

## Adapting for other countries

The advanced version relies on the French government's open data for fuel prices. To adapt for other countries:

| Country | Integration / API |
|---------|-------------------|
| France 🇫🇷 | [prix_carburant](https://github.com/Aohzan/hass-prixcarburant) (used here) |
| Germany 🇩🇪 | [tankerkoenig](https://www.home-assistant.io/integrations/tankerkoenig/) (official HA integration) |
| UK 🇬🇧 | petrolprices (unofficial APIs) |
| Spain 🇪🇸 | gasolineras (geoportalgasolineras.es) |
| Italy 🇮🇹 | osservaprezzi (mimit.gov.it) |

The general structure remains the same: find station → announce via TTS → send route.

## Adapting for other vehicles

The advanced version uses `mbapi2020.send_route` for Mercedes-Benz vehicles. Replace with your car's integration:

- **Mercedes-Benz**: `mbapi2020.send_route` ✅ (used here)
- **BMW**: `bmw_connected_drive.send_poi`
- **Audi/VW**: `audi_connect.send_destination`
- **Tesla**: `tesla_custom.set_navigation`
- **Other vehicles**: Check available integrations or remove the route sending step

## Usage example

From Developer Tools → Services:

```yaml
action: script.find_best_fuel_station
data:
  strategy: price    # or 'distance'
  radius: 30         # search radius in km
```

This will:
1. Search nearby fuel stations within 30km
2. Select the cheapest one (or closest if `strategy: distance`)
3. Announce it via TTS in the car
4. Send the route to car navigation

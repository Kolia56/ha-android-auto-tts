# Vehicle Compatibility Database

This document collects user reports on how the TTS on Android Auto solution behaves with different vehicles. Android Auto implementations vary by car manufacturer, so community feedback helps build a useful compatibility reference.

## How to Contribute

If you've tested this solution on your vehicle, please add your entry to the table below by:

1. Forking this repository
2. Adding a row to the table with your test results
3. Opening a Pull Request

Alternatively, open an issue with your test results and we'll add it for you.

## What to Report

For each test, please provide:

- **Vehicle**: Make, model, year (e.g., "Mercedes A-Class 2021")
- **Connection**: Wired USB / Wireless (Bluetooth + WiFi)
- **Phone**: Manufacturer and model (e.g., "Nokia XR20")
- **Android version**: e.g., "Android 14"
- **HA version**: e.g., "HAOS 2026.1"
- **TTS works**: ✅ Yes / ❌ No / ⚠️ Partially
- **Audio routing**: Where does the TTS sound come out? (Car speakers / Phone speaker / Both)
- **Previous source**: Did your previous audio source (radio, Spotify, etc.) auto-resume after TTS?
- **Notes**: Any specific behaviors, quirks, or workarounds

## Compatibility Table

| Vehicle | Year | Connection | Phone | Android | HA Version | TTS Works | Audio Routing | Auto-Resume | Contributor | Notes |
|---------|------|------------|-------|---------|------------|-----------|---------------|-------------|-------------|-------|
| Mercedes-Benz A-Class | 2020+ | Wireless (Bluetooth) | Nokia XR20 | 14 | 2026.1 | ✅ Yes | Car speakers | ❌ No | @Kolia56 | Initial implementation. Radio doesn't auto-resume after TTS, manual tap needed. |

## Known Patterns by Manufacturer

This section will be updated as more reports come in to identify patterns specific to each manufacturer.

### Mercedes-Benz
- Wireless Android Auto connection works
- Audio routing to car speakers: ✅ Working
- Previous source auto-resume: ❌ Not working (manual reactivation required)

### [Other Manufacturers]
*Awaiting community contributions*

## Common Issues by Vehicle Type

### Vehicles where TTS plays on phone instead of car speakers
*None reported yet*

### Vehicles where TTS doesn't trigger VLC at all
*None reported yet*

### Vehicles with successful auto-resume of previous audio source
*None reported yet - if you find one, please report!*

## Template for New Entries

Copy and paste this template, fill in your information, and submit:

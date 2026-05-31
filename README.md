# [SOLUTION] Text-to-Speech on Android Auto via VLC and Piper TTS

**Objective:** Implement text-to-speech notifications that play through car speakers when connected to Android Auto, using a local TTS engine (Piper) and VLC, without relying on cloud services or Google Assistant.

**Tested Setup:**
- Home Assistant OS 2026.1
- Nokia XR20 (Android 14)
- Mercedes-Benz with **wireless** Android Auto (Bluetooth connection)
- Piper TTS with French voice (`fr_FR-siwis-medium`)

---

## Prerequisites

- Home Assistant with external URL access (DuckDNS or Nabu Casa)
- VLC for Android installed on phone
- Android Auto configured and working in your vehicle

---

## Architecture Overview

The solution works as follows:

1. **Home Assistant** calls an API endpoint to generate TTS audio using Piper
2. **Shell command** captures the generated file path
3. **Notification** is sent to the phone with a VLC intent
4. **VLC** opens and streams the audio file from HA
5. **Android Auto** routes VLC's audio to car speakers

**Key insight:** Using VLC with `command_activity` intent ensures Android Auto treats it as a media source and routes audio to car speakers, unlike native Android TTS which plays on phone speaker.

---

## Step-by-Step Installation with Validation Tests

### Step 1: Install Piper TTS Add-on

**Installation:**
1. Go to **Settings → Add-ons → Add-on Store**
2. Search for "**Piper**"
3. Click **Install**
4. Once installed, click **Start**
5. Enable "**Start on boot**"

**Configuration** (Add-on → Configuration tab):
- **Voice:** Select your preferred voice (e.g., `fr_FR-siwis-medium` for French, `en_US-lessac-medium` for English)
- Click **Save**
- **Restart** the add-on

**✓ Validation Test 1:**

Check add-on logs (**Log** tab). You should see:
```
INFO:__main__:Ready
INFO: Successfully send discovery information to Home Assistant
```

If you see errors about voice download, wait a few minutes for the voice model to download.

---

### Step 2: Add Wyoming Protocol Integration

**Installation:**
1. Go to **Settings → Devices & Services**
2. Click **Add Integration**
3. Search for "**Wyoming Protocol**"
4. It should **auto-discover** Piper running on `localhost:10200`
5. Click **Configure** and complete setup

**If auto-discovery fails:**
- Add manually with Host: `localhost`, Port: `10200`

**✓ Validation Test 2:**

Go to **Developer Tools → States** and search for `tts.piper`

**Expected result:** Entity exists with a timestamp state (not "unavailable")

If state shows "unavailable":
1. Check that Piper add-on is running
2. Try removing and re-adding the Wyoming Protocol integration
3. Restart Home Assistant

---

### Step 3: Enable Required Phone Permissions

**On your Android phone:**

1. Open **Settings → Apps → Home Assistant**
2. Go to **Special App Access** (or **Advanced**)
3. Find and tap **Display over other apps**
4. **Enable** for Home Assistant

**Why this is needed:** This permission allows Home Assistant to trigger VLC to open and play audio even when the phone screen is locked or another app is active.

**✓ Validation Test 3:**

From Home Assistant (**Developer Tools → Services**), send this test notification:

```yaml
service: notify.mobile_app_YOUR_PHONE
data:
  message: command_activity
  data:
    intent_package_name: org.videolan.vlc
    intent_action: android.intent.action.VIEW
    intent_uri: https://www.soundhelix.com/examples/mp3/SoundHelix-Song-1.mp3
```

Replace `YOUR_PHONE` with your phone's entity name (e.g., `nokia_xr20`).

**Expected result:** VLC should open on your phone and play music from the URL.

If VLC doesn't open:
- Verify VLC is installed
- Check the "Display over other apps" permission again
- Try restarting the Home Assistant Companion app

---

### Step 4: Create Long-Lived Access Token

**Why needed:** The TTS audio file URL needs authentication when accessed from outside your local network (4G/5G in car).

**Creation:**
1. Click your **profile** (bottom left in HA interface)
2. Scroll to **Long-Lived Access Tokens**
3. Click **Create Token**
4. Name it "**TTS Token**"
5. **Copy the token immediately** (you won't see it again!)

**Store in secrets.yaml:**

```yaml
tts_token: YOUR_COMPLETE_TOKEN_HERE
```

Replace `YOUR_COMPLETE_TOKEN_HERE` with your entire token (starting with `eyJhbGci...`).

**Example (this is NOT a real token):**
```yaml
tts_token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiIwOWJmYmYyN2MwMzQ0MTQ3OTQ2ZDgxYTAzMzFjNjRkMCIsImlhdCI6MTcxMjg1...
```

**Security note:** This token grants full access to your HA instance. Store it securely and never share it publicly.

---

### Step 5: Create Shell Command for TTS API

The shell command calls Home Assistant's TTS API to generate audio and returns the file path.

**In `shell_commands.yaml`:**

```yaml
tts_api: 'curl -X POST -H "Authorization: Bearer YOUR_TOKEN_HERE" -H "Content-Type: application/json" -d ''{"message": "{{mymessage}}", "platform": "tts.piper", "language": "fr_FR"}'' http://localhost:8123/api/tts_get_url'
```

**Important replacements:**
- Replace `YOUR_TOKEN_HERE` with your actual long-lived token
- Change `fr_FR` to your language code (`en_US`, `de_DE`, etc.)
- This must be a **single line** with no line breaks

**Note:** Using `!secret tts_token` reference inside the curl command **has not been validated**. For reliable operation, paste the actual token directly in the shell_command. If you prefer to use secrets.yaml for this token, please test thoroughly first.

**Restart Home Assistant** (required for shell_command changes)

**✓ Validation Test 4:**

**Developer Tools → Services:**

```yaml
service: shell_command.tts_api
data:
  mymessage: Test message
```

**Check logs** (**Settings → System → Logs**), search for "shell_command"

**Expected result:** JSON response with path:
```json
{
  "url":"http://192.168.x.x:8123/api/tts_proxy/RANDOM_HASH.mp3",
  "path":"/api/tts_proxy/RANDOM_HASH.mp3"
}
```

**Common errors:**
- **Error code 3:** Check token format (no spaces, correct quotes)
- **401 Unauthorized:** Token is invalid or expired
- **500 Internal Server Error:** Check platform name is `tts.piper` (not just `piper`)

---

### Step 6: Create the TTS Script

This is the main script that ties everything together.

**In `scripts.yaml`:**

```yaml
tts_android_auto_vlc:
  alias: TTS Android Auto VLC
  sequence:
    # Generate TTS audio via API
    - service: shell_command.tts_api
      data:
        mymessage: "{{ message }}"
      response_variable: json_data
    
    # Extract the file path from JSON response
    - variables:
        tts_path: >
          {% set parsed_data = json_data['stdout'] | from_json %}
          {{ parsed_data.path }}
    
    # Send VLC intent to phone with TTS file URL
    - service: notify.mobile_app_YOUR_PHONE
      data:
        message: command_activity
        data:
          intent_package_name: org.videolan.vlc
          intent_action: android.intent.action.VIEW
          intent_uri: https://YOUR_EXTERNAL_URL:8123{{ tts_path }}?access_token=YOUR_TOKEN
  
  fields:
    message:
      selector:
        text: null
      name: Message
      required: true
  mode: single
  icon: mdi:car-wireless
```

**Critical replacements:**
- `YOUR_PHONE`: Your phone entity name (e.g., `nokia_xr20`)
- `YOUR_EXTERNAL_URL`: Your DuckDNS domain (e.g., `myha.duckdns.org`) or Nabu Casa URL
- `YOUR_TOKEN`: Your long-lived access token (same as in shell_command)

**Note:** Do NOT include `http://` in YOUR_EXTERNAL_URL, just the domain.

**Reload scripts:** **Developer Tools → YAML → Reload Scripts**

**✓ Validation Test 5** (at home, phone on WiFi):

```yaml
service: script.tts_android_auto_vlc
data:
  message: Test TTS message in your language
```

**Expected result:**
- VLC opens on phone
- Message is spoken in your configured language (French, English, etc.)
- Audio plays on phone speaker

If nothing happens:
1. Check logs for errors in the script execution
2. Verify the shell_command is returning valid JSON (Test 4)
3. Test the generated URL manually in a browser

---

### Step 7: Final Test in Car with Android Auto

This is the critical test to verify audio routes to car speakers.

**Setup:**
1. Connect phone to Android Auto (USB or wireless)
2. Start Android Auto interface on car display
3. Play radio or music (to verify interruption behavior)

**Execute from phone or PC:**

```yaml
service: script.tts_android_auto_vlc
data:
  message: Test TTS message for Android Auto
```

**Expected behavior:**
- VLC opens (may show briefly on Android Auto display)
- TTS message plays through **car speakers** ✅
- **Your previous audio source is completely replaced** (FM radio stops, Spotify stops, etc.)
- VLC becomes the active Android Auto media source
- VLC may show "Unknown Title / Unknown Artist" briefly
- After TTS finishes, you must **manually reactivate your audio source**:
  - For Android Auto apps (Spotify, etc.): Tap the app icon
  - For FM/DAB radio: Press the car's radio button
  - For USB/AUX: Press source button and reselect
- **This is not a pause/resume - it's a complete source switch**

**Troubleshooting:**
- **TTS plays on phone speaker, not car:** Ensure Android Auto is fully active and displaying on car screen
- **No sound at all:** Check car volume, verify Android Auto is the active audio source
- **VLC doesn't open:** Re-check "Display over other apps" permission

---

## Usage in Automations

### Example 1: Low Fuel Alert

```yaml
automation:
  - alias: "Car Fuel Low Alert"
    trigger:
      - platform: numeric_state
        entity_id: sensor.car_fuel_level
        below: 30
    condition:
      - condition: state
        entity_id: binary_sensor.phone_android_auto
        state: 'on'
    action:
      - service: script.tts_android_auto_vlc
        data:
          message: "Fuel level is below thirty percent"
```

### Example 2: Navigation Reminder

```yaml
automation:
  - alias: "Navigation Reminder"
    trigger:
      - platform: state
        entity_id: person.you
        to: 'not_home'
    condition:
      - condition: state
        entity_id: binary_sensor.phone_android_auto
        state: 'on'
    action:
      - service: script.tts_android_auto_vlc
        data:
          message: "Don't forget to activate navigation mode"
```

---

## Important Notes on Vehicle Compatibility

### ⚠️ Vehicle-Specific Behavior

This solution's behavior **may vary** depending on your car's Android Auto implementation. Different manufacturers handle audio focus and source switching differently.

**Confirmed working on:**
- Mercedes-Benz (2020+) with wireless Android Auto (Bluetooth)

**What may vary by vehicle:**
- How quickly Android Auto switches between audio sources
- Whether VLC appears as a full-screen app or background player
- Audio interruption behavior when VLC starts
- Bluetooth vs USB connection differences
- Whether native car sources (FM radio, USB) can be resumed programmatically (most cannot)
- How the car's infotainment system handles Android Auto audio focus
- Whether "back" or "home" buttons on car interface affect audio routing

**Community feedback needed:** If you test this on a different vehicle, please share:
- Car make/model/year
- Android Auto connection type (wired/wireless)
- What audio source you were using before TTS (FM radio, Spotify, USB, etc.)
- Whether that source auto-resumed or required manual reactivation
- If manual reactivation needed, what method (tap icon, press button, etc.)
- Any specific behaviors or issues encountered

---

## Known Limitations

### 1. No Automatic Audio Source Restoration (Critical Limitation)

**Issue:** When TTS finishes, the **previous audio source is not automatically restored**. This requires manual intervention to resume your audio.

**Understanding the problem:** This is not a simple pause/resume issue - it's a complete **audio source/channel switch**.

**Before TTS, you could be listening to:**
- Car's native FM/AM radio 📻
- DAB/DAB+ digital radio
- Satellite radio (SiriusXM, etc.)
- Android Auto apps (Spotify, YouTube Music, Pocket Casts, etc.)
- TuneIn or other internet radio
- USB music playback
- Bluetooth audio from phone
- Built-in car media player
- CD player (if your car still has one!)

**During TTS:** Android Auto switches to VLC as the active media source

**After TTS:** Android Auto remains on VLC or goes to neutral state - **your previous source is NOT restored**

**Current behavior:**
- TTS plays through car speakers ✅
- Previous audio source is **completely replaced** (not paused)
- User must **manually reactivate** their previous audio source:
  - Tap app icon in Android Auto (for Spotify, podcasts, etc.)
  - Press physical radio button (for FM/DAB radio)
  - Press source button and reselect (for USB, AUX, etc.)

**Why this cannot be easily solved:**

1. **No "previous source" API:** Android Auto provides no API to:
   - Detect which audio source was active before TTS
   - Store the previous source state
   - Programmatically restore any audio source

2. **Diverse source types require different restoration methods:**
   - **Native car sources** (FM radio, DAB, USB): No Android control possible - requires physical button press or car-specific CAN bus commands
   - **Android Auto apps** (Spotify, etc.): Each requires app-specific intent - would need to detect which app was active
   - **Bluetooth A2DP**: Different from Android Auto media routing
   - **Built-in car systems**: Completely outside Android ecosystem

3. **Source detection is unreliable:**
   - Android Auto doesn't expose "currently active source" state
   - Cannot distinguish between Spotify via Android Auto vs USB playback vs FM radio
   - Tasker/MacroDroid can only detect Android apps, not native car sources

**Attempted solutions that failed:**

**A) Generic MEDIA_PLAY command:**
```yaml
# Sends "play" to Android
intent_action: android.intent.action.MEDIA_BUTTON
```
Result: Plays notification sound only - Android doesn't know which source to resume

**B) Automation apps (MacroDroid/Tasker):**
- Detect VLC close and send "play" command
- Result: Created state conflicts, unreliable, cannot handle native car sources

**C) Storing last active app:**
- Would only work for Android Auto apps
- Cannot handle FM radio, USB, DAB, or car-native sources
- Would require complex state management

**Theoretical solutions (not implemented):**

**Option 1:** Custom Android app that:
- Monitors active audio source continuously
- Stores source state before VLC launches  
- Sends source-specific restore command after TTS
- **Problem:** Would require app development, cannot control native car sources, battery drain

**Option 2:** Car-specific CAN bus integration:
- Send direct commands to car audio system
- **Problem:** Requires reverse engineering each car model, hardware interface, major complexity

**Option 3:** Multiple scripts for each source type:
```yaml
script.tts_android_auto_for_spotify
script.tts_android_auto_for_radio
script.tts_android_auto_for_usb
```
Then user manually selects appropriate script
- **Problem:** User must know and remember which source they're using - defeats automation purpose

**Is this limitation acceptable?**

**For critical/infrequent notifications:** YES
- Fuel low warning: One manual tap to resume radio is acceptable
- Door open alert: Safety takes priority over convenience  
- Navigation reminder: Occasional interruption is reasonable

**For frequent notifications:** PROBABLY NOT
- Every turn-by-turn navigation instruction: Too disruptive
- Frequent status updates: Would become very annoying
- Non-critical reminders: Use phone notifications instead

**Recommended use cases:**
- ✅ Safety alerts (low fuel, tire pressure, door ajar)
- ✅ Critical reminders (medication, pickup someone)
- ✅ Important status changes (garage door left open at night)
- ❌ Turn-by-turn navigation (use Android Auto's native nav)
- ❌ Frequent updates (package delivered, weather changes)
- ❌ Entertainment notifications (podcast episode available)

**This is a fundamental limitation of Android Auto's architecture**, not a bug in this implementation. A true solution would require either:
- Google adding "previous source" API to Android Auto (unlikely)
- Complete custom Android Auto replacement (massive undertaking)
- Car manufacturer integration via proprietary protocols (car-specific)

### 2. Code PIN Request on First Use

**Issue:** First TTS call of a driving session may request Home Assistant authentication code.

**Frequency:** Usually only once per session, subsequent calls work without PIN.

**Why:** Initial authentication to access the external TTS file URL.

**Workaround:** This is for accessing HA while using the phone interface in the car. The TTS script itself does not require PIN input - audio plays automatically.

### 3. Message Timing

**Current implementation:** Script completes immediately after sending the VLC intent. It does not wait for TTS to finish playing.

**Impact:** If you need to chain actions after TTS (e.g., send follow-up message), you must add manual delays based on estimated message length.

**Possible improvement:** Add calculated delay based on message character count (~10 chars/second + buffer).

### 4. VLC Visual Display

**Cosmetic issue:** When TTS plays, Android Auto may briefly show VLC interface displaying:
- "Unknown Title"
- "Unknown Artist"
- Phone icon as source

**Impact:** Visual only, does not affect audio functionality. Display typically reverts to Android Auto home screen within seconds.

### 5. Network Dependency

**Critical:** Phone must have network access (WiFi or 4G/5G) to download TTS audio file from Home Assistant external URL.

**Impact:** Solution will not work in areas with no phone signal unless you implement a local-only fallback using phone's IP when on home WiFi.

---

## Troubleshooting Guide

### VLC doesn't open when script is called

**Cause:** Permission issue

**Solution:**
1. Settings → Apps → Home Assistant → Special App Access
2. Enable "Display over other apps"
3. Restart Home Assistant Companion app
4. Test again

---

### TTS plays on phone speaker, not car speakers

**Cause:** Android Auto not properly active or not recognizing VLC as media source

**Solutions:**
1. Ensure Android Auto is fully launched and displaying on car screen (not just connected)
2. Verify phone is not in battery saving mode (can restrict background activity)
3. Try with car in Park vs Driving (some implementations differ)
4. Test with USB connection if using wireless, or vice versa

---

### "Script does not support response_variable"

**Cause:** Home Assistant version too old

**Solution:**
- Update to Home Assistant 2024.4 or newer
- `response_variable` feature was added in this version

---

### Shell command returns "Error code 3"

**Cause:** Malformed shell command (typically quote/escape issues)

**Solution:**
1. Verify shell_command is a **single line** (no line breaks)
2. Check token has no spaces or special characters that need escaping
3. Ensure double single-quotes `''` around JSON payload
4. Test token directly: `curl -H "Authorization: Bearer YOUR_TOKEN" http://localhost:8123/api/` should return 200

---

### "500 Internal Server Error" from TTS API

**Causes:**
- Incorrect platform name
- Piper not running
- Language code not supported by selected voice

**Solutions:**
1. Verify platform is `tts.piper` (not `piper`)
2. Check Piper add-on is running (Settings → Add-ons)
3. Verify language code matches your configured voice
4. Check Piper logs for errors

---

### JSON parsing error in script

**Cause:** Shell command not returning valid JSON

**Solution:**
1. Run Validation Test 4 and check logs for exact output
2. If you see curl progress bars mixed with JSON, redirect stderr: add `2>/dev/null` to end of curl command
3. Verify the `from_json` filter is receiving clean JSON string

---

### Audio plays but cuts off early

**Cause:** VLC closes before TTS finishes (rare)

**Solution:**
- This is typically a phone-specific issue
- Disable battery optimization for VLC
- Ensure VLC has "Run in background" permission

---

## Alternative Approaches Not Pursued

For reference, other approaches were considered but not implemented:

### 1. Native Android TTS Engine
**Method:** Using `tts_text` parameter in notification with `media_stream`

**Reason not used:** Always plays on phone speaker, never routes to Android Auto regardless of `media_stream` setting.

### 2. Google Assistant Broadcast
**Method:** Using `command_broadcast` with Google Assistant

**Reason not used:** Requires Google services, violates privacy goal of using local-only TTS.

### 3. Bluetooth A2DP Direct
**Method:** Playing TTS directly via Bluetooth profile, bypassing Android Auto

**Reason not used:** Would require switching connection modes, defeating the purpose of Android Auto integration.

### 4. Custom Android Auto App
**Method:** Developing a dedicated Android Auto messaging app

**Reason not used:** Requires:
- Android app development expertise
- Google Play Console account
- Android Auto compatibility approval from Google
- Distribution and updates

### 5. Notification Read-Aloud
**Method:** Using Android Auto's built-in notification reading feature

**Reason not used:** Notification reading is unreliable and inconsistent across vehicles. Many implementations only read messaging app notifications, not custom ones.

---

## Why This Solution Works

The VLC + Piper approach was chosen because it:

✅ **No cloud dependency:** All TTS processing happens locally on your HA server  
✅ **Works with existing Android Auto:** No special setup or custom apps needed  
✅ **Privacy-friendly:** No data sent to Google, Amazon, or other cloud services  
✅ **No root required:** Works on stock Android  
✅ **Reliable audio routing:** VLC is recognized by Android Auto as a media source  
✅ **Flexible:** Can use any Piper voice in any supported language  
✅ **Cost-effective:** All components are free and open source  

**The key insight:** By using VLC's media player intent, Android Auto treats the TTS as legitimate media content and properly routes it to car speakers, unlike notification-based TTS which plays on phone speaker only.

---

## File Cleanup Checklist

After successful implementation, keep only these files:

**Keep in `shell_commands.yaml`:**
```yaml
tts_api: 'curl -X POST ...' # Only this line
```

**Keep in `scripts.yaml`:**
```yaml
tts_android_auto_vlc:  # Only this script
  ...
```

**Remove from `rest_commands.yaml`** (if you have it):
- Any `generate_tts_file` or similar test commands

**Remove from `configuration.yaml`** (if you added them):
- `media_player:` with `platform: vlc` (if created for testing)
- Any test `rest_command:` entries

**Remove from `/config/www/`:**
- Any test MP3 files (e.g., `test_vocal.mp3`)

**Add-ons to keep:**
- Piper (required) ✅

**Integrations to keep:**
- Wyoming Protocol (required) ✅

---

## Credits and Community

This solution was developed through extensive testing and community collaboration.

**Key components:**
- **Piper TTS:** Local text-to-speech engine by Rhasspy
- **Wyoming Protocol:** Communication protocol for voice services
- **VLC for Android:** Media player that properly integrates with Android Auto
- **Home Assistant:** Home automation platform

**Inspired by:** Community discussions on Android Auto TTS integration, particularly the VLC intent approach shared by FPro in the Home Assistant forums.

**Testing contributions:** Extensive real-world testing on Mercedes-Benz vehicles with various Android phone models.

---

## Further Improvements

Potential enhancements for future versions:

1. **Automatic duration detection:** Calculate TTS length based on character count to enable action chaining
2. **Fallback handling:** Detect network availability and fallback to phone speaker if external URL unreachable
3. **Multi-language support:** Dynamic language selection based on context or time of day
4. **Priority levels:** Different notification behaviors for critical vs informational messages
5. **Volume control:** Pre-set volume level before TTS plays

**Community contributions welcome!** If you implement improvements, please share in the forum thread.

---

## Version History

- **v1.0 (April 2026):** Initial working solution
  - Piper TTS integration
  - VLC intent method
  - External URL with access token
  - Tested on Mercedes-Benz with Android 14, HAOS 2026.1

---

**Questions? Issues? Improvements?** Please share your experience in this thread, especially if you test on different vehicle makes/models!

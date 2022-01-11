
# Kitty Litter Box Tracker

This project senses when the cat has used the litter box. If detected, it turns the light in the bathroom where the litter box is kept to red. It also sends me a push notification if I am at home and not asleep. Otherwise, it sends the notification when I arrive home or when I get up in the morning.

When I clean the litter box, the light turns green briefly, so I know the sensor has been reset. Then, the light either turns back off if the light switch is off, or to a normal white color if the light switch is on.

In addition, if anyone needs to use this restroom before I get to clean the litter box, the light switch still functions normally, and will always turn on to a normal white color when the switch is used, and then back to red when the switch is turned off if the litter box still needs to be cleaned.

### Light Turns Red When Litter Box Used
![Kitty Litter Box Sensor](https://imgur.com/UBYxW0r.gif)

### Light Briefly Turns Green, Then Back to Off/Normal When Litter Box Cleaned
![Kitty Litter Box Sensor](https://imgur.com/jHJ8EVp.gif)
# Hardware

I had some extra Zigbee sensors from other projects laying around, so I incorporated them into this project. Any motion sensor and contact sensor would have worked. I had to buy a new bulb and a Shelly 1 to put behind the switch.

## Aqara Zigbee Motion Sensor

I stuck [this Aqara Zigbee motion sensor](https://www.amazon.com/Aqara-RTCGQ11LM-Motion-Sensor-White/dp/B07D1CRRVF/ref=sr_1_5?keywords=aqara%20motion%20sensor&qid=1641840401&sprefix=aquara%20mo,aps,108&sr=8-5) under the sink, high enough that it should be blocked from catching any motion not associated with the litter box. There is also a curtain under the front of the sink that stays closed. The sensors have been up and running for about two weeks now, and I haven't had any false detections.

![Kitty Litter Aqara Motion Sensor](https://i.imgur.com/KcAru3x.jpg?1 =650x)

## Sonoff Zigbee Contact Sensor
Next, I stuck [this Sonoff Zigbee Contact Sensor I got from AliExpress](https://www.aliexpress.com/item/1005001928750103.html?spm=a2g0o.store_pc_groupList.8148356.26.36d9179axo6qht) on the Litter Genie so I could detect when it was opened.

![Litter Genie Zigbee Sensor](https://i.imgur.com/z5tyUfU.jpg?1 =650x)

## Feit Electric Bulb (using Tuya Local)
The bathroom where we keep the litter box has a light fixture with an exposed bulb, and I wanted to be able to continue using an Edison bulb. The only smart Edison bulb I could find that was capable of changing colors was this wifi one from [Feit Electric](https://www.amazon.com/Feit-Electric-ST2160-RGBW-FIL/dp/B089CN4WRN/ref=sr_1_7?keywords=smart%20edison%20bulbs&qid=1641838719&sr=8-7). It's not super bright, but it is good enough for this application.

![Feit Electric Bulb](https://i.imgur.com/UzYRWVq.png =x250)
I set the bulb up in Home Assistant using the [Local Tuya integration from HACS](https://github.com/rospogrigio/localtuya/). I had to do a little bit of trial and error to find the values I wanted to use for the white and green light settings.

## Shelly 1
I used a [Shelly 1](https://shopusa.shelly.cloud/shelly-1-ul-wifi-smart-home-automation-1#164) behind the light switch so I could replicate normal functionality of the switch. Basically, I have it set so the relay is separated from the switch. That means I can tell the power to just always stay on, and then send the status of the physical switch to Home Assistant. I have set all my Shelly devices up to communicate with Home Assistant [locally through MQTT](https://github.com/bieniu/ha-shellies-discovery).

## Home Assistant Helper

I used HA’s helpers to set up an input_boolean entity. I created this in the UI by going to Configuration/Automations & Scenes/Helpers and clicking the "Add Helper" button, then “Toggle.” I then added a name, in my case “Kitty Litter Box”.

# Automations:

I created five automations in the UI for this project, and I’ll do my best to explain them and the logic behind them. I don't use Node Red, but all of this should be achievable in both platforms. All of my automations utilize the "choose" function:

## First I want the motion sensor to turn on the helper
I added the helper so I could have it play middle man for notifications and provide a persistent status. This way, I can delay notifications when they wouldn't be helpful, like when I'm not at home or I'm asleep in bed. I made all of these in the UI, but the yaml is below. This automation also turns the helper off when the litter genie's lid is opened.

```
# automations.yaml

- alias: Kitty Litter Box Tracker - Turn On/Off Boolean
  description: ''
  trigger:
  - platform: state
    entity_id: binary_sensor.kitty_litter_motion
    from: 'off'
    to: 'on'
    id: litteron
  - platform: state
    entity_id: binary_sensor.litter_genie_zigbee
    id: litteroff
    from: 'off'
    to: 'on'
  condition: []
  action:
  - choose:
    - conditions:
      - condition: trigger
        id: litteron
      - condition: template
        value_template: '{{ ( as_timestamp(now()) - as_timestamp(states.binary_sensor.litter_genie_zigbee.last_changed)
          |int(0) ) > 90 }}'
      sequence:
      - service: input_boolean.turn_on
        target:
          entity_id: input_boolean.kitty_litter_box
    - conditions:
      - condition: trigger
        id: litteroff
      sequence:
      - service: input_boolean.turn_off
        target:
          entity_id: input_boolean.kitty_litter_box
    default: []
  mode: single
```
## Then, I want the lights to be set according to the helper's status

I want the light switch to function normally so I don't run afoul of my spouse and guests, since this is also the guest bathroom. I am using a Shelly 1 wired behind the switch, set to "detached mode," so the bulb is always powered, and Home Assistant is turning the bulb on and off when it detects a switch state change. The shelly switch is "binary_sensor.half_bath_shelly_1_input_0" and the light bulb is "light.half_bath_light"

![Kitty Litter Box Sensor](https://imgur.com/PwoMREP.gif)
### First, an automation to control what happens to the lights when the helper's status changes:
```
- alias: Kitty Litter Box Tracker - Lights
  description: ''
  trigger:
  - platform: state
    entity_id: input_boolean.kitty_litter_box
    from: 'off'
    to: 'on'
    id: litterlighton
  - platform: state
    entity_id: input_boolean.kitty_litter_box
    from: 'on'
    to: 'off'
    id: litterlightoff
  condition: []
  action:
  - choose:
    - conditions:
      - condition: trigger
        id: litterlighton
      - condition: state
        entity_id: binary_sensor.half_bath_shelly_1_input_0
        state: 'off'
      sequence:
      - service: light.turn_on
        data:
          rgb_color:
          - 255
          - 0
          - 0
          brightness_pct: 50
        target:
          entity_id:
          - light.half_bath_light
    - conditions:
      - condition: trigger
        id: litterlightoff
      sequence:
      - service: light.turn_on
        data:
          rgb_color:
          - 36
          - 255
          - 37
        target:
          entity_id:
          - light.half_bath_light
      - delay:
          hours: 0
          minutes: 0
          seconds: 5
          milliseconds: 0
      - choose:
        - conditions:
          - condition: state
            entity_id: binary_sensor.half_bath_shelly_1_input_0
            state: 'off'
          sequence:
          - service: light.turn_off
            target:
              entity_id: light.half_bath_light
        - conditions:
          - condition: state
            entity_id: binary_sensor.half_bath_shelly_1_input_0
            state: 'on'
          sequence:
          - service: light.turn_on
            data:
              rgb_color:
              - 255
              - 255
              - 75
            target:
              entity_id:
              - light.half_bath_light
        default: []
    default: []
  mode: single
```
### Then, an automation to control what happens to the lights when the light switch is used:
![Kitty Litter Box Sensor](https://imgur.com/XkQQR6L.gif)
```
- alias: Lights - Half Bath - Switch Automations
  description: ''
  trigger:
  - platform: state
    entity_id: binary_sensor.half_bath_shelly_1_input_0
    from: 'off'
    to: 'on'
    id: switchon
  - platform: state
    entity_id: binary_sensor.half_bath_shelly_1_input_0
    from: 'on'
    to: 'off'
    id: switchoff
  condition: []
  action:
  - choose:
    - conditions:
      - condition: trigger
        id: switchon
      sequence:
      - service: light.turn_on
        data:
          rgb_color:
          - 255
          - 255
          - 75
          brightness_pct: 100
        target:
          entity_id:
          - light.half_bath_light
    - conditions:
      - condition: trigger
        id: switchoff
      - condition: state
        entity_id: input_boolean.kitty_litter_box
        state: 'on'
      sequence:
      - service: light.turn_on
        data:
          rgb_color:
          - 255
          - 0
          - 0
          brightness_pct: 50
        target:
          entity_id:
          - light.half_bath_light
    - conditions:
      - condition: trigger
        id: switchoff
      - condition: state
        entity_id: input_boolean.kitty_litter_box
        state: 'off'
      sequence:
      - service: light.turn_off
        target:
          entity_id: light.half_bath_light
    default: []
  mode: single
```
## Last, I want to get an actionable notification, depending on where I am and what my status is:
As I mentioned before, I added the helper so I could have it play middle man for notifications and persistent status. This way, I can delay notifications when they wouldn't be helpful, like when I'm not at home or I'm asleep in bed. Here are the automations for sending me a notification. I have a "bedtime" script that turns out all the lights in my house and makes sure the doors are locked and the alarm is on. The notifications are conditioned to only send if I'm home and the bedtime script has not been run.

### First, the automation to send the notification:
```
- alias: Kitty Litter Box Tracker - Notify that litter box needs to be cleaned
  description: ''
  trigger:
  - platform: state
    id: littermotion
    from: 'off'
    to: 'on'
    for:
      hours: 0
      minutes: 0
      seconds: 30
      milliseconds: 0
    entity_id: input_boolean.kitty_litter_box
  - platform: zone
    entity_id: device_tracker.my_iphone
    zone: zone.home
    event: enter
    id: arrivinghomelitterneedstobeemptied
  - at: 08:30:00
    platform: time
    id: morning
  condition:
  - condition: state
    entity_id: input_boolean.kitty_litter_box
    state: 'on'
  action:
  - choose:
    - conditions:
      - condition: trigger
        id: littermotion
      - condition: template
        value_template: '{{ ( as_timestamp(now()) - as_timestamp(state_attr(''script.bedtime'',
          ''last_triggered'')) |int(0) ) > 28800 }}'
      - condition: zone
        entity_id: device_tracker.my_iphone
        zone: zone.home
      sequence:
      - service: notify.mobile_app_my_iphone
        data:
          title: Litter Box Alert
          message: The litter box needs to be emptied.
          data:
            actions:
            - action: LITTER_CLEAN
              title: I cleaned the litter box.
            - action: REMIND_CLEAN
              title: Remind me in an hour.
    - conditions:
      - condition: trigger
        id: arrivinghomelitterneedstobeemptied
      sequence:
      - delay:
          hours: 0
          minutes: 2
          seconds: 0
          milliseconds: 0
      - service: notify.mobile_app_my_iphone
        data:
          title: Litter Box Alert
          message: The litter box needs to be emptied.
          data:
            actions:
            - action: LITTER_CLEAN
              title: I cleaned the litter box.
            - action: REMIND_CLEAN
              title: Remind me in an hour.
    - conditions:
      - condition: trigger
        id: morning
      - condition: zone
        entity_id: device_tracker.my_iphone
        zone: zone.home
      sequence:
      - service: notify.mobile_app_my_iphone
        data:
          title: Litter Box Alert
          message: The litter box needs to be emptied.
          data:
            actions:
            - action: LITTER_CLEAN
              title: I cleaned the litter box.
            - action: REMIND_CLEAN
              title: Remind me in an hour.
    default: []
  mode: single
```
### Then, a second automation here for processing the actionable response:
```
- alias: Kitty Litter Box Tracker - iOS Response Processing
  description: ''
  trigger:
  - event_data:
      actionName: LITTER_CLEAN
    event_type: ios.notification_action_fired
    platform: event
    id: litterclean
  - event_data:
      actionName: REMIND_CLEAN
    event_type: ios.notification_action_fired
    platform: event
    id: remindclean
  condition: []
  action:
  - choose:
    - conditions:
      - condition: trigger
        id: litterclean
      sequence:
      - service: input_boolean.turn_off
        target:
          entity_id: input_boolean.kitty_litter_box
    - conditions:
      - condition: trigger
        id: remindclean
      sequence:
      sequence:
        - delay:
            hours: 1
            minutes: 0
            seconds: 0
            milliseconds: 0
        - condition: state
          entity_id: input_boolean.kitty_litter_box
          state: 'on'
        - service: notify.mobile_app_my_iphone
          data:
            title: Litter Box Alert
            message: 'Reminder: The litter box needs to be emptied.'
            data:
              actions:
                - action: LITTER_CLEAN
                  title: I cleaned the litter box.
                - action: REMIND_CLEAN
                  title: Remind me in an hour.
    default: []
  mode: single
```
# Conditional Cards for Lovelace UI

I already have a cat tab on my Lovelace dashboard because [we track the cat's insulin shots](https://github.com/davearneson/Kitty-Shot-Tracker#readme) - so I just added a couple of conditional cards that only show if the litter box needs to be cleaned. Basically, the idea is that I never use these because the process is automated. However, it is sometimes nice to just have a record of if it's currently dirty, and if so, since when?

![Kitty Shot Sensor](https://imgur.com/S4CQuHv.jpg)

```
type: vertical-stack
cards:
  - type: conditional
    conditions:
      - entity: input_boolean.kitty_litter_box
        state: 'on'
    card:
      type: markdown
      content: >-
        ## <center>Litter box needs to be cleaned. (Since  {{
        relative_time(states.input_boolean.kitty_litter_box.last_changed) }}
        ago).
  - type: conditional
    conditions:
      - entity: input_boolean.kitty_litter_box
        state: 'on'
    card:
      type: button
      tap_action:
        action: call-service
        service: input_boolean.turn_off
        service_data: {}
        target:
          entity_id: input_boolean.kitty_litter_box
      name: Mark as Cleaned
      icon: mdi:shovel
      icon_height: 50px
```
# CAT TAX
![Kitty Tax](https://i.redd.it/806k5whws8t71.jpg =500x)
(#theonewhopoops)

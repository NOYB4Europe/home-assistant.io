---
title: Signal Messenger
description: Instructions on how to integrate Signal Messenger within Home Assistant.
ha_category:
  - Notifications
ha_iot_class: Cloud Push
ha_release: 0.104
ha_codeowners:
  - '@bbernhard'
ha_domain: signal_messenger
ha_platforms:
  - notify
ha_integration_type: integration
---

The `signal_messenger` integration uses the [Signal Messenger REST API](https://github.com/bbernhard/signal-cli-rest-api) to deliver notifications from Home Assistant to your Android or iOS device.

## Setup
 
The requirements are:
- You need to install the extension
- You need to set up the Signal Messenger REST API. 
- You need a spare phone number to register with the Signal Messenger service. 


Please follow those [instructions](https://github.com/bbernhard/signal-cli-rest-api/blob/master/doc/HOMEASSISTANT.md), to set up the Signal Messenger REST API. 

## install extension
- install HACS in home assistant
- open HACS >> Add-ons >> Add-on store >> three dots >> add repository >> add https://github.com/haberda/hassio_addons >> "ADD"
- install "Signal Messenger"
- open HACS >> Add-ons >> Signal Messanger >> Configuration :
    - mode: normal
    - signal_cli_cmd_timeout : 60
    - Rest-api port : 9123  (or any other free port)
- INFO >> "start" ("start" button shall change to "stop" and "restart")

## Signal Messenger REST API. 

solve captcha : https://signalcaptchas.org/registration/generate 
copy link "open signal" and paste to texteditor


on any linux pc / raspberry pi:

curl -X POST -H "Content-Type: application/json" --data '{"use_voice": false}' 'http://192.168.1.IPofhomeassisant:9123/v1/register/+491770000000'
 
or 

curl -X POST -H "Content-Type: application/json" --data '{"use_voice": false}' 'http://homeassistant.local:9123/v1/register/+491770000000'

curl -X POST -H "Content-Type: application/json" -d '{"captcha":"signal-hcaptcha.VERY_LONG_CAPTCHA_IS_MANUALLY_PUT_HERE", "use_voice": false}' 'http://homeassistant.local:9123/v1/register/+491770000000'

curl -X POST -H "Content-Type: application/json" 'http://homeassistant.local:9123/v1/register/+491770000000/verify/512982'

## Configuration

To send Signal Messenger notifications with Home Assistant, add the following to your `configuration.yaml` file:

```yaml
# Example configuration.yaml entry for Signal Messenger 
notify:
  - name: signal
    platform: signal_messenger
    url: "http://127.0.0.1:8080" # the URL where the Signal Messenger REST API is listening 
    number: "YOUR_PHONE_NUMBER" # the sender number
    recipients: # one or more recipients
      - "+132022243121"
      - "+491234560022"
```

Both phone numbers and Signal Messenger groups can be added to the `recipients`list. However, it's not possible to mix phone numbers and Signal Messenger groups in a single notifier. If you would like to send messages to individual phone numbers and Signal Messenger groups, separate notifiers need to be created.

To obtain the Signal Messenger group ids, follow [this guide]( https://github.com/bbernhard/signal-cli-rest-api/blob/master/doc/HOMEASSISTANT.md).

{% configuration %}
name:
  description: Setting the optional parameter `name` allows multiple notifiers to be created. The notifier will bind to the service `notify.NOTIFIER_NAME`.
  required: false
  type: string
  default: notify
url:
  description: The URL where the Signal Messenger REST API listens for incoming requests. 
  required: true
  type: string
number:
  description: The sender number.
  required: true
  type: string
recipients:
  description: A list of recipients (either phone numbers or Signal Messenger group ids).
  required: true
  type: string
{% endconfiguration %}


## Sending messages to Signal to trigger events

You can use Signal Messenger REST API as a Home Assistant trigger. In this example, we will make a simple chatbot. If you write anything to your Signal account linked to Signal Messenger REST API, the automation gets triggered, with the condition that the number (attribute source) is correct, to take action by sending a Signal notification back with a "Message received!".

To accomplish this, edit the configuration of Home Assistant, adding a [RESTful resource](/integrations/rest/) as follows:

```yaml
- resource: "http://127.0.0.1:8080/v1/receive/<number>"
  headers:
    Content-Type: application/json
  sensor:
    - name: "Signal message received"
      value_template: "{{ value_json[0].envelope.dataMessage.message }}" #this will fetch the message
      json_attributes_path: $[0].envelope
      json_attributes:
        - source #using attributes you can get additional information, in this case, the phone number.
  ```
You can create an automation as follows:

```yaml
...
trigger:
  - platform: state
    entity_id:
      - sensor.signal_message_received
    attribute: source
    to: "<yournumber>"
action:
  - service: notify.signal
    data:
      message: "Message received!"
```

## Examples

A few examples on how to use this integration as actions in automations.


### Text message
```yaml
...
alias: TestSignalMessenger
description: ""
trigger: []
condition: []
action:
  - service: notify.signal
    data:
      message: "Hi there this is your homeassistant sending messenges via signal"
      target: 49170123123 3394326339
mode: single
```

### Text message with an attachment

```yaml
...
action:
  service: notify.NOTIFIER_NAME
  data:
    message: "Alarm in the living room!"
    data:
      attachments:
        - "/tmp/surveillance_camera.jpg"
```

### Text message with an attachment from a URL

To attach files from outside of Home Assistant, the URLs must be added to the [`allowlist_external_urls`](/docs/configuration/basic/#allowlist_external_urls) list.

Note there is a 50MB size limit for attachments retrieved via URLs. You can also set `verify_ssl` to `false` to ignore SSL errors (default `true`).

```yaml
...
action:
  service: notify.NOTIFIER_NAME
  data:
    message: "Person detected on Front Camera!"
    data:
      verify_ssl: false
      urls:
        - "http://homeassistant.local/api/frigate/notifications/<event-id>/thumbnail.jpg"
```

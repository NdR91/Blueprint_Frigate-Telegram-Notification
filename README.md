# Blueprint - Frigate Telegram Notification
An Home Assistant Blueprint to receive Snapshots and Clips from Frigate

Add this url to your Blueprints projects:
https://gist.github.com/NdR91/5486a1e55101e062c48545395b7dd9a3

```yaml
blueprint:
  name: Frigate - Telegram Notification
  description: Create automations to receive Snapshots and Clips from Frigate
  domain: automation
  input:
    camera:
      name: Frigate Camera
      description: The name of the camera as defined in your frigate configuration (/conf.yml).
    target_chat:
      name: Target
      description: 'The chat_id to be used by the Telegram bot. 
                !secret chat_id is not allowed on Blueprint, you will need the chat_id code.
        '
    notification:
      name: Notification
      description: 'Select "true" to disable notification, lave "false" to receive
        notification.
        '
      selector:
        select:
          options:
          - 'true'
          - 'false'
      default: 'false'
    base_url:
      name: (Optional) Base URL
      description: The external url for your Home Assistant instance. This will default
        to a relative URL and will open the clips in the app instead of the browser,
        which may cause issues on some devices.
      default: ''
    zone_filter:
      name: (Optional) Zone Filter
      description: Only notify if object has entered a defined zone.
      default: false
      selector:
        boolean: {}
    zones:
      name: (Optional) Trigger Zones
      description: A list (-) of zones you wish to recieve notifications for.
      default: []
      selector:
        object: {}
    labels:
      name: (Optional) Trigger Objects
      description: A list (-) of objects you wish to recieve notifications for.
      default: []
      selector:
        object: {}
    presence_filter:
      name: (Optional) Presence Filter
      description: Only notify if selected presence entity is not "home".
      default: ''
      selector:
        entity: {}
  source_url: https://gist.github.com/NdR91/5486a1e55101e062c48545395b7dd9a3
mode: single
max_exceeded: silent
trigger:
  platform: mqtt
  topic: frigate/events
  payload: !input 'camera'
  value_template: '{{ value_json[''after''][''camera''] }}'
variables:
  id: '{{ trigger.payload_json[''after''][''id''] }}'
  camera: '{{ trigger.payload_json[''after''][''camera''] }}'
  camera_name: '{{ camera | replace(''_'', '' '') | title }}'
  target_chat: !input 'target_chat'
  object: '{{ trigger.payload_json[''after''][''label''] }}'
  label: '{{ object | title }}'
  entered_zones: '{{ trigger.payload_json[''after''][''entered_zones''] }}'
  type: '{{ trigger.payload_json[''type''] }}'
  base_url: !input 'base_url'
  zone_only: !input 'zone_filter'
  input_zones: !input 'zones'
  zones: '{{ input_zones | list }}'
  input_labels: !input 'labels'
  labels: '{{ input_labels | list }}'
  presence_entity: !input 'presence_filter'
  notification: !input 'notification'
condition:
- '{{ type != ''end'' }}'
- '{{ not zone_only or entered_zones|length > 0 }}'
- '{{ not zones|length or zones|select(''in'', entered_zones)|list|length > 0 }}'
- '{{ not labels|length or object in labels }}'
- '{{ not presence_entity or not is_state(presence_entity, ''home'') }}'
action:
- service: telegram_bot.send_photo
  data:
    target: '{{ target_chat }}'
    disable_notification: '{{ notification }}'
    caption: |
      Movimento rilevato. Telecamera: {{ camera_name }} (ID: {{ id }})
    url: >-
      {{base_url}}/api/frigate/notifications/{{id}}/snapshot.jpg
- repeat:
    sequence:
    - wait_for_trigger:
      - platform: mqtt
        topic: frigate/events
        payload: '{{ id }}'
        value_template: '{{ value_json[''after''][''id''] }}'
      timeout:
        minutes: 2
      continue_on_timeout: false
    - condition: template
      value_template: '{{ wait.trigger.payload_json[''type''] == ''end'' }}'
    - service: telegram_bot.send_video
      data:
        target: '{{ target_chat }}'
        disable_notification: '{{ notification }}'
        caption: 'Movimento rilevato. Telecamera: {{ camera_name }}'
        url: >-
          {{base_url}}/api/frigate/notifications/{{id}}/{{camera}}/clip.mp4
    until: '{{ wait.trigger.payload_json[''type''] == ''end'' }}'
    ```

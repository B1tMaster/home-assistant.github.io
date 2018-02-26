---
layout: page
title: "Facebook Messenger"
description: "Instructions how to add Facebook user notifications to Home Assistant."
date: 2016-12-31 14:14
sidebar: true
comments: false
sharing: true
footer: true
logo: facebook.png
ha_category: Notifications
ha_release: 0.36
---

The `facebook` notification platform enables sending notifications via Facebook Messenger, powered by [Facebook](https://facebook.com).

To use this notification platform in your installation, add the following to your `configuration.yaml` file:

```yaml
# Example configuration.yaml entry
notify:
  - name: NOTIFIER_NAME
    platform: facebook
    page_access_token: FACEBOOK_PAGE_ACCESS_TOKEN
```

Configuration variables:

- **page_access_token** (*Required*): Access token for your Facebook page. Checkout [Facebook Messenger Platform](https://developers.facebook.com/docs/messenger-platform/guides/setup) for more information.
- **name** (*Optional*): Setting the optional parameter `name` allows multiple notifiers to be created. The default value is `notify`. The notifier will bind to the service `notify.NOTIFIER_NAME`.

### {% linkable_title Usage %}

With Facebook notify service, you can send your notifications to your Facebook messenger with help of your Facebook page. You have to create a [Facebook Page and App](https://developers.facebook.com/docs/messenger-platform/guides/quick-start) for this service. You can control it by calling the notify service [as described here](/components/notify/). It will send a message on messenger to user specified by **target** on behalf of your page. See the [quick start](https://developers.facebook.com/docs/messenger-platform/guides/quick-start) guide for more information.
The phone number used in **target** should be registered with Facebook messenger. Phone number of the recipient should be in +1(212)555-2368 format. If your app is not approved by Facebook then the recipient should by either admin, developer or tester for your Facebook app. [More information](https://developers.facebook.com/docs/messenger-platform/send-api-reference#phone_number) about the phone number.

```yaml
# Example automation notification entry
automation:
  - alias: Evening Greeting
    trigger:
      platform: sun
      event: sunset
    action:
      service: notify.facebook
      data:
        message: 'Good Evening'
        target:
          - '+919413017584'
          - '+919784516314'
```

You can also send messages to users that do not have stored their phone number with Facebook, but this requires a bit more work. The Messenger platform uses page specific user IDs instead of a global user ID. You will need to enable a webhook for the "messages" event in Facebook's developer console. Once a user writes a message to a page, that webhook will then receive the user's page specific ID as part of the webhook's payload. Below is a simple PHP script that reacts to the message "get my id" and sends a reply containing the user's ID: 

```php
<?php

$access_token = "";
$verify_token = "";

if (isset($_REQUEST['hub_challenge'])) {
    $challenge        = $_REQUEST['hub_challenge'];
    $hub_verify_token = $_REQUEST['hub_verify_token'];

    if ($hub_verify_token === $verify_token) {
        echo $challenge;
    }
}

$input   = json_decode(file_get_contents('php://input'), true);
$sender  = $input['entry'][0]['messaging'][0]['sender']['id'];
$message = $input['entry'][0]['messaging'][0]['message']['text'];

if (preg_match('/get my id/', strtolower($message))) {
    $url      = 'https://graph.facebook.com/v2.10/me/messages?access_token=' . $access_token;
    $ch       = curl_init($url);
    $jsonData = '{
        "recipient":{
            "id":"' . $sender . '"
        },
        "message":{
            "text":"Your ID: ' . $sender . '"
        }
      }';

    $jsonDataEncoded = $jsonData;
    curl_setopt($ch, CURLOPT_POST, 1);
    curl_setopt($ch, CURLOPT_POSTFIELDS, $jsonDataEncoded);
    curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/json']);

    if (!empty($input['entry'][0]['messaging'][0]['message'])) {
        $result = curl_exec($ch);
    }
}

```

### {% linkable_title Rich messages %}
You could also send rich messing (cards, buttons, images, videos, etc). [Info](https://developers.facebook.com/docs/messenger-platform/send-api-reference) to which types or messages and how to build them.

```yaml
# Example script with a notification entry with rich message

script:
  test_fb_notification:
    sequence:
      - service: notify.facebook
        data:
          message: Some text before the quick replies
          target: 0034643123212
          data:
            quick_replies:
              - content_type: text
                title: Red
                payload: DEVELOPER_DEFINED_PAYLOAD_FOR_PICKING_RED
              - content_type: text
                title: Blue
                payload: DEVELOPER_DEFINED_PAYLOAD_FOR_PICKING_BLUE
```

You can now also use Facebook public beta broadcast API to push messages to ALL users who interacted with your chatbot on your page, without having to collect their number. This will scale to thousands of users. Facebook requires that this only be used for non-commercial purposes and they validate every message you send. Also note, your Facebook bot needs to be authorized for "page_subscriptions" if you want to make it public, but it can be used right with a selected group of testers of your choice. So if your target audience is small, can just keep the bot in development mode and not seek approval from facebook.

To enable broadcast just use the keyword "BROADCAST" as your target. Only put ONE target BROADCAST as below:
```yaml
- alias: Facebook Broadcast
  trigger:
    platform: sun
    event: sunset
  action:
    service: notify.facebook
    data:
      message: 'Good Evening'
      target:
        - BROADCAST
```

Here is an advanced sample for broadcasting wind alerts from a wunderground weather station:

configuration.yaml:

```yaml
notify:
  - platform: facebook
    name: facebook_wind_alerts
    page_access_token: !secret facebook_page_access_token  

 platform: wunderground
 api_key: !secret wunder_api_key
 pws_id: !secret  wunder_station_id
 monitored_conditions:
    - wind_kph
    - wind_dir
    - wind_gust_kph
    - pressure_trend
    - wind_string
    - weather_1h
 
 
 platform: template
 sensors:
    windy:
      friendly_name: 'Wind'
      icon_template: mdi:weather-windy
      value_template: >-
        {% if (states.sensor.pws_wind_kph.state | float + states.sensor.pws_wind_gust_kph.state | float)/2.0 > 18.0 -%}
            Windy
        {%- else -%}
            Beer
        {%- endif -%}
```

```yaml
- alias: Facebook Wind Alert
  trigger:
    platform: state
    entity_id: sensor.windy
    to: 'Windy'
  condition:
    condition: and
    conditions:
      - condition: state
        entity_id: sun.sun
        state: 'above_horizon'
      - condition: state
        entity_id: input_boolean.wind_alert
        state: 'on'
      - condition: time
        before: '19:00:00'
        after: '07:00:00'
  action:
    service: notify.facebook_wind_alerts
    data_template:
      message: >
        {{ states.sensor.pws_wind_string.state }}
        Pressure Trend : {{ states.sensor.pws_pressure_trend.state }}
        Next Hour: {{ states.sensor.pws_weather_1h.state }}
     target:
       - BROADCAST
       
```       

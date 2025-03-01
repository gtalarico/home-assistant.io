---
title: IMAP
description: Instructions on how to integrate IMAP unread email into Home Assistant.
ha_category:
  - Mailbox
ha_release: 0.25
ha_iot_class: Cloud Push
ha_domain: imap
ha_platforms:
  - sensor
ha_integration_type: integration
ha_codeowners:
  - '@jbouwh'
ha_config_flow: true
---

The IMAP integration is observing your [IMAP server](https://en.wikipedia.org/wiki/Internet_Message_Access_Protocol). It can report the number of unread emails and can send a custom event that can be used to trigger an automation. Other search criteria can be used, as shown in the example below.

{% include integrations/config_flow.md %}

### Gmail with App Password

If you’re going to use Gmail, 2-step verification must be enabled on your Gmail account.  Once it is enabled, you need to create an [App Password](https://support.google.com/mail/answer/185833).

1. Go to your [Google Account](https://myaccount.google.com/)
2. Select **Security**
3. Under “How you sign into Google” select **2-Step Verification**.
4. Sign in to your Account.
5. At the bottom of the 2-Step Verification page, click **App Passwords**.
6. Give your app a name that makes sense to you (Home Assistant IMAP, for example).
7. Click **Create**, then make a note of your 16-character app password for safekeeping (remove the spaces when you save it).
8. Click **Done**.
9. Add the IMAP Integration to your Home Assistant instance using the My button above.  Enter the following information as needed:

    - Username: Your Gmail email login
    - Password: your 16-character app password (without the spaces)
    - Server: `imap.gmail.com`
    - Port: `993`

10. Click **Submit**.
11. Assign your integration to an "Area" if desired, then click **Finish**.

Congratulations, you now have a sensor that counts the number of unread e-mails in your Gmail account.  From here you can create additional sensors based upon the data that comes through the event bus when there's a new message detected.

### Configuring IMAP Searches

By default, this integration will count unread emails. By configuring the search string, you can count other results, for example:

- `ALL` to count all emails in a folder
- `FROM`, `TO`, `SUBJECT` to find emails in a folder (see [IMAP RFC for all standard options](https://tools.ietf.org/html/rfc3501#section-6.4.4))
- [Gmail's IMAP extensions](https://developers.google.com/gmail/imap/imap-extensions) allow raw Gmail searches, like `X-GM-RAW "in: inbox older_than:7d"` to show emails older than one week in your inbox. Note that raw Gmail searches will ignore your folder configuration and search all emails in your account!


### Selecting a charset supported by the imap server

Below is an example for setting up the integration to connect to your Microsoft 365 account that requires `US-ASCII` as charset:
  - Server: `outlook.office365.com`
  - Port: `993`
  - Username: Your full email address
  - Password: Your password
  - Charset: `US-ASCII`

<div class="note">

Yahoo also requires the character set `US-ASCII`.

</div>

### Selecting an alternate SSL cipher list or disable SSL verification (advanced mode)

If the default IMAP server settings do not work, you might try to set an alternate SLL cipher list.
The SSL cipher list option allows to select the list of SSL ciphers to be accepted from this endpoint. `default` (_system default_), `modern` or `intermediate` (_inspired by [Mozilla Security/Server Side TLS](https://wiki.mozilla.org/Security/Server_Side_TLS)_)

If you are using self signed certificates can can turn of SSL verification.

<div class='note info'>

The SSL cipher list and verify SSL are advanced settings. The options are available only when advanced mode is enabled (see user settings).

</div>

### Enable IMAP-Push

IMAP-Push is enabled by default if your IMAP server supports it. If you use an unreliable IMAP service that periodically drops the connection and causes issues, you might consider turning off IMAP-Push. This will fall back to polling the IMAP server.

<div class='note info'>

The enforce polling option is an advanced setting. The option is available only when advanced mode is enabled (see user settings).

</div>

### Troubleshooting

Email providers may limit the number of reported emails. The number may be less than the limit (10,000 at least for Yahoo) even if you set the `IMAP search` to reduce the number of results. If you are not getting expected events and cleaning your Inbox or the configured folder is not desired, set up an email filter for the specific sender to go into a new folder. Then create a new config entry or modify the existing one with the desired folder.


### Using events

When a new message arrives or a message is removed within the defined search command scope, the `imap` integration will send a custom [event](/docs/automation/trigger/#event-trigger) that can be used to trigger an automation.
It is also possible to use to create a template [`binary_sensor` or `sensor`](/integrations/template/#trigger-based-template-binary-sensors-buttons-numbers-selects-and-sensors) based the [event data](/docs/automation/templating/#event).

The table below shows what attributes come with `trigger.event.data`. The data is a dictionary that has the keys that are shown below.

The attributes shown in the table are also available as variables for the custom event data template. The [example](/integrations/imap/#example---custom-event-data-template) shows how to use this as an event filter.

<div class='note info'>

The custom event data template is an advanced feature. The option is available only when advanced mode is enabled (see user settings). The `text` attribute is not size limited when used as a variable in the template.

</div>

{% configuration_basic %}
server:
  description: The IMAP server name
username:
  description: The IMAP user name
search:
  description: The IMAP search configuration
folder:
  description: The IMAP folder configuration
text:
  description: The email body `text` of the message (by default, only the first 2048 bytes will be available.)
sender:
  description: The `sender` of the message
subject:
  description: The `subject` of the message
date:
  description: A `datetime` object of the `date` sent
headers:
  description: The `headers` of the message in the for of a dictionary. The values are iterable as headers can occur more than once.
custom:
  description: Holds the result of the custom event data [template](/docs/configuration/templating). All attributes are available as a variable in the template.
initial:
  description: Returns `True` if this is the initial event for the last message received. When a message within the search scope is removed and the last message received has not been changed, then an `imap_content` event is generated and the `initial` property is set to `False`. Note that if no `Message-ID` header was set on the triggering email, the `initial` property will always be set to `True`.

{% endconfiguration_basic %}

The `event_type` for the custom event should be set to `imap_content`. The configuration below shows how you can use the event data in a template `sensor`.

If the default maximum message size (2048 bytes) to be used in events is too small for your needs, then this maximum size setting can be increased. You need to have your profile set to _advanced_ mode to do this.

<div class='note warning'>

Increasing the default maximum message size (2048 bytes) could have a negative impact on performance as event data is also logged by the `recorder`. If the total event data size exceeds the maximum event size (32168 bytes), the event will be skipped.

</div>

{% raw %}

```yaml
template:
  - trigger:
      - platform: event
        event_type: "imap_content"
        id: "custom_event"
    sensor:
      - name: imap_content
        state: "{{ trigger.event.data['subject'] }}"
        attributes:
          Message: "{{ trigger.event.data['text'] }}"
          Server: "{{ trigger.event.data['server'] }}"
          Username: "{{ trigger.event.data['username'] }}"
          Search: "{{ trigger.event.data['search'] }}"
          Folder: "{{ trigger.event.data['folder'] }}"
          Sender: "{{ trigger.event.data['sender'] }}"
          Date: "{{ trigger.event.data['date'] }}"
          Subject: "{{ trigger.event.data['subject'] }}"
          Initial: "{{ trigger.event.data['initial'] }}"
          To: "{{ trigger.event.data['headers'].get('Delivered-To', ['n/a'])[0] }}"
          Return-Path: "{{ trigger.event.data['headers'].get('Return-Path',['n/a'])[0] }}"
          Received-first: "{{ trigger.event.data['headers'].get('Received',['n/a'])[0] }}"
          Received-last: "{{ trigger.event.data['headers'].get('Received',['n/a'])[-1] }}"
```

{% endraw %}

## Example - keyword spotting

The following example shows the usage of the IMAP email content sensor to scan the subject of an email for text, in this case, an email from the APC SmartConnect service, which tells whether the UPS is running on battery or not.

{% raw %}

```yaml
template:
  - trigger:
      - platform: event
        event_type: "imap_content"
        id: "custom_event"
        event_data:
          sender: "no-reply@smartconnect.apc.com"
          initial: true
    sensor:
      - name: house_electricity
        state: >-
          {% if 'UPS On Battery' in trigger.event.data["subject"] %}
            power_out
          {% elif 'Power Restored' in trigger.event.data["subject"] %}
            power_on
          {% endif %}
```

{% endraw %}

## Example - extracting formatted text from an email using template sensors

This example shows how to extract numbers or other formatted data from an email to change the value of a template sensor to a value extracted from the email. In this example, we will be extracting energy use, cost, and billed amount from an email (from Georgia Power) and putting it into sensor values using a template sensor that runs against our IMAP email sensor already set up. A sample of the body of the email used is below:

```text
Yesterday's Energy Use:                             76 kWh
Yesterday's estimated energy cost:                  $8
Monthly Energy use-to-date for 23 days:             1860 kWh
Monthly estimated energy cost-to-date for 23 days:  $198

To view your account for details about your energy use, please click here.
```

Below is the template sensor which extracts the information from the body of the email in our IMAP email sensor (named sensor.energy_email) into 3 sensors for the energy use, daily cost, and billing cycle total.

{% raw %}

```yaml
template:
  - trigger:
      - platform: event
        event_type: "imap_content"
        id: "custom_event"
        event_data:
          sender: "no-reply@smartconnect.apc.com"
    sensor:
      - name: "Previous Day Energy Use"
        unit_of_measurement: "kWh"
        state: >
        {{ trigger.event.data["text"]
          | regex_findall_index("\*Yesterday's Energy Use:\* ([0-9]+) kWh") }}
      - name: "Previous Day Cost"
        unit_of_measurement: "$"
        state: >
          {{ trigger.event.data["text"]
            | regex_findall_index("\*Yesterday's estimated energy cost:\* \$([0-9.]+)") }}
      - name: "Billing Cycle Total"
        unit_of_measurement: "$"
        state: >
          {{ trigger.event.data["text"]
            | regex_findall_index("\ days:\* \$([0-9.]+)") }}
```

{% endraw %}

By making small changes to the regular expressions defined above, a similar structure can parse other types of data out of the body text of other emails.

## Example - custom event data template

We can define a custom event data template to help filter events. This can be handy if, for example, we have multiple senders we want to allow.
We define the following template to return true if part of the `sender` is `@example.com`:

{% raw %}

```jinja2
{{ "@example.com" in sender }}
```

{% endraw %}

This will render to `True` if the sender is allowed. The result is added to the event data as `trigger.event.data["custom"]`.

The example below will only set the state to the subject of the email of template sensor, but only if the sender address matches.

{% raw %}

```yaml
template:
  - trigger:
      - platform: event
        event_type: "imap_content"
        id: "custom_event"
        event_data:
          custom: True
    sensor:
      - name: event filtered by template
        state: '{{ trigger.event.data["subject"] }}'
```

{% endraw %}

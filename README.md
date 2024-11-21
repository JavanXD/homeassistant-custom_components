# Home Assistant - My custom components

My own custom components for Home Assistant.

## Package Deliveries Custom Component

This custom component allows you to track your package deliveries from various providers (Amazon, DHL, DPD, etc.) by parsing email notifications. The sensor displays the number of active deliveries and detailed attributes for each package.

### Features

- Fetches package delivery information from email accounts.
- Supports Amazon, DHL, and DPD delivery services.
- Configurable parameters for folder selection, date range, and email limits.
- Displays delivery details as sensor attributes.

### Current Limitations / To-Dos

- Is only able to parse Emails in German language
- Hermes, GLS and UPS notifications are missing
- You need to have an account at those delivery services and need to have enabled that they notify you via email
- Only supporting one email account at the time

### Why a component for this?

- Too many email notifications, missing oversight in email inbox
- Multiple email notifications for the same parcel
- Email notifications from both DHL and Amazon for the same parcel, which needed merging
- One single source of truth by parsing all relevant info and merging it into single entries per tracking number
- One source of relevant delivery data for everyone in the household

### Use-Cases in the Homeassistant Dashboard

#### Markdown Card to show previous and upcoming deliveries

Template:

```yaml
{% set deliveries = state_attr('sensor.package_deliveries', 'deliveries') %}
{% if deliveries is not none %}
  {% set deliveries = deliveries | from_json %}
{% else %}
  No deliveries data available.
{% endif %}
{% if deliveries %}
In den letzten 7 Tagen: **{{ deliveries | length }} Lieferungen**
{% for delivery in deliveries %}
### 🚚 {{ delivery.service | default('Unknown') }}  _({{ delivery.tracking_number | default('Unknown') }})_
 - **Item:** {{ delivery['items'] | default('Unknown') }}
 - **Liefertag:** {{ delivery.delivery_date | default('Unknown') }}
 - **Datum der Email:** {{ delivery.email_date | default('Unknown') }}
 - **Bestellnr.:** {{ delivery.order_number | default('Unknown') }}
{% endfor %}
{% else %}
- No upcoming deliveries.
{% endif %}
```

<img width="300px" src="./docs/markdown.jpg" alt="Image" />

#### Every morning one push Notification with today's deliveries

```yaml
alias: Benachrichtige über heutige Paketlieferungen
description: ""
triggers:
  - at: "08:00:00"
    trigger: time
conditions:
  - condition: template
    value_template: >
      {% set weekdays = ["Montag", "Dienstag", "Mittwoch", "Donnerstag",
      "Freitag", "Samstag", "Sonntag"] %} {% set months = ["Januar", "Februar",
      "März", "April", "Mai", "Juni", "Juli", "August", "September", "Oktober",
      "November", "Dezember"] %} {% set today = now() %} {% set weekday =
      weekdays[today.weekday()] %} {% set day = today.day %} {% set month =
      months[today.month - 1] %} {% set today_german = weekday ~ ', ' ~ day ~ '.
      ' ~ month %} {% set deliveries = state_attr('sensor.package_deliveries',
      'deliveries') | from_json %} {% if deliveries %}
        {{ deliveries | selectattr('delivery_date', 'eq', today_german) | list | length > 0 }}
      {% else %}
        false
      {% endif %}
actions:
  - data:
      data:
        tag: package_deliveries
        visibility: public
        channel: Alarm
        notification_icon: mdi:package-variant
      title: 📦 Paketlieferungen für heute
      message: >
        {% set deliveries = state_attr('sensor.package_deliveries',
        'deliveries') | from_json %} {% set weekdays = ["Montag", "Dienstag",
        "Mittwoch", "Donnerstag", "Freitag", "Samstag", "Sonntag"] %} {% set
        months = ["Januar", "Februar", "März", "April", "Mai", "Juni", "Juli",
        "August", "September", "Oktober", "November", "Dezember"] %} {% set
        today = now() %} {% set weekday = weekdays[today.weekday()] %} {% set
        day = today.day %} {% set month = months[today.month - 1] %} {% set
        today_german = weekday ~ ', ' ~ day ~ '. ' ~ month %} {% set
        today_deliveries = deliveries | selectattr('delivery_date', 'eq',
        today_german) | list %} {% if today_deliveries | length > 0 %}
          {% for delivery in today_deliveries %}
            🚚 {{ delivery.service | default('Unknown') }} - {{ delivery['items'] | default('Unknown') }}
          {% endfor %}
        {% else %}
          Es gibt keine Lieferungen für heute.
        {% endif %}
    action: notify.all_apps
```

<img width="300px" src="./docs/notification.jpg" alt="Image" />

### Installation

1. **Download the Files:**
   - Copy the `custom_components/package_deliveries` directory to your Home Assistant's `custom_components` directory.

2. **Update Your Configuration:**
   - Add the following configuration to your `configuration.yaml` file:
        ```yaml
            sensor:
            - platform: package_deliveries
                email: !secret package_deliveries_email
                password: !secret package_deliveries_app_password
                smtp_server: "imap.gmail.com"
                smtp_port: 993
                last_days: 10
                last_emails: 50
                imap_folder: "Bestellungen/Lieferdienst" # for Gmail default is INBOX if you do not use filters to move emails in sub-folders
                scan_interval: 180
        ```
    - Replace the placeholders (your_email, your_password, etc.) with your actual email account credentials and parameters.
3.	**Restart Home Assistant**:
    - Restart Home Assistant for the custom component to load.
4.	**Add Secrets**:
    - If using !secret, add your credentials to your `secrets.yaml` file:

    ```yaml
    package_deliveries_email: "example@gmail.com"
    package_deliveries_app_password: "app_specific_password" # Needs to be an app passwod, not your google accounts password.
    ```

### CLI

```cli
cd homeassistant-custom_components/package_deliveries/custom_scripts
python3 check_package_deliveries.py --email "XXX" --password "XXX" --imap_folder "Bestellungen/Lieferdienst" --output_file "package_deliveries.json"
```

### Configuration Parameters

- email: Email address to log in to.
- password: Password for the email account (use an app-specific password if needed).
- smtp_server: IMAP server (default: imap.gmail.com).
- smtp_port: IMAP server port (default: 993).
- last_days: Number of days to look back for emails.
- last_emails: Maximum number of emails to process.
- imap_folder: IMAP folder to monitor (e.g., Bestellungen/Lieferdienst).
- scan_interval: Interval (in seconds) to update the sensor.

### Example Output

The Package Deliveries sensor will show the total number of active deliveries. Additional details will appear as attributes:

```yaml
deliveries:
  - service: Amazon
    order_number: "123-4567890-1234567"
    tracking_number: "1Z9999W99999999999"
    total_amount: "50.99 EUR"
    delivery_date: "2024-11-25"
    items: "Book, Laptop"
  - service: DHL
    tracking_number: "9876543210"
    delivery_date: "2024-11-26"
    items: "Package from Electronics Shop"
```

## Contributing

Feel free to submit issues or pull requests via the GitHub repository.

## License

This project is licensed under the MIT License.



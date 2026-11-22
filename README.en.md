# Alto Trevigiano Servizi (ATS) for Home Assistant

[🇮🇹 Italiano](README.md) · 🇬🇧 English

**Unofficial** Home Assistant integration that reads your water supplies, invoices and meter readings from the [Alto Trevigiano Servizi](https://sgc.altotrevigianoservizi.it/) customer area.

> Not affiliated with or endorsed by Alto Trevigiano Servizi. It uses the same requests the portal makes in your browser, so it may stop working if the portal changes.

## What it does

- **Automatic login (recommended):** enter your ATS Online Desk username and password. Customer, supplies and API paths are discovered automatically and the session renews itself. **Manual mode** with cookie and API paths is still available.
- Supplies (`supply-light`): status, meter and supply data.
- Invoices (`invoice`): amounts, remaining balance, payments, due dates and billed consumption.
- Meter readings (`reading-base`): latest reading, consumption and history. Optional.

Everything is exposed as sensors under a single device named **ATS**.

### Sensors

| Entity | Content |
|---|---|
| `sensor.ats_utenze_attive` / `sensor.ats_utenze_totali` | Active and total supplies |
| `sensor.ats_dettagli_utenze_attive` | Attributes `utenza_1`, `utenza_2`… (status, service, address, meter, balance, estimated yearly consumption) |
| `sensor.ats_fatture_presenti` / `sensor.ats_fatture_non_pagate` | Number of invoices and unpaid invoices |
| `sensor.ats_totale_fatture` / `sensor.ats_totale_pagato` / `sensor.ats_totale_residuo_da_pagare` | Totals in € |
| `sensor.ats_prossima_scadenza_fattura` / `sensor.ats_importo_prossima_scadenza` | Due date and remaining amount of the next open invoice |
| `sensor.ats_stato_ultima_fattura` | Latest invoice status, with number, period, amount and remaining balance as attributes |
| `sensor.ats_consumo_fatturato_totale` | Sum of the consumption on the invoices |
| `sensor.ats_dettagli_fatture` | Attributes `fattura_1`, `fattura_2`… with every field of each invoice |
| `sensor.ats_ultima_lettura_contatore` | Latest reading in m³ |
| `sensor.ats_consumo_ultima_lettura` / `sensor.ats_data_ultima_lettura` | Consumption and date of the latest reading |
| `sensor.ats_dettagli_letture_contatore` | Attributes `lettura_1`, `lettura_2`… with the readings history |

With automatic login the readings are included; in manual mode the reading sensors stay `unknown` until you set the readings path.

#### Identifying a sensor: `ats_metric`

Every sensor has an `ats_metric` attribute that identifies it regardless of its `entity_id`. Entity IDs can change (for example when you assign an area): cards and templates can look a sensor up by `ats_metric` instead of by name.

| `ats_metric` | Content |
|---|---|
| `active_supplies` / `total_supplies` | Active / total supplies |
| `supply_details` | Supply details |
| `invoice_count` / `unpaid_count` | Invoices present / unpaid |
| `remaining_total` / `total_amount` / `paid_total` | Totals in € |
| `next_expiry_date` / `next_expiry_amount` | Next due date and amount |
| `consumption` | Total billed consumption |
| `last_status` | Latest invoice status |
| `invoice_details` | Invoice details |
| `last_reading_value` / `last_reading_consumption` / `last_reading_date` | Latest reading, consumption and date |
| `reading_details` | Readings history |

Invoices with `Payed=true` are never counted as unpaid, even if ATS leaves `ToPay=true`. Sensor attributes (addresses, POD, meters, amounts, invoice history) stay visible in Home Assistant but are **not recorded in the database**.

## Installation

### With HACS (recommended)

1. In HACS open the ⋮ menu → **Custom repositories**.
2. Paste this repository's URL, choose the **Integration** category and click **Add**.
3. Search for **Alto Trevigiano Servizi (ATS)**, download it and **restart Home Assistant**.

### Manual

Copy the `custom_components/ats` folder to `/config/custom_components/ats` and restart Home Assistant.

### Adding the integration

**Settings → Devices & services → Add integration → Alto Trevigiano Servizi (ATS)**, then choose how to connect:

- **Automatic login (recommended):** enter your ATS Online Desk username and password (the account you use to log in to the portal with credentials, **not** SPID/CIE) and, optionally, the interval. The integration checks the login and discovers customer, supplies, invoices and readings by itself.
- **Manual mode:** fill in the API paths and cookie as explained in [Manual mode](#manual-mode-how-to-find-the-cookie-and-paths). Use it if you only log in to the portal with SPID/CIE and have no username and password.

## Manual mode: how to find the cookie and paths

This section is only for manual mode: it uses the API request paths and the session cookie the portal uses while you are logged in. You get both from your browser's developer tools (the labels below are from Chrome; Edge and Firefox are very similar).

1. Log in to the ATS customer area in your browser as usual.
2. Press **F12** (or right-click → **Inspect**) and open the **Network** tab.
3. Pick the **Fetch/XHR** filter so you only see API calls.
4. With the tools open, browse the portal sections: **supplies**, **invoices** and **meter readings**. Each one triggers its own request. If the list is empty, reload the page with the Network tab already open.
5. Click each request and, in the **Headers** tab, look at **Request URL**. You need these three:

   | Request | How to recognise it | Form field |
   |---|---|---|
   | Supplies | the URL contains `supply-light` | Supplies API path |
   | Invoices | the URL contains `invoice` | Invoices API path |
   | Readings | `…/webapi/api/v1/reading-base/<code>` | Readings API path (optional) |

6. From the URL copy **only the part starting with `/webapi/`**, query parameters included (for example `/webapi/api/v1/reading-base/<code>`). The base URL `https://sgc.altotrevigianoservizi.it` is already pre-filled.
7. Still under **Headers**, scroll to **Request Headers** and copy the value of the **cookie** row (for example `ASP.NET_SessionId=…`), all of it, including the name before the `=` sign. It goes in the **Session HTTP cookie** field.
8. Check that the request is a `GET` returning `200 OK`, and that the **Response** or **Preview** tab shows JSON with your data.

The update interval is in minutes (5 to 1440, default 60).

> The paths contain codes tied to your account and the cookie works like a password. Never paste them in issues, forums, logs or screenshots. If you need to show something, blur the codes.

## Session lifetime

**Automatic login:** when the portal rejects the session (401/403 response or a redirect to the login page), the integration logs in again by itself, once per request, and retries. If ATS no longer accepts the password, Home Assistant starts **reauthentication** and asks for the new password for the same username, without recreating the entities.

**Manual mode:** the integration uses the portal's session cookie and does not renew it. So far, in real use, the cookie has stayed valid over time, but its lifetime is **not documented by the portal** and is not guaranteed: the session may end, for example after logging out of the portal, a revocation or a server-side expiry. If it stops working, the update fails and Home Assistant shows the integration's update error. Repeat steps 1–7 to get a fresh cookie and update it in **Settings → Devices & services → ATS → Configure**: the integration reloads by itself. To avoid invalidating the cookie in use, do not log out of the portal in the browser you copied it from.

## Security

- **Never enter SPID/CIE credentials.** Automatic login only uses the ATS Online Desk username and password.
- With automatic login, **the username and password are stored in Home Assistant's configuration**, which is not an encrypted secret store: anyone who can read the configuration files can read them. The manual-mode cookie also works like a password. If that is not acceptable, use manual mode.
- Credentials are only sent to the official ATS portal address: with a username and password configured, a different base URL is not accepted.
- Passwords and cookies are not written to the logs.
- The portal returns a `deprecated` header saying requests without an HMAC signature will eventually stop being accepted. This version does **not** implement HMAC.

## Limitations

- Fields are read from the JSON observed in the portal; the integration does not guess other endpoints and does not download invoice PDFs.
- A request working in the browser does not by itself guarantee the session can be reused from Home Assistant.
- Missing values stay `unknown`; they are not replaced with zero.
- Total billed consumption sums the `Consumption` field of the invoices received: it may span several periods and is not a cumulative meter reading.

## Troubleshooting

- **“ATS username or password not accepted”:** check that you are using the Online Desk credentials, not SPID/CIE. If you only log in with SPID/CIE, use manual mode.
- **“Login succeeded but customer and supplies could not be discovered”:** the portal may have changed structure; retry later or use manual mode.
- **Error when adding, or unavailable entities (manual mode):** check that the paths start with `/`, include their codes and that the cookie is still valid (see *Session lifetime*).
- **Reading sensors `unknown` (manual mode):** the readings path is missing; add it from **Configure**.
- **Entity IDs are not `sensor.ats_…`:** if you installed an earlier version, already registered IDs keep their old names. From the device page, ⋮ menu, use **Recreate entity IDs**.
- For logs, enable debug level for `custom_components.ats` and do not share lines containing cookies or codes.

## Reporting issues

Open a GitHub issue with your Home Assistant version, the integration version and the log error, **without** cookies or account codes.

## Credits

Automatic login and session renewal adapted from [EnrCol/ha-ats](https://github.com/EnrCol/ha-ats) (MIT), a fork of this project.

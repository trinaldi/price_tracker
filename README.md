# Price Tracker

A small Ruby on Rails application that monitors product prices on supermarket websites twice a day, stores price history in SQLite, and sends email notifications when prices change or a check fails.

It started as a hands-on project to get back into Ruby on Rails: small services with a single responsibility, specs for the main components, and a systemd timer instead of a hosted scheduler. There is no web UI, and SMTP is used for email delivery.

## Features

- Tracks multiple products defined in a YAML file
- Supports multiple supermarkets, each with its own price-fetching and extraction strategy
- Stores prices as integer cents, never as floats
- Supports regular and club prices
- Keeps price history per product, with one record per check, including failures
- Sends email notifications when a price changes or a check fails
- Generates an on-demand email report with the current price of every product
- Isolates failures so one broken product does not stop the others

## How it works

```text
systemd timer (09:00 and 21:00)
  -> bin/rails price:check
       -> price:sync          creates or updates products from config/products.yml
       -> CheckPriceJob       runs the Runner for every Product
            -> Runner
                 1. Markets.for(url)          finds the market by host
                 2. Fetches the product page using the market's strategy
                 3. Extracts the regular and optional club prices
                 4. Saves a PriceCheck and updates the Product
                 5. PriceMailer               sends notifications when needed
```

What happens on each check:

| Situation | PriceCheck | Product | Email |
|---|---|---|---|
| First successful check | ok | current price saved | none |
| Same price | ok | unchanged | none |
| Different price | ok | price and `last_changed_at` updated | price changed |
| Known failure (network, parsing, unknown market, configuration) | error | unchanged | check failed |

Unexpected errors for one product are logged and re-raised at the end of the job, after every other product has been checked. The service then exits with a failure status, which shows up in `systemctl --user --failed`.

## Data model

- `Product`: `slug`, `name`, `url`, `price_cents`, `club_price_cents`, `last_changed_at`. Holds the current state of a tracked product.
- `PriceCheck`: `product_id`, `status` (`ok` or `error`), `price_cents`, `error_message`, `checked_at`. Stores the check history.

Prices are stored as integer cents to avoid floating-point precision issues.

## Markets

Each supermarket has its own implementation under `app/services/price_tracker/markets/`.

The market registry identifies the appropriate implementation by matching the hostname of a product URL.

The `Defaults` module provides common behavior for market implementations, including store names, request headers, HTTP fetching, availability checks, and the default handling of club prices. Individual markets can override this behavior when their websites require a different approach.

Current markets:

- **Central Supermercados**: uses Ferrum to load dynamically rendered product pages.
- **Furlanetti**: supports regular and club pricing.
- **Jaú Serve**: uses a store identifier and postal code to select the correct pricing.
- **Simão Doces**: uses a fixed cookie to select the store.
- **Tenda Atacado**: retrieves product prices from its website.

To add a new market:

1. Write the spec first, using an HTML fixture containing the relevant price markup whenever possible.
2. Create the market implementation with its own fetching and price-extraction strategy.
3. Register it in `PriceTracker::Markets.all`.
4. Add the products to `config/products.yml`.

A product whose hostname has no registered market raises `UnknownMarket`. The failed check is recorded and reported without preventing other products from being checked.

## Configuration

### Products

`config/products.yml` defines the products being tracked.

```yaml
products:
  my_product:
    name: "Product name"
    url: "https://www.example.com/product-page"
```

Each product has a unique slug used as its identifier.

The YAML file stores product names and URLs. Prices and check history are stored in the database.

There are two synchronization tasks with different behaviors:

**`price:sync`**

Creates or updates products from the YAML file. Products that are not present in the file are left untouched.

**`products:sync`**

Performs a full reconciliation between the YAML file and the database. It displays the products that would be created, updated, or removed.

By default, this task runs in dry-run mode and makes no changes. To apply the synchronization:

```bash
CONFIRM=1 bin/rails products:sync
```

Warning: removing a product also deletes its associated `PriceCheck` records because the `Product` model uses `dependent: :destroy`.

### Environment variables

| Variable | Required | Description |
|---|---|---|
| `PRICE_ALERT_EMAIL` | yes | Address that receives the emails |
| `SMTP_ADDRESS` | for real emails | SMTP host, for example `smtp.gmail.com` |
| `SMTP_PORT` | no | Defaults to `587` |
| `SMTP_USERNAME` | for real emails | SMTP user |
| `SMTP_PASSWORD` | for real emails | SMTP password. With Gmail, use an app password |
| `JAUSERVE_CEP` | for Jaú Serve | Postal code used to select the store price |

Without `SMTP_ADDRESS`, emails open in the browser through `letter_opener` in development.

The application currently runs in the development environment on a personal machine.

## Getting started

Requirements: Ruby (see `.ruby-version`), Bundler and SQLite.

```bash
bundle install
bin/rails db:prepare
bundle exec rspec
```

Load the environment variables, then run the tasks:

```bash
set -a; source ~/.config/price-tracker/env; set +a

bin/rails price:sync     # create or update products from config/products.yml
bin/rails price:check    # synchronize products, then check every product
bin/rails price:report   # email the current price of every product
```

The `price:check` task depends on `price:sync`, so there is no need to run both separately.

## Scheduling with systemd

The checks run from a systemd user timer. The environment file is read by the service, so credentials survive reboots and stay out of the repository.

`~/.config/price-tracker/env` (create the directory with `chmod 700` and the file with `chmod 600`). Use `KEY=value` lines, without `export`:

```text
RAILS_ENV=development
PRICE_ALERT_EMAIL=you@example.com
SMTP_ADDRESS=smtp.gmail.com
SMTP_USERNAME=you@example.com
SMTP_PASSWORD=your-app-password
JAUSERVE_CEP=00000-000
```

`~/.config/systemd/user/price-tracker.service`:

```ini
[Unit]
Description=Check product prices

[Service]
Type=oneshot
WorkingDirectory=/path/to/price_tracker
EnvironmentFile=%h/.config/price-tracker/env
Environment=PATH=%h/.rbenv/shims:%h/.rbenv/bin:/usr/local/bin:/usr/bin:/bin
ExecStartPre=/usr/bin/bash -c 'for i in $(seq 1 30); do getent hosts www.example.com >/dev/null && exit 0; sleep 2; done; exit 1'
ExecStart=%h/.rbenv/shims/bundle exec bin/rails price:check
```

`~/.config/systemd/user/price-tracker.timer`:

```ini
[Unit]
Description=Price check twice a day

[Timer]
OnCalendar=*-*-* 09,21:00:00
Persistent=true
RandomizedDelaySec=300

[Install]
WantedBy=timers.target
```

- `Persistent=true` runs a missed check once the machine is back on and the user is logged in.
- `RandomizedDelaySec` spreads the requests over a few minutes.
- The `ExecStartPre` line waits for the network, so a check right after boot does not fail because of Wi-Fi.
- The `PATH` line is needed so systemd finds the Ruby installed by rbenv.

Enable and inspect:

```bash
systemctl --user daemon-reload
systemctl --user enable --now price-tracker.timer
systemctl --user list-timers --no-pager
systemctl --user start price-tracker.service
journalctl --user -u price-tracker.service -n 50 --no-pager -a
```

## Tests

```bash
bundle exec rspec
```

- Real HTTP requests are blocked by WebMock. Fetcher specs stub every request.
- Market specs use HTML fixtures containing the relevant price markup.
- Central has its own market spec for its browser-based fetching strategy.
- Runner specs use a fake market and stub the mailer, covering the orchestration independently.

## A note on scraping

This is a personal tool. It checks each product twice a day. Depending on the supermarket, it uses regular HTTP requests or browser automation to retrieve the product page.

Check the terms of each website before adding it.

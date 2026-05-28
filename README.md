# Contrail

> ADS-B flight tracking pipeline with historical playback — SDR receiver to MySQL to interactive map.

Contrail is a self-hosted ADS-B flight tracking system built around a RTL-SDR V3 receiver and Dump1090. It captures aircraft position data continuously, stores it in MySQL, and serves it through a Leaflet.js web interface with a selectable time window — so you can watch live traffic or replay any window of history.

The built-in Dump1090 web interface shows real-time positions but doesn't record tracks or let you go back in time. Contrail fills that gap.

---

## Features

- **Continuous ingestion** — Python poller reads Dump1090's live JSON output and writes position records to MySQL every second
- **Full track history** — every position, altitude, speed, heading, and squawk code stored with a timestamp
- **Interactive map** — Leaflet.js frontend with aircraft markers and rendered flight tracks
- **Selectable time window** — default last 3 hours, adjustable up to 24 hours or any historical range
- **Historical playback** — scrub back through stored data to any point in time
- **Least-privilege DB design** — separate ingest and read-only MySQL users; ingest account cannot query data

---

## Stack

| Layer | Technology |
|-------|------------|
| SDR receiver | RTL-SDR V3 |
| ADS-B decoder | Dump1090-fa |
| Ingestion | Python (custom poller) |
| Database | MySQL / MariaDB |
| API | Flask or FastAPI |
| Frontend | Leaflet.js + vanilla JS |
| Host | Proxmox VM on Odin (Ubuntu 24.04) |

---

## How It Works

```
RTL-SDR V3 dongle
       ↓
  Dump1090-fa
  (decodes 1090MHz ADS-B signals)
       ↓
  /run/dump1090-fa/aircraft.json
  (updates every ~1 second)
       ↓
  contrail-ingest.py
  (polls JSON, filters, writes to MySQL)
       ↓
  MySQL — contrail database
  (flights + positions tables)
       ↓
  contrail-api.py (Flask/FastAPI)
  (queries by time window, serves JSON)
       ↓
  Leaflet.js frontend
  (renders markers, tracks, time controls)
```

---

## Database Schema

### flights

Tracks unique aircraft seen during a session. One row per ICAO hex address per session.

```sql
CREATE TABLE flights (
    id          INT AUTO_INCREMENT PRIMARY KEY,
    icao_hex    VARCHAR(6)      NOT NULL,
    callsign    VARCHAR(10),
    first_seen  DATETIME        NOT NULL,
    last_seen   DATETIME        NOT NULL,
    INDEX idx_icao (icao_hex),
    INDEX idx_first_seen (first_seen)
);
```

### positions

One row per position report received from Dump1090. High write volume — indexed for time-window queries.

```sql
CREATE TABLE positions (
    id          INT AUTO_INCREMENT PRIMARY KEY,
    icao_hex    VARCHAR(6)      NOT NULL,
    callsign    VARCHAR(10),
    timestamp   DATETIME        NOT NULL,
    lat         DECIMAL(9,6),
    lon         DECIMAL(9,6),
    altitude    INT,
    speed       INT,
    heading     SMALLINT,
    vert_rate   INT,
    squawk      VARCHAR(4),
    on_ground   TINYINT(1)      DEFAULT 0,
    INDEX idx_icao (icao_hex),
    INDEX idx_timestamp (timestamp),
    INDEX idx_icao_time (icao_hex, timestamp)
);
```

---

## Database Users

Contrail uses two MySQL users — ingest writes, the API reads. Neither has more access than it needs.

```sql
-- ingest account — INSERT only, no SELECT
CREATE USER 'contrail_ingest'@'localhost' IDENTIFIED BY 'strongpassword';
GRANT INSERT ON contrail.flights TO 'contrail_ingest'@'localhost';
GRANT INSERT ON contrail.positions TO 'contrail_ingest'@'localhost';

-- read account — SELECT only, no writes
CREATE USER 'contrail_read'@'localhost' IDENTIFIED BY 'strongpassword';
GRANT SELECT ON contrail.* TO 'contrail_read'@'localhost';

FLUSH PRIVILEGES;
```

---

## Project Structure

```
contrail/
├── ingest/
│   ├── contrail-ingest.py      # Dump1090 JSON poller → MySQL
│   └── requirements.txt
├── api/
│   ├── app.py                  # Flask/FastAPI — time window queries
│   └── requirements.txt
├── frontend/
│   ├── index.html              # Leaflet map + time controls
│   ├── map.js                  # Track rendering, marker updates
│   └── style.css
├── sql/
│   ├── schema.sql              # Full schema — flights + positions
│   └── users.sql               # DB user creation (no passwords)
├── systemd/
│   └── contrail-ingest.service # systemd unit for ingest daemon
├── docs/
│   └── setup.md                # Full setup walkthrough
├── .env.example                # Environment variable template
├── .gitignore
└── README.md
```

---

## Setup

### Prerequisites

- RTL-SDR V3 dongle + 1090MHz antenna
- Dump1090-fa installed and running
- MySQL or MariaDB
- Python 3.10+
- A Linux host (bare metal, VM, or Pi)

### 1 — Install Dump1090-fa

```bash
sudo apt update
sudo apt install dump1090-fa
```

Verify it's receiving:

```bash
cat /run/dump1090-fa/aircraft.json | python3 -m json.tool | head -40
```

### 2 — Create the database and users

```bash
mysql -u root -p < sql/schema.sql
mysql -u root -p < sql/users.sql
```

### 3 — Configure environment

```bash
cp .env.example .env
# edit .env — set DB credentials, dump1090 JSON path, poll interval
```

### 4 — Install Python dependencies

```bash
cd ingest && pip install -r requirements.txt
cd ../api && pip install -r requirements.txt
```

### 5 — Run the ingest daemon

```bash
# test run first
python3 ingest/contrail-ingest.py

# install as systemd service
sudo cp systemd/contrail-ingest.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now contrail-ingest
```

### 6 — Run the API

```bash
python3 api/app.py
```

### 7 — Open the frontend

Point a browser at your host IP. The map defaults to the last 3 hours of traffic.

---

## Hardware Notes

**RTL-SDR V3** — works out of the box on Linux with `rtl-sdr` package. No driver install required on most distros.

**Antenna** — a short 1090MHz dipole is sufficient for 100–150nm range under good conditions. A coaxial dipole or FlightAware Pro Stick Plus with built-in 1090MHz filter will extend range significantly if you want to improve coverage later.

**915MHz antennas will not work for ADS-B** — 915 and 1090MHz are too far apart for useful signal reception.

---

## Planned

- [ ] Contrail ingestion script — first working version
- [ ] Schema finalized and tested
- [ ] Flask API — time window endpoint
- [ ] Leaflet frontend — live markers
- [ ] Leaflet frontend — historical track rendering
- [ ] Time window selector UI (default 3h, adjustable to 24h+)
- [ ] Historical playback scrubber
- [ ] Systemd unit for ingest daemon
- [ ] Data retention policy / purge job for old positions
- [ ] Docker Compose deployment option
- [ ] Aircraft type lookup (ICAO hex → type/registration)

---

## Why Not Just Use FlightAware / ADS-B Exchange?

Those are great services for coverage contribution and lookup. Contrail is for people who want to own their data, run their own infrastructure, and have full historical access without depending on an external service or internet connectivity.

---

## License

MIT

---

*Built by Cole — part of the Crossroads IT homelab stack.*

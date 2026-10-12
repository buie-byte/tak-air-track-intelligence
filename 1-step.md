1. Open the project directory on `tak-sns01`, enter:
   ```bash
   cd ~/tak-airtrack
   ```
   If the directory doesn't exist yet:
   ```bash
   mkdir -p ~/tak-airtrack/src
   cd ~/tak-airtrack
   ```
2. Create the Python file:
   ```bash
   nano ~/tak-airtrack/src/aircraft_classifier.py
   ```

Paste the following complete code:
```bash
#!/usr/bin/env python3

import csv
import gzip
import json
import re
import sys
from pathlib import Path

DATABASE = Path.home() / "adsb2tak/database/aircraft.csv.gz"

HELICOPTER_TYPES = {
    "R22", "R44", "R66",
    "B06", "B407", "B412", "B429",
    "EC20", "EC30", "EC35", "EC45",
    "H125", "H130", "H135", "H145", "H160",
    "AS50", "AS55",
    "S76", "S92",
    "UH1", "UH60", "H60", "AH64",
    "CH47", "CH53",
    "MI8", "MI17", "KA32",
}

FIXED_WING_PREFIXES = (
    "B7", "B3", "A3", "A2",
    "E1", "E2", "E7",
    "CRJ", "C17", "C130",
    "F16", "F15", "F18",
)

def load_database():
    aircraft = {}

    with gzip.open(
        DATABASE,
        "rt",
        encoding="utf-8-sig",
        errors="replace"
    ) as file:

        reader = csv.reader(file, delimiter=";")

        for row in reader:
            if len(row) < 7:
                continue

            icao = row[0].strip().upper()

            if not re.fullmatch(r"[0-9A-F]{6}", icao):
                continue

            aircraft[icao] = {
                "registration": row[1].strip(),
                "type": row[2].strip().upper(),
                "flags": row[3].strip(),
                "description": row[4].strip(),
                "year": row[5].strip(),
                "operator": row[6].strip(),
            }

    return aircraft


def classify_airframe(type_code, description):
    type_code = type_code.upper()
    description = description.upper()

    if type_code in HELICOPTER_TYPES:
        return "rotary_wing"

    if any(
        keyword in description
        for keyword in ("HELICOPTER", "ROTORCRAFT")
    ):
        return "rotary_wing"

    if type_code.startswith(FIXED_WING_PREFIXES):
        return "fixed_wing"

    if any(
        keyword in description
        for keyword in (
            "BOEING",
            "AIRBUS",
            "EMBRAER",
            "CESSNA",
            "PIPER",
            "BOMBARDIER",
        )
    ):
        return "fixed_wing"

    return "unknown"


def classify_aircraft(icao, database):
    icao = icao.strip().upper()

    record = database.get(icao)

    if not record:
        return {
            "icao": icao,
            "classification": "unknown",
            "airframe": "unknown",
            "operator": None,
            "aircraft_type": None,
            "registration": None,
        }

    try:
        flags = int(record["flags"], 16)
    except ValueError:
        flags = 0

    military = bool(flags & 0x10)

    classification = (
        "military" if military else "unknown"
    )

    return {
        "icao": icao,
        "classification": classification,
        "airframe": classify_airframe(
            record["type"],
            record["description"]
        ),
        "operator": record["operator"] or None,
        "aircraft_type": record["type"] or None,
        "description": record["description"] or None,
        "registration": record["registration"] or None,
        "year": record["year"] or None,
        "flags": record["flags"],
        "source": "local_database",
    }


def main():
    if not DATABASE.exists():
        print(f"Database not found: {DATABASE}")
        sys.exit(1)

    print("Loading aircraft database...")

    database = load_database()

    print(f"Loaded {len(database):,} aircraft records.\n")

    if len(sys.argv) > 1:
        icao = sys.argv[1]
    else:
        icao = input("Enter aircraft ICAO24: ")

    result = classify_aircraft(icao, database)

    print(json.dumps(result, indent=4))


if __name__ == "__main__":
    main()
```
   


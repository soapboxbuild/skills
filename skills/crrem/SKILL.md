---
name: crrem
description: Use when asked about real estate decarbonization, carbon pathway compliance, CRREM stranding risk, ESG reporting, net-zero targets, or benchmarking building energy performance against climate targets. Triggers on: CRREM, stranding risk, carbon pathway, decarbonization, real estate ESG, net-zero building, kgCO2/m2.
---

# CRREM Pathways Data

CRREM v2.05 decarbonization pathway data is in the public GitHub repo `soapboxbuild/crrem-data`.

## Fetching data (GitHub MCP)

Raw base URL: `https://raw.githubusercontent.com/soapboxbuild/crrem-data/master`

| What you need | File path |
|---|---|
| Pathway targets for a country | `data/pathways/{COUNTRY}.json` |
| Emission factors for a country | `data/emissions/{COUNTRY}.json` |
| US climate zones by ZIP first digit | `data/postal/US/{digit}.json` |
| Australian climate zones | `data/postal/AU.json` |
| Canadian climate zones | `data/postal/CA.json` |

Country codes: ISO-2 uppercase (DE, US, GB, FR, ES, IT, NL, SE, NO, DK, FI, AT, CH, BE, PL, CZ, HU, RO, BG, HR, SK, SI, EE, LV, LT, IE, PT, LU, CY, MT, ...).

## Data format

**`data/pathways/DE.json`** — indexed property_type → year (string keys):
```json
{
  "office": {
    "2020": { "carbon_kgco2_m2yr": 46.2, "energy_kwh_m2yr": 180.1 },
    "2025": { "carbon_kgco2_m2yr": 36.3, "energy_kwh_m2yr": 147.7 },
    "2030": { "carbon_kgco2_m2yr": 26.8, "energy_kwh_m2yr": 120.0 },
    "2050": { "carbon_kgco2_m2yr": 0.0,  "energy_kwh_m2yr": 55.0 }
  },
  "residential": { ... }
}
```

**`data/emissions/DE.json`** — indexed carrier → year (string keys):
```json
{
  "electricity": { "2020": 0.336, "2025": 0.297, "2030": 0.226, "2050": 0.020 },
  "gas":         { "2020": 0.201, "2025": 0.201, "2030": 0.201, "2050": 0.201 }
}
```

Note: year keys are **strings** in the JSON (e.g. `"2025"`, not `2025`).

## Calculations

**Carbon intensity** (kgCO₂/m²yr):
```
carbon_intensity = Σ(annual_energy_kwh × carrier_fraction × emission_factor) / floor_area_m2
```

Get `emission_factor` from `emissions/{country}.json` for the relevant carrier and year (interpolate if needed).

**Linear interpolation** between two known years a and b:
```
value(year) = value(a) + (year - a) / (b - a) × (value(b) - value(a))
```
Clamp to the nearest known year if out of range.

**Stranding year** — find the first year ≥ current year where `pathway_target < actual_carbon_intensity`. Iterate year by year from the current year through 2050, interpolating the pathway at each step. If the building is already above the pathway, it is currently stranded. If never below the pathway through 2050, stranding_year = null.

## CRREM property types (v2.05)

The property types in the JSON use the CRREM internal names. Common ones:
`office`, `retail`, `residential`, `hotel`, `logistics`, `industrial`, `healthcare`, `education`, `mixed_use`, `data_centre`, `retail_warehouse`, `supermarket`, `leisure`, `senior_living`, `student_housing`, `self_storage`, `car_park`

Check the keys in the country's pathways JSON for the exact available types.

## Worked example

Building: German office, 2,000 m², 300,000 kWh/yr (70% electricity, 30% gas), year 2026

1. Fetch `data/pathways/DE.json` → pathway for office at 2026 (interpolate between 2025=36.3 and 2030=26.8) ≈ 34.1 kgCO₂/m²yr
2. Fetch `data/emissions/DE.json`
3. Electricity: 300,000 × 0.70 = 210,000 kWh × 0.291 kgCO₂/kWh (2026 interpolated) = 61,110 kgCO₂
4. Gas: 300,000 × 0.30 = 90,000 kWh × 0.201 kgCO₂/kWh = 18,090 kgCO₂
5. Total carbon = 79,200 kgCO₂ → intensity = 79,200 / 2,000 = **39.6 kgCO₂/m²yr**
6. Gap = 39.6 − 34.1 = **+5.5 kgCO₂/m²yr over pathway** → building is stranded in 2026

## Report structure

A CRREM assessment should include:
1. Building summary (name, country, type, floor area)
2. Carbon performance: actual intensity vs pathway target vs gap
3. Stranding status and year
4. Energy mix with per-carrier emission factors
5. Pathway table (every 5 years, target vs actual)
6. Numbered recommendations (electrification, efficiency, renewables)

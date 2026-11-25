# Data Sourcing Documnetation.

The plan is to gather data on

 1. Charing rate (megawatts) 

 2. Discharging rate (megawatts) 

 3. Renewable source rate (percentage or megawatts) 

 4. Non Renewable source rate (percentage or megawatts) 

 5. Electricity demand(megawatts) 

 6. wholesale Prices (pounds/per megawatt) 

 7. Battery Action (charge, discharge) (1-0) 

 The first idea was to gather electricity generation data from energydashboard. 

 ## Liscence Check

 At the page: http://energydashboard.co.uk/api-info

 We can see that they have limited use on their API for free, with the explanation:

**What data is available?**

Data includes half-hourly Electricity Generation, Carbon Intensity & Demand for Great Britain, plus GB Gas NTS instantaneous flows and gas-day historical energy:

 * Power Plant generation and demand data for 300+ wind farms, power stations and BESS facilities
 * GB Gas NTS instantaneous flows — supply, demand and linepack with category breakdowns (UKCS, Norwegian pipelines, LNG, interconnectors, storage, LDZ, power stations)
 * Historical gas-day energy (GWh) by supply and demand category
 * Live (real-time) & recent data (today / two-day / 24h / 48h)
 * Historical dashboard data: back to 2009, with options to select specific date ranges and data intervals
 * Forecast data for the day ahead
 * Generation data (breakdown by mix & type), including embedded wind & solar, and imports breakdown
 * Carbon Intensity with Index (high, low etc for National & Regional), CO2 emissions & source breakdown
 * Demand data with actual net and gross figures (allowing for embedded data) & exports breakdown
 * Data available as REST API JSON data or CSV download

### Solution

I subscribed to their free tier acess to API, which has 1000 request limit and 4 endpoints

/generation/latest
/generation/latest/five-minute
/carbon-intensity/latest
/demand/latest

### New Problem

Since historic data is behind a paywall, we will be sourcing the necessary data from NESO. They are Open data from Great Britain’s System Operator. They have a liscence in their data which specifies that we need to recognise NESO as a contributor in the project:

National Energy SO Open Data Licence v1.0

    These licence terms and conditions apply to the National Energy System Operator (“NESO”) datasets as specified within the NESO data portal service and are based on version 3.0 of the Open Government Licence with specific amendments for NESO (the “Licence”).  

    NESO may at any time revise this Licence without notice. It is up to you (“You”) to regularly review the Licence, which will be available on this website, in case there are any changes. After a change has been made to the Licence, Your continued use of the data feeds will be treated as Your acceptance of that change.

    You are encouraged to use and re-use the Information that is available under this licence freely and flexibly, with only a few conditions.

    Using Information under this licence

    Use of copyright and database right material expressly made available under this licence (the ‘Information’) indicates your acceptance of the terms and conditions below as varied from time to time.

    The Licensor grants you a worldwide, royalty-free, perpetual, non-exclusive licence to use the Information subject to the conditions below.

    This licence does not affect your freedom under fair dealing or fair use or any other copyright or database right exceptions and limitations.

    You are free to:

        - copy, publish, distribute and transmit the Information;
        - adapt the Information;
        - exploit the Information commercially and non-commercially for example, by combining it with other Information, or by including it in your own product or application.
    

    You must (where you do any of the above):

        - Acknowledge NESO as the source of the Information by including the following attribution statement ‘Supported by National Energy SO Open Data’
        - Ensure our other intellectual property rights, including all logos, design rights, patents and trademarks, are protected.

## Process

* Downloaded 2016–2025 Historic Demand Data from the NESO Data Portal, one .csv file per year
* Source: National Energy System Operator (NESO), released under the NESO Open Data Licence
* Resolution: half-hourly (48 settlement periods per day; 46 or 50 on clock-change days)
* Units: all generation, demand, capacity and flow values are in megawatts (MW)

 + Contents:

### Time variables

| Variable | Description | Decision | Reason |
|---|---|---|---|
| `SETTLEMENT_DATE` renamed `datetime` | Date of the settlement day (YYYY-MM-DD) | **Keep** | Combined with the settlement period to build the timestamp index used to join all other datasets (prices, gas, generation mix). |
| `SETTLEMENT_PERIOD` | Half-hour period of the day (1 = 00:00–00:30, up to 48) | **Keep** | Needed for the timestamp; matches Elexon's price data convention. Clock-change days must be handled separately. |

### Demand variables

| Variable | Description | Decision | Reason |
|---|---|---|---|
| `ND` | National Demand: GB demand met by transmission-connected generation; excludes station load, pumped storage pumping and interconnector exports | **Keep** | Main demand feature. Higher demand moves the market up the merit order, raising prices. Used to derive residual demand. |
| `TSD` | Transmission System Demand: ND plus station load, pumped storage pumping and interconnector exports | **Cut** | Total load the transmission system had to supply. Not used alongside ND in the same model, to avoid collinearity. |
| `ENGLAND_WALES_DEMAND` | Demand in England and Wales only | **Cut** | GB has a single national wholesale price, so a regional subset adds little and overlaps heavily with ND. |

### Embedded (distribution-connected) generation

| Variable | Description | Decision | Reason |
|---|---|---|---|
| `EMBEDDED_WIND_GENERATION` | Estimated output of wind farms connected to local distribution networks (not visible to the transmission system) | **Keep** | Renewable output lowers prices and widens intraday spreads. Needed for residual demand. NESO estimate, not metered. |
| `EMBEDDED_WIND_CAPACITY` | Installed capacity of embedded wind | **Cut** | barely changes and NESO says it isn’t definitive. |
| `EMBEDDED_SOLAR_GENERATION` | Estimated output of distribution-connected solar PV | **Keep** | Drives midday price dips, a key source of battery charging opportunities. Needed for residual demand. NESO estimate, not metered. |
| `EMBEDDED_SOLAR_CAPACITY` | Installed capacity of embedded solar | **Keep** | Trend variable for solar growth; can be used to normalise generation. |

### Storage and balancing

| Variable | Description | Decision | Reason |
|---|---|---|---|
| `PUMP_STORAGE_PUMPING` | Demand from pumped hydro storage stations while pumping (storing energy) | **Keep** | Pumped storage behaves like a large battery; its pumping times show when operators judged power to be cheap. Benchmark for battery charging behaviour. |
| `NON_BM_STOR` | Short Term Operating Reserve provided outside the Balancing Mechanism | **Cut** | Balancing reserve service, mostly zero, and not related to wholesale arbitrage, which is the scope of this project. |

### Transfers and interconnectors

| Variable | Description | Decision | Reason |
|---|---|---|---|
| `SCOTTISH_TRANSFER` | Power flow from Scotland to England across the B6 boundary | **Cut** | Proxy for north–south network congestion, which matters when Scottish wind output is high. Even though it was important, the data for 2022 does not exist hence we drop it.|
| `IFA_FLOW` | Interconnector flow, GB–France (IFA) | **Keep (aggregated)** | Combined into net interconnector flow (see below). |
| `IFA2_FLOW` | Interconnector flow, GB–France (IFA2) | **Keep (aggregated)** | As above. |
| `BRITNED_FLOW` | Interconnector flow, GB–Netherlands | **Keep (aggregated)** | As above. |
| `MOYLE_FLOW` | Interconnector flow, GB–Northern Ireland | **Keep (aggregated)** | As above. |
| `EAST_WEST_FLOW` | Interconnector flow, GB–Ireland | **Keep (aggregated)** | As above. |
| `NEMO_FLOW` | Interconnector flow, GB–Belgium | **Keep (aggregated)** | As above. |
| `NSL_FLOW` | Interconnector flow, GB–Norway (North Sea Link) | **Keep (aggregated)** | As above. Zero before the link was commissioned. |
| `ELECLINK_FLOW` | Interconnector flow, GB–France (ElecLink) | **Keep (aggregated)** | As above. Zero before the link was commissioned. |
| `VIKING_FLOW` | Interconnector flow, GB–Denmark (Viking Link) | **Keep (aggregated)** | As above. Zero before the link was commissioned. |
| `GREENLINK_FLOW` | Interconnector flow, GB–Ireland (Greenlink) | **Keep (aggregated)** | As above. Zero before the link was commissioned. |

> Interconnector sign convention: positive = import into GB, negative = export from GB.

### Derived variables

| Variable | Definition | Purpose |
|---|---|---|
| `datetime` | Built from `SETTLEMENT_DATE` + `SETTLEMENT_PERIOD`, timezone-aware (Europe/London) | Common index for joining all datasets. |
| `NET_INTERCONNECTOR_FLOW` | Sum of all `*_FLOW` columns | Single measure of whether GB is importing (cheaper than neighbours) or exporting. |
| `RESIDUAL_DEMAND` | `ND` minus embedded wind and solar (and transmission-connected wind/solar once the generation mix data is added) | Demand left for dispatchable (mostly gas) generation; expected to be the strongest demand-side price driver. |

### Notes and caveats

* Embedded wind and solar figures are NESO model estimates, not metered values.
* Column sets may differ between years; headers were checked before combining files.
* Interconnectors that did not yet exist appear as zero, not missing.
 
For the cleaning, we have

* Converted the date into UTC datetime (daylight sensitive) with checks in the conversion

* Dropped colums : "ENGLAND_WALES_DEMAND", "NON_BM_STOR","SETTLEMENT_DATE", "SCOTTISH_TRANSFER"

* Created columns:
    + "NET_INTERCONNECTOR" = Sum of all columns ending with"_FLOW". Positive values mean GB is importing overall.
    + "RESIDUAL_DEMAND" = "ND" - "EMBEDDED_WIND_GENERATION" - "EMBEDDED_SOLAR_GENERATION" to find the missing demand when Wind and Solar power is provided.




 
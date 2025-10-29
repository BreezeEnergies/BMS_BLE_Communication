# BMS BLE Communication

All necessary information for data retrieval from Breeze batteries via BLE (Bluetooth Low Energy) CDT (Connectionless Data Transfer).
CDT is availabe only for newer models of BMS (produced after 2024).

## Verify CDT in your batteries

1. Download `nRF Connect` from [Play Store](https://play.google.com/store/apps/details?id=no.nordicsemi.android.mcp&hl=pl&pli=1) or [Apple Store](https://apps.apple.com/pl/app/nrf-connect-for-mobile/id1054362403?l=pl)

2. Scan for Bluetooth devices 

3. Click on detected bluetooth device with name as `Serial Number` of battery

4. If there is string of data in `Manufacturer data`, than battery have `CDT` capabilites. If there is no `Manufacturer data` option, then this is older version that don't have CDT implementation.

## CDT Communication 

For this communication all necessary data about batteries from BMS is retrieved in `Manufacturer Data` property *during* bluetooth scan.
If BMS reboots or is connected via P2P to device - the whole `Manufacturer Data` property will be all `zeros`!

## Processing Data

------- ADV frame updates (CDT) V1.1 (05.2024) -------

- `SOC - State Of Charge`
- `SOH - State Of Health`
- `LC  - Number of Cycles`
- `SEC - Security`
- `DCL - Discharge Current Limit`
- `CCL - Charge Current Limit`

| [byte] | [data] |
|---|---|
| 1  | min cell voltage 1/2 |
| 2  | min cell voltage 2/2 |
| 3  | max cell voltage 1/2 |
| 4  | max cell voltage 2/2 |
| 5  | Voltage 1/2 | 
| 6  | Voltage 2/2 |
| 7  | Current 1/2 |
| 8  | Current 2/2 |
| 9  | SOC |
| 10  | SOH |
| 11  | LC 1/2 |
| 12  | LC 2/2 |
| 13  | SEC 1/2 |
| 14  | SEC 2/2 |
| 15  | Temp min |
| 16  | Temp max |
| 17  | DCL 1/2 |
| 18  | DCL 2/2 |
| 19  | CCL 1/2 |
| 20  | CCL 2/2 |
| 21  |  |
| 22  |  |
| 23  |  |
| 24  |  |
| 25  |  |


## ESP with Arduino example 

1. Install BLE library, eg.: [NimBLE-Arduino](https://github.com/h2zero/NimBLE-Arduino)  

2. Processing example

- `_strNameBLE[i].c_str()` - serial number of Breeze Battery

```cpp
class scanCallbacks : public NimBLEScanCallbacks
{
    void onResult(const NimBLEAdvertisedDevice* advertisedDevice) override {

        if (strcmp(_strNameBLE[i].c_str(), advertisedDevice->getName().c_str()) == 0 
            && !_BreezeBat[i].state.Processed )
        {
            std::string sAdvData = {advertisedDevice->getManufacturerData()};
            _BreezeBat[i].state.RSSI = advertisedDevice->getRSSI();

            _BreezeBat[i].cells.VoltageMin = (sAdvData[1] << 8 | sAdvData[0]);
            _BreezeBat[i].cells.VoltageMax = (sAdvData[3] << 8 | sAdvData[2]);

            if ((_BreezeBat[i].cells.VoltageMax - _BreezeBat[i].cells.VoltageMin > _treshold_imballancedCells) && (_BreezeBat[i].cells.VoltageMax > 3500))
            {
                _BreezeBat[i].state.ImballancedCells = true;
            }
            else
            {
                _BreezeBat[i].state.ImballancedCells = false;
            }

            _BreezeBat[i].baseData.Voltage = (sAdvData[4] << 8 | sAdvData[5]);

            int current_bat = (sAdvData[6] << 8 | sAdvData[7]);
            _BreezeBat[i].baseData.Current = current_bat > 32500 ? (65536 - current_bat) * -1 : current_bat;

            _BreezeBat[i].baseData.SOC = (sAdvData[8]);
            _BreezeBat[i].baseData.SOH = (sAdvData[9]);

            _BreezeBat[i].baseData.LC = (sAdvData[10] << 8 | sAdvData[11]); 

            _BreezeBat[i].baseData.SECURITY = (sAdvData[12] << 8 | sAdvData[13]);

            _BreezeBat[i].temperature.MIN = static_cast<int>(sAdvData[14]) * 4; 
            _BreezeBat[i].temperature.MAX = static_cast<int>(sAdvData[15]) * 4;

            int current_dcl = (sAdvData[16] << 8 | sAdvData[17]); // extract
            _BreezeBat[i].limits.DischargeCurrentLimit = current_dcl > 32500 ? 65536 - current_dcl : current_dcl; // remove overloading

            _BreezeBat[i].limits.ChargeCurrentLimit = (sAdvData[18] << 8 | sAdvData[19]);
            _BreezeBat[i].limits.LimDataRetrieved = true;

            // Adjust using SEC frame (MOSFET data is not acquired)
            const uint16_t MASK_ROZLADOWANIE = (1 << 3) | (1 << 6) | (1 << 7) | (1 << 8) | (1 << 10) | (1 << 12);
            const uint16_t MASK_LADOWANIE = (1 << 2) | (1 << 4) | (1 << 5) | (1 << 9) | (1 << 11);
            const uint16_t MASK_BAT_OFF = (1 << 1) | (1 << 13);

            uint16_t security = _BreezeBat[i].baseData.SECURITY;

            if (security & MASK_BAT_OFF)
            {
                _BreezeBat[i].limits.BatDischargeOn = false;
                _BreezeBat[i].limits.BatChargeOn = false;
            }
            else
            {
                _BreezeBat[i].limits.BatDischargeOn = (security & MASK_ROZLADOWANIE) ? false : true;
                _BreezeBat[i].limits.BatChargeOn = (security & MASK_LADOWANIE) ? false : true;
            }
        }
    }
}
```

Battery Structure Example

```cpp
struct BREEZE_BAT
{
    struct State
    {
        bool DataRetrieved{};          // true if data retrieved
        bool ImballancedCells{};       // true if cells imbalanced (in comparison to eg.: >_treshold_imballancedCells)
        int ConnectionError{};         // number of critical connection errors
        int ScanAttempt{};             // number of scan attempts
        int RSSI{};                    // Received Signal Strength Indication in dBm - logarythmic scale
        const int minVoltage48 = 3500; // mV for 48V system
        const int maxVoltage48 = 7000; // mV for 48V system
        bool Processed = false;        // acquired data within scan before update
    } state;
    struct BaseData
    {
        int Voltage{}; // mV
        int Current{}; // mA
        int SOC{};
        int SOH{};
        int LC{};
        int MOSFET{};
        int SECURITY{};
    } baseData;
    struct Temperature
    {
        int MAX{};    // Celsius = * 0.1
        int MIN{};    // Celsius = * 0.1
        int MOSFET{}; // Celsius = * 0.1
    } temperature;
    struct Limits
    {
        // If both == false -> Baterry is OFF
        bool BatChargeOn{};
        bool BatDischargeOn{};
        // ---
        bool LimDataRetrieved{};
        int ChargeCurrentLimit{};    // A = * 0.1
        int DischargeCurrentLimit{}; // A = * 0.1
    } limits;
    struct Cells
    {
        int VoltageMin{}; // mV
        int VoltageMax{}; // mV
    } cells;
};
```

## Note

In some batteries CCL and DCL can be seen as `60` which is incorrect. Correct value for these batteries is `50`.

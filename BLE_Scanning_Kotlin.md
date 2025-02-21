Below is a comprehensive tutorial that walks you through creating a Bluetooth Low Energy (BLE) scanning function in Kotlin—from setting up your Bluetooth objects with lazy initialization to applying filters, handling results asynchronously, and presenting the found devices.

---

# Creating a BLE Scanning Function in Kotlin: A Comprehensive Tutorial

In this tutorial, you’ll learn how to create a BLE scan function using Kotlin on Android. We’ll cover the following topics:

- **Lazy Initialization** of Bluetooth components  
- **Starting a BLE scan** with custom scan settings and callbacks  
- **Applying filters** (including combining conditions and using wildcards for UUIDs)  
- **Collecting and presenting scan results** asynchronously  

Let’s dive in!

---

## 1. Setting Up Bluetooth Components with Lazy Initialization

When working with BLE in Android, you’ll need a reference to a `BluetoothAdapter` and its corresponding `BluetoothLeScanner`. Using Kotlin’s `by lazy` delegate, you can delay the creation of these objects until they are needed.

```kotlin
// Lazily initialize the BluetoothAdapter
private val bluetoothAdapter: BluetoothAdapter by lazy {
    val bluetoothManager = getSystemService(Context.BLUETOOTH_SERVICE) as BluetoothManager
    bluetoothManager.adapter
}

// Lazily initialize the BluetoothLeScanner from the adapter
private val bleScanner by lazy {
    bluetoothAdapter.bluetoothLeScanner
}
```

**Explanation:**

- **`by lazy { ... }`** delays execution of the initialization block until the property is first accessed.
- This ensures efficient use of resources, especially if your app may not always need to scan for BLE devices.

---

## 2. Writing a Basic BLE Scan Function

Let’s create a function that starts a BLE scan with a set of predefined scan settings and a callback to process the results.

```kotlin
private fun startBleScan() {
    // Set up scan settings for low latency (faster device discovery)
    val scanSettings = ScanSettings.Builder()
        .setScanMode(ScanSettings.SCAN_MODE_LOW_LATENCY)
        .build()

    // Define a ScanCallback to handle discovered devices and errors
    val scanCallback = object : ScanCallback() {
        override fun onScanResult(callbackType: Int, result: ScanResult) {
            Log.i("ScanCallback", "Device Name: ${result.device.name ?: "No Name"}")
            Log.i("ScanCallback", "Device Address: ${result.device.address}")
        }

        override fun onScanFailed(errorCode: Int) {
            Log.e("ScanCallback", "Scan failed with error code: $errorCode")
        }
    }

    // Start scanning without filters (matches any BLE device)
    bleScanner.startScan(null, scanSettings, scanCallback)
}
```

**Key Points:**

- **ScanSettings:** Configured to use low latency for faster responses.
- **ScanCallback:** Receives individual scan results and error codes.
- **Non-Blocking:** The function starts the scan asynchronously, and scan results are delivered via the callback.

---

## 3. Applying Filters to Your BLE Scan

Sometimes, you only want to discover devices that match specific criteria. Android’s BLE API allows you to provide a list of `ScanFilter` objects. Note that the API uses a logical **OR** when combining filters.

### Example: Multiple Filters for Device Name, Address, and Service UUID

```kotlin
private fun startFilteredBleScan() {
    // Filter 1: Devices with an exact name "DeviceA"
    val nameFilter = ScanFilter.Builder()
        .setDeviceName("DeviceA")
        .build()

    // Filter 2: Devices with a specific MAC address
    val addressFilter = ScanFilter.Builder()
        .setDeviceAddress("00:11:22:33:44:55")
        .build()

    // Filter 3: Devices advertising a specific service UUID
    val serviceUuid = ParcelUuid.fromString("0000180d-0000-1000-8000-00805f9b34fb")
    val serviceFilter = ScanFilter.Builder()
        .setServiceUuid(serviceUuid, null) // null mask for an exact match
        .build()

    // Combine filters; the scanner will match if any filter is satisfied.
    val filters = listOf(nameFilter, addressFilter, serviceFilter)

    // Configure scan settings as before
    val scanSettings = ScanSettings.Builder()
        .setScanMode(ScanSettings.SCAN_MODE_LOW_LATENCY)
        .build()

    // Define the scan callback to process results.
    val scanCallback = object : ScanCallback() {
        override fun onScanResult(callbackType: Int, result: ScanResult) {
            Log.i("ScanCallback", "Found Device: ${result.device.name ?: "No Name"} (${result.device.address})")
        }

        override fun onScanFailed(errorCode: Int) {
            Log.e("ScanCallback", "Scan failed with error code: $errorCode")
        }
    }

    // Start scanning using the filters, settings, and callback.
    bleScanner.startScan(filters, scanSettings, scanCallback)
}
```

**Advanced Filtering Considerations:**

- **Combining Conditions (AND):**  
  If you need to require multiple conditions (e.g., device name **and** address), combine them in one filter:
  
  ```kotlin
  val combinedFilter = ScanFilter.Builder()
      .setDeviceName("DeviceA")
      .setDeviceAddress("00:11:22:33:44:55")
      .build()
  ```
  
- **Using Wildcards with UUIDs:**  
  For service UUIDs, you can specify a mask to treat certain bits as wildcards:
  
  ```kotlin
  val myServiceUuid = ParcelUuid.fromString("0000180d-0000-1000-8000-00805f9b34fb")
  val uuidMask = ParcelUuid.fromString("0000ffff-0000-0000-0000-000000000000")
  val wildcardFilter = ScanFilter.Builder()
      .setServiceUuid(myServiceUuid, uuidMask)
      .build()
  ```
  
- **Device Names:**  
  The API does not support wildcards or masks for device names. To implement partial matches or regular expressions, filter the results manually in your callback:

  ```kotlin
  override fun onScanResult(callbackType: Int, result: ScanResult) {
      val name = result.device.name ?: ""
      if (name.contains("Target", ignoreCase = true)) {
          // Process device if it contains "Target" in its name.
      }
  }
  ```

---

## 4. Collecting and Presenting Discovered Devices

Below is an enhanced example that scans with filters, collects unique devices into a set, and then stops the scan after 10 seconds to present the results (e.g., via logs or UI updates).

```kotlin
private fun scanForFilteredDevices() {
    // Define multiple filters (as shown earlier)
    val nameFilter = ScanFilter.Builder()
        .setDeviceName("DeviceA")
        .build()

    val addressFilter = ScanFilter.Builder()
        .setDeviceAddress("00:11:22:33:44:55")
        .build()

    val serviceUuid = ParcelUuid.fromString("0000180d-0000-1000-8000-00805f9b34fb")
    val serviceFilter = ScanFilter.Builder()
        .setServiceUuid(serviceUuid, null)
        .build()

    val filters = listOf(nameFilter, addressFilter, serviceFilter)

    val scanSettings = ScanSettings.Builder()
        .setScanMode(ScanSettings.SCAN_MODE_LOW_LATENCY)
        .build()

    // Set to hold discovered devices uniquely.
    val discoveredDevices = mutableSetOf<BluetoothDevice>()

    // Create a scan callback that adds devices to our set.
    val scanCallback = object : ScanCallback() {
        override fun onScanResult(callbackType: Int, result: ScanResult) {
            discoveredDevices.add(result.device)
            Log.i("ScanCallback", "Found: ${result.device.name ?: "Unknown"} (${result.device.address})")
        }

        override fun onBatchScanResults(results: List<ScanResult>) {
            results.forEach { result -> discoveredDevices.add(result.device) }
        }

        override fun onScanFailed(errorCode: Int) {
            Log.e("ScanCallback", "Scan failed with error code: $errorCode")
        }
    }

    // Start the BLE scan.
    bleScanner.startScan(filters, scanSettings, scanCallback)

    // Stop the scan after 10 seconds and present the results.
    Handler(Looper.getMainLooper()).postDelayed({
        bleScanner.stopScan(scanCallback)
        Log.i("Scan", "Scan complete. Found ${discoveredDevices.size} unique devices:")
        discoveredDevices.forEach { device ->
            Log.i("Scan", "Device: ${device.name ?: "Unknown"} - ${device.address}")
        }
        // Update UI or notify user as needed.
    }, 10000)  // 10 seconds delay
}
```

**Highlights:**

- **Asynchronous Execution:**  
  The scan starts and runs asynchronously. The function returns immediately, while results are processed via the callback.

- **Unique Device Collection:**  
  Using a `MutableSet` ensures that duplicate devices aren’t added.

- **Delayed Stop:**  
  A `Handler` stops the scan after a predetermined period (10 seconds), at which point you can present the results.

---

## 5. Summary

- **Lazy Initialization:** Use `by lazy` to set up Bluetooth components only when needed.
- **BLE Scan Setup:** Create `ScanSettings` and a custom `ScanCallback` to handle results.
- **Filtering:** Use `ScanFilter.Builder()` to filter by device name, address, or service UUID. Combine conditions within a single filter for an AND effect, and supply a list of filters for an OR effect.
- **Result Collection:** Gather discovered devices in a set and stop the scan after a delay.
- **Non-Blocking:** The scanning function is asynchronous and does not block the main thread.

By following this guide, you now have a solid foundation for implementing BLE scanning in Kotlin, complete with flexible filtering and asynchronous handling of scan results.

Feel free to modify the code samples to fit your specific use case or to integrate with your user interface for presenting the discovered devices.

Happy coding!

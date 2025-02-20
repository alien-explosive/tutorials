# Part 1: Connecting to BLE Peripheral with L2CAP CoC in Kotlin

In this tutorial, we will implement how to scan, connect, and establish an L2CAP CoC (Credit-Based Flow Control) channel with a BLE peripheral using Kotlin. This sets up the foundation for file transfer, which will be covered in Part 2.

---

## **1. Scanning for BLE Devices**

To scan for BLE peripherals, first, ensure the necessary permissions are included in `AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
<uses-permission android:name="android.permission.BLUETOOTH_SCAN" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
```

Define the Bluetooth adapter and scanner:

```kotlin
private val bluetoothAdapter: BluetoothAdapter by lazy {
    val bluetoothManager = getSystemService(Context.BLUETOOTH_SERVICE) as BluetoothManager
    bluetoothManager.adapter
}

private val bleScanner by lazy {
    bluetoothAdapter.bluetoothLeScanner
}
```

Start scanning for BLE devices:

```kotlin
private fun startBleScan() {
    val scanSettings = ScanSettings.Builder()
        .setScanMode(ScanSettings.SCAN_MODE_LOW_LATENCY)
        .build()

    val scanCallback = object : ScanCallback() {
        override fun onScanResult(callbackType: Int, result: ScanResult) {
            Log.i("ScanCallBack","Device Name: ${result.device.name ?: "No Name"}")
            Log.i("ScanCallBack","Device Address: ${result.device.address}")
        }
        override fun onScanFailed(errorCode: Int) {
            Log.e("Scan", "Scan failed with error code: $errorCode")
        }
    }

    bleScanner.startScan(null, scanSettings, scanCallback)
}
```

---

## **2. Connecting to the BLE Peripheral**

Once a device is found, initiate a connection:

```kotlin
fun connectToDevice(device: BluetoothDevice) {
    device.connectGatt(context, false, object : BluetoothGattCallback() {
        override fun onConnectionStateChange(gatt: BluetoothGatt, status: Int, newState: Int) {
            if (newState == BluetoothProfile.STATE_CONNECTED) {
                Log.d("GATT", "Connected to device")
                gatt.discoverServices()
            }
        }
    })
}
```

---

## **3. Establishing an L2CAP CoC Connection**

Once connected, establish an L2CAP channel:

```kotlin
private lateinit var l2capSocket: BluetoothSocket

fun connectToL2CAP(device: BluetoothDevice, psm: Int) {
    try {
        l2capSocket = device.createInsecureL2capChannel(psm)
        l2capSocket.connect()
        Log.d("L2CAP", "L2CAP channel connected!")
    } catch (e: IOException) {
        Log.e("L2CAP", "Failed to connect: ${e.message}")
    }
}
```

---

# Part 2: Implementing File Transfer over L2CAP CoC in Kotlin

This part explains how to receive files from a BLE peripheral over L2CAP CoC using **credit-based flow control**. The peripheral provides the file size and metadata via GATT characteristics.

---

## **1. Receiving File Metadata from GATT**

Retrieve file size and metadata before starting the file transfer:

```kotlin
private fun readFileMetadata(device: BluetoothDevice) {
    val gatt = device.connectGatt(context, false, object : BluetoothGattCallback() {
        override fun onServicesDiscovered(gatt: BluetoothGatt, status: Int) {
            if (status == BluetoothGatt.GATT_SUCCESS) {
                val service = gatt.getService(UUID.fromString(METADATA_SERVICE_UUID))
                val characteristic = service?.getCharacteristic(UUID.fromString(FILE_SIZE_UUID))
                characteristic?.let {
                    gatt.readCharacteristic(it)
                }
            }
        }

        override fun onCharacteristicRead(gatt: BluetoothGatt, characteristic: BluetoothGattCharacteristic, status: Int) {
            if (status == BluetoothGatt.GATT_SUCCESS) {
                val fileSize = ByteBuffer.wrap(characteristic.value).int
                Log.d("GATT", "File size: $fileSize bytes")
            }
        }
    })
}
```

---

## **2. Receiving File Data Efficiently**

Once the L2CAP connection is established, continuously read incoming data:

```kotlin
fun receiveFile(filePath: String) {
    Thread {
        try {
            val inputStream = l2capSocket.inputStream
            val outputStream = FileOutputStream(File(filePath))
            val buffer = ByteArray(441) // SDU size
            var bytesRead: Int

            while (true) {
                bytesRead = inputStream.read(buffer)
                if (bytesRead == -1) break
                outputStream.write(buffer, 0, bytesRead)
            }
            outputStream.close()
            Log.d("L2CAP", "File received and saved at: $filePath")
        } catch (e: IOException) {
            Log.e("L2CAP", "Error during file transfer: ${e.message}")
        }
    }.start()
}
```

---

## **3. Saving the File to Android’s Local File System**

### **Internal Storage (Private to App)**
```kotlin
val file = File(context.filesDir, "downloaded_file.bin")
FileOutputStream(file).use { output ->
    output.write(receivedData)
}
```

### **External Storage (Accessible by Other Apps)**
For Android 10+, use `MediaStore` instead of `WRITE_EXTERNAL_STORAGE` permission.

```kotlin
val file = File(Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_DOWNLOADS), "downloaded_file.bin")
FileOutputStream(file).use { output ->
    output.write(receivedData)
}
Log.d("Storage", "File saved at: ${file.absolutePath}")
```

---

## **Final Thoughts**
This tutorial covered:
✅ Scanning and connecting to a BLE device.
✅ Establishing an L2CAP CoC channel.
✅ Reading file metadata from GATT.
✅ Receiving large files efficiently with credit-based flow control.
✅ Saving files locally on an Android device.

Let me know if you need further refinements! 🚀


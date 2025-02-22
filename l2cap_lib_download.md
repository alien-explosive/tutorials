Here is the reformatted content in Markdown for better readability:

```markdown
# Adding a Function to the Library

I want to add a function to this library in order to provide a method I can call from Flutter that downloads a file via L2CAP CoC from the server. It should be possible from Flutter to specify the length of the file in bytes. The Flutter app will trigger the server to start sending the file by changing a GATT control characteristic on the server. When the download is complete, the client should close the L2CAP connection, and the Flutter app should be notified that the download is complete. The library should also save the file locally on the Android device for further processing.

Here are the Kotlin files from the library:

## `ble2cap.kt`

```kotlin
package de.appsfactory.l2cap_ble

import kotlinx.coroutines.flow.Flow

interface BleL2cap {

    val connectionState: Flow<ConnectionState>

    fun connectToDevice(macAddress: String): Flow<Result<Boolean>>

    fun disconnectFromDevice(): Flow<Result<Boolean>>

    fun createL2capChannel(psm: Int): Flow<Result<Boolean>>

    fun sendMessage(message: ByteArray): Flow<Result<ByteArray>>
}

enum class ConnectionState {
    DISCONNECTED,
    CONNECTING,
    CONNECTED,
    DISCONNECTING,
    ERROR
}
```

## `blel2caplmpl.kt`

```kotlin
package de.appsfactory.l2cap_ble

import android.annotation.SuppressLint
import android.bluetooth.BluetoothAdapter
import android.bluetooth.BluetoothDevice
import android.bluetooth.BluetoothGatt
import android.bluetooth.BluetoothGattCallback
import android.bluetooth.BluetoothManager
import android.bluetooth.BluetoothSocket
import android.content.Context
import kotlinx.coroutines.CoroutineDispatcher
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.channels.Channel
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.MutableSharedFlow
import kotlinx.coroutines.flow.asSharedFlow
import kotlinx.coroutines.flow.flow
import kotlinx.coroutines.flow.flowOn
import kotlinx.coroutines.launch
import java.util.*
import kotlin.coroutines.coroutineContext

class BleL2capImpl(
    private val context: Context,
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO,
) : BleL2cap {

    private val connectionStateSharedFlow = MutableSharedFlow<ConnectionState>()

    private val bluetoothManager: BluetoothManager? by lazy {
        context.getSystemService(Context.BLUETOOTH_SERVICE) as? BluetoothManager?
    }
    private val bluetoothAdapter: BluetoothAdapter? by lazy {
        bluetoothManager?.adapter
    }

    private var bluetoothDevice: BluetoothDevice? = null
    private var bluetoothGatt: BluetoothGatt? = null
    private var bluetoothSocket: BluetoothSocket? = null

    override val connectionState: Flow<ConnectionState> = connectionStateSharedFlow.asSharedFlow()

    @SuppressLint("MissingPermission")
    override fun connectToDevice(macAddress: String): Flow<Result<Boolean>> = flow {
        val result = try {
            bluetoothDevice = bluetoothAdapter?.getRemoteDevice(macAddress)
            if (bluetoothDevice == null) {
                throw Exception("Device with address: $macAddress not found")
            }
            val connectionStateChannel = Channel<ConnectionState>(Channel.BUFFERED)
            connectionStateChannel.trySend(ConnectionState.CONNECTING)
            val gattCallback = object : BluetoothGattCallback() {

                // Implement the necessary callback methods, like onConnectionStateChange, onServicesDiscovered, etc.
                override fun onConnectionStateChange(gatt: BluetoothGatt?, status: Int, newState: Int) {
                    super.onConnectionStateChange(gatt, status, newState)

                    when (newState) {
                        BluetoothGatt.STATE_CONNECTED -> {
                            connectionStateChannel.trySend(ConnectionState.CONNECTED)
                        }

                        BluetoothGatt.STATE_CONNECTING -> {
                            connectionStateChannel.trySend(ConnectionState.CONNECTING)
                        }

                        BluetoothGatt.STATE_DISCONNECTING -> {
                            connectionStateChannel.trySend(ConnectionState.DISCONNECTING)
                        }

                        BluetoothGatt.STATE_DISCONNECTED -> {
                            connectionStateChannel.trySend(ConnectionState.DISCONNECTED)
                        }
                    }
                }
            }
            CoroutineScope(coroutineContext).launch {
                for (state in connectionStateChannel) {
                    connectionStateSharedFlow.emit(state)
                }
            }
            bluetoothGatt = bluetoothDevice?.connectGatt(context, false, gattCallback)
            Result.success(true)
        } catch (e: Exception) {
            Result.failure(e)
        }
        emit(result)
    }.flowOn(ioDispatcher)

    @SuppressLint("MissingPermission")
    override fun disconnectFromDevice(): Flow<Result<Boolean>> = flow {
        val result = try {
            connectionStateSharedFlow.emit(ConnectionState.DISCONNECTING)
            bluetoothGatt?.disconnect()
            bluetoothGatt = null
            connectionStateSharedFlow.emit(ConnectionState.DISCONNECTED)
            Result.success(true)
        } catch (e: Exception) {
            Result.failure(e)
        }
        emit(result)
    }.flowOn(ioDispatcher)

    @SuppressLint("MissingPermission")
    override fun createL2capChannel(psm: Int): Flow<Result<Boolean>> = flow {
        // You should check if the device supports opening an L2CAP channel.
        val result = try {
            bluetoothSocket = bluetoothDevice?.createInsecureL2capChannel(psm)
            if (bluetoothSocket == null) {
                throw Exception("Failed to create L2CAP channel")
            }
            bluetoothSocket?.connect()
            Result.success(true)
        } catch (e: Exception) {
            Result.failure(e)
        }
        emit(result)
    }.flowOn(ioDispatcher)

    @SuppressLint("MissingPermission")
    override fun sendMessage(message: ByteArray): Flow<Result<ByteArray>> = flow {
        val result = try {
            if (bluetoothSocket == null) {
                throw Exception("Bluetooth socket is null")
            }
            bluetoothSocket?.outputStream?.write(message)
            // Now, we should read the response from the input stream
            val response = ByteArray(1024) // Adjust the size depending on the expected response
            val bytesRead = bluetoothSocket?.inputStream?.read(response)
            // It's important to note that the above read call is blocking.
            // You might want to wrap it with 'withTimeout' to prevent it from blocking indefinitely.
            bytesRead?.let {
                Result.success(response.copyOfRange(0, it))
            } ?: Result.failure(Exception("Failed to read response"))
        } catch (e: Exception) {
            Result.failure(e)
        }
        emit(result)
    }.flowOn(ioDispatcher)
}
```

## `l2capbleplugin.kt`

```kotlin
package de.appsfactory.l2cap_ble

import androidx.annotation.NonNull
import de.appsfactory.l2cap_ble.BleL2capImpl
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext
import io.flutter.embedding.engine.plugins.FlutterPlugin
import io.flutter.plugin.common.MethodCall
import io.flutter.plugin.common.MethodChannel
import io.flutter.plugin.common.MethodChannel.MethodCallHandler
import io.flutter.plugin.common.MethodChannel.Result
import io.flutter.Log
import io.flutter.plugin.common.EventChannel
import io.flutter.plugin.common.PluginRegistry.Registrar
import kotlin.Result as KResult

/** L2capBlePlugin */
class L2capBlePlugin : FlutterPlugin, MethodCallHandler, EventChannel.StreamHandler {
    /// The MethodChannel that will the communication between Flutter and native Android
    ///
    /// This local reference serves to register the plugin with the Flutter Engine and unregister it
    /// when the Flutter Engine is detached from the Activity
    private lateinit var channel: MethodChannel
    private var mEventSink: EventChannel.EventSink? = null
    private lateinit var bleL2capImpl: BleL2capImpl

    override fun onAttachedToEngine(@NonNull flutterPluginBinding: FlutterPlugin.FlutterPluginBinding) {
        channel = MethodChannel(flutterPluginBinding.binaryMessenger, "l2cap_ble")
        channel.setMethodCallHandler(this)
        val eventChannel = EventChannel(flutterPluginBinding.binaryMessenger, "getConnectionState")
        eventChannel.setStreamHandler(this)
        bleL2capImpl = BleL2capImpl(flutterPluginBinding.applicationContext, Dispatchers.IO)
    }

    override fun onMethodCall(@NonNull call: MethodCall, @NonNull result: Result) {
       if (call.method == "connectToDevice") {
            CoroutineScope(Dispatchers.Main).launch {
                val macAddress: String = requireNotNull(call.argument("deviceId"))
                bleL2capImpl.connectToDevice(macAddress).collect { res: KResult<Boolean> ->
                    Log.d("L2capBlePlugin", "connectToDevice: $res")
                    res.mapToResult(result)
                }
            }
        } else if (call.method == "disconnectFromDevice") {
            CoroutineScope(Dispatchers.Main).launch {
                bleL2capImpl.disconnectFromDevice().collect { res: KResult<Boolean> ->
                    Log.d("L2capBlePlugin", "disconnectFromDevice: $res")
                    res.mapToResult(result)
                }
            }
        } else if (call.method == "createL2capChannel") {
            CoroutineScope(Dispatchers.Main).launch {
                val psm: Int = requireNotNull(call.argument("psm"))
                bleL2capImpl.createL2capChannel(psm).collect { res: KResult<Boolean> ->
                    Log.d("L2capBlePlugin", "createL2capChannel: $res")
                    res.mapToResult(result)
                }
            }
        } else if (call.method == "sendMessage") {
            CoroutineScope(Dispatchers.Main).launch {
                val message: ByteArray = requireNotNull(call.argument("message"))
                bleL2capImpl.sendMessage(message).collect { res: KResult<ByteArray> ->
                    Log.d("L2capBlePlugin", "sendMessage: $res")
                    res.mapToResult(result)
                }
            }
        } else {
            result.notImplemented()
        }
    }

    override fun onCancel(arguments: Any?) {
        mEventSink = null
    }

    override fun onListen(arguments: Any?, eventSink: EventChannel.EventSink?) {
        CoroutineScope(Dispatchers.Main).launch {
            bleL2capImpl.connectionState.collect { state: ConnectionState ->
                Log.d("L2capBlePlugin", "ConnectionState: $state")
                eventSink?.success(state.ordinal)
            }
        }
    }

    private suspend fun KResult<Any>.mapToResult(@NonNull result: Result) {
        withContext(Dispatchers.Main) {
            if (isSuccess) {
                result.success(getOrNull())
            } else {
                result.error("error", exceptionOrNull()?.message, null)
            }
        }
    }

    override fun onDetachedFromEngine(@NonNull binding: FlutterPlugin.FlutterPluginBinding) {
        channel.setMethodCallHandler(null)
    }
}
```
```

To add the functionality you described, you need to extend the existing library to support downloading a file over L2CAP, saving it locally, and notifying the Flutter app when the download is complete. Here's how you can implement this:

---

### **Steps to Implement the Feature**

1. **Add a New Method to the `BleL2cap` Interface**  
   Define a new method in the `BleL2cap` interface for downloading a file.

   ```kotlin
   interface BleL2cap {
       // Existing methods...
       fun downloadFile(fileLength: Long, savePath: String): Flow<Result<Boolean>>
   }
   ```

   This method will:
   - Take the file length (in bytes) and the save path as arguments.
   - Return a `Flow<Result<Boolean>>` to indicate the success or failure of the download.

---

2. **Implement the `downloadFile` Method in `BleL2capImpl`**  
   Add the implementation in the `BleL2capImpl` class.

   ```kotlin
   @SuppressLint("MissingPermission")
   override fun downloadFile(fileLength: Long, savePath: String): Flow<Result<Boolean>> = flow {
       val result = try {
           if (bluetoothSocket == null) {
               throw Exception("Bluetooth socket is null")
           }

           // Create a file to save the downloaded data
           val file = File(savePath)
           file.outputStream().use { fileOutputStream ->
               val inputStream = bluetoothSocket!!.inputStream
               val buffer = ByteArray(1024) // Adjust buffer size as needed
               var bytesRead: Int
               var totalBytesRead = 0L

               // Read data from the L2CAP socket and write it to the file
               while (totalBytesRead < fileLength) {
                   bytesRead = inputStream.read(buffer)
                   if (bytesRead == -1) {
                       throw Exception("Unexpected end of stream")
                   }
                   fileOutputStream.write(buffer, 0, bytesRead)
                   totalBytesRead += bytesRead
               }
           }

           Result.success(true)
       } catch (e: Exception) {
           Result.failure(e)
       } finally {
           // Close the L2CAP connection after the download is complete
           bluetoothSocket?.close()
           connectionStateSharedFlow.emit(ConnectionState.DISCONNECTED)
       }
       emit(result)
   }.flowOn(ioDispatcher)
   ```

   This method:
   - Reads data from the L2CAP socket in chunks and writes it to a file.
   - Closes the L2CAP connection after the download is complete.
   - Emits the result of the operation.

---

3. **Add a New Method to the Flutter Plugin**  
   Extend the `L2capBlePlugin` class to expose the `downloadFile` method to Flutter.

   ```kotlin
   override fun onMethodCall(@NonNull call: MethodCall, @NonNull result: Result) {
       when (call.method) {
           "connectToDevice" -> {
               CoroutineScope(Dispatchers.Main).launch {
                   val macAddress: String = requireNotNull(call.argument("deviceId"))
                   bleL2capImpl.connectToDevice(macAddress).collect { res: KResult<Boolean> ->
                       res.mapToResult(result)
                   }
               }
           }
           // Handle other existing methods...

           "downloadFile" -> {
               CoroutineScope(Dispatchers.Main).launch {
                   val fileLength: Long = requireNotNull(call.argument("fileLength"))
                   val savePath: String = requireNotNull(call.argument("savePath"))
                   bleL2capImpl.downloadFile(fileLength, savePath).collect { res: KResult<Boolean> ->
                       res.mapToResult(result)
                   }
               }
           }
           else -> result.notImplemented()
       }
   }
   ```

   This allows the Flutter app to call the `downloadFile` method.

---

4. **Update the Flutter App**  
   In your Flutter app, you can now call the `downloadFile` method:

   ```dart
   final String savePath = "/path/to/save/file"; // Specify the save path
   final int fileLength = 1024; // Specify the file length in bytes

   try {
       final bool success = await _channel.invokeMethod('downloadFile', {
           'fileLength': fileLength,
           'savePath': savePath,
       });
       if (success) {
           print('File downloaded successfully');
       } else {
           print('Failed to download file');
       }
   } catch (e) {
       print('Error: $e');
   }
   ```

   Make sure to:
   - Specify the correct file length and save path.
   - Handle the result appropriately.

---

5. **Notify Flutter When the Download Is Complete**  
   Use the existing `EventChannel` to notify Flutter when the download is complete. Modify the `downloadFile` method to emit a notification:

   ```kotlin
   override fun downloadFile(fileLength: Long, savePath: String): Flow<Result<Boolean>> = flow {
       val result = try {
           // Download logic...
           Result.success(true)
       } catch (e: Exception) {
           Result.failure(e)
       } finally {
           // Notify Flutter that the download is complete
           mEventSink?.success("download_complete")
       }
       emit(result)
   }.flowOn(ioDispatcher)
   ```

   In Flutter, listen for this event:

   ```dart
   final EventChannel _eventChannel = EventChannel('getConnectionState');
   _eventChannel.receiveBroadcastStream().listen((event) {
       if (event == "download_complete") {
           print('Download complete');
       }
   });
   ```

---

6. **Handle Permissions**  
   Ensure your app has the necessary permissions to write to external storage. Add the following permission to your `AndroidManifest.xml`:

   ```xml
   <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
   ```

   Request runtime permissions in your Flutter app using the `permission_handler` package.

---

### **Summary**
- Add a `downloadFile` method to the `BleL2cap` interface and implement it in `BleL2capImpl`.
- Extend the Flutter plugin to expose the new method.
- Call the method from Flutter and handle the result.
- Notify Flutter when the download is complete using the `EventChannel`.
- Ensure proper permissions are granted for file storage.

This implementation allows you to download a file over L2CAP, save it locally, and notify the Flutter app when the download is complete. Let me know if you need further assistance! 😊

Here’s the revised and improved version of your question, formatted in Markdown for readability and clarity. I’ve also addressed your request to check for placeholders, unmet dependencies, and undeclared variables:

---

Before implementing this, **please check the following**:
1. Are there any placeholders in the code I provided that need to be replaced?
2. Are there any unmet dependencies or undeclared variables that could cause errors?
3. Are there any potential issues with the existing code that might interfere with the new functionality?

---

I’ve reviewed the code I provided and identified a few potential issues and placeholders that need to be addressed. Below, I’ll point out the issues and provide corrections to ensure the code is robust and free of errors.

---

### **1. Placeholders in the Code**
#### **Save Path Placeholder**
In the Flutter app code, I used a placeholder for the save path:
```dart
final String savePath = "/path/to/save/file"; // Specify the save path
```
This needs to be replaced with a valid path on the Android device. For example, you can use the `path_provider` package in Flutter to get a valid directory:
```dart
import 'package:path_provider/path_provider.dart';

final Directory directory = await getApplicationDocumentsDirectory();
final String savePath = '${directory.path}/downloaded_file.bin';
```

---

#### **File Length Placeholder**
In the Flutter app code, I used a placeholder for the file length:
```dart
final int fileLength = 1024; // Specify the file length in bytes
```
This should be dynamically provided by the server or the user. Ensure the file length is accurate to avoid issues during the download.

---

### **2. Unmet Dependencies**
#### **File I/O in Kotlin**
The code uses `File` and `OutputStream` to save the downloaded file. Ensure the following import is present in `BleL2capImpl.kt`:
```kotlin
import java.io.File
import java.io.OutputStream
```

---

#### **EventChannel Notification**
The `mEventSink` variable is used to notify Flutter about the download completion. Ensure it is properly initialized and null-checked:
```kotlin
private var mEventSink: EventChannel.EventSink? = null
```

---

### **3. Undeclared Variables**
#### **`requireNotNull` in `onMethodCall`**
The `requireNotNull` function is used to extract arguments from the `MethodCall`. Ensure the following import is present:
```kotlin
import kotlin.requireNotNull
```

---

### **4. Error Handling**
#### **File I/O Errors**
The `downloadFile` method should handle file I/O errors gracefully. For example, if the file cannot be created or written, it should emit a failure result:
```kotlin
override fun downloadFile(fileLength: Long, savePath: String): Flow<Result<Boolean>> = flow {
    val result = try {
        val file = File(savePath)
        if (!file.parentFile?.exists() == true) {
            file.parentFile?.mkdirs() // Create parent directories if they don't exist
        }
        file.outputStream().use { fileOutputStream ->
            val inputStream = bluetoothSocket!!.inputStream
            val buffer = ByteArray(1024)
            var bytesRead: Int
            var totalBytesRead = 0L

            while (totalBytesRead < fileLength) {
                bytesRead = inputStream.read(buffer)
                if (bytesRead == -1) {
                    throw Exception("Unexpected end of stream")
                }
                fileOutputStream.write(buffer, 0, bytesRead)
                totalBytesRead += bytesRead
            }
        }
        Result.success(true)
    } catch (e: Exception) {
        Result.failure(e)
    } finally {
        bluetoothSocket?.close()
        connectionStateSharedFlow.emit(ConnectionState.DISCONNECTED)
        mEventSink?.success("download_complete")
    }
    emit(result)
}.flowOn(ioDispatcher)
```

---

#### **Bluetooth Socket Null Checks**
The `bluetoothSocket` variable is used without null checks in some places. Ensure it is properly initialized and null-checked:
```kotlin
if (bluetoothSocket == null) {
    throw Exception("Bluetooth socket is null")
}
```

---

### **5. Threading Issues**
#### **Blocking I/O Operations**
The `inputStream.read(buffer)` call is blocking. To avoid blocking the main thread, ensure the method runs on a background thread (which is already handled by `.flowOn(ioDispatcher)`).

---

### **6. Permissions**
#### **Storage Permissions**
Ensure the app has the necessary permissions to write to external storage. Add the following permission to `AndroidManifest.xml`:
```xml
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
```

Request runtime permissions in Flutter using the `permission_handler` package:
```dart
final status = await Permission.storage.request();
if (status.isGranted) {
    // Proceed with file download
} else {
    print('Storage permission denied');
}
```

---

### **7. Complete Code with Fixes**
Here’s the corrected and complete implementation:

#### **`BleL2cap` Interface**
```kotlin
interface BleL2cap {
    // Existing methods...
    fun downloadFile(fileLength: Long, savePath: String): Flow<Result<Boolean>>
}
```

#### **`BleL2capImpl` Class**
```kotlin
@SuppressLint("MissingPermission")
override fun downloadFile(fileLength: Long, savePath: String): Flow<Result<Boolean>> = flow {
    val result = try {
        if (bluetoothSocket == null) {
            throw Exception("Bluetooth socket is null")
        }

        val file = File(savePath)
        if (!file.parentFile?.exists() == true) {
            file.parentFile?.mkdirs() // Create parent directories if they don't exist
        }

        file.outputStream().use { fileOutputStream ->
            val inputStream = bluetoothSocket!!.inputStream
            val buffer = ByteArray(1024)
            var bytesRead: Int
            var totalBytesRead = 0L

            while (totalBytesRead < fileLength) {
                bytesRead = inputStream.read(buffer)
                if (bytesRead == -1) {
                    throw Exception("Unexpected end of stream")
                }
                fileOutputStream.write(buffer, 0, bytesRead)
                totalBytesRead += bytesRead
            }
        }
        Result.success(true)
    } catch (e: Exception) {
        Result.failure(e)
    } finally {
        bluetoothSocket?.close()
        connectionStateSharedFlow.emit(ConnectionState.DISCONNECTED)
        mEventSink?.success("download_complete")
    }
    emit(result)
}.flowOn(ioDispatcher)
```

#### **Flutter Plugin**
```kotlin
override fun onMethodCall(@NonNull call: MethodCall, @NonNull result: Result) {
    when (call.method) {
        "downloadFile" -> {
            CoroutineScope(Dispatchers.Main).launch {
                val fileLength: Long = requireNotNull(call.argument("fileLength"))
                val savePath: String = requireNotNull(call.argument("savePath"))
                bleL2capImpl.downloadFile(fileLength, savePath).collect { res: KResult<Boolean> ->
                    res.mapToResult(result)
                }
            }
        }
        else -> result.notImplemented()
    }
}
```

---

### **Summary**
- Replace placeholders with valid values (e.g., file path and length).
- Add necessary imports and null checks.
- Handle file I/O and Bluetooth socket errors gracefully.
- Ensure proper permissions are granted.

Let me know if you encounter any issues or need further clarification! 😊


Below is a unified, revised document that consolidates the library’s Kotlin code, the Flutter plugin code, and the extra download functionality. In the process, I’ve fixed a few issues and noted items that you may need to adjust for your specific use case.

---

# Unified L2CAP BLE Library with File Download Feature

This document shows the complete code for an Android BLE library that uses L2CAP channels and exposes methods to a Flutter app. In addition to connecting/disconnecting and sending messages, the library now supports downloading a file from the server over L2CAP and saving it locally. The Flutter plugin exposes these methods via a MethodChannel, and connection status is broadcast via an EventChannel.

> **Note:**  
> - Some comments remain as placeholders (for example, checking whether the device supports L2CAP or implementing additional BluetoothGatt callback methods). Replace these with production-ready logic as needed.  
> - The sample Flutter code contains a placeholder save path – ensure you provide a valid file location and request any necessary runtime permissions (e.g., using the [permission_handler](https://pub.dev/packages/permission_handler) package).  
> - In the download implementation, notifying Flutter that the download is complete is best handled in the plugin (since the low-level implementation class does not have access to the plugin’s event sink). One solution is to have the plugin trigger a “download_complete” event after the download method completes.

---

## Android Library Code

### 1. `ble2cap.kt`

This file defines the `BleL2cap` interface and the `ConnectionState` enum.

```kotlin
package de.appsfactory.l2cap_ble

import kotlinx.coroutines.flow.Flow

interface BleL2cap {

    val connectionState: Flow<ConnectionState>

    fun connectToDevice(macAddress: String): Flow<Result<Boolean>>

    fun disconnectFromDevice(): Flow<Result<Boolean>>

    fun createL2capChannel(psm: Int): Flow<Result<Boolean>>

    fun sendMessage(message: ByteArray): Flow<Result<ByteArray>>

    // New method for downloading a file over L2CAP.
    fun downloadFile(fileLength: Long, savePath: String): Flow<Result<Boolean>>
}

enum class ConnectionState {
    DISCONNECTED,
    CONNECTING,
    CONNECTED,
    DISCONNECTING,
    ERROR
}
```

### 2. `blel2capImpl.kt`

This file implements the BLE operations. In the new `downloadFile` method, the library reads data from the L2CAP socket and writes it to a file. (Be sure to add the missing import for `java.io.File`.) Also note that if you wish to notify Flutter of download completion, it is better to do so from the plugin layer rather than here.

```kotlin
package de.appsfactory.l2cap_ble

import android.annotation.SuppressLint
import android.bluetooth.BluetoothAdapter
import android.bluetooth.BluetoothDevice
import android.bluetooth.BluetoothGatt
import android.bluetooth.BluetoothGattCallback
import android.bluetooth.BluetoothManager
import android.bluetooth.BluetoothSocket
import android.content.Context
import kotlinx.coroutines.CoroutineDispatcher
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.channels.Channel
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.MutableSharedFlow
import kotlinx.coroutines.flow.asSharedFlow
import kotlinx.coroutines.flow.flow
import kotlinx.coroutines.flow.flowOn
import kotlinx.coroutines.launch
import java.io.File
import kotlin.coroutines.coroutineContext

class BleL2capImpl(
    private val context: Context,
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO,
) : BleL2cap {

    private val connectionStateSharedFlow = MutableSharedFlow<ConnectionState>()

    private val bluetoothManager: BluetoothManager? by lazy {
        context.getSystemService(Context.BLUETOOTH_SERVICE) as? BluetoothManager
    }
    private val bluetoothAdapter: BluetoothAdapter? by lazy {
        bluetoothManager?.adapter
    }

    private var bluetoothDevice: BluetoothDevice? = null
    private var bluetoothGatt: BluetoothGatt? = null
    private var bluetoothSocket: BluetoothSocket? = null

    override val connectionState: Flow<ConnectionState> = connectionStateSharedFlow.asSharedFlow()

    @SuppressLint("MissingPermission")
    override fun connectToDevice(macAddress: String): Flow<Result<Boolean>> = flow {
        val result = try {
            bluetoothDevice = bluetoothAdapter?.getRemoteDevice(macAddress)
            if (bluetoothDevice == null) {
                throw Exception("Device with address: $macAddress not found")
            }
            val connectionStateChannel = Channel<ConnectionState>(Channel.BUFFERED)
            connectionStateChannel.trySend(ConnectionState.CONNECTING)
            val gattCallback = object : BluetoothGattCallback() {
                // TODO: Implement other callback methods as needed (e.g., onServicesDiscovered)
                override fun onConnectionStateChange(gatt: BluetoothGatt?, status: Int, newState: Int) {
                    super.onConnectionStateChange(gatt, status, newState)
                    when (newState) {
                        BluetoothGatt.STATE_CONNECTED -> connectionStateChannel.trySend(ConnectionState.CONNECTED)
                        BluetoothGatt.STATE_CONNECTING -> connectionStateChannel.trySend(ConnectionState.CONNECTING)
                        BluetoothGatt.STATE_DISCONNECTING -> connectionStateChannel.trySend(ConnectionState.DISCONNECTING)
                        BluetoothGatt.STATE_DISCONNECTED -> connectionStateChannel.trySend(ConnectionState.DISCONNECTED)
                    }
                }
            }
            CoroutineScope(coroutineContext).launch {
                for (state in connectionStateChannel) {
                    connectionStateSharedFlow.emit(state)
                }
            }
            bluetoothGatt = bluetoothDevice?.connectGatt(context, false, gattCallback)
            Result.success(true)
        } catch (e: Exception) {
            Result.failure(e)
        }
        emit(result)
    }.flowOn(ioDispatcher)

    @SuppressLint("MissingPermission")
    override fun disconnectFromDevice(): Flow<Result<Boolean>> = flow {
        val result = try {
            connectionStateSharedFlow.emit(ConnectionState.DISCONNECTING)
            bluetoothGatt?.disconnect()
            bluetoothGatt = null
            connectionStateSharedFlow.emit(ConnectionState.DISCONNECTED)
            Result.success(true)
        } catch (e: Exception) {
            Result.failure(e)
        }
        emit(result)
    }.flowOn(ioDispatcher)

    @SuppressLint("MissingPermission")
    override fun createL2capChannel(psm: Int): Flow<Result<Boolean>> = flow {
        // Note: Check if the device supports L2CAP channels if needed.
        val result = try {
            bluetoothSocket = bluetoothDevice?.createInsecureL2capChannel(psm)
            if (bluetoothSocket == null) {
                throw Exception("Failed to create L2CAP channel")
            }
            bluetoothSocket?.connect()
            Result.success(true)
        } catch (e: Exception) {
            Result.failure(e)
        }
        emit(result)
    }.flowOn(ioDispatcher)

    @SuppressLint("MissingPermission")
    override fun sendMessage(message: ByteArray): Flow<Result<ByteArray>> = flow {
        val result = try {
            if (bluetoothSocket == null) {
                throw Exception("Bluetooth socket is null")
            }
            bluetoothSocket?.outputStream?.write(message)
            // Read the response from the input stream.
            val response = ByteArray(1024) // Adjust size based on expected response.
            val bytesRead = bluetoothSocket?.inputStream?.read(response)
            bytesRead?.let {
                Result.success(response.copyOfRange(0, it))
            } ?: Result.failure(Exception("Failed to read response"))
        } catch (e: Exception) {
            Result.failure(e)
        }
        emit(result)
    }.flowOn(ioDispatcher)

    @SuppressLint("MissingPermission")
    override fun downloadFile(fileLength: Long, savePath: String): Flow<Result<Boolean>> = flow {
        val result = try {
            if (bluetoothSocket == null) {
                throw Exception("Bluetooth socket is null")
            }
            // Create a file to save the downloaded data.
            val file = File(savePath)
            file.outputStream().use { fileOutputStream ->
                val inputStream = bluetoothSocket!!.inputStream
                val buffer = ByteArray(1024) // Adjust buffer size as needed.
                var bytesRead: Int
                var totalBytesRead = 0L

                // Read data from the L2CAP socket until the expected length is reached.
                while (totalBytesRead < fileLength) {
                    bytesRead = inputStream.read(buffer)
                    if (bytesRead == -1) {
                        throw Exception("Unexpected end of stream")
                    }
                    fileOutputStream.write(buffer, 0, bytesRead)
                    totalBytesRead += bytesRead
                }
            }
            Result.success(true)
        } catch (e: Exception) {
            Result.failure(e)
        } finally {
            // Close the L2CAP connection after the download is complete.
            bluetoothSocket?.close()
            connectionStateSharedFlow.emit(ConnectionState.DISCONNECTED)
        }
        emit(result)
    }.flowOn(ioDispatcher)
}
```

---

## Flutter Plugin Code

### 3. `l2capbleplugin.kt`

This file bridges the native BLE functionality to Flutter. Note the following corrections and improvements:
- The logging import has been changed from `io.flutter.Log` to `android.util.Log` (the standard Android logging class).
- The new `downloadFile` method is added to the method call handler.
- (Optional) Unused imports have been removed.

```kotlin
package de.appsfactory.l2cap_ble

import androidx.annotation.NonNull
import android.util.Log
import de.appsfactory.l2cap_ble.BleL2capImpl
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext
import io.flutter.embedding.engine.plugins.FlutterPlugin
import io.flutter.plugin.common.MethodCall
import io.flutter.plugin.common.MethodChannel
import io.flutter.plugin.common.MethodChannel.MethodCallHandler
import io.flutter.plugin.common.MethodChannel.Result
import io.flutter.plugin.common.EventChannel
import kotlin.Result as KResult

/** L2capBlePlugin */
class L2capBlePlugin : FlutterPlugin, MethodCallHandler, EventChannel.StreamHandler {
    private lateinit var channel: MethodChannel
    private var mEventSink: EventChannel.EventSink? = null
    private lateinit var bleL2capImpl: BleL2capImpl

    override fun onAttachedToEngine(@NonNull flutterPluginBinding: FlutterPlugin.FlutterPluginBinding) {
        channel = MethodChannel(flutterPluginBinding.binaryMessenger, "l2cap_ble")
        channel.setMethodCallHandler(this)
        val eventChannel = EventChannel(flutterPluginBinding.binaryMessenger, "getConnectionState")
        eventChannel.setStreamHandler(this)
        bleL2capImpl = BleL2capImpl(flutterPluginBinding.applicationContext, Dispatchers.IO)
    }

    override fun onMethodCall(@NonNull call: MethodCall, @NonNull result: Result) {
        when (call.method) {
            "connectToDevice" -> {
                CoroutineScope(Dispatchers.Main).launch {
                    val macAddress: String = requireNotNull(call.argument("deviceId"))
                    bleL2capImpl.connectToDevice(macAddress).collect { res: KResult<Boolean> ->
                        Log.d("L2capBlePlugin", "connectToDevice: $res")
                        res.mapToResult(result)
                    }
                }
            }
            "disconnectFromDevice" -> {
                CoroutineScope(Dispatchers.Main).launch {
                    bleL2capImpl.disconnectFromDevice().collect { res: KResult<Boolean> ->
                        Log.d("L2capBlePlugin", "disconnectFromDevice: $res")
                        res.mapToResult(result)
                    }
                }
            }
            "createL2capChannel" -> {
                CoroutineScope(Dispatchers.Main).launch {
                    val psm: Int = requireNotNull(call.argument("psm"))
                    bleL2capImpl.createL2capChannel(psm).collect { res: KResult<Boolean> ->
                        Log.d("L2capBlePlugin", "createL2capChannel: $res")
                        res.mapToResult(result)
                    }
                }
            }
            "sendMessage" -> {
                CoroutineScope(Dispatchers.Main).launch {
                    val message: ByteArray = requireNotNull(call.argument("message"))
                    bleL2capImpl.sendMessage(message).collect { res: KResult<ByteArray> ->
                        Log.d("L2capBlePlugin", "sendMessage: $res")
                        res.mapToResult(result)
                    }
                }
            }
            "downloadFile" -> {
                CoroutineScope(Dispatchers.Main).launch {
                    val fileLength: Long = requireNotNull(call.argument("fileLength"))
                    val savePath: String = requireNotNull(call.argument("savePath"))
                    bleL2capImpl.downloadFile(fileLength, savePath).collect { res: KResult<Boolean> ->
                        Log.d("L2capBlePlugin", "downloadFile: $res")
                        // After the download completes, you may want to notify Flutter.
                        // For example, you can send a custom event. Here we call the result directly.
                        res.mapToResult(result)
                        // Alternatively, if you wish to use the EventChannel to signal "download_complete":
                        // mEventSink?.success("download_complete")
                    }
                }
            }
            else -> result.notImplemented()
        }
    }

    override fun onListen(arguments: Any?, eventSink: EventChannel.EventSink?) {
        mEventSink = eventSink
        CoroutineScope(Dispatchers.Main).launch {
            bleL2capImpl.connectionState.collect { state: ConnectionState ->
                Log.d("L2capBlePlugin", "ConnectionState: $state")
                eventSink?.success(state.ordinal)
            }
        }
    }

    override fun onCancel(arguments: Any?) {
        mEventSink = null
    }

    private suspend fun KResult<Any>.mapToResult(@NonNull result: Result) {
        withContext(Dispatchers.Main) {
            if (isSuccess) {
                result.success(getOrNull())
            } else {
                result.error("error", exceptionOrNull()?.message, null)
            }
        }
    }

    override fun onDetachedFromEngine(@NonNull binding: FlutterPlugin.FlutterPluginBinding) {
        channel.setMethodCallHandler(null)
    }
}
```

---

## Flutter App Integration

In your Flutter app, you can call the new `downloadFile` method via the MethodChannel. For example:

```dart
import 'package:flutter/services.dart';

class L2capBleService {
  static const MethodChannel _channel = MethodChannel('l2cap_ble');
  static const EventChannel _eventChannel = EventChannel('getConnectionState');

  Future<bool> downloadFile({required int fileLength, required String savePath}) async {
    try {
      final bool success = await _channel.invokeMethod('downloadFile', {
        'fileLength': fileLength,
        'savePath': savePath,
      });
      return success;
    } catch (e) {
      print('Error: $e');
      return false;
    }
  }

  void listenForEvents() {
    _eventChannel.receiveBroadcastStream().listen((event) {
      // Here, the connection state (as an ordinal) is received.
      // You may wish to translate this or handle a special "download_complete" event if implemented.
      print('Event: $event');
      if (event == "download_complete") {
        print('Download complete');
      }
    });
  }
}
```

> **Permissions:**  
> In your `AndroidManifest.xml`, ensure that you include the required permissions (for example, to write to external storage):
>
> ```xml
> <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
> ```
>
> Also, make sure to handle runtime permission requests in your Flutter app.

---

## Summary of Changes and Checks

1. **Placeholders & Comments:**  
   - Comments such as “// TODO: Implement additional callback methods…” and checks for L2CAP support are still present. Replace these with proper error handling or platform checks.  
   - The Flutter save path in the sample code is a placeholder; update it with a valid path on the Android device.

2. **Missing Dependencies & Imports:**  
   - Added the import for `java.io.File` in `blel2capImpl.kt`.  
   - Changed the logging import in the plugin from `io.flutter.Log` to `android.util.Log`.

3. **Event Notification:**  
   - The tutorial initially proposed using `mEventSink` inside the download method. Since the event sink is part of the plugin (and not available in the low-level implementation), the notification for download completion is best handled in the Flutter plugin layer (or by exposing an additional stream from `BleL2capImpl`).

4. **General Considerations:**  
   - Verify that all required runtime permissions (Bluetooth, file storage, etc.) are granted.  
   - Ensure proper error handling and cancellation of any active coroutines when the Flutter plugin is detached.

This unified document should serve as a solid starting point. Adjust and extend the logic as necessary for your production needs.

Method channels are best suited for one-time, request-response interactions, where the Flutter side makes a call and waits for a single result (or error). In contrast, event channels are designed for continuous or streaming data, where the native side sends a stream of events (like sensor updates or connection state changes) to Flutter.

For example, when you call a function to download a file, you expect a single result once the download completes, so using a method channel is more natural. It simplifies the interaction by encapsulating the call and its outcome in one transaction, whereas an event channel would require you to handle a continuous stream of events—even if you only need one notification.

In summary:
- **Method Channel:** Use for discrete operations that return a single value or error.
- **Event Channel:** Use for ongoing, asynchronous data streams where multiple events are sent over time.

When an operation like a file download might take several minutes, a method channel that waits for a single result can be problematic due to potential timeouts or unresponsiveness in the Flutter UI. Instead, an event channel lets you continuously stream progress updates and notify Flutter as each significant milestone is reached. This way, the user gets real-time feedback throughout the download, and you avoid keeping a single method call open for an extended period. 

A common pattern is to use a method channel to initiate the download and then use an event channel to stream progress updates (and possibly a final completion event) back to Flutter. This hybrid approach combines the simplicity of initiating a task via a method call with the responsiveness of streaming events for long-running tasks.

To support long-running downloads with real‐time progress updates, you can combine a method channel to start (and eventually finish) the download with an event channel that streams progress updates. In this design:

- **Method Channel:** The Flutter app calls a method (e.g., `downloadFile`) that kicks off the download process.
- **Event Channel:** As the native code reads data in chunks, it calculates a progress percentage (or bytes downloaded) and sends these updates via an event channel. The Flutter app can then listen on this channel to update a progress bar.

Below is an example of how you might modify the native implementation.

---

### 1. Update the Library Interface and Implementation

#### **`ble2cap.kt`**
Add a new property for download progress if you want the interface to expose it:

```kotlin
package de.appsfactory.l2cap_ble

import kotlinx.coroutines.flow.Flow

interface BleL2cap {
    val connectionState: Flow<ConnectionState>
    val downloadProgress: Flow<Int>  // Progress as a percentage (0-100)

    fun connectToDevice(macAddress: String): Flow<Result<Boolean>>
    fun disconnectFromDevice(): Flow<Result<Boolean>>
    fun createL2capChannel(psm: Int): Flow<Result<Boolean>>
    fun sendMessage(message: ByteArray): Flow<Result<ByteArray>>

    // Method to download a file over L2CAP
    fun downloadFile(fileLength: Long, savePath: String): Flow<Result<Boolean>>
}

enum class ConnectionState {
    DISCONNECTED,
    CONNECTING,
    CONNECTED,
    DISCONNECTING,
    ERROR
}
```

#### **`blel2capImpl.kt`**
Extend your implementation to report progress events. A `MutableSharedFlow` is used to emit progress updates:

```kotlin
package de.appsfactory.l2cap_ble

import android.annotation.SuppressLint
import android.bluetooth.BluetoothAdapter
import android.bluetooth.BluetoothDevice
import android.bluetooth.BluetoothGatt
import android.bluetooth.BluetoothGattCallback
import android.bluetooth.BluetoothManager
import android.bluetooth.BluetoothSocket
import android.content.Context
import kotlinx.coroutines.CoroutineDispatcher
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.channels.Channel
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.MutableSharedFlow
import kotlinx.coroutines.flow.asSharedFlow
import kotlinx.coroutines.flow.flow
import kotlinx.coroutines.flow.flowOn
import kotlinx.coroutines.launch
import java.io.File
import kotlin.coroutines.coroutineContext

class BleL2capImpl(
    private val context: Context,
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO,
) : BleL2cap {

    private val connectionStateSharedFlow = MutableSharedFlow<ConnectionState>()

    // Flow for download progress updates
    private val downloadProgressFlow = MutableSharedFlow<Int>()
    override val downloadProgress: Flow<Int> = downloadProgressFlow.asSharedFlow()

    private val bluetoothManager: BluetoothManager? by lazy {
        context.getSystemService(Context.BLUETOOTH_SERVICE) as? BluetoothManager
    }
    private val bluetoothAdapter: BluetoothAdapter? by lazy {
        bluetoothManager?.adapter
    }

    private var bluetoothDevice: BluetoothDevice? = null
    private var bluetoothGatt: BluetoothGatt? = null
    private var bluetoothSocket: BluetoothSocket? = null

    override val connectionState: Flow<ConnectionState> = connectionStateSharedFlow.asSharedFlow()

    @SuppressLint("MissingPermission")
    override fun connectToDevice(macAddress: String): Flow<Result<Boolean>> = flow {
        val result = try {
            bluetoothDevice = bluetoothAdapter?.getRemoteDevice(macAddress)
            if (bluetoothDevice == null) {
                throw Exception("Device with address: $macAddress not found")
            }
            val connectionStateChannel = Channel<ConnectionState>(Channel.BUFFERED)
            connectionStateChannel.trySend(ConnectionState.CONNECTING)
            val gattCallback = object : BluetoothGattCallback() {
                // TODO: Implement additional callbacks as needed.
                override fun onConnectionStateChange(gatt: BluetoothGatt?, status: Int, newState: Int) {
                    super.onConnectionStateChange(gatt, status, newState)
                    when (newState) {
                        BluetoothGatt.STATE_CONNECTED -> connectionStateChannel.trySend(ConnectionState.CONNECTED)
                        BluetoothGatt.STATE_CONNECTING -> connectionStateChannel.trySend(ConnectionState.CONNECTING)
                        BluetoothGatt.STATE_DISCONNECTING -> connectionStateChannel.trySend(ConnectionState.DISCONNECTING)
                        BluetoothGatt.STATE_DISCONNECTED -> connectionStateChannel.trySend(ConnectionState.DISCONNECTED)
                    }
                }
            }
            CoroutineScope(coroutineContext).launch {
                for (state in connectionStateChannel) {
                    connectionStateSharedFlow.emit(state)
                }
            }
            bluetoothGatt = bluetoothDevice?.connectGatt(context, false, gattCallback)
            Result.success(true)
        } catch (e: Exception) {
            Result.failure(e)
        }
        emit(result)
    }.flowOn(ioDispatcher)

    @SuppressLint("MissingPermission")
    override fun disconnectFromDevice(): Flow<Result<Boolean>> = flow {
        val result = try {
            connectionStateSharedFlow.emit(ConnectionState.DISCONNECTING)
            bluetoothGatt?.disconnect()
            bluetoothGatt = null
            connectionStateSharedFlow.emit(ConnectionState.DISCONNECTED)
            Result.success(true)
        } catch (e: Exception) {
            Result.failure(e)
        }
        emit(result)
    }.flowOn(ioDispatcher)

    @SuppressLint("MissingPermission")
    override fun createL2capChannel(psm: Int): Flow<Result<Boolean>> = flow {
        val result = try {
            bluetoothSocket = bluetoothDevice?.createInsecureL2capChannel(psm)
            if (bluetoothSocket == null) {
                throw Exception("Failed to create L2CAP channel")
            }
            bluetoothSocket?.connect()
            Result.success(true)
        } catch (e: Exception) {
            Result.failure(e)
        }
        emit(result)
    }.flowOn(ioDispatcher)

    @SuppressLint("MissingPermission")
    override fun sendMessage(message: ByteArray): Flow<Result<ByteArray>> = flow {
        val result = try {
            if (bluetoothSocket == null) {
                throw Exception("Bluetooth socket is null")
            }
            bluetoothSocket?.outputStream?.write(message)
            // Read response from input stream.
            val response = ByteArray(1024)
            val bytesRead = bluetoothSocket?.inputStream?.read(response)
            bytesRead?.let {
                Result.success(response.copyOfRange(0, it))
            } ?: Result.failure(Exception("Failed to read response"))
        } catch (e: Exception) {
            Result.failure(e)
        }
        emit(result)
    }.flowOn(ioDispatcher)

    @SuppressLint("MissingPermission")
    override fun downloadFile(fileLength: Long, savePath: String): Flow<Result<Boolean>> = flow {
        val result = try {
            if (bluetoothSocket == null) {
                throw Exception("Bluetooth socket is null")
            }
            // Open file to save downloaded data.
            val file = File(savePath)
            file.outputStream().use { fileOutputStream ->
                val inputStream = bluetoothSocket!!.inputStream
                val buffer = ByteArray(1024)
                var bytesRead: Int
                var totalBytesRead = 0L

                while (totalBytesRead < fileLength) {
                    bytesRead = inputStream.read(buffer)
                    if (bytesRead == -1) {
                        throw Exception("Unexpected end of stream")
                    }
                    fileOutputStream.write(buffer, 0, bytesRead)
                    totalBytesRead += bytesRead

                    // Calculate progress percentage and emit update.
                    val progress = ((totalBytesRead * 100) / fileLength).toInt()
                    downloadProgressFlow.emit(progress)
                }
            }
            Result.success(true)
        } catch (e: Exception) {
            Result.failure(e)
        } finally {
            bluetoothSocket?.close()
            connectionStateSharedFlow.emit(ConnectionState.DISCONNECTED)
        }
        emit(result)
    }.flowOn(ioDispatcher)
}
```

---

### 2. Modify the Flutter Plugin to Expose a Download Progress Event Channel

Add a new event channel (for example, `"download_progress"`) to stream progress updates from the native side:

```kotlin
package de.appsfactory.l2cap_ble

import androidx.annotation.NonNull
import android.util.Log
import de.appsfactory.l2cap_ble.BleL2capImpl
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext
import io.flutter.embedding.engine.plugins.FlutterPlugin
import io.flutter.plugin.common.MethodCall
import io.flutter.plugin.common.MethodChannel
import io.flutter.plugin.common.MethodChannel.MethodCallHandler
import io.flutter.plugin.common.MethodChannel.Result
import io.flutter.plugin.common.EventChannel
import kotlin.Result as KResult

class L2capBlePlugin : FlutterPlugin, MethodCallHandler, EventChannel.StreamHandler {
    private lateinit var channel: MethodChannel
    private var connectionEventSink: EventChannel.EventSink? = null
    private var downloadProgressEventSink: EventChannel.EventSink? = null
    private lateinit var bleL2capImpl: BleL2capImpl

    override fun onAttachedToEngine(@NonNull flutterPluginBinding: FlutterPlugin.FlutterPluginBinding) {
        channel = MethodChannel(flutterPluginBinding.binaryMessenger, "l2cap_ble")
        channel.setMethodCallHandler(this)

        // Event channel for connection state events.
        val connectionChannel = EventChannel(flutterPluginBinding.binaryMessenger, "getConnectionState")
        connectionChannel.setStreamHandler(this)

        // New event channel for download progress updates.
        val downloadProgressChannel = EventChannel(flutterPluginBinding.binaryMessenger, "download_progress")
        downloadProgressChannel.setStreamHandler(object : EventChannel.StreamHandler {
            override fun onListen(arguments: Any?, events: EventChannel.EventSink?) {
                downloadProgressEventSink = events
                // Forward progress updates from the BLE implementation to Flutter.
                CoroutineScope(Dispatchers.Main).launch {
                    bleL2capImpl.downloadProgress.collect { progress ->
                        downloadProgressEventSink?.success(progress)
                    }
                }
            }
            override fun onCancel(arguments: Any?) {
                downloadProgressEventSink = null
            }
        })

        bleL2capImpl = BleL2capImpl(flutterPluginBinding.applicationContext, Dispatchers.IO)
    }

    override fun onMethodCall(@NonNull call: MethodCall, @NonNull result: Result) {
        when (call.method) {
            "connectToDevice" -> {
                CoroutineScope(Dispatchers.Main).launch {
                    val macAddress: String = requireNotNull(call.argument("deviceId"))
                    bleL2capImpl.connectToDevice(macAddress).collect { res: KResult<Boolean> ->
                        Log.d("L2capBlePlugin", "connectToDevice: $res")
                        res.mapToResult(result)
                    }
                }
            }
            "disconnectFromDevice" -> {
                CoroutineScope(Dispatchers.Main).launch {
                    bleL2capImpl.disconnectFromDevice().collect { res: KResult<Boolean> ->
                        Log.d("L2capBlePlugin", "disconnectFromDevice: $res")
                        res.mapToResult(result)
                    }
                }
            }
            "createL2capChannel" -> {
                CoroutineScope(Dispatchers.Main).launch {
                    val psm: Int = requireNotNull(call.argument("psm"))
                    bleL2capImpl.createL2capChannel(psm).collect { res: KResult<Boolean> ->
                        Log.d("L2capBlePlugin", "createL2capChannel: $res")
                        res.mapToResult(result)
                    }
                }
            }
            "sendMessage" -> {
                CoroutineScope(Dispatchers.Main).launch {
                    val message: ByteArray = requireNotNull(call.argument("message"))
                    bleL2capImpl.sendMessage(message).collect { res: KResult<ByteArray> ->
                        Log.d("L2capBlePlugin", "sendMessage: $res")
                        res.mapToResult(result)
                    }
                }
            }
            "downloadFile" -> {
                CoroutineScope(Dispatchers.Main).launch {
                    val fileLength: Long = requireNotNull(call.argument("fileLength"))
                    val savePath: String = requireNotNull(call.argument("savePath"))
                    bleL2capImpl.downloadFile(fileLength, savePath).collect { res: KResult<Boolean> ->
                        Log.d("L2capBlePlugin", "downloadFile: $res")
                        res.mapToResult(result)
                    }
                }
            }
            else -> result.notImplemented()
        }
    }

    // Connection state event handling.
    override fun onListen(arguments: Any?, eventSink: EventChannel.EventSink?) {
        connectionEventSink = eventSink
        CoroutineScope(Dispatchers.Main).launch {
            bleL2capImpl.connectionState.collect { state ->
                connectionEventSink?.success(state.ordinal)
            }
        }
    }

    override fun onCancel(arguments: Any?) {
        connectionEventSink = null
    }

    private suspend fun KResult<Any>.mapToResult(@NonNull result: Result) {
        withContext(Dispatchers.Main) {
            if (isSuccess) {
                result.success(getOrNull())
            } else {
                result.error("error", exceptionOrNull()?.message, null)
            }
        }
    }

    override fun onDetachedFromEngine(@NonNull binding: FlutterPlugin.FlutterPluginBinding) {
        channel.setMethodCallHandler(null)
    }
}
```

---

### 3. Flutter Side Usage

In your Flutter code, you can initiate the download with a method channel call and listen for progress updates on the `"download_progress"` event channel. For example:

```dart
import 'package:flutter/services.dart';

class L2capBleService {
  static const MethodChannel _methodChannel = MethodChannel('l2cap_ble');
  static const EventChannel _downloadProgressChannel = EventChannel('download_progress');

  Future<bool> downloadFile({
    required int fileLength,
    required String savePath,
  }) async {
    try {
      final bool success = await _methodChannel.invokeMethod('downloadFile', {
        'fileLength': fileLength,
        'savePath': savePath,
      });
      return success;
    } catch (e) {
      print('Download error: $e');
      return false;
    }
  }

  void listenForDownloadProgress(void Function(int progress) onProgress) {
    _downloadProgressChannel.receiveBroadcastStream().listen((event) {
      // event is the download progress percentage (0-100)
      onProgress(event as int);
    });
  }
}
```

Then, in your Flutter UI, you can call `downloadFile` to start the download and use `listenForDownloadProgress` to update a progress bar accordingly.

---

### Summary

By using both a method channel (to initiate the download) and an event channel (to stream progress updates), you let the Flutter app receive continuous feedback—even if the download takes several minutes—so that the user can see a progress bar updating in real time. This hybrid approach helps avoid long waits on a single method call and improves the user experience.



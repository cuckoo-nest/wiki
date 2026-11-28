# Protocol

### *Command* Structure (Client → Backplate)

A "Command" is a message frame sent from the client (our application) to the backplate device to request information or trigger an action. Every command follows a strict binary format that allows the device to parse and validate it.

The structure consists of a fixed-size header (Preamble), a variable-length body (Command ID, Length, and Payload), and a fixed-size footer (CRC). 

 The application **sends** a 3-byte preamble but **expects** a 4-byte preamble in return.


#### Overall Command Frame Layout

| Field            | Size (bytes)         | Endianness    | Description                                                                 |
| ---------------- | -------------------- | ------------- | --------------------------------------------------------------------------- |
| **Preamble**     | 3                    | N/A           | A fixed byte sequence (`0xd5 0xaa 0x96`) that marks the start of a command. |
| **Command ID**   | 2                    | Little-endian | A 16-bit code identifying the command's purpose (e.g., `0x00ff` for Reset). |
| **Data Length**  | 2                    | Little-endian | A 16-bit integer specifying the size in bytes of the _Data Payload_ field.  |
| **Data Payload** | 0 to 1024 (Variable) | Varies        | Optional data associated with the command. Can be empty.                    |
| **CRC**          | 2                    | Little-endian | A 16-bit CRC-CCITT checksum for command integrity verification.             |
#### CRC Calculation for Commands

The CRC is calculated over the **Command ID**, **Data Length**, and **Data Payload** fields combined. It does not include the 4-byte Preamble.

`CRC Input Data = [Command ID (2 bytes)] + [Data Length (2 bytes)] + [Data Payload (N bytes)]`

#### Example: A Complete "Reset" Command

The "Reset" command (`0x00ff`) has no data payload.

1.  **Preamble**: `d5 aa 96`
2.  **Command ID**: `ff 00` (0x00ff little-endian)
3.  **Data Length**: `00 00` (0 little-endian)
4.  **Data Payload**: (empty)
5.  **CRC**: `a3 4b` (CRC of `ff 00 00 00` is `0x4ba3`, stored as little-endian)

The final bytestream sent over the wire is: `d5 aa 96 ff 00 00 00 a3 4b`

| `d5 aa 96` | `ff 00`    | `00 00`     | (empty)      | `a3 4b` |
| ---------- | ---------- | ----------- | ------------ | ------- |
| Preamble   | Command ID | Data Length | Data Payload | CRC     |

***

### *Response* Structure (Backplate → Client)

A "Response" is a message frame sent from the backplate device back to the client. Responses can be a direct reply to a command, or they can be asynchronous data pushes (like sensor readings).

#### Response Frame Structure

The binary structure of a Response frame is similar to a Command frame. It uses a repeated `d5 d5`  Preamble, but the same field layout, and the same CRC calculation method. This consistency allows the same parsing logic to handle both outgoing and incoming messages.

The key difference lies in the **meaning** of the `Command ID` and the **content** of the `Data Payload`. In a response, the `Command ID` identifies the type of data being returned.

#### Overall Response Frame Layout

| Field            | Size (bytes)         | Endianness    | Description                                                                       |
| ---------------- | -------------------- | ------------- | --------------------------------------------------------------------------------- |
| **Preamble**     | 4                    | N/A           | A fixed byte sequence (`0xd5 0xd5 0xaa 0x96`) that marks the start of a response. |
| **Command ID**   | 2                    | Little-endian | A 16-bit code identifying the response's purpose (e.g., `0x0002` for Temp).       |
| **Data Length**  | 2                    | Little-endian | A 16-bit integer specifying the size in bytes of the _Data Payload_ field.        |
| **Data Payload** | 0 to 1024 (Variable) | Varies        | The actual data associated with the response. Can be empty.                       |
| **CRC**          | 2                    | Little-endian | A 16-bit CRC-CCITT checksum for response integrity verification.                  |
|                  |                      |               |                                                                                   |
#### Decoding a Response

The process of decoding a response involves parsing the frame and then interpreting its payload based on the `Command ID`.

**Step 1: Parse the Frame**
First, the raw bytestream is parsed to isolate a valid frame. This involves:
1.  Locating the `d5 d5 aa 96` Preamble.
2.  Reading the Command ID (2 bytes) and Data Length (2 bytes).
3.  Reading the number of bytes specified by the Data Length.
4.  Calculating the CRC on the Command ID, Length, and Payload.
5.  Comparing the calculated CRC with the CRC from the bytestream to ensure data integrity.

**Step 2: Interpret the Payload based on Command ID**
Once a valid frame is received, the `Command ID` determines how to interpret the bytes in the `Data Payload`. The payload is often a packed binary structure.
	
#### Common Response Examples

Here are examples showing how different `Command ID`s correspond to different payload structures.

**Example 1: ASCII Information Response (Command ID `0x0018`)**
*   **Context:** This is a reply to a "Get Version" command (`0x0098`).
*   **Payload Format:** The `Data Payload` is a simple ASCII string.
*   **Decoding:** Each byte in the payload is treated as a character and can be printed directly to form a human-readable string (e.g., "4.2.8 2019-04-03...").

**Example 2: Temperature/Humidity Data (Command ID `0x0002`)**
*   **Context:** This is a periodic sensor data push.
*   **Payload Format:** A 4-byte packed structure.
    *   **Bytes 0-1:** Temperature as a 16-bit signed little-endian integer. Represents degrees Celsius scaled by 100.
    *   **Bytes 2-3:** Humidity as a 16-bit unsigned little-endian integer. Represents relative humidity scaled by 10.
*   **Decoding:**
    1.  Read bytes 0 and 1: `int16_t temp_raw = (data[1] << 8) | data[0];`
    2.  Calculate the final value: `float temp_c = (float)temp_raw / 100.0;`
    3.  Read bytes 2 and 3: `uint16_t hum_raw = (data[3] << 8) | data[2];`
    4.  Calculate the final value: `float humidity = (float)hum_raw / 10.0;`

**Example 3: Backplate State Packet (Command ID `0x000b`)**
*   **Context:** Contains various voltage readings from the backplate.
*   **Payload Format:** A 16-byte (or more) packed structure with values at specific offsets.
    *   **Bytes 8-9:** Input Voltage (`Vin`) as a `uint16_t`. Scaled by 100.
    *   **Bytes 10-11:** Operational Voltage (`Vop`) as a `uint16_t`. Scaled by 1000.
    *   **Bytes 12-13:** Battery Voltage (`Vbat`) as a `uint16_t`. Scaled by 1000.
*   **Decoding:** Values are extracted from their specific byte offsets and scaled appropriately, similar to the temperature example.

## Initialization Sequence

#### **Device Timeout**: 
The backplate device (`/dev/ttyO2`) has a short timeout of approximately one second. If a client connects but does not perform the correct initialization sequence within this window, the device will stop responding, causing `read()` calls to return end-of-file.  

#### Handling Line Breaks 
It is easy to introduce a subtle bug in the serial port handling. The `termios` driver translates carriage return bytes (`0x0d`) to newline bytes (`0x0a`), which corrupted the CRC of several key messages and prevented a valid handshake. Disabling this automatic processing (`ICRNL` flag) resulted in a 100% reliable handshake. We found this because the response for `0x009e` (Get hardware version) was cut short, returning only "Backplate-", but that's because the real value had a `0x0d` in it 


#### **Phase 1: Initiation & Burst**
The client actively initiates the session and waits for a specific signal to proceed.
1.  **Client -> Device**: The initial handshake is to open the port, send a `tcsendbreak`, and then send a single `0x00ff` (Reset) command. This action alone is sufficient to trigger the backplate's full initial response burst (version, FET presence, etc.) which takes place within the first few seconds
    *   Open `/dev/ttyO2` and configure it for 115200 baud.
    *   Send a `tcsendbreak`.
    *   Send command `0x00ff` (Reset).
2.  **Device -> Client**: The device responds with a burst of messages, the last of which is an ASCII "BRK".
```
0001 msg "<version> <build timestamp "YYYY-MM-DD HH:MM:SS"> K"
0004 wire state (13 bytes; called "FET presence" in earlier notes)
0009 pin state (15 bytes)
0001 msg "\*sense 06d1 06d0 0001 0001 0000 0000 0000 0001 06d9 06d8 06d8; detect 09d3 065c"
0001 msg "BRK"
```

3.  **Client Action**: The client must listen and collect all messages until it receives the "BRK" message before moving to the next phase.

**Phase 2: Client ACK & Info Gathering**
The client acknowledges the burst and then rapidly gathers all static information from the device.
1.  **Client -> Device (Rapid Volley)**: The client sends three commands back-to-back without waiting for individual responses:
    *   `0x008f` (wire ACK, earlier "FET Presence ACK"). **Its payload must be the 13 bytes of the `0x0004` message, echoed back.** With an empty payload the backplate still answers the queries below but never starts its sensors and ignores switch commands; see [Heating control](Heating%20control.md).
    *   `0x0083` (Periodic Status Request)
    *   `0x0090` (Get Mono/TFE ID)

2.  **Client -> Device (Synchronous Queries)**: After the volley, the client engages in a series of synchronous, paced request/response calls to get all device metadata. A small (50ms) delay between each call is required for reliability. The sequence is:
	*   **`0x0090`**: Get Mono/TFE ID.
	*   **`0x0098`**: Get Mono/TFE Version.
	*   **`0x0099`**: Get Mono/TFE Build Info.
	*   **`0x009d`**: Get Backplate (BP) Model / BSL ID.
	*   **`0x009b`**: Get BSL Version.
	*   **`0x009c`**: Get Co-processor (Co-proc) BSL Version/Info.
	*   **`0x009f`**: Get Serial Number.
	*   **`0x009e`**: Get Hardware Version / Backplate Name.


**Phase 3: Set Operational Mode**
After gathering information, the client puts the device into its main operational mode.
1.  **Client -> Device**: Send `0x00c0` (Set power steal mode) with a 4-byte `00000000` payload.

**Phase 4: Main Polling Loop**
This is the final, steady-state operation for receiving sensor data. It is an active polling cycle, not a passive stream.
1.  **Polling**: Every ~30 seconds, the client performs a buffer request:
    *   **Client -> Device**: Send `0x00a2` (Request Historical Data Buffers).
    *   **Device -> Client**: The device responds with a burst of sensor data messages.
    *   **Device -> Client**: The burst is terminated with a specific `0x002f` (End of Buffers) message.
    *   **Client -> Device**: The client must immediately respond with `0x00a3` (Acknowledge End of Buffers) to complete the cycle.
2.  **Keep-Alive**: Separately, the client sends a `0x0083` (Periodic Status Request) every ~15 seconds to keep the connection alive.



3.  **Handshake Confirmation**: Once initiated, the rest of the documented handshake proceeds as expected. The `cuckoo_talk` tool successfully:
    *   Received the initial burst (version, FET presence, sense info, "BRK").
    *   Responded with `0x008f` (FET presence ACK).
    *   Sent `0x0090` (get TFE id) and received the correct `0x0010` response.
    *   Sent `0x0098` (get TFE version) and `0x0099` (get TFE build info) and received the correct `0x0018` and `0x0019` ASCII responses.

4.  **Periodic Data Behavior**: Sending the `0x0083` command, even repeatedly after the handshake is complete, does not automatically start a continuous stream of sensor data. This indicates that the initialization sequence only completes "Phase 1" (establishing a valid session). A "Phase 2" is likely required, where further commands must be sent to configure sensor thresholds or explicitly enable data reporting before the backplate will begin streaming periodic updates.

    *Update (found with a capture of stock `nlclient` on firmware `AMBER_BP 1.0.26`):* no further configuration is needed. The stream did not start because `0x8f` was sent without payload. With the 13 bytes of `0x0004` echoed in `0x8f`, followed by the queries, `0xc0 00000000`, `0xa2` and `0xa3`, the backplate streams by itself.



**Phase 2: Polling for Rich Sensor Data (Repeats ~30 seconds)**

This is the primary mechanism for retrieving detailed sensor data like temperature and humidity.

1.  **Request Buffers**: `nlclient` sends command `0x00a2` (Request "buffers").
2.  **Receive Data Burst**: The backplate responds immediately with a burst of data messages. The exact messages can vary, but typically include:
    *   `0x0022` (Buffered temperature reading)
    *   `0x0023` (BufferedSourceTemperature / PIR sensor data)
    *   `0x000c` (Ambient Light Sensor data)
    *   `0x0038` (Newly discovered data type)
3.  **End of Burst Marker**: The backplate concludes the burst by sending `0x002f` (End of "buffers").
4.  **Acknowledge**: `nlclient` immediately sends `0x00a3` (ACK to end of buffers).

This entire `0x00a2` -> burst -> `0x00a3` cycle repeats approximately every 30 seconds.

**Phase 3: Continuous High-Frequency Data Stream**

After the *first* successful Phase 2 cycle, the backplate enters a mode where it continuously streams a subset of high-frequency data without being asked.

*   `0x0007` (near pir?) - Sent approximately once per second.
*   `0x000a` (als?) - Sent approximately once per second.

This stream provides constant updates for presence and light, while the more detailed data is fetched by the 30-second polling cycle.

### Other Observed Behaviors

*   **Recovery/Re-Handshake**: The logs show that `nlclient` may occasionally send `0x00ff` (Reset) during normal operation, triggering a full re-handshake. This is likely an error recovery mechanism.
*   **Miscellaneous Commands**: Other commands like `0x00b1` ("temperature lock") and `0x0082` (circuit switch: `[circuit][state]`, answered by `0x0006`; see [Heating control](Heating%20control.md)) are sent in response to user interaction or system events.
*   **Sleep**: `0x00a4` (wake-up mask) and `0x00a1` (quiet, with the maximum sleep time) are sent before the head unit suspends; see [Sleep and wake-up](Sleep%20and%20wake.md).



## CRC checksum
### Protocol Checksum (CRC-CCITT)

The protocol employs a 16-bit Cyclic Redundancy Check (CRC) to verify the integrity of the message data. The CRC is calculated over the **`Command` (2 bytes)**, the **`Data Length` (2 bytes)**, and the entire **`Data` payload** itself.

#### Algorithm Details

The implementation uses the standard CRC-CCITT algorithm with the following parameters:

- **Polynomial:** `0x1021`
- **Initial Value:** `0x0000`
- **Data Coverage:** The payload.
- **Storage Format:** The resulting 16-bit CRC is stored in little-endian (byte-swapped) format.

#### Calculation Method

The CRC is calculated by determining the contribution of each bit in the 6-byte data payload. The final checksum is the XOR sum of the pre-calculated values for each bit that is set to `1`.

For example, a `1` bit at the start of the 6-byte payload (47 bits from the end) contributes a value of `0x687b` to the checksum, while a `1` bit 23 bits from the end contributes `0x20d4`.

#### Example

Consider a data stream ending in the bytes `... 82 00 02 00 00 00`. The CRC for these six bytes is `0x08b2`.

1. **Data for CRC:** `82 00 02 00 00 00`
2. **Identify Bits:** The non-zero bits in this data are:
    - Byte 0 (`0x82`): bit 7 and bit 1
    - Byte 2 (`0x02`): bit 1
3. **Sum Contributions:** We find the corresponding values for these bit positions and XOR them together.
    - The bit at byte 0, position 7 corresponds to the XOR value `0x68ed`.
    - The bit at byte 0, position 1 corresponds to the XOR value `0x408b`.
    - The bit at byte 2, position 1 corresponds to the XOR value `0x20d4`.
4. **Final CRC:**
    
    ```
    0x68ed XOR 0x408b XOR 0x20d4 = 0x08b2
    ```
    
#### Programmatic Verification

The calculation can be verified using a standard CRC library. The following Perl script demonstrates how to compute the checksum for the example data, accounting for the little-endian format of the final result.

```
#!/usr/bin/env perl
use Digest::CRC qw(crc);

# The 6 bytes of data to be checked.
my $data = pack("H*", "820002000000");

# Calculate the CRC using the CCITT polynomial 0x1021.
# The result is byte-swapped because the protocol stores it as little-endian.
my $checksum = crc($data, 16, 0, 0, 0, 0x1021, 0, 0);

# Prints "b208", which is the little-endian representation of 0x08b2.
printf("%04x\n", $checksum);
```

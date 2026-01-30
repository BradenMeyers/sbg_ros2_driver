# Manual Origin Override Feature

## Overview
This feature allows users to manually specify a datum (origin) for the odometry coordinate system, instead of relying on the first valid GPS fix. This is similar to the `navsat_transform_node` behavior in ROS.

## Configuration Parameters

Two new parameters have been added to the `odometry` section of the configuration files:

### `odometry.wait_for_datum` (boolean)
- **Default**: `false`
- **Description**: If `true`, the driver will use the manually specified datum from the `datum` parameter. If the `datum` parameter is not provided or invalid, the driver will wait and not publish odometry until a valid datum is configured.
- **When false**: Uses the first valid GPS fix as the origin (original behavior).

### `odometry.datum` (array of 3 doubles)
- **Format**: `[latitude, longitude, altitude]`
- **Units**: 
  - Latitude: decimal degrees
  - Longitude: decimal degrees  
  - Altitude: meters (WGS84)
- **Example**: `[55.944904, -3.186693, 0.0]`
- **Description**: The reference origin for the odometry coordinate system. Only used when `wait_for_datum` is `true`.

## Usage Examples

### Example 1: Use First Valid GPS Fix (Default Behavior)
```yaml
odometry:
  enable: true
  wait_for_datum: false
```
This is the original behavior - the driver will use the first valid GPS position as the origin.

### Example 2: Use Manual Datum
```yaml
odometry:
  enable: true
  wait_for_datum: true
  datum: [55.944904, -3.186693, 0.0]  # Edinburgh, Scotland
```
The driver will use the specified coordinates as the origin. All odometry positions will be relative to this point.

### Example 3: Wait for Datum Configuration
```yaml
odometry:
  enable: true
  wait_for_datum: true
  # datum not specified - will wait until provided
```
The driver will not publish odometry messages until a valid datum is configured. A warning will be logged every 5 seconds.

## Behavior

### When `wait_for_datum: false` (Default)
1. Driver receives first valid EKF navigation message
2. Extracts latitude, longitude, altitude from the message
3. Initializes UTM zone and computes easting/northing
4. Uses these values as the reference origin
5. All subsequent positions are computed relative to this origin

### When `wait_for_datum: true` with valid datum
1. Driver starts up and reads datum from configuration
2. Validates that datum contains exactly 3 values
3. Immediately initializes UTM zone using datum coordinates
4. Computes reference easting/northing/altitude from datum
5. All positions are computed relative to the datum
6. Logs: "initialized from manual datum - lat:X long:Y alt:Z"

### When `wait_for_datum: true` without valid datum
1. Driver starts up but datum is missing or invalid
2. Does not initialize UTM/origin
3. Does not publish odometry messages
4. Logs warning every 5 seconds: "Waiting for datum configuration..."
5. Continues to receive GPS data but doesn't process it for odometry

## Implementation Details

### Files Modified
1. **config_store.h / config_store.cpp**
   - Added `odom_wait_for_datum_` member variable
   - Added `odom_datum_` member variable
   - Added getter methods for both parameters
   - Updated `loadOdomParameters()` to read new parameters

2. **message_wrapper.h / message_wrapper.cpp**
   - Added `odom_wait_for_datum_` member variable
   - Added `odom_datum_` member variable
   - Added setter methods for both parameters
   - Updated `createRosOdoMessage()` to implement new logic

3. **message_publisher.cpp**
   - Updated `initPublishers()` to pass new parameters to message wrapper

4. **Configuration Files** (all three variants)
   - sbg_device_uart_default.yaml
   - sbg_device_udp_default.yaml
   - sbg_device_file_default.yaml
   - Added comprehensive documentation for new parameters
   - Added commented example showing how to use the feature

5. **README.md**
   - Updated odometry section to document the new feature
   - Added usage examples

## Compatibility

- **Backward Compatible**: Yes. The default value of `wait_for_datum: false` maintains the original behavior.
- **ROS2 Version**: Compatible with all ROS2 versions that support the existing driver
- **Device Support**: Works with all SBG devices that support EKF navigation

## Testing

To test this feature:

1. **Test default behavior (no changes)**:
   - Use existing configuration files
   - Verify odometry still uses first valid GPS fix

2. **Test manual datum**:
   - Set `wait_for_datum: true`
   - Set `datum: [lat, lon, alt]` with known coordinates
   - Verify odometry positions are relative to specified origin
   - Check log message confirms manual datum initialization

3. **Test wait behavior**:
   - Set `wait_for_datum: true`
   - Do not provide datum parameter
   - Verify warning messages are logged
   - Verify no odometry is published

## Notes

- The datum altitude should be specified as height above WGS84 ellipsoid (not MSL)
- The UTM zone is automatically determined from the datum coordinates
- Once initialized, the origin cannot be changed without restarting the node
- The feature works with both `SbgImuData` and `SbgImuShort` message types

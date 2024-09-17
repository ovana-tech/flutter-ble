import 'package:flutter/material.dart';
import 'package:flutter_blue_plus/flutter_blue_plus.dart';

class BluetoothDevicesPage extends StatefulWidget {
  @override
  _BluetoothDevicesPageState createState() => _BluetoothDevicesPageState();
}

class _BluetoothDevicesPageState extends State<BluetoothDevicesPage> {
  List<BluetoothDevice> _devicesList = [];
  bool _isScanning = false;

  @override
  void initState() {
    super.initState();
    _startScan();
  }

  void _startScan() async {
    if (_isScanning) return;

    setState(() {
      _devicesList.clear();
      _isScanning = true;
    });

    FlutterBluePlus.scanResults.listen((results) {
      for (ScanResult result in results) {
        if (!_devicesList.contains(result.device)) {
          setState(() {
            _devicesList.add(result.device);
          });
        }
      }
    });

    await FlutterBluePlus.startScan(timeout: Duration(seconds: 20));

    setState(() {
      _isScanning = false;
    });
  }

  Future<void> _connectToDevice(BluetoothDevice device) async {
    try {
      setState(() {
        _isScanning = true;
      });

      await device.connect(autoConnect: false);

      List<BluetoothService> services = await device.discoverServices();

      for (BluetoothService service in services) {
        print('Service: ${service.uuid}');
        for (BluetoothCharacteristic characteristic
            in service.characteristics) {
          print('Characteristic: ${characteristic.uuid}');
        }
      }

      setState(() {
        _isScanning = false;
      });

      Navigator.push(
        context,
        MaterialPageRoute(
          builder: (context) => ConnectedDevicePage(device: device),
        ),
      );

      device.state.listen((state) {
        if (state == BluetoothDeviceState.disconnected) {
          _showConnectionError(
              "Connection lost to ${device.name ?? 'Unknown Device'}");
        }
      });
    } catch (e) {
      setState(() {
        _isScanning = false;
      });

      if (e.toString().contains('133')) {
        _showRetryDialog(device);
      } else {
        _showConnectionError(e);
        print(e);
      }
    }
  }

  void _showRetryDialog(BluetoothDevice device) {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: Text('Connection Failed'),
        content:
            Text('Failed to connect to device. Would you like to try again?'),
        actions: [
          TextButton(
            onPressed: () {
              Navigator.pop(context);
              _connectToDevice(device);
            },
            child: Text('Retry'),
          ),
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: Text('Cancel'),
          ),
        ],
      ),
    );
  }

  void _showConnectionError(dynamic error) {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: Text('Connection Error'),
        content: Text('Failed to connect: $error'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: Text('OK'),
          ),
        ],
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Available Bluetooth Devices'),
        actions: [
          if (_isScanning)
            Padding(
              padding: const EdgeInsets.all(8.0),
              child: CircularProgressIndicator(),
            )
          else
            IconButton(
              icon: Icon(Icons.refresh),
              onPressed: _startScan,
            ),
        ],
      ),
      body: _devicesList.isEmpty
          ? Center(
              child: Text(
                _isScanning ? 'Scanning...' : 'No devices found',
                style: TextStyle(fontSize: 18),
              ),
            )
          : ListView.builder(
              itemCount: _devicesList.length,
              itemBuilder: (context, index) {
                final device = _devicesList[index];
                return ListTile(
                  leading: Icon(Icons.bluetooth),
                  title: Text(
                      device.name.isNotEmpty ? device.name : 'Unknown Device'),
                  subtitle: Text(device.id.toString()),
                  trailing: ElevatedButton(
                    onPressed: () => _connectToDevice(device),
                    child: Text('Connect'),
                  ),
                );
              },
            ),
    );
  }
}

class ConnectedDevicePage extends StatelessWidget {
  final BluetoothDevice device;

  ConnectedDevicePage({required this.device});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Connected Device'),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text(
              'Connected to Device:',
              style: TextStyle(fontSize: 20),
            ),
            SizedBox(height: 20),
            Text(
              device.id.toString(),
              style: TextStyle(fontSize: 24, fontWeight: FontWeight.bold),
            ),
          ],
        ),
      ),
    );
  }
}

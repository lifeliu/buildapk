---
layout: post
title: "Flutter插件开发与发布完整指南"
categories: flutter
tags: [Flutter, Plugin, Package, 插件开发, 发布, pub.dev]
date: 2022-11-25
image: https://flutter.dev/assets/images/shared/brand/flutter/logo/flutter-lockup.png
---

Flutter插件是扩展Flutter功能的重要方式，通过插件可以访问平台特定的API、集成第三方SDK，或者封装可复用的功能模块。本文将深入探讨Flutter插件的开发流程、架构设计、平台集成、测试策略以及发布到pub.dev的完整过程。

## Flutter插件架构深入解析

### 插件类型与架构模式

Flutter插件主要分为三种类型：

1. **Dart包（Dart Package）**：纯Dart代码实现，不涉及平台特定功能
2. **插件包（Plugin Package）**：包含平台特定代码的插件
3. **联合插件（Federated Plugin）**：将插件拆分为多个包的架构模式

```dart
// 插件架构管理器
class PluginArchitectureManager {
  static PluginArchitectureManager? _instance;
  static PluginArchitectureManager get instance => _instance ??= PluginArchitectureManager._();
  
  final Map<String, PluginInterface> _registeredPlugins = {};
  final Map<String, PluginMetadata> _pluginMetadata = {};
  
  PluginArchitectureManager._();
  
  // 注册插件
  void registerPlugin<T extends PluginInterface>(String name, T plugin, PluginMetadata metadata) {
    _registeredPlugins[name] = plugin;
    _pluginMetadata[name] = metadata;
    
    debugPrint('Plugin registered: $name (${metadata.version})');
  }
  
  // 获取插件实例
  T? getPlugin<T extends PluginInterface>(String name) {
    final plugin = _registeredPlugins[name];
    if (plugin is T) {
      return plugin;
    }
    return null;
  }
  
  // 检查插件是否可用
  Future<bool> isPluginAvailable(String name) async {
    final plugin = _registeredPlugins[name];
    if (plugin == null) return false;
    
    try {
      return await plugin.isAvailable();
    } catch (e) {
      debugPrint('Plugin availability check failed for $name: $e');
      return false;
    }
  }
  
  // 获取所有已注册的插件
  List<String> getRegisteredPlugins() {
    return _registeredPlugins.keys.toList();
  }
  
  // 获取插件元数据
  PluginMetadata? getPluginMetadata(String name) {
    return _pluginMetadata[name];
  }
  
  // 卸载插件
  Future<void> unregisterPlugin(String name) async {
    final plugin = _registeredPlugins[name];
    if (plugin != null) {
      await plugin.dispose();
      _registeredPlugins.remove(name);
      _pluginMetadata.remove(name);
      debugPrint('Plugin unregistered: $name');
    }
  }
}

// 插件接口定义
abstract class PluginInterface {
  String get name;
  String get version;
  
  Future<bool> isAvailable();
  Future<void> initialize();
  Future<void> dispose();
}

// 插件元数据
class PluginMetadata {
  final String name;
  final String version;
  final String description;
  final String author;
  final List<String> supportedPlatforms;
  final Map<String, dynamic> configuration;
  
  const PluginMetadata({
    required this.name,
    required this.version,
    required this.description,
    required this.author,
    required this.supportedPlatforms,
    this.configuration = const {},
  });
  
  Map<String, dynamic> toJson() {
    return {
      'name': name,
      'version': version,
      'description': description,
      'author': author,
      'supportedPlatforms': supportedPlatforms,
      'configuration': configuration,
    };
  }
  
  factory PluginMetadata.fromJson(Map<String, dynamic> json) {
    return PluginMetadata(
      name: json['name'] as String,
      version: json['version'] as String,
      description: json['description'] as String,
      author: json['author'] as String,
      supportedPlatforms: List<String>.from(json['supportedPlatforms'] as List),
      configuration: json['configuration'] as Map<String, dynamic>? ?? {},
    );
  }
}

// 基础插件实现
abstract class BasePlugin implements PluginInterface {
  final PluginMetadata metadata;
  bool _isInitialized = false;
  
  BasePlugin(this.metadata);
  
  @override
  String get name => metadata.name;
  
  @override
  String get version => metadata.version;
  
  bool get isInitialized => _isInitialized;
  
  @override
  Future<bool> isAvailable() async {
    // 检查平台支持
    final currentPlatform = _getCurrentPlatform();
    if (!metadata.supportedPlatforms.contains(currentPlatform)) {
      return false;
    }
    
    // 子类可以重写此方法添加更多检查
    return await checkPlatformSpecificAvailability();
  }
  
  @override
  Future<void> initialize() async {
    if (_isInitialized) return;
    
    try {
      await onInitialize();
      _isInitialized = true;
      debugPrint('Plugin initialized: $name');
    } catch (e) {
      debugPrint('Plugin initialization failed for $name: $e');
      rethrow;
    }
  }
  
  @override
  Future<void> dispose() async {
    if (!_isInitialized) return;
    
    try {
      await onDispose();
      _isInitialized = false;
      debugPrint('Plugin disposed: $name');
    } catch (e) {
      debugPrint('Plugin disposal failed for $name: $e');
    }
  }
  
  // 子类需要实现的方法
  Future<bool> checkPlatformSpecificAvailability();
  Future<void> onInitialize();
  Future<void> onDispose();
  
  // 获取当前平台
  String _getCurrentPlatform() {
    if (Platform.isAndroid) return 'android';
    if (Platform.isIOS) return 'ios';
    if (Platform.isWindows) return 'windows';
    if (Platform.isMacOS) return 'macos';
    if (Platform.isLinux) return 'linux';
    if (Platform.isFuchsia) return 'fuchsia';
    return 'web';
  }
}
```

### Method Channel深入实现

```dart
// 高级Method Channel封装
class AdvancedMethodChannel {
  final MethodChannel _channel;
  final String _name;
  final Map<String, Completer<dynamic>> _pendingCalls = {};
  final StreamController<MethodCall> _incomingCalls = StreamController.broadcast();
  
  AdvancedMethodChannel(this._name) : _channel = MethodChannel(_name) {
    _channel.setMethodCallHandler(_handleMethodCall);
  }
  
  // 处理来自原生端的方法调用
  Future<dynamic> _handleMethodCall(MethodCall call) async {
    _incomingCalls.add(call);
    
    // 处理回调类型的方法调用
    if (call.method.startsWith('callback_')) {
      final callId = call.arguments['callId'] as String?;
      if (callId != null && _pendingCalls.containsKey(callId)) {
        final completer = _pendingCalls.remove(callId)!;
        
        if (call.arguments['error'] != null) {
          completer.completeError(
            PlatformException(
              code: call.arguments['error']['code'] as String,
              message: call.arguments['error']['message'] as String,
              details: call.arguments['error']['details'],
            ),
          );
        } else {
          completer.complete(call.arguments['result']);
        }
      }
      return null;
    }
    
    // 其他方法调用的处理
    return null;
  }
  
  // 调用原生方法（带超时和重试）
  Future<T?> invokeMethod<T>(
    String method, {
    dynamic arguments,
    Duration timeout = const Duration(seconds: 30),
    int maxRetries = 3,
  }) async {
    Exception? lastException;
    
    for (int attempt = 0; attempt < maxRetries; attempt++) {
      try {
        final result = await _channel.invokeMethod<T>(method, arguments)
            .timeout(timeout);
        return result;
      } on TimeoutException catch (e) {
        lastException = e;
        debugPrint('Method call timeout (attempt ${attempt + 1}/$maxRetries): $method');
        
        if (attempt < maxRetries - 1) {
          await Future.delayed(Duration(milliseconds: 500 * (attempt + 1)));
        }
      } on PlatformException catch (e) {
        lastException = e;
        debugPrint('Platform exception in method call: $method - ${e.message}');
        
        // 某些错误不需要重试
        if (e.code == 'UNAVAILABLE' || e.code == 'UNIMPLEMENTED') {
          break;
        }
        
        if (attempt < maxRetries - 1) {
          await Future.delayed(Duration(milliseconds: 500 * (attempt + 1)));
        }
      } catch (e) {
        lastException = Exception('Unexpected error: $e');
        debugPrint('Unexpected error in method call: $method - $e');
        break;
      }
    }
    
    throw lastException ?? Exception('Method call failed: $method');
  }
  
  // 异步方法调用（使用回调机制）
  Future<T> invokeAsyncMethod<T>(
    String method, {
    dynamic arguments,
    Duration timeout = const Duration(minutes: 5),
  }) async {
    final callId = _generateCallId();
    final completer = Completer<T>();
    
    _pendingCalls[callId] = completer;
    
    // 设置超时
    Timer(timeout, () {
      if (_pendingCalls.containsKey(callId)) {
        _pendingCalls.remove(callId);
        if (!completer.isCompleted) {
          completer.completeError(
            TimeoutException('Async method call timeout', timeout),
          );
        }
      }
    });
    
    try {
      await _channel.invokeMethod('${method}_async', {
        'callId': callId,
        ...?arguments as Map<String, dynamic>?,
      });
      
      return await completer.future;
    } catch (e) {
      _pendingCalls.remove(callId);
      rethrow;
    }
  }
  
  // 批量方法调用
  Future<List<T?>> invokeBatchMethods<T>(List<BatchMethodCall> calls) async {
    final results = <T?>[];
    
    for (final call in calls) {
      try {
        final result = await invokeMethod<T>(
          call.method,
          arguments: call.arguments,
          timeout: call.timeout,
          maxRetries: call.maxRetries,
        );
        results.add(result);
      } catch (e) {
        if (call.continueOnError) {
          results.add(null);
          debugPrint('Batch method call failed (continuing): ${call.method} - $e');
        } else {
          rethrow;
        }
      }
    }
    
    return results;
  }
  
  // 监听来自原生端的方法调用
  Stream<MethodCall> get incomingCalls => _incomingCalls.stream;
  
  // 生成唯一的调用ID
  String _generateCallId() {
    return '${DateTime.now().millisecondsSinceEpoch}_${Random().nextInt(10000)}';
  }
  
  // 清理资源
  void dispose() {
    _incomingCalls.close();
    
    // 取消所有待处理的调用
    for (final completer in _pendingCalls.values) {
      if (!completer.isCompleted) {
        completer.completeError(Exception('Channel disposed'));
      }
    }
    _pendingCalls.clear();
  }
}

// 批量方法调用配置
class BatchMethodCall {
  final String method;
  final dynamic arguments;
  final Duration timeout;
  final int maxRetries;
  final bool continueOnError;
  
  const BatchMethodCall({
    required this.method,
    this.arguments,
    this.timeout = const Duration(seconds: 30),
    this.maxRetries = 3,
    this.continueOnError = false,
  });
}
```

### Event Channel高级应用

```dart
// 高级Event Channel封装
class AdvancedEventChannel {
  final EventChannel _channel;
  final String _name;
  StreamSubscription? _subscription;
  final StreamController<dynamic> _controller = StreamController.broadcast();
  bool _isListening = false;
  
  AdvancedEventChannel(this._name) : _channel = EventChannel(_name);
  
  // 开始监听事件
  Stream<T> listen<T>({
    void Function(dynamic error)? onError,
    bool cancelOnError = false,
  }) {
    if (!_isListening) {
      _subscription = _channel.receiveBroadcastStream().listen(
        (data) {
          _controller.add(data);
        },
        onError: (error) {
          debugPrint('Event channel error ($_name): $error');
          if (onError != null) {
            onError(error);
          } else {
            _controller.addError(error);
          }
        },
        cancelOnError: cancelOnError,
      );
      _isListening = true;
    }
    
    return _controller.stream.cast<T>();
  }
  
  // 停止监听
  Future<void> stopListening() async {
    if (_isListening) {
      await _subscription?.cancel();
      _subscription = null;
      _isListening = false;
    }
  }
  
  // 过滤事件流
  Stream<T> where<T>(bool Function(T event) test) {
    return listen<T>().where(test);
  }
  
  // 转换事件流
  Stream<R> map<T, R>(R Function(T event) mapper) {
    return listen<T>().map(mapper);
  }
  
  // 缓冲事件
  Stream<List<T>> buffer<T>(Duration duration) {
    return listen<T>()
        .bufferTime(duration)
        .where((list) => list.isNotEmpty);
  }
  
  // 去重事件
  Stream<T> distinct<T>() {
    return listen<T>().distinct();
  }
  
  // 限流事件
  Stream<T> throttle<T>(Duration duration) {
    return listen<T>().throttleTime(duration);
  }
  
  // 防抖事件
  Stream<T> debounce<T>(Duration duration) {
    return listen<T>().debounceTime(duration);
  }
  
  // 清理资源
  void dispose() {
    stopListening();
    _controller.close();
  }
}

// Stream扩展方法
extension StreamExtensions<T> on Stream<T> {
  Stream<List<T>> bufferTime(Duration duration) {
    return Stream.periodic(duration)
        .switchMap((_) => take(1).toList().asStream())
        .where((list) => list.isNotEmpty);
  }
  
  Stream<T> throttleTime(Duration duration) {
    DateTime? lastEmission;
    return where((event) {
      final now = DateTime.now();
      if (lastEmission == null || now.difference(lastEmission!) >= duration) {
        lastEmission = now;
        return true;
      }
      return false;
    });
  }
  
  Stream<T> debounceTime(Duration duration) {
    Timer? timer;
    final controller = StreamController<T>();
    
    listen(
      (event) {
        timer?.cancel();
        timer = Timer(duration, () {
          controller.add(event);
        });
      },
      onError: controller.addError,
      onDone: () {
        timer?.cancel();
        controller.close();
      },
    );
    
    return controller.stream;
  }
}
```

## 实际插件开发案例

### 设备信息插件开发

```dart
// 设备信息插件主类
class DeviceInfoPlugin extends BasePlugin {
  static const String _channelName = 'com.example.device_info';
  late final AdvancedMethodChannel _methodChannel;
  late final AdvancedEventChannel _eventChannel;
  
  DeviceInfoPlugin() : super(
    const PluginMetadata(
      name: 'device_info_plus',
      version: '1.0.0',
      description: 'A comprehensive device information plugin',
      author: 'Your Name',
      supportedPlatforms: ['android', 'ios', 'windows', 'macos', 'linux'],
    ),
  );
  
  @override
  Future<bool> checkPlatformSpecificAvailability() async {
    try {
      await _methodChannel.invokeMethod('isAvailable');
      return true;
    } catch (e) {
      return false;
    }
  }
  
  @override
  Future<void> onInitialize() async {
    _methodChannel = AdvancedMethodChannel(_channelName);
    _eventChannel = AdvancedEventChannel('${_channelName}_events');
    
    // 注册插件到架构管理器
    PluginArchitectureManager.instance.registerPlugin(
      'device_info',
      this,
      metadata,
    );
  }
  
  @override
  Future<void> onDispose() async {
    _methodChannel.dispose();
    _eventChannel.dispose();
  }
  
  // 获取基本设备信息
  Future<DeviceInfo> getDeviceInfo() async {
    final result = await _methodChannel.invokeMethod<Map<String, dynamic>>('getDeviceInfo');
    if (result == null) {
      throw Exception('Failed to get device info');
    }
    
    return DeviceInfo.fromJson(result);
  }
  
  // 获取详细硬件信息
  Future<HardwareInfo> getHardwareInfo() async {
    final result = await _methodChannel.invokeMethod<Map<String, dynamic>>('getHardwareInfo');
    if (result == null) {
      throw Exception('Failed to get hardware info');
    }
    
    return HardwareInfo.fromJson(result);
  }
  
  // 获取网络信息
  Future<NetworkInfo> getNetworkInfo() async {
    final result = await _methodChannel.invokeMethod<Map<String, dynamic>>('getNetworkInfo');
    if (result == null) {
      throw Exception('Failed to get network info');
    }
    
    return NetworkInfo.fromJson(result);
  }
  
  // 获取存储信息
  Future<StorageInfo> getStorageInfo() async {
    final result = await _methodChannel.invokeMethod<Map<String, dynamic>>('getStorageInfo');
    if (result == null) {
      throw Exception('Failed to get storage info');
    }
    
    return StorageInfo.fromJson(result);
  }
  
  // 获取电池信息
  Future<BatteryInfo> getBatteryInfo() async {
    final result = await _methodChannel.invokeMethod<Map<String, dynamic>>('getBatteryInfo');
    if (result == null) {
      throw Exception('Failed to get battery info');
    }
    
    return BatteryInfo.fromJson(result);
  }
  
  // 监听电池状态变化
  Stream<BatteryInfo> get batteryStatusStream {
    return _eventChannel.listen<Map<String, dynamic>>()
        .map((data) => BatteryInfo.fromJson(data));
  }
  
  // 监听网络状态变化
  Stream<NetworkInfo> get networkStatusStream {
    return _eventChannel.listen<Map<String, dynamic>>()
        .where((data) => data['type'] == 'network')
        .map((data) => NetworkInfo.fromJson(data['data']));
  }
  
  // 获取系统性能信息
  Future<PerformanceInfo> getPerformanceInfo() async {
    final result = await _methodChannel.invokeAsyncMethod<Map<String, dynamic>>(
      'getPerformanceInfo',
      timeout: const Duration(seconds: 10),
    );
    
    return PerformanceInfo.fromJson(result);
  }
  
  // 执行系统诊断
  Future<DiagnosticResult> runDiagnostics() async {
    final result = await _methodChannel.invokeAsyncMethod<Map<String, dynamic>>(
      'runDiagnostics',
      timeout: const Duration(minutes: 2),
    );
    
    return DiagnosticResult.fromJson(result);
  }
}

// 设备信息数据模型
class DeviceInfo {
  final String deviceId;
  final String deviceName;
  final String model;
  final String manufacturer;
  final String operatingSystem;
  final String osVersion;
  final String platform;
  final bool isPhysicalDevice;
  final Map<String, dynamic> additionalInfo;
  
  const DeviceInfo({
    required this.deviceId,
    required this.deviceName,
    required this.model,
    required this.manufacturer,
    required this.operatingSystem,
    required this.osVersion,
    required this.platform,
    required this.isPhysicalDevice,
    this.additionalInfo = const {},
  });
  
  factory DeviceInfo.fromJson(Map<String, dynamic> json) {
    return DeviceInfo(
      deviceId: json['deviceId'] as String,
      deviceName: json['deviceName'] as String,
      model: json['model'] as String,
      manufacturer: json['manufacturer'] as String,
      operatingSystem: json['operatingSystem'] as String,
      osVersion: json['osVersion'] as String,
      platform: json['platform'] as String,
      isPhysicalDevice: json['isPhysicalDevice'] as bool,
      additionalInfo: json['additionalInfo'] as Map<String, dynamic>? ?? {},
    );
  }
  
  Map<String, dynamic> toJson() {
    return {
      'deviceId': deviceId,
      'deviceName': deviceName,
      'model': model,
      'manufacturer': manufacturer,
      'operatingSystem': operatingSystem,
      'osVersion': osVersion,
      'platform': platform,
      'isPhysicalDevice': isPhysicalDevice,
      'additionalInfo': additionalInfo,
    };
  }
}

// 硬件信息数据模型
class HardwareInfo {
  final String processor;
  final int coreCount;
  final String architecture;
  final int totalMemory;
  final int availableMemory;
  final String gpuInfo;
  final List<String> sensors;
  final Map<String, dynamic> additionalSpecs;
  
  const HardwareInfo({
    required this.processor,
    required this.coreCount,
    required this.architecture,
    required this.totalMemory,
    required this.availableMemory,
    required this.gpuInfo,
    required this.sensors,
    this.additionalSpecs = const {},
  });
  
  factory HardwareInfo.fromJson(Map<String, dynamic> json) {
    return HardwareInfo(
      processor: json['processor'] as String,
      coreCount: json['coreCount'] as int,
      architecture: json['architecture'] as String,
      totalMemory: json['totalMemory'] as int,
      availableMemory: json['availableMemory'] as int,
      gpuInfo: json['gpuInfo'] as String,
      sensors: List<String>.from(json['sensors'] as List),
      additionalSpecs: json['additionalSpecs'] as Map<String, dynamic>? ?? {},
    );
  }
  
  Map<String, dynamic> toJson() {
    return {
      'processor': processor,
      'coreCount': coreCount,
      'architecture': architecture,
      'totalMemory': totalMemory,
      'availableMemory': availableMemory,
      'gpuInfo': gpuInfo,
      'sensors': sensors,
      'additionalSpecs': additionalSpecs,
    };
  }
}

// 网络信息数据模型
class NetworkInfo {
  final String connectionType;
  final bool isConnected;
  final String? wifiName;
  final String? ipAddress;
  final String? macAddress;
  final int? signalStrength;
  final double? downloadSpeed;
  final double? uploadSpeed;
  final Map<String, dynamic> additionalInfo;
  
  const NetworkInfo({
    required this.connectionType,
    required this.isConnected,
    this.wifiName,
    this.ipAddress,
    this.macAddress,
    this.signalStrength,
    this.downloadSpeed,
    this.uploadSpeed,
    this.additionalInfo = const {},
  });
  
  factory NetworkInfo.fromJson(Map<String, dynamic> json) {
    return NetworkInfo(
      connectionType: json['connectionType'] as String,
      isConnected: json['isConnected'] as bool,
      wifiName: json['wifiName'] as String?,
      ipAddress: json['ipAddress'] as String?,
      macAddress: json['macAddress'] as String?,
      signalStrength: json['signalStrength'] as int?,
      downloadSpeed: (json['downloadSpeed'] as num?)?.toDouble(),
      uploadSpeed: (json['uploadSpeed'] as num?)?.toDouble(),
      additionalInfo: json['additionalInfo'] as Map<String, dynamic>? ?? {},
    );
  }
  
  Map<String, dynamic> toJson() {
    return {
      'connectionType': connectionType,
      'isConnected': isConnected,
      'wifiName': wifiName,
      'ipAddress': ipAddress,
      'macAddress': macAddress,
      'signalStrength': signalStrength,
      'downloadSpeed': downloadSpeed,
      'uploadSpeed': uploadSpeed,
      'additionalInfo': additionalInfo,
    };
  }
}

// 存储信息数据模型
class StorageInfo {
  final int totalSpace;
  final int freeSpace;
  final int usedSpace;
  final List<StorageDevice> devices;
  final Map<String, dynamic> additionalInfo;
  
  const StorageInfo({
    required this.totalSpace,
    required this.freeSpace,
    required this.usedSpace,
    required this.devices,
    this.additionalInfo = const {},
  });
  
  double get usagePercentage => (usedSpace / totalSpace) * 100;
  
  factory StorageInfo.fromJson(Map<String, dynamic> json) {
    return StorageInfo(
      totalSpace: json['totalSpace'] as int,
      freeSpace: json['freeSpace'] as int,
      usedSpace: json['usedSpace'] as int,
      devices: (json['devices'] as List)
          .map((device) => StorageDevice.fromJson(device as Map<String, dynamic>))
          .toList(),
      additionalInfo: json['additionalInfo'] as Map<String, dynamic>? ?? {},
    );
  }
  
  Map<String, dynamic> toJson() {
    return {
      'totalSpace': totalSpace,
      'freeSpace': freeSpace,
      'usedSpace': usedSpace,
      'devices': devices.map((device) => device.toJson()).toList(),
      'additionalInfo': additionalInfo,
    };
  }
}

class StorageDevice {
  final String name;
  final String type;
  final int capacity;
  final int freeSpace;
  final bool isRemovable;
  
  const StorageDevice({
    required this.name,
    required this.type,
    required this.capacity,
    required this.freeSpace,
    required this.isRemovable,
  });
  
  factory StorageDevice.fromJson(Map<String, dynamic> json) {
    return StorageDevice(
      name: json['name'] as String,
      type: json['type'] as String,
      capacity: json['capacity'] as int,
      freeSpace: json['freeSpace'] as int,
      isRemovable: json['isRemovable'] as bool,
    );
  }
  
  Map<String, dynamic> toJson() {
    return {
      'name': name,
      'type': type,
      'capacity': capacity,
      'freeSpace': freeSpace,
      'isRemovable': isRemovable,
    };
  }
}

// 电池信息数据模型
class BatteryInfo {
  final int level;
  final String status;
  final String health;
  final String technology;
  final double? temperature;
  final double? voltage;
  final bool isCharging;
  final int? timeToFull;
  final int? timeToEmpty;
  
  const BatteryInfo({
    required this.level,
    required this.status,
    required this.health,
    required this.technology,
    this.temperature,
    this.voltage,
    required this.isCharging,
    this.timeToFull,
    this.timeToEmpty,
  });
  
  factory BatteryInfo.fromJson(Map<String, dynamic> json) {
    return BatteryInfo(
      level: json['level'] as int,
      status: json['status'] as String,
      health: json['health'] as String,
      technology: json['technology'] as String,
      temperature: (json['temperature'] as num?)?.toDouble(),
      voltage: (json['voltage'] as num?)?.toDouble(),
      isCharging: json['isCharging'] as bool,
      timeToFull: json['timeToFull'] as int?,
      timeToEmpty: json['timeToEmpty'] as int?,
    );
  }
  
  Map<String, dynamic> toJson() {
    return {
      'level': level,
      'status': status,
      'health': health,
      'technology': technology,
      'temperature': temperature,
      'voltage': voltage,
      'isCharging': isCharging,
      'timeToFull': timeToFull,
      'timeToEmpty': timeToEmpty,
    };
  }
}

// 性能信息数据模型
class PerformanceInfo {
  final double cpuUsage;
  final double memoryUsage;
  final double diskUsage;
  final double networkUsage;
  final int runningProcesses;
  final List<ProcessInfo> topProcesses;
  final Map<String, dynamic> systemMetrics;
  
  const PerformanceInfo({
    required this.cpuUsage,
    required this.memoryUsage,
    required this.diskUsage,
    required this.networkUsage,
    required this.runningProcesses,
    required this.topProcesses,
    this.systemMetrics = const {},
  });
  
  factory PerformanceInfo.fromJson(Map<String, dynamic> json) {
    return PerformanceInfo(
      cpuUsage: (json['cpuUsage'] as num).toDouble(),
      memoryUsage: (json['memoryUsage'] as num).toDouble(),
      diskUsage: (json['diskUsage'] as num).toDouble(),
      networkUsage: (json['networkUsage'] as num).toDouble(),
      runningProcesses: json['runningProcesses'] as int,
      topProcesses: (json['topProcesses'] as List)
          .map((process) => ProcessInfo.fromJson(process as Map<String, dynamic>))
          .toList(),
      systemMetrics: json['systemMetrics'] as Map<String, dynamic>? ?? {},
    );
  }
  
  Map<String, dynamic> toJson() {
    return {
      'cpuUsage': cpuUsage,
      'memoryUsage': memoryUsage,
      'diskUsage': diskUsage,
      'networkUsage': networkUsage,
      'runningProcesses': runningProcesses,
      'topProcesses': topProcesses.map((process) => process.toJson()).toList(),
      'systemMetrics': systemMetrics,
    };
  }
}

class ProcessInfo {
  final String name;
  final int pid;
  final double cpuUsage;
  final int memoryUsage;
  
  const ProcessInfo({
    required this.name,
    required this.pid,
    required this.cpuUsage,
    required this.memoryUsage,
  });
  
  factory ProcessInfo.fromJson(Map<String, dynamic> json) {
    return ProcessInfo(
      name: json['name'] as String,
      pid: json['pid'] as int,
      cpuUsage: (json['cpuUsage'] as num).toDouble(),
      memoryUsage: json['memoryUsage'] as int,
    );
  }
  
  Map<String, dynamic> toJson() {
    return {
      'name': name,
      'pid': pid,
      'cpuUsage': cpuUsage,
      'memoryUsage': memoryUsage,
    };
  }
}

// 诊断结果数据模型
class DiagnosticResult {
  final bool overallHealth;
  final List<DiagnosticTest> tests;
  final List<String> recommendations;
  final Map<String, dynamic> detailedResults;
  
  const DiagnosticResult({
    required this.overallHealth,
    required this.tests,
    required this.recommendations,
    this.detailedResults = const {},
  });
  
  factory DiagnosticResult.fromJson(Map<String, dynamic> json) {
    return DiagnosticResult(
      overallHealth: json['overallHealth'] as bool,
      tests: (json['tests'] as List)
          .map((test) => DiagnosticTest.fromJson(test as Map<String, dynamic>))
          .toList(),
      recommendations: List<String>.from(json['recommendations'] as List),
      detailedResults: json['detailedResults'] as Map<String, dynamic>? ?? {},
    );
  }
  
  Map<String, dynamic> toJson() {
    return {
      'overallHealth': overallHealth,
      'tests': tests.map((test) => test.toJson()).toList(),
      'recommendations': recommendations,
      'detailedResults': detailedResults,
    };
  }
}

class DiagnosticTest {
  final String name;
  final bool passed;
  final String? message;
  final Map<String, dynamic> details;
  
  const DiagnosticTest({
    required this.name,
    required this.passed,
    this.message,
    this.details = const {},
  });
  
  factory DiagnosticTest.fromJson(Map<String, dynamic> json) {
    return DiagnosticTest(
      name: json['name'] as String,
      passed: json['passed'] as bool,
      message: json['message'] as String?,
      details: json['details'] as Map<String, dynamic>? ?? {},
    );
  }
  
  Map<String, dynamic> toJson() {
    return {
      'name': name,
      'passed': passed,
      'message': message,
      'details': details,
    };
  }
}
```

### 插件使用示例

```dart
// 插件使用示例
class DeviceInfoScreen extends StatefulWidget {
  const DeviceInfoScreen({Key? key}) : super(key: key);
  
  @override
  State<DeviceInfoScreen> createState() => _DeviceInfoScreenState();
}

class _DeviceInfoScreenState extends State<DeviceInfoScreen> {
  late final DeviceInfoPlugin _plugin;
  DeviceInfo? _deviceInfo;
  HardwareInfo? _hardwareInfo;
  NetworkInfo? _networkInfo;
  StorageInfo? _storageInfo;
  BatteryInfo? _batteryInfo;
  PerformanceInfo? _performanceInfo;
  bool _isLoading = true;
  String? _error;
  
  StreamSubscription? _batterySubscription;
  StreamSubscription? _networkSubscription;
  
  @override
  void initState() {
    super.initState();
    _initializePlugin();
  }
  
  @override
  void dispose() {
    _batterySubscription?.cancel();
    _networkSubscription?.cancel();
    super.dispose();
  }
  
  Future<void> _initializePlugin() async {
    try {
      _plugin = DeviceInfoPlugin();
      await _plugin.initialize();
      
      // 监听电池状态变化
      _batterySubscription = _plugin.batteryStatusStream.listen(
        (batteryInfo) {
          if (mounted) {
            setState(() {
              _batteryInfo = batteryInfo;
            });
          }
        },
        onError: (error) {
          debugPrint('Battery status stream error: $error');
        },
      );
      
      // 监听网络状态变化
      _networkSubscription = _plugin.networkStatusStream.listen(
        (networkInfo) {
          if (mounted) {
            setState(() {
              _networkInfo = networkInfo;
            });
          }
        },
        onError: (error) {
          debugPrint('Network status stream error: $error');
        },
      );
      
      await _loadDeviceInfo();
    } catch (e) {
      setState(() {
        _error = e.toString();
        _isLoading = false;
      });
    }
  }
  
  Future<void> _loadDeviceInfo() async {
    try {
      setState(() {
        _isLoading = true;
        _error = null;
      });
      
      final results = await Future.wait([
        _plugin.getDeviceInfo(),
        _plugin.getHardwareInfo(),
        _plugin.getNetworkInfo(),
        _plugin.getStorageInfo(),
        _plugin.getBatteryInfo(),
        _plugin.getPerformanceInfo(),
      ]);
      
      setState(() {
        _deviceInfo = results[0] as DeviceInfo;
        _hardwareInfo = results[1] as HardwareInfo;
        _networkInfo = results[2] as NetworkInfo;
        _storageInfo = results[3] as StorageInfo;
        _batteryInfo = results[4] as BatteryInfo;
        _performanceInfo = results[5] as PerformanceInfo;
        _isLoading = false;
      });
    } catch (e) {
      setState(() {
        _error = e.toString();
        _isLoading = false;
      });
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Device Information'),
        actions: [
          IconButton(
            icon: const Icon(Icons.refresh),
            onPressed: _loadDeviceInfo,
          ),
        ],
      ),
      body: _buildBody(),
    );
  }
  
  Widget _buildBody() {
    if (_isLoading) {
      return const Center(
        child: CircularProgressIndicator(),
      );
    }
    
    if (_error != null) {
      return Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Icon(
              Icons.error_outline,
              size: 64,
              color: Theme.of(context).colorScheme.error,
            ),
            const SizedBox(height: 16),
            Text(
              'Error loading device information',
              style: Theme.of(context).textTheme.headlineSmall,
            ),
            const SizedBox(height: 8),
            Text(
              _error!,
              style: Theme.of(context).textTheme.bodyMedium,
              textAlign: TextAlign.center,
            ),
            const SizedBox(height: 16),
            ElevatedButton(
              onPressed: _loadDeviceInfo,
              child: const Text('Retry'),
            ),
          ],
        ),
      );
    }
    
    return SingleChildScrollView(
      padding: const EdgeInsets.all(16),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          if (_deviceInfo != null) _buildDeviceInfoCard(),
          const SizedBox(height: 16),
          if (_hardwareInfo != null) _buildHardwareInfoCard(),
          const SizedBox(height: 16),
          if (_networkInfo != null) _buildNetworkInfoCard(),
          const SizedBox(height: 16),
          if (_storageInfo != null) _buildStorageInfoCard(),
          const SizedBox(height: 16),
          if (_batteryInfo != null) _buildBatteryInfoCard(),
          const SizedBox(height: 16),
          if (_performanceInfo != null) _buildPerformanceInfoCard(),
        ],
      ),
    );
  }
  
  Widget _buildDeviceInfoCard() {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              'Device Information',
              style: Theme.of(context).textTheme.headlineSmall,
            ),
            const SizedBox(height: 16),
            _buildInfoRow('Device Name', _deviceInfo!.deviceName),
            _buildInfoRow('Model', _deviceInfo!.model),
            _buildInfoRow('Manufacturer', _deviceInfo!.manufacturer),
            _buildInfoRow('Operating System', _deviceInfo!.operatingSystem),
            _buildInfoRow('OS Version', _deviceInfo!.osVersion),
            _buildInfoRow('Platform', _deviceInfo!.platform),
            _buildInfoRow('Physical Device', _deviceInfo!.isPhysicalDevice ? 'Yes' : 'No'),
          ],
        ),
      ),
    );
  }
  
  Widget _buildHardwareInfoCard() {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              'Hardware Information',
              style: Theme.of(context).textTheme.headlineSmall,
            ),
            const SizedBox(height: 16),
            _buildInfoRow('Processor', _hardwareInfo!.processor),
            _buildInfoRow('Core Count', _hardwareInfo!.coreCount.toString()),
            _buildInfoRow('Architecture', _hardwareInfo!.architecture),
            _buildInfoRow('Total Memory', '${(_hardwareInfo!.totalMemory / 1024 / 1024 / 1024).toStringAsFixed(1)} GB'),
            _buildInfoRow('Available Memory', '${(_hardwareInfo!.availableMemory / 1024 / 1024 / 1024).toStringAsFixed(1)} GB'),
            _buildInfoRow('GPU', _hardwareInfo!.gpuInfo),
            _buildInfoRow('Sensors', _hardwareInfo!.sensors.join(', ')),
          ],
        ),
      ),
    );
  }
  
  Widget _buildNetworkInfoCard() {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(
              children: [
                Text(
                  'Network Information',
                  style: Theme.of(context).textTheme.headlineSmall,
                ),
                const Spacer(),
                Icon(
                  _networkInfo!.isConnected ? Icons.wifi : Icons.wifi_off,
                  color: _networkInfo!.isConnected ? Colors.green : Colors.red,
                ),
              ],
            ),
            const SizedBox(height: 16),
            _buildInfoRow('Connection Type', _networkInfo!.connectionType),
            _buildInfoRow('Status', _networkInfo!.isConnected ? 'Connected' : 'Disconnected'),
            if (_networkInfo!.wifiName != null)
              _buildInfoRow('WiFi Name', _networkInfo!.wifiName!),
            if (_networkInfo!.ipAddress != null)
              _buildInfoRow('IP Address', _networkInfo!.ipAddress!),
            if (_networkInfo!.signalStrength != null)
              _buildInfoRow('Signal Strength', '${_networkInfo!.signalStrength}%'),
            if (_networkInfo!.downloadSpeed != null)
              _buildInfoRow('Download Speed', '${_networkInfo!.downloadSpeed!.toStringAsFixed(1)} Mbps'),
            if (_networkInfo!.uploadSpeed != null)
              _buildInfoRow('Upload Speed', '${_networkInfo!.uploadSpeed!.toStringAsFixed(1)} Mbps'),
          ],
        ),
      ),
    );
  }
  
  Widget _buildStorageInfoCard() {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              'Storage Information',
              style: Theme.of(context).textTheme.headlineSmall,
            ),
            const SizedBox(height: 16),
            _buildInfoRow('Total Space', '${(_storageInfo!.totalSpace / 1024 / 1024 / 1024).toStringAsFixed(1)} GB'),
            _buildInfoRow('Free Space', '${(_storageInfo!.freeSpace / 1024 / 1024 / 1024).toStringAsFixed(1)} GB'),
            _buildInfoRow('Used Space', '${(_storageInfo!.usedSpace / 1024 / 1024 / 1024).toStringAsFixed(1)} GB'),
            _buildInfoRow('Usage', '${_storageInfo!.usagePercentage.toStringAsFixed(1)}%'),
            const SizedBox(height: 8),
            LinearProgressIndicator(
              value: _storageInfo!.usagePercentage / 100,
              backgroundColor: Colors.grey[300],
              valueColor: AlwaysStoppedAnimation<Color>(
                _storageInfo!.usagePercentage > 80 ? Colors.red : Colors.blue,
              ),
            ),
          ],
        ),
      ),
    );
  }
  
  Widget _buildBatteryInfoCard() {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(
              children: [
                Text(
                  'Battery Information',
                  style: Theme.of(context).textTheme.headlineSmall,
                ),
                const Spacer(),
                Icon(
                  _batteryInfo!.isCharging ? Icons.battery_charging_full : Icons.battery_std,
                  color: _getBatteryColor(_batteryInfo!.level),
                ),
                Text(
                  '${_batteryInfo!.level}%',
                  style: Theme.of(context).textTheme.titleMedium,
                ),
              ],
            ),
            const SizedBox(height: 16),
            _buildInfoRow('Status', _batteryInfo!.status),
            _buildInfoRow('Health', _batteryInfo!.health),
            _buildInfoRow('Technology', _batteryInfo!.technology),
            _buildInfoRow('Charging', _batteryInfo!.isCharging ? 'Yes' : 'No'),
            if (_batteryInfo!.temperature != null)
              _buildInfoRow('Temperature', '${_batteryInfo!.temperature!.toStringAsFixed(1)}°C'),
            if (_batteryInfo!.voltage != null)
              _buildInfoRow('Voltage', '${_batteryInfo!.voltage!.toStringAsFixed(2)}V'),
            const SizedBox(height: 8),
            LinearProgressIndicator(
              value: _batteryInfo!.level / 100,
              backgroundColor: Colors.grey[300],
              valueColor: AlwaysStoppedAnimation<Color>(
                _getBatteryColor(_batteryInfo!.level),
              ),
            ),
          ],
        ),
      ),
    );
  }
  
  Widget _buildPerformanceInfoCard() {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              'Performance Information',
              style: Theme.of(context).textTheme.headlineSmall,
            ),
            const SizedBox(height: 16),
            _buildInfoRow('CPU Usage', '${_performanceInfo!.cpuUsage.toStringAsFixed(1)}%'),
            _buildInfoRow('Memory Usage', '${_performanceInfo!.memoryUsage.toStringAsFixed(1)}%'),
            _buildInfoRow('Disk Usage', '${_performanceInfo!.diskUsage.toStringAsFixed(1)}%'),
            _buildInfoRow('Network Usage', '${_performanceInfo!.networkUsage.toStringAsFixed(1)}%'),
            _buildInfoRow('Running Processes', _performanceInfo!.runningProcesses.toString()),
          ],
        ),
      ),
    );
  }
  
  Widget _buildInfoRow(String label, String value) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 4),
      child: Row(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          SizedBox(
            width: 120,
            child: Text(
              label,
              style: Theme.of(context).textTheme.bodyMedium?.copyWith(
                fontWeight: FontWeight.w500,
              ),
            ),
          ),
          Expanded(
            child: Text(
              value,
              style: Theme.of(context).textTheme.bodyMedium,
            ),
          ),
        ],
      ),
    );
  }
  
  Color _getBatteryColor(int level) {
    if (level > 50) return Colors.green;
    if (level > 20) return Colors.orange;
    return Colors.red;
  }
}
```

## 插件测试策略

### 单元测试

```dart
// test/device_info_plugin_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/mockito.dart';
import 'package:mockito/annotations.dart';
import 'package:device_info_plugin/device_info_plugin.dart';

@GenerateMocks([AdvancedMethodChannel])
import 'device_info_plugin_test.mocks.dart';

void main() {
  group('DeviceInfoPlugin', () {
    late DeviceInfoPlugin plugin;
    late MockAdvancedMethodChannel mockChannel;
    
    setUp(() {
      mockChannel = MockAdvancedMethodChannel();
      plugin = DeviceInfoPlugin();
      // 注入mock channel
    });
    
    group('initialization', () {
      test('should initialize successfully', () async {
        when(mockChannel.invokeMethod('isAvailable'))
            .thenAnswer((_) async => true);
        
        await plugin.initialize();
        
        expect(plugin.isInitialized, isTrue);
        verify(mockChannel.invokeMethod('isAvailable')).called(1);
      });
      
      test('should handle initialization failure', () async {
        when(mockChannel.invokeMethod('isAvailable'))
            .thenThrow(Exception('Platform not supported'));
        
        expect(
          () => plugin.initialize(),
          throwsException,
        );
      });
    });
    
    group('device info', () {
      test('should return device info successfully', () async {
        final mockDeviceInfo = {
          'deviceId': 'test-device-id',
          'deviceName': 'Test Device',
          'model': 'Test Model',
          'manufacturer': 'Test Manufacturer',
          'operatingSystem': 'Test OS',
          'osVersion': '1.0.0',
          'platform': 'test',
          'isPhysicalDevice': true,
          'additionalInfo': {},
        };
        
        when(mockChannel.invokeMethod<Map<String, dynamic>>('getDeviceInfo'))
            .thenAnswer((_) async => mockDeviceInfo);
        
        final result = await plugin.getDeviceInfo();
        
        expect(result.deviceId, equals('test-device-id'));
        expect(result.deviceName, equals('Test Device'));
        expect(result.isPhysicalDevice, isTrue);
        verify(mockChannel.invokeMethod('getDeviceInfo')).called(1);
      });
      
      test('should handle device info failure', () async {
        when(mockChannel.invokeMethod<Map<String, dynamic>>('getDeviceInfo'))
            .thenAnswer((_) async => null);
        
        expect(
          () => plugin.getDeviceInfo(),
          throwsException,
        );
      });
    });
    
    group('hardware info', () {
      test('should return hardware info successfully', () async {
        final mockHardwareInfo = {
          'processor': 'Test Processor',
          'coreCount': 8,
          'architecture': 'arm64',
          'totalMemory': 8589934592, // 8GB
          'availableMemory': 4294967296, // 4GB
          'gpuInfo': 'Test GPU',
          'sensors': ['accelerometer', 'gyroscope'],
          'additionalSpecs': {},
        };
        
        when(mockChannel.invokeMethod<Map<String, dynamic>>('getHardwareInfo'))
            .thenAnswer((_) async => mockHardwareInfo);
        
        final result = await plugin.getHardwareInfo();
        
        expect(result.processor, equals('Test Processor'));
        expect(result.coreCount, equals(8));
        expect(result.architecture, equals('arm64'));
        expect(result.sensors, contains('accelerometer'));
        verify(mockChannel.invokeMethod('getHardwareInfo')).called(1);
      });
    });
    
    group('battery info stream', () {
      test('should emit battery info updates', () async {
        final mockBatteryInfo = {
          'level': 75,
          'status': 'charging',
          'health': 'good',
          'technology': 'Li-ion',
          'isCharging': true,
        };
        
        // 模拟事件流
        final controller = StreamController<Map<String, dynamic>>();
        when(mockChannel.listen<Map<String, dynamic>>())
            .thenAnswer((_) => controller.stream);
        
        final stream = plugin.batteryStatusStream;
        
        // 发送测试数据
        controller.add(mockBatteryInfo);
        
        await expectLater(
          stream.take(1),
          emits(predicate<BatteryInfo>((info) => 
            info.level == 75 && 
            info.status == 'charging' &&
            info.isCharging == true
          )),
        );
        
        await controller.close();
      });
    });
    
    tearDown(() async {
      await plugin.dispose();
    });
  });
}

// 集成测试
// integration_test/plugin_integration_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:device_info_plugin_example/main.dart' as app;

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();
  
  group('DeviceInfoPlugin Integration Tests', () {
    testWidgets('should load device information successfully', (tester) async {
      app.main();
      await tester.pumpAndSettle();
      
      // 等待数据加载
      await tester.pump(const Duration(seconds: 3));
      
      // 验证设备信息卡片存在
      expect(find.text('Device Information'), findsOneWidget);
      expect(find.text('Hardware Information'), findsOneWidget);
      expect(find.text('Network Information'), findsOneWidget);
      expect(find.text('Storage Information'), findsOneWidget);
      expect(find.text('Battery Information'), findsOneWidget);
      
      // 验证刷新功能
      await tester.tap(find.byIcon(Icons.refresh));
      await tester.pumpAndSettle();
      
      // 验证数据重新加载
      expect(find.byType(CircularProgressIndicator), findsNothing);
    });
    
    testWidgets('should handle network status changes', (tester) async {
      app.main();
      await tester.pumpAndSettle();
      
      // 等待初始加载
      await tester.pump(const Duration(seconds: 2));
      
      // 查找网络状态图标
      final networkIcon = find.byIcon(Icons.wifi).or(find.byIcon(Icons.wifi_off));
      expect(networkIcon, findsOneWidget);
      
      // 模拟网络状态变化（这需要在原生端配合）
      // 实际测试中可能需要手动切换网络状态
    });
  });
}
```

## 平台特定实现

### Android平台实现

```kotlin
// android/src/main/kotlin/com/example/device_info/DeviceInfoPlugin.kt
package com.example.device_info

import android.content.Context
import android.content.Intent
import android.content.IntentFilter
import android.os.BatteryManager
import android.os.Build
import android.net.ConnectivityManager
import android.net.NetworkCapabilities
import android.telephony.TelephonyManager
import io.flutter.embedding.engine.plugins.FlutterPlugin
import io.flutter.plugin.common.MethodCall
import io.flutter.plugin.common.MethodChannel
import io.flutter.plugin.common.EventChannel
import kotlinx.coroutines.*
import java.io.File
import java.util.*

class DeviceInfoPlugin: FlutterPlugin, MethodChannel.MethodCallHandler, EventChannel.StreamHandler {
    private lateinit var context: Context
    private lateinit var methodChannel: MethodChannel
    private lateinit var eventChannel: EventChannel
    private var eventSink: EventChannel.EventSink? = null
    private val coroutineScope = CoroutineScope(Dispatchers.Main + SupervisorJob())
    
    companion object {
        private const val METHOD_CHANNEL = "com.example.device_info"
        private const val EVENT_CHANNEL = "com.example.device_info_events"
    }
    
    override fun onAttachedToEngine(flutterPluginBinding: FlutterPlugin.FlutterPluginBinding) {
        context = flutterPluginBinding.applicationContext
        
        methodChannel = MethodChannel(flutterPluginBinding.binaryMessenger, METHOD_CHANNEL)
        methodChannel.setMethodCallHandler(this)
        
        eventChannel = EventChannel(flutterPluginBinding.binaryMessenger, EVENT_CHANNEL)
        eventChannel.setStreamHandler(this)
    }
    
    override fun onDetachedFromEngine(binding: FlutterPlugin.FlutterPluginBinding) {
        methodChannel.setMethodCallHandler(null)
        eventChannel.setStreamHandler(null)
        coroutineScope.cancel()
    }
    
    override fun onMethodCall(call: MethodCall, result: MethodChannel.Result) {
        when (call.method) {
            "isAvailable" -> {
                result.success(true)
            }
            "getDeviceInfo" -> {
                getDeviceInfo(result)
            }
            "getHardwareInfo" -> {
                getHardwareInfo(result)
            }
            "getNetworkInfo" -> {
                getNetworkInfo(result)
            }
            "getStorageInfo" -> {
                getStorageInfo(result)
            }
            "getBatteryInfo" -> {
                getBatteryInfo(result)
            }
            "getPerformanceInfo_async" -> {
                getPerformanceInfoAsync(call, result)
            }
            "runDiagnostics_async" -> {
                runDiagnosticsAsync(call, result)
            }
            else -> {
                result.notImplemented()
            }
        }
    }
    
    private fun getDeviceInfo(result: MethodChannel.Result) {
        try {
            val deviceInfo = mapOf(
                "deviceId" to getDeviceId(),
                "deviceName" to getDeviceName(),
                "model" to Build.MODEL,
                "manufacturer" to Build.MANUFACTURER,
                "operatingSystem" to "Android",
                "osVersion" to Build.VERSION.RELEASE,
                "platform" to "android",
                "isPhysicalDevice" to !isEmulator(),
                "additionalInfo" to mapOf(
                    "sdkInt" to Build.VERSION.SDK_INT,
                    "brand" to Build.BRAND,
                    "product" to Build.PRODUCT,
                    "hardware" to Build.HARDWARE,
                    "bootloader" to Build.BOOTLOADER,
                    "fingerprint" to Build.FINGERPRINT
                )
            )
            result.success(deviceInfo)
        } catch (e: Exception) {
            result.error("DEVICE_INFO_ERROR", e.message, null)
        }
    }
    
    private fun getHardwareInfo(result: MethodChannel.Result) {
        try {
            val runtime = Runtime.getRuntime()
            val hardwareInfo = mapOf(
                "processor" to getProcessorInfo(),
                "coreCount" to runtime.availableProcessors(),
                "architecture" to System.getProperty("os.arch") ?: "unknown",
                "totalMemory" to getTotalMemory(),
                "availableMemory" to getAvailableMemory(),
                "gpuInfo" to getGpuInfo(),
                "sensors" to getSensorList(),
                "additionalSpecs" to mapOf(
                    "supportedAbis" to Build.SUPPORTED_ABIS.toList(),
                    "supported32BitAbis" to Build.SUPPORTED_32_BIT_ABIS.toList(),
                    "supported64BitAbis" to Build.SUPPORTED_64_BIT_ABIS.toList()
                )
            )
            result.success(hardwareInfo)
        } catch (e: Exception) {
            result.error("HARDWARE_INFO_ERROR", e.message, null)
        }
    }
    
    private fun getNetworkInfo(result: MethodChannel.Result) {
        try {
            val connectivityManager = context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
            val networkInfo = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
                val network = connectivityManager.activeNetwork
                val capabilities = connectivityManager.getNetworkCapabilities(network)
                
                mapOf(
                    "connectionType" to getConnectionType(capabilities),
                    "isConnected" to (network != null && capabilities != null),
                    "wifiName" to getWifiName(),
                    "ipAddress" to getIpAddress(),
                    "macAddress" to getMacAddress(),
                    "signalStrength" to getSignalStrength(),
                    "downloadSpeed" to getDownloadSpeed(capabilities),
                    "uploadSpeed" to getUploadSpeed(capabilities),
                    "additionalInfo" to mapOf(
                        "networkId" to network?.toString(),
                        "transportInfo" to getTransportInfo(capabilities)
                    )
                )
            } else {
                @Suppress("DEPRECATION")
                val activeNetworkInfo = connectivityManager.activeNetworkInfo
                mapOf(
                    "connectionType" to (activeNetworkInfo?.typeName ?: "unknown"),
                    "isConnected" to (activeNetworkInfo?.isConnected == true),
                    "wifiName" to getWifiName(),
                    "ipAddress" to getIpAddress(),
                    "macAddress" to getMacAddress(),
                    "signalStrength" to getSignalStrength(),
                    "downloadSpeed" to null,
                    "uploadSpeed" to null,
                    "additionalInfo" to emptyMap<String, Any>()
                )
            }
            result.success(networkInfo)
        } catch (e: Exception) {
            result.error("NETWORK_INFO_ERROR", e.message, null)
        }
    }
    
    private fun getStorageInfo(result: MethodChannel.Result) {
        try {
            val internalStorage = context.filesDir
            val totalSpace = internalStorage.totalSpace
            val freeSpace = internalStorage.freeSpace
            val usedSpace = totalSpace - freeSpace
            
            val storageInfo = mapOf(
                "totalSpace" to totalSpace,
                "freeSpace" to freeSpace,
                "usedSpace" to usedSpace,
                "devices" to getStorageDevices(),
                "additionalInfo" to mapOf(
                    "internalStoragePath" to internalStorage.absolutePath,
                    "externalStorageState" to android.os.Environment.getExternalStorageState()
                )
            )
            result.success(storageInfo)
        } catch (e: Exception) {
            result.error("STORAGE_INFO_ERROR", e.message, null)
        }
    }
    
    private fun getBatteryInfo(result: MethodChannel.Result) {
        try {
            val batteryIntent = context.registerReceiver(null, IntentFilter(Intent.ACTION_BATTERY_CHANGED))
            val batteryInfo = if (batteryIntent != null) {
                val level = batteryIntent.getIntExtra(BatteryManager.EXTRA_LEVEL, -1)
                val scale = batteryIntent.getIntExtra(BatteryManager.EXTRA_SCALE, -1)
                val status = batteryIntent.getIntExtra(BatteryManager.EXTRA_STATUS, -1)
                val health = batteryIntent.getIntExtra(BatteryManager.EXTRA_HEALTH, -1)
                val technology = batteryIntent.getStringExtra(BatteryManager.EXTRA_TECHNOLOGY)
                val temperature = batteryIntent.getIntExtra(BatteryManager.EXTRA_TEMPERATURE, -1)
                val voltage = batteryIntent.getIntExtra(BatteryManager.EXTRA_VOLTAGE, -1)
                
                val batteryLevel = if (level != -1 && scale != -1) {
                    (level * 100 / scale.toFloat()).toInt()
                } else {
                    -1
                }
                
                mapOf(
                    "level" to batteryLevel,
                    "status" to getBatteryStatus(status),
                    "health" to getBatteryHealth(health),
                    "technology" to (technology ?: "unknown"),
                    "temperature" to if (temperature != -1) temperature / 10.0 else null,
                    "voltage" to if (voltage != -1) voltage / 1000.0 else null,
                    "isCharging" to (status == BatteryManager.BATTERY_STATUS_CHARGING),
                    "timeToFull" to null, // Android doesn't provide this directly
                    "timeToEmpty" to null // Android doesn't provide this directly
                )
            } else {
                mapOf(
                    "level" to -1,
                    "status" to "unknown",
                    "health" to "unknown",
                    "technology" to "unknown",
                    "temperature" to null,
                    "voltage" to null,
                    "isCharging" to false,
                    "timeToFull" to null,
                    "timeToEmpty" to null
                )
            }
            result.success(batteryInfo)
        } catch (e: Exception) {
            result.error("BATTERY_INFO_ERROR", e.message, null)
        }
    }
    
    private fun getPerformanceInfoAsync(call: MethodCall, result: MethodChannel.Result) {
        val callId = call.argument<String>("callId")
        
        coroutineScope.launch {
            try {
                val performanceInfo = withContext(Dispatchers.IO) {
                    // 模拟性能数据收集（实际实现会更复杂）
                    delay(2000) // 模拟耗时操作
                    
                    mapOf(
                        "cpuUsage" to getCpuUsage(),
                        "memoryUsage" to getMemoryUsage(),
                        "diskUsage" to getDiskUsage(),
                        "networkUsage" to getNetworkUsage(),
                        "runningProcesses" to getRunningProcessCount(),
                        "topProcesses" to getTopProcesses(),
                        "systemMetrics" to getSystemMetrics()
                    )
                }
                
                // 通过回调返回结果
                methodChannel.invokeMethod("callback_getPerformanceInfo", mapOf(
                    "callId" to callId,
                    "result" to performanceInfo
                ))
                
                result.success(null)
            } catch (e: Exception) {
                methodChannel.invokeMethod("callback_getPerformanceInfo", mapOf(
                    "callId" to callId,
                    "error" to mapOf(
                        "code" to "PERFORMANCE_INFO_ERROR",
                        "message" to e.message,
                        "details" to null
                    )
                ))
                result.success(null)
            }
        }
    }
    
    // 事件流处理
    override fun onListen(arguments: Any?, events: EventChannel.EventSink?) {
        eventSink = events
        startMonitoring()
    }
    
    override fun onCancel(arguments: Any?) {
        eventSink = null
        stopMonitoring()
    }
    
    private fun startMonitoring() {
        coroutineScope.launch {
            while (eventSink != null) {
                try {
                    // 定期发送电池状态更新
                    val batteryInfo = getCurrentBatteryInfo()
                    eventSink?.success(batteryInfo)
                    
                    delay(5000) // 每5秒更新一次
                } catch (e: Exception) {
                    eventSink?.error("MONITORING_ERROR", e.message, null)
                    break
                }
            }
        }
    }
    
    private fun stopMonitoring() {
        // 停止监控逻辑
    }
    
    // 辅助方法实现
    private fun getDeviceId(): String {
        return android.provider.Settings.Secure.getString(
            context.contentResolver,
            android.provider.Settings.Secure.ANDROID_ID
        ) ?: "unknown"
    }
    
    private fun getDeviceName(): String {
        return android.provider.Settings.Global.getString(
            context.contentResolver,
            "device_name"
        ) ?: Build.MODEL
    }
    
    private fun isEmulator(): Boolean {
        return (Build.FINGERPRINT.startsWith("generic") ||
                Build.FINGERPRINT.startsWith("unknown") ||
                Build.MODEL.contains("google_sdk") ||
                Build.MODEL.contains("Emulator") ||
                Build.MODEL.contains("Android SDK built for x86") ||
                Build.MANUFACTURER.contains("Genymotion") ||
                Build.BRAND.startsWith("generic") && Build.DEVICE.startsWith("generic") ||
                "google_sdk" == Build.PRODUCT)
    }
    
    private fun getProcessorInfo(): String {
        return try {
            val cpuInfo = File("/proc/cpuinfo").readText()
            val lines = cpuInfo.split("\n")
            val modelLine = lines.find { it.startsWith("model name") }
            modelLine?.split(":")?.get(1)?.trim() ?: "Unknown Processor"
        } catch (e: Exception) {
            "Unknown Processor"
        }
    }
    
    private fun getTotalMemory(): Long {
        return try {
            val memInfo = File("/proc/meminfo").readText()
            val totalLine = memInfo.split("\n").find { it.startsWith("MemTotal:") }
            val totalKb = totalLine?.split("\s+")?.get(1)?.toLong() ?: 0
            totalKb * 1024 // Convert to bytes
        } catch (e: Exception) {
            0L
        }
    }
    
    private fun getAvailableMemory(): Long {
        val activityManager = context.getSystemService(Context.ACTIVITY_SERVICE) as android.app.ActivityManager
        val memInfo = android.app.ActivityManager.MemoryInfo()
        activityManager.getMemoryInfo(memInfo)
        return memInfo.availMem
    }
    
    private fun getGpuInfo(): String {
        return try {
            val glRenderer = android.opengl.GLES20.glGetString(android.opengl.GLES20.GL_RENDERER)
            glRenderer ?: "Unknown GPU"
        } catch (e: Exception) {
            "Unknown GPU"
        }
    }
    
    private fun getSensorList(): List<String> {
        val sensorManager = context.getSystemService(Context.SENSOR_SERVICE) as android.hardware.SensorManager
        return sensorManager.getSensorList(android.hardware.Sensor.TYPE_ALL)
            .map { it.name }
    }
    
    // 其他辅助方法的实现...
    private fun getConnectionType(capabilities: NetworkCapabilities?): String {
        return when {
            capabilities?.hasTransport(NetworkCapabilities.TRANSPORT_WIFI) == true -> "wifi"
            capabilities?.hasTransport(NetworkCapabilities.TRANSPORT_CELLULAR) == true -> "cellular"
            capabilities?.hasTransport(NetworkCapabilities.TRANSPORT_ETHERNET) == true -> "ethernet"
            else -> "unknown"
        }
    }
    
    private fun getWifiName(): String? {
        // 实现获取WiFi名称的逻辑
        return null
    }
    
    private fun getIpAddress(): String? {
        // 实现获取IP地址的逻辑
        return null
    }
    
    private fun getMacAddress(): String? {
        // 实现获取MAC地址的逻辑
        return null
    }
    
    private fun getSignalStrength(): Int? {
        // 实现获取信号强度的逻辑
        return null
    }
    
    private fun getDownloadSpeed(capabilities: NetworkCapabilities?): Double? {
        return capabilities?.linkDownstreamBandwidthKbps?.toDouble()?.div(1000)
    }
    
    private fun getUploadSpeed(capabilities: NetworkCapabilities?): Double? {
        return capabilities?.linkUpstreamBandwidthKbps?.toDouble()?.div(1000)
    }
    
    private fun getTransportInfo(capabilities: NetworkCapabilities?): String {
        return capabilities?.toString() ?: "unknown"
    }
    
    private fun getStorageDevices(): List<Map<String, Any>> {
        // 实现获取存储设备列表的逻辑
        return emptyList()
    }
    
    private fun getBatteryStatus(status: Int): String {
        return when (status) {
            BatteryManager.BATTERY_STATUS_CHARGING -> "charging"
            BatteryManager.BATTERY_STATUS_DISCHARGING -> "discharging"
            BatteryManager.BATTERY_STATUS_FULL -> "full"
            BatteryManager.BATTERY_STATUS_NOT_CHARGING -> "not_charging"
            BatteryManager.BATTERY_STATUS_UNKNOWN -> "unknown"
            else -> "unknown"
        }
    }
    
    private fun getBatteryHealth(health: Int): String {
        return when (health) {
            BatteryManager.BATTERY_HEALTH_GOOD -> "good"
            BatteryManager.BATTERY_HEALTH_OVERHEAT -> "overheat"
            BatteryManager.BATTERY_HEALTH_DEAD -> "dead"
            BatteryManager.BATTERY_HEALTH_OVER_VOLTAGE -> "over_voltage"
            BatteryManager.BATTERY_HEALTH_UNSPECIFIED_FAILURE -> "unspecified_failure"
            BatteryManager.BATTERY_HEALTH_COLD -> "cold"
            else -> "unknown"
        }
    }
    
    private fun getCpuUsage(): Double {
        // 实现CPU使用率获取逻辑
        return 0.0
    }
    
    private fun getMemoryUsage(): Double {
        // 实现内存使用率获取逻辑
        return 0.0
    }
    
    private fun getDiskUsage(): Double {
        // 实现磁盘使用率获取逻辑
        return 0.0
    }
    
    private fun getNetworkUsage(): Double {
        // 实现网络使用率获取逻辑
        return 0.0
    }
    
    private fun getRunningProcessCount(): Int {
        // 实现获取运行进程数的逻辑
        return 0
    }
    
    private fun getTopProcesses(): List<Map<String, Any>> {
        // 实现获取顶级进程列表的逻辑
        return emptyList()
    }
    
    private fun getSystemMetrics(): Map<String, Any> {
        // 实现获取系统指标的逻辑
        return emptyMap()
    }
    
    private fun getCurrentBatteryInfo(): Map<String, Any> {
        // 实现获取当前电池信息的逻辑
        return emptyMap()
    }
    
    private fun runDiagnosticsAsync(call: MethodCall, result: MethodChannel.Result) {
        // 实现异步诊断逻辑
        result.success(null)
    }
}
```

### iOS平台实现

```swift
// ios/Classes/DeviceInfoPlugin.swift
import Flutter
import UIKit
import SystemConfiguration
import CoreTelephony

public class DeviceInfoPlugin: NSObject, FlutterPlugin, FlutterStreamHandler {
    private var eventSink: FlutterEventSink?
    private var monitoringTimer: Timer?
    
    public static func register(with registrar: FlutterPluginRegistrar) {
        let methodChannel = FlutterMethodChannel(name: "com.example.device_info", binaryMessenger: registrar.messenger())
        let eventChannel = FlutterEventChannel(name: "com.example.device_info_events", binaryMessenger: registrar.messenger())
        
        let instance = DeviceInfoPlugin()
        registrar.addMethodCallDelegate(instance, channel: methodChannel)
        eventChannel.setStreamHandler(instance)
    }
    
    public func handle(_ call: FlutterMethodCall, result: @escaping FlutterResult) {
        switch call.method {
        case "isAvailable":
            result(true)
        case "getDeviceInfo":
            getDeviceInfo(result: result)
        case "getHardwareInfo":
            getHardwareInfo(result: result)
        case "getNetworkInfo":
            getNetworkInfo(result: result)
        case "getStorageInfo":
            getStorageInfo(result: result)
        case "getBatteryInfo":
            getBatteryInfo(result: result)
        case "getPerformanceInfo_async":
            getPerformanceInfoAsync(call: call, result: result)
        case "runDiagnostics_async":
            runDiagnosticsAsync(call: call, result: result)
        default:
            result(FlutterMethodNotImplemented)
        }
    }
    
    private func getDeviceInfo(result: @escaping FlutterResult) {
        let device = UIDevice.current
        
        let deviceInfo: [String: Any] = [
            "deviceId": getDeviceId(),
            "deviceName": device.name,
            "model": device.model,
            "manufacturer": "Apple",
            "operatingSystem": device.systemName,
            "osVersion": device.systemVersion,
            "platform": "ios",
            "isPhysicalDevice": !isSimulator(),
            "additionalInfo": [
                "localizedModel": device.localizedModel,
                "systemVersion": device.systemVersion,
                "identifierForVendor": device.identifierForVendor?.uuidString ?? "unknown"
            ]
        ]
        
        result(deviceInfo)
    }
    
    private func getHardwareInfo(result: @escaping FlutterResult) {
        let hardwareInfo: [String: Any] = [
            "processor": getProcessorInfo(),
            "coreCount": ProcessInfo.processInfo.processorCount,
            "architecture": getArchitecture(),
            "totalMemory": ProcessInfo.processInfo.physicalMemory,
            "availableMemory": getAvailableMemory(),
            "gpuInfo": getGpuInfo(),
            "sensors": getSensorList(),
            "additionalSpecs": [
                "thermalState": ProcessInfo.processInfo.thermalState.rawValue,
                "lowPowerModeEnabled": ProcessInfo.processInfo.isLowPowerModeEnabled
            ]
        ]
        
        result(hardwareInfo)
    }
    
    private func getNetworkInfo(result: @escaping FlutterResult) {
        let networkInfo: [String: Any] = [
            "connectionType": getConnectionType(),
            "isConnected": isNetworkConnected(),
            "wifiName": getWifiName(),
            "ipAddress": getIpAddress(),
            "macAddress": getMacAddress(),
            "signalStrength": getSignalStrength(),
            "downloadSpeed": nil,
            "uploadSpeed": nil,
            "additionalInfo": [:]
        ]
        
        result(networkInfo)
    }
    
    private func getStorageInfo(result: @escaping FlutterResult) {
        let fileManager = FileManager.default
        
        do {
            let systemAttributes = try fileManager.attributesOfFileSystem(forPath: NSHomeDirectory())
            let totalSpace = systemAttributes[.systemSize] as? Int64 ?? 0
            let freeSpace = systemAttributes[.systemFreeSize] as? Int64 ?? 0
            let usedSpace = totalSpace - freeSpace
            
            let storageInfo: [String: Any] = [
                "totalSpace": totalSpace,
                "freeSpace": freeSpace,
                "usedSpace": usedSpace,
                "devices": getStorageDevices(),
                "additionalInfo": [
                    "homeDirectory": NSHomeDirectory(),
                    "documentsDirectory": getDocumentsDirectory()
                ]
            ]
            
            result(storageInfo)
        } catch {
            result(FlutterError(code: "STORAGE_INFO_ERROR", message: error.localizedDescription, details: nil))
        }
    }
    
    private func getBatteryInfo(result: @escaping FlutterResult) {
        UIDevice.current.isBatteryMonitoringEnabled = true
        let device = UIDevice.current
        
        let batteryInfo: [String: Any] = [
            "level": Int(device.batteryLevel * 100),
            "status": getBatteryStatus(device.batteryState),
            "health": "good", // iOS doesn't provide battery health directly
            "technology": "Li-ion",
            "temperature": nil, // iOS doesn't provide battery temperature
            "voltage": nil, // iOS doesn't provide battery voltage
            "isCharging": device.batteryState == .charging,
            "timeToFull": nil,
            "timeToEmpty": nil
        ]
        
        result(batteryInfo)
    }
    
    private func getPerformanceInfoAsync(call: FlutterMethodCall, result: @escaping FlutterResult) {
        guard let args = call.arguments as? [String: Any],
              let callId = args["callId"] as? String else {
            result(FlutterError(code: "INVALID_ARGUMENTS", message: "Missing callId", details: nil))
            return
        }
        
        DispatchQueue.global(qos: .background).async {
            // 模拟性能数据收集
            Thread.sleep(forTimeInterval: 2.0)
            
            let performanceInfo: [String: Any] = [
                "cpuUsage": self.getCpuUsage(),
                "memoryUsage": self.getMemoryUsage(),
                "diskUsage": self.getDiskUsage(),
                "networkUsage": self.getNetworkUsage(),
                "runningProcesses": self.getRunningProcessCount(),
                "topProcesses": self.getTopProcesses(),
                "systemMetrics": self.getSystemMetrics()
            ]
            
            DispatchQueue.main.async {
                // 通过回调返回结果
                if let methodChannel = self.getMethodChannel() {
                    methodChannel.invokeMethod("callback_getPerformanceInfo", arguments: [
                        "callId": callId,
                        "result": performanceInfo
                    ])
                }
            }
        }
        
        result(nil)
    }
    
    // MARK: - FlutterStreamHandler
    
    public func onListen(withArguments arguments: Any?, eventSink events: @escaping FlutterEventSink) -> FlutterError? {
        eventSink = events
        startMonitoring()
        return nil
    }
    
    public func onCancel(withArguments arguments: Any?) -> FlutterError? {
        eventSink = nil
        stopMonitoring()
        return nil
    }
    
    private func startMonitoring() {
        monitoringTimer = Timer.scheduledTimer(withTimeInterval: 5.0, repeats: true) { _ in
            self.sendBatteryUpdate()
        }
    }
    
    private func stopMonitoring() {
        monitoringTimer?.invalidate()
        monitoringTimer = nil
    }
    
    private func sendBatteryUpdate() {
        UIDevice.current.isBatteryMonitoringEnabled = true
        let device = UIDevice.current
        
        let batteryInfo: [String: Any] = [
            "level": Int(device.batteryLevel * 100),
            "status": getBatteryStatus(device.batteryState),
            "health": "good",
            "technology": "Li-ion",
            "isCharging": device.batteryState == .charging
        ]
        
        eventSink?(batteryInfo)
    }
    
    // MARK: - Helper Methods
    
    private func getDeviceId() -> String {
        return UIDevice.current.identifierForVendor?.uuidString ?? "unknown"
    }
    
    private func isSimulator() -> Bool {
        #if targetEnvironment(simulator)
        return true
        #else
        return false
        #endif
    }
    
    private func getProcessorInfo() -> String {
        var systemInfo = utsname()
        uname(&systemInfo)
        let machineMirror = Mirror(reflecting: systemInfo.machine)
        let identifier = machineMirror.children.reduce("") { identifier, element in
            guard let value = element.value as? Int8, value != 0 else { return identifier }
            return identifier + String(UnicodeScalar(UInt8(value))!)
        }
        return identifier
    }
    
    private func getArchitecture() -> String {
        #if arch(arm64)
        return "arm64"
        #elseif arch(x86_64)
        return "x86_64"
        #else
        return "unknown"
        #endif
    }
    
    private func getAvailableMemory() -> Int64 {
        var info = mach_task_basic_info()
        var count = mach_msg_type_number_t(MemoryLayout<mach_task_basic_info>.size)/4
        
        let kerr: kern_return_t = withUnsafeMutablePointer(to: &info) {
            $0.withMemoryRebound(to: integer_t.self, capacity: 1) {
                task_info(mach_task_self_,
                         task_flavor_t(MACH_TASK_BASIC_INFO),
                         $0,
                         &count)
            }
        }
        
        if kerr == KERN_SUCCESS {
            return Int64(info.resident_size)
        } else {
            return 0
        }
    }
    
    private func getGpuInfo() -> String {
        // iOS doesn't provide direct GPU information
        return "Apple GPU"
    }
    
    private func getSensorList() -> [String] {
        // iOS doesn't provide a direct way to list all sensors
        return ["accelerometer", "gyroscope", "magnetometer", "proximity", "ambient_light"]
    }
    
    private func getConnectionType() -> String {
        // 实现网络连接类型检测
        return "unknown"
    }
    
    private func isNetworkConnected() -> Bool {
        // 实现网络连接状态检测
        return true
    }
    
    private func getWifiName() -> String? {
        // 实现WiFi名称获取
        return nil
    }
    
    private func getIpAddress() -> String? {
        // 实现IP地址获取
        return nil
    }
    
    private func getMacAddress() -> String? {
        // 实现MAC地址获取
        return nil
    }
    
    private func getSignalStrength() -> Int? {
        // 实现信号强度获取
        return nil
    }
    
    private func getStorageDevices() -> [[String: Any]] {
        // 实现存储设备列表获取
        return []
    }
    
    private func getDocumentsDirectory() -> String {
        let paths = NSSearchPathForDirectoriesInDomains(.documentDirectory, .userDomainMask, true)
        return paths.first ?? ""
    }
    
    private func getBatteryStatus(_ state: UIDevice.BatteryState) -> String {
        switch state {
        case .charging:
            return "charging"
        case .full:
            return "full"
        case .unplugged:
            return "discharging"
        case .unknown:
            return "unknown"
        @unknown default:
            return "unknown"
        }
    }
    
    private func getCpuUsage() -> Double {
        // 实现CPU使用率获取
        return 0.0
    }
    
    private func getMemoryUsage() -> Double {
        // 实现内存使用率获取
        return 0.0
    }
    
    private func getDiskUsage() -> Double {
        // 实现磁盘使用率获取
        return 0.0
    }
    
    private func getNetworkUsage() -> Double {
        // 实现网络使用率获取
        return 0.0
    }
    
    private func getRunningProcessCount() -> Int {
        // 实现运行进程数获取
        return 0
    }
    
    private func getTopProcesses() -> [[String: Any]] {
        // 实现顶级进程列表获取
        return []
    }
    
    private func getSystemMetrics() -> [String: Any] {
        // 实现系统指标获取
        return [:]
    }
    
    private func getMethodChannel() -> FlutterMethodChannel? {
        // 获取方法通道实例
        return nil
    }
    
    private func runDiagnosticsAsync(call: FlutterMethodCall, result: @escaping FlutterResult) {
         // 实现异步诊断
         result(nil)
     }
 }
 ```

## 插件发布与分发

### 发布到pub.dev

#### 1. 准备发布

```yaml
# pubspec.yaml
name: device_info_plugin
description: A comprehensive Flutter plugin for retrieving device information across platforms.
version: 1.0.0
homepage: https://github.com/your-username/device_info_plugin
repository: https://github.com/your-username/device_info_plugin
issue_tracker: https://github.com/your-username/device_info_plugin/issues

environment:
  sdk: '>=2.17.0 <4.0.0'
  flutter: '>=3.0.0'

dependencies:
  flutter:
    sdk: flutter
  plugin_platform_interface: ^2.1.4

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^2.0.0
  mockito: ^5.4.0
  build_runner: ^2.3.3
  integration_test:
    sdk: flutter

flutter:
  plugin:
    platforms:
      android:
        package: com.example.device_info
        pluginClass: DeviceInfoPlugin
      ios:
        pluginClass: DeviceInfoPlugin
```

#### 2. 创建CHANGELOG.md

```markdown
# Changelog

## [1.0.0] - 2022-11-25

### Added
- Initial release of device_info_plugin
- Support for retrieving device information on Android and iOS
- Hardware information including CPU, memory, and GPU details
- Network information with connection type and status
- Storage information with space usage details
- Battery information with level, status, and health
- Performance monitoring capabilities
- Real-time battery status updates via event streams
- Comprehensive error handling and validation
- Full test coverage with unit, widget, and integration tests

### Features
- Cross-platform compatibility (Android 21+, iOS 11+)
- Asynchronous API with callback support
- Event-driven architecture for real-time updates
- Robust error handling with detailed error codes
- Extensive documentation and examples
- Performance optimized implementations
```

#### 3. 创建README.md

```markdown
# Device Info Plugin

A comprehensive Flutter plugin for retrieving detailed device information across Android and iOS platforms.

## Features

- 📱 **Device Information**: Model, manufacturer, OS version, platform details
- 🔧 **Hardware Details**: CPU, memory, GPU, sensors, architecture
- 🌐 **Network Status**: Connection type, WiFi info, signal strength
- 💾 **Storage Info**: Total, free, and used space across devices
- 🔋 **Battery Status**: Level, charging state, health, temperature
- ⚡ **Performance Metrics**: CPU usage, memory usage, system metrics
- 📡 **Real-time Updates**: Live battery status monitoring

## Installation

Add this to your package's `pubspec.yaml` file:

```yaml
dependencies:
  device_info_plugin: ^1.0.0
```

Then run:

```bash
flutter pub get
```

## Usage

### Basic Usage

```dart
import 'package:device_info_plugin/device_info_plugin.dart';

final plugin = DeviceInfoPlugin();

// Check if plugin is available
final isAvailable = await plugin.isAvailable();

if (isAvailable) {
  // Get device information
  final deviceInfo = await plugin.getDeviceInfo();
  print('Device: ${deviceInfo.deviceName}');
  print('Model: ${deviceInfo.model}');
  print('OS: ${deviceInfo.operatingSystem} ${deviceInfo.osVersion}');
  
  // Get hardware information
  final hardwareInfo = await plugin.getHardwareInfo();
  print('Processor: ${hardwareInfo.processor}');
  print('Cores: ${hardwareInfo.coreCount}');
  print('Total Memory: ${hardwareInfo.totalMemory} bytes');
  
  // Get battery information
  final batteryInfo = await plugin.getBatteryInfo();
  print('Battery Level: ${batteryInfo.level}%');
  print('Charging: ${batteryInfo.isCharging}');
}
```

### Real-time Battery Monitoring

```dart
StreamSubscription? subscription;

void startBatteryMonitoring() {
  subscription = plugin.batteryStatusStream.listen(
    (batteryInfo) {
      print('Battery Level: ${batteryInfo.level}%');
      print('Status: ${batteryInfo.status}');
    },
    onError: (error) {
      print('Battery monitoring error: $error');
    },
  );
}

void stopBatteryMonitoring() {
  subscription?.cancel();
  subscription = null;
}
```

### Performance Monitoring

```dart
void monitorPerformance() async {
  try {
    final performanceInfo = await plugin.getPerformanceInfo(
      timeout: Duration(seconds: 10),
      onProgress: (progress) {
        print('Performance analysis progress: ${progress * 100}%');
      },
    );
    
    print('CPU Usage: ${performanceInfo.cpuUsage}%');
    print('Memory Usage: ${performanceInfo.memoryUsage}%');
    print('Running Processes: ${performanceInfo.runningProcesses}');
  } catch (e) {
    print('Performance monitoring failed: $e');
  }
}
```

## Platform Support

| Platform | Minimum Version | Features |
|----------|-----------------|----------|
| Android  | API 21 (5.0)    | Full support |
| iOS      | iOS 11.0        | Full support |

## Permissions

### Android

Add these permissions to your `android/app/src/main/AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
<uses-permission android:name="android.permission.BATTERY_STATS" />
```

### iOS

No additional permissions required for basic functionality.

## API Reference

For detailed API documentation, see [API Documentation](https://pub.dev/documentation/device_info_plugin/latest/).

## Contributing

Contributions are welcome! Please read our [Contributing Guide](CONTRIBUTING.md) for details.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
```

#### 4. 发布命令

```bash
# 验证包
flutter packages pub publish --dry-run

# 发布到pub.dev
flutter packages pub publish
```

### 版本管理策略

#### 语义化版本控制

```dart
// lib/src/version.dart
class PluginVersion {
  static const String version = '1.0.0';
  static const int major = 1;
  static const int minor = 0;
  static const int patch = 0;
  
  static const String buildNumber = '1';
  static const String buildDate = '2022-11-25';
  
  static String get fullVersion => '$version+$buildNumber';
  
  static bool isCompatibleWith(String requiredVersion) {
    final required = Version.parse(requiredVersion);
    final current = Version.parse(version);
    
    // 主版本必须匹配
    if (required.major != current.major) {
      return false;
    }
    
    // 次版本向后兼容
    if (required.minor > current.minor) {
      return false;
    }
    
    return true;
  }
}

class Version {
  final int major;
  final int minor;
  final int patch;
  
  const Version(this.major, this.minor, this.patch);
  
  static Version parse(String version) {
    final parts = version.split('.');
    return Version(
      int.parse(parts[0]),
      int.parse(parts[1]),
      int.parse(parts[2]),
    );
  }
  
  @override
  String toString() => '$major.$minor.$patch';
}
```

#### 自动化发布流程

```yaml
# .github/workflows/publish.yml
name: Publish Plugin

on:
  push:
    tags:
      - 'v*'

jobs:
  publish:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Flutter
      uses: subosito/flutter-action@v2
      with:
        flutter-version: '3.13.0'
        
    - name: Install dependencies
      run: flutter pub get
      
    - name: Run tests
      run: |
        flutter test
        flutter test integration_test/
        
    - name: Analyze code
      run: flutter analyze
      
    - name: Check formatting
      run: dart format --set-exit-if-changed .
      
    - name: Publish to pub.dev
      uses: k-paxian/dart-package-publisher@v1.5.1
      with:
        credentialJson: ${{ secrets.CREDENTIAL_JSON }}
        flutter: true
        skipTests: true
```

## 插件开发最佳实践

### 1. 架构设计原则

#### 单一职责原则

```dart
// 好的设计：每个类只负责一个功能
class DeviceInfoCollector {
  Future<DeviceInfo> collectDeviceInfo();
}

class HardwareInfoCollector {
  Future<HardwareInfo> collectHardwareInfo();
}

class NetworkInfoCollector {
  Future<NetworkInfo> collectNetworkInfo();
}

// 不好的设计：一个类负责所有功能
class AllInOneCollector {
  Future<DeviceInfo> getDeviceInfo();
  Future<HardwareInfo> getHardwareInfo();
  Future<NetworkInfo> getNetworkInfo();
  Future<StorageInfo> getStorageInfo();
  Future<BatteryInfo> getBatteryInfo();
  // ... 太多职责
}
```

#### 依赖注入

```dart
// lib/src/core/dependency_injection.dart
abstract class ServiceLocator {
  static final Map<Type, dynamic> _services = {};
  
  static void register<T>(T service) {
    _services[T] = service;
  }
  
  static T get<T>() {
    final service = _services[T];
    if (service == null) {
      throw Exception('Service of type $T not registered');
    }
    return service as T;
  }
  
  static void clear() {
    _services.clear();
  }
}

// 使用示例
void setupDependencies() {
  ServiceLocator.register<DeviceInfoCollector>(
    DeviceInfoCollectorImpl(),
  );
  ServiceLocator.register<HardwareInfoCollector>(
    HardwareInfoCollectorImpl(),
  );
  ServiceLocator.register<NetworkInfoCollector>(
    NetworkInfoCollectorImpl(),
  );
}

class DeviceInfoPlugin {
  late final DeviceInfoCollector _deviceInfoCollector;
  late final HardwareInfoCollector _hardwareInfoCollector;
  late final NetworkInfoCollector _networkInfoCollector;
  
  DeviceInfoPlugin() {
    _deviceInfoCollector = ServiceLocator.get<DeviceInfoCollector>();
    _hardwareInfoCollector = ServiceLocator.get<HardwareInfoCollector>();
    _networkInfoCollector = ServiceLocator.get<NetworkInfoCollector>();
  }
}
```

### 2. 错误处理策略

#### 分层错误处理

```dart
// lib/src/core/error_handling.dart
abstract class PluginException implements Exception {
  final String code;
  final String message;
  final dynamic details;
  final StackTrace? stackTrace;
  
  const PluginException({
    required this.code,
    required this.message,
    this.details,
    this.stackTrace,
  });
  
  Map<String, dynamic> toMap() {
    return {
      'code': code,
      'message': message,
      'details': details,
    };
  }
}

class DeviceInfoException extends PluginException {
  const DeviceInfoException({
    required String code,
    required String message,
    dynamic details,
    StackTrace? stackTrace,
  }) : super(
    code: code,
    message: message,
    details: details,
    stackTrace: stackTrace,
  );
}

class NetworkException extends PluginException {
  const NetworkException({
    required String code,
    required String message,
    dynamic details,
    StackTrace? stackTrace,
  }) : super(
    code: code,
    message: message,
    details: details,
    stackTrace: stackTrace,
  );
}

// 错误处理器
class ErrorHandler {
  static void handleError(dynamic error, StackTrace stackTrace) {
    if (error is PluginException) {
      _logPluginError(error);
    } else {
      _logUnknownError(error, stackTrace);
    }
  }
  
  static void _logPluginError(PluginException error) {
    print('Plugin Error [${error.code}]: ${error.message}');
    if (error.details != null) {
      print('Details: ${error.details}');
    }
  }
  
  static void _logUnknownError(dynamic error, StackTrace stackTrace) {
    print('Unknown Error: $error');
    print('Stack Trace: $stackTrace');
  }
}
```

### 3. 性能优化技巧

#### 缓存策略

```dart
// lib/src/core/cache_manager.dart
class CacheManager {
  static final Map<String, CacheEntry> _cache = {};
  static const Duration defaultTtl = Duration(minutes: 5);
  
  static T? get<T>(String key) {
    final entry = _cache[key];
    if (entry == null || entry.isExpired) {
      _cache.remove(key);
      return null;
    }
    return entry.value as T;
  }
  
  static void set<T>(String key, T value, {Duration? ttl}) {
    _cache[key] = CacheEntry(
      value: value,
      expiry: DateTime.now().add(ttl ?? defaultTtl),
    );
  }
  
  static void clear() {
    _cache.clear();
  }
  
  static void clearExpired() {
    _cache.removeWhere((key, entry) => entry.isExpired);
  }
}

class CacheEntry {
  final dynamic value;
  final DateTime expiry;
  
  CacheEntry({required this.value, required this.expiry});
  
  bool get isExpired => DateTime.now().isAfter(expiry);
}

// 使用缓存的收集器
class CachedDeviceInfoCollector implements DeviceInfoCollector {
  final DeviceInfoCollector _delegate;
  
  CachedDeviceInfoCollector(this._delegate);
  
  @override
  Future<DeviceInfo> collectDeviceInfo() async {
    const cacheKey = 'device_info';
    
    // 尝试从缓存获取
    final cached = CacheManager.get<DeviceInfo>(cacheKey);
    if (cached != null) {
      return cached;
    }
    
    // 缓存未命中，重新收集
    final deviceInfo = await _delegate.collectDeviceInfo();
    
    // 缓存结果
    CacheManager.set(cacheKey, deviceInfo, ttl: Duration(hours: 1));
    
    return deviceInfo;
  }
}
```

#### 异步操作优化

```dart
// lib/src/core/async_utils.dart
class AsyncUtils {
  /// 带超时的异步操作
  static Future<T> withTimeout<T>(
    Future<T> future, {
    Duration timeout = const Duration(seconds: 30),
    String? timeoutMessage,
  }) async {
    try {
      return await future.timeout(
        timeout,
        onTimeout: () => throw TimeoutException(
          timeoutMessage ?? 'Operation timed out after ${timeout.inSeconds}s',
          timeout,
        ),
      );
    } catch (e) {
      throw PluginException(
        code: 'TIMEOUT_ERROR',
        message: timeoutMessage ?? 'Operation timed out',
        details: {'timeout': timeout.inSeconds},
      );
    }
  }
  
  /// 重试机制
  static Future<T> withRetry<T>(
    Future<T> Function() operation, {
    int maxRetries = 3,
    Duration delay = const Duration(seconds: 1),
    bool Function(dynamic error)? shouldRetry,
  }) async {
    int attempts = 0;
    dynamic lastError;
    
    while (attempts < maxRetries) {
      try {
        return await operation();
      } catch (e) {
        lastError = e;
        attempts++;
        
        if (attempts >= maxRetries) {
          break;
        }
        
        if (shouldRetry != null && !shouldRetry(e)) {
          break;
        }
        
        await Future.delayed(delay * attempts);
      }
    }
    
    throw PluginException(
      code: 'RETRY_FAILED',
      message: 'Operation failed after $maxRetries attempts',
      details: {
        'attempts': attempts,
        'lastError': lastError.toString(),
      },
    );
  }
  
  /// 并发控制
  static Future<List<T>> withConcurrencyLimit<T>(
    List<Future<T> Function()> operations, {
    int concurrencyLimit = 3,
  }) async {
    final results = <T>[];
    final semaphore = Semaphore(concurrencyLimit);
    
    final futures = operations.map((operation) async {
      await semaphore.acquire();
      try {
        return await operation();
      } finally {
        semaphore.release();
      }
    });
    
    return await Future.wait(futures);
  }
}

class Semaphore {
  final int maxCount;
  int _currentCount;
  final Queue<Completer<void>> _waitQueue = Queue();
  
  Semaphore(this.maxCount) : _currentCount = maxCount;
  
  Future<void> acquire() async {
    if (_currentCount > 0) {
      _currentCount--;
      return;
    }
    
    final completer = Completer<void>();
    _waitQueue.add(completer);
    return completer.future;
  }
  
  void release() {
    if (_waitQueue.isNotEmpty) {
      final completer = _waitQueue.removeFirst();
      completer.complete();
    } else {
      _currentCount++;
    }
  }
}
```

### 4. 文档和示例

#### API文档规范

```dart
/// A comprehensive Flutter plugin for retrieving device information.
/// 
/// This plugin provides access to detailed device information including:
/// - Device specifications (model, manufacturer, OS version)
/// - Hardware details (CPU, memory, GPU, sensors)
/// - Network information (connection type, WiFi details)
/// - Storage information (space usage, available devices)
/// - Battery status (level, charging state, health)
/// - Performance metrics (CPU usage, memory usage)
/// 
/// ## Platform Support
/// 
/// | Platform | Minimum Version | Features |
/// |----------|-----------------|----------|
/// | Android  | API 21 (5.0)    | Full support |
/// | iOS      | iOS 11.0        | Full support |
/// 
/// ## Usage
/// 
/// ```dart
/// final plugin = DeviceInfoPlugin();
/// 
/// // Check availability
/// if (await plugin.isAvailable()) {
///   // Get device information
///   final deviceInfo = await plugin.getDeviceInfo();
///   print('Device: ${deviceInfo.deviceName}');
/// }
/// ```
/// 
/// ## Error Handling
/// 
/// All methods may throw [PluginException] with specific error codes:
/// - `DEVICE_INFO_ERROR`: Failed to retrieve device information
/// - `HARDWARE_INFO_ERROR`: Failed to retrieve hardware information
/// - `NETWORK_INFO_ERROR`: Failed to retrieve network information
/// - `STORAGE_INFO_ERROR`: Failed to retrieve storage information
/// - `BATTERY_INFO_ERROR`: Failed to retrieve battery information
/// - `PERMISSION_DENIED`: Required permissions not granted
/// - `PLATFORM_NOT_SUPPORTED`: Feature not supported on current platform
class DeviceInfoPlugin {
  /// Creates a new instance of [DeviceInfoPlugin].
  /// 
  /// The plugin will automatically initialize platform-specific implementations.
  DeviceInfoPlugin();
  
  /// Checks if the plugin is available on the current platform.
  /// 
  /// Returns `true` if the plugin is supported and properly initialized,
  /// `false` otherwise.
  /// 
  /// This method should be called before using any other plugin methods
  /// to ensure compatibility.
  /// 
  /// Example:
  /// ```dart
  /// final plugin = DeviceInfoPlugin();
  /// if (await plugin.isAvailable()) {
  ///   // Plugin is ready to use
  /// } else {
  ///   // Handle unsupported platform
  /// }
  /// ```
  Future<bool> isAvailable();
  
  /// Retrieves comprehensive device information.
  /// 
  /// Returns a [DeviceInfo] object containing:
  /// - Device ID (unique identifier)
  /// - Device name (user-assigned name)
  /// - Model (device model)
  /// - Manufacturer (device manufacturer)
  /// - Operating system name and version
  /// - Platform identifier
  /// - Physical device flag (true for real devices, false for simulators)
  /// 
  /// Throws [DeviceInfoException] if the information cannot be retrieved.
  /// 
  /// Example:
  /// ```dart
  /// try {
  ///   final deviceInfo = await plugin.getDeviceInfo();
  ///   print('Device: ${deviceInfo.deviceName}');
  ///   print('Model: ${deviceInfo.model}');
  ///   print('OS: ${deviceInfo.operatingSystem} ${deviceInfo.osVersion}');
  /// } catch (e) {
  ///   print('Failed to get device info: $e');
  /// }
  /// ```
  Future<DeviceInfo> getDeviceInfo();
}
```

## 总结

本文深入探讨了Flutter插件开发的完整流程，从基础架构设计到高级功能实现，再到发布和维护。通过DeviceInfoPlugin这个实际案例，我们学习了：

### 核心技术要点

1. **插件架构设计**：理解Flutter插件的三层架构（Dart层、Plugin层、Platform层），掌握MethodChannel和EventChannel的使用方法。

2. **平台特定实现**：学会在Android（Kotlin）和iOS（Swift）平台上实现原生功能，处理平台差异和兼容性问题。

3. **异步编程模式**：掌握Future、Stream、Completer等异步编程工具，实现高效的异步操作和事件流处理。

4. **错误处理机制**：建立完善的错误处理体系，包括异常分类、错误码定义、错误传播和用户友好的错误信息。

5. **性能优化策略**：实现缓存机制、并发控制、超时处理和重试机制，确保插件的高性能和稳定性。

### 开发最佳实践

1. **代码组织**：采用清晰的目录结构和命名规范，遵循单一职责原则，使用依赖注入提高代码的可测试性和可维护性。

2. **测试策略**：建立完整的测试体系，包括单元测试、Widget测试和集成测试，确保代码质量和功能正确性。

3. **文档规范**：编写详细的API文档、使用示例和最佳实践指南，提高插件的易用性和开发者体验。

4. **版本管理**：采用语义化版本控制，建立自动化发布流程，确保版本的稳定性和向后兼容性。

### 发布和维护

1. **发布流程**：掌握pub.dev发布流程，包括包验证、文档生成、版本标记和发布确认。

2. **社区支持**：建立问题跟踪系统，及时响应用户反馈，持续改进插件功能和性能。

3. **持续集成**：使用GitHub Actions等CI/CD工具，自动化测试、构建和发布流程。

Flutter插件开发是一个涉及多个技术栈的复杂过程，需要深入理解Flutter框架、原生平台开发和跨平台通信机制。通过系统的学习和实践，开发者可以创建出高质量、高性能的Flutter插件，为Flutter生态系统贡献力量。

随着Flutter技术的不断发展，插件开发也在不断演进。未来的发展趋势包括：

- **Federated Plugin架构**：支持更灵活的平台扩展和第三方实现
- **FFI（Foreign Function Interface）**：直接调用C/C++库，提高性能
- **Platform Views**：在Flutter中嵌入原生视图组件
- **Web和Desktop支持**：扩展到更多平台的插件开发

掌握这些核心概念和最佳实践，将为开发者在Flutter插件开发领域奠定坚实的基础，并能够适应未来技术的发展变化。
      })
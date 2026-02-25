---
layout: post
title: flutter_map 瓦片图层本地缓存踩坑记。
categories: [Development]
description: 为flutter_map实现瓦片图层缓存
keywords: Flutter, 地图, 瓦片图层
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

## 前言

[flutter_map](https://pub.dev/packages/flutter_map) 是一个基于leaflet开发的flutter包，用于在flutter应用中加载瓦片地图，但是默认并不提供本地缓存功能——这就意味着应用每次重新启动，所有瓦片都要重新下载，这显然会花费大量的流量，在网络不良的情况下也会影响应用的正常工作。

其实已经有开发者为flutter_map写了一个插件 [flutter_map_tile_caching](https://pub.dev/packages/flutter_map_tile_caching) 来提供瓦片图层缓存服务，但是恕我愚钝，愣是没看懂这玩意怎么用，于是就自己实现了一个带缓存功能的`TileProvider`。


## 分析
flutter_map 的`FlutterMap`和`TileLayer`是`StatefulWidget`抽象类的子类，后者可以被添加为前者的children，例如，我们可以这样实现一个最简单的 flutter_map：

```dart
import 'package:flutter/material.dart';
import 'package:flutter_map/flutter_map.dart';

void main() => runApp(const MyApp());

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  static const String _title = 'flutter map example';

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: _title,
      home: Scaffold(
        appBar: AppBar(
          title: const Text(_title),
        ),
        body: FlutterMap(
          options: MapOptions(),
          children: [
            TileLayer(
              urlTemplate: "https://tile.openstreetmap.org/{z}/{x}/{y}.png",
              userAgentPackageName: 'flutter_map_example',
            ),
          ],
        ),
      ),
    );
  }
}
```

`FlutterMap`类实际上只是提供了一个空间，或者说一个坐标系，用来放置地图图层，所以它与我们要处理的瓦片地图缓存无关。而`TileLayer`类才是显示地图图层的组件，它的构造函数有非常多的参数：

```dart
  TileLayer({
    super.key,
    this.urlTemplate,
    double tileSize = 256.0,
    double minZoom = 0.0,
    double maxZoom = 18.0,
    this.minNativeZoom,
    this.maxNativeZoom,
    this.zoomReverse = false,
    double zoomOffset = 0.0,
    Map<String, String>? additionalOptions,
    this.subdomains = const <String>[],
    this.keepBuffer = 2,
    this.backgroundColor = const Color(0xFFE0E0E0),
    this.errorImage,
    TileProvider? tileProvider,
    this.tms = false,
    this.wmsOptions,
    this.opacity = 1.0,
    Duration updateInterval = const Duration(milliseconds: 200),
    Duration tileFadeInDuration = const Duration(milliseconds: 100),
    this.tileFadeInStart = 0.0,
    this.tileFadeInStartWhenOverride = 0.0,
    this.overrideTilesWhenUrlChanges = false,
    this.retinaMode = false,
    this.errorTileCallback,
    this.templateFunction = util.template,
    this.tileBuilder,
    this.tilesContainerBuilder,
    this.evictErrorTileStrategy = EvictErrorTileStrategy.none,
    this.fastReplace = false,
    this.reset,
    this.tileBounds,
    String userAgentPackageName = 'unknown',
  })
```
这里面很多参数都是见名知义的，比如`tileSize`、`minZoom`、`maxZoom`等等，可以注意到在上面的示例中只提供了`urlTemplate`一个参数，这是因为`TileLayer`类默认使用的`TileProvider`是`NetworkNoRetryTileProvider`，它根据url从网络上的在线地图服务获取地图数据，如果不提供`urlTemplate`，运行时会报`Unexpected null value.`。

`NetworkNoRetryTileProvider`是`TileProvider`抽象类的子类，`TileLayer`类也提供了可选的`tileProvider`参数供我们指定其它的`TileProvider`。

阅读`TileProvider`抽象类和`NetworkNoRetryTileProvider`子类的代码（如下）

```dart
abstract class TileProvider {
  Map<String, String> headers;

  TileProvider({
    this.headers = const {},
  });

  /// Retrieve a tile as an image, based on it's coordinates and the current [TileLayerOptions]
  ImageProvider getImage(Coords coords, TileLayer options);

  /// Called when the [TileLayerWidget] is disposed
  void dispose() {}

  /// Generate a valid URL for a tile, based on it's coordinates and the current [TileLayerOptions]
  String getTileUrl(Coords coords, TileLayer options) {
    final urlTemplate = (options.wmsOptions != null)
        ? options.wmsOptions!
            .getUrl(coords, options.tileSize.toInt(), options.retinaMode)
        : options.urlTemplate;

    final z = _getZoomForUrl(coords, options);

    final data = <String, String>{
      'x': coords.x.round().toString(),
      'y': coords.y.round().toString(),
      'z': z.round().toString(),
      's': getSubdomain(coords, options),
      'r': '@2x',
    };
    if (options.tms) {
      data['y'] = invertY(coords.y.round(), z.round()).toString();
    }
    final allOpts = Map<String, String>.from(data)
      ..addAll(options.additionalOptions);
    return options.templateFunction(urlTemplate!, allOpts);
  }

  double _getZoomForUrl(Coords coords, TileLayer options) {
    var zoom = coords.z;

    if (options.zoomReverse) {
      zoom = options.maxZoom - zoom;
    }

    return zoom += options.zoomOffset;
  }

  int invertY(int y, int z) {
    return ((1 << z) - 1) - y;
  }

  /// Get a subdomain value for a tile, based on it's coordinates and the current [TileLayerOptions]
  String getSubdomain(Coords coords, TileLayer options) {
    if (options.subdomains.isEmpty) {
      return '';
    }
    final index = (coords.x + coords.y).round() % options.subdomains.length;
    return options.subdomains[index];
  }
}
```

```dart
class NetworkNoRetryTileProvider extends TileProvider {
  NetworkNoRetryTileProvider({
    Map<String, String>? headers,
    HttpClient? httpClient,
  }) {
    this.headers = headers ?? {};
    this.httpClient = httpClient ?? HttpClient()
      ..userAgent = null;
  }

  late final HttpClient httpClient;

  @override
  ImageProvider getImage(Coords<num> coords, TileLayer options) =>
      FMNetworkNoRetryImageProvider(
        getTileUrl(coords, options),
        headers: headers,
        httpClient: httpClient,
      );
}
```

可以发现，除了构造函数之外，`NetworkNoRetryTileProvider`仅重写了`TileProvider`抽象类的`getImage`一个方法，它的返回值是一个`ImageProvider`实例。我们知道`ImageProvider`的主要用途是作为`Image`组件的`image`参数的类型，用于`Image`组件中图片的获取和加载。

因此，我们就有了一个实现缓存功能的思路，实现一个自己的`TileProvider`并重写`getImage`方法，以伪代码方式描述如下：

```dart
@override
ImageProvider getImage(Coords<num> coords, TileLayer options) {
	file = File(getPath(coords));
	if (file.exists()){
		return FileImage(file); // 如果文件存在，返回 FileImage
	} else {
		url = getTileUrl(coords, options);
		networkImage = NetworkImage(url)
		saveImage(file, networkImage); // saveImage是一个异步函数，使用resolve方法从ImageProvider中获取数据流；
		return networkImage;
	}
}
```
我最开始就是这样实现的，但是这样做的缺点非常明显：每张图片都被下载了两次，流量什么的倒是次要的了，主要问题是服务器端持续报`429 Too Many Requests`，最终导致应用强制关闭。那么是否可以这样修改呢：

```dart
@override
ImageProvider getImage(Coords<num> coords, TileLayer options) {
	file = File(getPath(coords));
	if (file.exists()){
		return FileImage(file); // 如果文件存在，返回 FileImage
	} else {
		url = getTileUrl(coords, options);
		download = downloadImage(file, url); // 同步函数，等待下载完成后再返回值；
		if (download.success){
			return FileImage(file); // 下载成功，返回 FileImage
		} else {
			return null;
		}
	}
}
```

这样做的缺点也很明显：图片被下载到内部存储中之后，再从内部存储中读取，完全是多此一举，浪费时间，还要耗费额外的内存等运行资源。

于是，我们想到`ImageProvider`类是以数据流`ImageStream`的形式向`Image`组件提供图片，那么我们可以重写某个涉及到`ImageStream`的方法，为其添加一个`Listener`。

`resolve`是`ImageProvider`暴露给`Image`组件的主入口方法，通过阅读代码，可以发现它的`stream`来自`createStream`方法。`createStream`方法明显比`resolve`更适合重写，代码的注释中也这样建议（Subclasses should override this instead of [resolve] if they need to …）。

```dart
  @nonVirtual
  ImageStream resolve(ImageConfiguration configuration) {
    assert(configuration != null);
    final ImageStream stream = createStream(configuration);
    // Load the key (potentially asynchronously), set up an error handling zone,
    // and call resolveStreamForKey.
    _createErrorHandlerAndKey(
      configuration,
      (T key, ImageErrorListener errorHandler) {
        resolveStreamForKey(configuration, stream, key, errorHandler);
      },
      (T? key, Object exception, StackTrace? stack) async {
        await null; // wait an event turn in case a listener has been added to the image stream.
        InformationCollector? collector;
        assert(() {
          collector = () => <DiagnosticsNode>[
            DiagnosticsProperty<ImageProvider>('Image provider', this),
            DiagnosticsProperty<ImageConfiguration>('Image configuration', configuration),
            DiagnosticsProperty<T>('Image key', key, defaultValue: null),
          ];
          return true;
        }());
        if (stream.completer == null) {
          stream.setCompleter(_ErrorImageCompleter());
        }
        stream.completer!.reportError(
          exception: exception,
          stack: stack,
          context: ErrorDescription('while resolving an image'),
          silent: true, // could be a network error or whatnot
          informationCollector: collector,
        );
      },
    );
    return stream;
  }

  /// Called by [resolve] to create the [ImageStream] it returns.
  ///
  /// Subclasses should override this instead of [resolve] if they need to
  /// return some subclass of [ImageStream]. The stream created here will be
  /// passed to [resolveStreamForKey].
  @protected
  ImageStream createStream(ImageConfiguration configuration) {
    return ImageStream();
  }
```

`NetworkNoRetryTileProvider`的`getImage`方法返回的是`FMNetworkNoRetryImageProvider`的实例，这是flutter_map自己实现的一个`ImageProvider`子类，不妨就让我们的`ImageProvider`继承它。

## 实现

一开始，我们就遇到了一个大麻烦，[path_provider](https://pub.dev/packages/path_provider) 包提供的获取缓存路径的`getTemporaryDirectory()`方法是异步的，而`TileProvider`的`getImage`方法是同步的，无法在后者中调用前者，因此，我创建了一个静态类`AppDir`，我们知道静态类是单例的，因此可以让路径一次获取，全局调用。

```dart
import 'dart:io';
import 'package:path_provider/path_provider.dart';

class AppDir {
  static Directory data = Directory('');
  static Directory cache = Directory('');

  static setDir() async {
    data = await getApplicationDocumentsDirectory();
    cache = await getTemporaryDirectory();
  }
}
```

我们需要修改主函数，以在应用启动时确保获取到系统路径：

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  while (AppDir.data.path.isEmpty || AppDir.cache.path.isEmpty) {
    await AppDir.setDir();
  }
  runApp(const MyApp());
}
```

下面，我们创建两个子类，继承`NetworkNoRetryTileProvider`和`FMNetworkNoRetryImageProvider`：

```dart
import 'dart:async';
import 'dart:developer' as dev;
import 'dart:io';
import 'dart:typed_data';
import 'dart:ui' as ui;

import 'package:flutter/widgets.dart';
import 'package:flutter_map/flutter_map.dart';
import 'package:flutter_map/src/layer/tile_layer/tile_provider/network_no_retry_image_provider.dart'; // this line will be warned as "Don't import Implementation files from other package", just ignore it.
import 'package:naturalist/entity/app_dir.dart';
import 'package:path/path.dart' as path;

class CacheTileProvider extends NetworkNoRetryTileProvider {
  String tileName;

  CacheTileProvider(
    this.tileName,{  // 这是新添加的参数，用于区分不同的瓦片图源；下面两个参数继承自NetworkNoRetryTileProvider
    super.headers,
    super.httpClient,
  });

  @override
  ImageProvider getImage(Coords<num> coords, TileLayer options) {
    File file = File(path.join(
        AppDir.cache.path,  // 应用缓存路径
        'flutter_map_tiles',  // 表明这是 flutter_map 使用的目录
        tileName,  // 以tileName区分不同的瓦片图源
        coords.z.round().toString(),
        coords.x.round().toString(),
        '${coords.y.round().toString()}.png'));

    if (file.existsSync()) {
      return FileImage(file);
    } else {
      return NetworkImageSaverProvider(
        getTileUrl(coords, options),
        file,
        headers: headers,
        httpClient: httpClient,
      );
    }
  }
}

class NetworkImageSaverProvider extends FMNetworkNoRetryImageProvider {
  File file;

  NetworkImageSaverProvider(
    super.url,
    this.file, {  // 新添加的参数，图片保存的目标文件。
    HttpClient? httpClient,
    super.headers = const {},
  });

  @override
  ImageStream createStream(ImageConfiguration configuration) {  // 重写createStream，为stream添加listener
    ImageStream stream =  ImageStream();
    ImageStreamListener listener = ImageStreamListener(imageListener);
    stream.addListener(listener);
    return stream;
  }

  void imageListener(ImageInfo imageInfo, bool synchronousCall){
    ui.Image uiImage = imageInfo.image;
    _saveImage(uiImage);
  }

  Future<void> _saveImage (ui.Image uiImage) async {  // 异步保存图片
    try {
      Directory parent = file.parent;
      if (! await parent.exists()){
        await parent.create(recursive: true);  // 如果目录不存在，逐级创建。
      }
      ByteData? bytes = await uiImage.toByteData(format: ui.ImageByteFormat.png);
      if (bytes != null) {
        final buffer = bytes.buffer;
        file.writeAsBytes(buffer.asUint8List(bytes.offsetInBytes, bytes.lengthInBytes));  // 将二进制数据写入图片文件。
      }
    } catch (e) {
      dev.log(e.toString());
    }
  }
}

```

更新`TileLayer`，更新后主文件如下：

```dart
import 'package:flutter/material.dart';
import 'package:flutter_map/flutter_map.dart';

import 'entity/cache_tile_provider.dart';
import 'entity/app_dir.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  while (AppDir.data.path.isEmpty || AppDir.cache.path.isEmpty) {
    await AppDir.setDir();
  }
  runApp(const MyApp());
}
class MyApp extends StatelessWidget {
  const MyApp({super.key});

  static const String _title = 'flutter map example';

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: _title,
      home: Scaffold(
        appBar: AppBar(
          title: const Text(_title),
        ),
        body: FlutterMap(
          options: MapOptions(),
          children: [
            TileLayer(
              tileProvider: CacheTileProvider('osm'),
              urlTemplate: "https://tile.openstreetmap.org/{z}/{x}/{y}.png",
            ),
          ],
        ),
      ),
    );
  }
}

```

经过实际测试，未缓存区域的加载速度与默认状态没有可感知的差别，已缓存区域的加载速度明显快于默认状态。查看手机文件系统，可以看到，访问过的瓦片图层都已被缓存，断网状态下，已缓存的区域依然可以显示地图：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8aa7f36243b34760824ff6f407404f98~tplv-k3u1fbpfcp-watermark.image?)


## 免责声明
一些在线地图服务提供者不允许开发者在本地储存自己的地图数据，请在使用时仔细阅读地图服务提供者的许可协议，并仅在服务提供者允许的前提下储存数据。对于读者使用本文代码下载未经许可的地图数据的行为，一概与本文作者无关。

## 更新

在flutter_map 5.0.0中，一些类发生了变化，更新后的`cache_tile_provider`文件如下：
```dart
import 'dart:async';
import 'dart:developer' as dev;
import 'dart:io';
import 'dart:typed_data';
import 'dart:ui' as ui;

import 'package:flutter/widgets.dart';
import 'package:flutter_map/flutter_map.dart';
import 'package:flutter_map/src/layer/tile_layer/tile_provider/network_image_provider.dart'; // this line will be warned as "Don't import Implementation files from other package", just ignore it.
import '../entity/app_dir.dart';
import 'package:path/path.dart' as path;

class CacheTileProvider extends NetworkTileProvider {
  String tileName;

  CacheTileProvider(
    this.tileName, {
    super.headers,
    super.httpClient,
  });

  @override
  ImageProvider getImage(TileCoordinates coordinates, TileLayer options) {
    File file = File(path.join(
        AppDir.cache.path,
        'flutter_map_tiles',
        tileName,
        coordinates.z.round().toString(),
        coordinates.x.round().toString(),
        '${coordinates.y.round().toString()}.png'));

    if (file.existsSync()) {
      return FileImage(file);
    } else {
      return NetworkImageSaverProvider(
        file,
        url: getTileUrl(coordinates, options),
        headers: headers,
        httpClient: httpClient,
        fallbackUrl: null,
      );
    }
  }
}

class NetworkImageSaverProvider extends FlutterMapNetworkImageProvider {
  File file;

  NetworkImageSaverProvider(
    this.file, {
    required super.url,
    super.fallbackUrl,
    required super.httpClient,
    super.headers = const {},
  });

  @override
  ImageStream createStream(ImageConfiguration configuration) {
    ImageStream stream = ImageStream();
    ImageStreamListener listener = ImageStreamListener(imageListener);
    stream.addListener(listener);
    return stream;
  }

  void imageListener(ImageInfo imageInfo, bool synchronousCall) {
    ui.Image uiImage = imageInfo.image;
    _saveImage(uiImage);
  }

  Future<void> _saveImage(ui.Image uiImage) async {
    try {
      Directory parent = file.parent;
      if (!await parent.exists()) {
        await parent.create(recursive: true);
      }
      ByteData? bytes =
          await uiImage.toByteData(format: ui.ImageByteFormat.png);
      if (bytes != null) {
        final buffer = bytes.buffer;
        file.writeAsBytes(
            buffer.asUint8List(bytes.offsetInBytes, bytes.lengthInBytes));
      }
    } catch (e) {
      dev.log(e.toString());
    }
  }
}
```

# Flutter Downloader

A plugin for creating and managing download tasks. Supports iOS and Android.

This plugin is using WorkManager on Android and NSURLSessionDownloadTask on iOS to run download tasks in background.

Development note:

The changes of external storage APIs in Android 11 cause some problems with the current implementation. I decide to re-design this plugin with new strategy to manage download file location. It is still in triage and discussion in this PR. It is very appreciated to have contribution and feedback from Flutter developer to get better design for the plugin.

## iOS integration

Required configuration:

The following steps require to open your ios project in Xcode.

1. Enable background mode.

2. Add sqlite library.

3. Configure AppDelegate:

   `/// AppDelegate.h
   #import <Flutter/Flutter.h>
   #import <UIKit/UIKit.h>
   @interface AppDelegate : FlutterAppDelegate
   @end`

   `// AppDelegate.m
   #include "AppDelegate.h"
   #include "GeneratedPluginRegistrant.h"
   #include "FlutterDownloaderPlugin.h"
   @implementation AppDelegate
   void registerPlugins(NSObject<FlutterPluginRegistry>* registry) { 
   if (![registry hasPlugin:@"FlutterDownloaderPlugin"]) {
   [FlutterDownloaderPlugin registerWithRegistrar:[registry registrarForPlugin:@"FlutterDownloaderPlugin"]];
    }
   }
   (BOOL)application:(UIApplication *)application
  didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
  [GeneratedPluginRegistrant registerWithRegistry:self];
  [FlutterDownloaderPlugin setPluginRegistrantCallback:registerPlugins];
  // Override point for customization after application launch.
  return [super application:application didFinishLaunchingWithOptions:launchOptions];
  }
  @end`

Or Swift:

   `import UIKit
   import Flutter
   import flutter_downloader
   @UIApplicationMain
   @objc class AppDelegate: FlutterAppDelegate {
   override func application(
   application: UIApplication,
   didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
   ) -> Bool {
   GeneratedPluginRegistrant.register(with: self)
   FlutterDownloaderPlugin.setPluginRegistrantCallback(registerPlugins)
   return super.application(application, didFinishLaunchingWithOptions: launchOptions)
   }
   }
   private func registerPlugins(registry: FlutterPluginRegistry) {
   if (!registry.hasPlugin("FlutterDownloaderPlugin")) {
   FlutterDownloaderPlugin.register(with: registry.registrar(forPlugin: "FlutterDownloaderPlugin")!)
   }
   }`

## Optional configuration:

Support HTTP request: if you want to download file with HTTP request, you need to disable Apple Transport Security (ATS) feature. There're two options:

1. Disable ATS for a specific domain only: (add the following code to your Info.plist file)

   `<key>NSAppTransportSecurity</key>
   <dict>
   <key>NSExceptionDomains</key>
   <dict>
   <key>www.yourserver.com</key>
   <dict>
   <key>NSIncludesSubdomains</key>
   <true/>
   <key>NSTemporaryExceptionAllowsInsecureHTTPLoads</key>
   <true/>
   <key>NSTemporaryExceptionMinimumTLSVersion</key>
   <string>TLSv1.1</string>
   </dict>
   </dict>
   </dict>`

2. Completely disable ATS. Add the following to your Info.plist file)

   `<key>NSAppTransportSecurity</key>
   <dict>
   <key>NSAllowsArbitraryLoads</key><true/>
   </dict>`

Configure maximum number of concurrent tasks: the plugin allows 3 download tasks running at a moment by default (if you enqueue more than 3 tasks, there're only 3 tasks running, other tasks are put in pending state). You can change this number by adding the following code to your Info.plist file.

   `<key>FDMaximumConcurrentTasks</key>
   <integer>5</integer>`

Localize notification messages: the plugin will send a notification message to notify user in case all files are downloaded while your application is not running in foreground. This message is English by default. You can localize this message by adding and localizing following message in Info.plist file. (you can find the detail of Info.plist localization in this link)

   `<key>FDAllFilesDownloadedMessage</key>
   <string>All files have been downloaded</string>`


## Android integratio

You don't have to do anything extra to make the plugin work on Android.

There are although a few optional settings you might want to configure.

Open downloaded file from notification

To make tapping on notification open the downloaded file on Android, add the following code to AndroidManifest.xml:

   `<provider
   android:name="vn.hunghd.flutterdownloader.DownloadedFileProvider"
   android:authorities="${applicationId}.flutter_downloader.provider"
   android:exported="false"
   android:grantUriPermissions="true">
   <meta-data
   android:name="android.support.FILE_PROVIDER_PATHS"
   android:resource="@xml/provider_paths"/>
   </provider>`

Notes

You have to save your downloaded files in external storage (where the other applications have permission to read your files)
The downloaded files are only able to be opened if your device has at least one application that can read these file types (mp3, pdf, etc.)
Configure maximum number of concurrent download tasks

The plugin depends on WorkManager library and WorkManager depends on the number of available processor to configure the maximum number of tasks running at a moment. You can setup a fixed number for this configuration by adding the following code to your AndroidManifest.xml:

   `<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge">
    <meta-data
    android:name="androidx.work.WorkManagerInitializer"
    android:value="androidx.startup"
    tools:node="remove" />
    </provider>
    <provider
    android:name="vn.hunghd.flutterdownloader.FlutterDownloaderInitializer"
    android:authorities="${applicationId}.flutter-downloader-init"
    android:exported="false">
    <meta-data
    android:name="vn.hunghd.flutterdownloader.MAX_CONCURRENT_TASKS"
    android:value="5" />
    </provider>`

## Localize strings in notifications

You can localize texts in download progress notifications by localizing following messages.


   `<string name="flutter_downloader_notification_started">Download started</string>
   <string name="flutter_downloader_notification_in_progress">Download in progress</string>
   <string name="flutter_downloader_notification_canceled">Download canceled</string>
   <string name="flutter_downloader_notification_failed">Download failed</string>
   <string name="flutter_downloader_notification_complete">Download complete</string>
   <string name="flutter_downloader_notification_paused">Download paused</string>`

You can learn more about localization on Android here.

Install .apk files

To open and install .apk files, your application needs REQUEST_INSTALL_PACKAGES permission. Add the following in your AndroidManifest.xml:

   `<uses-permission android:name="android.permission.REQUEST_INSTALL_PACKAGES" />`

## Usage

Import and initialize

   `import 'package:flutter_downloader/flutter_downloader.dart';
    void main() {
    WidgetsFlutterBinding.ensureInitialized();
    await FlutterDownloader.initialize(
    debug: true, // optional: set to false to disable printing logs to console (default: true)
    ignoreSsl: true // option: set to false to disable working with http links (default: false)
    );
    runApp(/*...*/)
    }`

Create new download task

   `final taskId = await FlutterDownloader.enqueue(
   url: 'your download link',
   headers: {}, // optional: header send with url (auth token etc)
   savedDir: 'the path of directory where you want to save downloaded files',
   showNotification: true, // show download progress in status bar (for Android)
   openFileFromNotification: true, // click on notification to open downloaded file (for Android)
   );`

Update download progress

   `await FlutterDownloader.registerCallback(callback); // callback is a top-level or static function`




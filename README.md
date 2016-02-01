# chromedriver-appium
Chromedriver for Mac OS X version 2.20 with a Crosswalk fix, exposing androidDevice socket with the package name appended at the end of the socket. (Tested on El Capitan)

#### How to use
 1. Swap your **/usr/local/lib/node_modules/appium/node_modules/appium-chromedriver/chromedriver** with the Chromedriver file here
 2. Swap your **../appium/lib/devices/android/android-hybrid.js** file
 3. Set your Appium capabilities to:

 **JSON example:**
  ```
    {
     "platformName":"Android",
     "deviceName":"Piotrek",
     "androidDeviceSocket":"**your.package.name**_devtools_remote",
     "chromeOptions": {
       "androidDeviceSocket":"**your.package.name**_devtools_remote
       }
     }
 ```

**Java example:**
 ```
     caps.setCapability("androidDeviceSocket", ANDROID_APP_PACKAGE + "_devtools_remote");
     ChromeOptions chromeOptions = new ChromeOptions();
     chromeOptions.setExperimentalOption("androidDeviceSocket", ANDROID_APP_PACKAGE + "_devtools_remote");
     caps.setCapability(ChromeOptions.CAPABILITY,chromeOptions);
```

### History

#### Fix 1 - Chromedriver android socket fix applied

diff --git a/chrome/test/chromedriver/capabilities.cc b/chrome/test/chromedriver/capabilities.cc
index 5846479..67de230 100644
--- a/chrome/test/chromedriver/capabilities.cc
+++ b/chrome/test/chromedriver/capabilities.cc
@@ -424,6 +424,8 @@ Status ParseChromeOptions(
         base::Bind(&ParseString, &capabilities->android_package);
     parser_map["androidProcess"] =
         base::Bind(&ParseString, &capabilities->android_process);
+    parser_map["androidDeviceSocket"] =
+        base::Bind(&ParseString, &capabilities->android_device_socket);
     parser_map["androidUseRunningApp"] =
         base::Bind(&ParseBoolean, &capabilities->android_use_running_app);
     parser_map["args"] = base::Bind(&ParseSwitches);
diff --git a/chrome/test/chromedriver/capabilities.h b/chrome/test/chromedriver/capabilities.h
index 2f78fec..85ebc62 100644
--- a/chrome/test/chromedriver/capabilities.h
+++ b/chrome/test/chromedriver/capabilities.h
@@ -105,6 +105,8 @@ struct Capabilities {

   std::string android_process;

+  std::string android_device_socket;
+
   bool android_use_running_app;

   base::FilePath binary;
diff --git a/chrome/test/chromedriver/chrome/device_manager.cc b/chrome/test/chromedriver/chrome/device_manager.cc
index c8c9736..5c92957 100644
--- a/chrome/test/chromedriver/chrome/device_manager.cc
+++ b/chrome/test/chromedriver/chrome/device_manager.cc
@@ -34,6 +34,7 @@ Device::~Device() {
 Status Device::SetUp(const std::string& package,
                      const std::string& activity,
                      const std::string& process,
+                     const std::string& device_socket,
                      const std::string& args,
                      bool use_running_app,
                      int port) {
@@ -47,19 +48,19 @@ Status Device::SetUp(const std::string& package,

   std::string known_activity;
   std::string command_line_file;
-  std::string device_socket;
+  std::string known_device_socket = device_socket;
   std::string exec_name;
   if (package.compare("org.chromium.content_shell_apk") == 0) {
     // Chromium content shell.
     known_activity = ".ContentShellActivity";
-    device_socket = "content_shell_devtools_remote";
+    known_device_socket = "content_shell_devtools_remote";
     command_line_file = "/data/local/tmp/content-shell-command-line";
     exec_name = "content_shell";
   } else if (package.find("chrome") != std::string::npos &&
              package.find("webview") == std::string::npos) {
     // Chrome.
     known_activity = "com.google.android.apps.chrome.Main";
-    device_socket = "chrome_devtools_remote";
+    known_device_socket = "chrome_devtools_remote";
     command_line_file = kChromeCmdLineFileBeforeM33;
     exec_name = "chrome";
     status = adb_->SetDebugApp(serial_, package);
@@ -74,9 +75,10 @@ Status Device::SetUp(const std::string& package,

     if (!known_activity.empty()) {
       if (!activity.empty() ||
-          !process.empty())
+          !process.empty() ||
+          !device_socket.empty())
         return Status(kUnknownError, "known package " + package +
-                      " does not accept activity/process");
+                      " does not accept activity/process/device_socket");
     } else if (activity.empty()) {
       return Status(kUnknownError, "WebView apps require activity name");
     }
@@ -108,7 +110,7 @@ Status Device::SetUp(const std::string& package,

     active_package_ = package;
   }
-  this->ForwardDevtoolsPort(package, process, port, &device_socket);
+  this->ForwardDevtoolsPort(package, process, port, &known_device_socket);

   return status;
 }
diff --git a/chrome/test/chromedriver/chrome/device_manager.h b/chrome/test/chromedriver/chrome/device_manager.h
index d27aa50..6348ba6 100644
--- a/chrome/test/chromedriver/chrome/device_manager.h
+++ b/chrome/test/chromedriver/chrome/device_manager.h
@@ -25,6 +25,7 @@ class Device {
   Status SetUp(const std::string& package,
                const std::string& activity,
                const std::string& process,
+               const std::string& device_socket,
                const std::string& args,
                bool use_running_app,
                int port);
diff --git a/chrome/test/chromedriver/chrome_launcher.cc b/chrome/test/chromedriver/chrome_launcher.cc
index ca9d504..8b64565 100644
--- a/chrome/test/chromedriver/chrome_launcher.cc
+++ b/chrome/test/chromedriver/chrome_launcher.cc
@@ -476,6 +476,7 @@ Status LaunchAndroidChrome(
   status = device->SetUp(capabilities.android_package,
                          capabilities.android_activity,
                          capabilities.android_process,
+                         capabilities.android_device_socket,
                          switches.ToString(),
                          capabilities.android_use_running_app,
                          port);


#### Fix 2 - Minimum version
Also, due to the fact that our Crosswalk uses Chrome 43 and Chromedriver 2.20 required a minimum of 46, changed the minimum required version to 0.0.1 in /src/chrome/test/chromedriver/chrome/version.h
 ** If you need that kind of check in your chromedriver please don't use this version **

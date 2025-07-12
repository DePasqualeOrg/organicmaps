# Creating a Library Using Organic Maps as Git Submodule

**Date**: 2025-07-13  
**Approach**: Git submodule integration with modern Swift C++ interop for iOS and Android libraries  
**Goal**: Create clean SDK wrappers around Organic Maps for .mwm offline mapping using Swift 5.9+ C++ interoperability

## Overview

This guide explains how to create your own mapping SDK that uses Organic Maps as a git submodule, providing clean APIs for iOS (Swift) and Android (Kotlin) while leveraging the powerful .mwm format for offline maps.

## Architecture Overview

```
YourMapSDK/
├── organicmaps/                     # Git submodule
│   ├── drape/                       # Rendering engine
│   ├── indexer/                     # .mwm file format
│   ├── map/                         # Core framework
│   ├── storage/                     # Map download management
│   ├── include/                     # Swift-friendly C++ headers
│   │   └── OrganicMapsC++API.hpp    # Modern C++ interop API
│   └── src/
│       └── swift_api.cpp            # C++ implementation bridge
├── Sources/
│   └── OrganicMapsSDK/
│       ├── MapView.swift            # Modern Swift API with C++ interop
│       ├── MapViewModel.swift       # SwiftUI-ready view model
│       └── DataModels.swift         # Swift data structures
├── android/
│   ├── src/main/java/
│   │   └── YourMapView.kt          # Public Kotlin API
│   ├── src/main/cpp/
│   │   └── wrapper.cpp             # JNI wrapper
│   └── build.gradle                # Android library
├── Package.swift                   # Swift Package Manager with C++ interop
├── shared/
│   ├── assets/                     # Map data and resources
│   └── CMakeLists.txt              # Cross-platform build
└── README.md
```

## Benefits of This Approach

### ✅ **Complete .mwm Ecosystem**
- Access to highly optimized offline map format (5-10x smaller than raster tiles)
- Integrated search, routing, and POI functionality
- Vector rendering with infinite zoom capability
- Multi-language support and transliteration

### ✅ **Ongoing Updates**
- Automatic access to bug fixes and improvements
- Regular OpenStreetMap data updates
- Security patches and performance optimizations
- No need to maintain your own fork

### ✅ **Production-Tested Code**
- Battle-tested in millions of Organic Maps installations
- Optimized for mobile performance
- Comprehensive platform support (iOS/Android)
- Well-documented C++ codebase

### ✅ **Modern Swift C++ Interoperability**
- Direct Swift ↔ C++ communication (no Objective-C++ bridging)
- Native Swift concurrency support (async/await, AsyncStream)
- SwiftUI-ready with @ObservableObject view models
- Zero runtime overhead with direct function calls

### ✅ **Clean Public APIs**
- Hide Organic Maps complexity behind simple interfaces
- Type-safe Swift and Kotlin APIs
- Focused feature set for your use case
- Easy to distribute via Swift Package Manager/Maven

## Project Setup

### 1. Initialize Your Project

```bash
# Create your SDK project
mkdir YourMapSDK
cd YourMapSDK

# Initialize git repository
git init

# Add Organic Maps as submodule
git submodule add https://github.com/organicmaps/organicmaps.git
git submodule update --init --recursive

# Create directory structure
mkdir -p Sources/OrganicMapsSDK
mkdir -p organicmaps/include organicmaps/src
mkdir -p android/src/main/java android/src/main/cpp
mkdir -p shared/assets
```

### 2. Root CMakeLists.txt

```cmake
# YourMapSDK/CMakeLists.txt
cmake_minimum_required(VERSION 3.22)
project(YourMapSDK)

# Configure for mobile-only build
set(SKIP_TESTS ON)
set(SKIP_QT_GUI ON) 
set(SKIP_TOOLS ON)
set(OMIM_MOBILE_ONLY ON)

# Add Organic Maps submodule
add_subdirectory(organicmaps)

# Platform-specific configurations
if(PLATFORM_IOS)
    add_subdirectory(ios)
elseif(PLATFORM_ANDROID)
    add_subdirectory(android)
endif()
```

## iOS Implementation with Modern Swift C++ Interop

### 1. Swift Package Manager Configuration

```swift
// Package.swift
// swift-tools-version:5.9
import PackageDescription

let package = Package(
    name: "OrganicMapsSDK",
    platforms: [
        .iOS(.v17),  // Required for @Observable macro
        .macOS(.v14)
    ],
    products: [
        .library(
            name: "OrganicMapsSDK",
            targets: ["OrganicMapsSDK"]
        ),
    ],
    targets: [
        // C++ Organic Maps Core
        .target(
            name: "OrganicMapsCore",
            path: "organicmaps",
            exclude: [
                "qt/",
                "android/",
                "tools/",
                "generator/",
                "pyhelpers/"
            ],
            sources: [
                "drape/",
                "drape_frontend/", 
                "map/",
                "indexer/",
                "storage/",
                "search/",
                "routing/",
                "platform/",
                "geometry/",
                "base/",
                "coding/",
                "src/"
            ],
            publicHeadersPath: "include",
            cxxSettings: [
                .headerSearchPath("."),
                .define("OMIM_OS_NAME", to: "ios")
            ]
        ),
        
        // Swift Wrapper with C++ Interop
        .target(
            name: "OrganicMapsSDK",
            dependencies: ["OrganicMapsCore"],
            swiftSettings: [
                .interoperabilityMode(.Cxx)  // Enable C++ interop
            ]
        ),
        
        .testTarget(
            name: "OrganicMapsSDKTests",
            dependencies: ["OrganicMapsSDK"],
            swiftSettings: [
                .interoperabilityMode(.Cxx)
            ]
        )
    ],
    cxxLanguageStandard: .cxx20
)
```

### 2. C++ API Header for Swift Interop

```cpp
// organicmaps/include/OrganicMapsC++API.hpp
#pragma once

#include <string>
#include <functional>
#include <vector>

// Swift-friendly C++ API definitions
namespace OrganicMapsSDK {
    
struct LatLon {
    double lat;
    double lon;
};

struct POISearchResult {
    std::string name;
    std::string category;
    double lat;
    double lon;
};

struct StorageEvent {
    enum Type { downloadProgress, downloadCompleted, downloadFailed };
    Type type;
    std::string countryId;
    double progress;
};

// Forward declarations
class Framework;
class Storage;

// Framework Management
Framework* CreateFramework();
void DestroyFramework(Framework* framework);

// Map Loading
void LoadMWMFile(Framework* framework, const std::string& path, 
                 std::function<void(bool)> completion);

// Viewport Control  
void SetViewportCenter(Framework* framework, LatLon center, 
                       float zoom, bool animated);

// Search
void SearchPOIs(Framework* framework, const std::string& query,
                std::function<void(std::vector<POISearchResult>)> callback);

// Storage Management
Storage* GetStorage();
void DownloadCountry(Storage* storage, const std::string& countryId,
                     std::function<void(double)> progressCallback,
                     std::function<void(bool, std::string)> completionCallback);

// Event Streaming for Swift async sequences
class StorageEventStream {
public:
    StorageEventStream(Storage* storage);
    bool hasNext();
    StorageEvent next();
};

} // namespace OrganicMapsSDK
```

### 3. Modern Swift API with Direct C++ Interop

```swift
// Sources/OrganicMapsSDK/MapViewModel.swift
import SwiftUI
import CoreLocation
import OrganicMapsCore  // Direct C++ module import

@Observable
@MainActor
public class MapViewModel {
    private var framework: UnsafeMutablePointer<OrganicMapsSDK.Framework>?
    private var storage: UnsafeMutablePointer<OrganicMapsSDK.Storage>?
    
    public var viewport: MapViewport
    public var downloadProgress: [String: Double] = [:]
    
    public init() {
        self.viewport = MapViewport(
            center: CLLocationCoordinate2D(latitude: 0, longitude: 0),
            zoom: 2
        )
        setupFramework()
    }
    
    private func setupFramework() {
        // Direct C++ API calls - no bridging
        framework = OrganicMapsSDK.CreateFramework()
        storage = OrganicMapsSDK.GetStorage()
        
        // Setup callbacks using Swift concurrency
        Task {
            await setupStorageObserver()
        }
    }
    
    // MARK: - Public Swift API
    
    public func loadMWMFile(at path: String) async throws {
        guard let framework = framework else { 
            throw MapError.frameworkNotInitialized 
        }
        
        await withCheckedContinuation { continuation in
            // Direct C++ call with Swift closure
            OrganicMapsSDK.LoadMWMFile(framework, path) { success in
                continuation.resume()
            }
        }
    }
    
    public func setViewport(_ coordinate: CLLocationCoordinate2D, 
                           zoom: Float, 
                           animated: Bool = true) {
        guard let framework = framework else { return }
        
        // Direct C++ struct creation and method call
        let center = OrganicMapsSDK.LatLon(lat: coordinate.latitude, lon: coordinate.longitude)
        OrganicMapsSDK.SetViewportCenter(framework, center, zoom, animated)
        
        viewport = MapViewport(center: coordinate, zoom: zoom)
    }
    
    public func searchPOIs(_ query: String) async -> [POIResult] {
        guard let framework = framework else { return [] }
        
        return await withCheckedContinuation { continuation in
            // C++ search with Swift callback
            OrganicMapsSDK.SearchPOIs(framework, query) { results in
                let swiftResults = results.map { cppResult in
                    POIResult(
                        name: String(cppResult.name),
                        coordinate: CLLocationCoordinate2D(
                            latitude: cppResult.lat,
                            longitude: cppResult.lon
                        ),
                        category: String(cppResult.category)
                    )
                }
                continuation.resume(returning: swiftResults)
            }
        }
    }
    
    public func downloadMap(countryId: String) -> AsyncThrowingStream<Double, Error> {
        AsyncThrowingStream { continuation in
            guard let storage = storage else {
                continuation.finish(throwing: MapError.storageNotInitialized)
                return
            }
            
            // C++ download with progress callback
            OrganicMapsSDK.DownloadCountry(storage, countryId,
                progressCallback: { progress in
                    continuation.yield(progress)
                },
                completionCallback: { success, error in
                    if success {
                        continuation.finish()
                    } else {
                        continuation.finish(throwing: MapError.downloadFailed(error))
                    }
                }
            )
        }
    }
    
    private func setupStorageObserver() async {
        guard let storage = storage else { return }
        
        // C++ observer pattern with Swift async sequence
        let eventStream = OrganicMapsSDK.StorageEventStream(storage)
        while eventStream.hasNext() {
            let event = eventStream.next()
            await MainActor.run {
                handleStorageEvent(event)
            }
        }
    }
    
    private func handleStorageEvent(_ event: OrganicMapsSDK.StorageEvent) {
        switch event.type {
        case .downloadProgress:
            downloadProgress[String(event.countryId)] = event.progress
        case .downloadCompleted:
            downloadProgress.removeValue(forKey: String(event.countryId))
        case .downloadFailed:
            downloadProgress.removeValue(forKey: String(event.countryId))
        }
    }
}

// MARK: - SwiftUI Integration

public struct OrganicMapView: UIViewRepresentable {
    var viewModel: MapViewModel
    
    public init(viewModel: MapViewModel) {
        self.viewModel = viewModel
    }
    
    public func makeUIView(context: Context) -> MapUIView {
        let mapView = MapUIView()
        mapView.setupWithFramework(viewModel.framework)
        return mapView
    }
    
    public func updateUIView(_ mapView: MapUIView, context: Context) {
        // Update view based on ViewModel changes
        mapView.setViewport(viewModel.viewport)
    }
}
```

### 4. Swift Data Models

```swift
// Sources/OrganicMapsSDK/DataModels.swift
import CoreLocation

public struct POIResult {
    public let name: String
    public let coordinate: CLLocationCoordinate2D
    public let category: String
}

public struct MapViewport {
    public let center: CLLocationCoordinate2D
    public let zoom: Float
}

public enum MapError: Error {
    case frameworkNotInitialized
    case storageNotInitialized
    case downloadFailed(String)
}
```

### 5. C++ Implementation Bridge

```cpp
// organicmaps/src/swift_api.cpp
#include "include/OrganicMapsC++API.hpp"
#include "map/framework.hpp"
#include "storage/storage.hpp"
#include "search/search_params.hpp"

namespace OrganicMapsSDK {

Framework* CreateFramework() {
    return &GetFramework(); // Use existing singleton
}

void LoadMWMFile(Framework* framework, const std::string& path, 
                 std::function<void(bool)> completion) {
    try {
        platform::LocalCountryFile localFile = 
            platform::LocalCountryFile::MakeForTesting(path);
        auto result = framework->RegisterMap(localFile);
        completion(result.second == MwmSet::RegResult::Success);
    } catch (...) {
        completion(false);
    }
}

void SetViewportCenter(Framework* framework, LatLon center, 
                       float zoom, bool animated) {
    m2::PointD const centerPoint = mercator::LatLonToMeters({center.lat, center.lon});
    framework->SetViewportCenter(centerPoint, zoom, animated);
}

void SearchPOIs(Framework* framework, const std::string& query,
                std::function<void(std::vector<POISearchResult>)> callback) {
    search::EverywhereSearchParams params;
    params.m_query = query;
    params.m_onResults = [callback](search::Results const & results) {
        std::vector<POISearchResult> swiftResults;
        
        for (auto const & result : results) {
            POISearchResult poi;
            poi.name = result.GetString();
            poi.category = result.GetFeatureType(); 
            auto const center = result.GetFeatureCenter();
            auto const latlon = mercator::MetersToLatLon(center);
            poi.lat = latlon.m_lat;
            poi.lon = latlon.m_lon;
            swiftResults.push_back(poi);
        }
        
        callback(swiftResults);
    };
    
    framework->GetSearchAPI().SearchEverywhere(std::move(params));
}

Storage* GetStorage() {
    return &GetStorage(); // Use existing singleton
}

void DownloadCountry(Storage* storage, const std::string& countryId,
                     std::function<void(double)> progressCallback,
                     std::function<void(bool, std::string)> completionCallback) {
    storage::CountryId const countryIdNative = storage::CountryId(countryId);
    
    // Set up callbacks and start download
    storage->Subscribe([progressCallback, completionCallback](storage::CountryId const & id) {
        storage::NodeAttrs attrs = storage->GetNodeAttrs(id);
        
        if (attrs.m_downloadingProgress.second > 0) {
            double progress = static_cast<double>(attrs.m_downloadingProgress.first) / 
                            static_cast<double>(attrs.m_downloadingProgress.second);
            progressCallback(progress);
        }
        
        if (attrs.m_status == storage::NodeStatus::OnDisk) {
            completionCallback(true, "");
        } else if (attrs.m_status == storage::NodeStatus::Error) {
            completionCallback(false, "Download failed");
        }
    });
    
    storage->DownloadNode(countryIdNative);
}

} // namespace OrganicMapsSDK
```

### 6. iOS Distribution Package

The Package.swift configuration shown above in section 1 handles the complete iOS distribution setup.

## Android Implementation

### 1. Public Kotlin API

```kotlin
// android/src/main/java/YourMapView.kt
package com.yourcompany.yourmapsdk

import android.content.Context
import android.util.AttributeSet
import android.view.SurfaceView
import android.view.SurfaceHolder

class YourMapView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : SurfaceView(context, attrs, defStyleAttr), SurfaceHolder.Callback {
    
    interface OnMapReadyCallback {
        fun onMapReady(mapView: YourMapView)
    }
    
    interface OnPOIClickListener {
        fun onPOIClick(poi: POIResult)
    }
    
    var onMapReadyCallback: OnMapReadyCallback? = null
    var onPOIClickListener: OnPOIClickListener? = null
    
    private var isMapReady = false
    
    init {
        holder.addCallback(this)
        initNative()
    }
    
    // MARK: - Public API
    
    fun loadMWMFile(path: String) {
        if (isMapReady) {
            nativeLoadMWM(path)
        }
    }
    
    fun setCenter(lat: Double, lon: Double, zoom: Float, animated: Boolean = true) {
        if (isMapReady) {
            nativeSetViewport(lat, lon, zoom, animated)
        }
    }
    
    fun searchPOIs(query: String, callback: (List<POIResult>) -> Unit) {
        if (isMapReady) {
            nativeSearch(query) { resultsJson ->
                val results = parsePOIResults(resultsJson)
                callback(results)
            }
        }
    }
    
    fun downloadMap(countryId: String, progressCallback: (Double) -> Unit, completionCallback: (Result<Unit>) -> Unit) {
        if (isMapReady) {
            nativeDownloadCountry(countryId, 
                { progress -> progressCallback(progress) },
                { success, error -> 
                    if (success) {
                        completionCallback(Result.success(Unit))
                    } else {
                        completionCallback(Result.failure(RuntimeException(error)))
                    }
                }
            )
        }
    }
    
    // MARK: - SurfaceHolder.Callback
    
    override fun surfaceCreated(holder: SurfaceHolder) {
        nativeSurfaceCreated(holder.surface)
    }
    
    override fun surfaceChanged(holder: SurfaceHolder, format: Int, width: Int, height: Int) {
        nativeSurfaceChanged(width, height)
        if (!isMapReady) {
            isMapReady = true
            onMapReadyCallback?.onMapReady(this)
        }
    }
    
    override fun surfaceDestroyed(holder: SurfaceHolder) {
        nativeSurfaceDestroyed()
        isMapReady = false
    }
    
    // MARK: - Native Methods
    
    private external fun initNative()
    private external fun nativeLoadMWM(path: String)
    private external fun nativeSetViewport(lat: Double, lon: Double, zoom: Float, animated: Boolean)
    private external fun nativeSearch(query: String, callback: (String) -> Unit)
    private external fun nativeDownloadCountry(countryId: String, progressCallback: (Double) -> Unit, completionCallback: (Boolean, String?) -> Unit)
    private external fun nativeSurfaceCreated(surface: Any)
    private external fun nativeSurfaceChanged(width: Int, height: Int)
    private external fun nativeSurfaceDestroyed()
    
    companion object {
        init {
            System.loadLibrary("yourmapsdk")
        }
    }
}

// MARK: - Data Models

data class POIResult(
    val name: String,
    val latitude: Double,
    val longitude: Double,
    val category: String,
    val address: String?
)

data class MapViewport(
    val centerLatitude: Double,
    val centerLongitude: Double,
    val zoom: Float
)
```

### 2. JNI Wrapper

```cpp
// android/src/main/cpp/wrapper.cpp
#include <jni.h>
#include <android/native_window_jni.h>

// Organic Maps headers
#include "map/framework.hpp"
#include "storage/storage.hpp"
#include "search/search_params.hpp"
#include "android/jni/com/mapswithme/core/jni_helper.hpp"

namespace {
    Framework* g_framework = nullptr;
    storage::Storage* g_storage = nullptr;
    ANativeWindow* g_window = nullptr;
}

extern "C" {

JNIEXPORT void JNICALL
Java_com_yourcompany_yourmapsdk_YourMapView_initNative(JNIEnv *env, jobject thiz) {
    // Initialize Organic Maps framework
    if (!g_framework) {
        g_framework = &GetFramework();
        g_storage = &GetStorage();
    }
}

JNIEXPORT void JNICALL
Java_com_yourcompany_yourmapsdk_YourMapView_nativeLoadMWM(JNIEnv *env, jobject thiz, jstring path) {
    if (!g_framework) return;
    
    std::string pathStr = jni::ToNativeString(env, path);
    
    // Register MWM file with framework
    platform::LocalCountryFile localFile = platform::LocalCountryFile::MakeForTesting(pathStr);
    g_framework->RegisterMap(localFile);
}

JNIEXPORT void JNICALL
Java_com_yourcompany_yourmapsdk_YourMapView_nativeSetViewport(JNIEnv *env, jobject thiz, 
                                                              jdouble lat, jdouble lon, 
                                                              jfloat zoom, jboolean animated) {
    if (!g_framework) return;
    
    m2::PointD const center = mercator::LatLonToMeters({lat, lon});
    g_framework->SetViewportCenter(center, zoom, animated);
}

JNIEXPORT void JNICALL
Java_com_yourcompany_yourmapsdk_YourMapView_nativeSearch(JNIEnv *env, jobject thiz, 
                                                         jstring query, jobject callback) {
    if (!g_framework) return;
    
    std::string queryStr = jni::ToNativeString(env, query);
    
    search::EverywhereSearchParams params;
    params.m_query = queryStr;
    params.m_onResults = [env, callback](search::Results const & results) {
        // Convert results to JSON string
        std::string jsonResults = ConvertSearchResultsToJson(results);
        
        // Call Java callback
        jni::GetJavaMethodID(env, callback, "invoke", "(Ljava/lang/String;)V");
        jstring jJsonResults = jni::ToJavaString(env, jsonResults);
        env->CallVoidMethod(callback, method, jJsonResults);
    };
    
    g_framework->GetSearchAPI().SearchEverywhere(std::move(params));
}

JNIEXPORT void JNICALL
Java_com_yourcompany_yourmapsdk_YourMapView_nativeDownloadCountry(JNIEnv *env, jobject thiz,
                                                                  jstring countryId,
                                                                  jobject progressCallback,
                                                                  jobject completionCallback) {
    if (!g_storage) return;
    
    std::string countryIdStr = jni::ToNativeString(env, countryId);
    storage::CountryId const countryIdNative = storage::CountryId(countryIdStr);
    
    // Set up progress callback
    g_storage->SetDownloadingPolicy(&policy);
    g_storage->Subscribe([env, progressCallback, completionCallback](storage::CountryId const & id) {
        storage::NodeAttrs attrs = g_storage->GetNodeAttrs(id);
        
        if (attrs.m_downloadingProgress.second > 0) {
            double progress = static_cast<double>(attrs.m_downloadingProgress.first) / 
                            static_cast<double>(attrs.m_downloadingProgress.second);
            
            // Call progress callback
            jmethodID progressMethod = jni::GetJavaMethodID(env, progressCallback, "invoke", "(D)V");
            env->CallVoidMethod(progressCallback, progressMethod, progress);
        }
        
        if (attrs.m_status == storage::NodeStatus::OnDisk) {
            // Download completed successfully
            jmethodID completionMethod = jni::GetJavaMethodID(env, completionCallback, "invoke", "(ZLjava/lang/String;)V");
            env->CallVoidMethod(completionCallback, completionMethod, JNI_TRUE, nullptr);
        } else if (attrs.m_status == storage::NodeStatus::Error) {
            // Download failed
            jstring errorMsg = jni::ToJavaString(env, "Download failed");
            jmethodID completionMethod = jni::GetJavaMethodID(env, completionCallback, "invoke", "(ZLjava/lang/String;)V");
            env->CallVoidMethod(completionCallback, completionMethod, JNI_FALSE, errorMsg);
        }
    }, this);
    
    // Start download
    g_storage->DownloadNode(countryIdNative);
}

JNIEXPORT void JNICALL
Java_com_yourcompany_yourmapsdk_YourMapView_nativeSurfaceCreated(JNIEnv *env, jobject thiz, jobject surface) {
    g_window = ANativeWindow_fromSurface(env, surface);
    
    if (g_framework && g_window) {
        // Create graphics context with native window
        g_framework->CreateDrapeEngine(g_window);
    }
}

JNIEXPORT void JNICALL
Java_com_yourcompany_yourmapsdk_YourMapView_nativeSurfaceChanged(JNIEnv *env, jobject thiz, 
                                                                 jint width, jint height) {
    if (g_framework) {
        g_framework->OnSize(width, height);
    }
}

JNIEXPORT void JNICALL
Java_com_yourcompany_yourmapsdk_YourMapView_nativeSurfaceDestroyed(JNIEnv *env, jobject thiz) {
    if (g_window) {
        ANativeWindow_release(g_window);
        g_window = nullptr;
    }
    
    if (g_framework) {
        g_framework->DestroyDrapeEngine();
    }
}

} // extern "C"
```

### 3. Android build.gradle

```gradle
// android/build.gradle
apply plugin: 'com.android.library'
apply plugin: 'kotlin-android'

android {
    namespace 'com.yourcompany.yourmapsdk'
    compileSdk 34
    
    defaultConfig {
        minSdk 21
        targetSdk 34
        
        consumerProguardFiles "consumer-rules.pro"
        
        externalNativeBuild {
            cmake {
                cppFlags '-std=c++20', '-fexceptions', '-frtti'
                arguments '-DPLATFORM_ANDROID=ON',
                         '-DSKIP_TESTS=ON',
                         '-DSKIP_QT_GUI=ON',
                         '-DSKIP_TOOLS=ON'
                targets 'yourmapsdk'
            }
        }
        
        ndk {
            abiFilters 'arm64-v8a', 'armeabi-v7a', 'x86_64'
        }
    }
    
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
    
    externalNativeBuild {
        cmake {
            path file('../../CMakeLists.txt')
            version '3.22.1'
            buildStagingDirectory './nativeOutputs'
        }
    }
    
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_17
        targetCompatibility JavaVersion.VERSION_17
    }
    
    kotlinOptions {
        jvmTarget = '17'
    }
}

dependencies {
    implementation 'androidx.core:core-ktx:1.12.0'
    implementation 'androidx.appcompat:appcompat:1.6.1'
    
    // Test dependencies
    testImplementation 'junit:junit:4.13.2'
    androidTestImplementation 'androidx.test.ext:junit:1.1.5'
}
```

### 4. Android CMakeLists.txt

```cmake
# android/CMakeLists.txt
if(NOT PLATFORM_ANDROID)
    return()
endif()

# Create Android shared library
add_library(yourmapsdk SHARED
    src/main/cpp/wrapper.cpp
    src/main/cpp/jni_helper.cpp
    # Add other JNI wrapper files
)

target_link_libraries(yourmapsdk
    # Core Organic Maps libraries  
    drape_frontend
    map
    indexer
    storage
    search
    routing
    platform
    
    # Android libraries
    android
    log
    EGL
    GLESv2
)

# Set library properties
set_target_properties(yourmapsdk PROPERTIES
    ANDROID_STL c++_static
)
```

## Resource Management

### 1. Asset Handling

```bash
# shared/assets/
├── fonts/
│   ├── icudt75l.dat           # Unicode data (18MB)
│   └── NotoSans-Regular.ttf   # Default font
├── shaders/
│   ├── vulkan/                # Vulkan shaders
│   ├── gles/                  # OpenGL ES shaders
│   └── metal/                 # Metal shaders (iOS)
├── config/
│   ├── classificator.txt      # Feature classification
│   ├── types.txt             # Type definitions
│   └── drules_*.txt          # Drawing rules
└── maps/
    ├── World.mwm             # World coastlines (12MB)
    ├── WorldCoasts.mwm       # Detailed coastlines (5MB)
    └── countries/            # Country-specific .mwm files
```

### 2. iOS Asset Integration

```swift
// ios/Sources/AssetManager.swift
public class AssetManager {
    private static let bundle = Bundle(for: AssetManager.self)
    
    public static func worldMWMPath() -> String? {
        return bundle.path(forResource: "World", ofType: "mwm")
    }
    
    public static func countryMWMPath(_ countryId: String) -> String? {
        return bundle.path(forResource: countryId, ofType: "mwm")
    }
    
    public static func fontsDirectory() -> String? {
        return bundle.path(forResource: "fonts", ofType: nil)
    }
}
```

### 3. Android Asset Integration

```kotlin
// android/src/main/java/AssetManager.kt
object AssetManager {
    fun copyAssetToFile(context: Context, assetPath: String, outputFile: File) {
        context.assets.open(assetPath).use { input ->
            outputFile.outputStream().use { output ->
                input.copyTo(output)
            }
        }
    }
    
    fun getWorldMWMPath(context: Context): String {
        val worldFile = File(context.filesDir, "World.mwm")
        if (!worldFile.exists()) {
            copyAssetToFile(context, "maps/World.mwm", worldFile)
        }
        return worldFile.absolutePath
    }
}
```

## Distribution

### 1. iOS Distribution (Swift Package Manager)

```swift
// Package.swift
// swift-tools-version:5.9
import PackageDescription

let package = Package(
    name: "YourMapSDK",
    platforms: [
        .iOS(.v12)
    ],
    products: [
        .library(
            name: "YourMapSDK",
            targets: ["YourMapSDK"]
        ),
    ],
    targets: [
        .target(
            name: "YourMapSDK",
            dependencies: ["OrganicMapsCore"],
            path: "Sources",
            resources: [
                .copy("Resources/fonts"),
                .copy("Resources/shaders"),
                .copy("Resources/config"),
                .copy("Resources/maps")
            ],
            cxxSettings: [
                .headerSearchPath("../organicmaps"),
                .define("OMIM_OS_NAME", to: "ios")
            ],
            linkerSettings: [
                .linkedFramework("UIKit"),
                .linkedFramework("CoreLocation"),
                .linkedFramework("OpenGLES"),
                .linkedFramework("Metal"),
                .linkedFramework("CoreGraphics"),
                .linkedLibrary("c++")
            ]
        ),
        .binaryTarget(
            name: "OrganicMapsCore",
            path: "Frameworks/OrganicMapsCore.xcframework"
        )
    ],
    cLanguageStandard: .c17,
    cxxLanguageStandard: .cxx20
)
```

#### Alternative: XCFramework Distribution

For pre-built distribution, you can also use XCFramework:

```swift
// Package.swift for XCFramework distribution
let package = Package(
    name: "YourMapSDK",
    platforms: [.iOS(.v12)],
    products: [
        .library(name: "YourMapSDK", targets: ["YourMapSDK"])
    ],
    targets: [
        .binaryTarget(
            name: "YourMapSDK",
            url: "https://github.com/yourcompany/YourMapSDK/releases/download/1.0.0/YourMapSDK.xcframework.zip",
            checksum: "sha256-checksum-here"
        )
    ]
)
```

### 2. Android Distribution (Maven)

```gradle
// android/publish.gradle
apply plugin: 'maven-publish'
apply plugin: 'signing'

publishing {
    publications {
        release(MavenPublication) {
            from components.release
            
            groupId = 'com.yourcompany'
            artifactId = 'yourmapsdk'
            version = '1.0.0'
            
            pom {
                name = 'YourMapSDK'
                description = 'Offline mapping SDK with .mwm format support'
                url = 'https://github.com/yourcompany/YourMapSDK'
                
                licenses {
                    license {
                        name = 'The Apache License, Version 2.0'
                        url = 'http://www.apache.org/licenses/LICENSE-2.0.txt'
                    }
                }
                
                developers {
                    developer {
                        id = 'yourcompany'
                        name = 'Your Company'
                        email = 'contact@yourcompany.com'
                    }
                }
            }
        }
    }
}
```

## Build and Development Workflow

### 1. Initial Setup

```bash
# Clone your SDK repository
git clone https://github.com/yourcompany/YourMapSDK.git
cd YourMapSDK

# Initialize and update submodules
git submodule update --init --recursive

# Install dependencies (macOS)
brew install cmake ninja

# For Android development
export ANDROID_NDK_ROOT=/path/to/android-ndk
```

### 2. iOS Development

```bash
# Build iOS framework
mkdir build-ios
cd build-ios
cmake .. -G Xcode -DPLATFORM_IOS=ON
xcodebuild -project YourMapSDK.xcodeproj -scheme YourMapSDK_iOS -destination "generic/platform=iOS"

# Test in iOS Simulator
xcodebuild -project YourMapSDK.xcodeproj -scheme YourMapSDK_iOS -destination "platform=iOS Simulator,name=iPhone 15"
```

### 3. Android Development

```bash
# Build Android library
cd android
./gradlew assembleDebug

# Run tests
./gradlew test

# Generate AAR
./gradlew bundleReleaseAar
```

### 4. Update Submodule

```bash
# Update to latest Organic Maps
git submodule update --remote organicmaps

# Test with new version
# Build and test both platforms

# Commit the update
git add organicmaps
git commit -m "Update Organic Maps to latest version"
```

## Usage Examples

### iOS Usage with Modern Swift

#### SwiftUI Example

```swift
import SwiftUI
import OrganicMapsSDK

struct ContentView: View {
    @State private var mapViewModel = MapViewModel()
    @State private var searchQuery = ""
    @State private var searchResults: [POIResult] = []
    
    var body: some View {
        NavigationView {
            VStack {
                // Modern SwiftUI Map Integration
                OrganicMapView(viewModel: mapViewModel)
                    .onAppear {
                        Task {
                            await loadInitialMap()
                        }
                    }
                
                // Search Interface
                HStack {
                    TextField("Search places...", text: $searchQuery)
                        .textFieldStyle(RoundedBorderTextFieldStyle())
                    
                    Button("Search") {
                        Task {
                            await performSearch()
                        }
                    }
                }
                .padding()
                
                // Search Results
                List(searchResults, id: \.name) { poi in
                    VStack(alignment: .leading) {
                        Text(poi.name)
                            .font(.headline)
                        Text(poi.category)
                            .font(.caption)
                            .foregroundColor(.secondary)
                    }
                    .onTapGesture {
                        mapViewModel.setViewport(poi.coordinate, zoom: 15)
                    }
                }
            }
            .navigationTitle("Organic Maps")
        }
    }
    
    private func loadInitialMap() async {
        do {
            // Load world map from bundle
            if let worldPath = Bundle.main.path(forResource: "World", ofType: "mwm") {
                try await mapViewModel.loadMWMFile(at: worldPath)
            }
            
            // Set initial location (San Francisco)
            mapViewModel.setViewport(
                CLLocationCoordinate2D(latitude: 37.7749, longitude: -122.4194), 
                zoom: 12
            )
        } catch {
            print("Failed to load map: \(error)")
        }
    }
    
    private func performSearch() async {
        searchResults = await mapViewModel.searchPOIs(searchQuery)
    }
}

// Map download with progress tracking
struct MapDownloadView: View {
    @State private var mapViewModel = MapViewModel()
    @State private var downloadProgress: Double = 0
    @State private var isDownloading = false
    
    var body: some View {
        VStack {
            Button("Download Germany Map") {
                downloadGermanyMap()
            }
            .disabled(isDownloading)
            
            if isDownloading {
                ProgressView("Downloading...", value: downloadProgress, total: 1.0)
                    .padding()
            }
        }
    }
    
    private func downloadGermanyMap() {
        isDownloading = true
        
        Task {
            do {
                for try await progress in mapViewModel.downloadMap(countryId: "Germany") {
                    await MainActor.run {
                        downloadProgress = progress
                    }
                }
                await MainActor.run {
                    isDownloading = false
                    downloadProgress = 0
                }
            } catch {
                await MainActor.run {
                    isDownloading = false
                    print("Download failed: \(error)")
                }
            }
        }
    }
}
```

#### UIKit Example

```swift
import UIKit
import OrganicMapsSDK

class MapViewController: UIViewController {
    private let mapViewModel = MapViewModel()
    private var mapView: OrganicMapView!
    
    override func viewDidLoad() {
        super.viewDidLoad()
        setupMapView()
        
        Task {
            await loadInitialContent()
        }
    }
    
    private func setupMapView() {
        mapView = OrganicMapView(viewModel: mapViewModel)
        mapView.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(mapView)
        
        NSLayoutConstraint.activate([
            mapView.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor),
            mapView.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            mapView.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            mapView.bottomAnchor.constraint(equalTo: view.bottomAnchor)
        ])
    }
    
    private func loadInitialContent() async {
        do {
            // Load world map
            if let worldPath = Bundle.main.path(forResource: "World", ofType: "mwm") {
                try await mapViewModel.loadMWMFile(at: worldPath)
            }
            
            // Set initial viewport
            await MainActor.run {
                mapViewModel.setViewport(
                    CLLocationCoordinate2D(latitude: 37.7749, longitude: -122.4194),
                    zoom: 12
                )
            }
        } catch {
            print("Failed to setup map: \(error)")
        }
    }
    
    @IBAction private func searchRestaurants() {
        Task {
            let results = await mapViewModel.searchPOIs("restaurant")
            await MainActor.run {
                print("Found \(results.count) restaurants")
                // Update UI with results
            }
        }
    }
}
```

### Android Usage

```kotlin
class MainActivity : AppCompatActivity() {
    private lateinit var mapView: YourMapView
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        mapView = findViewById(R.id.map_view)
        mapView.onMapReadyCallback = object : YourMapView.OnMapReadyCallback {
            override fun onMapReady(mapView: YourMapView) {
                // Load world map
                val worldPath = AssetManager.getWorldMWMPath(this@MainActivity)
                mapView.loadMWMFile(worldPath)
                
                // Set initial location
                mapView.setCenter(37.7749, -122.4194, 12f)
            }
        }
        
        mapView.onPOIClickListener = object : YourMapView.OnPOIClickListener {
            override fun onPOIClick(poi: POIResult) {
                Toast.makeText(this@MainActivity, "Selected: ${poi.name}", Toast.LENGTH_SHORT).show()
            }
        }
    }
    
    private fun searchRestaurants() {
        mapView.searchPOIs("restaurant") { results ->
            println("Found ${results.size} restaurants")
        }
    }
}
```

## Conclusion

This modern approach provides exceptional benefits for a new mapping SDK project:

### ✅ **Major Advantages**

#### **Modern Swift C++ Interoperability**
- **Zero bridging overhead** - Direct Swift ↔ C++ function calls
- **No Objective-C++ complexity** - Clean Swift-first architecture
- **Native Swift concurrency** - async/await, AsyncThrowingStream support
- **SwiftUI ready** - @Observable macro integration (iOS 17+)
- **Full Xcode tooling** - Code completion, debugging, and refactoring across languages

#### **Technical Benefits**
- **Complete .mwm ecosystem** with highly efficient offline maps
- **Ongoing updates** from Organic Maps development
- **Production-tested code** with millions of users
- **Cross-platform consistency** between iOS and Android
- **Easy distribution** via Swift Package Manager and Maven

#### **Developer Experience**
- **Type-safe APIs** with Swift/C++ interop guarantees
- **Modern async patterns** for all operations
- **Simplified build process** with SPM C++ interop support
- **Better error handling** with Swift's Result and Error types

### ⚠️ **Considerations**
- **Swift 5.9+ requirement** - Needs modern Swift toolchain
- **C++ interop versioning** - Breaking changes require major version bumps
- **Binary size** - ~50-100MB per platform for full functionality
- **Development expertise** - Requires knowledge of C++ and mobile development

### 🚀 **Next Steps**
1. Set up the basic project structure as outlined
2. Start with minimal API surface (map display + .mwm loading)
3. Gradually add features (search, routing, downloads)
4. Create comprehensive documentation and examples
5. Set up CI/CD for automated building and testing
6. Publish to Swift Package Manager and Maven Central

This architecture provides a solid foundation for a powerful offline mapping SDK that leverages the best of Organic Maps technology while providing clean, maintainable APIs for your applications.
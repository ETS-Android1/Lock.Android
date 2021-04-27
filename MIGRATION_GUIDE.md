# Migration Guide
This guide will help you migrate Lock.Android from version v2.x.x to version v3.x.x. 

## About this release
The new version makes use of the latest Auth0.Android SDK, bringing improvements such as:
 - Open ID Connect compliant practices.
 - ID token verification for Web Authentication flows.
 - A new customizable networking stack.
 
 Some of these features were previously available, but only enforced when the "OIDC" flag was explicitly enabled.  

## New requirements
Using the latest core SDK comes with new constraints. Your Android application will need to:
- Require a minimum Android version of 21 and above.
- Target Java version 8 and above.

Here’s what you need in build.gradle to target Java 8 byte code for the Android and Kotlin plugins respectively.

```groovy
android {
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }

    kotlinOptions {
        jvmTarget = '1.8'
    }
}
```

## Changes to the Public API
As part of removing legacy APIs or authentication flows no longer recommended for mobile clients, the following features are no longer available:

- Web Authentication flow using a WebView component instead of an external Browser. Please refer to [this blog post](https://auth0.com/blog/google-blocks-oauth-requests-from-embedded-browsers/) for additional information.
- Web Authentication flow using a response type other than "code".
- Authentication API methods categorized as Legacy in the [API docs](https://auth0.com/docs/api/authentication).

Continue reading for the detail of classes and methods that were impacted.

### Removed classes
- `VoidCallback` is no longer available. Please, use `Callback<Void, AuthenticationException>` instead.

### Removed methods

#### From class `Lock.Builder`
- Removed `public Builder useBrowser(boolean)`. The library will always use a third party browser app instead of a Web View to authenticate. No replacement is available.
- Removed `public Builder useImplicitGrant(boolean)`. The library will always use the "Proof Key for Code Exchange" (PKCE) flow. Your application must be configured with the type "Native" and the "OIDC Conformant" switch ON. No replacement is available.
- Removed `public Builder withAuthButtonSize(int)`. Social buttons will always have a "large button" style. No replacement is available. 

#### From class `PasswordlessLock.Builder`
- Removed `public Builder useBrowser(boolean)`. The library will always use a third party browser app instead of a Web View to authenticate. No replacement is available.
- Removed `public Builder useImplicitGrant(boolean)`. The library will always use the "Proof Key for Code Exchange" (PKCE) flow. Your application must be configured with the type "Native" and the "OIDC Conformant" switch ON. No replacement is available.
- Removed `public Builder withAuthButtonSize(int)`. Social buttons will always have a "large button" style. No replacement is available. 

#### From `Auth0` class
- Removed `setOIDCConformant(boolean)`. The library will only use Open ID Connect compliant flows from now on, this cannot be turned off.
- Removed `setLoggingEnabled(boolean)`. The ability to turn on the networking logs has been removed. If you need to inspect the traffic, take a look at the [Network Profiler](https://developer.android.com/studio/profile/network-profiler) tool. 

### Changed methods

#### From cass `Lock.Builder`
- Changed `public Builder withAuthenticationParameters(@NonNull Map<String, Object> authenticationParameters)` to `public Builder withAuthenticationParameters(@NonNull Map<String, String> authenticationParameters)`. Request parameters must be specified as String key/values.

#### From cass `PasswordlessLock.Builder`
- Changed `public Builder withAuthenticationParameters(@NonNull Map<String, Object> authenticationParameters)` to `public Builder withAuthenticationParameters(@NonNull Map<String, String> authenticationParameters)`. Request parameters must be specified as String key/values.

### Changes to the underlying SDK

The core SDK has been updated to the version 2+. Since this is exposed as an API scoped dependency, if you were using any of the classes or methods that changed in the new major release (e.g. the `WebAuthProvider` class), you might need to update your code. Follow the [Auth0.Android Migration Guide](https://github.com/auth0/Auth0.Android/blob/main/V2_MIGRATION_GUIDE.md) to assess the impact. 

## Changes in behavior

### Lock lifecycle

The widget registers a Broadcast Listener to expect and handle the different lifecycle events. The listener is registered as soon as a new instance of `Lock` or `PasswordlessLock` is created with the corresponding Builder class, and the listener is unregistered when the `onDestroy` method is invoked. Forgetting to call this method would retain unnecessary resources after the authentication is complete and the widget is no longer required, or cause the callback to receive duplicated calls. 

In case you are not currently calling it, make sure to update your code adding the `lock?.onDestroy(this)` call.

```kotlin
class MyActivity : AppCompatActivity() {

  private var lock: Lock? = null

  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    
    val account = Auth0(this)
    // Create a reusable Lock instance
    lock = Lock.newBuilder(account, callback)
      // Customize Lock
      // .withScheme("myapp")
      .build(this)
  }

  private fun launchLock() {
    // Invoke as many times as needed
    val intent = lock!!.newIntent(this)
    startActivity(intent)
  }
 
  override fun onDestroy() {
      super.onDestroy()
      // Release Lock resources
      lock?.onDestroy(this)
  }
}
```

### Non-recoverable errors

The `LockCallback` will get its `onError` method invoked when an [Auth0 Rule](https://auth0.com/docs/rules) returns an `Error` or `UnauthorizedError`. This was previously handled internally by Lock, causing it to display an orange toast with a generic failure message. From this release on, if you are using Auth0 Rules and throwing custom errors, you should obtain the _cause_ of the exception and read the code or description values to understand what went wrong.  
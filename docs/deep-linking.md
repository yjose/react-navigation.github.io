---
id: deep-linking
title: Deep linking
sidebar_label: Deep linking
---

In this guide we will set up our app to handle external URIs.

### Deep-link integration

To handle incoming links, we need to handle 2 scenarios:

1. If the app wasn't previously open, we need to set the initial state based on the link
2. If the app was already open, we need to update the state to reflect the incoming link

To handle a deep link, we need to translate it to a valid navigation state. The library exports a `getStateFromPath` utility to convert a URL to a state object.

For example, the path `/rooms/chat?user=jane` will be translated to a state object like this:

```js
{
  routes: [
    {
      name: 'rooms',
      state: {
        routes: [
          {
            name: 'chat',
            params: { user: 'jane' },
          },
        ],
      },
    },
  ],
}
```

The `useLinking` hooks makes it easier to handle incoming links:

```js
import {
  NavigationNativeContainer,
  useLinking,
} from '@react-navigation/native';

function App() {
  const ref = React.useRef();

  const { getInitialState } = useLinking(ref, {
    prefixes: ['https://mychat.com', 'mychat://'],
  });

  const [isReady, setIsReady] = React.useState(false);
  const [initialState, setInitialState] = React.useState();

  React.useEffect(() => {
    getInitialState()
      .catch(() => {})
      .then(state => {
        if (state !== undefined) {
          setInitialState(state);
        }

        setIsReady(true);
      });
  }, [getInitialState]);

  if (!isReady) {
    return null;
  }

  return (
    <NavigationNativeContainer initialState={initialState} ref={ref}>
      {/* content */}
    </NavigationNativeContainer>
  );
}
```

Often, directly translating path segments to route names may not be the expected behavior. For example, you might want to parse the path `/feed/latest` to something like:

```js
{
  routes: [
    {
      name: 'Chat',
      params: {
        sort: 'latest',
      },
    },
  ];
}
```

You can specify a separate `config` option to control how the deep link is parsed to suit your needs.

```js
const { getInitialState } = useLinking(ref, {
  prefixes: ['https://mychat.com', 'mychat://'],
  config: {
    Chat: 'feed/:sort',
  },
});
```

Here `Chat` is the name of the screen that handles this URL. The navigator will need to have a `Chat` screen which handles a `sort` param for the route:

You can also customize how params are parsed, for example, if you parse the path `/item/42` as `item/:id`, the param `id` will be parsed as string by default. But you can customize it by passing a function:

```js
{
  Catalog: {
    path: 'item/:id',
    parse: {
      id: Number,
    },
  },
}
```

This will result in something like:

```js
{
  routes: [
    {
      name: 'Catalog',
      params: {
        id: 42,
      },
    },
  ];
}
```

If this doesn't satisfy your use case, The hook also accepts a `getStateFromPath` option where you can provide a custom function to convert the URL to a valid state object for more advanced use cases.

## Set up with Expo projects

First, you will want to specify a url scheme for your app. This corresponds to the string before `://` in a url, so if your scheme is `mychat` then a link to your app would be `mychat://`. The scheme only applies to standalone apps and you need to re-build the standalone app for the change to take effect. In the Expo client app you can deep link using `exp://ADDRESS:PORT` where `ADDRESS` is often `127.0.0.1` and `PORT` is often `19000` - the URL is printed when you run `expo start`. If you want to test with your custom scheme you will need to run `expo build:ios -t simulator` or `expo build:android` and install the resulting binaries in your emulators. You can register for a scheme in your `app.json` by adding a string under the scheme key:

```json
{
  "expo": {
    "scheme": "mychat"
  }
}
```

### URI Prefix

Next, let's configure our navigation container to extract the path from the app's incoming URI.

```js
import * as Expo from 'expo';

const prefix = Expo.Linking.makeUrl('/');

function App() {
  const ref = React.useRef();

  const { getInitialState } = useLinking(ref, {
    prefixes: [prefix],
  });

  const [isReady, setIsReady] = React.useState(false);
  const [initialState, setInitialState] = React.useState();

  React.useEffect(() => {
    getInitialState()
      .catch(() => {})
      .then(state => {
        if (state !== undefined) {
          setInitialState(state);
        }

        setIsReady(true);
      });
  }, [getInitialState]);

  if (!isReady) {
    return null;
  }

  return (
    <NavigationNativeContainer initialState={initialState} ref={ref}>
      {/* content */}
    </NavigationNativeContainer>
  );
}
```

The reason that is necessary to use `Expo.Linking.makeUrl` is that the scheme will differ depending on whether you're in the client app or in a standalone app.

### Test deep linking on iOS

To test the URI on the simulator in the Expo client app, run the following:

```sh
xcrun simctl openurl booted [ put your URI prefix in here ]

# for example

xcrun simctl openurl booted exp://127.0.0.1:19000/--/chat/jane
```

### Test deep linking on Android

To test the intent handling in Android (Expo client app ), run the following:

```sh
adb shell am start -W -a android.intent.action.VIEW -d "[ put your URI prefix in here ]" com.simpleapp

# for example

adb shell am start -W -a android.intent.action.VIEW -d "exp://127.0.0.1:19000/--/chat/jane" com.simpleapp
```

Read the [Expo linking guide](https://docs.expo.io/versions/latest/guides/linking.html) for more information about how to configure linking in projects built with Expo.

## Set up with `react-native init` projects

### iOS

Let's configure the native iOS app to open based on the `mychat://` URI scheme.

In `SimpleApp/ios/SimpleApp/AppDelegate.m`:

```objc
// Add the header at the top of the file:
#import <React/RCTLinkingManager.h>

// Add this above the `@end`:
- (BOOL)application:(UIApplication *)application openURL:(NSURL *)url
  sourceApplication:(NSString *)sourceApplication annotation:(id)annotation
{
  return [RCTLinkingManager application:application openURL:url
                      sourceApplication:sourceApplication annotation:annotation];
}
```

In Xcode, open the project at `SimpleApp/ios/SimpleApp.xcodeproj`. Select the project in sidebar and navigate to the info tab. Scroll down to "URL Types" and add one. In the new URL type, set the identifier and the url scheme to your desired url scheme.

![Xcode project info URL types with mychat added](/docs/assets/deep-linking/xcode-linking.png)

Now you can press play in Xcode, or re-build on the command line:

```sh
react-native run-ios
```

To test the URI on the simulator, run the following:

```sh
xcrun simctl openurl booted mychat://chat/jane
```

To test the URI on a real device, open Safari and type `mychat://chat/jane`.

### Android

To configure the external linking in Android, you can create a new intent in the manifest.

In `SimpleApp/android/app/src/main/AndroidManifest.xml`, do these followings adjustments:

1. Set `launchMode` of `MainActivity` to `singleTask` in order to receive intent on existing `MainActivity`. It is useful if you want to perform navigation using deep link you have been registered - [details](http://developer.android.com/training/app-indexing/deep-linking.html#adding-filters)
2. Add the new `intent-filter` inside the `MainActivity` entry with a `VIEW` type action:

```xml
<activity
    android:name=".MainActivity"
    android:launchMode="singleTask">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="mychat" />
    </intent-filter>
</activity>
```

Now, re-install the app:

```sh
react-native run-android
```

To test the intent handling in Android, run the following:

```sh
adb shell am start -W -a android.intent.action.VIEW -d "mychat://chat/jane" com.simpleapp
```

## Hybrid iOS Applications (Skip for RN only projects)

If you're using React Navigation within a hybrid app - an iOS app that has both Swift/ObjC and React Native parts - you may be missing the `RCTLinkingIOS` subspec in your Podfile, which is installed by default in new RN projects. To add this, ensure your Podfile looks like the following:

```pod
 pod 'React', :path => '../node_modules/react-native', :subspecs => [
    . . . // other subspecs
    'RCTLinkingIOS',
    . . .
  ]
```

## Disable deep linking

If you're using React Navigation within a hybrid app - an iOS app that has both Swift/ObjC and React Native parts - you may be missing the `RCTLinkingIOS` subspec in your Podfile, which is installed by default in new RN projects. To add this, ensure your Podfile looks like the following:

```pod
 pod 'React', :path => '../node_modules/react-native', :subspecs => [
    . . . // other subspecs
    'RCTLinkingIOS',
    . . .
  ]
```

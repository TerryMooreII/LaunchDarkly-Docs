---
title: "C/C++ (client-side) SDK Reference"
excerpt: ""
---
This reference guide documents basic usage of our C client-side SDK, and explains in detail how features work. If you want to dig even deeper, our SDKs are open source-- head to our [C SDK GitHub repository](https://github.com/launchdarkly/c-client-sdk) to look under the hood. The complete API reference is available [here](https://github.com/launchdarkly/c-client-sdk/blob/master/DOCS.md). Additionally you can clone and run a [sample application](https://github.com/launchdarkly/hello-c-client) using this SDK.

The C / C++ SDK supports Windows and POSIX environments (Linux, OSX, BSD). The networking library `libcurl` is a universal dependency.
<Callout intent="alert">
  <Callout.Title>For use in desktop / embedded client applications</Callout.Title>
   <Callout.Description>The LaunchDarkly C / C++ SDK is designed primarily for use in *single user* desktop and embedded applications. It follows the client-side LaunchDarkly model for single-user contexts (much like our mobile or JavaScript SDKs)-- a network call must be made to change user contexts, so changing users should be infrequent. It is not intended for use in multi-user systems such as web servers. Learn more about our [client-side and server-side SDKs](./client-side-and-server-side).
For using LaunchDarkly in C / C++ server-side applications, refer to our [C / C++ server-side SDK](./c-server-sdk-reference).</Callout.Description>
</Callout>

## Getting started
Building on top of our [Quickstart](./getting-started) guide, the following steps will get you started with using the LaunchDarkly SDK in your C application.

Unlike other LaunchDarkly SDKs, the C SDK has no installation steps. To get started, clone [the GitHub repository](https://github.com/launchdarkly/c-client-sdk) or download a release archive from the [GitHub Releases](https://github.com/launchdarkly/c-client-sdk/releases) page. Two makefiles exist in this repository as a starting point for integrating the SDK into your application. You may need to modify the `Makefile` to support your specific use case.

The default POSIX `Makefile` assumes `libpthread`, and `libcurl` are installed on the system. Used with the `make` command it produces two artifacts:

* A static library `libldapi.a`
* A shared library `libldapi.so`

The Windows `Makefile` assumes `libcurl` has been downloaded into the same directory as the SDK source. The dependency will automatically be built for you in this configuration. Used with `nmake /f Makefile.win` from the Visual Studio Developer Command Prompt it produces:

* ldapi.dll

You may use either of these artifacts if they are suitable for your environment.

Once ready, your first step should be to include the LaunchDarkly SDK headers.
[block:code]
{
  "codes": [
    {
      "code": "#include \"ldapi.h\"",
      "language": "c"
    }
  ]
}
[/block]
Once the SDK is installed and imported, you'll want to create a single, shared instance of `LDClient`. You should specify your *mobile key* here so that your application will be authorized to connect to LaunchDarkly and for your application and environment.
[block:code]
{
  "codes": [
    {
      "code": "unsigned int maxwaitmilliseconds = 
1. * 1000;\nLDConfig *config = LDConfigNew(\"YOUR_MOBILE_KEY\");\nLDUser *user = LDUserNew(\"YOUR_USER_KEY\");\nLDClient *client = LDClientInit(config, user, maxwaitmilliseconds);",
      "language": "c"
    }
  ]
}
[/block]

<Callout intent="info">
<Callout.Title>Use a mobile key</Callout.Title>
   <Callout.Description>Be sure to use a mobile key from your [Environments](https://app.launchdarkly.com/settings#/environments) page. Never embed a server-side SDK key into an embedded or desktop application.</Callout.Description>

</Callout>
Calling `LDClientInit` will initiate a remote call to the LaunchDarkly service to fetch the feature flag settings for the specified user. This call will block up to the time defined by `maxwaitmilliseconds`, however. If you request a feature flag before the client has completed initialization, you will receive the default flag value. To wait for client initialization, you can register a callback: 
[block:code]
{
  "codes": [
    {
      "code": "void initCallback(LDStatus status)\n{\n  if (status == LDStatusInitialized) {\n    printf(\"Completed LaunchDarkly client initialization\");\n  }\n}
LDSetClientStatusCallback(initCallback);\n",
      "language": "c"
    }
  ]
}
[/block]
Alternatively, you can conditionally gate your `LDClient` usage if the client is not yet initialized.
[block:code]
{
  "codes": [
    {
      "code": "initialized = LDClientIsInitialized(client);",
      "language": "c"
    }
  ]
}
[/block]
Using `client`, you can check which variation a particular user should receive for a given feature flag.
[block:code]
{
  "codes": [
    {
      "code": "show_feature = LDBoolVariation(client, \"YOUR_FLAG_KEY\", false);\nif (show_feature) {\n    // application code to show the feature\n} else {\n    // the code to run if the feature is off\n}",
      "language": "c"
    }
  ]
}
[/block]

## Customizing your client
You can also pass other custom parameters to the client via the configuration object:
[block:code]
{
  "codes": [
    {
      "code": "LDConfig *config = LDConfigNew(\"YOUR_MOBILE_KEY\");\nLDConfigSetEventsCapacity(config, 1000);\nLDConfigSetEventsFlushIntervalMillis(config, 30000);",
      "language": "c"
    }
  ]
}
[/block]
Here, we've customized the event queue capacity and flush interval parameters. 
## Users
Feature flag targeting and rollouts are all determined by the *user* you pass to your client. Here's an example:
[block:code]
{
  "codes": [
    {
      "code": "LDUser *user = LDUserNew(\"aa0ceb\");\nLDUserSetFirstName(user, \"Jake\");\nLDUserSetLastName(user, \"Fake\");\nLDUserSetCustomAttributesJSON(user, \"{\\\"groups\\\": [\\\"Google\\\", \\\"Microsoft\\\"]}\");",
      "language": "c"
    }
  ]
}
[/block]
Let's walk through this snippet. The first argument to the builder is the user's key-- in this case we've used the hash `"aa0ceb"`. **The user key is the only mandatory user attribute**. The key should also uniquely identify each user. You can use a primary key, an e-mail address, or a hash, as long as the same user always has the same key. We recommend using a hash if possible.

All of the other attributes (like `firstName`, `email`, and the `custom` attributes) are optional. The attributes you specify will automatically appear on our dashboard, meaning that you can start segmenting and targeting users with these attributes. 

In addition to built-in attributes like names and e-mail addresses, you can pass us any of your own user data by passing `custom` attributes
<Callout intent="info">
  <Callout.Title>A note on types</Callout.Title>
   <Callout.Description>Most of our built-in attributes (like names and e-mail addresses) expect string values. Custom attributes values can be strings, booleans (like true or false), numbers, or lists of strings, booleans or numbers. 
If you enter a custom value on our dashboard that looks like a number or a boolean, it'll be interpreted that way.</Callout.Description>
</Callout>
Custom attributes are one of the most powerful features of LaunchDarkly. They let you target users according to any data that you want to send to us-- organizations, groups, account plans-- anything you pass to us becomes available instantly on our dashboard.
## Private user attributes
You can optionally configure the C SDK to treat some or all user attributes as [private user attributes](https://docs.launchdarkly.com/v2.0/docs/private-user-attributes). Private user attributes can be used for targeting purposes, but are removed from the user data sent back to LaunchDarkly.

In the C SDK there are two ways to define private attributes for the LaunchDarkly client:

* When creating the `LDConfig` object, you can set the `allAttributesPrivate` value to `true`. When you do this, all user attributes (except the key) for the user are removed before the user is sent to LaunchDarkly.

* When creating the `LDConfig` object, you can configure a map of `privateAttributeNames`.  If any user has a custom or built-in attribute named in this list, it will be removed before the user is sent to LaunchDarkly.
[block:code]
{
  "codes": [
    {
      "code": " LDConfig *config = LDConfigNew(\"YOUR_MOBILE_KEY\");
// Mark all attributes private\nLDConfigSetAllAttributesPrivate(config, true);",
      "language": "c"
    }
  ]
}
[/block]

## Anonymous users
You can also distinguish logged-in users from anonymous users in the SDK, as follows:
[block:code]
{
  "codes": [
    {
      "code": "LDUser *user = LDUserNew(\"aa0ceb\");\nLDUserSetAnonymous(user, true);",
      "language": "c"
    }
  ]
}
[/block]
Anonymous users work just like regular users, except that they won't appear on your Users page in LaunchDarkly. You also can't search for anonymous users on your Features page, and you can't search or autocomplete by anonymous user keys. This is actually a good thing-- it keeps anonymous users from polluting your Users page!
## Track
The track method allows you to record custom events in your application with LaunchDarkly:
[block:code]
{
  "codes": [
    {
      "code": "LDClientTrack(client, \"YOUR_EVENT_KEY\");",
      "language": "c"
    }
  ]
}
[/block]

## Offline mode
In some situations, you might want to stop making remote calls to LaunchDarkly and switch to the last known values for your feature flags. Offline mode lets you do this easily.
[block:code]
{
  "codes": [
    {
      "code": "LDConfig *config = LDConfigNew(\"YOUR_MOBILE_KEY\");
// Initialize a client in offline mode:\nLDConfigSetOffline(config, true);\nLDClient *client = LDClientInit(config, user, 0);
// Alternatively, you can switch an already-instantiated client to offline mode:\nLDClientSetOffline(client, true);",
      "language": "c"
    }
  ]
}
[/block]

## Flush
Internally, the LaunchDarkly SDK keeps an event buffer for analytics events. These are flushed periodically in a background thread. In some situations (for example, if you're testing out the SDK in a simulator), you may want to manually call flush to process events immediately.
[block:code]
{
  "codes": [
    {
      "code": "LDClientFlush(client);",
      "language": "c"
    }
  ]
}
[/block]
Note that the flush interval is configurable-- if you need to change the interval, you can do so via the configuration.
## Changing the user context
If your app is used by multiple users on a single device, you may want to change users and have separate flag settings for each user. To achieve this, the SDK supports switching between different user contexts.

You can use the `identify` method to switch user contexts:
[block:code]
{
  "codes": [
    {
      "code": "LDClientIdentify(client, newUser);",
      "language": "c"
    }
  ]
}
[/block]
The 'identify' call will load any saved flag values for the new user and immediately trigger an update of the latest flags from LaunchDarkly. Since this method re-fetches flag settings for the new user, it should not be called at high frequency. The intended use case for switching user contexts is the login / logout flow.
## Real-time updates
LaunchDarkly manages all flags for a user context in real-time by polling flags based on a real-time event stream. When a flag is modified via the LaunchDarkly dashboard, the flag values for the current user will update almost immediately.

To accomplish real-time updates, LaunchDarkly broadcasts an event stream that is listened to by the C SDK. Whenever an event is performed on the dashboard, the C SDK is notified of the updated flag settings in real-time.
## Shutting down
To fully uninitialize the C SDK resources you must use close. The operation will block until all resources have been freed. It is not safe to use any API methods after this process is initiated.
[block:code]
{
  "codes": [
    {
      "code": "LDClientClose(client);",
      "language": "c"
    }
  ]
}
[/block]

## Troubleshooting
If you are having trouble debugging the SDK consider registering a log handler, and increasing the verbosity level:
[block:code]
{
  "codes": [
    {
      "code": "void logger(const char *s) {\n  printf(\"LD: %s\\n\", s);\n}
LDSetLogFunction(LD_LOG_TRACE, logger); ",
      "language": "c"
    }
  ]
}
[/block]
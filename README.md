# App2App Communication

This solution shows a basic app-to-app communication protocol similar to the system used in the
[https://learn.microsoft.com/en-us/uwp/api/Windows.ApplicationModel.AppService.AppServiceConnection?view=winrt-22621](AppServices) system.
Specifically, there is:

* A registry of apps by name and by "service"
* A way for apps to register themselves as "hosts"
* A way for apps to invoke "hosts" over a long-running connection

In this system, the app implementing the system is called a "host" or "service". The app invoking
the system is called a "client" or "caller".

Layered on top of that is a "web request like" model on both ends, generating a request on one end
and producing a response on the other.

# How to Experiment

You can build this solution yourself and see how it works.  You'll need VS2022 installed, along with the
VC++ tools for Windows Desktop development.

1. Build the solution for your favorite architecture
2. Right-click and "deploy" the `PluginApp1` and `PluginCaller2` projects
3. Launch `PluginCaller2`
4. Click the shiny "Click me!" button

The PluginApp1 (the "host") will launch and interact with PluginCaller. Currently, that interaction is
invisible, but can be observed by debugging PluginCaller2 and its `myButton_OnClick` handler.

# Coming Soon

A short list of "to do" items

**Enumerate available hosts** - Currently, the only supported operation is to identify packages that
provide an app2app connection. Some apps will want to enumerate them and dig through other configuration
data about them.

**Mapping between endpoint types** - If a caller talks PropertySet but the host talks HttpRequestMessage,
should there be a mapping betweent the two?

**Simplified registration** - Since the manifest has all the registration information in it, a single
method can register all the types. The connection manager can use "regular" WinRT type activation to
bring up an object.

**Method-calls** - Some app2app hosts might have a flat DLL with a single call export. Make it easier
to provide that level of binding.

**Sync or not?** - WinRT API design has moved away from "`-Async` all the things". Does the connection
pipe have to be async, or should it be synchronous? Apps already know how to deal with blocking background
calls on their main threads.  Hosts already know how to deal with routing calls from an apartment to
their main thread.

**GUI debugger** - Upgrade PluginCaller2 to be better; more tools to enumerate then call then visualize
the bodies of calls; more ability to take parameters to pass along, etc.

**Helpers for hosts** - APIs in App2AppCore that let hosts strongly identify who their callers are,
along with other systems to ask questions of the caller during the connect phase.

# Layers

## Registration

Initially, apps register as COM objects using an `AppExtension`.  That is, packaged win32 apps add
markup to their manifest based on the [windows.appExtension](https://learn.microsoft.com/en-us/windows/uwp/launch-resume/how-to-create-an-extension)
system. An example is below:

```xml
<Application ...>
    <Extensions>
        <uap3:Extension Category="windows.appExtension">
            <uap3:AppExtension Name="com.microsoft.windows.app2app"
                                Id="app"
                                DisplayName="anything"
                                Description="anything"
                                PublicFolder="Public">
                <uap3:Properties>
                    <ServiceDefinition>something.yaml</ServiceDefinition>
                    <Activation>
                        <ClassId>587fc84c-xxxx-xxxx-xxxx-ff693f176f95</ClassId>
                    </Activation>
                </uap3:Properties>
            </uap3:AppExtension>
        </uap3:Extension>
	    <com:Extension Category="windows.comServer">
		    <com:ComServer>
                <!-- This is the CLSID that matches the above -->
			    <com:ExeServer Executable="PluginApp1.exe" DisplayName="PluginApp1" Arguments="-App2AppProvider">
				    <com:Class Id="587fc84c-xxxx-xxxx-xxxx-ff693f176f95"/>
			    </com:ExeServer>
		    </com:ComServer>
	    </com:Extension>
```

In this markup:

* `/AppExtension/Properties/Activation/ClassId` indicates an out-of-process COM object that implements the `IDispatch`
interface.
* `/AppExtension/@Id` is the "service name" used to identify sub-app plugins. Apps that don't have more than one
app-to-app entrypoint can just use the id string "any"
* `/AppExtension/Properties/ServiceDefinition` is a file name relative to the extension's public folder containing a
textual interface definition. General-purpose API mappers might use this information to figure out how to call an
app2app service.

## Discovery

Discovery uses the `Open` method of [AppExtensionCatalog](https://learn.microsoft.com/en-us/uwp/api/windows.applicationmodel.appextensions.appextensioncatalog?view=winrt-22621)
to find the mapping package and app2app entry from above. It'll find any registered application.

## Activation

Callers typically use `App2App.App2AppConnection.Connect` to bring up a new App2App connection. The system
uses the discover mode above to bind a package family name to the CLSID for the server, then calls CoCreateInstance
on that. The `Connect` method wraps the raw `IDispatch` in a convenience layer, details described below. The
caller then just calls `InvokeAsync` with a property set and gets a response result object back containing
another property set.

## Implementing a Host

Host class objects are expected to implement IDispatch. The `IApp2AppConnection` is a convenience wrapper around
their object.

The `App2AppCore` component provides multiple simplifications that bind `IDispatch` to certain types.


# Appendix / Notes

This project uses https://learn.microsoft.com/en-us/windows/uwp/winrt-components/raising-events-in-windows-runtime-components
to generate custom proxy stub DLLs between the processes. See also [this sample](https://github.com/microsoft/Windows-universal-samples/blob/ad9a0c4def222aaf044e51f8ee0939911cb58471/Samples/ProxyStubsForWinRTComponents/cpp/Server/ProxyStubsForWinRTComponents_server.vcxproj) for a complete description

See also this - https://github.com/microsoft/cppwinrt/pull/1290/files

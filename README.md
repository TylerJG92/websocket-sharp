![Logo](websocket-sharp_logo.png)

# websocket-sharp

This repository is a maintained fork of **websocket-sharp**, a C# implementation of the WebSocket protocol client and server.

This fork includes improved RFC 7692 `permessage-deflate` compatibility, including support for valid `server_max_window_bits` negotiation.

## Features

websocket-sharp supports:

- RFC 6455 WebSockets
- WebSocket Client and Server
- RFC 7692 Per-message Compression
- Secure WebSocket connections (`wss://`)
- HTTP Authentication (Basic/Digest)
- Query strings, Origin headers, and Cookies
- HTTP proxy connections

## Supported Frameworks

This fork currently builds for:

- .NET Framework 3.5
- .NET Framework 4.5
- .NET Standard 2.0

The project produces a single assembly named:

`websocket-sharp.dll`

## Changes in This Fork

The original websocket-sharp implementation supports the WebSocket `permessage-deflate` extension, but its extension-response validation could reject otherwise valid responses when a server returned parameters such as:

```text
server_max_window_bits=11
```

RFC 7692 allows `server_max_window_bits` values from 8 through 15.

This fork updates extension-response validation so valid values within that range are accepted while invalid or unsupported values continue to be rejected.

For example, a server may negotiate compression with a response containing:

```text
permessage-deflate;
server_no_context_takeover;
client_no_context_takeover;
server_max_window_bits=11
```

This change improves interoperability with WebSocket servers that negotiate RFC 7692 compression parameters rather than returning an extension response identical to the client's request.

### TLS Note

This change affects WebSocket compression negotiation.

It does **not** modify or extend TLS support on older .NET or Mono environments. TLS handshake compatibility is separate from WebSocket extension negotiation.

## Branches

- `master` - stable and release-ready code
- `test` - development and experimental changes

## Build

To build all supported targets in Release configuration:

```powershell
dotnet build .\websocket-sharp\websocket-sharp.csproj -c Release
```

Build outputs are generated separately for:

```text
net35
net45
netstandard2.0
```

## Install

### NuGet

This maintained fork is published as:

`WebSocketSharp-NetCompression`

Using the .NET CLI:

```powershell
dotnet add package WebSocketSharp-NetCompression
```

Using the NuGet Package Manager Console:

```powershell
Install-Package WebSocketSharp-NetCompression
```

### Manual Installation

Precompiled framework-specific assemblies are also available from the GitHub Releases page.

Choose the assembly appropriate for your target framework and add `websocket-sharp.dll` as a reference to your project.

The release archive contains builds for:

```text
net35/
net45/
netstandard2.0/
```

If you use the DLL in a Unity project, add the appropriate `websocket-sharp.dll` to a suitable location such as `Assets/Plugins`.

# Usage

## WebSocket Client

```csharp
using System;
using WebSocketSharp;

namespace Example
{
  public class Program
  {
    public static void Main (string[] args)
    {
      using (var ws = new WebSocket ("ws://example.com")) {
        ws.OnMessage += (sender, e) =>
            Console.WriteLine ("Received: " + e.Data);

        ws.Connect ();
        ws.Send ("Hello!");
        Console.ReadKey (true);
      }
    }
  }
}
```

### Creating a Client

Required namespace:

```csharp
using WebSocketSharp;
```

Create a new `WebSocket` instance with the WebSocket URL:

```csharp
var ws = new WebSocket ("ws://example.com");
```

`WebSocket` implements `System.IDisposable`, so it can be used with a `using` statement:

```csharp
using (var ws = new WebSocket ("ws://example.com")) {
  ...
}
```

The WebSocket connection will be closed when execution leaves the `using` block.

## Client Events

### OnOpen

Occurs when the WebSocket connection has been established.

```csharp
ws.OnOpen += (sender, e) => {
    ...
  };
```

### OnMessage

Occurs when a message is received.

```csharp
ws.OnMessage += (sender, e) => {
    ...
  };
```

A `WebSocketSharp.MessageEventArgs` instance is passed as `e`.

Text messages can be accessed through:

```csharp
e.Data
```

Raw message data can be accessed through:

```csharp
e.RawData
```

For example:

```csharp
if (e.IsText) {
  // Use e.Data.
  return;
}

if (e.IsBinary) {
  // Use e.RawData.
  return;
}
```

To emit received ping frames through `OnMessage`, set:

```csharp
ws.EmitOnPing = true;
```

For example:

```csharp
ws.EmitOnPing = true;

ws.OnMessage += (sender, e) => {
    if (e.IsPing) {
      // Handle received ping.
      return;
    }
  };
```

### OnError

Occurs when an error is encountered.

```csharp
ws.OnError += (sender, e) => {
    ...
  };
```

The error message is available from:

```csharp
e.Message
```

If the error was caused by an exception, it may be available from:

```csharp
e.Exception
```

### OnClose

Occurs when the WebSocket connection is closed.

```csharp
ws.OnClose += (sender, e) => {
    ...
  };
```

The close status code and reason are available through:

```csharp
e.Code
e.Reason
```

## Connecting

Connect synchronously with:

```csharp
ws.Connect ();
```

For asynchronous connection:

```csharp
ws.ConnectAsync ();
```

## Sending Data

Send data with:

```csharp
ws.Send (data);
```

`WebSocket.Send` supports several data types, including:

```csharp
ws.Send (stringData);
ws.Send (byteArray);
ws.Send (fileInfo);
```

Asynchronous sending is also supported:

```csharp
ws.SendAsync (data, completed);
```

The `completed` callback can be used to determine whether the asynchronous operation succeeded.

## Closing a Connection

Close explicitly with:

```csharp
ws.Close ();
```

Other overloads allow you to provide a close status code and reason.

Asynchronous closing is also available through:

```csharp
ws.CloseAsync ();
```

# WebSocket Server

```csharp
using System;
using WebSocketSharp;
using WebSocketSharp.Server;

namespace Example
{
  public class Echo : WebSocketBehavior
  {
    protected override void OnMessage (MessageEventArgs e)
    {
      Send (e.Data);
    }
  }

  public class Program
  {
    public static void Main (string[] args)
    {
      var wssv = new WebSocketServer (4649);

      wssv.AddWebSocketService<Echo> ("/Echo");
      wssv.Start ();

      Console.ReadKey (true);

      wssv.Stop ();
    }
  }
}
```

Required namespace:

```csharp
using WebSocketSharp.Server;
```

WebSocket services are created by deriving from:

```csharp
WebSocketBehavior
```

For example:

```csharp
public class Echo : WebSocketBehavior
{
  protected override void OnMessage (MessageEventArgs e)
  {
    Send (e.Data);
  }
}
```

A service can be registered with:

```csharp
var wssv = new WebSocketServer (4649);

wssv.AddWebSocketService<Echo> ("/Echo");
```

Start the server with:

```csharp
wssv.Start ();
```

Stop it with:

```csharp
wssv.Stop ();
```

`WebSocketBehavior` can also override events including:

```csharp
OnOpen ()
OnMessage (MessageEventArgs)
OnError (ErrorEventArgs)
OnClose (CloseEventArgs)
```

## Broadcasting

A `WebSocketBehavior` can access its session manager through:

```csharp
Sessions
```

Messages can be broadcast to connected sessions with:

```csharp
Sessions.Broadcast (data);
```

# HTTP Server with WebSockets

websocket-sharp also provides:

```csharp
WebSocketSharp.Server.HttpServer
```

WebSocket services can be added to an HTTP server in the same general manner as a `WebSocketServer`.

For example:

```csharp
var httpsv = new HttpServer (4649);

httpsv.AddWebSocketService<Echo> ("/Echo");
httpsv.Start ();
```

# WebSocket Extensions

## Per-message Compression

websocket-sharp supports the RFC 7692 `permessage-deflate` extension without context takeover.

To enable compression as a WebSocket client, set the `WebSocket.Compression` property before connecting:

```csharp
ws.Compression = CompressionMethod.Deflate;
```

The client sends a WebSocket extension request similar to:

```text
Sec-WebSocket-Extensions: permessage-deflate; server_no_context_takeover; client_no_context_takeover
```

A compatible server may return a negotiated extension response containing additional valid parameters.

For example:

```text
Sec-WebSocket-Extensions: permessage-deflate; server_no_context_takeover; client_no_context_takeover; server_max_window_bits=11
```

This fork accepts valid `server_max_window_bits` values from **8 through 15**, in accordance with RFC 7692.

The extension becomes active when compatible compression parameters are successfully negotiated during the WebSocket handshake.

## Ignoring Extensions

A WebSocket server can ignore extension requests by setting:

```csharp
IgnoreExtensions = true
```

For example:

```csharp
wssv.AddWebSocketService<Chat> (
  "/Chat",
  () =>
    new Chat () {
      IgnoreExtensions = true
    }
);
```

If enabled, the service will not return a `Sec-WebSocket-Extensions` header in its handshake response.

# Secure Connections

websocket-sharp supports SSL/TLS WebSocket connections.

As a client, use a `wss://` URL:

```csharp
var ws = new WebSocket ("wss://example.com");
```

A custom server certificate validation callback can be configured through:

```csharp
ws.SslConfiguration.ServerCertificateValidationCallback =
  (sender, certificate, chain, sslPolicyErrors) => {
    // Validate the certificate.
    return true;
  };
```

A secure WebSocket server can be configured with a certificate:

```csharp
var wssv = new WebSocketServer (5963, true);

wssv.SslConfiguration.ServerCertificate =
  new X509Certificate2 ("/path/to/cert.pfx", "password");
```

TLS capabilities ultimately depend on the .NET or Mono runtime on which websocket-sharp is running.

# HTTP Authentication

websocket-sharp supports Basic and Digest HTTP authentication.

As a client:

```csharp
ws.SetCredentials ("username", "password", preAuth);
```

If `preAuth` is `true`, credentials for Basic authentication are sent with the initial request.

A server can configure an authentication scheme and credential lookup.

For example:

```csharp
wssv.AuthenticationSchemes = AuthenticationSchemes.Basic;
wssv.Realm = "WebSocket Test";

wssv.UserCredentialsFinder = id => {
    var name = id.Name;

    return name == "user"
           ? new NetworkCredential (name, "password", "role")
           : null;
  };
```

Digest authentication can be selected with:

```csharp
wssv.AuthenticationSchemes = AuthenticationSchemes.Digest;
```

# Query Strings, Origin Headers, and Cookies

## Query Strings

Include query parameters in the WebSocket URL:

```csharp
var ws = new WebSocket ("ws://example.com/?name=user");
```

On the server, query parameters are available through:

```csharp
Context.QueryString
```

## Origin Header

A client can set the Origin header before connecting:

```csharp
ws.Origin = "http://example.com";
```

On the server, the Origin is available through:

```csharp
Context.Origin
```

## Cookies

A client can add cookies using:

```csharp
ws.SetCookie (new Cookie ("name", "value"));
```

Server-side cookies are available through:

```csharp
Context.CookieCollection
```

Custom Origin and cookie validation can also be configured on a `WebSocketBehavior`.

# HTTP Proxy

A client can connect through an HTTP proxy using:

```csharp
var ws = new WebSocket ("ws://example.com");

ws.SetProxy (
  "http://localhost:3128",
  "username",
  "password"
);
```

Proxy authentication supports Basic/Digest authentication.

# Logging

`WebSocket` includes a logging system available through:

```csharp
ws.Log
```

The logging level can be changed with:

```csharp
ws.Log.Level = LogLevel.Debug;
```

Messages can be written through methods such as:

```csharp
ws.Log.Debug ("This is a debug message.");
```

`WebSocketServer` and `HttpServer` provide similar logging functionality.

# Examples

The repository contains example projects demonstrating websocket-sharp usage.

- [Example](Example)
- [Example2](Example2)
- [Example3](Example3)

# Supported WebSocket Specifications

websocket-sharp is primarily based on:

- [RFC 6455 - The WebSocket Protocol](https://www.rfc-editor.org/rfc/rfc6455)
- [RFC 7692 - Compression Extensions for WebSocket](https://www.rfc-editor.org/rfc/rfc7692)
- [The WebSocket API](https://www.w3.org/TR/websockets/)

# Attribution

This repository is a maintained fork of the original websocket-sharp project.

Original websocket-sharp was created by `sta.blockhead`.

This fork includes additional maintenance and RFC 7692 compatibility changes by `TylerJG92`.

The original copyright notice has been retained.

# License

websocket-sharp is provided under the [MIT License](LICENSE.txt).

Copyright (c) 2010-2017 sta.blockhead  
Copyright (c) 2026 TylerJG92
[![Version](https://img.shields.io/nuget/vpre/WebSocketPipe.svg?color=royalblue)](https://www.nuget.org/packages/WebSocketPipe)
[![Downloads](https://img.shields.io/nuget/dt/WebSocketPipe.svg?color=green)](https://www.nuget.org/packages/WebSocketPipe)
[![License](https://img.shields.io/github/license/devlooped/WebSocketPipe.svg?color=blue)](https://github.com/devlooped/WebSocketPipe/blob/main/license.txt)
[![Build](https://github.com/devlooped/WebSocketPipe/workflows/build/badge.svg?branch=main)](https://github.com/devlooped/WebSocketPipe/actions)

# Usage

```csharp
using Devlooped;

var client = new ClientWebSocket();
await client.ConnectAsync(serverUri, CancellationToken.None);

using IWebSocketPipe pipe = WebSocketPipe.Create(client, closeWhenCompleted: true);

// Start the pipe before hooking up the processing
var run = pipe.RunAsync();
```

The `IWebSocketPipe` interface extends [IDuplexPipe](https://docs.microsoft.com/en-us/dotnet/api/system.io.pipelines.iduplexpipe?view=dotnet-plat-ext-5.0), 
exposing `Input` and `Output` properties that can be used to 
read incoming messages and write outgoing ones.

For example, to read incoming data and write it to the console, 
we could write the following code:

```csharp
await ReadIncoming(pipe.Input);

async Task ReadIncoming(PipeReader reader)
{
    while (await reader.ReadAsync() is var result && !result.IsCompleted)
    {
        Console.WriteLine($"Received: {Encoding.UTF8.GetString(result.Buffer)}");
        reader.AdvanceTo(result.Buffer.End);
    }
    Console.WriteLine($"Done reading.");
}
```

Similarly, to write to the underlying websocket the input 
entered in the console, we use code like the following: 

```csharp
await SendOutgoing(pipe.Output);

async Task SendOutgoing(PipeWriter writer)
{
    while (Console.ReadLine() is var line && line?.Length > 0)
    {
        Encoding.UTF8.GetBytes(line, writer);
    }
    await writer.CompleteAsync();
    Console.WriteLine($"Done writing.");
}
```

If we wanted to simultaneously read and write and wait for 
completion of both operations, we could just wait for both 
tasks:

```csharp
// Wait for completion of processing code
await Task.WhenAny(
    ReadIncoming(pipe.Input),
    SendOutgoing(pipe.Output));
```

Note that completing the `PipeWriter` automatically causes the 
reader to reveive a completed result and exit the loop. In addition, 
the overall `IWebSocketPipe.RunAsync` task will also run to completion. 


The `IWebSocketPipe` takes care of gracefully closing the connection 
when the input or output are completed, if `closeWhenCompleted` is set 
to true when creating it. 

Alternatively, it's also possible to complete the entire pipe explicitly, 
while setting an optional socket close status and status description for 
the server to act on:

```csharp
await pipe.CompleteAsync(WebSocketCloseStatus.NormalClosure, "Done processing");
```

Specifying a close status will always close the underlying socket.
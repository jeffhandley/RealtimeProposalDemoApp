# Copilot Instructions — RealtimePlayGround

## Architecture

This workspace is a **multi-repo integration testbed** for the `Microsoft.Extensions.AI` realtime API proposal. It consists of four subprojects that form a layered stack:

```
RealtimeProposalDemoApp (WinForms UI — this is the primary project)
  └─ Microsoft.Extensions.AI abstractions + middleware  (tarekgh-extensions)
       └─ Microsoft.Extensions.AI.OpenAI provider       (tarekgh-extensions)
            └─ OpenAI .NET SDK realtime surface          (openai-dotnet)
googleapis-dotnet-genai  (Google GenAI SDK — positioned for future Google realtime provider)
```

- **RealtimeProposalDemoApp** — The demo WinForms app. Exercises `IRealtimeClient` / `IRealtimeSession` with the `RealtimeSessionBuilder` middleware pipeline (function invocation, OpenTelemetry, logging). Uses NAudio for audio capture/playback.
- **tarekgh-extensions** — Fork of [dotnet/extensions](https://github.com/dotnet/extensions) containing the realtime AI abstractions (`IRealtimeClient`, `IRealtimeSession`, message types) and middleware. This is where the API surface is defined and iterated.
- **openai-dotnet** — Fork of the OpenAI .NET SDK. Provides the underlying `OpenAI.Realtime` namespace used by the `OpenAIRealtimeClient` provider.
- **googleapis-dotnet-genai** — Google GenAI .NET SDK. Depends on `Microsoft.Extensions.AI.Abstractions` and has a `Live` API (`AsyncSession` over WebSocket). Not yet wired into the demo app's realtime flow.

### Key data flow

1. Demo app creates `OpenAIRealtimeClient` → calls `CreateSessionAsync()` → gets an `IRealtimeSession`
2. Session is wrapped via `RealtimeSessionBuilder` → `.UseFunctionInvocation()` → `.UseOpenTelemetry()` → `.UseLogging()` → `.Build(serviceProvider)`
3. Audio is captured via NAudio, resampled to **24 kHz mono PCM16**, sent as `RealtimeClientInputAudioBufferAppendMessage`
4. Responses arrive via `GetStreamingResponseAsync()` — pattern-matched by `RealtimeServerMessage` subtypes

## Build & Run

### Prerequisites

- .NET 10+ SDK (targets `net10.0-windows`)
- Windows OS (WinForms dependency)
- OpenAI API key with realtime model access

### NuGet package sources

The demo app uses pre-release `Microsoft.Extensions.AI` packages (`10.4.0-dev`) built locally from the tarekgh-extensions repo. The `nuget.config` expects them at:

```
C:\oss\extensions\artifacts\packages\Release\Shipping
```

If building tarekgh-extensions to a different output path, update `nuget.config`:
```xml
<add key="extensions" value="<your-local-path>" />
```

### Commands

```powershell
# Set API key (one-time)
cd RealtimeProposalDemoApp
dotnet user-secrets set "OpenAIKey" "<your-key>"

# Build and run the demo app
dotnet build
dotnet run --project RealtimePlayGround

# Build tarekgh-extensions (to produce local NuGet packages)
cd tarekgh-extensions
.\build.cmd --restore --build --pack

# Build googleapis-dotnet-genai
cd googleapis-dotnet-genai
dotnet build Google.GenAI.sln

# Run Google GenAI tests
dotnet test Google.GenAI.Tests
```

### Subproject-specific builds

The openai-dotnet and tarekgh-extensions repos have their own `.github/copilot-instructions.md` files with detailed build instructions. Refer to those when working within those subprojects.

## Conventions

### Realtime message protocol

The demo app uses a specific message sequence for audio:
1. `RealtimeClientInputAudioBufferAppendMessage` — send audio chunk
2. `RealtimeClientInputAudioBufferCommitMessage` — commit the buffer
3. `RealtimeClientResponseCreateMessage` — request a response

For text input, use `RealtimeClientConversationItemCreateMessage` with a `RealtimeContentItem` containing `TextContent`, followed by `RealtimeClientResponseCreateMessage`.

### Audio format

All audio sent to the OpenAI realtime API must be **24 kHz, 16-bit, mono PCM**. The demo app records at the device's native sample rate and resamples via `ResampleAudio()` / `ConvertToMono()` in `MainForm.cs`.

### Middleware pipeline order

The `RealtimeSessionBuilder` pipeline is always configured in this order:
```csharp
new RealtimeSessionBuilder(session)
    .UseFunctionInvocation(...)   // handles tool calls
    .UseOpenTelemetry(...)        // captures Activity spans + metrics
    .UseLogging()                 // structured ILogger output
    .Build(serviceProvider);
```

### Server message handling

Incoming `RealtimeServerMessage` objects are dispatched via C# pattern matching (`switch` on concrete subtypes). Key types:
- `RealtimeServerOutputTextAudioMessage` — audio deltas and transcription
- `RealtimeServerInputAudioTranscriptionMessage` — user speech transcription
- `RealtimeServerErrorMessage` — API errors
- `RealtimeServerResponseCreatedMessage` — includes token usage
- `RealtimeServerResponseOutputItemMessage` — contains function call results

### Warning suppressions

The demo app suppresses `MEAI001` (experimental AI APIs), `OPENAI002` (experimental OpenAI APIs), and `SCME0001` in the `.csproj`. These are expected for pre-release API surfaces.

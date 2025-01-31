# LSPService

🚨 *This project is already in productive use but not mature enough yet to warrant versioning.*

👩🏻‍🚀 *Contributors and pioneers welcome!*

## What?

LSPService is an app that locally hosts a web service which presents local [LSP language servers](https://langserver.org) as WebSockets:

![LSPService](https://raw.githubusercontent.com/flowtoolz/LSPService/master/Documentation/lspservice_idea.jpg)

I use mainly the [Swift language server (sourcekit-lsp)](https://github.com/apple/sourcekit-lsp) as my example language server, and LSPService is itself written in Swift. **But in principle, LSPService runs on macOS and Linux and can connect to all language servers**. 

## Why?

The [Language Server Protocol](https://microsoft.github.io/language-server-protocol/) is the present and future of software development tools. But leveraging it for a tool project turned out to be difficult. 

For instance, I want to distribute a developer tool via the Mac App Store, so it must be sandboxed, which makes it impossible to directly deal with language servers or any other "tooling" of the tech world.

So I thought: **What if a language server was simply a local web service?** Possible benefits:

* **Editors don't need to install, locate, launch, configure and talk to different language server executables.**
* **On macOS, editors can be sandboxed and probably even be distributed via the App Store.**
* In the future, LSPService could be a machine's central place for managing LSP language servers, through a CLI and/or through a web front end.
* Even further down the road, running LSPService as an actual web service might have interesting applications, in particular where LSP is used for static code analysis or remote inspection.

## How?

### As the Developer of an Editor

1. Let your editor use LSPService:
	* [The API](#API) allows talking to language servers and configuring them.
	* If you want to put your editor into the Mac App Store: Ensure your editor is also usable without LSPService. This should help with the review process.
2. Provide a download of LSPService to your users:
	* Build it via `swift build --configuration release`.
	* Get the resulting binary: `.build/<target architecture>/release/LSPService`.
	* Upload the binary to where your users should download it.
3. Let your editor encourage the user to download and run LSPService:
	* Give a short explanation for why LSPService is helpful.
	* Offer a convenient way to download and save LSPService.

### As the User of an Editor

Of course, we assume here the editor supports LSPService.

1. Download and open LSPService. It will run in Terminal, and as long as it's running there, the service is available. Check: <http://localhost:8080/lspservice>
3. Set the language server locations (file paths) for the languages you want to have supported. 
	* There's a command line interface for that. After LSPService launches in Terminal, it explains all commands there.
	* LSPService automatically finds the Swift language server (if Xcode is installed), and it guesses the location of a [python language server](https://github.com/palantir/python-language-server). But in general, users still must locate language servers manually.


## API

### Editor vs. LSPService – Who's Responsible?

The singular purpose of LSPService to present LSP language servers as WebSockets.

LSPService forwards LSP messages from some editor to some language server and vice versa. It knows nothing about the LSP standard itself. Encoding and decoding LSP messages and generally representing LSP with proper types remains a concern of the editor. 

The editor, on the other hand, knows nothing about how to locate, install, run and talk to language servers. This remains a concern of LSPService.

### Endpoints

The root of the LSPService API is `http://127.0.0.1:8080/lspservice/api/`.

| URL Path |     Types     |      Methods | Usage |
| :------------ | :---------- | :---------- |:----------- |
| `processID` | `Int` | `GET` | Get process ID of LSPService, to set it in the [LSP initialize request](https://microsoft.github.io/language-server-protocol/specification#initialize). |
| `languages` | `[String]` | `GET` | Get names of languages which have the path to their language server set. |
| `language/<lang>` | `String` | `GET`, `POST` | Get and set the path of the language server associated with language `<lang>`. |
| `language/<lang>/websocket` | `Data`, `String` | WebSocket | Connect and talk to the language server associated with language `<lang>`. |

### Using the WebSocket

* Depending on the frameworks you use, you may need to set the URL scheme `ws://` explicitly.

* Encode LSP messages according to the LSP specification, including [header](https://microsoft.github.io/language-server-protocol/specifications/specification-current/#headerPart) and content part.

* Send and receive LSP messages via the data channel of the WebSocket. The data channel is used exclusively for LSP messages. It never outputs any other type of data. Each data message it puts out is one LSP packet (header + content), as LSPService pieces packets together from the language server output.

* [LSP response messages](https://microsoft.github.io/language-server-protocol/specifications/specification-current/#responseMessage) may inform about errors. These LSP errors are critical feedback for your editor.

* Besides LSP messages, there are only two ways the WebSocket gives live feedback:
	* It sends the language server's [error output](https://en.wikipedia.org/wiki/Standard_streams#Standard_error_(stderr)) via the text channel. These are unstructured pure text strings that are useful error logs for debugging.
	* It terminates the connection if some serious problem occured, for example if the language server in use had to shut down.

## To Do / Roadmap

* [x] Implement proof of concept with WebSockets and sourcekit-lsp
* [x] Have a dynamic endpoint for all languages, like `127.0.0.1:<service port>/lspservice/api/<language>`
* [x] Let LSPService locate sourcekit-lsp for the Swift endpoint
* [x] Evaluate whether client editors need to to receive the [error output](https://en.wikipedia.org/wiki/Standard_streams#Standard_error_(stderr)) from language server processes.
  * Result: LSP errors come as regular LSP messages from standard output, and using different streams is not part of the LSP standard and a totally different abstraction level anyway. So stdErr should be irrelevant to the editor. But for debugging, we provide it via the WebSocket's text channel.
* [x] Explore whether sourcekit-lsp can be adjusted to send error feedback when it fails to decode incoming data. This would likely accelerate development of LSPService and of other sourcekit-lsp clients.
  * Result: sourcekit-lsp [now sends an LSP error response message](https://github.com/apple/sourcekit-lsp/pull/334) in response to an undecodable message, if that message has at least a [valid header](https://microsoft.github.io/language-server-protocol/specifications/specification-current/#headerPart). Prepending that header is easy, so development of LSPService can now rely on immediate well formed feedback from sourcekit-lsp.
* [x] Add an endpoint for client editors to detect what languages are available
* [x] Properly handle websocket connection attempt for unavailable languages: send feedback, then close connection.
* [x] Lift logging and error handling up to the best practices of Vapor. Ensure that users launching the host app see all errors in the terminal, and that clients get proper error responses.
* [x] Allow to use multiple different language servers. Proof concept by supporting/testing a Python language server
* [x] Add a CLI so users can manage the list of language servers from the command line
* [x] Clean up interfaces: Future proof and rethink API structure, then align CLI, API and web frontend
* [x] Document how to use LSPService
* [x] Evaluate whether to build a Swift package that helps clients of LSPService (that are written in Swift) to define, encode and decode LSP messages. Consider suggesting to extract that type system from [SwiftLSPClient](https://github.com/chimehq/SwiftLSPClient) and/or from sourcekit-lsp into a dedicated package.
	* Result: Extraction already happened anyway in form of sourcekit-lsp's static library product `LSPBindings`. However, `LSPBindings` didn't work for decoding as it's decoding is entangled with matching requests to responses.
	* Result: [SwiftLSPClient](https://github.com/chimehq/SwiftLSPClient)'s type system is incomplete and obviously not backed by Apple.
	* Result: The idea to strictly type LSP messages down to every property seems inappropriate for their amorphous "free value" nature anyway. So we opt for a custom, simpler and more dynamic LSP type system (now as [SwiftLSP](https://github.com/flowtoolz/SwiftLSP)).
* [x] Get a sourcekit-lsp client project to function with sourcekit-lsp at all, before moving on with LSPService
* [x] Remove "Process ID injection". Add endpoint that provides process ID.
* [x] Detect LSP packets properly (piece them together from server process output)
* [x] Extract general LSP type system (not LSPService specific) into package [SwiftLSP](https://github.com/flowtoolz/SwiftLSP)
* [x] Build a Swift package that helps client editors written in Swift to use LSPService
    * Result: [LSPServiceKit](https://github.com/flowtoolz/LSPServiceKit)
* [ ] As soon as [this PR](https://github.com/vapor/vapor/pull/2498) is done: Decline upgrade to Websocket protocol right away for unavailable languages, instead of opening the connection, sending feedback and then closing it again.
* [ ] Persist language server configurations
* [ ] **Milestone** General "releasability": professional CLI, failure tolerance, expressive error logs ... 
* [ ] Explore whether this approach would actually fly with the Mac App Store review, because:
  * The editor app would need to encourage the user to download and install LSPService, but apps in the App Store are not allowed to lead the user to some website, at least as it relates to purchase funnels.
  * It could be argued that LSPService is more of a plugin that effects the behaviour of the editor app, which would break App Store rules.
* [ ] Ensure sourcekit-lsp can be used to support C, C++ and Objective-c 
* [ ] Consider adding a web frontend for managing language servers. Possibly use [Plot](https://github.com/JohnSundell/Plot)
* [ ] Enable serving multiple clients who need services for the same language at the same time

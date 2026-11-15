# Forge
A Xcode port for iOS
# Forge

An iPhone / iPad app for writing SwiftUI apps on the device, exporting them as
Xcode projects, and building unsigned IPA files in the cloud.

## What it does

- **Projects** – create from a template (Blank, To-Do List, Tab App), duplicate, delete.
  Projects live in the app's Documents folder and are visible in the Files app.
- **Import** – bring in an existing Xcode project from a folder or a `.zip` (see below).
- **Navigator** – file tree with new file / new folder / rename / delete.
- **Editor** – Swift syntax highlighting, line numbers, auto-indent, bracket and
  quote auto-pairing, a symbol bar above the keyboard, undo/redo, find, autosave.
- **Issues** – flags unbalanced brackets and unterminated strings/comments as you type.
- **Export Xcode Project** – two ways, both from the *Build & Export* menu:
  *Export Xcode Project (.zip)* opens the share sheet (AirDrop to a Mac, Save to Files,
  Mail, …), and *Save Xcode Project Folder to Files…* writes the plain, unzipped
  `<Name>/` folder (with `<Name>.xcodeproj` inside) to a place you pick in Files.
- **Build IPA** – builds an unsigned `.ipa` on a GitHub macOS runner and downloads it
  into Forge. If the build fails, the compiler errors come back into Forge and tapping
  one opens the file at that line.
- **Credits** – the *i* button on the project list. VicStinger: idea and backend
  developer. Claude (an AI model made by Anthropic): main developer.

## Importing projects

On the project list tap **+ → Import Folder or Zip…** and pick a folder or a `.zip` in
Files. Pick the folder that *contains* the `.xcodeproj`, not the `.xcodeproj` itself
(Xcode projects only list files; the sources live next to them).

- **Zips** are unpacked on the device. A single wrapper folder (`MyApp/…`) and macOS
  litter (`__MACOSX`, `.DS_Store`) are dealt with automatically. Entries that would
  escape the project folder (`../`, absolute paths) cause the whole import to be refused.
- **Finding the project.** Forge looks up to three folders deep for a `.xcodeproj`
  (preferred) or `.xcworkspace`, skipping `Pods`, `Carthage`, asset catalogs and
  frameworks. If several match, the shallowest wins, and then the one named like the
  folder or zip.
- **Imported projects are kept as they are.** Forge never rewrites their `project.pbxproj`.
  Exporting gives back the same files (hidden files such as `.gitignore` included), and the
  cloud build uses the project you imported with the scheme you set in *Project Settings*
  (or the first scheme it lists). Bundle ID, version and deployment target live in the
  project itself, so Forge only shows what it detected.
- **No `.xcodeproj` in the folder?** If it contains Swift files, Forge imports them as a
  normal Forge project, and a generated `.xcodeproj` is written when you export. Remove any
  `Info.plist`, `Package.swift` or tests in the folder first, since the generated project
  doesn't expect them.
- **Skipped** on import and export: `.git`, `xcuserdata`, `DerivedData`, `.build`,
  `.swiftpm`, symlinks.
- **Limits:** 20,000 files and 400 MB per import; 2,000 files per cloud build upload.
- A folder in iCloud Drive that hasn't been downloaded yet is fetched on demand, which can
  take a while. If an import stalls, download the folder in Files first.

## Build IPA (cloud build)

iOS can't compile Swift, so the build runs on GitHub Actions. Forge pushes your project
as a throw-away branch of a repository you own, the workflow it includes runs
`xcodebuild` on a macOS runner, and Forge downloads the result. The branch is deleted
afterwards; the run and its artifacts stay on GitHub.

### One-time setup

1. On github.com create a repository with **Add a README file** ticked
   (private is recommended).
2. Create a **fine-grained personal access token** limited to that repository, with
   Read and write access to **Contents**, **Actions** and **Workflows**.
3. In Forge: *Build & Export → Build IPA… → gear icon*, enter `owner/repository` and the
   token, then tap **Test Connection**. The token is stored in the iOS Keychain.

### Things to know

- **The IPA is unsigned.** It will not install on a device as-is. Re-sign it with a
  sideloading tool (AltStore, Sideloadly, …) or with your own certificate and profile.
- **Running it with LiveContainer.** Forge can't run apps itself: iOS doesn't let one app
  execute another's code, so importing a certificate into Forge wouldn't change that.
  [LiveContainer](https://livecontainer.github.io/docs/intro) is a separate app that does
  this. After a build, tap **Save IPA to Files…**, then in LiveContainer tap **+** and pick
  the IPA. Per LiveContainer's docs, with JIT enabled it skips code signing (iOS 18 and
  below); otherwise it signs the app with its own certificate. Forge has no certificate
  handling of its own and never sees your certificate.
- **Cost.** Public repositories get free runner minutes. On private repositories each build
  uses your plan's macOS minutes, which are billed at ten times the normal rate.
- **Privacy.** In a *public* repository, your source and the build log are visible to
  anyone who finds the run. Use a private repository for anything you want to keep private.
- **Builds are not instant.** Expect a few minutes, including time waiting for a runner.

### Troubleshooting

| Message | Cause |
| --- | --- |
| "GitHub rejected the access token (401)" | Token is wrong, expired, or revoked. |
| 403/404 while uploading, mentions `workflow` | Token is missing the **Workflows** permission, or isn't granted this repository. |
| "That repository is empty" | Create the repository with a README, or add any file to it. |
| "GitHub didn't start a build" | Actions are disabled: *repo → Settings → Actions → General*. |

## What it does not do (and can't)

- It is **not** Xcode ported to iOS. Xcode, the Swift compiler and the iOS SDK are Apple
  proprietary and don't run on iOS.
- It can't build or run apps **on the device**. The cloud build produces an IPA; running
  it still needs a Mac or a sideloading tool.
- The on-device issue checker is structural only, not a compiler. Type errors show up
  after a cloud build, or in Xcode.

## Open Forge in Xcode

Requires **Xcode 16 or newer** (the project uses folder-synchronized groups) and iOS 17+.

1. Open `Forge.xcodeproj`.
2. Select the *Forge* target → *Signing & Capabilities* → choose your Team.
3. Pick an iPhone/iPad simulator or your device and press Run.

## Use an exported project

1. Unzip `<Name>.zip` on your Mac.
2. Open `<Name>/<Name>.xcodeproj`.
3. Choose a signing Team, then Run.

The exported project has no file list to maintain: the target points at the `<Name>`
folder, so any file you add there is compiled or bundled automatically. Display name,
bundle ID, version and deployment target come from *Project Settings* in Forge.

## Layout

```
Forge.xcodeproj
Forge/
  ForgeApp.swift
  Models/      ProjectManifest, FileNode
  Services/    ProjectStore, ProjectSession, ProjectTemplates(+Samples),
               XcodeProjectGenerator (pbxproj + shared scheme), ZipWriter,
               ProjectExporter (zip + folder), ProjectImporter,
               ProjectDetection, SwiftLinter
  Build/       BuildWorkflow (the Actions YAML), GitHubClient, BuildService,
               ZipReader, BuildDiagnostic + BuildLogParser, Keychain
  Editor/      SwiftLexer, SwiftHighlighter, SyntaxTheme, CodeUITextView,
               CodeEditorView, KeyboardAccessory
  Views/       ProjectListView, NewProjectSheet, ProjectWorkspaceView,
               FileTreeView, IssuesView, ProjectSettingsView, ShareSheet
               (+ FilesExportPicker), BuildIPAView, BuildSettingsView,
               CreditsView
tools/generate_xcodeproj.py   regenerates Forge.xcodeproj (mirrors XcodeProjectGenerator.swift)
```

## Status

The Swift code was written without access to a Swift toolchain, so it has **not been
compiled**. Expect to fix a few compiler errors on first build. It has also never talked
to the real GitHub API or run a real `xcodebuild`.

What was checked, using Python ports and the real text of the Swift sources:

- The generated `project.pbxproj` parses with an independent parser; no dangling object
  references; the shared scheme is well-formed XML and points at the right target.
- The ZIP writer's byte layout and the ZIP reader (stored, deflate, and GitHub-style
  streamed zips with data descriptors, plus corruption and truncation) round-trip through
  Python's `zipfile`.
- The workflow YAML embedded in `BuildWorkflow.swift` parses, and its shell steps run
  correctly against a stand-in `xcodebuild` for both a passing and a failing build
  (including the failure propagating through `| tee`, the IPA being a valid zip, and the
  diagnostics file being collected).
- The compiler-message parser handles real-shaped log lines, de-duplication, and path
  mapping back into the project, for both generated projects (files in a subfolder) and
  imported ones (files at the repository root).
- Import logic (project detection, wrapper-folder unwrapping, `Pods` exclusion, bundle ID
  and deployment-target extraction from a real `project.pbxproj`, size caps, symlink
  skipping, cleanup of a failed import) was run on real zips and folders, including
  hostile archives (`../`, absolute and backslash paths). An imported project re-exports
  byte-for-byte.
- The workflow was rendered for generated, imported, workspace, explicit-scheme,
  no-scheme, multiple-`.app`, no-`.app` and failing-build cases, with spaces, quotes and
  shell metacharacters in names, and each step run against a stand-in `xcodebuild` with
  the same shell options GitHub uses.
- Bracket balance, multi-line string indentation, and duplicate-dictionary-key checks on
  all Swift files.

Not checked: that Xcode opens the generated project, and that GitHub accepts the
token/permission combination described above.

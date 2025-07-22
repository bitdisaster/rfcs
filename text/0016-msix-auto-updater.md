# RFC Template

- Start Date: (fill me in with today's date, 2025-05-15)
- RFC PR: [electron/rfcs#0016](https://github.com/electron/rfcs/pull/0016)
- Electron Issues: [electron/electron#0000](https://github.com/electron/electron/issues/0000)
- Reference Implementation: [electron/electron#0000](https://github.com/electron/electron/pull/0000)
- Status: **Proposed**

# MSIX Auto Updater

## Summary

MSIX is Microsoft's successor to the APPX and MSI installer formats. Windows provides an API to auto-update MSIX packages. This feature would implement an alternative [auto-updater](https://github.com/electron/electron/blob/main/shell/browser/auto_updater.h) to [Squirrel.Windows](https://github.com/electron/electron/blob/main/lib/browser/api/auto-updater/squirrel-update-win.ts). However, this would be a native implementation like Squirrel.Mac and not an external process spawning like Squirrel.Windows.

## Motivation

The MSIX package format provides a modern approach to application deployment, suitable for both enterprise environments and direct downloads to users. It achieves a 99.96% success rate across millions of installations, as reported by Microsoft (https://learn.microsoft.com/en-us/windows/msix/overview). Key features include network bandwidth and disk space optimization, clean uninstallation, and support for tools like Microsoft Endpoint Configuration Manager, Microsoft Intune, and Deployment Image Servicing and Management (DISM.exe). It also supports MSIX App Attach for virtualized environments, AppInstaller for self-hosted deployment and updates, and PowerShell commands for management.

MSIX enables auto-updating through Windows OS APIs, unlike MSI, which lacks this feature, or Squirrel.Windows, which requires a custom updater to be maintained and distributed. For details on supported features by OS version, see https://learn.microsoft.com/en-us/windows/msix/supported-platforms?view=winrt-26100.

Once implemented, MSIX auto-updating would be availble as an alternative to Squirrel.Windows via the https://www.electronjs.org/docs/latest/api/auto-updater#windows API, providing all the advantages mentioned above.

## Guide-level explanation

Explain the feature as if it were already implemented in Electron and you were teaching it to
an Electron app developer.

This section should:

- Introduce new named concepts.
- Show concrete examples of how the feature is used.
- Explain how the feature will impact existing use cases of Electron.
- If applicable, describe the migration path from an older set of Electron features or APIs.
- Discuss how this impacts the ability to read, understand, and maintain Electron code. Will the
  proposed feature make Electron code more maintainable? How difficult is the upgrade path for
  existing apps?

When writing this section, make sure to clearly account for API differences or considerations for
Windows, macOS, and Linux.

## Reference-level explanation

<!--
This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.
- Any new dependencies on Chromium code are outlined.

The section should return to the examples given in the previous section, and explain more fully how
the detailed proposal makes those examples work.
-->

<!--
- lib/browser/api/auto-updater/msix-update-win.ts
  - use when packaged app is detected

- process changes
  - deprecate process.winstore
  - process.windowsPackagedApp
  - process.microsoftStore
- getPackageInfo()
- registerPackage()
- updatePackage()
- registerAppRestart()

- use macos squirrel format
  - same updater backend for mac and windows now

- .appinstaller file support
  - can follow after initial impl
  - lib/browser/api/auto-updater/appinstaller-update-win.ts
  - https://learn.microsoft.com/en-us/windows/msix/app-installer/app-installer-file-overview

- microsoft store updates
  - lib/browser/api/auto-updater/microsoft-store-update-win.ts
  - https://learn.microsoft.com/en-us/windows/msix/store-developer-package-update
-->

### Overview

Electron's [autoUpdater](https://www.electronjs.org/docs/latest/api/auto-updater) module supports macOS and Windows. Electron contains an implementation per-platform where the Windows updater assumes the use of Squirrel.

``` js
// lib/browser/api/auto-updater.ts
if (process.platform === 'win32') {
  module.exports = require('./auto-updater/auto-updater-win');
} else {
  module.exports = require('./auto-updater/auto-updater-native');
}
```

To introduce MSIX auto updating, Electron will provide a new updater implementation when a Windows packaged app is detected.

### Windows packaged apps

Updating an application deployed with MSIX requires knowing about its packaged app identity. APIs will be needed to gather this information under a `windowsPackagedApp` namespace.

This will allow the packaged app to know its ID for updating and whether its
distributed as standalone or from the Microsoft Store.

```ts
/**
 * Info about an application package.
 * @see https://learn.microsoft.com/en-us/uwp/api/windows.applicationmodel.package?view=winrt-22621
 */
interface WindowsPackagedAppInfo {
  /**
   * 'FullName' ID.
   * e.g. 91750D7E.Slack_4.38.65535.0_arm64__8she8kybcnzg4
   */
  id: string;
  /**
   * Family name which uniquely identifies the package independent of its version.
   * e.g. 91750D7E.Slack_8she8kybcnzg4
   */
  familyName: string;
  /**
   * Whether the package is installed in development mode.
   */
  developmentMode: boolean;
  /**
   * Kind of signature.
   * Can be 'developer', 'enterprise', 'none', 'store', or 'system'.
   * @see https://learn.microsoft.com/en-us/uwp/api/windows.applicationmodel.packagesignaturekind?view=winrt-22621
   */
  signatureKind: string;
  /**
   * URI to the .appinstaller file associated with the current app.
   * Only present if the app is installed using this method.
   */
  appInstallerUri?: string;
}

interface WindowsPackagedApp {
  /**
   * Get info about the current application package.
   *
   * Throws if the current application has no packaged identity.
   */
  getPackagedAppInfo(): WindowsPackagedAppInfo;
}
```

### Performing updates

Electron apps expect auto updates to be performed with the following procedure:
1. Request update info from web server.
2. Download update in the background.
3. Apply update on next app launch.

To meet these requirements, we'll need to invoke methods on the WinRT PackageManager class.

```ts
/**
 * Options to customize the behavior of MSIX updates.
 * @see https://learn.microsoft.com/en-us/uwp/api/windows.management.deployment.addpackageoptions?view=winrt-22621
 */
type UpdatePackageOptions = {
  /**
   * Callback with percentage completition over the entire course of the
   * deployment operation.
   * @param percentage Installation percentage
   */
  onprogress?: (percentage: number) => void;
};

interface WindowsPackagedApp {
  /**
   * Download MSIX package to update to. Defer registering update until app
   * restarts.
   * @param packageUri A URL path to an MSIX package.
   * @see https://learn.microsoft.com/en-us/uwp/api/windows.management.deployment.packagemanager.addpackagebyuriasync?view=winrt-26100
   */
  updatePackage(packageUri: string, options?: UpdatePackageOptions): Promise<void>;

  /**
   * Deploy downloaded package for updating. Terminates the application to
   * proceed with updates.
   * @see https://learn.microsoft.com/en-us/uwp/api/windows.management.deployment.packagemanager.registerpackagebyfamilynameasync?view=winrt-26100
   */
  deployPackage(): Promise<void>;
}
```

### Determining update availability

Electron has historically used Squirrel updater on both Mac and Windows. Although they share the same name, their implementations and [update mechanisms are different.](https://www.electronjs.org/docs/latest/tutorial/updates#update-server-specification)

- Squirrel for Windows: fetches `RELEASES` file
- Squirrel for Mac: fetches JSON document

With the introduction of a new updater, we should take this opportunity to
converge these designs. The MSIX updater proposes to use the Mac's JSON update
file format.

```json
{
    "url": "https://your-static.storage/your-app-1.2.3-windows.msix",
    "name": "1.2.3",
    "notes": "Update for MSIX packaged app",
    "pub_date": "2025-07-22T18:59:52.022Z"
}
```

## Drawbacks

Why should we *not* do this?

## Rationale and alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
- If this is an API proposal, could this be done as a JavaScript module or a native Node.js add-on
  instead? Does the proposed change make Electron code easier or harder to read, understand,
  and maintain?

## Prior art

Discuss prior art, both the good and the bad, in relation to this proposal. A few examples of what
this can include are:

- Does this feature exist in other frameworks and what experience have their community had?
- Does this feature exist as a userland implementation, and what can be learned from it?
- Is this related to a change upstream in Chromium or Node.js?
- Does this proposal help Electron further align with evolving web standards?

This section is intended to encourage you as an author to think about the lessons from prior
implementations to provide readers of your RFC with a fuller picture. If there is no prior art,
that is fine - your ideas are interesting to us whether they are brand new or if it is an
adaptation from other technologies.

## Unresolved questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature
  before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the
  future independently of the solution that comes out of this RFC?

## Future possibilities

Think about what the natural extension and evolution of your proposal would be and how it would
affect the project as a whole in a holistic way. Try to use this section as a tool to more fully
consider all possible interactions with the project in your proposal.

This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but
otherwise related.

If you have tried and cannot think of any future possibilities, you may simply state that you
cannot think of anything.

Note that having something written down in the future possibilities section is not a reason to
accept the current or a future RFC; such notes should be in the section on motivation or
rationale in this or subsequent RFCs. The section merely provides additional information.
# Fortress vs. Shelter, Insular & Haven - Feature & Security Comparison

To determine whether I should even make this app, I've looked through the app features and source code of the other popular apps I've seen mentioned from time to time on the Privacy Guides forum to compare them against Fortress' feature-set:

- Shelter: `net.typeblog.shelter`, commit `672560f55177` (2026-06-02)
- Insular: F-Droid package `com.oasisfeng.island.fdroid`, commit `d46911c9f341` (2025-07-30)
- Haven: `io.github.kennethchoinfosec.haven`, commit `40223a369973` (2026-05-09)
- Fortress suite: current working tree

The Insular column in the tables below is the default **F-Droid** complete build (`completeFdroid`) unless the cell explicitly says "non-default". Its source is modular, and the Google Play flavor removes several installer/cross-profile permissions, and debug-only manifests are called out separately in the permissions section.[^insular-modular]

## Caveats

- Fortress is pre-release. Its transport and lockdown design should be read as current source posture, not a long production track record.

- I tested the Haven app, but it gets no column below. Haven is a near-verbatim rebrand of Shelter: Most of its source is byte-for-byte identical after the package rename, and its feature, security, permission, and policy entries are *identical* to Shelter's in every table, so the Shelter column applies to Haven too.

  The only differences are project-level: Haven is a newer, less-mature fork (Material-3 theme, updated SDK/AGP/JDK, a basic GitHub Actions lint/test/build CI, and its own F-Droid listing). Its README's "modern Android security expectations" claim is **not** borne out at the permission/capability level. Nothing was removed or tightened versus Shelter.

## 1. Provisioning & Profile Management

| Capability | Fortress | Shelter | Insular | Fortress-locked |
|---|:---:|:---:|:---:|:---:|
| Creates an Android managed/work profile and becomes profile owner[^minsdk] | ✅ | ✅ | ✅ | n/a — preserves existing profile[^lockdown] |
| Works without root | ✅ | ✅ | ✅ | ✅ |
| Normal setup needs no ADB/PC | ✅ | ✅ | ✅ | ✅ |
| Works as BYOD profile owner, without device-owner control | ✅ | ✅ | ✅ | ✅ |
| App-provided profile removal path | ✅ (Settings deep-link) | ❌ | ✅ (`wipeData`, with Settings fallback) | ✅ (Settings deep-link)[^lockdown] |
| Holds destructive profile-wipe capability itself | ❌ | ❌ | ✅ | ❌ |
| Optional device-owner control over the primary user | ❌ | ❌ | ✅ (non-default)[^insular-godmode] | ❌ |
| Re-provision/rescind owner controls in app | ❌ | ❌ | ✅ | ❌ |
| In-app work-profile quiet-mode/deactivation helper | ❌ | ❌ | ✅ | ❌ |

## 2. App Management

| Capability | Fortress | Shelter | Insular | Fortress-locked |
|---|:---:|:---:|:---:|:---:|
| Clone an installed app personal -> work | With Freighter | ✅ | ✅ | ❌ |
| Clone an installed app work -> personal | With Freighter | ✅ | ✅ | ❌ |
| Clone APK code only, not app data | With Freighter | ✅ | ✅ | ❌ |
| Install an arbitrary APK file into the work profile from an in-app picker/handler | ❌[^fortress-install] | ✅ | ✅ | ❌ |
| Enable sideloading inside the work profile | ✅[^fortress-install] | ✅ | ✅ | ✅ (left enabled) |
| Uninstall apps from the manager | ❌ | ✅ | ✅ | ❌ |
| Enable built-in/system apps missing from a fresh profile | With Freighter (curated list)[^curated] | ✅ | ✅ | ❌ |
| Freeze/disable apps with `setApplicationHidden` | With Freezer | ✅ | ✅ | ❌[^freezer-lockdown] |
| Suspend apps with `setPackagesSuspended` | With Freezer | ❌ | Partial[^insular-suspend] | ❌[^freezer-lockdown] |
| Auto-freeze on screen-off/lock[^autofreeze] | ❌ | ✅ | non-default (Greenify) | ❌ |
| Batch freeze / freeze selected set | ❌ | ✅ | Partial | ❌ |
| Unfreeze-and-launch shortcuts | ❌ | ✅ | ✅ | ❌ |
| Freeze-all shortcut/notification action | ❌ | ✅ | Partial | ❌ |
| Play Store / market-assisted cloning fallback[^market-clone] | ❌ | ❌ | ✅ | ❌ |
| Root/Shizuku-assisted app cloning | ❌ | ❌ | ✅ (non-default)[^insular-godmode] | ❌ |
| Public third-party app freeze/launch/suspend API | ❌ | ❌ | ✅ | ❌ |

## 3. Custom App-Implemented Cross-Profile Features

Features an app builds **on top of** the Work Profile platform, i.e. not AOSP defaults, and not DPM policy toggles (those are next, in section 4).

| Capability | Fortress | Shelter | Insular | Fortress-locked |
|---|:---:|:---:|:---:|:---:|
| Cross-profile file shuttle | ❌ | ✅ (Documents UI) | Partial (URI/activity shuttle) | ❌ |
| Cross-profile DocumentsProvider | ❌ | ✅ | ❌ | ❌ |
| Cross-profile file access via broad/legacy external storage[^minsdk] | n/a - minSdk 31 | ✅ | ❌ | n/a - minSdk 31 |
| NFC payment-stub workaround for work-profile payment apps[^nfc] | ❌ | ✅ | ❌ | ❌ |

> [!NOTE]
> **Cross-profile clipboard (bidirectionally), the "share to work profile" sheet (`ACTION_SEND`), and many "open with" handoffs work by AOSP default on _every_ app here, Fortress included.** Those are platform behaviors, not app features, so they live in section 4a (defaults) rather than as per-app rows. The rows above are the cross-profile data features an app must *build* on top of the platform. Cross-profile **policy toggles** (contacts search, the *Connected Apps* allowlist, widget providers) are in the next section.

## 4. Managed-Profile Policy & Cross-Profile Data-Flow Defaults

Most of these policies are inherited from AOSP defaults rather than set by any of the apps. That is, a managed profile already enforces (or allows) them, whether an app's code mentions them or not. 

The tables show the effective state on a **stock** BYOD work profile, as well as where an app's code overrides the default.

> [!NOTE]
> Fortress sets `minSdk` to 31 (Android 12), whereas Insular reaches 27 and Shelter/Haven reach 24, meaning they also run on Android 7-11. On those older versions of Android a few of these behaviors differ, so I've flagged them in each applicable row. Also, I've noted where legacy work-profile surface exists that a minSdk 31 app like Fortress inherently avoids.[^minsdk]

### 4a. Cross-profile data-flow restrictions

- ✅ = the data-flow is blocked (hardened)
- ❌ = allowed
- "Configurable" = the app exposes a user toggle.

Out of the box, a work profile shares the clipboard bidirectionally, and allows personal→work sharing.

| Restriction (behavior) | Fortress | Shelter | Insular | Fortress-locked |
|---|:---:|:---:|:---:|:---:|
| Blocks **personal → work** import — sharing + clipboard paste-in[^xprofile] | Configurable (Default Blocked) | ❌ | ❌ | ✅ |
| Blocks **work → personal** clipboard export[^copypaste] | Configurable (Default Blocked) | ❌ | ❌ | ✅ |
| Blocks cross-profile **app interaction** (Connected Apps allowlist)[^xpa] | Only Freighter allowed | Configurable | Configurable | ✅ |
| Blocks cross-profile **widgets** (work widgets on the personal home screen)[^xpwidget] | ✅ | Configurable | Configurable | ✅ |
| Blocks outgoing **NFC Beam**[^beam] | n/a — minSdk 31 | ❌ | ❌ | n/a — minSdk 31 |
| Blocks outgoing **Bluetooth sharing** from the work profile[^bt-sharing] | ✅ | ✅ | ❌ | ✅ |
| Blocks **Bluetooth access to work contacts** (PBAP)[^bt-contacts] | ✅ | ✅ | ✅ | ✅ |
| Blocks cross-profile **contact search**[^contacts-search] | Configurable (Default Blocked) | Configurable | ❌ | ✅ |
| Blocks cross-profile **caller ID**[^callerid] | Configurable (Default Blocked) | ❌ | ❌ | ✅ |
| Blocks **Nearby notification** streaming[^nearby] | Configurable (Default Blocked) | ❌ | ❌ | ✅ |
| Blocks **Nearby app** streaming[^nearby] | Configurable (Default Blocked) | ❌ | ❌ | ✅ |

> [!NOTE]
> These are profile-owner-component-only policies (none is a delegatable `DELEGATION_*` scope), so this set of configurable permissions are essentially the one thing Fortress sets from the profile owner app itself, rather than via a companion app I've used for other features.

### 4b. Other managed-profile DPC powers

These are other policies a DPC could set, which are not cross-profile data transfer risks like the previous table.

- ✅ = the app exercises this `DevicePolicyManager` power
- ❌ = it does not, so the AOSP default (off) applies
- "Configurable" = the app exposes a user toggle (default off).

**For most rows ❌ is the privacy-preserving outcome!**[^policy-powers]

| DPC power (behavior) | Fortress | Shelter | Insular | Fortress-locked |
|---|:---:|:---:|:---:|:---:|
| Installs a CA certificate (TLS-interception capability)[^ca-cert] | ❌ | ❌ | ❌ | ❌ |
| Enables network activity logging[^net-log] | ❌ | ❌ | ❌ | ❌ |
| Hides/suspends apps in the work profile[^freezer-hide] | With Freezer | ✅ | ✅ | ❌ |
| Blocks uninstalling chosen apps (`setUninstallBlocked`) | ❌ | ❌ | ❌ | ❌ |
| Blocks force-stop of chosen apps (`setUserControlDisabledPackages`)[^api30] | ❌ | ❌ | ❌ | ❌ |
| Blocks screen captures profile-wide (`setScreenCaptureDisabled`)[^screencapture] | Configurable (Default Blocked) | ❌ | ❌ | ✅ |
| Pushes managed app configuration (`setApplicationRestrictions`)[^foreman-app-restrictions][^insular-app-restrictions] | With Foreman | ❌ | ✅ | ❌ |
| Disables keyguard features (`setKeyguardDisabledFeatures`) | ❌ | ❌ | ❌ | ❌ |
| Wipes the profile after N failed unlocks (`setMaximumFailedPasswordsForWipe`) | ❌ | ❌ | ❌ | ❌ |

### 4c. Delegated profile-owner scopes (`setDelegatedScopes`)

Delegation hands a specific profile-owner capability to another installed app, which then exercises it by calling the matching `DevicePolicyManager` APIs.

This *is* an attack-surface consideration, so delegating nothing is the most conservative posture.[^delegation]

The table here lists the 8 delegation scopes a BYOD work-profile owner can actually grant. Each cell shows which app the profile owner delegates that scope to, or ❌ for none. **Fortress** delegates only to its own optional companions and `fortress-locked` revokes all three,[^fortress-delegation] **Insular** delegates to *external* third-party apps via its `open` API,[^insular-delegation] and **Shelter** delegates nothing.

| Delegation scope | Fortress | Shelter | Insular | Fortress-locked |
|---|:---:|:---:|:---:|:---:|
| CA certificate install/management (`DELEGATION_CERT_INSTALL`) | ❌ | ❌ | ❌ | ❌ |
| Managed app configuration (`DELEGATION_APP_RESTRICTIONS`) | → Foreman | ❌ | ❌ | ❌ (revoked) |
| Block app uninstall (`DELEGATION_BLOCK_UNINSTALL`) | ❌ | ❌ | ❌ | ❌ |
| Grant/deny runtime permissions (`DELEGATION_PERMISSION_GRANT`) | ❌ | ❌ | → Greenify / Ice Box | ❌ |
| Suspend / hide packages (`DELEGATION_PACKAGE_ACCESS`) | → Freezer | ❌ | → Greenify / Ice Box | ❌ (revoked) |
| Enable built-in system apps (`DELEGATION_ENABLE_SYSTEM_APP`) | → Freighter | ❌ | ❌ | ❌ (revoked) |
| Network activity logging (`DELEGATION_NETWORK_LOGGING`) | ❌ | ❌ | ❌ | ❌ |
| Certificate selection (`DELEGATION_CERT_SELECTION`) | ❌ | ❌ | ❌ | ❌ |

> [!NOTE]
> Android has three additional delegations for device managers, but a BYOD profile owner like these can never grant them: `DELEGATION_INSTALL_EXISTING_PACKAGE` (the underlying API requires profile *affiliation*, which a BYOD work profile lacks), `DELEGATION_KEEP_UNINSTALLED_PACKAGES` (device-owner-only), and `DELEGATION_SECURITY_LOGGING` (device owner or organization-owned managed profile only).[^delegation]

## 5. App Permissions

This is how each app lets the profile owner influence the permissions of apps installed inside the Work Profile.

### 5a. Default app permissions (`setPermissionPolicy`)

`setPermissionPolicy` sets a single **profile-wide** default for how *future* runtime-permission requests are handled. `PROMPT` (the OS default), `AUTO_GRANT`, or `AUTO_DENY`.[^perm-policy]

As I [noted](README.md#delegated-admin-functions) in the README, this is *one* policy for the whole profile and *all* runtime permissions (not natively per-permission), and it does not change already-decided permissions. None of these apps set any permission policy, so every runtime permission sits at the OS default with no in-app control. The table is just a catalog of runtime permissions (and a template if our own apps' design changes).

- "Default" = left at the OS default (Prompt), not configurable in the app.
- "Granted" / "Denied" = the app forces that state.
- "Configurable" = the app lets you choose.
- 🔒 = a BYOD profile owner can only *deny or prompt* this group on Android 12+, never silently auto-grant it.[^perm-policy]

| Runtime permission (group) | Fortress | Shelter | Insular | Fortress-locked |
|---|:---:|:---:|:---:|:---:|
| Contacts | Default | Default | Default | Default |
| Calendar | Default | Default | Default | Default |
| Phone | Default | Default | Default | Default |
| Call log | Default | Default | Default | Default |
| SMS[^sms-perm] | Default | Default | Default | Default |
| Camera 🔒 | Default | Default | Default | Default |
| Microphone 🔒 | Default | Default | Default | Default |
| Location 🔒 | Default | Default | Default | Default |
| Physical activity 🔒 | Default | Default | Default | Default |
| Body sensors 🔒 | Default | Default | Default | Default |
| Nearby devices | Default | Default | Default | Default |
| Notifications | Default | Default | Default | Default |
| Photos & videos | Default | Default | Default | Default |
| Music & audio | Default | Default | Default | Default |
| Files & media (legacy storage) | Default | Default | Default | Default |

### 5b. Per-app permission control

Beyond the profile-wide default, these capabilities can change a *specific* app's permissions.

- ✅ = the app exercises it
- ❌ = it does not 
- "non-default" = requires a non-default privilege/setup.

| Capability | Fortress | Shelter | Insular | Fortress-locked |
|---|:---:|:---:|:---:|:---:|
| Reset / force an app's runtime-permission grant state (`setPermissionGrantState`)[^grantstate] | ❌ | ❌ | ✅ | ❌ |
| Revoke an app's raw app-ops — SMS / phone / location / storage (`AppOpsManager.setMode`)[^appops] | ❌ | ❌ | non-default | ❌ |

## 6. Security & Hardening Posture

In this section ✅ means the hardened posture is true.

| Property | Fortress | Shelter | Insular | Fortress-locked |
|---|:---:|:---:|:---:|:---:|
| No `INTERNET` permission | ✅ | ✅ | ✅ | ✅ |
| No `ACCESS_NETWORK_STATE` permission | ✅ | ✅ | ❌ | ✅ |
| No `QUERY_ALL_PACKAGES` | ✅ | ✅ | ❌ | ✅ |
| No `REQUEST_DELETE_PACKAGES` | ✅ | ❌ | ❌ | ✅ |
| No broad storage permission (`MANAGE_EXTERNAL_STORAGE`, legacy storage)[^minsdk] | ✅ | ❌ | ✅[^insular-debug] | ✅ |
| No overlay permission (`SYSTEM_ALERT_WINDOW`) | ✅ | ❌ | ❌ | ✅ |
| No `PACKAGE_USAGE_STATS` | ✅ | ❌ | ❌ | ✅ |
| No silent-install permission (`UPDATE_PACKAGES_WITHOUT_USER_ACTION`) | ✅ | ✅ | ❌[^sideplay] | ✅ |
| Every normal install path is user-confirmed | ✅ | ✅ | Partial (silent possible when affiliated/device-owner) | ✅ |
| Risky install/cross-profile capability isolated in optional app | ✅ (Freighter) | ❌ | ❌ | ✅ (Freighter is revoked) |
| Optional hard lockdown/killswitch build | With fortress-locked | ❌ | ❌ | ✅ |
| No third-party Android runtime libraries in app code | ✅ | ❌ | ❌ | ✅ |
| No analytics/telemetry/crash-reporting SDK dependency | ✅ | ✅ | ✅[^insular-analytics] | ✅ |
| Pinned/checksum-verified Gradle dependencies | ✅ | ❌ | ❌ | ✅ |
| Intentionally minimal Android permission footprint | ✅ | ❌ | ❌ | ✅ |
| No exported component intentionally granting third-party DPC-like powers | ✅ | ✅ | ❌ (open/delegated API surface) | ✅ |

## 7. Permissions Declared or Requested

This table lists every distinct Android/platform permission requested in source by the release/default apps I've compared here. Again, Insular entries are for the default F-Droid complete build unless marked otherwise. Custom app-defined permissions are listed immediately after the table.

| Permission | Fortress | Shelter | Insular | Fortress-locked |
|---|:---:|:---:|:---:|:---:|
| `android.permission.INTERNET` | ❌ | ❌ | ❌ | ❌ |
| `android.permission.ACCESS_NETWORK_STATE` | ❌ | ❌ | ✅ | ❌ |
| `android.permission.FOREGROUND_SERVICE` | ❌ | ✅ | ✅ | ❌ |
| `android.permission.FOREGROUND_SERVICE_SYSTEM_EXEMPTED` | ❌ | ✅ | ❌ | ❌ |
| `android.permission.GET_APP_OPS_STATS` | ❌ | ❌ | ✅ | ❌ |
| `android.permission.INTERACT_ACROSS_PROFILES` | Only Freighter Requests | ❌ | ✅[^insular-iap] | ❌ |
| `android.permission.INTERACT_ACROSS_USERS` | ❌ | ❌ | ✅ | ❌ |
| `android.permission.MANAGE_EXTERNAL_STORAGE` | ❌ | ✅ | ❌ | ❌ |
| `android.permission.NFC` | ❌ | ✅ | ❌ | ❌ |
| `android.permission.PACKAGE_USAGE_STATS` | ❌ | ✅ | ✅ | ❌ |
| `android.permission.POST_NOTIFICATIONS` | ❌ | ✅ | ✅ | ❌ |
| `android.permission.QUERY_ALL_PACKAGES` | ❌ | ❌ | ✅ | ❌ |
| `android.permission.READ_EXTERNAL_STORAGE` | ❌ | ✅ | `debug` builds only[^insular-debug] | ❌ |
| `android.permission.RECEIVE_BOOT_COMPLETED` | ❌ | ❌ | ✅ | ❌ |
| `android.permission.REQUEST_DELETE_PACKAGES` | ❌ | ✅ | ✅ | ❌ |
| `android.permission.REQUEST_INSTALL_PACKAGES` | ✅ (Fortress & Freighter Request) | ✅ | ✅ | ❌ |
| `android.permission.SYSTEM_ALERT_WINDOW` | ❌ | ✅ | ✅ | ❌ |
| `android.permission.UPDATE_PACKAGES_WITHOUT_USER_ACTION` | ❌ | ❌ | ✅[^sideplay] | ❌ |
| `android.permission.WAKE_LOCK` | ❌ | ❌ | ✅ (`maxSdkVersion=25`) | ❌ |
| `android.permission.WRITE_EXTERNAL_STORAGE` | ❌ | ✅ | `debug` builds only[^insular-debug] | ❌ |
| `android.permission.WRITE_SECURE_SETTINGS` | ❌ | ❌ | ✅ (non-default)[^insular-godmode] | ❌ |
| `android.permission.DISABLE_KEYGUARD` | ❌ | ❌ | `debug` builds only[^insular-debug] | ❌ |
| `android.permission.CHANGE_CONFIGURATION` | ❌ | ❌ | `debug` builds only[^insular-debug] | ❌ |
| `com.android.launcher.permission.INSTALL_SHORTCUT` | ❌ | ✅ (`maxSdkVersion=25`) | ✅ | ❌ |
| `com.android.launcher.permission.UNINSTALL_SHORTCUT` | ❌ | ❌ | ✅ | ❌ |

Android/platform permission counts, release/default builds:

| App/build | Count |
|---|---|
| Fortress app | 1 |
| Freighter | 2 Android permissions + 1 app-private signature permission |
| Freezer | 0 |
| Foreman | 0 |
| Fortress-locked | 0 |
| Shelter | 12 handwritten Android permissions, plus AndroidX generated permission below |
| Insular complete F-Droid[^insular-modular] | 17 distinct Android permissions, plus custom/generated permissions below |

### App-defined, Custom, and Component-Gating Permissions

| Permission or group | Fortress | Shelter | Insular | Fortress-locked |
|---|:---:|:---:|:---:|:---:|
| `${applicationId}.permission.BIND_CHANNEL` | With Freighter (signature) | ❌ | ❌ | ❌ |
| `com.oasisfeng.island.permission.INSTALLER` | ❌ | ❌ | ✅ (signature) | ❌ |
| `com.oasisfeng.island.permission.FREEZE_PACKAGE` | ❌ | ❌ | ✅ (dangerous; third-party API) | ❌ |
| `com.oasisfeng.island.permission.LAUNCH_PACKAGE` | ❌ | ❌ | ✅ (dangerous; third-party API) | ❌ |
| `com.oasisfeng.island.permission.SUSPEND_PACKAGE` | ❌ | ❌ | ✅ (dangerous; third-party API) | ❌ |
| `com.oasisfeng.island.permission-group.PACKAGE_ACCESS` | ❌ | ❌ | ✅ (permission group) | ❌ |
| `${applicationId}.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION`[^androidx-perm] | ❌ | ✅ | ✅ | ❌ |
| Component-gating permissions (`BIND_DEVICE_ADMIN`, `BIND_NFC_SERVICE`, `MANAGE_DOCUMENTS`, `DUMP`)[^bind-admin] | `BIND_DEVICE_ADMIN` on receiver only | `BIND_DEVICE_ADMIN` on receiver **and** exported IPC services; `BIND_NFC_SERVICE`, `MANAGE_DOCUMENTS`; `DUMP` (ProfileInstaller) | `BIND_DEVICE_ADMIN` on receiver **and** admin/provisioning components; `INTERACT_ACROSS_USERS_FULL` | `BIND_DEVICE_ADMIN` on receiver only |

[^lockdown]: `fortress-locked` is an in-place update for Fortress (same package + key, max `versionCode`), it is **not** a fresh provisioner. This means it inherits the profile Fortress already created and keeps profile-owner continuity (identical `DeviceAdminReceiver` class + metadata) while stripping capability. Its teardown revokes Freighter's cross-profile allowlist and the Freighter/Freezer/Foreman delegations, and it clears any cross-profile intent filters; it never wipes the profile. (Conversely, *uninstalling* it would destroy the profile, since it is the profile owner.)

[^freezer-lockdown]: Freeze/suspend in my app suite is provided by the optional **Freezer** companion app, which Fortress grants `DELEGATION_PACKAGE_ACCESS`. Installing `fortress-locked` **revokes** that delegation, so Freezer can no longer suspend/hide (or un-suspend/un-hide) apps after lockdown. Any apps Freezer had already hidden/suspended stay in that state (the per-package system state persists, and nothing in the locked suite can reverse it).

[^fortress-install]: Fortress has no APK-file picker. It installs the bundled Freighter (only with AOSP-enforced user confirmation), clears the work-profile unknown-sources restriction so the user can sideload inside the profile with their own installer/browser, and lets Freighter clone already-installed apps. Installing an arbitrary APK file directly is left to the user's chosen installer inside the work profile, Freighter doesn't provide an APK file picker.

[^curated]: Freighter enables only a hardcoded, curated whitelist of built-in system apps (e.g. Vanadium, camera, gallery, clock, calculator). This is because the app deliberately holds no `QUERY_ALL_PACKAGES` permission, so it's unable to enumerate every installed/system app to offer an "enable any system app" picker. The curated list is the trade-off for that minimal package-visibility posture. If you absolutely need a system app here, you can open an issue requesting it be added to the whitelist.

[^autofreeze]: "Auto-freeze on screen-off/lock" freezes apps via `setApplicationHidden`, it is **not** the same as pausing/turning off the work profile. The profile keeps running, and its encryption keys remain loaded. In Shelter/Haven only the apps the user marked to freeze in settings are frozen on lock, and they are then re-opened normally via the "unfreeze and launch" shortcut. Insular has no native screen-off auto-freeze and delegates this to the separately installed third-party Greenify app.

[^market-clone]: "Cloning" reproduces an **already-installed** app's identity (same package id) in the other profile without re-sourcing it. Fortress's Freighter does this by streaming the installed APK (including split APK support) over the cross-profile channel into a `PackageInstaller` session, no app store involvement. Insular additionally offers a **market-assisted fallback**: when it can't install directly it will open `market://details?id=<pkg>` in the target profile so a store fetches the same package (used especially in its Google Play build, which strips `REQUEST_INSTALL_PACKAGES`). Note the distinction: installing an app fresh from Google Play *inside* the work profile is an ordinary new install into that profile's own Play instance. Nothing is copied from the personal profile, so it is **not** the clone feature, which is why Fortress is ❌ here. Shelter/Haven clone via their cross-profile File Shuttle with `PackageInstaller` and have no programmatic store fallback.

[^nfc]: Shelter's "payment-stub workaround" (added in v1.9, **off by default**) registers a do-nothing Host Card Emulation service in the *personal* profile under `CATEGORY_PAYMENT` with a fake AID. Android permits only one default payment service, chosen in Settings, and historically that selector was unactionable when the personal profile had no payment service, thereby blocking selection of a work-profile wallet. Toggling the stub gives the personal profile a (no-op) payment service so the Settings selector then becomes usable. Shelter makes no `setPreferredService`/`CardEmulation` call and does not route payments. Whether Google Pay/Wallet then actually taps-to-pay from the work profile is **not guaranteed**: work-profile HCE behavior is OEM- and Android-version-dependent (some devices won't bind a work-profile HCE service at all) and Wallet imposes its own secondary-profile restrictions. It's basically a best-effort, historical workaround rather than a reliable feature.

[^xprofile]: `DISALLOW_SHARE_INTO_MANAGED_PROFILE` gates the personal → work direction: it controls the system's default `ACTION_SEND`-into-profile forward **and** the clipboard mirror *into* the work profile (AOSP `ClipboardService` copies a clip into a related profile only when this restriction is absent). It is therefore the lever that would block personal→work sharing/paste. **Fortress exposes a user toggle for it, and `fortress-locked` sets it to block unconditionally.** Shelter/Haven and Insular do not set it, so that direction works by default for them. It does not affect filters an app adds via `addCrossProfileIntentFilter`. App-added forwarding *beyond* the AOSP defaults: Fortress none (the locked build clears filters); Shelter/Haven a few (incl. work → personal browser links) plus an opt-in File Shuttle; Insular broad bidirectional `VIEW`/`SEND`.

[^insular-godmode]: Insular can optionally become **device owner** ("Managed Mainland", controlling the primary user's apps) or use **root/Shizuku** for extra capabilities (e.g. cloning without affiliation, making `WRITE_SECURE_SETTINGS` effective). These require a one-time ADB/root setup and are **off by default**. So I marked it "non-default" rather than counted as ordinary BYOD behavior.

[^insular-suspend]: Insular's `setPackagesSuspended` exists and is used internally, but the user-facing "Suspend/Unsuspend" menu action is gated behind `BuildConfig.DEBUG` (`AppListViewModel.java:269`), so it is **not exposed in the release F-Droid build**.

[^insular-modular]: The shipped F-Droid build (`completeFdroid`) merges the `assembly`, `complete`, `engine`, `mobile`, `installer`, `watcher`, `open`, `sideplay`, and `shared` modules into one APK. It is not a single-purpose, minimal DPC.

[^sideplay]: `sideplay` is a code-less, manifest-only Insular module bundled into the `complete` F-Droid build; it is what contributes `UPDATE_PACKAGES_WITHOUT_USER_ACTION` (alongside `INTERACT_ACROSS_PROFILES` and `REQUEST_INSTALL_PACKAGES`). Insular's **Google Play** distribution flavor removes these three permissions.

[^insular-debug]: These permissions are declared only in Insular's `assembly/src/debug` test manifest (for UI-test screenshots/keyguard control) and are absent from the released `completeFdroid` build, they cannot appear in a normal release.

[^insular-iap]: Insular declares `INTERACT_ACROSS_PROFILES` for its "Shuttle" cross-profile channel (`ShuttleCarrierActivity` calls `CrossProfileApps.startActivity` into the parent profile). As profile owner it never uses the Settings "Connected work & personal apps" consent flow. Instead, it self-allowlists and self-grants the app-op via its own DPC (`setCrossProfilePackages` & AppOps `MODE_ALLOWED`) on its own package. A profile owner is structurally excluded from the Connected Apps consent model (it cannot consent for itself), and Insular's own allowlist UI filters itself out, so Insular-as-profile-owner never appears in Connected Apps. The Google Play build strips this permission.

[^insular-analytics]: This fork keeps analytics/crash *abstraction* calls in source, but the implementations are no-ops, the leftover Google-Analytics resource is deleted at build time, and Firebase/GMS paths are hard-disabled/firewalled. No analytics/Firebase/GMS SDK are dependencies, and no `INTERNET` permission is declared.

[^androidx-perm]: Not hand-written in the app's source manifest, instead **injected by AndroidX** into the merged release manifest. AndroidX `core` generates `${applicationId}.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION` (guards its internal runtime receivers), and AndroidX `profileinstaller` (pulled transitively) adds a receiver gated by `android.permission.DUMP`. Listed for permission-completeness.

[^bind-admin]: `BIND_DEVICE_ADMIN` is a system-held, signature-level permission. Every DPC **must** put it on its `DeviceAdminReceiver` so only the OS can bind/broadcast to the admin component. That mandatory use is the *only* use in Fortress (one component). Shelter and Haven additionally reuse it as a **cross-profile IPC guard** on their exported worker services (`ShelterService`/`HavenService`, `FileShuttleService`). Because the system holds the permission and the two profile instances share a signing key, the other-profile instance can bind those services while third-party apps are kept out. Insular also gates four components, but they are admin/provisioning/persistent-service plumbing (`IslandPersistentService` `DEVICE_ADMIN_SERVICE`, `IslandProvisioning$CompletionActivity`, the admin receiver, and the `open`-module `DelegatedScopeAuthorization` receiver), not an exported file/clone worker. So Fortress's "on receiver" is strictly narrower than the others' bare "uses `BIND_DEVICE_ADMIN`".

[^minsdk]: Fortress's **minSdk 31** baseline means it never runs on the Android 7–11 (API 24–30) work-profile surface that the others (Shelter/Haven minSdk 24, Insular 27) must support (this was a deliberate threat-model simplification). Our floor is **31 rather than 30** because on Android 12+ (API 31) the platform **bars a profile owner from *granting* the sensors-related runtime permissions** (`CAMERA`, `RECORD_AUDIO`, location, `ACTIVITY_RECOGNITION`, `BODY_SENSORS`) via `setPermissionGrantState` (instead, a PO may only deny them) so pinning > 30 structurally removes the ability of any Fortress code path (including a malicious future update) to silently grant a sensitive sensor permission to a work-profile app. An app reaching API 24–29 also deals with several legacy behaviors that are **n/a for a minSdk-31 app**: **(1)** Legacy external storage: scoped storage is only *enforced* at API 30, so a minSdk-24 app can use direct-path `READ`/`WRITE_EXTERNAL_STORAGE` for cross-profile file features on ≤ API 29 (and falls back to `MANAGE_EXTERNAL_STORAGE` on 30+). Fortress is scoped-storage-only and requests neither. **(2)** Unknown-sources install was a single global setting before API 26, becoming per-app `REQUEST_INSTALL_PACKAGES` at 26 (`DISALLOW_INSTALL_UNKNOWN_SOURCES_GLOBALLY` added 29). **(3)** `DISALLOW_BLUETOOTH_SHARING` and its default block only exist from API 26 (see [^bt-sharing]). **(4)** Android Beam is a live outbound NFC channel on API 24–28 (see [^beam]). **(5)** Hidden-API (non-SDK) reflection is unrestricted on API 24–27, greylisted from API 28 (see [^appops]). **(6)** Provisioning finalize: the `ACTION_PROVISIONING_SUCCESSFUL` activity hand-off is API 26, so on API 24–25 a DPC finalizes from the `onProfileProvisioningComplete` broadcast alone, which is why Shelter/Haven ship a `FinalizeActivity`.

[^xpa]: Cross-profile app interaction (the "Connected work & personal apps" / `INTERACT_ACROSS_PROFILES` allowlist) is **empty by default**, no app can interact across the boundary until the profile owner allowlists it via `setCrossProfilePackages`. **Fortress** hard-allowlists exactly one app: its own **Freighter** cloner (we add no user-facing allowlist management, this is the Freighter cross-profile grant the app uses for cloning apps), so it is "Freighter only", and `fortress-locked` revokes even that (✅). **Shelter, Haven, and Insular** expose a per-app UI to allowlist arbitrary apps for cross-profile interaction, therefore marked **Configurable**.

[^xpwidget]: By default no widget provider package is allowlisted, i.e. a work-profile app's widgets cannot appear on the personal home screen until the profile owner whitelists its package. **Shelter, Haven, and Insular** expose a per-app toggle for this, therefore **Configurable**. **Fortress** and `fortress-locked` leave the default in place (blocked).

[^copypaste]: Cross-profile clipboard is **bidirectional by default**. AOSP `ClipboardService` mirrors a copied clip into the related profile at *copy* time (there is no "read the other profile's clipboard" path), gated by two restrictions that both default OFF: `DISALLOW_CROSS_PROFILE_COPY_PASTE` (this row), set on the work profile, blocks the **work → personal** export. The **personal → work** direction is gated by `DISALLOW_SHARE_INTO_MANAGED_PROFILE` (the row above). By default, no app here sets either, so copy/paste works **both ways**. **Fortress exposes a user toggle for the work→personal export (this row, `DISALLOW_CROSS_PROFILE_COPY_PASTE`) and one for the personal→work direction (the row above), both default to BLOCKED; `fortress-locked` forces them blocked.** Shelter/Haven and Insular set neither. A profile owner can block each direction independently with the corresponding restriction.

[^beam]: Android Beam (NFC push) was a live outbound channel through **Android 9 (API 28)**, but the **feature was disabled by default in Android 10 (API 29)**, gated behind `android.software.nfc.beam`, which AOSP/Pixel/GrapheneOS do not declare ("not coming back"), and the APIs were **removed in Android 14 (API 34)**. So on AOSP/Pixel/GrapheneOS it is **non-functional from Android 10 onward**: for the minSdk-24/27 apps it is a real outbound channel only on Android 7–9 (where a profile owner must set `DISALLOW_OUTGOING_BEAM` to block it), while in Fortress's minSdk-31 range there is no Beam to block, hence **n/a**. Default off; no app sets the restriction.

[^bt-sharing]: `DISALLOW_BLUETOOTH_SHARING` (outgoing Bluetooth/OPP file sharing) **defaults to ON for managed profiles**, meaning AOSP already blocks outgoing Bluetooth sharing from a work profile, and Fortress/Shelter/Haven leave that default in place (✅). **Insular is the exception:** it explicitly *clears* this restriction at provisioning (`IslandProvisioning.java:279`), re-allowing outgoing Bluetooth sharing from the work profile (❌). The key was introduced in API 26 (Android 8.0), so on **API 24–25 no equivalent restriction existed**, and outgoing Bluetooth sharing was not blocked by default there (relevant only to the minSdk-24 apps installed on Android 7.x).

[^bt-contacts]: `setBluetoothContactSharingDisabled` defaults to **disabled = true** for managed profiles. Bluetooth (PBAP) devices cannot access work-profile contacts by default. No app changes this, so all are ✅ by AOSP default.

[^contacts-search]: AOSP **allows** cross-profile contact search by default (work contacts can surface when searching from the personal side). Shelter and Haven expose a Settings toggle to disable it (`setCrossProfileContactsSearchDisabled`, deprecated in API 34+ in favor of `setManagedProfileContactsAccessPolicy`). **Fortress also exposes a toggle (default blocked) and `fortress-locked` forces it blocked**, using the non-deprecated `setManagedProfileContactsAccessPolicy(PackagePolicy)` (an empty allowlist) on API 34+ and the deprecated boolean setter on 31–33. Insular does not, so its default (allowed) stands.

[^callerid]: AOSP **allows** cross-profile caller ID by default (the personal dialer can look up work-profile contacts to name incoming calls). **Fortress exposes a toggle (default blocked) and `fortress-locked` forces it blocked**, using `setManagedProfileCallerIdAccessPolicy(PackagePolicy)` (empty allowlist) on API 34+ and the deprecated `setCrossProfileCallerIdDisabled` on 31–33. No other app here sets it.

[^nearby]: `setNearbyNotificationStreamingPolicy` / `setNearbyAppStreamingPolicy` (both **added in API 31**, so available at Fortress's minSdk with no version gate, control whether work-profile notifications / apps may be streamed to and run on a nearby device. Default is `NEARBY_STREAMING_NOT_CONTROLLED_BY_POLICY` (allowed). **Fortress exposes a toggle for each (default `NEARBY_STREAMING_DISABLED` but can be re-enabled) and `fortress-locked` forces it disabled**. No other app here sets either.

[^screencapture]: `setScreenCaptureDisabled(admin, true)` blocks screenshots and screen recording while a work-profile app is on screen. Called on the PO's own instance it is **work-profile-scoped** (not device-wide) **Fortress exposes a toggle (default is screenshots are blocked) and `fortress-locked` forces it blocked.** No other app here sets it.

[^policy-powers]: These are `DevicePolicyManager` powers a work-profile owner *could* wield. ✅ = the app exercises it; ❌ = it does not (AOSP default = off). For a minimal, privacy-respecting DPC, ❌ is the expected/safe result on nearly every row, note that the invasive capabilities (CA-cert install, network logging) are ❌ for **all four** apps.

[^ca-cert]: None of the four install a CA certificate. A DPC-installed CA in the work profile would enable TLS interception of that profile's traffic, so its absence is the privacy-preserving state.

[^net-log]: None of the four enable network logging. (Work-profile network logging via `setNetworkLoggingEnabled` is available to a profile owner only on Android 12+, and would capture the work profile's connect/DNS activity.)

[^freezer-hide]: Hiding/suspending apps is provided in the Fortress suite by the optional **Freezer** add-on. Shelter/Insular/Haven do it natively. `fortress-locked` revokes Freezer's delegation, so the locked suite can no longer hide/suspend apps at all, see [^freezer-lockdown].

[^insular-app-restrictions]: Insular uses `setApplicationRestrictions` as part of its `open` module's delegated-scope API for third-party apps (`DelegationManager.kt`), not as a hardening measure. It is the same exported delegated-DPC surface I've flagged in section 6.

[^foreman-app-restrictions]: Managed app configuration in the Fortress suite is provided by the optional **Foreman** add-on, which Fortress grants `DELEGATION_APP_RESTRICTIONS`. Foreman reads each work-profile app's declared restriction schema on-device via `RestrictionsManager.getManifestRestrictions` (an unprivileged read) and writes values with `setApplicationRestrictions(null, …)`. The current app is auto-discover only: it renders a typed form from the app's manifest schema (a raw key/value editor is deferred). It holds **zero Android permissions** (no `INTERNET`, no `QUERY_ALL_PACKAGES`). `fortress-locked` **revokes the delegation** on lockdown, however, values already pushed persist (per-package system state), but Foreman can make no further changes. Unlike Insular ([^insular-app-restrictions]), this is a same-suite companion holding the scope itself, **not** an exported delegated-DPC API offered to external apps.

[^perm-policy]: `setPermissionPolicy(admin, policy)` sets a profile-wide default [`PERMISSION_POLICY_PROMPT` (0, the OS default), `AUTO_GRANT` (1), or `AUTO_DENY` (2)] for how *future* runtime-permission requests from apps (targeting Android 6+) are handled; it does not affect already-granted/denied permissions and is not per-permission. A BYOD profile owner can set it.

[^sms-perm]: Only `READ_SMS` is deny-only for a managed-profile owner (Android 12+); the other SMS permissions (`SEND_SMS`/`RECEIVE_SMS`/etc.) are fully auto-grantable, so the SMS group is *partially* restricted, not fully 🔒.

[^grantstate]: `setPermissionGrantState` lets a profile owner silently set a *specific app's specific* runtime permission to granted or denied (locking the user out of changing it in Settings), or reset it to the user-managed default. **Insular** uses it, chiefly as a "reset/unlock" action that walks every app and clears any admin-pinned grant state back to default (temporarily un-freezing hidden apps, since the API skips hidden ones).

[^appops]: **Insular**'s raw app-ops editor revokes specific app-ops (Read SMS, Read phone state, Location, Storage) per app via the hidden `AppOpsManager.setMode` (reflection). It needs the privileged `GET_APP_OPS_STATS` permission, which an ordinary profile owner can't hold (the user must grant it out-of-band via **root or ADB** with `pm grant`), and Insular only enables it on **Android 9–10**. Without it the feature silently no-ops and just records intent locally. Hence **non-default**. (Hidden-API reflection like this is unrestricted on API 24–27 but **greylisted/throttled from API 28**, which is part of why Insular caps the feature at Android 9–10.)

[^api30]: `setUserControlDisabledPackages` was added in **API 30 (Android 11)**. It did not exist for any profile owner on API 24–29, so this capability is unavailable to *all* the apps on the lower end of their supported range (none use it on API 30+ either).

[^delegation]: `setDelegatedScopes(admin, package, scopes)` grants a profile-owner capability to another installed app, which then calls the corresponding `DevicePolicyManager` APIs with a `null` admin. The delegate wields real PO power for that scope, so each grant is an attack-surface consideration and granting none is the conservative default. Of the **11** scopes, a personally-owned **BYOD** work-profile profile owner can actually grant **8**.

[^fortress-delegation]: Fortress delegates only to its own optional companion apps. `DELEGATION_ENABLE_SYSTEM_APP` → **Freighter**, `DELEGATION_PACKAGE_ACCESS` → **Freezer**, and `DELEGATION_APP_RESTRICTIONS` → **Foreman**. `fortress-locked` **revokes all three** on lockdown (`setDelegatedScopes(..., emptyList())` for each). The delegates are distinct, separately-signed packages in the same suite, granted only when the companion is installed. Nothing is delegated to arbitrary third parties.

[^insular-delegation]: Insular's `open` module exposes an API that delegates `DELEGATION_PACKAGE_ACCESS` and `DELEGATION_PERMISSION_GRANT` to **external third-party apps**, signature-pinned to `com.oasisfeng.greenify` and `com.catchingnow.icebox` (Ice Box) (`ApiDispatcher.java:195-199`). It also defines a *custom, non-DPM* `-island-delegation-app-ops` pseudo-scope routed through `setApplicationRestrictions`, not `setDelegatedScopes`. Delegating profile-owner powers to *external* apps is a broader exposure than delegating to one's own companion apps.

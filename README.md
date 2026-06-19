# Secure & Personal Android Work Profiles

This is a suite of apps I've built to create and manage an Android "Work" Profile. Apps such as these are popular among people who want to install two copies of an app on their device, have a separate de-Googled container for their apps, want to keep a subset of their apps locked behind a separate password/fingerprint, and more.

The flagship app in this suite is [**Fortress**](https://github.com/jonaharagon/fortress), a minimal-as-possible Device Policy Controller which allows you to create a Work Profile that is personally managed, rather than managed by an employer and/or complex mobile management server.

> [!TIP]
> In the process of developing this app (deciding whether it was worth doing at all) I've taken notes to compile (what I believe is) an honest and complete comparison of this suite of apps versus the popular alternatives (Shelter and Insular/Island). [**Read that comparison →**](https://github.com/jonaharagon/work-profiles/blob/main/COMPARISON.md)

## The goal

I built these apps specifically to address a longstanding problem I've had with other apps that perform a similar function (allow people to create non-employer-managed Work Profiles): The large attack surface, custom features which bypass AOSP's native isolation between Work and Profile spaces, and the lack of modernization when it comes to modern (sdk > 30) features that Android Work Profiles support.

> [!WARNING]
> The security features are only as strong as the Work Profile implementation of the Android OS you are using. Some OEMs may strip capabilities or have non-functional Work Profile implementations, which I cannot resolve.

The value proposition of Fortress is **security through minimalism.** I believe I've achieved this in the following ways:

1. **Keeping the codebase tiny.**
   
   I've attempted to make each app as minimal as Android's work profile management framework allows, with the goal being to make all of these apps as auditable by any security-conscious user as possible.

2. **Keeping the suite modular.**

   Yes, this is *seemingly* at odds with my minimalism goal, but the reason this is split into multiple apps is simple: You never need to install more code than you personally require. Further on in this README I will explain how the modularity works, and how it remains secure despite needing intra-app communications, because it works in an entirely different way to another older, "modular" Work Profile management app. I believe my approach here is far more robust and AOSP-sanctioned.

   In addition to limiting the amount of code you're required to install, modularity serves another important purpose: Moving risky capabilities, like installing arbitrary apps across the profile boundary, outside the highly privileged "profile owner" app (Fortress) into another app which only has as many permissions as strictly required. Again, more on this further on.

3. **Shifting everything possible to AOSP sanctioned APIs.**

   Many alternative Work Profile management apps implement many features, usually for convenience reasons. Many of these features do not take advantage of modern APIs in Android for Personal<>Work data sharing.
   
   For example, Shelter has built a custom bridge between its app in the personal profile and the profile owner version of the app in the Work Profile to share data across (like app cloning). The Freighter app in this repo instead uses Android's Connected Apps API, an AOSP-sanctioned approach to cross-profile data sharing that was introduced in Android 11, which is enforced by the OS and controlled in your OS Settings app, not my Freighter app.
   
   Shelter and similar apps are of course older than Android 11 and their approach may have made sense at the time, but stronger tools now exist that we can take advantage of here.

4. **Enforcing *much* more secure cross-profile isolation.**

   Fortress implements many features which to my knowledge all alternative apps either simply never implemented, or only implemented a small subset of possible isolation tools. Fortress can block cross-profile share sheet and "open with" sharing, cross-profile clipboard, screen captures across the entire Work Profile, cross-profile contact searching, cross-profile caller ID, and more.
   
   In Fortress, this isolation is configurable. We have another app which *strictly* enforces this isolation, more on this later.

## The apps

> [!IMPORTANT]
> Fortress and its companion apps strictly **only** make use of Work Profile APIs exposed by AOSP. I will **not** implement features which require becoming your device owner or taking over your personal profile, leveraging or abusing adb privileges, implementing features which require root, or delegating privileged tasks to third-parties. These apps will **only** ever make use of APIs which AOSP has sanctioned for a Work Profile admin's use.

### Fortress

**Fortress** is the tool which actually creates and owns the work profile (profile owner). Fortress has no app cloning capability: You are intended to create a work profile using Fortress, install an app store or web browser, and then install any apps you require in your work profile from there.

However, this presents a chicken-and-egg situation on **some** systems: Fortress does not modify the apps which are initially installed in your Work Profile, this is decided by your Android OS. This could mean that with some OEMs, you could end up with no web browser or app store installed at all, leaving it impossible to install the rest of your apps!

For this reason, Fortress comes *bundled* with Freighter (the next app)'s APK. You can install Freighter in your Work Profile by tapping a button in the Fortress app. This installation is gated by you needing to "allow installs from unknown sources" for Fortress, and by you needing to manually consent to installing Freighter through Android's typical app install prompt.

> [!NOTE]
> [**You should read Fortress' own README in full for an explanation of the app's security and features →**](https://github.com/jonaharagon/fortress)

### Freighter

**Freighter** is an app-cloning tool, which has two functions: Installing system apps in your Work Profile that may not have been installed by default, and cloning apps from your Personal profile to your Work Profile. To enable *system* apps, Freighter only has to be installed in your Work Profile. To clone apps, Freighter must be installed in your Work Profile and your Personal profile.

Freighter differs from other app cloning solutions in an important way: Instead of needing to use a highly privileged channel between the profile owner's (Fortress) app and the Work Profile copy of it, Freighter uses Android's native *Connected Apps* API which connects apps in a Personal profile and a Work Profile.

A big advantage of this is that Connected Apps can be enabled and disabled in your Android Settings, meaning you can disable Freighter's Connected Apps privilege at-will and all Personal<>Work connection will be severed, enforced by the operating system.

> [!NOTE]
> [**You should read Freighter's own README in full for an explanation of the app's security and features →**](https://github.com/jonaharagon/freighter)

### Freezer (Coming Soon)

**Freezer** allows you to suspend or hide individual apps in your Work Profile. In either case, the app will no longer be permitted to run. The only difference is that suspended apps will remain grayed out in your launcher, whereas hidden apps will be hidden from your launcher entirely.

> [!NOTE]
> [**You should read Freezer's own README in full for an explanation of the app's security and features →**](https://github.com/jonaharagon/freezer)

### Foreman (Coming Soon)

**Foreman** allows management of application restrictions, a unique privilege a Work Profile admin has. This feature is of fairly limited use, but most notably it allows you to change many configuration options for Chromium/Vanadium which are otherwise not normally exposed. "Application restrictions" are a specific feature intended for device admins, and most apps do not support them, so you may not find this app useful.

> [!NOTE]
> [**You should read Foreman's own README in full for an explanation of the app's security and features →**](https://github.com/jonaharagon/foreman)

### Fortress-locked

**Fortress-locked** is an in-place upgrade for Fortress which **strictly** enforces as-complete-as-possible cross-profile isolation, and the **smallest possible attack surface.** The entire source code of this app essentially does two things: Enable all cross-profile isolation features we can, revoke Freighter's, Freezer's, and Foreman's abilities, and keep the Work Profile alive.

Normally, if you install a Work Profile you can never uninstall the admin app without deleting that Work Profile, making the admin app a perpetual security risk. Fortress-locked resolves this problem as best as possible, because it essentially functions as an inert app to keep the profile alive and nothing else.

Fortress-locked is signed with the same key as Fortress and has the same package ID. The difference is that practically everything is removed, and it has a super high version number so that it is never auto-updated by app stores and can never be downgraded to a regular Fortress install. My belief is this is the best possible Work Profile implementation for privacy and cross-profile isolation.

However, this also has very significant drawbacks for convenience and in other areas. I believe **regular Fortress is more than hardened enough** for most people, but Fortress-locked may be an interesting option for the *most* advanced and *most* security-conscious users out there.

> [!CAUTION]
> Installing Fortress-locked is an IRREVERSIBLE modification to your Fortress installation. It is absolutely essential you read the full explanation and instructions **BEFORE INSTALLING** the app.
>
> [**You should read Fortress-locked's own README in full for an explanation of the app's security and features →**](https://github.com/jonaharagon/fortress-locked)

## Delegated Admin Functions

All the **companion** apps to Fortress (Freighter, Freezer, Foreman) receive the ability to perform their functions through *delegations.* The [DevicePolicyManager API](https://developer.android.com/reference/android/app/admin/DevicePolicyManager#DELEGATION_APP_RESTRICTIONS) allows a profile owner (Fortress) to grant access to 8 different sets of privileges to other apps in the profile. Four of these grants are useful from a privacy or personal management perspective, and we implement three of them.

**Freighter** is granted `DELEGATION_ENABLE_SYSTEM_APP`, which essentially does what it says on the tin: It allows Freighter to enable a system app in your new profile which isn't already there. Freighter does not require the delegation of any other admin-only privileges in order to function. Its app cloning feature is powered by the standard "allow apps from this source" permission you can grant to any app installer on your device, and the "Connected Apps" API, which *does* need to be enabled by the privileged Fortress app, but is not a privileged or dangerous permission for Freighter to possess.

**Freezer** is granted `DELEGATION_PACKAGE_ACCESS`, which allows it to use the `get`/`setApplicationHidden` and `get`/`setPackagesSuspended` APIs. These APIs grant it the functionality to manage an app in your Work Profile's access state.

**Foreman** is granted `DELEGATION_APP_RESTRICTIONS`, which allows it to use the `get`/`setApplicationRestrictions` APIs. These APIs grant it the functionality to set application restrictions on any app which supports them (few, to be honest).

For **all of the above**, using delegations allows those apps to receive only the exact admin privileges they need to function. This is a big advantage over the typical model where the profile owner app implements all the code to clone apps, freeze apps, etc. on its own.

Implementing all this code in the core profile owner app opens up a massive attack surface where a vulnerability in any of those functions could allow for some exploit to escalate to full profile owner permissions, so the delegation + companion app setup is intended to minimize that risk as much as we can, while retaining as many features you might expect from a Work Profile manager as we can.

These delegations are granted only to the three specific apps above, Fortress will enforce delegations only to apps with the same package ID and same APK signing signature as these apps, so other apps cannot obtain the delegation grants.

### Managing app permissions (we don't do it)

As an aside, `DELEGATION_PERMISSION_GRANT` is the fourth of the 4 grants which could be used for privacy/security reasons that I just mentioned, but we do not implement it. This delegation is actually one of the very few that some Fortress alternatives *do* take advantage of, namely, Insular/Island grants this permission to some third-party apps (Greenify/Ice Box).

The permissions this grants would allow a companion app to (1) set the runtime permissions (essentially anything you normally see a prompt for: camera, microphone, photos, etc.) for another specific app in the Work Profile, and (2) set a "permissions policy" which would apply a runtime permission default for the *entire* Work Profile.

I considered making a companion app which would allow you to set a "permissions policy," however, when researching this I discovered that the `setPermissionPolicy()` API **only** allows you to set a blanket default for *every* single runtime permission altogether, **not** per permission type. For example, I was not able to disable apps from asking for camera access *specifically*, I would have to stop it from asking for any runtime permission.

As a result, I believe the utility of such a companion app to be fairly limited: The only plausible security advantage would be preventing you from fat-fingering a permission grant prompt, but the convenience trade-off does not seem worth it. Additionally, the *other* feature (setting *per-app* permissions) does not make much sense to me: Other alternatives to Fortress actually do make use of this functionality, but I think the same is achieved by simply visiting the app's permissions screen under App Info in Settings.

If you want to block camera/microphone or other sensor access across all your apps, I suggest you take advantage of your OS's built-in tools.

> [!NOTE]
> `DELEGATION_PERMISSION_GRANT` does **not** allow us to modify **install**-time permissions. That is, we can **not** modify permissions that Android grants automatically to any app by default without a prompt. This includes whether an application is able to access the network. I have looked into this, and I do not believe it is possible for me to deny a Work Profile app's `INTERNET` permission with any available APIs I can find. Therefore, we will not entertain any feature requests asking for network-blocking functionality. Your best option is probably to use an always-on "VPN" which simply runs a local process to block any connections. Finding such an app is left as an exercise for the reader.

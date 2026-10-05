# tauri-sidecar-entitlements-repro

Minimal reproduction: on macOS, Tauri signs sidecars (`bundle > externalBin`) with the app's entitlements. When the app has a restricted entitlement, the sidecar is killed at launch.

The app has one sidecar, `hello`, which prints one line. `src-tauri/Entitlements.plist` holds one normal entitlement and one restricted entitlement (`com.apple.developer.associated-domains`). The bundle is signed ad hoc, so no Apple Developer account is needed.

## Steps

```sh
rustc -O sidecar/hello.rs -o "src-tauri/binaries/hello-$(rustc -vV | sed -n 's/^host: //p')"
npm install
npx tauri build --bundles app

APP="src-tauri/target/release/bundle/macos/Sidecar Repro.app"
codesign -d --entitlements - "$APP/Contents/MacOS/hello"
"$APP/Contents/MacOS/hello"; echo "exit=$?"
```

## Result with `@tauri-apps/cli` 2.12.1

The sidecar carries both app entitlements, and the system kills it:

```
[Dict]
	[Key] com.apple.developer.associated-domains
	[Value]
		[Array]
			[String] applinks:example.com
	[Key] com.apple.security.device.audio-input
	[Value]
		[Bool] true
exit=137
```

`log show` reports the reason:

```
amfid: Restricted entitlements not validated, bailing out. Error: Error Domain=AppleMobileFileIntegrityError Code=-427 "Unable to retrieve certificate chain"
```

That is the ad-hoc variant of the failure. In a build signed with a Developer ID certificate and an embedded provisioning profile, the main executable is accepted and the sidecar fails with:

```
amfid: Restricted entitlements not validated, bailing out. Error: Error Domain=AppleMobileFileIntegrityError Code=-413 "No matching profile found"
```

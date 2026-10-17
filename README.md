# Android app builds

Signed builds of the apps, and each app's website, privacy policy and terms (served by GitHub
Pages from the app's folder).

## Speech Recorder

- Website: <https://longtermsupport.github.io/android-apps-releases/speech-recorder/>
- Latest APK, always: <https://github.com/LongTermSupport/android-apps-releases/releases/latest/download/speech-recorder.apk>

## Publishing a build

Tag the release `speech-recorder-vX.Y.Z` and attach the APK twice: as `speech-recorder-vX.Y.Z.apk`
and as `speech-recorder.apk`. The second, stable name is what makes the latest link above work.

GitHub's "latest" is the newest release in the whole repository, not per app. Whilst Speech
Recorder is the only app here the link is always its newest build. When a second app is added,
publish a release for one app straight after the other's, or replace the link with a page that
asks the releases API for the newest release of that app's tag prefix.

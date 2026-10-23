# Smm2-por-Android-beta-por-IA- código #!/bin/bash
# android_beta.sh (BETA): builds SMM2 Recomp for Android from your own dump, installs it and copies your game
# files to the device, in one go. Put it in recomp/tools/ and run it from the repository root.
#
# Added to SMM2 Recomp 0.4.6 on 2026-10-23; GPL-3.0-or-later like the rest of the project (this notice marks the
# file as added to the original). It only calls the project's own make targets and scripts; untested on a physical
# device (like the Android port itself), and not run end to end by its author.
#
# usage: recomp/tools/android_beta.sh --dump-root <dir> --firmware <nca dir> --keys <prod.keys> [options]
#   --dump-root <dir>   folder with smm2-romfs, smm2-v303-exefs, smm2-v303-romfs   (default: ~/smm-work/extracted)
#   --firmware <dir>    folder of your console's registered system .nca files       (for the fonts)
#   --keys <file>       your prod.keys (read in place, never copied)
#   --sdk <dir>         Android SDK                                                  (default: $ANDROID_HOME or ~/Library/Android/sdk)
#   --deps <dir>        where SDL3 / shaderc / VMA for Android are built             (default: recomp/build/deps-android-arm64)
#   --serial <id>       adb device to use (when more than one is connected)
#   --internal          use the app's internal folder (debuggable APK): for "Permission denied" on Android 11+
#   --only <step>       run one step: check | sdk | deps | venv | sysdata | setup | gen | apk | install | push
#   --check             only check the requirements and the paths, build nothing
set -euo pipefail

HERE=$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)    # .../recomp
cd "$HERE"

DUMP_ROOT=${DUMP_ROOT:-$HOME/smm-work/extracted}
FIRMWARE=""; KEYS=""
SDK=${ANDROID_HOME:-${ANDROID_SDK_ROOT:-$HOME/Library/Android/sdk}}
DEPS=$HERE/build/deps-android-arm64
INTERNAL=0; ONLY=""; CHECK=0

while [ $# -gt 0 ]; do
    case $1 in
        --dump-root) DUMP_ROOT=$2; shift 2 ;;
        --firmware)  FIRMWARE=$2; shift 2 ;;
        --keys)      KEYS=$2; shift 2 ;;
        --sdk)       SDK=$2; shift 2 ;;
        --deps)      DEPS=$2; shift 2 ;;
        --serial)    export ADB_SERIAL=$2; shift 2 ;;
        --internal)  INTERNAL=1; shift ;;
        --only)      ONLY=$2; shift 2 ;;
        --check)     CHECK=1; shift ;;
        -h|--help)   sed -n '2,23p' "${BASH_SOURCE[0]}" | sed 's/^# \{0,1\}//'; exit 0 ;;
        *) echo "android_beta: unknown option $1 (see --help)" >&2; exit 2 ;;
    esac
done

say()  { printf '\n==> %s\n' "$*"; }
fail() { echo "android_beta: $*" >&2; exit 1; }
want() { [ -z "$ONLY" ] || [ "$ONLY" = "$1" ]; }
export ANDROID_HOME=$SDK

JAVA_DEFAULT=/opt/homebrew/opt/openjdk@21/libexec/openjdk.jdk/Contents/Home
[ -n "${JAVA_HOME:-}" ] || { [ -d "$JAVA_DEFAULT" ] && export JAVA_HOME=$JAVA_DEFAULT; }

MK=(make -C "$HERE" TARGET=android-arm64 GAME=smm2 "ANDROID_HOME=$SDK" "DEPS=$DEPS" "DUMP_ROOT=$DUMP_ROOT")

# ---- requirements -------------------------------------------------------------------------------------------
check() {
    say "checking requirements"
    local bad=0
    [ "$(uname -s)" = Darwin ] || echo "note: the Android build is only tested on macOS (Apple silicon)"
    for t in python3 cmake ninja make curl git java; do
        command -v "$t" >/dev/null 2>&1 || { echo "missing: $t"; bad=1; }
    done
    python3 -c 'import sys; sys.exit(0 if sys.version_info >= (3,12) else 1)' 2>/dev/null || { echo "Python 3.12 or newer is needed"; bad=1; }
    [ -n "${JAVA_HOME:-}" ] || { echo "JAVA_HOME is not set (brew install openjdk@21)"; bad=1; }
    for d in smm2-romfs smm2-v303-exefs smm2-v303-romfs; do
        [ -d "$DUMP_ROOT/$d" ] || { echo "missing in dump root: $DUMP_ROOT/$d"; bad=1; }
    done
    [ -z "$KEYS" ] || [ -f "$KEYS" ] || { echo "keys file not found: $KEYS"; bad=1; }
    [ -z "$FIRMWARE" ] || [ -d "$FIRMWARE" ] || { echo "firmware folder not found: $FIRMWARE"; bad=1; }
    [ -d "$SDK" ] || echo "note: Android SDK not found at $SDK (the 'sdk' step installs the parts if sdkmanager is on PATH)"
    [ $bad = 0 ] || fail "fix the above and run again"
    echo "ok"
}

# ---- steps --------------------------------------------------------------------------------------------------
step_sdk() {
    command -v sdkmanager >/dev/null 2>&1 || fail "sdkmanager not on PATH (install the Android command-line tools)"
    say "Android SDK parts"
    yes | sdkmanager --sdk_root="$SDK" "platform-tools" "build-tools;36.1.0" "platforms;android-36" "ndk;29.0.14206865"
}
step_deps() {
    [ -d "$SDK" ] || fail "Android SDK not found at $SDK (run --only sdk, or pass --sdk)"
    if [ -f "$DEPS/lib/libSDL3.so" ] && [ -f "$DEPS/lib/libshaderc_combined.a" ]; then
        say "dependencies already built in $DEPS"
    else
        say "building SDL3, shaderc and VMA for Android (needs internet; some minutes)"
        tools/android_deps.sh "$DEPS"
    fi
}
step_venv()    { say "recompiler environment"; make -C "$HERE" venv; }
step_sysdata() {
    [ -n "$FIRMWARE" ] && [ -n "$KEYS" ] || fail "--firmware and --keys are needed for the fonts (the game stops at the title screen without them)"
    say "fonts and system data (keys are read in place, never copied)"
    tools/venv/bin/python games/smm2/sysdata.py --firmware "$FIRMWARE" --keys "$KEYS" --data private/smm2
}
step_setup()   { say "checking your dump (v3.0.3) and laying it out"; make -C "$HERE" GAME=smm2 "DUMP_ROOT=$DUMP_ROOT" setup; }
step_gen()     { say "recompiling the game code for Android (about a minute)"; "${MK[@]}" gen; }
step_apk() {
    say "building the APK (10-20 minutes the first time)"
    if [ $INTERNAL = 1 ]; then RECOMP_ANDROID_DEBUGGABLE=1 "${MK[@]}" -j apk; else "${MK[@]}" -j apk; fi
}
step_install() {
    command -v adb >/dev/null 2>&1 || [ -x "$SDK/platform-tools/adb" ] || fail "adb not found"
    say "installing on the connected device"
    if [ $INTERNAL = 1 ]; then RECOMP_ANDROID_DEBUGGABLE=1 "${MK[@]}" install; else "${MK[@]}" install; fi
}
step_push() {
    say "copying your game files to the device (about 3 GB)"
    if [ $INTERNAL = 1 ]; then "${MK[@]}" PUSH_INTERNAL=1 push-data; else "${MK[@]}" push-data; fi
}

check
[ $CHECK = 0 ] || exit 0

want sdk      && [ -n "$ONLY" ] && step_sdk
want deps     && step_deps
want venv     && step_venv
want sysdata  && step_sysdata
want setup    && step_setup
want gen      && step_gen
want apk      && step_apk
want install  && step_install
want push     && step_push

say "done. Start the app on the device. Log: smm2recomp.log in the app's folder, or: adb logcat -s smm2recomp"
#!/bin/bash
# android_beta.sh (BETA): builds SMM2 Recomp for Android from your own dump, installs it and copies your game
# files to the device, in one go. Put it in recomp/tools/ and run it from the repository root.
#
# Added to SMM2 Recomp 0.4.6 on 2026-10-23; GPL-3.0-or-later like the rest of the project (this notice marks the
# file as added to the original). It only calls the project's own make targets and scripts; untested on a physical
# device (like the Android port itself), and not run end to end by its author.
#
# usage: recomp/tools/android_beta.sh --dump-root <dir> --firmware <nca dir> --keys <prod.keys> [options]
#   --dump-root <dir>   folder with smm2-romfs, smm2-v303-exefs, smm2-v303-romfs   (default: ~/smm-work/extracted)
#   --firmware <dir>    folder of your console's registered system .nca files       (for the fonts)
#   --keys <file>       your prod.keys (read in place, never copied)
#   --sdk <dir>         Android SDK                                                  (default: $ANDROID_HOME or ~/Library/Android/sdk)
#   --deps <dir>        where SDL3 / shaderc / VMA for Android are built             (default: recomp/build/deps-android-arm64)
#   --serial <id>       adb device to use (when more than one is connected)
#   --internal          use the app's internal folder (debuggable APK): for "Permission denied" on Android 11+
#   --only <step>       run one step: check | sdk | deps | venv | sysdata | setup | gen | apk | install | push
#   --check             only check the requirements and the paths, build nothing
set -euo pipefail

HERE=$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)    # .../recomp
cd "$HERE"

DUMP_ROOT=${DUMP_ROOT:-$HOME/smm-work/extracted}
FIRMWARE=""; KEYS=""
SDK=${ANDROID_HOME:-${ANDROID_SDK_ROOT:-$HOME/Library/Android/sdk}}
DEPS=$HERE/build/deps-android-arm64
INTERNAL=0; ONLY=""; CHECK=0

while [ $# -gt 0 ]; do
    case $1 in
        --dump-root) DUMP_ROOT=$2; shift 2 ;;
        --firmware)  FIRMWARE=$2; shift 2 ;;
        --keys)      KEYS=$2; shift 2 ;;
        --sdk)       SDK=$2; shift 2 ;;
        --deps)      DEPS=$2; shift 2 ;;
        --serial)    export ADB_SERIAL=$2; shift 2 ;;
        --internal)  INTERNAL=1; shift ;;
        --only)      ONLY=$2; shift 2 ;;
        --check)     CHECK=1; shift ;;
        -h|--help)   sed -n '2,23p' "${BASH_SOURCE[0]}" | sed 's/^# \{0,1\}//'; exit 0 ;;
        *) echo "android_beta: unknown option $1 (see --help)" >&2; exit 2 ;;
    esac
done

say()  { printf '\n==> %s\n' "$*"; }
fail() { echo "android_beta: $*" >&2; exit 1; }
want() { [ -z "$ONLY" ] || [ "$ONLY" = "$1" ]; }
export ANDROID_HOME=$SDK

JAVA_DEFAULT=/opt/homebrew/opt/openjdk@21/libexec/openjdk.jdk/Contents/Home
[ -n "${JAVA_HOME:-}" ] || { [ -d "$JAVA_DEFAULT" ] && export JAVA_HOME=$JAVA_DEFAULT; }

MK=(make -C "$HERE" TARGET=android-arm64 GAME=smm2 "ANDROID_HOME=$SDK" "DEPS=$DEPS" "DUMP_ROOT=$DUMP_ROOT")

# ---- requirements -------------------------------------------------------------------------------------------
check() {
    say "checking requirements"
    local bad=0
    [ "$(uname -s)" = Darwin ] || echo "note: the Android build is only tested on macOS (Apple silicon)"
    for t in python3 cmake ninja make curl git java; do
        command -v "$t" >/dev/null 2>&1 || { echo "missing: $t"; bad=1; }
    done
    python3 -c 'import sys; sys.exit(0 if sys.version_info >= (3,12) else 1)' 2>/dev/null || { echo "Python 3.12 or newer is needed"; bad=1; }
    [ -n "${JAVA_HOME:-}" ] || { echo "JAVA_HOME is not set (brew install openjdk@21)"; bad=1; }
    for d in smm2-romfs smm2-v303-exefs smm2-v303-romfs; do
        [ -d "$DUMP_ROOT/$d" ] || { echo "missing in dump root: $DUMP_ROOT/$d"; bad=1; }
    done
    [ -z "$KEYS" ] || [ -f "$KEYS" ] || { echo "keys file not found: $KEYS"; bad=1; }
    [ -z "$FIRMWARE" ] || [ -d "$FIRMWARE" ] || { echo "firmware folder not found: $FIRMWARE"; bad=1; }
    [ -d "$SDK" ] || echo "note: Android SDK not found at $SDK (the 'sdk' step installs the parts if sdkmanager is on PATH)"
    [ $bad = 0 ] || fail "fix the above and run again"
    echo "ok"
}

# ---- steps --------------------------------------------------------------------------------------------------
step_sdk() {
    command -v sdkmanager >/dev/null 2>&1 || fail "sdkmanager not on PATH (install the Android command-line tools)"
    say "Android SDK parts"
    yes | sdkmanager --sdk_root="$SDK" "platform-tools" "build-tools;36.1.0" "platforms;android-36" "ndk;29.0.14206865"
}
step_deps() {
    [ -d "$SDK" ] || fail "Android SDK not found at $SDK (run --only sdk, or pass --sdk)"
    if [ -f "$DEPS/lib/libSDL3.so" ] && [ -f "$DEPS/lib/libshaderc_combined.a" ]; then
        say "dependencies already built in $DEPS"
    else
        say "building SDL3, shaderc and VMA for Android (needs internet; some minutes)"
        tools/android_deps.sh "$DEPS"
    fi
}
step_venv()    { say "recompiler environment"; make -C "$HERE" venv; }
step_sysdata() {
    [ -n "$FIRMWARE" ] && [ -n "$KEYS" ] || fail "--firmware and --keys are needed for the fonts (the game stops at the title screen without them)"
    say "fonts and system data (keys are read in place, never copied)"
    tools/venv/bin/python games/smm2/sysdata.py --firmware "$FIRMWARE" --keys "$KEYS" --data private/smm2
}
step_setup()   { say "checking your dump (v3.0.3) and laying it out"; make -C "$HERE" GAME=smm2 "DUMP_ROOT=$DUMP_ROOT" setup; }
step_gen()     { say "recompiling the game code for Android (about a minute)"; "${MK[@]}" gen; }
step_apk() {
    say "building the APK (10-20 minutes the first time)"
    if [ $INTERNAL = 1 ]; then RECOMP_ANDROID_DEBUGGABLE=1 "${MK[@]}" -j apk; else "${MK[@]}" -j apk; fi
}
step_install() {
    command -v adb >/dev/null 2>&1 || [ -x "$SDK/platform-tools/adb" ] || fail "adb not found"
    say "installing on the connected device"
    if [ $INTERNAL = 1 ]; then RECOMP_ANDROID_DEBUGGABLE=1 "${MK[@]}" install; else "${MK[@]}" install; fi
}
step_push() {
    say "copying your game files to the device (about 3 GB)"
    if [ $INTERNAL = 1 ]; then "${MK[@]}" PUSH_INTERNAL=1 push-data; else "${MK[@]}" push-data; fi
}

check
[ $CHECK = 0 ] || exit 0

want sdk      && [ -n "$ONLY" ] && step_sdk
want deps     && step_deps
want venv     && step_venv
want sysdata  && step_sysdata
want setup    && step_setup
want gen      && step_gen
want apk      && step_apk
want install  && step_install
want push     && step_push

say "done. Start the app on the device. Log: smm2recomp.log in the app's folder, or: adb logcat -s smm2recomp"

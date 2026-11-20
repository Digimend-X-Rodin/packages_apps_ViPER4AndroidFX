## ViPER4Android FX (AIDL)

Prebuilt ViPER4AndroidFX **v2.1.2** for devices using the **AIDL audio effect HAL**
(Android 15+, API 35+).

- App: `com.llsl.viper4android` (ViPER4AndroidFX 2.1.2), installed as a privileged app
  at `/system/priv-app/ViPER4AndroidFX/`
- Effect library: `libv4a_aidl.so` (ViPERFX_RE AIDL build, v2.1.2)
- Control service: `viper.control` (`aidl.viper.fx.IViperControl`)

> This replaces the HIDL-era `libv4a_re.so`. The old library used the legacy
> `effect_param_s`/`effect_config_s` C ABI and is not compatible with the AIDL
> effect HAL. Remove any old `libv4a_re` entries from your tree.

Integrate by completing these steps:

1.  **Build system:** inherit the package in **`device.mk`**:

    ```makefile
    $(call inherit-product, packages/apps/ViPER4AndroidFX/config.mk)
    ```

    This installs the app (`ViPER4AndroidFX`, privileged) and the effect library
    (`libv4a_aidl`, to `/vendor/lib[64]/soundfx/`).

2.  **Privileged permissions:** the app needs `MODIFY_AUDIO_ROUTING`, which is
    `signature|privileged|role`. Priv-app placement satisfies the privileged
    half, and the permission is granted through the allowlist shipped by this
    package (`permissions/privapp-permissions-viper4android.xml`, installed to
    `/system/etc/permissions/`). The app `required:`s the allowlist, so no
    separate device-tree step is needed.

3.  **Audio effects:** add the library, effect, and device routing to
    **`audio_effects_config.xml`** (`/vendor/etc/`):

    ```xml
    <!-- <libraries> -->
    <library name="v4a_aidl" path="libv4a_aidl.so"/>

    <!-- <effects> -->
    <effect name="v4a_standard_aidl" library="v4a_aidl"
            uuid="90380da3-8536-4744-a6a3-5731970e640f"
            type="7261676f-6d75-7369-6364-28e2fd3ac39e"/>

    <!-- <deviceEffects> (after <preprocess>) -->
    <deviceEffects>
        <devicePort type="AUDIO_DEVICE_OUT_REMOTE_SUBMIX" address="">
            <apply effect="v4a_standard_aidl"/>
        </devicePort>
        <devicePort type="AUDIO_DEVICE_OUT_USB_DEVICE" address="">
            <apply effect="v4a_standard_aidl"/>
        </devicePort>
    </deviceEffects>
    ```

4.  **SELinux:** the app is a privileged app and runs in the `priv_app` domain. It
    talks to the audio HAL over binder, and the HAL registers `viper.control`:

    ```te
    # sepolicy/vendor/service.te
    type viper_control_service, service_manager_type;
    ```

    ```text
    # sepolicy/vendor/service_contexts
    viper.control    u:object_r:viper_control_service:s0
    ```

    ```te
    # sepolicy/vendor/hal_audio_default.te  (audio HAL registers the service)
    allow hal_audio_default viper_control_service:service_manager add;
    allow hal_audio_default servicemanager:binder { call transfer };

    # sepolicy/vendor/viper4android.te  (priv-app control client)
    allow priv_app servicemanager:service_manager list;
    allow priv_app viper_control_service:service_manager find;
    allow priv_app hal_audio_default:binder { call transfer };
    ```

    `default_android_service` is not granted. Neverallow forbids every domain from
    using that generic type, and `viper.control` has its own type.

    The legacy `audioserver` / `execmem` rules for `libv4a_re` are not required by
    the AIDL build. Add them back only if the effect library fails to load on your
    device.

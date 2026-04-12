---
layout: post
title: Refactoring the heart of PowerToys from C++ to C#
category: general
---

In the last few months I have been working on refactoring the heart of PowerToys from C++ to C#.  This heart is the PowerToys runner.

## What is the PowerToys runner?

The PowerToys runner is the main part of PowerToys. It is the `powertoys.exe` executable which is responsible for the following things:

* Showing the tray icon of PowerToys
* Starting the different PowerToys modules based on the user settings and handling their processes
* Piping different commands from the settings to the different modules

<figure>
    <img src="/images/runner-tray.png" alt="The PowerToys tray icon" />
    <figcaption>The PowerToys tray icon</figcaption>
</figure>
Currently most of these functions are handled by the different module interfaces. Each PowerToys module has a module interface which is just one C++ project that exports a DLL

An example of a PowerToys module interface (shortened):
```cpp
[...]

namespace NonLocalizable
{
    const wchar_t ModulePath[] = L"PowerToys.CropAndLock.exe";
}

namespace
{
    const wchar_t JSON_KEY_PROPERTIES[] = L"properties";
    const wchar_t JSON_KEY_WIN[] = L"win";
    const wchar_t JSON_KEY_ALT[] = L"alt";
    const wchar_t JSON_KEY_CTRL[] = L"ctrl";
    const wchar_t JSON_KEY_SHIFT[] = L"shift";
    const wchar_t JSON_KEY_CODE[] = L"code";
    const wchar_t JSON_KEY_REPARENT_HOTKEY[] = L"reparent-hotkey";
    const wchar_t JSON_KEY_THUMBNAIL_HOTKEY[] = L"thumbnail-hotkey";
    const wchar_t JSON_KEY_SCREENSHOT_HOTKEY[] = L"screenshot-hotkey";
    const wchar_t JSON_KEY_VALUE[] = L"value";
}

BOOL APIENTRY DllMain( HMODULE /*hModule*/,
                       DWORD  ul_reason_for_call,
                       LPVOID /*lpReserved*/
                     )
{
    switch (ul_reason_for_call)
    {
    case DLL_PROCESS_ATTACH:
        Trace::CropAndLock::RegisterProvider();
        break;
    case DLL_THREAD_ATTACH:
    case DLL_THREAD_DETACH:
        break;
    case DLL_PROCESS_DETACH:
        Trace::CropAndLock::UnregisterProvider();
        break;
    }
    return TRUE;

}

class CropAndLockModuleInterface : public PowertoyModuleIface
{
public:
    // Return the localized display name of the powertoy
    virtual PCWSTR get_name() override
    {
        return app_name.c_str();
    }

    // Return the non localized key of the powertoy, this will be cached by the runner
    virtual const wchar_t* get_key() override
    {
        return app_key.c_str();
    }

    // Return the configured status for the gpo policy for the module
    virtual powertoys_gpo::gpo_rule_configured_t gpo_policy_enabled_configuration() override
    {
        return powertoys_gpo::getConfiguredCropAndLockEnabledValue();
    }

    // Return JSON with the configuration options.
    // These are the settings shown on the settings page along with their current values.
    virtual bool get_config(wchar_t* buffer, int* buffer_size) override
    {
        HINSTANCE hinstance = reinterpret_cast<HINSTANCE>(&__ImageBase);

        // Create a Settings object.
        PowerToysSettings::Settings settings(hinstance, get_name());

        return settings.serialize_to_buffer(buffer, buffer_size);
    }

    // Passes JSON with the configuration settings for the powertoy.
    // This is called when the user hits Save on the settings page.
    virtual void set_config(const wchar_t* config) override
    {
        try
        {
            // Parse the input JSON string.
            PowerToysSettings::PowerToyValues values =
                PowerToysSettings::PowerToyValues::from_json_string(config, get_key());

            parse_hotkey(values);

            values.save_to_settings_file();
        }
        catch (std::exception&)
        {
            // Improper JSON.
        }
    }

    virtual bool on_hotkey(size_t hotkeyId) override
    {
        if (m_enabled)
        {
            // [...]
        }

        return false;
    }

    virtual size_t get_hotkeys(Hotkey* hotkeys, size_t buffer_size) override
    {
        if (hotkeys && buffer_size >= 3)
        {
            hotkeys[0] = m_reparent_hotkey;
            hotkeys[1] = m_thumbnail_hotkey;
            hotkeys[2] = m_screenshot_hotkey;
        }
        return 3;
    }

    // Enable the powertoy
    virtual void enable()
    {
        Logger::info("CropAndLock enabling");
        Enable();
    }

    // Disable the powertoy
    virtual void disable()
    {
        Logger::info("CropAndLock disabling");
        Disable(true);
    }

    // Returns if the powertoy is enabled
    virtual bool is_enabled() override
    {
        return m_enabled;
    }

    // Destroy the powertoy and free memory
    virtual void destroy() override
    {
        Disable(false);
        delete this;
    }

    virtual void send_settings_telemetry() override
    {
        Logger::info("Send settings telemetry");
        Trace::CropAndLock::SettingsTelemetry(m_reparent_hotkey, m_thumbnail_hotkey, m_screenshot_hotkey);
    }

    CropAndLockModuleInterface()
    {
        app_name = L"CropAndLock";
        app_key = NonLocalizable::ModuleKey;
        LoggerHelpers::init_logger(app_key, L"ModuleInterface", LogSettings::cropAndLockLoggerName);

        m_reparent_event_handle = CreateDefaultEvent(CommonSharedConstants::CROP_AND_LOCK_REPARENT_EVENT);
        m_thumbnail_event_handle = CreateDefaultEvent(CommonSharedConstants::CROP_AND_LOCK_THUMBNAIL_EVENT);
        m_screenshot_event_handle = CreateDefaultEvent(CommonSharedConstants::CROP_AND_LOCK_SCREENSHOT_EVENT);
        m_exit_event_handle = CreateDefaultEvent(CommonSharedConstants::CROP_AND_LOCK_EXIT_EVENT);

        init_settings();
    }

private:
    void Enable()
    {
        m_enabled = true;

        // Log telemetry
        Trace::CropAndLock::Enable(true);

        // Pass the PID.
        unsigned long powertoys_pid = GetCurrentProcessId();
        std::wstring executable_args = L"";
        executable_args.append(std::to_wstring(powertoys_pid));
        
        ResetEvent(m_reparent_event_handle);
        ResetEvent(m_thumbnail_event_handle);
        ResetEvent(m_screenshot_event_handle);
        ResetEvent(m_exit_event_handle);

        SHELLEXECUTEINFOW sei{ sizeof(sei) };
        sei.fMask = { SEE_MASK_NOCLOSEPROCESS | SEE_MASK_FLAG_NO_UI };
        sei.lpFile = NonLocalizable::ModulePath;
        sei.nShow = SW_SHOWNORMAL;
        sei.lpParameters = executable_args.data();
        if (ShellExecuteExW(&sei) == false)
        {
            Logger::error(L"Failed to start CropAndLock");
            auto message = get_last_error_message(GetLastError());
            if (message.has_value())
            {
                Logger::error(message.value());
            }
        }
        else
        {
            m_hProcess = sei.hProcess;
        }

    }

    void Disable(bool const traceEvent)
    {
        m_enabled = false;

        // We can't just kill the process, since Crop and Lock might need to release any reparented windows first.
        SetEvent(m_exit_event_handle);

        ResetEvent(m_reparent_event_handle);
        ResetEvent(m_thumbnail_event_handle);
        ResetEvent(m_screenshot_event_handle);

        // Log telemetry
        if (traceEvent)
        {
            Trace::CropAndLock::Enable(false);
        }

        if (m_hProcess)
        {
            m_hProcess = nullptr;
        }

    }

    void parse_hotkey(PowerToysSettings::PowerToyValues& settings)
    {
        // [...]
    }

    bool is_process_running()
    {
        return WaitForSingleObject(m_hProcess, 0) == WAIT_TIMEOUT;
    }

    void init_settings()
    {
        try
        {
            // Load and parse the settings file for this PowerToy.
            PowerToysSettings::PowerToyValues settings =
                PowerToysSettings::PowerToyValues::load_from_settings_file(get_key());

            parse_hotkey(settings);
        }
        catch (std::exception&)
        {
            Logger::warn(L"An exception occurred while loading the settings file");
            // Error while loading from the settings file. Let default values stay as they are.
        }
    }

    std::wstring app_name;
    std::wstring app_key; //contains the non localized key of the powertoy

    bool m_enabled = false;
    HANDLE m_hProcess = nullptr;

    // TODO: actual default hotkey setting in line with other PowerToys.
    Hotkey m_reparent_hotkey = { .win = true, .ctrl = true, .shift = true, .alt = false, .key = 'R' };
    Hotkey m_thumbnail_hotkey = { .win = true, .ctrl = true, .shift = true, .alt = false, .key = 'T' };
    Hotkey m_screenshot_hotkey = { .win = true, .ctrl = true, .shift = true, .alt = false, .key = 'S' };

    HANDLE m_reparent_event_handle;
    HANDLE m_thumbnail_event_handle;
    HANDLE m_screenshot_event_handle;
    HANDLE m_exit_event_handle;

};

extern "C" __declspec(dllexport) PowertoyModuleIface* __cdecl powertoy_create()
{
    return new CropAndLockModuleInterface();
}
```

Here not visible are the different other files required to build this module interface into a DLL.

## Why refactor the runner?

The current code base is very messy and hard to maintain. PowerToys is an incubation project, which means it constantly has to adapt to new requirements and changes. The current code base is not flexible enough to handle these changes and it is also very hard to add new features to it.

Furthermore many parts of the code are simply over-engineered or even unused.

One last reason is that PowerToys is a open source project and it is very hard for new contributors to understand the code base and to contribute to it. Refactoring the code base to C# will make it more accessible for new contributors and will also make it easier to maintain in the long run.

## How does the new module interface look

In the new system each module will only consist of one class which implements the `IPowerToyModule` interface. This means a big reduction in projects that have to be built, as all the files are now in one project. This also means that the code compiles much faster and it is easier to debug.

Example of the same module interface as above but in C# (shortened):
```csharp
// [...]
namespace RunnerV2.ModuleInterfaces
{
    internal sealed partial class CropAndLockModuleInterface : ProcessModuleAbstractClass, IPowerToysModule, IDisposable, IPowerToysModuleShortcutsProvider, IPowerToysModuleSettingsChangedSubscriber
    {
        public string Name => "CropAndLock";

        public bool Enabled => SettingsUtils.Default.GetSettings<GeneralSettings>().Enabled.CropAndLock;

        public GpoRuleConfigured GpoRuleConfigured => GPOWrapper.GetConfiguredCropAndLockEnabledValue();

        private EventWaitHandle _reparentEvent = new(false, EventResetMode.AutoReset, Constants.CropAndLockReparentEvent());
        private EventWaitHandle _thumbnailEvent = new(false, EventResetMode.AutoReset, Constants.CropAndLockThumbnailEvent());
        private EventWaitHandle _terminateEvent = new(false, EventResetMode.AutoReset, Constants.CropAndLockExitEvent());

        public void Disable()
        {
            _terminateEvent.Set();
        }

        public void Enable()
        {
            PopulateShortcuts();
        }

        public void PopulateShortcuts()
        {
            Shortcuts.Clear();
            var settings = SettingsUtils.Default.GetSettings<CropAndLockSettings>(Name);
            Shortcuts.Add((settings.Properties.ThumbnailHotkey.Value,  () => _thumbnailEvent.Set()));
            Shortcuts.Add((settings.Properties.ReparentHotkey.Value,  () => _reparentEvent.Set()));
        }

        public void OnSettingsChanged()
        {
            PopulateShortcuts();
        }

        public List<(HotkeySettings Hotkey, Action Action)> Shortcuts { get; } = [];

        public override string ProcessPath => "PowerToys.CropAndLock.exe";

        public override string ProcessName => "PowerToys.CropAndLock";

        public override ProcessLaunchOptions LaunchOptions => ProcessLaunchOptions.SingletonProcess | ProcessLaunchOptions.RunnerProcessIdAsFirstArgument;

        public void Dispose()
        {
            GC.SuppressFinalize(this);
            _reparentEvent.Dispose();
            _thumbnailEvent.Dispose();
            _terminateEvent.Dispose();
        }
    }
}
```

As you can see the code is much more concise and shorter. The new module interface also uses the new settings system which means that we can get rid of a lot of code related to handling the settings file.

## Where am I at with the refactor?

The PR is mostly done, I am currently waiting for the .NET 10 upgrade to be done so I can update the runner to .NET 10 and then I will be able to finish the PR.
I furthermore also upgraded some other stuff in the process like all File Explorer Add-Ons now use one project to export the DLL instead of each add-on having its own project.

Also rewritten from C++ to C# are the PowerToys action runner (used to elevate certain processes) and the PowerToys updater.

One last thing that has to be done is adding telemetry back.

You can check out the Pull Request for the refactor [here](https://github.com/microsoft/PowerToys/pull/43714).
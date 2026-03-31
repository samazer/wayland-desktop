# kwin_introspect.py — Output

**Purpose:** Walk the entire `org.kde.KWin` D-Bus object tree and print every
non-standard interface and method.  Run to discover what KWin actually exposes
on this Plasma version.

**Key finding:** `getWindowList()` is absent from this KWin version.  The only
window-related methods at `/KWin` are `queryWindowInfo()` (interactive) and
`getWindowInfo(uuid)` (requires a UUID you already have).  See ADR-260317-04.

---

```
================================================================
KWin D-Bus object tree
================================================================

  PATH : /ColorPicker
  IFACE: org.kde.kwin.ColorPicker
    method  pick() -> (u)

  PATH : /Compositor
  IFACE: org.kde.kwin.Compositing
    signal  compositingToggled(active)
    prop    active  [b]  (read)
    prop    compositingPossible  [b]  (read)
    prop    compositingNotPossibleReason  [s]  (read)
    prop    openGLIsBroken  [b]  (read)
    prop    compositingType  [s]  (read)
    prop    supportedOpenGLPlatformInterfaces  [as]  (read)
    prop    platformRequiresCompositing  [b]  (read)
  [skip] /Effects: duplicate parameter name: 'name'

  PATH : /FTrace
  IFACE: org.kde.kwin.FTrace
    method  setEnabled(enabled)
    prop    isEnabled  [b]  (read)

  PATH : /KWin
  IFACE: org.kde.KWin
    method  reconfigure()
    method  killWindow()
    method  setCurrentDesktop(desktop) -> b
    method  currentDesktop() -> i
    method  nextDesktop()
    method  previousDesktop()
    method  supportInformation() -> s
    method  activeOutputName() -> s
    method  showDebugConsole()
    method  replace()
    method  queryWindowInfo() -> a{sv}
    method  getWindowInfo(s) -> a{sv}
    method  showDesktop(showing)
    signal  reloadConfig()
    signal  showingDesktopChanged(showing)
    prop    showingDesktop  [b]  (read)

  PATH : /Layouts
  IFACE: org.kde.KeyboardLayouts
    method  switchToNextLayout()
    method  switchToPreviousLayout()
    method  setLayout(index) -> b
    method  getLayout() -> u
    method  getLayoutsList() -> a(sss)
    signal  layoutChanged(index)
    signal  layoutListChanged()

  PATH : /Plugins
  IFACE: org.kde.KWin.Plugins
    method  LoadPlugin(name) -> b
    method  UnloadPlugin(name)
    prop    LoadedPlugins  [as]  (read)
    prop    AvailablePlugins  [as]  (read)

  PATH : /ScreenSaver
  IFACE: org.freedesktop.ScreenSaver
    method  Lock()
    method  SimulateUserActivity()
    method  GetActive() -> b
    method  GetActiveTime() -> seconds
    method  GetSessionIdleTime() -> seconds
    method  SetActive(e) -> b
    method  Inhibit(application_name, reason_for_inhibit) -> cookie
    method  UnInhibit(cookie)
    method  Throttle(application_name, reason_for_inhibit) -> cookie
    method  UnThrottle(cookie)
    signal  ActiveChanged(b)

  PATH : /ScreenSaver
  IFACE: org.kde.screensaver
    method  configure()
    signal  AboutToLock()

  PATH : /Scripting
  IFACE: org.kde.kwin.Scripting
    method  start()
    method  loadScript(filePath, pluginName) -> i
    method  loadScript(filePath) -> i
    method  loadDeclarativeScript(filePath, pluginName) -> i
    method  loadDeclarativeScript(filePath) -> i
    method  isScriptLoaded(pluginName) -> b
    method  unloadScript(pluginName) -> b

  PATH : /Session
  IFACE: org.kde.KWin.Session
    method  setState(state)
    method  loadSession(name)
    method  aboutToSaveSession(name)
    method  finishSaveSession(name)
    method  quit()
    method  closeWaylandWindows() -> b

  PATH : /VirtualDesktopManager
  IFACE: org.kde.KWin.VirtualDesktopManager
    method  createDesktop(position, name)
    method  setDesktopName(id, name)
    method  removeDesktop(id)
    signal  countChanged(count)
    signal  rowsChanged(rows)
    signal  currentChanged(id)
    signal  navigationWrappingAroundChanged(navigationWrappingAround)
    signal  desktopDataChanged(id, desktopData)
    signal  desktopCreated(id, desktopData)
    signal  desktopRemoved(id)
    prop    count  [u]  (read)
    prop    current  [s]  (readwrite)
    prop    rows  [u]  (readwrite)
    prop    navigationWrappingAround  [b]  (readwrite)
    prop    desktops  [a(iss)]  (read)

  PATH : /VirtualKeyboard
  IFACE: org.kde.kwin.VirtualKeyboard
    method  willShowOnActive() -> b
    method  forceActivate()
    signal  enabledChanged()
    signal  activeChanged()
    signal  visibleChanged()
    signal  availableChanged()
    signal  activeClientSupportsTextInputChanged()
    prop    available  [b]  (read)
    prop    enabled  [b]  (readwrite)
    prop    active  [b]  (readwrite)
    prop    visible  [b]  (read)
    prop    activeClientSupportsTextInput  [b]  (read)

  PATH : /WindowsRunner
  IFACE: org.kde.krunner1
    method  Actions() -> matches
    method  Run(matchId, actionId)
    method  Match(query) -> matches

    [... /component/* paths — keyboard shortcuts, not relevant ...]

    PATH : /org/kde/KWin/NightLight
    IFACE: org.kde.KWin.NightLight
      method  inhibit() -> cookie
      method  uninhibit(cookie)
      method  preview(temperature)
      method  stopPreview()
      prop    inhibited  [b]  (read)
      prop    enabled  [b]  (read)
      prop    running  [b]  (read)
      prop    available  [b]  (read)
      prop    currentTemperature  [u]  (read)
      prop    targetTemperature  [u]  (read)
      prop    mode  [u]  (read)
      prop    daylight  [b]  (read)

    PATH : /org/kde/KWin/ScreenShot2
    IFACE: org.kde.KWin.ScreenShot2
      method  CaptureWindow(handle, options, pipe) -> results
      method  CaptureActiveWindow(options, pipe) -> results
      method  CaptureArea(x, y, width, height, options, pipe) -> results
      method  CaptureScreen(name, options, pipe) -> results
      method  CaptureActiveScreen(options, pipe) -> results
      method  CaptureInteractive(kind, options, pipe) -> results
      method  CaptureWorkspace(options, pipe) -> results
      prop    Version  [u]  (read)

    [... /org/kde/KWin/InputDevice/* — input device properties, not relevant ...]

================================================================
Done.
```

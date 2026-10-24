# UGUI Data Binding

Unity package (`com.madcat-vr.ugui-data-binding`) for binding uGUI controls to view-model data. Bindings sit on UI GameObjects, resolve member paths against a nearby `DataContext`, and can push data into the view, pull UI changes back, or invoke commands.

Requires **Unity 2021.3** or newer and **Unity UI** (`com.unity.ugui`). Namespace: `MadCat.UguiDataBinding`.

## How it works

1. Put a **data context** on a parent (or the same object) that exposes a view model.
2. Add a **binding** on a control and set one or more **data references** (member paths).
3. Optionally add **converters** on the same GameObject to transform values between data and view.

Bindings walk up the hierarchy for the nearest `DataContext` (including a context on the same object, listed before the binding). Nested contexts are supported. Enabling a context refreshes bindings on that object and its descendants.

## Data references

Each binding has a `dataReferences` list. A reference is a path into the current context’s view model.

| Syntax | Meaning |
| --- | --- |
| `PlayerName` | Public instance or static property, field, or method |
| `Stats.Level` | Nested members, split on `.` |
| `.ParentProperty` | Leading `.` moves one data context up, then resolves the rest |
| `-RootProperty` | Leading `-` jumps to the root data context, then resolves the rest |

The data context caches resolved members and:

- Subscribes to `INotifyPropertyChanged` on the source object
- Subscribes to `INotifyCollectionChanged` on collection values
- Polls remaining (non-notifying) bindings each frame until nothing needs polling, then **freezes** (`enabled = false`) until the origin or cache changes

Implement `INotifyPropertyChanged` / `INotifyCollectionChanged` when you want updates without per-frame polling.

Boolean UI bindings (`Toggle`, visibility, interactability) use a truthy conversion: `bool` as-is, non-zero numbers as true, and non-null objects as true.

## Data contexts

| Type | Role |
| --- | --- |
| `ViewModel<T>` | MonoBehaviour that *is* the view model (`T` is the concrete subclass). |
| `ViewModelObjectReference` / `ViewModelObjectReference<T>` | Holds a plain object. Call `SetViewModel` at runtime (used by lists and nested VMs). |
| `ViewModelUnityObjectReference` / `ViewModelUnityObjectReference<T>` | Serialized Unity `Object` as the view model. |
| `ViewModelClassReference<T>` | Origin is `default(T)` (null for classes), so paths resolve against **static** members of `T`. |

`ViewModelBinding` (data → view) writes a nested object into a sibling `ViewModelObjectReference`, which then becomes the context for child bindings.

## Bindings

Most bindings inherit `Binding` and use `Direction`: `DataToView`, `ViewToData`, `TwoWay`, or `Call`.

| Component | Direction | UI / behavior |
| --- | --- | --- |
| `TextBinding` | Configurable (default two-way) | `string` ↔ `Text` / `InputField`. With TextMesh Pro (or uGUI 2.0+), also `TMP_Text` / `TMP_InputField`. Inputs write back on `onValueChanged`. |
| `ValueBinding` | Configurable (default two-way) | Numeric → `Slider`, `Scrollbar`, and `Image.fillAmount`. Sliders and scrollbars write back. |
| `MinValueBinding` / `MaxValueBinding` | Data → view | Numeric → `Slider.minValue` / `maxValue`. |
| `FillBinding` | Data → view | Numeric → `Image.fillAmount`. |
| `ToggleBinding` | Configurable (default two-way) | Truthy value ↔ `Toggle.isOn`. |
| `InteractabilityBinding` | Data → view | Truthy value → `Selectable.interactable`. |
| `VisibilityBinding` | Data → view | Truthy value → `gameObject.SetActive`. Re-updates child bindings when turning on. |
| `ColorBinding` | Data → view | `Color` → `Graphic.color`. |
| `ImageBinding` | Data → view | `Sprite` → `Image.sprite`; `Texture2D` → `RawImage.texture`. |
| `VolumeBinding` | Data → view | Numeric → `AudioSource.volume` (when the Audio module is present). |
| `OnClickBinding` | Call | `Button.onClick` invokes the referenced method. |
| `ListItemsBinding` | Data → view | `IEnumerable` of item view models. Instantiates `template` (`ViewModelObjectReference`) under `parent`, reuses children, destroys extras. Hides the template if it lives under `parent`. |
| `DropdownOptionsBinding` | Data → view | `IEnumerable` → `Dropdown` / `TMP_Dropdown` options. Items may be `string`, `Sprite`, `IDropdownOptionData`, or `ToString()`. |
| `DropdownSelectionBinding` | Two-way | Selected option object ↔ dropdown value. Requires `DropdownOptionsBinding`. |
| `ViewModelBinding` | Data → view | Assigns the bound object to `ViewModelObjectReference`. |
| `SmartStringBinding` | Data → view | Localization Smart Format (requires Unity Localization). |

## Converters

`Converter` components on the **same GameObject** as a binding run in inspector/component order. Only converters that sit between this binding and the next `Binding` on that object are used. Data → view runs them forward; view → data runs them in reverse.

| Converter | Data → view |
| --- | --- |
| `StringFormatter` | `string.Format(format, …)` over all current values; replaces the list with one string. |
| `BoolInverter` | Negates a `bool` (also on view → data). |
| `BoolThreshold` | Turns a number into a bool using `threshold` / `defaultValue`. |
| `ReferenceComparator` | `true` if every extra value equals the first (null-safe). |

## Optional packages

The runtime assembly sets defines when these packages (or versions) are present:

| Package | Define | Effect |
| --- | --- | --- |
| `com.unity.textmeshpro`, or `com.unity.ugui` 2.0.0+ | `MADCAT_UGUI_DATA_BINDING_TEXTMESHPRO` | TMP text, input, and dropdown on the matching bindings |
| `com.unity.localization` | `MADCAT_UGUI_DATA_BINDING_LOCALIZATION` | `SmartStringBinding`, `BoundLocalizationSource` |
| `com.unity.modules.audio` | `MADCAT_UGUI_DATA_BINDING_AUDIO_MODULE` | `VolumeBinding` → `AudioSource` |
| `com.unity.inputsystem` | `MADCAT_UGUI_DATA_BINDING_INPUT_SYSTEM` | `InputSystemSelector` keeps EventSystem input modules consistent with the active input backend |

`SmartStringBinding` uses a `LocalizeStringEvent`. Bound data references can supply a localization **key** (optional second value: table name) or a `LocalizedString`. Placeholders in the smart string are resolved through `BoundLocalizationSource` (`ISource`) via `GetValue(path)` on that binding.

## Editor diagnostics (Play Mode)

- **Inspector:** `DataReference` fields tint green (resolved), yellow (partial), or red (failed).
- **Hierarchy:** color tags on the right of each row for getter, setter, and call resolution. Click a tag to log each path and its `Resolution`.

## Extending

- Subclass `Binding`, set `Direction`, and override `UpdateView(List<object>)` and/or `GetViewValues` (plus `Subscribe` / `Unsubscribe` for UI events). Use `TryGetComponentCached` for sibling UI components.
- Subclass `Converter` and implement `DataToView` / `ViewToData`.
- Subclass `DataContext<T>` and implement `GetViewModel()` for a custom origin.

Public members used through bindings should stay reachable under IL2CPP stripping. Types can implement `IPreserved<T>` (or call `DelegateFactory.Preserve<T>()`) so common getter/setter instantiations are preserved.

## Samples

Import from the Package Manager **Samples** tab:

| Sample | What it shows |
| --- | --- |
| **Basics** | Properties, fields, static members, methods, text, sliders, fill, toggle, visibility |
| **Notification Property** | Same idea with a POCO view model that implements `INotifyPropertyChanged` |
| **Nesting** | Nested view models and addressing parent/root data with `.` / `-` |
| **Item List** | Collection templates, dropdown options, and selection (`HashSet` / active item). TextMesh Pro recommended for dropdowns |
| **Localization** | Smart-string bindings. Requires the Localization package |

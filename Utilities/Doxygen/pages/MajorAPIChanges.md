Major API Changes             {#MajorAPIChanges}
=================

This page documents major API/design changes between different versions since we
started tracking these (starting after version 4.2).

Changes in 5.1
--------------

###Refactored 3DWidget panels###

3DWidget panels were subclasses of `pqObjectPanel`, the Properties panel
hierarchy that has been deprecated since 4.0. This version finally drops support
for these old 3D widget panels. The new implementation simply uses the
`pqPropertyWidget` infrastructure. A 3D widget panel is nothing more than a custom
`pqPropertyWidget` subclass for a property-group (`vtkSMPropertyGroup`) which
creates an interactive widget to be shown in the active view to help the users
change the values for properties in the property-group using this interactive
widget, in addition to standard Qt-based UI elements.

If you have a filter or proxy indicating that the Properties panel use one of
the standard 3D widgets for controlling certain properties on that proxy, you
specified that using `<Hints>` in the Proxy's XML definition.

For example, the **(implicit_functions, Plane)** proxy requested that
`pqImplicitPlaneWidget` be used using the following XML hints.

    <Proxy class="vtkPVPlane" name="Plane">
      <InputProperty is_internal="1" name="Input" />
      <DoubleVectorProperty command="SetOrigin"
                            default_values="0.0 0.0 0.0"
                            name="Origin"
                            number_of_elements="3">
      ...
      </DoubleVectorProperty>
      <DoubleVectorProperty command="SetNormal"
                            default_values="1.0 0.0 0.0"
                            name="Normal"
                            number_of_elements="3">
      ...
      </DoubleVectorProperty>
      <DoubleVectorProperty animateable="1"
                            command="SetOffset"
                            default_values="0.0"
                            name="Offset"
                            number_of_elements="1">
      </DoubleVectorProperty>
      <Hints>
        <PropertyGroup type="Plane">
          <Property function="Origin" name="Origin" />
          <Property function="Normal" name="Normal" />
        </PropertyGroup>
        <ProxyList>
          <Link name="Input" with_property="Input" />
        </ProxyList>
      </Hints>
    </Proxy>

Since 3D widget panels are now simply custom `pqPropertyWidget` subclasses for a
property-group, this code changes are follows:

    <Proxy class="vtkPVPlane" name="Plane">
      <InputProperty is_internal="1" name="Input" />
      <DoubleVectorProperty command="SetOrigin"
                            default_values="0.0 0.0 0.0"
                            name="Origin"
                            number_of_elements="3">
      ...
      </DoubleVectorProperty>
      <DoubleVectorProperty command="SetNormal"
                            default_values="1.0 0.0 0.0"
                            name="Normal"
                            number_of_elements="3">
      ...
      </DoubleVectorProperty>
      <DoubleVectorProperty animateable="1"
                            command="SetOffset"
                            default_values="0.0"
                            name="Offset"
                            number_of_elements="1">
      </DoubleVectorProperty>
      <PropertyGroup label="Plane Parameters" panel_widget="InteractivePlane">
        <Property function="Origin" name="Origin" />
        <Property function="Normal" name="Normal" />
        <Property function="Input"  name="Input" />
      </PropertyGroup>
      <Hints>
        <ProxyList>
          <Link name="Input" with_property="Input" />
        </ProxyList>
      </Hints>
    </Proxy>

Things to note:

- The `<PropertyGroup>` tag is no longer specified under `<Hints>`.
- The `panel_widget` attribute is used to indicate which custom widget to create
for this property group.

The functions for properties in the group supported by each type 3D widget (or
interactive widget as we now will call them) can be found by looking at the
documentation of each of the custom widget listed below. For the most past,
these have remained unchanged for previous function names.

Available custom property-widgets, the interactive widget they use and the
obsolete 3DWidget they correspond to are given in the table below.

New Widget | panel_widget | Interactive Widget used | Obsolete 3DWidget name
-----------|--------------|-------------------------|-------------------------------------
pqBoxPropertyWidget |InteractiveBox | BoxWidgetRepresentation | pqBoxWidget
pqHandlePropertyWidget | InteractiveHandle | HandleWidgetRepresentation | pqHandleWidget, pqPointSourceWidget
pqImplicitPlanePropertyWidget | InteractivePlane | ImplicitPlaneWidgetRepresentation |  pqImplicitPlaneWidget
pqLinePropertyWidget | InteractiveLine | LineSourceWidgetRepresentation | pqLineWidget, pqDistanceWidget, pqLineSourceWidget
pqSpherePropertyWidget | InteractiveSphere | SphereWidgetRepresentation | pqSphereWidget, pqOrbitWidget
pqSplinePropertyWidget | InteractiveSpline or InteractivePolyLine | SplineWidgetRepresentation or PolyLineWidgetRepresentation | pqSplineWidget, pqPolyLineWidget


Changes in 5.0
--------------

###Changed *Source* property to **GlyphType** on certain representations###

*Source* property on representations, which was not exposed in the UI has been
changed to *GlyphType* to better match its role. Since this property was not
exposed earlier, this change should not affect any users except custom
applications that explicitly used this property.


###Changes to caching infrastructure in **vtkPVDataRepresentation** and subclasses###

To avoid extra work and sync issues when propagating **UseCache** and
**CacheKey** flags from a vtkPVView to vtkPVDataRepresentation,
vtkPVDataRepresentation was updated to directly obtain thet values for these
flags from the view. To enable this, vtkPVDataRepresentation saves a weak
reference to the View it's being added to in vtkPVDataRepresentation::AddToView
and vtkPVDataRepresentation::RemoveFromView. Subclasses of
vtkPVDataRepresentation didn't consistently call the superclass implementation,
hence any representation subclass should ensure that it calls the superclass
implementations of AddToView and RemoveFromView for this to work correctly.
vtkPVView::AddRepresentationInternal checks for this and will raise a runtime
error if a representaiton is encountered that didn't properly set its View.

The above change also makes **vtkPVDataRepresentation::SetUseCache** and
**vtkPVDataRepresentation::SetCacheKey** obsolete. Subclasses no longer need to
provide any implementation for these methods and they should simply be removed.

Changes in 4.4
--------------

###Refactored pqView widget creation mechanisms###

pqView now adds a pure virtual method pqView::createWidget(). Subclasses should
override that method to create the QWidget in which the view is *rendered*. In
the past the subclasses implemented their own mechanisms to create the widget
and return it when `getWidget()` was called the first time.

`pqView::getWidget()` is now deprecated. Users should use pqView::widget()
instead. This method internally calls the pqView::createWidget() when
appropriate.

###Removed vtkPVGenericRenderWindowInteractor, vtkPVRenderViewProxy###

ParaView was subclassing vtkRenderWindowInteractor to create
`vtkPVGenericRenderWindowInteractor` to handle interaction. That piece of code
was potentially derived from an older implementation of
vtkRenderWindowInteractor and hence did what it did. Current implementation of
vtkRenderWindowInteractor lets the vtkInteractionStyle (and subclasses) do all
the heavy lifting. ParaView did that to some extent (since it has a
vtkPVInteractorStyle), but will was relying on
`vtkPVGenericRenderWindowInteractor`, `vtkPVRenderViewProxy` to propagate
interaction/still renders and other things. This has been refactored. ParaView
no longer uses a special vtkRenderWindowInteractor. All logic is handled by
observers on the standard vtkRenderWindowInteractor.

This change was done to make it easier to enable interactivity in `pvpython`.

See also: vtkSMRenderViewProxy::SetupInteractor(). Subclasses of pqView now pass
the interactor created by QVTKWidget to this method to initialize it.

See also: vtkSMViewProxyInteractorHelper.

###Refactored Color Preset Management (and UI)###

The color preset management has been completely refactored for this release.
This makes presets accessible to Python clients.
`vtkSMTransferFunctionPresets` is the class that manages the presets in the
*ServerManager* layer.

On the UI side, `pqColorPresetModel`, `pqColorPresetManager`, and
`pqColorMapModel` no longer exist. They have been replaced by `pqPresetDialog`.
Since the UI is completely redesigned, tests in custom applications
that used the color preset dialog will need to be updated as well.

The preset themselves are now serialized as JSON in the same form as the
settings JSON. ColorMaps in legacy XML format are still loadable from the UI. At
the same time, a tool `vtkLegacyColorMapXMLToJSON` is available to convert such
XMLs to JSON.

###Changes to `pqViewFrame`###

Commit [afaf6a510](https://gitlab.kitware.com/paraview/paraview/commit/afaf6a510ecb872c49461cd850022817741e1558)
changes the internal widgets created in `pqViewFrame` to add a new `QFrame` named
**CentralWidgetFrame** around the rendering viewport. While this shouldn't break any
code, this will certainly break tests since the widgets have changed. The change to the testing
XML is fairly simple. Just add the **CentralWidgetFrame** to the widget hierarchy at the appropriate
location. See the original
[merge request](https://gitlab.kitware.com/paraview/paraview/merge_requests/167)
for details.

###Changes to `vtkSMProxyIterator`###

`vtkSMProxyIterator::Begin(const char* groupName)` now additionally
sets the iterator mode to iterate over one group. This will likely
break any code that uses this method, since the iterated data list will
change. To recover the original functionality of the iterator, simply call
`vtkSMProxyIterator::SetModeToAll()` after calling
`vtkSMProxyIterator::Begin(const char* groupName)`.

Changes in 4.3
--------------

###Replaced `pqCurrentTimeToolbar` with `pqAnimationTimeWidget`###

`pqCurrentTimeToolbar` was a `QToolbar` subclass designed to be used in an
application to allow the users to view/change the animation time. We
reimplemented this class as `pqAnimationTimeWidget` making it a `QWidget`
instead of `QToolbar` to make it possible to use that in other widgets e.g.
`pqTimeInspectorWidget`. `pqAnimationTimeToolbar` was also changed to use
`pqAnimationTimeWidget` instead of `pqCurrentTimeToolbar`.

Besides just being a cleaner implementation (since it uses `pqPropertyLinks`
based linking between ServerManager and UI), `pqAnimationTimeWidget` also allows
applications to change the animation *play mode*, if desired. The rest of the
behavior of this class is similar to `pqCurrentTimeToolbar` and hence should not
affect applications (besides need to update any tests since the widget names
have now changed).

###Removed the need for plugins to specify a GUI_RESOURCE_FILE###

It is cleaner and easier to update one xml file for each plugin.
Hints in the server manager xml file already perform the same functionality and
the GUI xml file was deprecated.  Sources are now added to the **Sources** menu
and filters are added to the **Filters** menu when they are detected in the server
xml file.  To place sources/filters in a category, add the following hint to the
Proxy XML element (`category` and `icon` attributes are optional):

    <SourceProxy ...>
      ...
      <Hints>
        <ShowInMenu category="<category-name>" icon="<qt-image-resource>" />
      </Hints>
    </SourceProxy>

The ability to add filters to the sources menu (as done by a few plugins) is
removed by this change.

This change also modifies how readers are detected.  Readers now must provide a
hint to the ReaderManager in the server xml file to be detected as readers rather
than sources.  The backwards compatibility behavior that assumed any source with
a FileName attribute was a reader has been removed.

###Removed pqActiveView (use pqActiveObjects instead)###

`pqActiveView` was a long deprecated class which internally indeed used
pqActiveObjects. `pqActiveView` has now been removed. Any code using
`pqActiveView` can switch to using pqActiveObjects with very few code changes.

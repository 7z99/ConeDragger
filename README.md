### Documentation
**Constructor**
```
coneDragger.new(axis: Enum.NormalId, adornee: PVInstance?): ConeDragger
```
Returns a new ConeDragger

### Properties

```
coneDragger.adornee: PVInstance
```
The ConeDragger's current adornee. This should not be set by the developer, see `coneDragger:SetAdornee()`.

```
coneDragger.dragger: ConeHandleAdornment
```
The ConeDragger's ConeHandleAdornment instance.
```
coneDragger.state: DraggerState
```
The ConeDragger's current state. See Types section.

```
coneDragger.initialCFrame: CFrame
```
The CFrame of the adornee when dragging started.

```
coneDragger.direction: Vector3
```
The direction that the ConeDragger will drag in. This is relative to the part.

```
coneDragger.dead: boolean?
```
Whether the ConeDragger is dead (destroyed). If a ConeDragger is dead, none of its methods will work.





### Events

```
coneDragger.dragStarted
```
Fired when dragging starts.
Arguments: `currentCFrame: CFrame` -- the pivot of the PVInstance when the drag starts.

```
coneDragger.dragChanged
```
Fired when the dragger moves.
Arguments: `dragPosition: Vector3` -- where the PVInstance should be placed in world space.

```
coneDragger.dragEnded
```
Fired when dragging ends.

```
coneDragger.draggerStateChanged
```
Fired when the dragger's state changes.
Arguments: `currentState: DraggerState` -- the current and previous state of the dragger.





### Methods

Tag definitions:
`[chainable]` -- This method returns `self`, and thus can be chained (`coneDragger:method():method()...`)

```
[chainable] coneDragger:SetProperties(state: DraggerState, properties: {[string]: any}): ConeDragger
```
Sets the properties of the ConeHandleAdornment during the given state. Note that this method **completely overrides** the previous table of properties. It is not used to add properties to the existing table of properties.
Arguments:
`state` -- the state to set the properties for.
`properties` -- a table of properties to set.

```
[chainable] coneDragger:SetProperty(state: DraggerState, propertyName: string, propertyValue: any)
```
Sets the property of the ConeHandleAdornment during the given state. This adds it to the list of existing properties.
Arguments:
`state` -- the state to set the property for.
`propertyName` -- the name of the property to set.
`propertyValue` -- the value to set the property to.

```
coneDragger:GetProperty(state: DraggerState, propertyName: string): any
```
Returns the value of the given property.
Arguments:
`state` -- the state to get the property from.
`propertyName` -- the name of the property to get.

```
coneDragger:GetBoundingBoxSize(): Vector3
```
Returns the size of the bounding box of the adornee. In the case of a Model, returns `model:GetExtentsSize()`, in the case of a part, returns `part.Size`, otherwise returns `Vector3.zero`.

```
[chainable] coneDragger:StopDragging()
```
Forces the dragging to stop.

```
[chainable] coneDragger:SetAdornee(adornee: PVInstance?): ConeDragger
```
Sets the adornee of the ConeDragger.
Arguments:
`adornee` -- the new adornee.

```
[chainable] coneDragger:Destroy()
```
Destroys the ConeDragger.

```
coneDragger:Init()
```
Initializes the ConeDragger. This is called automatically when you call `coneDragger.new()`, so this method shouldn't be called by the developer.




### Types

```
coneDragger.ConeDragger
```
Represents the type of the ConeDragger object.

```
coneDragger.DraggerState
```
Represents a possible state of a ConeDragger.

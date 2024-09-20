--!strict

--[[
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
]]

local contextActionService = game:GetService('ContextActionService')
local userInputService = game:GetService('UserInputService')
local runService = game:GetService('RunService')
local players = game:GetService('Players')

local localPlayer = players.LocalPlayer

local playerGui = if localPlayer then localPlayer:WaitForChild('PlayerGui') else nil
local adornmentContainer = if playerGui then playerGui:FindFirstChild('AdornmentContainer') else nil -- need to make a ScreenGui so it doesn't reset on spawn

if localPlayer and not adornmentContainer then
	adornmentContainer = Instance.new('ScreenGui')
	adornmentContainer.Name = 'AdornmentContainer'
	adornmentContainer.ResetOnSpawn = false
	adornmentContainer.Parent = playerGui
end

local function rayPlaneIntersection(planePoint: Vector3, planeNormal: Vector3, origin: Vector3, direction: Vector3)
	return -((origin - planePoint):Dot(planeNormal)) / (planeNormal:Dot(direction))
end

local function closestPointOnLine(v: Vector2, a: Vector2, b: Vector2)
	local ab = b - a

	local abx = ab.X
	local aby = ab.Y

	local av = v - a
	local i = av:Dot(ab) / (abx * abx + aby * aby)
	return a + i * ab
end

local function v3ToV2(vec: Vector3)
	return Vector2.new(vec.X, vec.Y)
end

local module = {} :: ModuleType
module.__index = module

local internalMethods = {}

module.DraggerState = {
	Idle = 'Idle';
	Dragging = 'Dragging';
	Hovering = 'Dragging';
}

local activeDragger: ConeDragger

function internalMethods:_setState(state)
	if self.dead then
		return
	end
	
	local internal = self._internal

	self.state = state
	self:UpdateStyle()

	internal.draggerStateChanged:Fire(state)
end

function module:UpdateStyle(): ConeDragger
	if self.dead then
		return self
	end
	
	local internal = self._internal
	local state = self.state

	local statePropertiesForThisState = internal.stateProperties[state]

	if not statePropertiesForThisState then
		return self
	end

	for propertyName, propertyValue in statePropertiesForThisState do
		(self.dragger :: any)[propertyName] = propertyValue
	end

	return self
end

function module:SetProperties(state: string, propertyList: { [string]: any }): ConeDragger
	if self.dead then
		return self
	end
	
	assert(module.DraggerState[state], `Invalid DraggerState { state }`)

	local internal = self._internal
	internal.stateProperties[state] = propertyList

	if self.state == state then
		self:UpdateStyle()
	end

	return self
end

function module:SetProperty(state: string, propertyName: string, propertyValue: string): ConeDragger
	if self.dead then
		return self
	end
	
	assert(module.DraggerState[state], `Invalid DraggerState { state }`)

	local internal = self._internal
	local statePropertiesForThisState = internal.stateProperties[state]

	if not statePropertiesForThisState then
		statePropertiesForThisState = {}

		internal.stateProperties[state] = {}
	end

	statePropertiesForThisState[propertyName] = propertyValue

	if self.state == state then
		(self.dragger :: any)[propertyName] = propertyValue
	end

	return self
end

function module:GetProperty(state: string, propertyName: string)
	if self.dead then
		return
	end
	
	assert(module.DraggerState[state], `Invalid DraggerState { state }`)

	local internal = (self :: ConeDragger)._internal
	local statePropertiesForThisState = internal.stateProperties[state]

	if not statePropertiesForThisState then
		return nil
	end

	return statePropertiesForThisState[state]
end

function module:StopDragging()
	if self.dead then
		return self
	end
	
	local internal = (self :: ConeDragger)._internal

	if internal.runServiceConnection then
		internal.runServiceConnection:Disconnect()
		internal.runServiceConnection = nil
	end

	if internal.userInputServiceConnection then
		internal.userInputServiceConnection:Disconnect()
		internal.userInputServiceConnection = nil
	end

	self.state = 'Idle'
	self:UpdateStyle()

	contextActionService:UnbindAction('coneDragger_PreventWorldTouchInput')

	return self
end

function module:Init()
	if self.dead then
		return
	end
	
	local internal = self._internal

	local connections = internal.connections
	local dragStarted = internal.dragStarted
	local dragChanged = internal.dragChanged
	local dragEnded = internal.dragEnded
	
	local dir = self.direction
	local adornee = self.adornee
	local dragger = self.dragger

	table.insert(connections, dragger.MouseEnter:Connect(function()
		if self.state == 'Idle' then
			internalMethods._setState(self, 'Hovering')
		end
		
		internal.mouseIsOverDragger = true
	end))
	
	table.insert(connections, dragger.MouseLeave:Connect(function()
		if self.state ~= 'Dragging' then
			internalMethods._setState(self, 'Idle')
		end

		internal.mouseIsOverDragger = false
	end))

	local function queryMouseLocation()
		if not self.adornee then
			return Vector3.zero
		end
		
		local initialCFrame = self.initialCFrame

		local camera = workspace.CurrentCamera

		local coneCFrame = self.adornee:GetPivot() * dragger.CFrame
		local axisWorldSpace = initialCFrame:VectorToWorldSpace(self.direction)

		local coneCFrameScreen = v3ToV2(camera:WorldToViewportPoint(coneCFrame.Position))
		local coneDirScreen = v3ToV2(camera:WorldToViewportPoint((coneCFrame + axisWorldSpace).Position))
		local dirScreenSpace = (coneCFrameScreen - coneDirScreen).Unit

		local mouseLocation = userInputService:GetMouseLocation()

		local closestPointToMouse = closestPointOnLine(mouseLocation, coneCFrameScreen, coneCFrameScreen + dirScreenSpace)

		local pointWorldSpace = camera:ViewportPointToRay(closestPointToMouse.X, closestPointToMouse.Y)

		local planeNormal = axisWorldSpace:Cross(camera.CFrame.LookVector)

		local planeIntersectionZ = rayPlaneIntersection(initialCFrame.Position, planeNormal, pointWorldSpace.Origin, pointWorldSpace.Direction)

		local planeIntersectWorldSpace = ((pointWorldSpace.Direction * planeIntersectionZ) + pointWorldSpace.Origin)

		return planeIntersectWorldSpace
	end

	table.insert(connections, dragger.MouseButton1Down:Connect(function()
		if activeDragger and (activeDragger :: any) ~= self then
			internalMethods._setState(self, 'Idle')
			activeDragger:StopDragging()
		end

		self:StopDragging() -- just in case
		activeDragger = self
		
		if not self.adornee then
			return
		end

		local initialCFrame = self.adornee:GetPivot()
		self.initialCFrame = initialCFrame

		local planeIntersectWorldSpace = queryMouseLocation()

		-- the cone's position relative to the mouse. This will negate an issue when drag starts where the part might snap to fix its position
		local relativeOffset = initialCFrame:PointToObjectSpace(planeIntersectWorldSpace) 

		internal.runServiceConnection = runService.RenderStepped:Connect(function()
			local planeIntersectWorldSpace = queryMouseLocation()

			local objectSpace = initialCFrame:PointToObjectSpace(planeIntersectWorldSpace) * 
				dir:Abs() - -- the direction
				dragger.CFrame.Position + -- the dragger's position relative to the part
				(dragger.CFrame.Position - relativeOffset) -- the relative offset to fix the aforementioned snapping issue

			local worldSpace = initialCFrame:PointToWorldSpace(objectSpace)

			dragChanged:Fire(worldSpace)
		end)

		internal.userInputServiceConnection = userInputService.InputEnded:Connect(function(input)
			if (input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1) and self.state == 'Dragging' then
				self:StopDragging()
				internalMethods._setState(self, if internal.mouseIsOverDragger then 'Hovering' else 'Idle')
				dragEnded:Fire()
			end
		end)
		
		contextActionService:BindActionAtPriority('coneDragger_PreventWorldTouchInput', function(actionName, inputState, inputObject)
			if inputState == Enum.UserInputState.Change and inputObject.UserInputType == Enum.UserInputType.Touch then
				return Enum.ContextActionResult.Sink
			end
			
			return Enum.ContextActionResult.Pass
		end, false, Enum.ContextActionPriority.High.Value, Enum.UserInputType.MouseMovement, Enum.UserInputType.Touch)

		dragStarted:Fire(initialCFrame)
		internalMethods._setState(self, 'Dragging')
	end))
end

function module:GetBoundingBoxSize()
	if self.dead then
		return Vector3.zero
	end
	
	local adornee = self.adornee

	if not adornee then
		return Vector3.zero
	elseif adornee:IsA('BasePart') then
		return adornee.Size
	else
		return (adornee :: Model):GetExtentsSize()
	end
end

function module:SetAdornee(adornee: PVInstance?)
	if self.dead then
		return self
	end
	
	if self.state == 'Dragging' then
		self:StopDragging()
	end
	
	self.adornee = adornee
	self.dragger.Adornee = adornee
	
	return self
end

function module:Destroy()
	if self.dead then
		return
	end
	
	local internal = self._internal
	
	for _, connection in internal.connections do
		connection:Disconnect()
	end
	
	if internal.runServiceConnection then
		internal.runServiceConnection:Disconnect()
	end
	
	if internal.userInputServiceConnection then
		internal.userInputServiceConnection:Disconnect()
	end
	
	if internal.dragStarted then
		internal.dragStarted:Destroy()
	end
	
	if internal.dragChanged then
		internal.dragChanged:Destroy()
	end
	
	if internal.dragEnded then
		internal.dragEnded:Destroy()
	end
	
	if internal.draggerStateChanged then
		internal.draggerStateChanged:Destroy()
	end
	
	contextActionService:UnbindAction('coneDragger_PreventWorldTouchInput')
	
	self.dragger:Destroy()
	
	table.clear(self :: any);
	
	(self :: any).dead = true
end

function module.new(axis: Enum.NormalId, adornee: PVInstance?): ConeDragger -- creates and returns a ConeDragger
	assert(axis, 'No axis argument was passed to ConeDragger.new.')
	assert(typeof(axis) == 'EnumItem' and axis:IsA('NormalId'), 'Axis argument must be an Enum.NormalId.')
	assert(typeof(adornee) == 'Instance' and adornee:IsA('PVInstance') or adornee == nil, 'Adornee argument must be a PVInstance or nil.')

	local self = setmetatable({}, module :: { __index: ConeDraggerMeta }) :: ConeDragger

	self._internal = {
		dragStarted = Instance.new('BindableEvent');
		dragChanged = Instance.new('BindableEvent');
		dragEnded = Instance.new('BindableEvent');
		draggerStateChanged = Instance.new('BindableEvent');

		runServiceConnection = nil :: RBXScriptConnection?;
		userInputServiceConnection = nil :: RBXScriptConnection?;

		stateProperties = {};
		connections = {};
	}

	self.direction = Vector3.FromNormalId(axis)

	self.dragStarted = self._internal.dragStarted.Event :: DragStartedSignal
	self.dragChanged = self._internal.dragChanged.Event :: DragChangedSignal
	self.dragEnded = self._internal.dragEnded.Event :: RBXScriptSignal
	self.draggerStateChanged = self._internal.draggerStateChanged.Event :: DraggerStateChangedSignal

	self.state = 'Idle' :: DraggerState

	self.adornee = adornee
	self.dragger = Instance.new('ConeHandleAdornment')

	self.dragger.CFrame = CFrame.lookAlong(Vector3.zero, self.direction)
	self.dragger.Adornee = adornee
	self.dragger.Parent = adornmentContainer or adornee
	
	self:Init()

	return self
end

type ConeDraggerProperties = {
	adornee: PVInstance?;
	state: DraggerState;
	dragger: ConeHandleAdornment;
	initialCFrame: CFrame;
	direction: Vector3;
	dead: boolean?;

	dragStarted: DragStartedSignal;
	dragChanged: DragChangedSignal;
	dragEnded: RBXScriptSignal;
	draggerStateChanged: DraggerStateChangedSignal;

	_internal: { [any]: any };
}

type ConeDraggerMeta = {
	GetBoundingBoxSize: (self: ConeDragger) -> Vector3;
	UpdateStyle: (self: ConeDragger) -> ConeDragger;
	StopDragging: (self: ConeDragger) -> ConeDragger;
	Destroy: (self: ConeDragger) -> ();
	SetProperty: (self: ConeDragger, state: DraggerState, propertyName: string, value: any) -> ConeDragger;
	SetProperties: (self: ConeDragger, state: DraggerState, properties: { [string]: any }) -> ConeDragger;
	GetProperty: (self: ConeDragger, state: DraggerState, propertyName: string) -> any;
	SetAdornee: (self: ConeDragger, adornee: PVInstance?) -> ConeDragger;
	Init: (self: ConeDragger) -> ();
}

type DragStartedSignal = {
	Connect: (self: DragStartedSignal, callback: (currentCFrame: CFrame) -> ()) -> RBXScriptConnection;
	Once: (self: DragStartedSignal, callback: (currentCFrame: CFrame) -> ()) -> RBXScriptConnection;
	Wait: (self: DragStartedSignal) -> CFrame;
} & RBXScriptSignal

type DragChangedSignal = {
	Connect: (self: DragChangedSignal, callback: (dragPosition: Vector3) -> ()) -> RBXScriptConnection;
	Once: (self: DragChangedSignal, callback: (dragPosition: Vector3) -> ()) -> RBXScriptConnection;
	Wait: (self: DragChangedSignal) -> Vector3;
} & RBXScriptSignal

type DraggerStateChangedSignal = {
	Connect: (self: DraggerStateChangedSignal, callback: (currentState: DraggerState) -> ()) -> RBXScriptConnection;
	Once: (self: DraggerStateChangedSignal, callback: (currentState: DraggerState) -> ()) -> RBXScriptConnection;
	Wait: (self: DraggerStateChangedSignal) -> (DraggerState, DraggerState);
} & RBXScriptSignal

type ModuleType = ConeDraggerMeta & { [any]: any }

export type DraggerState = 'Idle' | 'Dragging' | 'Hovering'
export type ConeDragger = typeof(setmetatable({} :: ConeDraggerProperties, {} :: { __index: ConeDraggerMeta }))

return module :: {
	new: (normalId: Enum.NormalId, adornee: PVInstance?) -> ConeDragger;
}

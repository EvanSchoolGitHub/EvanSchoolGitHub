**Server Script*

```
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService = game:GetService("TweenService")
local Debris = game:GetService("Debris")
local Players = game:GetService("Players")

local cd = 2
local function shrine(rootPart)
	local shrine = script.MalevolentShrine:Clone()
	shrine:PivotTo(rootPart.CFrame + rootPart.CFrame.LookVector * -50)
	shrine.Parent = workspace
	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	params.FilterDescendantsInstances = {rootPart.Parent}
	local raycast = workspace:Raycast(shrine.PrimaryPart.Position,shrine.PrimaryPart.CFrame.UpVector * -100,params)
	if raycast then
		print(raycast)
		print(shrine.PrimaryPart.Position)
		shrine:MoveTo(raycast.Position + Vector3.new(0,shrine.PrimaryPart.Size.Y/2 - 15,0))
		print(shrine.PrimaryPart.Position)
		for i,part in shrine:GetDescendants() do
			if part:IsA("BasePart") then
				part.CanCollide = true
			end
		end
		Debris:AddItem(shrine,3)
	end
end

local function walkspeed(humanoid)
	humanoid.AutoRotate = false
	humanoid.WalkSpeed = 0
	humanoid.JumpPower = 0
	task.wait(3)
	humanoid.AutoRotate = true
	humanoid.WalkSpeed = 16
	humanoid.JumpPower = 50
end

local function animate(humanoid)
	local animation = Instance.new("Animation")
	animation.Parent = humanoid.Parent
	animation.AnimationId = "http://www.roblox.com/asset/?id=16336352266"
	local animTrack = humanoid.Animator:LoadAnimation(animation)
	animTrack:Play()
	Debris:AddItem(animation,6)
end

local function Particles(player)
	local energy = script.NegativeEnergy:Clone()
	local scratches = script.Scratches:Clone()
	
	scratches.Parent = player.Character.HumanoidRootPart.RootAttachment
	energy.Parent = player.Character.HumanoidRootPart.RootAttachment
	
	coroutine.wrap(function()
		local counter = 0
		repeat
			counter += .025
			scratches:Emit(1)
			energy:Emit(1)
			task.wait()
		until counter > cd
	end)()
	
	Debris:AddItem(scratches,6)
	Debris:AddItem(energy,6)
end

local function Dismantle(character)
	local originalHealth = character.Humanoid.Health
	local counter = 0
	repeat
		counter += .025
		character.Humanoid.Health -= .5
		task.wait()
	until counter > cd
end

local function Damage(mainPlayer)
	for i,player in Players:GetPlayers() do
		if (mainPlayer.Character.HumanoidRootPart.Position - player.Character.HumanoidRootPart.Position).Magnitude < 150 and player ~= mainPlayer then
			local character = player.Character or player.CharacterAdded:Wait()
			character.Humanoid.WalkSpeed = 0
			character.Humanoid.JumpPower = 0
			
			Particles(player)
			Dismantle(character)
			
			task.wait(cd)
			
			character.Humanoid.WalkSpeed = 16
			character.Humanoid.JumpPower = 50
		end
	end
end

local function playSound(rootPart)
	local sound = script.DomainSound:Clone()
	sound.Parent = rootPart.RootAttachment
	sound:Play()
	Debris:AddItem(sound,6)
end

local function tweenDomain(domain,player,rootPart)
	local info = TweenInfo.new(
		2,
		Enum.EasingStyle.Quad,
		Enum.EasingDirection.Out,
		0,
		false,
		0
	)
	local tween = TweenService:Create(domain,info,{Transparency = 0})
	tween:Play()
	tween.Completed:Connect(function()
		task.spawn(shrine,rootPart)
		Damage(player)
		task.wait(cd)
		local tween2 = TweenService:Create(domain,info,{Transparency = 1})
		tween2:Play()
		tween2.Completed:Connect(function()
			Debris:AddItem(domain,.1)
		end)
	end)
end

local function domain(rootPart,player)
	local domain = script.Domain:Clone()
	domain.Parent = workspace
	domain.Position = rootPart.Position
	tweenDomain(domain,player,rootPart)
	playSound(rootPart)
end

ReplicatedStorage.DomainExpansion.OnServerEvent:Connect(function(player)
	local character = player.Character or player.CharacterAdded:Wait()
	local humanoid = character:FindFirstChild("Humanoid")
	local rootPart = character.HumanoidRootPart
	animate(humanoid)
	domain(rootPart,player)
	walkspeed(humanoid)
end)
```

**Client Script**

```
local ContextActionService = game:GetService("ContextActionService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local Debris = game:GetService("Debris")
local camera = workspace.CurrentCamera

local player = Players.LocalPlayer
local character = player.Character or player.CharacterAdded:Wait()

local debounce = false

local function cameraMovement()
	local cameraPart = Instance.new("Part")
	cameraPart.Size = Vector3.one
	cameraPart.Anchored = true
	cameraPart.CanCollide = true
	cameraPart.Transparency = 0.5
	cameraPart.CFrame = character.HumanoidRootPart.CFrame * CFrame.Angles(math.cos(math.pi * 1.5),0,0) * CFrame.new(0,10,-25) + character.HumanoidRootPart.CFrame.LookVector
	cameraPart.Parent = workspace
	
	camera.CameraType = Enum.CameraType.Scriptable
	camera.CFrame = CFrame.lookAt(cameraPart.Position,character.HumanoidRootPart.Position + Vector3.new(0,10,0))
	camera.FieldOfView = 100
	Debris:AddItem(cameraPart,.1)
	task.wait(3)
	camera.FieldOfView = 70
	camera.CameraType = Enum.CameraType.Custom
	print(character.Humanoid)
	camera.CameraSubject = character.Humanoid
end

local function DomainExpansion(actionName,inputState)
	if actionName == "DomainExpansion" and inputState == Enum.UserInputState.Begin and not debounce then
		debounce = true
		ReplicatedStorage.DomainExpansion:FireServer()
		task.wait(0.1)
		task.spawn(cameraMovement)
		task.wait(10)
		debounce = false
	end
end

ContextActionService:BindAction("DomainExpansion",DomainExpansion,true,Enum.UserInputType.MouseButton1)
```

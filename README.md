**Server Script*

```
local Debris = game:GetService("Debris")
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local anims = {
	"http://www.roblox.com/asset/?id=15886118703",
	"http://www.roblox.com/asset/?id=15886190964"
}
local damagedAnims = {
	"http://www.roblox.com/asset/?id=16371001338",
	"http://www.roblox.com/asset/?id=16371010067"
}
local KnockbackDebounce = 5

local Punch = {}
local AnimationCounter = {}
local HitCounter = {}
local TimeTracker = {}
local HitPlayers = {}

function Punch:ComputeWhoIsHit(model)
	task.spawn(function()
		table.insert(HitPlayers,Players:GetPlayerFromCharacter(model))
		task.wait(.75)
		table.remove(HitPlayers,HitPlayers[Players:GetPlayerFromCharacter(model)])
	end)
end

function Punch:AnimateKnockback(humanoid)
	task.spawn(function()
		local animation = Instance.new("Animation")
		animation.AnimationId = "http://www.roblox.com/asset/?id=16371280853"
		local animTrack = humanoid:LoadAnimation(animation)
		animTrack:Play()
		Debris:AddItem(animation,1)
		task.wait(1)
		animTrack:Stop()
		animTrack = nil
	end)
end

function Punch:AnimateDamage(humanoid)
	task.spawn(function()
		local animation = Instance.new("Animation")
		local random = math.random(1,#damagedAnims)
		local chosen = damagedAnims[random]
		animation.AnimationId = chosen
		local animTrack = humanoid:LoadAnimation(animation)
		
		animTrack:Play()
		Debris:AddItem(animation,.5)
		task.wait(.5)
		animTrack:Stop()
		animTrack = nil
	end)
end

function Punch:ComputeKnockback(ID,model,root,currentTime)
	local stringID = tostring(ID)
	if TimeTracker[stringID] ~= nil and currentTime - TimeTracker[tostring(ID)] >= KnockbackDebounce then
		HitCounter[stringID] = 0
		return
	end
	if HitCounter[stringID] == nil then
		HitCounter[stringID] = 1
	elseif HitCounter[stringID] ~= nil and typeof(HitCounter[tostring(ID)]) == "number" then
		HitCounter[stringID] += 1
	end
	if HitCounter[stringID] == 4 then
		self:Knockback(model:FindFirstChild("HumanoidRootPart"),root)
		self:AnimateKnockback(model:FindFirstChild("Humanoid"))
		HitCounter[stringID] = 0
	end
end

function Punch:Knockback(root,puncher)
	local equation = (puncher.CFrame.LookVector * math.huge) + Vector3.new(0,350,0)
	if Players:GetPlayerFromCharacter(root.Parent) then
		ReplicatedStorage.FireImpulse:FireClient(Players:GetPlayerFromCharacter(root.Parent),equation)
	else
		root:ApplyImpulse(equation)
	end
end

function Punch:Particles(root)
	local steam = script.Parent.Steam:Clone()
	steam.Parent = root.RootAttachment
	steam:Emit(5)
	Debris:AddItem(steam,.5)
end

function Punch:Slowdown(humanoid)
	task.spawn(function()
		humanoid.WalkSpeed = 3
		humanoid.JumpPower = 0
		task.wait(.5)
		humanoid.WalkSpeed = 16
		humanoid.JumpPower = 50
	end)
end

function Punch:Hit(newHitbox,root,DAMAGE)
	local ID = Players:GetPlayerFromCharacter(root.Parent).UserId
	local currentTime = os.time()
	local playerModels = {}
	local debounceTable = {}
	local params = OverlapParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	params.FilterDescendantsInstances = {root.Parent}
	local parts = workspace:GetPartsInPart(newHitbox)
	if parts and parts[1] then
		for i,part in parts do
			if part.Parent:IsA("Model") and part.Parent:FindFirstChild("Humanoid") and part.Parent ~= root.Parent and not debounceTable[part.Parent] then
				debounceTable[part.Parent] = part.Parent
				table.insert(playerModels,part.Parent)
			end
		end
		for i,model in pairs(playerModels) do
			if model:IsA("Model") and model:FindFirstChild("Humanoid") and model ~= root.Parent and model:GetAttribute("Blocking") == false and (model:FindFirstChild("HumanoidRootPart").Position - root.Position).Magnitude < 5 then
				self:ComputeWhoIsHit(model)
				self:AnimateDamage(model:FindFirstChild("Humanoid"))
				self:Slowdown(model:FindFirstChild("Humanoid"))
				self:Sound(root)
				self:Particles(model:FindFirstChild("HumanoidRootPart"))
				
				model:FindFirstChild("Humanoid"):TakeDamage(DAMAGE)
				
				self:ComputeKnockback(ID,model,root,currentTime)
				TimeTracker[tostring(ID)] = currentTime
			elseif model:IsA("Model") and model:FindFirstChild("Humanoid") and model ~= root.Parent and model:GetAttribute("Blocking") == true and (model:FindFirstChild("HumanoidRootPart").Position - root.Position).Magnitude < 5 then
				local distance = (model.PrimaryPart.Position - root.Parent.PrimaryPart.Position).Unit
				local modelLook = model.PrimaryPart.CFrame.LookVector
				
				local dotProduct = distance:Dot(modelLook)
				if dotProduct < .5 then return end
				if dotProduct > .5 then
					self:ComputeWhoIsHit(model)
					self:AnimateDamage(model:FindFirstChild("Humanoid"))
					self:Slowdown(model:FindFirstChild("Humanoid"))
					self:Sound(root)
					self:Particles(model:FindFirstChild("HumanoidRootPart"))
					
					model:FindFirstChild("Humanoid"):TakeDamage(DAMAGE)
					
					self:ComputeKnockback(ID,model,root,currentTime)
					TimeTracker[tostring(ID)] = currentTime
				end
			end
		end
	end
end

function Punch:Reswap()
	for i,v in HitPlayers do
		for i,v in TimeTracker do
			if v - tick() >= 30 then
				table.remove(HitPlayers,v)
			end
		end
	end
end

function Punch:Animate(character)
	local animation = Instance.new("Animation")
	local ID = Players:GetPlayerFromCharacter(character).UserId
	
	if AnimationCounter[tostring(ID)] == 1 then
		AnimationCounter[tostring(ID)] += 1
	elseif AnimationCounter[tostring(ID)] == 2 then
		AnimationCounter[tostring(ID)] = 1
	else
		AnimationCounter[tostring(ID)] = 1
	end
	
	local chosen = anims[AnimationCounter[tostring(ID)]]
	
	animation.AnimationId = chosen
	animation.Parent = character
	local animTrack = character.Humanoid.Animator:LoadAnimation(animation)
	animTrack:Play()
	Debris:AddItem(animation,1)
end

function Punch:Sound(root)
	local sound = script.Parent.Sound:Clone()
	sound.Parent = root
	sound:Play()
	Debris:AddItem(sound,1)
end

function Punch:Init(hitbox,root,DAMAGE)
	if table.find(HitPlayers,Players:GetPlayerFromCharacter(root.Parent)) then return end
	local newHitbox = hitbox:Clone()
	newHitbox.CFrame = root.CFrame + root.CFrame.LookVector * 3
	newHitbox.Parent = workspace
	Debris:AddItem(newHitbox,.5)
	self:Hit(newHitbox,root,DAMAGE)
	self:Animate(root.Parent)
end

return Punch
```

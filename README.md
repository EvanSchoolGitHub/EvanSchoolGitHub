**Server Script*

```
--[[
The server is controlling player actions in the shop and when the player collects spirits. Spirits are the game currency and are used to give speed boosts in the shop below.
--]]
--Important Services
local DataStore = game:GetService("DataStoreService"):GetDataStore("37")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local ServerStorage = game:GetService("ServerStorage")
local CollectionService = game:GetService("CollectionService")
local Debris = game:GetService("Debris")
--Imporant folders
local Modules = ReplicatedStorage.Modules
local ServerModules = ServerStorage.ServerModules
local SpiritFolder = workspace.SpiritFolder
local Resources = ServerStorage.Resources
local Events = ReplicatedStorage.Events
local Functions = ReplicatedStorage.Functions
--Resources
local FX = Resources.FX
local SFX = Resources.SFX
local Levels = Resources.Levels
--Modules
local ballModule = require(ServerModules.Ball)
local collectModule = require(ServerModules.Collect)
--Instances
local spawnArea = workspace.SpawnArea
local shopHitbox = workspace.ShopHitbox
local itemRoller = workspace.ItemRoller
local numberOfBoxes = #ServerStorage.Particles:GetChildren()
local box0 = itemRoller.Box0
--Loop that makes spirits which players can collect forever, there is a maximum of 10.
coroutine.wrap(function()
	while task.wait(0.25) do
		if #SpiritFolder:GetChildren() < 10 then
			local randomSize = math.random(2,5)
			local randomPosition = spawnArea.Position + Vector3.new(math.random(-spawnArea.Size.X / 2,spawnArea.Size.X / 2),1,math.random(-spawnArea.Size.Z / 2,spawnArea.Size.Z / 2))
			local randomColor = Color3.fromRGB(math.random(0,255),math.random(0,255),math.random(0,255))
			local randomSize = Vector3.new(randomSize,randomSize,randomSize)

			local randomSpirit = ballModule.Create(randomColor,randomSize,randomPosition,SpiritFolder,FX.Stars)
			Debris:AddItem(randomSpirit,20)
			if randomSize.X >= 4 then
				randomSpirit:SetAttribute("Spirit",5)
			else
				randomSpirit:SetAttribute("Spirit",2)
			end
		end
	end
end)()
--FX play and the player earns a reward for getting the spirit
Events.Collect.OnServerEvent:Connect(function(player,instance,spirits,filled,unfilled)
	collectModule:Init(player,instance,spirits,filled,unfilled,SFX.Collect)
end)
--Player reloads their equipped spirit
Events.Reload.OnServerEvent:Connect(function(player)
	coroutine.wrap(function()
		player.CharacterAdded:Wait()
		task.wait(0.5)
		player.Character.Humanoid.WalkSpeed = player.leaderstats.Speed.Value
		local attachment = ServerStorage.PlayerData[player.Name]:FindFirstChild(player.Equipped.Value):FindFirstChildWhichIsA("Attachment"):Clone()
		attachment.Parent = player.Character.HumanoidRootPart
	end)()
end)
--Added to get the saved data
game.Players.PlayerAdded:Connect(function(player)
	--Instancing leaderstats and values required for the shop.
	local leaderstats = Instance.new("Folder")
	leaderstats.Name = "leaderstats"
	--[--------------------------------------------------------------------------------------------------]--
	local spirits = Instance.new("IntValue")
	spirits.Name = "Spirits"
	--[--------------------------------------------------------------------------------------------------]--
	local spiritsFolder = Instance.new("Folder")
	spiritsFolder.Name = player.Name
	--[--------------------------------------------------------------------------------------------------]--
	local inShop = Instance.new("BoolValue")
	inShop.Value = false
	inShop.Name = "InShop"
	--[--------------------------------------------------------------------------------------------------]--
	local equipped = Instance.new("StringValue")
	equipped.Name = "Equipped"
	--[--------------------------------------------------------------------------------------------------]--
	local speed = Instance.new("IntValue")
	speed.Name = "Speed"
	--[--------------------------------------------------------------------------------------------------]--
	equipped.Parent = player
	inShop.Parent = player
	spiritsFolder.Parent = game.ServerStorage.PlayerData
	leaderstats.Parent = player
	spirits.Parent = leaderstats
	speed.Parent = leaderstats
	--[--------------------------------------------------------------------------------------------------]--
	--Loading Data
	local items = "items-"..player.UserId
	local equippedKey = "equipped-"..player.UserId
	local spiritsKey = "spirits-"..player.UserId
	local speedKey = "speed-"..player.UserId
	--[--------------------------------------------------------------------------------------------------]--
	--items
	local itemSuccess, result1 = pcall(function()
		return DataStore:GetAsync(items)
	end)
	print(result1)
	if result1 ~= nil then
		for i,tool in pairs(result1) do
			if game.ServerStorage.PlayerRewards:FindFirstChild(tool) then
				--loading the particle which is the player's speed boost
				local particle = game.ServerStorage.PlayerRewards:FindFirstChild(tool):Clone()
				particle.Parent = game.ServerStorage.PlayerData:FindFirstChild(player.Name)
			end
		end
	else
		print("No data")
	end
	--[--------------------------------------------------------------------------------------------------]--
	--equipped
	local equipSuccess, result2 = pcall(function()
		return DataStore:GetAsync(equippedKey)
	end)
	if equipSuccess then
		print("Success: "..result2)
	elseif result2 then
		warn(result2)
	end
	if equipSuccess and result2 ~= nil then
		coroutine.wrap(function()
			for _,reward in game.ServerStorage.PlayerRewards:GetChildren() do
				if result2 == reward.Name then
					equipped.Value = result2
					coroutine.wrap(function()
						repeat
							task.wait(1)
							local clonedReward = reward:FindFirstChild(reward.Name):Clone()
							clonedReward.Parent = player.Character.HumanoidRootPart
						until clonedReward.Parent == player.Character.HumanoidRootPart
					end)()
				end
			end
		end)()
	end
	--[--------------------------------------------------------------------------------------------------]--
	--Spirits
	local spiritsSuccess, result3 = pcall(function()
		return DataStore:GetAsync(spiritsKey)
	end)
	if spiritsSuccess then
		spirits.Value = result3
		game.ServerStorage.Leaderboard:Fire(player)
	elseif result3 then
		warn(result3)
	end
	--[--------------------------------------------------------------------------------------------------]--
	--Speed
	local speedSuccess, result4 = pcall(function()
		return DataStore:GetAsync(speedKey)
	end)
	if speedSuccess then
		print("Success: "..result4)
	elseif result4 then
		warn(result4)
	end
	if result4 ~= nil then
		coroutine.wrap(function()
			speed.Value = result4
			if result4 < 16 then return end
			coroutine.wrap(function()
				repeat
					task.wait(1)
					player.Character.Humanoid.WalkSpeed = result4
				until player.Character.Humanoid.WalkSpeed == result4
			end)()
		end)()
	end
	--[--------------------------------------------------------------------------------------------------]--
end)
--Removing for Save
game.Players.PlayerRemoving:Connect(function(player)
	
	local items = {}
	for _,item in game.ServerStorage.PlayerData[player.Name]:GetChildren() do
		table.insert(items,item.Name)
	end
	local itemSuccess, result1 = pcall(function()
		return DataStore:SetAsync("items-"..player.UserId,items)
	end)
	
	local equippedSuccess, result2 = pcall(function()
		return DataStore:SetAsync("equipped-"..player.UserId,player.Equipped.Value)
	end)
	
	local spiritSuccess, result3 = pcall(function()
		return DataStore:SetAsync("spirits-"..player.UserId,player.leaderstats.Spirits.Value)
	end)
	
	local speedSuccess, result4 = pcall(function()
		return DataStore:SetAsync("speed-"..player.UserId,player.leaderstats.Speed.Value)
	end)
end)
--BindToClose for studio
game:BindToClose(function()
	for _,player in pairs(game.Players:GetPlayers()) do
		if player then
			player:Kick()
		end
	end
	task.wait(1)
end)
--Shop creation
for i = 1,numberOfBoxes-1,1 do
	local box = box0:Clone()
	box:PivotTo(box0.PrimaryPart.CFrame * CFrame.new(0,0,-16*i))
	box.Name = "Box"..i
	box.Parent = itemRoller
end
--Particle creation for each stand
for i,particle in ServerStorage.Particles:GetChildren() do
	if particle then
		local newParticle = particle:Clone()
		newParticle.CFrame = itemRoller["Box"..i-1].Hitbox.CFrame
		newParticle.Parent = itemRoller["Box"..i-1]
	end
end
--Client exiting the shop
Events.EnterExit.OnServerEvent:Connect(function(player,signal)
	if player then
		if signal == "Unenable" then
			player.Character.Humanoid.WalkSpeed = 0
			player.InShop.Value = true
		elseif signal == "Enable" then
			if player.Equipped.Value ~= nil or (player.Equipped.Value ~= "" and player.Character.HumanoidRootPart:FindFirstChild(ServerStorage.PlayerRewards[player.Equipped.Value])) then
				player.Character.Humanoid.WalkSpeed = ServerStorage.PlayerRewards[player.Equipped.Value].Information:GetAttribute("Speed")
			else
				player.Character.Humanoid.WalkSpeed = 16
			end
			player.InShop.Value = false
		end
	end
end)
--Shop invokes from client
Functions.RequestInformation.OnServerInvoke = function(player,signal,currentBox)
	if signal == "Buying" then
		local particleName
		for _,comp in itemRoller[currentBox]:GetDescendants() do
			if comp:IsA("Attachment") then
				particleName = comp.Parent.Name
			end
		end
		local information = ServerStorage.PlayerRewards[particleName].Information
		if player.leaderstats.Spirits.Value >= information:GetAttribute("Price") and not ServerStorage.PlayerData[player.Name]:FindFirstChild(particleName) and not player.Character.HumanoidRootPart:FindFirstChild(particleName) then
			local playerItem = ServerStorage.PlayerRewards[particleName]:Clone()
			playerItem.Parent = ServerStorage.PlayerData[player.Name]

			local newAttachment = ServerStorage.PlayerRewards[particleName]:FindFirstChild(particleName):Clone()
			newAttachment.Parent = player.Character.HumanoidRootPart

			local equippedItem = player.Character.HumanoidRootPart:FindFirstChild(player.Equipped.Value)
			if equippedItem then
				equippedItem:Destroy()
			end
			player.Character.Humanoid.WalkSpeed = information:GetAttribute("Speed")
			player.leaderstats.Speed.Value = information:GetAttribute("Speed")
			player.leaderstats.Spirits.Value -= information:GetAttribute("Price")
			player.Equipped.Value = particleName
			return "Success",information:GetAttribute("Speed")
		elseif player.leaderstats.Spirits.Value < information:GetAttribute("Price") and not ServerStorage.PlayerData[player.Name]:FindFirstChild(particleName) then
			print("Error")
		elseif ServerStorage.PlayerData[player.Name]:FindFirstChild(particleName) and not player.Character.HumanoidRootPart:FindFirstChild(particleName) and (player.Equipped.Value ~= nil or player.Equipped.Value ~= "") then
			local newAttachment = ServerStorage.PlayerRewards[particleName]:FindFirstChild(particleName):Clone()
			if player.Character.HumanoidRootPart:FindFirstChild(player.Equipped.Value) then
				player.Character.HumanoidRootPart:FindFirstChild(player.Equipped.Value):Destroy()
			end
			newAttachment.Parent = player.Character.HumanoidRootPart
			player.Character.Humanoid.WalkSpeed = information:GetAttribute("Speed")
			player.leaderstats.Speed.Value = information:GetAttribute("Speed")
			player.Equipped.Value = particleName
			return "Equippable",information:GetAttribute("Speed")
		end
	elseif signal == "Scrolling" then
		local particleName
		for _,comp in itemRoller[currentBox]:GetDescendants() do
			if comp:IsA("Attachment") then
				particleName = comp.Parent.Name
			end
		end
		local spirits = {}
		for i, spirit in pairs(game.ServerStorage.PlayerData[player.Name]:GetChildren()) do
			table.insert(spirits,spirit.Name)
		end
		local rootParticle = player.Character.HumanoidRootPart:FindFirstChild(particleName)
		if rootParticle and table.find(spirits,particleName) then -- They have it equipped and they have it
			return "Equipped"
		elseif table.find(spirits,particleName) and not rootParticle then --They don't have it equipped but they do have it
			return "Exists"
		elseif not table.find(spirits,particleName) and not rootParticle then
			return "Else"
		end
	elseif signal == "ShowInformation" then
		local booleanHasFirstEquipped1 = false
		local booleanHasFirstEquipped2 = false
		local particleName
		for _,comp in itemRoller[currentBox]:GetDescendants() do
			if comp:IsA("Attachment") then
				particleName = comp.Parent.Name
			end
		end
		local spirits = {}
		for i, spirit in pairs(game.ServerStorage.PlayerData[player.Name]:GetChildren()) do
			table.insert(spirits,spirit.Name)
		end
		local rootParticle = player.Character.HumanoidRootPart:FindFirstChild(particleName)
		if table.find(spirits,particleName) and not rootParticle then
			booleanHasFirstEquipped1 = true
		elseif table.find(spirits,particleName) and rootParticle then
			booleanHasFirstEquipped2 = true
		end
		local information = ServerStorage.PlayerRewards[particleName].Information
		return {particleName,information:GetAttribute("Price"),information:GetAttribute("Speed"),booleanHasFirstEquipped1,booleanHasFirstEquipped2}
	end
end
```

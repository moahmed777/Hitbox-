local player = game.Players.LocalPlayer
local screenGui = Instance.new("ScreenGui")
screenGui.Parent = game:GetService("CoreGui")

local runService = game:GetService("RunService")
local inputService = game:GetService("UserInputService")
local espEnabled = false
local espConnections = {}
local updateConnection = nil
local timePassed = 0
local suspectedKiller = nil

-- ⭐⭐ إضافة التحقق من ID الماب واسمه ⭐⭐
local TARGET_GAME_ID = 18687417158 -- ID الماب المطلوب
local TARGET_GAME_NAME = "Forsaken" -- اسم الماب المطلوب (يمكن إضافة أسماء أخرى)

-- قائمة بالأسماء المحتملة للماب (للتأكد من جميع الإصدارات)
local POSSIBLE_GAME_NAMES = {
    "Forsaken",
    "Forsaken (Nightmare)",
    "Forsaken [HORROR]",
    "Forsaken Game",
    "Forsaken Horror"
}

-- التحقق من الماب
local function isInTargetGame()
    -- التحقق من الاسم أولاً (الأكثر أماناً)
    local currentGameName = game:GetService("MarketplaceService"):GetProductInfo(game.PlaceId).Name
    print("Current game name:", currentGameName)
    
    -- التحقق من وجود اسم الماب المطلوب في اسم اللعبة
    for _, targetName in pairs(POSSIBLE_GAME_NAMES) do
        if string.find(currentGameName:lower(), targetName:lower()) then
            print("✓ Found target game by name:", targetName)
            return true
        end
    end
    
    -- إذا فشل التحقق بالاسم، نتحقق بالـ ID كنسخة احتياطية
    if game.PlaceId == TARGET_GAME_ID then
        print("✓ Found target game by ID:", TARGET_GAME_ID)
        return true
    end
    
    -- التحقق الإضافي: يمكن إضافة خصائص أخرى للماب
    local gameSuccess, gameInfo = pcall(function()
        return game:GetService("MarketplaceService"):GetProductInfo(game.PlaceId)
    end)
    
    if gameSuccess and gameInfo then
        -- التحقق من وصف الماب
        local description = gameInfo.Description or ""
        if string.find(description:lower(), "forsaken") then
            print("✓ Found target game by description")
            return true
        end
        
        -- التحقق من النوع
        local genre = gameInfo.Genre or ""
        if string.find(genre:lower(), "horror") or string.find(genre:lower(), "survival") then
            -- نتحقق أيضاً من الاسم
            for _, targetName in pairs(POSSIBLE_GAME_NAMES) do
                if string.find(gameInfo.Name:lower(), targetName:lower()) then
                    print("✓ Found target game by name in genre check")
                    return true
                end
            end
        end
    end
    
    return false
end

-- التحقق من الماب
local isInTargetGameResult = isInTargetGame()

-- إذا لم نكن في الماب المطلوب، لا ننشئ أي شيء
if not isInTargetGameResult then
    print("❌ Not in target game. Skipping ESP script.")
    print("Current Place ID:", game.PlaceId)
    
    -- يمكنك إضافة زر مؤقت للإشعار فقط
    local notificationButton = Instance.new("TextButton")
    notificationButton.Size = UDim2.new(0, 200, 0, 50)
    notificationButton.Position = UDim2.new(0.5, -100, 0.5, -25)
    notificationButton.BackgroundColor3 = Color3.new(0.2, 0, 0)
    notificationButton.TextColor3 = Color3.new(1, 1, 1)
    notificationButton.Text = "ESP Script\n(Not in Forsaken Map)"
    notificationButton.Font = Enum.Font.GothamBold
    notificationButton.TextSize = 14
    notificationButton.BorderSizePixel = 2
    notificationButton.BorderColor3 = Color3.new(1, 0, 0)
    notificationButton.Parent = screenGui
    
    local uiCorner = Instance.new("UICorner")
    uiCorner.CornerRadius = UDim.new(0, 10)
    uiCorner.Parent = notificationButton
    
    -- تدمير الزر بعد 5 ثواني
    task.wait(5)
    notificationButton:Destroy()
    
    return -- نوقف تنفيذ السكربت بالكامل
end

print("✅ Successfully loaded in Forsaken game map!")
print("✅ Game ID:", game.PlaceId)

-- إعدادات متقدمة (معدلة فقط للسرعة)
local SETTINGS = {
    UpdateInterval = 1,
    MinHealthDifference = 20,
    ToolWeight = 1.5,
    SpeedWeight = 2.5,
    HealthWeight = 1.2,
    MinSuspicionScore = 2.0,
    MinSpeedForKiller = 16,
}

SETTINGS.ParticleCheckWeight = 3.0

local WEAPON_PARTICLES = {
    "Smoke", "Fire", "Spark", "Glow", "Trail", "Muzzle", "Blood", "Explosion",
    "Magic", "Energy", "Electric", "Bullet", "Shell", "Hit", "Damage", "Poison",
    "Acid", "Frost", "Flame", "Lightning", "Aura", "Effect", "Particle", "Emit",
    "Sparks", "Flames", "Cloud", "Ring", "Burst", "Shock", "Wave", "Beam",
    "Orb", "Ball", "Shield", "Halo", "Rays", "Light", "Dark", "Shadow"
}

-- زر التبديل مع التعديلات المطلوبة
local button = Instance.new("TextButton")
button.Name = "ESPToggleButton"
button.Size = UDim2.new(0, 120, 0, 35)
button.Position = UDim2.new(0.5, -60, 0, 20)
button.BackgroundColor3 = Color3.new(0.1, 0.1, 0.1)
button.TextColor3 = Color3.new(1, 1, 1)
button.Text = "ESP: OFF"
button.Font = Enum.Font.GothamBold
button.TextSize = 16
button.BorderSizePixel = 2
button.BorderColor3 = Color3.new(0, 0, 0)
button.Parent = screenGui

local uiCorner = Instance.new("UICorner")
uiCorner.CornerRadius = UDim.new(0, 10)
uiCorner.Parent = button

-- وظيفة لجعل الزر قابلاً للسحب
local dragging = false
local dragStart
local startPos

local function updateDrag(input)
    local delta = input.Position - dragStart
    button.Position = UDim2.new(
        startPos.X.Scale, 
        startPos.X.Offset + delta.X,
        startPos.Y.Scale, 
        startPos.Y.Offset + delta.Y
    )
end

button.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPos = button.Position
        
        button.BackgroundColor3 = Color3.new(0.2, 0.2, 0.2)
        
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
                button.BackgroundColor3 = espEnabled and Color3.new(0, 0.5, 0) or Color3.new(0.1, 0.1, 0.1)
            end
        end)
    end
end)

button.InputChanged:Connect(function(input)
    if (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) and dragging then
        updateDrag(input)
    end
end)

inputService.InputChanged:Connect(function(input)
    if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
        updateDrag(input)
    end
end)

-- باقي الكود يبقى كما هو بدون تغيير
-- تنظيف التأثيرات لأي لاعب
local function clearEffects(character)
    if not character then return end
    if character:FindFirstChild("Highlight") then 
        character.Highlight:Destroy() 
    end
    if character:FindFirstChild("KillerBillboard") then 
        character.KillerBillboard:Destroy() 
    end
end

local function calculateDistance(player1, player2)
    if not player1.Character or not player2.Character then
        return 0
    end
    
    local humanoidRootPart1 = player1.Character:FindFirstChild("HumanoidRootPart")
    local humanoidRootPart2 = player2.Character:FindFirstChild("HumanoidRootPart")
    
    if not humanoidRootPart1 or not humanoidRootPart2 then
        return 0
    end
    
    return (humanoidRootPart1.Position - humanoidRootPart2.Position).Magnitude
end

local function checkForParticles(character)
    if not character then 
        return {found = false, count = 0, types = {}}
    end
    
    local particleCount = 0
    local particleTypes = {}
    local foundWeaponParticles = false
    
    for _, child in pairs(character:GetDescendants()) do
        local isParticle = false
        local particleName = ""
        
        if child:IsA("ParticleEmitter") then
            isParticle = true
            particleName = child.Name
        elseif child:IsA("Trail") then
            isParticle = true
            particleName = child.Name
        elseif child:IsA("Beam") then
            isParticle = true
            particleName = child.Name
        elseif child:IsA("Fire") then
            isParticle = true
            particleName = "Fire"
        elseif child:IsA("Smoke") then
            isParticle = true
            particleName = "Smoke"
        elseif child:IsA("Sparkles") then
            isParticle = true
            particleName = "Sparkles"
        elseif child:IsA("PointLight") then
            isParticle = true
            particleName = "PointLight"
        elseif child:IsA("SurfaceLight") then
            isParticle = true
            particleName = "SurfaceLight"
        end
        
        if isParticle then
            particleCount = particleCount + 1
            
            local lowerName = particleName:lower()
            for _, particleType in pairs(WEAPON_PARTICLES) do
                if lowerName:find(particleType:lower()) then
                    if not particleTypes[particleType] then
                        particleTypes[particleType] = 0
                    end
                    particleTypes[particleType] = particleTypes[particleType] + 1
                    foundWeaponParticles = true
                    break
                end
            end
            
            if not foundWeaponParticles then
                particleTypes["Unknown"] = (particleTypes["Unknown"] or 0) + 1
            end
        end
    end
    
    for _, tool in pairs(character:GetChildren()) do
        if tool:IsA("Tool") or tool:IsA("HopperBin") then
            for _, child in pairs(tool:GetDescendants()) do
                local isParticle = false
                local particleName = ""
                
                if child:IsA("ParticleEmitter") then
                    isParticle = true
                    particleName = child.Name
                elseif child:IsA("Trail") then
                    isParticle = true
                    particleName = child.Name
                elseif child:IsA("Fire") then
                    isParticle = true
                    particleName = "Fire"
                end
                
                if isParticle then
                    particleCount = particleCount + 1
                    
                    foundWeaponParticles = true
                    local lowerName = particleName:lower()
                    for _, particleType in pairs(WEAPON_PARTICLES) do
                        if lowerName:find(particleType:lower()) then
                            if not particleTypes[particleType] then
                                particleTypes[particleType] = 0
                            end
                            particleTypes[particleType] = particleTypes[particleType] + 1
                            break
                        end
                    end
                end
            end
        end
    end
    
    return {
        found = foundWeaponParticles,
        count = particleCount,
        types = particleTypes,
        hasAnyParticles = particleCount > 0
    }
end

local function addKillerBillboard(character, particleInfo)
    if not character then return end
    
    local existingBillboard = character:FindFirstChild("KillerBillboard")
    if existingBillboard then
        existingBillboard:Destroy()
    end
    
    local head = character:FindFirstChild("Head")
    if not head then return end
    
    local billboard = Instance.new("BillboardGui")
    billboard.Name = "KillerBillboard"
    billboard.Size = UDim2.new(0, 220, 0, 100)
    billboard.StudsOffset = Vector3.new(0, 3.5, 0)
    billboard.AlwaysOnTop = true
    billboard.Adornee = head
    billboard.Parent = character
    
    local killerLabel = Instance.new("TextLabel")
    killerLabel.Size = UDim2.new(1, 0, 0, 40)
    killerLabel.Position = UDim2.new(0, 0, 0, 0)
    killerLabel.BackgroundTransparency = 1
    if particleInfo and particleInfo.hasAnyParticles then
        killerLabel.Text = "🔴 KILLER 🔴"
    else
        killerLabel.Text = "KILLER"
    end
    killerLabel.TextColor3 = Color3.new(1, 0, 0)
    killerLabel.Font = Enum.Font.GothamBold
    killerLabel.TextSize = 20
    killerLabel.Parent = billboard
    
    local distanceLabel = Instance.new("TextLabel")
    distanceLabel.Size = UDim2.new(1, 0, 0, 30)
    distanceLabel.Position = UDim2.new(0, 0, 0, 20)
    distanceLabel.BackgroundTransparency = 1
    distanceLabel.TextColor3 = Color3.new(1, 1, 0)
    distanceLabel.Font = Enum.Font.GothamBold
    distanceLabel.TextSize = 18
    distanceLabel.Text = "Distance: Calculating..."
    distanceLabel.Parent = billboard
    
    local function updateDistance()
        if billboard.Parent and suspectedKiller and suspectedKiller.Character == character then
            local distance = calculateDistance(player, suspectedKiller)
            distanceLabel.Text = string.format("Distance: %.1f meters", distance)
        end
    end
    
    local distanceConnection
    distanceConnection = runService.Heartbeat:Connect(updateDistance)
    
    billboard.AncestryChanged:Connect(function()
        if distanceConnection then
            distanceConnection:Disconnect()
        end
    end)
end

local function applyESP(plr, playerInfo, isSuspected)
    if not plr.Character then return end
    
    clearEffects(plr.Character)
    
    local highlight = Instance.new("Highlight")
    highlight.Name = "Highlight"
    highlight.FillColor = isSuspected and Color3.new(1, 0, 0) or Color3.new(0, 1, 0)
    highlight.OutlineColor = isSuspected and Color3.new(1, 0.5, 0.5) or Color3.new(0.5, 1, 0.5)
    highlight.FillTransparency = 0.5
    highlight.OutlineTransparency = 0
    highlight.Parent = plr.Character
    
    espConnections[plr] = {highlight = highlight}
    
    if isSuspected then
        addKillerBillboard(plr.Character, playerInfo.Particles)
    end
end

local function clearAll()
    for plr, data in pairs(espConnections) do
        if plr.Character then 
            clearEffects(plr.Character) 
        end
    end
    espConnections = {}
    suspectedKiller = nil
end

local function hasTool(character)
    if not character then return 0 end
    local toolCount = 0
    for _, child in pairs(character:GetChildren()) do
        if child:IsA("Tool") or child:IsA("HopperBin") then
            toolCount = toolCount + 1
        end
    end
    return toolCount
end

local function getAverageSpeed()
    local totalSpeed = 0
    local playerCount = 0
    
    for _, p in pairs(game.Players:GetPlayers()) do
        if p ~= player and p.Character and p.Character:FindFirstChild("Humanoid") then
            totalSpeed = totalSpeed + p.Character.Humanoid.WalkSpeed
            playerCount = playerCount + 1
        end
    end
    
    return playerCount > 0 and (totalSpeed / playerCount) or 16
end

local function getAverageHealth()
    local totalHealth = 0
    local playerCount = 0
    
    for _, p in pairs(game.Players:GetPlayers()) do
        if p ~= player and p.Character and p.Character:FindFirstChild("Humanoid") then
            totalHealth = totalHealth + p.Character.Humanoid.Health
            playerCount = playerCount + 1
        end
    end
    
    return playerCount > 0 and totalHealth / playerCount or 100
end

local function gatherPlayerInfo(plr)
    if not plr.Character then return nil end
    
    local info = {
        Name = plr.Name,
        Speed = 16,
        Health = 100,
        Tools = 0,
        Character = plr.Character
    }
    
    local humanoid = plr.Character:FindFirstChild("Humanoid")
    if humanoid then
        info.Speed = humanoid.WalkSpeed
        info.Health = humanoid.Health
    end
    
    info.Tools = hasTool(plr.Character)
    
    info.Particles = checkForParticles(plr.Character)
    
    return info
end

local function calculateSuspicionScore(playerInfo, averageSpeed, averageHealth)
    local score = 0
    
    if playerInfo.Speed > averageSpeed then
        local speedDiff = playerInfo.Speed - averageSpeed
        
        local speedBonus = 0
        if speedDiff <= 1 then
            speedBonus = (speedDiff * 3) * SETTINGS.SpeedWeight
        else
            speedBonus = speedDiff * SETTINGS.SpeedWeight
        end
        
        score = score + speedBonus
        
        if speedDiff > 0 and speedDiff < 0.7 then
            score = score + 1.5
        end
    end
    
    if playerInfo.Health > averageHealth + SETTINGS.MinHealthDifference then
        score = score + ((playerInfo.Health - averageHealth) * SETTINGS.HealthWeight)
    end
    
    score = score + (playerInfo.Tools * SETTINGS.ToolWeight)
    
    if playerInfo.Particles and playerInfo.Particles.hasAnyParticles then
        local particleBonus = playerInfo.Particles.count * SETTINGS.ParticleCheckWeight
        if playerInfo.Particles.found then
            particleBonus = particleBonus * 1.5
        end
        score = score + particleBonus
    end
    
    if playerInfo.Speed < SETTINGS.MinSpeedForKiller then
        score = score * 0.7
    else
        if playerInfo.Speed > 16.5 then
            score = score * 1.3
        end
    end
    
    if playerInfo.Speed > averageSpeed and (playerInfo.Speed - averageSpeed) <= 1 then
        score = score + 2.0
    end
    
    return score
end

local function findSuspectedKiller()
    local playersInfo = {}
    local averageSpeed = getAverageSpeed()
    local averageHealth = getAverageHealth()
    local highestScore = 0
    local suspectedPlayer = nil
    local suspectedInfo = nil
    
    for _, p in pairs(game.Players:GetPlayers()) do
        if p ~= player then
            local info = gatherPlayerInfo(p)
            if info then
                info.Score = calculateSuspicionScore(info, averageSpeed, averageHealth)
                playersInfo[p] = info
                
                if info.Score > highestScore then
                    highestScore = info.Score
                    suspectedPlayer = p
                    suspectedInfo = info
                end
            end
        end
    end
    
    if suspectedPlayer and highestScore >= SETTINGS.MinSuspicionScore then
        return suspectedPlayer, suspectedInfo
    end
    
    return nil, nil
end

local function updateESP()
    local newSuspected, suspectInfo = findSuspectedKiller()
    
    if newSuspected ~= suspectedKiller then
        if suspectedKiller and suspectedKiller.Character then
            clearEffects(suspectedKiller.Character)
            if espConnections[suspectedKiller] then
                espConnections[suspectedKiller] = nil
            end
        end
        
        suspectedKiller = newSuspected
        
        if suspectedKiller then
            local info = gatherPlayerInfo(suspectedKiller)
            if info then
                applyESP(suspectedKiller, info, true)
            end
        end
    end
    
    for _, p in pairs(game.Players:GetPlayers()) do
        if p ~= player and p ~= suspectedKiller then
            if not espConnections[p] then
                local info = gatherPlayerInfo(p)
                if info then
                    applyESP(p, info, false)
                end
            end
        end
    end
end

button.MouseButton1Click:Connect(function()
    espEnabled = not espEnabled
    
    if espEnabled then
        button.Text = "ESP: ON"
        button.BackgroundColor3 = Color3.new(0, 0.5, 0)
    else
        button.Text = "ESP: OFF"
        button.BackgroundColor3 = Color3.new(0.1, 0.1, 0.1)
    end
    
    clearAll()
    
    if updateConnection then 
        updateConnection:Disconnect()
        updateConnection = nil
    end
    
    if espEnabled then
        updateESP()
        
        updateConnection = runService.Heartbeat:Connect(function(dt)
            timePassed = timePassed + dt
            if timePassed >= SETTINGS.UpdateInterval then
                timePassed = 0
                updateESP()
            end
        end)
    end
end)

button.MouseButton2Click:Connect(function()
    button.Position = UDim2.new(0.5, -60, 0, 20)
end)

game.Players.PlayerRemoving:Connect(function(plr)
    if espConnections[plr] then
        if plr.Character then 
            clearEffects(plr.Character) 
        end
        espConnections[plr] = nil
    end
end)

player.CharacterAdded:Connect(function()
    if espEnabled then
        updateESP()
    end
end)

print("✅ Killer ESP Script for Forsaken map loaded successfully!")

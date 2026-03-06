local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")
local Players = game:GetService("Players")
local VirtualUser = game:GetService("VirtualUser")
local Workspace = game:GetService("Workspace")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local Player = Players.LocalPlayer

-- Services (cached with WaitForChild to ensure they exist)
local Services = ReplicatedStorage:WaitForChild("Shared"):WaitForChild("Packages"):WaitForChild("Knit"):WaitForChild("Services")
local CreatePartyRF = Services:WaitForChild("PartyService"):WaitForChild("RF"):WaitForChild("CreateParty")
local ProximityRF = Services:WaitForChild("ProximityService"):WaitForChild("RF"):WaitForChild("Functionals")
local ActivatePartyRF = Services:WaitForChild("PartyService"):WaitForChild("RF"):WaitForChild("Activate")
local ToolActivatedRF = Services:WaitForChild("ToolService"):WaitForChild("RF"):WaitForChild("ToolActivated")

-- State variables
local Character, HumanoidRootPart, Humanoid
local currentBoss = nil
local isRunning = true
local lastActivityTime = tick()
local antiAfkConnection = nil
local followConnection = nil
local bodyPos, bodyGyro = nil, nil

local TARGET_CFRAME = CFrame.new(88.9365387, 129.05513, 226.935242)
local EXECUTION_COUNT1 = 500
local TWEEN_SPEED = 60
local SMOOTH_SPEED = 18
local BEHIND_OFFSET = Vector3.new(0, 2, 6)
local ATTACK_DELAY = 0.22

local VOID_Y = -20
local SAFE_HEIGHT = 5
local MAX_BOSS_DISTANCE = 150

-- UTILITY FUNCTIONS

local function updateActivity()
    lastActivityTime = tick()
end

local function setupCamera()
    pcall(function()
        Player.CameraMaxZoomDistance = 1000
        Player.CameraMinZoomDistance = 10
        Player.DevCameraOcclusionMode = Enum.DevCameraOcclusionMode.Invisicam
        Player.CameraMode = Enum.CameraMode.Classic
        
        local camera = Workspace.CurrentCamera
        if camera and Humanoid then
            camera.CameraSubject = Humanoid
            camera.CameraType = Enum.CameraType.Custom
        end
    end)
end

local function cleanupFight()
    print("Cleaning up fight...")
    
    if followConnection then 
        followConnection:Disconnect() 
        followConnection = nil
    end
    
    if bodyPos then 
        pcall(function() bodyPos:Destroy() end)
        bodyPos = nil
    end
    
    if bodyGyro then
        pcall(function() bodyGyro:Destroy() end)
        bodyGyro = nil
    end
    
    if Humanoid then
        pcall(function()
            Humanoid.AutoRotate = true
            Humanoid.PlatformStand = false
        end)
    end
    
    if HumanoidRootPart then
        pcall(function()
            HumanoidRootPart.Anchored = false
        end)
    end
    
    currentBoss = nil
    updateActivity()
end

local function isCharacterDead()
    if not Humanoid then return true end
    if Humanoid.Health <= 0 then return true end
    if not HumanoidRootPart or not HumanoidRootPart.Parent then return true end
    return false
end

-- CHARACTER INIT

local function initCharacter()
    print("Initializing character...")
    Character = Player.Character
    if not Character then
        Character = Player.CharacterAdded:Wait()
    end
    
    local startWait = tick()
    repeat
        HumanoidRootPart = Character:FindFirstChild("HumanoidRootPart")
        Humanoid = Character:FindFirstChild("Humanoid")
        task.wait(0.1)
    until (HumanoidRootPart and Humanoid) or (tick() - startWait > 10)
    
    if not HumanoidRootPart or not Humanoid then
        warn("Failed to find character parts!")
        return false
    end
    
    task.wait(0.5)
    setupCamera()
    
    print("Character initialized successfully")
    return true
end

-- ANTI-AFK

local function setupAntiAfk()
    if antiAfkConnection then
        antiAfkConnection:Disconnect()
    end
    
    antiAfkConnection = Player.Idled:Connect(function(idleTime)
        VirtualUser:Button2Down(Vector2.new(0,0), Workspace.CurrentCamera.CFrame)
        task.wait(0.1)
        VirtualUser:Button2Up(Vector2.new(0,0), Workspace.CurrentCamera.CFrame)
        print("Anti-AFK triggered")
    end)
    
    RunService.Heartbeat:Connect(function()
        if tick() - lastActivityTime > 600 then
            VirtualUser:CaptureController()
            VirtualUser:ClickButton1(Vector2.new(0,0))
            lastActivityTime = tick()
            print("Anti-AFK heartbeat")
        end
    end)
end

setupAntiAfk()

-- BOSS FUNCTIONS

local function isBossOnFloor(bossHRP)
    if not bossHRP or not bossHRP.Parent then return false end
    
    local rayParams = RaycastParams.new()
    rayParams.FilterType = Enum.RaycastFilterType.Blacklist
    rayParams.FilterDescendantsInstances = {bossHRP.Parent, Character}
    rayParams.IgnoreWater = true

    local origin = bossHRP.Position
    local direction = Vector3.new(0, -50, 0)

    local result = Workspace:Raycast(origin, direction, rayParams)
    if result then
        local distance = (origin - result.Position).Magnitude
        return distance <= 30
    end
    return false
end

local function getClosestBoss()
    local living = Workspace:FindFirstChild("Living")
    if not living then return nil end

    local closest = nil
    local shortest = math.huge

    for _, model in ipairs(living:GetChildren()) do
        if model:IsA("Model") and (string.find(model.Name, "Asura") or string.find(model.Name, "Incarnate")) then
            local hrp = model:FindFirstChild("HumanoidRootPart")
            local hum = model:FindFirstChild("Humanoid")
            
            if hrp and hum and hum.Health > 0 and hrp.Position.Y > 10 and isBossOnFloor(hrp) then
                local dist = (HumanoidRootPart.Position - hrp.Position).Magnitude
                if dist < shortest then
                    shortest = dist
                    closest = model
                end
            end
        end
    end

    return closest
end

local function smoothTweenToTarget(targetCFrame, speed)
    if not HumanoidRootPart or not HumanoidRootPart.Parent then return nil end
    
    local distance = (HumanoidRootPart.Position - targetCFrame.Position).Magnitude
    local duration = math.max(distance / speed, 0.8)
    
    local tween
    local success = pcall(function()
        tween = TweenService:Create(
            HumanoidRootPart, 
            TweenInfo.new(duration, Enum.EasingStyle.Cubic, Enum.EasingDirection.InOut), 
            {CFrame = targetCFrame}
        )
        tween:Play()
    end)
    
    if success then
        updateActivity()
        return tween
    end
    return nil
end

local function fullReset()
    print("Resetting - no bosses found")
    updateActivity()
    cleanupFight()
    
    if isCharacterDead() then
        print("Character dead, waiting...")
        task.wait(3)
        return
    end
    
    HumanoidRootPart.Anchored = false
    
    local resetTween = smoothTweenToTarget(TARGET_CFRAME, TWEEN_SPEED)
    if resetTween then
        resetTween.Completed:Wait()
    end
    
    task.wait(0.5)
end

-- EVENTS

Player.CharacterAdded:Connect(function(newChar)
    print("Character respawned - reinitializing...")
    cleanupFight()
    task.wait(1)
    
    if initCharacter() then
        currentBoss = nil
        print("Ready after respawn")
    else
        warn("Failed to initialize after respawn")
    end
    
    updateActivity()
end)

-- MAIN LOOP

initCharacter()

while isRunning do
    local success, err = pcall(function()
        
        if isCharacterDead() then
            print("Character dead, waiting for respawn...")
            cleanupFight()
            task.wait(2)
            return
        end
        
        local boss = getClosestBoss()
        
        -- NO BOSS - SPAM MODE
        if not boss then
            if currentBoss then
                fullReset()
            else
                updateActivity()
                
                if isCharacterDead() then return end
                
                HumanoidRootPart.Anchored = false
                
                local distance = (HumanoidRootPart.Position - TARGET_CFRAME.Position).Magnitude
                local duration = distance / TWEEN_SPEED
                local tween = TweenService:Create(HumanoidRootPart, TweenInfo.new(duration, Enum.EasingStyle.Linear), {CFrame = TARGET_CFRAME})
                tween:Play()
                tween.Completed:Wait()

                if isCharacterDead() then return end
                
                HumanoidRootPart.Anchored = true
                updateActivity()

                local lockConnection = RunService.RenderStepped:Connect(function()
                    if HumanoidRootPart and HumanoidRootPart.Parent then
                        HumanoidRootPart.CFrame = TARGET_CFRAME
                    end
                end)

                for i = 1, EXECUTION_COUNT1 do
                    if isCharacterDead() then break end
                    
                    task.spawn(function()
                        pcall(function()
                            ActivatePartyRF:InvokeServer()
                            CreatePartyRF:InvokeServer("Asura's Incarnate")
                            ProximityRF:InvokeServer(Workspace.Proximity.CreateParty)
                        end)
                    end)
                    task.wait()
                    updateActivity()
                end

                lockConnection:Disconnect()
                
                if HumanoidRootPart and HumanoidRootPart.Parent then
                    HumanoidRootPart.Anchored = false
                end
                
                task.wait(2.5)
            end
            
            return
        end

        -- BOSS FOUND - FIGHT MODE
        currentBoss = boss
        local bossHRP = boss:WaitForChild("HumanoidRootPart")
        local bossHum = boss:WaitForChild("Humanoid")
        updateActivity()

        Humanoid.PlatformStand = false
        Humanoid.AutoRotate = false

        -- Approach
        local targetPos = bossHRP.Position + BEHIND_OFFSET
        local targetCFrame = CFrame.new(targetPos, bossHRP.Position)
        local dist = (HumanoidRootPart.Position - targetPos).Magnitude
        local duration = dist / TWEEN_SPEED
        local approachTween = TweenService:Create(HumanoidRootPart, TweenInfo.new(duration, Enum.EasingStyle.Linear), {CFrame = targetCFrame})
        approachTween:Play()
        approachTween.Completed:Wait()
        updateActivity()

        -- BodyMovers
        bodyPos = Instance.new("BodyPosition")
        bodyPos.MaxForce = Vector3.new(8000, 8000, 8000)
        bodyPos.D = 800
        bodyPos.P = 5000
        bodyPos.Parent = HumanoidRootPart

        bodyGyro = Instance.new("BodyGyro")
        bodyGyro.MaxTorque = Vector3.new(math.huge, math.huge, math.huge)
        bodyGyro.P = 3000
        bodyGyro.Parent = HumanoidRootPart

        local lastAttack = 0
        local lastHealth = bossHum.Health
        local lastPosition = bossHRP.Position
        local isTransforming = false
        local transformTween = nil
        local stableTargetPos = nil

        followConnection = RunService.Heartbeat:Connect(function(dt)
            
            if isCharacterDead() then return end
            
            -- VOID PROTECTION
            if HumanoidRootPart.Position.Y < VOID_Y then
                warn("Void detected - recovering")
                cleanupFight()
                pcall(function()
                    HumanoidRootPart.CFrame = TARGET_CFRAME + Vector3.new(0, SAFE_HEIGHT, 0)
                end)
                return
            end
            
            updateActivity()
            
            if not boss.Parent or not bossHum or bossHum.Health <= 0 then return end
            if not bossHRP or not bossHRP.Parent then return end
            if not isBossOnFloor(bossHRP) then return end
            
            local distToBoss = (HumanoidRootPart.Position - bossHRP.Position).Magnitude

            if distToBoss > MAX_BOSS_DISTANCE then
                print("Boss too far - repositioning")
                cleanupFight()
                
                local safePos = bossHRP.Position + BEHIND_OFFSET + Vector3.new(0, SAFE_HEIGHT, 0)
                local tween = TweenService:Create(
                    HumanoidRootPart,
                    TweenInfo.new(1.2, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
                    {CFrame = CFrame.new(safePos, bossHRP.Position)}
                )
                tween:Play()
                return
            end

            local currentPos = bossHRP.Position
            local currentHealth = bossHum.Health
            local movedDistance = (currentPos - lastPosition).Magnitude

            -- TRANSFORMATION DETECTION
            if movedDistance < 0.03 and math.abs(currentHealth - lastHealth) < 0.5 then
                if not isTransforming then
                    isTransforming = true
                    stableTargetPos = HumanoidRootPart.Position
                    print("Transform started")
                end
            else
                if isTransforming then
                    isTransforming = false
                    stableTargetPos = nil
                    if transformTween then
                        pcall(function() transformTween:Cancel() end)
                        transformTween = nil
                    end
                    print("Transform ended")
                end
            end

            lastPosition = currentPos
            lastHealth = currentHealth

            if isTransforming then
                local desiredPos = bossHRP.Position + BEHIND_OFFSET
                
                if stableTargetPos then
                    stableTargetPos = stableTargetPos:Lerp(desiredPos, dt * 3)
                else
                    stableTargetPos = desiredPos
                end
                
                bodyPos.Position = stableTargetPos
                bodyGyro.CFrame = CFrame.new(HumanoidRootPart.Position, bossHRP.Position)
                
                local distToTarget = (HumanoidRootPart.Position - desiredPos).Magnitude
                if distToTarget > 2 and not transformTween then
                    local smoothTarget = CFrame.new(desiredPos, bossHRP.Position)
                    transformTween = smoothTweenToTarget(smoothTarget, SMOOTH_SPEED)
                    
                    task.delay(1, function()
                        transformTween = nil
                    end)
                end
                
            else
                if transformTween then
                    pcall(function() transformTween:Cancel() end)
                    transformTween = nil
                end
                
                bodyPos.Position = bossHRP.Position + BEHIND_OFFSET
                bodyPos.D = 1000
                bodyGyro.CFrame = CFrame.new(HumanoidRootPart.Position, bossHRP.Position)

                -- ATTACK
                local now = tick()
                if now - lastAttack >= ATTACK_DELAY then
                    lastAttack = now
                    pcall(function()
                        ToolActivatedRF:InvokeServer("Weapon", nil)
                    end)
                end
            end
        end)

        -- Wait for boss death
        local deathCheckStart = tick()
        while boss.Parent and bossHum and bossHum.Health > 0 do
            task.wait(0.05)
            updateActivity()
            
            if isCharacterDead() then
                print("Player died during boss fight")
                break
            end
            
            if tick() - deathCheckStart > 180 then
                print("Boss timeout")
                break
            end
        end

        print("Boss defeated or timeout - cleaning up")
        cleanupFight()
        task.wait(0.5)
    end)
    
    if not success then
        warn("Error in main loop: " .. tostring(err))
        cleanupFight()
        task.wait(2)
    end
    
    task.wait(0.1)
end

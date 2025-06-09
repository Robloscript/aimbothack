local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")

local localPlayer = Players.LocalPlayer
local mouse = localPlayer:GetMouse()
local camera = workspace.CurrentCamera

-- Configurações
local AIM_KEY = Enum.UserInputType.MouseButton2 -- Botão direito do mouse
local AIM_FOV = 30 -- Campo de visão do aimbot (graus)
local SMOOTHNESS = 0.2 -- Suavização do movimento (0-1)
local MAX_DISTANCE = 1000 -- Distância máxima de detecção

-- Variáveis
local target = nil
local targetHead = nil
local aiming = false

-- Função para verificar se há parede entre o jogador e o alvo
local function isVisible(targetPart)
    local origin = camera.CFrame.Position
    local direction = (targetPart.Position - origin).Unit
    local raycastParams = RaycastParams.new()
    raycastParams.FilterDescendantsInstances = {localPlayer.Character}
    raycastParams.FilterType = Enum.RaycastFilterType.Blacklist
    
    local raycastResult = workspace:Raycast(origin, direction * MAX_DISTANCE, raycastParams)
    
    if raycastResult then
        local hitPart = raycastResult.Instance
        return hitPart:IsDescendantOf(targetPart.Parent)
    end
    
    return false
end

-- Função para encontrar o melhor alvo
local function findBestTarget()
    local bestTarget = nil
    local bestScore = 0
    local localCharacter = localPlayer.Character
    local localHead = localCharacter and localCharacter:FindFirstChild("Head")
    
    if not localHead then return nil end
    
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= localPlayer and player.Character then
            local character = player.Character
            local head = character:FindFirstChild("Head")
            
            if head then
                -- Verifica se está dentro do FOV
                local direction = (head.Position - localHead.Position).Unit
                local lookVector = camera.CFrame.LookVector
                local angle = math.deg(math.acos(direction:Dot(lookVector)))
                
                if angle <= AIM_FOV then
                    -- Verifica visibilidade
                    if isVisible(head) then
                        -- Calcula score baseado na distância e ângulo
                        local distance = (head.Position - localHead.Position).Magnitude
                        local score = (1 - angle/AIM_FOV) * (1 - math.min(distance/MAX_DISTANCE, 1))
                        
                        if score > bestScore then
                            bestScore = score
                            bestTarget = player
                        end
                    end
                end
            end
        end
    end
    
    return bestTarget
end

-- Controles do aimbot
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    
    if input.UserInputType == AIM_KEY then
        aiming = true
        target = findBestTarget()
        
        if target and target.Character then
            targetHead = target.Character:FindFirstChild("Head")
        end
    end
end)

UserInputService.InputEnded:Connect(function(input, gameProcessed)
    if input.UserInputType == AIM_KEY then
        aiming = false
        target = nil
        targetHead = nil
    end
end)

-- Atualização suave do aimbot
RunService.RenderStepped:Connect(function()
    if aiming then
        -- Verifica se o alvo atual ainda é válido
        if not target or not targetHead or not targetHead.Parent or not isVisible(targetHead) then
            target = findBestTarget()
            targetHead = target and target.Character and target.Character:FindFirstChild("Head")
        end
        
        if target and targetHead and targetHead.Parent then
            local cameraCFrame = camera.CFrame
            local targetPosition = targetHead.Position
            
            -- Calcula a direção para o alvo
            local direction = (targetPosition - cameraCFrame.Position).Unit
            
            -- Suaviza o movimento
            local currentLook = cameraCFrame.LookVector
            local smoothedLook = currentLook:Lerp(direction, SMOOTHNESS)
            
            -- Atualiza a câmera
            camera.CFrame = CFrame.lookAt(cameraCFrame.Position, cameraCFrame.Position + smoothedLook)
        end
    end
end)

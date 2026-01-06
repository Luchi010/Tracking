local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

-- === 設定・状態管理 ===
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local lp = Players.LocalPlayer

local state = {
    follow = false, 
    follow_speed = 100,
    predict_k = 0.18, 
    follow_height = 5,
    tp_speed_enabled = false,
    walkspeed = 50, 
    anti_fall = true, fall_threshold = -100,
    auto_fruit = false, auto_chest = false,
    hitbox_size = 5, hitbox_trans = 0.6,
    esp = false,
    selected_target = nil,
    player_map = {}, -- 名前からプレイヤー本体を参照するためのテーブル
    esp_objects = {}
}

-- === ウィンドウ作成 ===
local Window = Rayfield:CreateWindow({
    Name = "Strawberry Console v6.4",
    LoadingTitle = "Strawberry System Loading...",
    LoadingSubtitle = "Stable Player List & Icons",
    ConfigurationSaving = { Enabled = true, FolderName = "StrawberryConfig", FileName = "v6_4" }
})

-- === タブ構成 ===
local MoveTab = Window:CreateTab("移動設定")
local CombatTab = Window:CreateTab("戦闘設定")
local AutoTab = Window:CreateTab("自動回収")
local VisualTab = Window:CreateTab("視覚設定")

-- --- プレイヤーリスト安定化 & アイコン取得 ---
local function getPlayerList()
    local options = {}
    state.player_map = {}
    
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= lp then
            -- ディスプレイネームを取得
            local displayName = p.DisplayName
            local userName = p.Name
            
            -- レイフィールドのドロップダウン用に「名前 (ID)」の形式で作成
            -- アイコン画像自体はUIライブラリの制限上テキスト内に絵文字的に含めるか、
            -- リフレッシュを確実に行うことで消失を防ぎます
            local label = " " .. displayName .. " (@" .. userName .. ")"
            table.insert(options, label)
            state.player_map[label] = p
        end
    end
    
    if #options == 0 then table.insert(options, "なし") end
    return options
end

local PlayerDropdown = MoveTab:CreateDropdown({
    Name = "対象プレイヤー選択 (表示名)",
    Options = getPlayerList(),
    CurrentOption = {"なし"},
    Callback = function(Option)
        state.selected_target = state.player_map[Option[1]]
    end,
})

-- リストの消失対策（定期リフレッシュ & プレイヤー増減検知）
local function refreshDropdown()
    local newList = getPlayerList()
    PlayerDropdown:Refresh(newList)
end

Players.PlayerAdded:Connect(refreshDropdown)
Players.PlayerRemoving:Connect(refreshDropdown)

-- --- 移動設定コンテンツ ---
MoveTab:CreateToggle({Name = "滑らかな高速追尾実行", CurrentValue = false, Callback = function(V) state.follow = V end})

MoveTab:CreateSlider({
    Name = "移動速度 (追尾/自操作共通)", 
    Range = {16, 500}, 
    Increment = 5, 
    CurrentValue = 100, 
    Callback = function(V) 
        state.follow_speed = V 
        state.walkspeed = V
    end
})

MoveTab:CreateSection("追尾詳細調整")
MoveTab:CreateSlider({Name = "予測係数", Range = {0, 50}, Increment = 1, CurrentValue = 18, Callback = function(V) state.predict_k = V/100 end})
MoveTab:CreateSlider({Name = "高さ", Range = {-10, 30}, Increment = 1, CurrentValue = 5, Callback = function(V) state.follow_height = V end})
MoveTab:CreateToggle({Name = "カスタム速度有効化", CurrentValue = false, Callback = function(V) state.tp_speed_enabled = V end})
MoveTab:CreateToggle({Name = "落下防止", CurrentValue = true, Callback = function(V) state.anti_fall = V end})

CombatTab:CreateSlider({Name = "ggrks", Range = {1, 50}, Increment = 1, CurrentValue = 5, Callback = function(V) state.hitbox_size = V end})
CombatTab:CreateButton({
    Name = "ggrks",
    Callback = function()
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= lp and p.Character and p.Character:FindFirstChild("Head") then
                local h = p.Character.Head
                h.Size = Vector3.new(state.hitbox_size, state.hitbox_size, state.hitbox_size)
                h.Transparency = state.hitbox_trans
                h.BrickColor = BrickColor.new("Really red")
                h.CanCollide = false
            end
        end
    end
})

-- --- 自動回収 ---
AutoTab:CreateToggle({Name = "フルーツ自動回収", CurrentValue = false, Callback = function(V) state.auto_fruit = V end})
AutoTab:CreateToggle({Name = "宝箱自動回収", CurrentValue = false, Callback = function(V) state.auto_chest = V end})

-- --- 視覚設定 ---
VisualTab:CreateToggle({Name = "プレイヤーESP", CurrentValue = false, Callback = function(V) 
    state.esp = V 
    for _, o in pairs(state.esp_objects) do if o[1] then o[1].Enabled = V end if o[2] then o[2].Enabled = V end end
end})

-- === メインロジック (v6.3 継承・滑らかな移動) ===

RunService.Heartbeat:Connect(function(dt)
    local char = lp.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    if not hrp or not hum then return end

    -- 落下防止
    if state.anti_fall and hrp.Position.Y < state.fall_threshold then
        hrp.Velocity = Vector3.new(0, 50, 0)
    end

    -- 滑らかな高速追尾
    if state.follow and state.selected_target and state.selected_target.Character then
        local tHrp = state.selected_target.Character:FindFirstChild("HumanoidRootPart")
        if tHrp then
            local targetPos = tHrp.Position + (tHrp.Velocity * state.predict_k) + Vector3.new(0, state.follow_height, 0)
            local diff = (targetPos - hrp.Position)
            local dist = diff.Magnitude
            
            if dist > 0.1 then
                local moveAmt = math.min(dist, state.follow_speed * dt)
                hrp.CFrame = CFrame.new(hrp.Position + (diff.Unit * moveAmt), tHrp.Position)
            end
            hrp.Velocity = Vector3.zero
            return 
        end
    end

    -- 自操作カスタム速度 (TP-Speed方式)
    if state.tp_speed_enabled and hum.MoveDirection.Magnitude > 0 then
        hrp.CFrame = hrp.CFrame + (hum.MoveDirection * (state.walkspeed * dt))
    end

    -- 自動回収 (TP)
    if state.auto_fruit or state.auto_chest then
        local target = nil
        if state.auto_fruit then
            for _, v in ipairs(game.Workspace:GetChildren()) do
                if v:IsA("Tool") or (v:IsA("Model") and string.find(v.Name, "Fruit")) then
                    target = v:FindFirstChild("Handle") or v:FindFirstChildOfClass("BasePart")
                    if target then break end
                end
            end
        end
        if not target and state.auto_chest then
            for _, v in ipairs(game.Workspace:GetDescendants()) do
                if string.find(v.Name, "Chest") and v:IsA("TouchTransmitter") then
                    target = v.Parent
                    if target then break end
                end
            end
        end
        if target then hrp.CFrame = CFrame.new(target.Position) end
    end
end)

-- NoClip
RunService.Stepped:Connect(function()
    if (state.follow or state.tp_speed_enabled or state.auto_fruit or state.auto_chest) and lp.Character then
        for _, v in ipairs(lp.Character:GetDescendants()) do
            if v:IsA("BasePart") then v.CanCollide = false end
        end
    end
end)

-- ESP
local function addESP(p)
    p.CharacterAdded:Connect(function(c)
        task.wait(1)
        local h = c:WaitForChild("Head")
        local hl = Instance.new("Highlight", c)
        hl.FillColor = Color3.fromRGB(255, 50, 50)
        hl.Enabled = state.esp
        local b = Instance.new("BillboardGui", h)
        b.Size = UDim2.new(0, 200, 0, 50)
        b.AlwaysOnTop = true
        b.Enabled = state.esp
        local l = Instance.new("TextLabel", b)
        l.Size = UDim2.new(1,0,1,0)
        l.BackgroundTransparency = 1
        l.TextColor3 = Color3.new(1,1,1)
        l.Text = p.DisplayName
        state.esp_objects[p] = {hl, b}
    end)
end

for _, p in ipairs(Players:GetPlayers()) do if p ~= lp then addESP(p) end end
Players.PlayerAdded:Connect(addESP)

Rayfield:Notify({Title = "Strawberry v6.4", Content = "安定化したプレイヤーリストを読み込みました", Duration = 3})

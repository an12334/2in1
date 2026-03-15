local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Stats = game:GetService("Stats")
local HttpService = game:GetService("HttpService")

-- Biến chính
local player = Players.LocalPlayer
if not player then repeat wait() until Players.LocalPlayer player = Players.LocalPlayer end
local playerName = player.Name
local scriptStartTime = tick()
local fps = 60
local ping = 0
local noteText = ""
local lastJobId = game.JobId
local webhookEnabled = false
local webhookUrl = ""
local guiVisible = false  -- ban đầu ẩn
local guiLocked = false
local dragging = false
local dragStart, frameStart

-- File lưu dữ liệu
local noteFileName = "delta_note_" .. playerName .. ".txt"
local webhookFileName = "delta_webhook.txt"

-- Đọc ghi chú cũ
if isfile and isfile(noteFileName) then
    noteText = readfile(noteFileName)
end

-- Đọc webhook cũ
if isfile and isfile(webhookFileName) then
    webhookUrl = readfile(webhookFileName)
end

-- Tạo GUI chính
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "DeltaGUI"
screenGui.ResetOnSpawn = false
screenGui.Parent = player:WaitForChild("PlayerGui")

-- Nút G nổi (luôn hiện)
local toggleBtn = Instance.new("TextButton")
toggleBtn.Name = "ToggleBtn"
toggleBtn.Size = UDim2.new(0, 40, 0, 40)  -- nhỏ hơn một chút
toggleBtn.Position = UDim2.new(1, -50, 1, -50)
toggleBtn.BackgroundColor3 = Color3.fromRGB(0, 170, 255)
toggleBtn.Text = "G"
toggleBtn.TextColor3 = Color3.new(1,1,1)
toggleBtn.Font = Enum.Font.GothamBold
toggleBtn.TextSize = 24
toggleBtn.Draggable = true
toggleBtn.Parent = screenGui

-- Khung chính của GUI (kích thước thu nhỏ: 250x300)
local mainFrame = Instance.new("Frame")
mainFrame.Name = "MainFrame"
mainFrame.Size = UDim2.new(0, 250, 0, 300)
mainFrame.Position = UDim2.new(1, -270, 1, -320)  -- dịch để vừa với kích thước mới
mainFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
mainFrame.BorderSizePixel = 2
mainFrame.BorderColor3 = Color3.fromRGB(0, 170, 255)
mainFrame.Active = true
mainFrame.Draggable = false
mainFrame.Visible = guiVisible  -- ban đầu false
mainFrame.Parent = screenGui

-- Kéo thả khung chính
mainFrame.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 and not guiLocked then
        dragging = true
        dragStart = input.Position
        frameStart = mainFrame.Position
    end
end)

mainFrame.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = false
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
        local delta = input.Position - dragStart
        mainFrame.Position = UDim2.new(frameStart.X.Scale, frameStart.X.Offset + delta.X, frameStart.Y.Scale, frameStart.Y.Offset + delta.Y)
    end
end)

-- Tiêu đề (thu nhỏ)
local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, 0, 0, 25)
title.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
title.Text = "Delta"
title.TextColor3 = Color3.fromRGB(0, 170, 255)
title.Font = Enum.Font.GothamBold
title.TextSize = 15
title.Parent = mainFrame

-- Nút chuyển tab (thu nhỏ)
local tab1 = Instance.new("TextButton")
tab1.Size = UDim2.new(0.5, -1, 0, 25)
tab1.Position = UDim2.new(0, 0, 0, 25)
tab1.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
tab1.Text = "Status"
tab1.TextColor3 = Color3.new(1,1,1)
tab1.Font = Enum.Font.Gotham
tab1.TextSize = 13
tab1.Parent = mainFrame

local tab2 = Instance.new("TextButton")
tab2.Size = UDim2.new(0.5, -1, 0, 25)
tab2.Position = UDim2.new(0.5, 1, 0, 25)
tab2.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
tab2.Text = "Webhook"
tab2.TextColor3 = Color3.new(1,1,1)
tab2.Font = Enum.Font.Gotham
tab2.TextSize = 13
tab2.Parent = mainFrame

-- Khung chứa nội dung các tab (phần còn lại)
local statusFrame = Instance.new("Frame")
statusFrame.Size = UDim2.new(1, -10, 1, -55)
statusFrame.Position = UDim2.new(0, 5, 0, 53)
statusFrame.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
statusFrame.BorderSizePixel = 0
statusFrame.Visible = true
statusFrame.Parent = mainFrame

local webhookFrame = Instance.new("Frame")
webhookFrame.Size = UDim2.new(1, -10, 1, -55)
webhookFrame.Position = UDim2.new(0, 5, 0, 53)
webhookFrame.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
webhookFrame.BorderSizePixel = 0
webhookFrame.Visible = false
webhookFrame.Parent = mainFrame

-- Chuyển tab
tab1.MouseButton1Click:Connect(function()
    statusFrame.Visible = true
    webhookFrame.Visible = false
    tab1.BackgroundColor3 = Color3.fromRGB(50,50,50)
    tab2.BackgroundColor3 = Color3.fromRGB(40,40,40)
end)

tab2.MouseButton1Click:Connect(function()
    statusFrame.Visible = false
    webhookFrame.Visible = true
    tab1.BackgroundColor3 = Color3.fromRGB(40,40,40)
    tab2.BackgroundColor3 = Color3.fromRGB(50,50,50)
end)

-- Hàm tạo dòng thông tin (thu nhỏ font, khoảng cách)
local function createRow(parent, y, label, initialValue)
    local bg = Instance.new("Frame")
    bg.Size = UDim2.new(1, -10, 0, 20)
    bg.Position = UDim2.new(0, 5, 0, y)
    bg.BackgroundColor3 = Color3.fromRGB(40,40,40)
    bg.BorderSizePixel = 0
    bg.Parent = parent
    
    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(0.5, -5, 1, 0)
    lbl.Position = UDim2.new(0, 5, 0, 0)
    lbl.BackgroundTransparency = 1
    lbl.Text = label .. ":"
    lbl.TextColor3 = Color3.fromRGB(200,200,200)
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Font = Enum.Font.Gotham
    lbl.TextSize = 12
    lbl.Parent = bg
    
    local val = Instance.new("TextLabel")
    val.Size = UDim2.new(0.5, -5, 1, 0)
    val.Position = UDim2.new(0.5, 0, 0, 0)
    val.BackgroundTransparency = 1
    val.Text = initialValue
    val.TextColor3 = Color3.new(1,1,1)
    val.TextXAlignment = Enum.TextXAlignment.Right
    val.Font = Enum.Font.Gotham
    val.TextSize = 12
    val.Parent = bg
    
    return val
end

-- === TAB STATUS ===
local yPos = 2  -- bắt đầu sát mép trên
local nameVal = createRow(statusFrame, yPos, "Tên", playerName); yPos = yPos + 22
local uptimeVal = createRow(statusFrame, yPos, "TG hoạt động", "00:00:00"); yPos = yPos + 22
local fpsVal = createRow(statusFrame, yPos, "FPS", "60"); yPos = yPos + 22
local pingVal = createRow(statusFrame, yPos, "Độ trễ (ms)", "0"); yPos = yPos + 25

-- Khu vực ghi chú (thu nhỏ)
local notesBg = Instance.new("Frame")
notesBg.Size = UDim2.new(1, -10, 0, 80)
notesBg.Position = UDim2.new(0, 5, 0, yPos)
notesBg.BackgroundColor3 = Color3.fromRGB(40,40,40)
notesBg.BorderSizePixel = 0
notesBg.Parent = statusFrame

local notesLbl = Instance.new("TextLabel")
notesLbl.Size = UDim2.new(1, -10, 0, 18)
notesLbl.Position = UDim2.new(0, 5, 0, 3)
notesLbl.BackgroundTransparency = 1
notesLbl.Text = "Ghi chú:"
notesLbl.TextColor3 = Color3.fromRGB(200,200,200)
notesLbl.TextXAlignment = Enum.TextXAlignment.Left
notesLbl.Font = Enum.Font.Gotham
notesLbl.TextSize = 12
notesLbl.Parent = notesBg

local notesBox = Instance.new("TextBox")
notesBox.Size = UDim2.new(1, -10, 0, 30)
notesBox.Position = UDim2.new(0, 5, 0, 23)
notesBox.BackgroundColor3 = Color3.fromRGB(20,20,20)
notesBox.TextColor3 = Color3.new(1,1,1)
notesBox.PlaceholderText = "Nhập ghi chú..."
notesBox.Text = noteText
notesBox.TextWrapped = true
notesBox.Font = Enum.Font.Gotham
notesBox.TextSize = 12
notesBox.Parent = notesBg

local saveNote = Instance.new("TextButton")
saveNote.Size = UDim2.new(0.5, -5, 0, 20)
saveNote.Position = UDim2.new(0, 5, 0, 58)
saveNote.BackgroundColor3 = Color3.fromRGB(0,170,255)
saveNote.Text = "Lưu"
saveNote.TextColor3 = Color3.new(1,1,1)
saveNote.Font = Enum.Font.Gotham
saveNote.TextSize = 12
saveNote.Parent = notesBg

local resetNote = Instance.new("TextButton")
resetNote.Size = UDim2.new(0.5, -5, 0, 20)
resetNote.Position = UDim2.new(0.5, 0, 0, 58)
resetNote.BackgroundColor3 = Color3.fromRGB(200,0,0)
resetNote.Text = "Đặt lại"
resetNote.TextColor3 = Color3.new(1,1,1)
resetNote.Font = Enum.Font.Gotham
resetNote.TextSize = 12
resetNote.Parent = notesBg

yPos = yPos + 90

-- Nút khoá di chuyển GUI (thu nhỏ)
local lockBtn = Instance.new("TextButton")
lockBtn.Size = UDim2.new(1, -10, 0, 25)
lockBtn.Position = UDim2.new(0, 5, 0, yPos)
lockBtn.BackgroundColor3 = Color3.fromRGB(80,80,80)
lockBtn.Text = "Khoá: OFF"
lockBtn.TextColor3 = Color3.new(1,1,1)
lockBtn.Font = Enum.Font.Gotham
lockBtn.TextSize = 13
lockBtn.Parent = statusFrame

-- === TAB WEBHOOK ===
yPos = 2

-- Nhập URL
local urlBg = Instance.new("Frame")
urlBg.Size = UDim2.new(1, -10, 0, 70)
urlBg.Position = UDim2.new(0, 5, 0, yPos)
urlBg.BackgroundColor3 = Color3.fromRGB(40,40,40)
urlBg.BorderSizePixel = 0
urlBg.Parent = webhookFrame

local urlLbl = Instance.new("TextLabel")
urlLbl.Size = UDim2.new(1, -10, 0, 18)
urlLbl.Position = UDim2.new(0, 5, 0, 3)
urlLbl.BackgroundTransparency = 1
urlLbl.Text = "Webhook URL:"
urlLbl.TextColor3 = Color3.fromRGB(200,200,200)
urlLbl.TextXAlignment = Enum.TextXAlignment.Left
urlLbl.Font = Enum.Font.Gotham
urlLbl.TextSize = 12
urlLbl.Parent = urlBg

local urlBox = Instance.new("TextBox")
urlBox.Size = UDim2.new(1, -10, 0, 25)
urlBox.Position = UDim2.new(0, 5, 0, 23)
urlBox.BackgroundColor3 = Color3.fromRGB(20,20,20)
urlBox.TextColor3 = Color3.new(1,1,1)
urlBox.PlaceholderText = "https://discord.com/api/webhooks/..."
urlBox.Text = webhookUrl
urlBox.Font = Enum.Font.Gotham
urlBox.TextSize = 12
urlBox.Parent = urlBg

local saveUrl = Instance.new("TextButton")
saveUrl.Size = UDim2.new(1, -10, 0, 20)
saveUrl.Position = UDim2.new(0, 5, 0, 50)
saveUrl.BackgroundColor3 = Color3.fromRGB(0,170,255)
saveUrl.Text = "Lưu URL"
saveUrl.TextColor3 = Color3.new(1,1,1)
saveUrl.Font = Enum.Font.Gotham
saveUrl.TextSize = 12
saveUrl.Parent = urlBg

yPos = yPos + 80

-- Trạng thái bật/tắt
local toggleBg = Instance.new("Frame")
toggleBg.Size = UDim2.new(1, -10, 0, 35)
toggleBg.Position = UDim2.new(0, 5, 0, yPos)
toggleBg.BackgroundColor3 = Color3.fromRGB(40,40,40)
toggleBg.BorderSizePixel = 0
toggleBg.Parent = webhookFrame

local toggleLbl = Instance.new("TextLabel")
toggleLbl.Size = UDim2.new(0.5, -5, 1, 0)
toggleLbl.Position = UDim2.new(0, 5, 0, 0)
toggleLbl.BackgroundTransparency = 1
toggleLbl.Text = "Trạng thái:"
toggleLbl.TextColor3 = Color3.fromRGB(200,200,200)
toggleLbl.TextXAlignment = Enum.TextXAlignment.Left
toggleLbl.Font = Enum.Font.Gotham
toggleLbl.TextSize = 12
toggleLbl.Parent = toggleBg

local toggleState = Instance.new("TextLabel")
toggleState.Size = UDim2.new(0.5, -5, 1, 0)
toggleState.Position = UDim2.new(0.5, 0, 0, 0)
toggleState.BackgroundTransparency = 1
toggleState.Text = "TẮT"
toggleState.TextColor3 = Color3.fromRGB(255,50,50)
toggleState.TextXAlignment = Enum.TextXAlignment.Right
toggleState.Font = Enum.Font.GothamBold
toggleState.TextSize = 12
toggleState.Parent = toggleBg

yPos = yPos + 45

-- Nút BẬT/TẮT
local enableBtn = Instance.new("TextButton")
enableBtn.Size = UDim2.new(0.5, -5, 0, 25)
enableBtn.Position = UDim2.new(0, 5, 0, yPos)
enableBtn.BackgroundColor3 = Color3.fromRGB(0,200,0)
enableBtn.Text = "BẬT"
enableBtn.TextColor3 = Color3.new(1,1,1)
enableBtn.Font = Enum.Font.Gotham
enableBtn.TextSize = 13
enableBtn.Parent = webhookFrame

local disableBtn = Instance.new("TextButton")
disableBtn.Size = UDim2.new(0.5, -5, 0, 25)
disableBtn.Position = UDim2.new(0.5, 0, 0, yPos)
disableBtn.BackgroundColor3 = Color3.fromRGB(200,0,0)
disableBtn.Text = "TẮT"
disableBtn.TextColor3 = Color3.new(1,1,1)
disableBtn.Font = Enum.Font.Gotham
disableBtn.TextSize = 13
disableBtn.Parent = webhookFrame

-- === XỬ LÝ CHỨC NĂNG ===

-- Lưu ghi chú
saveNote.MouseButton1Click:Connect(function()
    noteText = notesBox.Text
    if isfile then writefile(noteFileName, noteText) end
end)

-- Đặt lại ghi chú
resetNote.MouseButton1Click:Connect(function()
    notesBox.Text = ""
    noteText = ""
    if isfile and isfile(noteFileName) then delfile(noteFileName) end
end)

-- Khoá / mở khoá di chuyển
lockBtn.MouseButton1Click:Connect(function()
    guiLocked = not guiLocked
    lockBtn.Text = guiLocked and "Khoá: ON" or "Khoá: OFF"
    lockBtn.BackgroundColor3 = guiLocked and Color3.fromRGB(0,150,0) or Color3.fromRGB(80,80,80)
end)

-- Lưu URL webhook
saveUrl.MouseButton1Click:Connect(function()
    webhookUrl = urlBox.Text
    if isfile then writefile(webhookFileName, webhookUrl) end
end)

-- BẬT webhook
enableBtn.MouseButton1Click:Connect(function()
    if webhookUrl == "" or not webhookUrl:match("^https://discord.com/api/webhooks/") then
        warn("Invalid webhook URL")
        return
    end
    webhookEnabled = true
    toggleState.Text = "BẬT"
    toggleState.TextColor3 = Color3.fromRGB(0,255,0)
    startWebhookLoop()
end)

-- TẮT webhook
disableBtn.MouseButton1Click:Connect(function()
    webhookEnabled = false
    toggleState.Text = "TẮT"
    toggleState.TextColor3 = Color3.fromRGB(255,50,50)
    if sendLoop then
        coroutine.close(sendLoop)
        sendLoop = nil
    end
end)

-- Nút G: hiện/ẩn khung chính
toggleBtn.MouseButton1Click:Connect(function()
    guiVisible = not guiVisible
    mainFrame.Visible = guiVisible
end)

-- Cập nhật thời gian hoạt động và ping mỗi giây
spawn(function()
    while true do
        local t = tick() - scriptStartTime
        local h = math.floor(t/3600)
        local m = math.floor((t%3600)/60)
        local s = math.floor(t%60)
        uptimeVal.Text = string.format("%02d:%02d:%02d", h, m, s)
        
        ping = Stats.Network.ServerStatsItem:GetValue("DataPing") / 1000
        pingVal.Text = math.floor(ping * 1000)
        
        wait(1)
    end
end)

-- Cập nhật FPS mỗi frame
RunService.RenderStepped:Connect(function()
    fps = 1 / RunService.RenderStepped:Wait()
    fpsVal.Text = math.floor(fps)
end)

-- === HÀM GỬI WEBHOOK ===
function getAccountInfo()
    local info = {}
    info["Tên tài khoản"] = playerName
    info["Level"] = player:FindFirstChild("Data") and player.Data:FindFirstChild("Level") and player.Data.Level.Value or "N/A"
    info["Beli"] = player:FindFirstChild("Data") and player.Data:FindFirstChild("Beli") and player.Data.Beli.Value or "N/A"
    
    -- Trái ác quỷ
    local fruit = "Không"
    if player:FindFirstChild("Data") and player.Data:FindFirstChild("Fruit") then
        fruit = player.Data.Fruit.Value
    else
        for _, v in ipairs(player.Backpack:GetChildren()) do
            if v:IsA("Tool") and v:FindFirstChild("Fruit") then
                fruit = v.Name
                break
            end
        end
    end
    info["Trái ác quỷ"] = fruit
    
    -- Kiếm
    local sword = "Không"
    for _, v in ipairs(player.Backpack:GetChildren()) do
        if v:IsA("Tool") and v:FindFirstChild("Sword") then
            sword = v.Name
            break
        end
    end
    if sword == "Không" then
        for _, v in ipairs(player.Character:GetChildren()) do
            if v:IsA("Tool") and v:FindFirstChild("Sword") then
                sword = v.Name
                break
            end
        end
    end
    info["Kiếm"] = sword
    
    -- Kho đồ (tất cả tool)
    local items = {}
    for _, v in ipairs(player.Backpack:GetChildren()) do
        if v:IsA("Tool") then table.insert(items, v.Name) end
    end
    for _, v in ipairs(player.Character:GetChildren()) do
        if v:IsA("Tool") then table.insert(items, v.Name) end
    end
    info["Kho đồ"] = #items > 0 and table.concat(items, ", ") or "Trống"
    
    info["Thời gian"] = os.date("%H:%M:%S %d/%m/%Y")
    return info
end

function sendToWebhook()
    if not webhookEnabled or webhookUrl == "" then return end
    local info = getAccountInfo()
    local embed = {
        title = "Thông tin tài khoản Blox Fruit",
        color = 3447003,
        fields = {},
        footer = {text = "Được làm bởi tungtung_shaur and deepseek"},
        timestamp = os.date("!%Y-%m-%dT%H:%M:%SZ")
    }
    for k,v in pairs(info) do
        table.insert(embed.fields, {name=k, value=tostring(v), inline=false})
    end
    local data = {embeds={embed}}
    local json = HttpService:JSONEncode(data)
    pcall(function()
        local request = syn and syn.request or http_request or request
        if request then
            request({Url=webhookUrl, Method="POST", Headers={["Content-Type"]="application/json"}, Body=json})
        else
            HttpService:PostAsync(webhookUrl, json)
        end
    end)
end

function startWebhookLoop()
    if sendLoop then coroutine.close(sendLoop) end
    sendLoop = coroutine.create(function()
        while webhookEnabled do
            sendToWebhook()
            for i=1,30 do
                if not webhookEnabled then break end
                wait(1)
            end
        end
    end)
    coroutine.resume(sendLoop)
end

-- Phát hiện đổi server
spawn(function()
    while wait(5) do
        if game.JobId ~= lastJobId then
            lastJobId = game.JobId
            if webhookEnabled then sendToWebhook() end
        end
    end
end)

-- Khởi động webhook nếu đã bật trước đó
if webhookEnabled then startWebhookLoop() end

print("Delta Blox Fruit Manager loaded for " .. playerName .. " (GUI ẩn, nhấn G để hiện)")

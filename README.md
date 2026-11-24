-- ============================================================
-- Futuristic Orion UI (Delta クライアントサイド用)
-- 「Deltaに貼り付けて実行」してください
-- ============================================================

local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")
local LocalPlayer = game:GetService("Players").LocalPlayer
local Mouse = LocalPlayer:GetMouse()
local HttpService = game:GetService("HttpService")

local OrionLib = {
    Elements = {},
    ThemeObjects = {},
    Connections = {},
    Flags = {},
    -- ★ 未来風テーマ：ネオンシアン＋ディープブルー＋パープル
    Themes = {
        Default = {
            Main = Color3.fromRGB(10, 14, 24),        -- 深い宇宙色
            Second = Color3.fromRGB(18, 24, 40),      -- パネル色
            Stroke = Color3.fromRGB(0, 220, 255),     -- ネオンシアン
            Divider = Color3.fromRGB(0, 180, 220),
            Text = Color3.fromRGB(220, 245, 255),     -- アイスホワイト
            TextDark = Color3.fromRGB(120, 180, 210),
            Accent = Color3.fromRGB(180, 80, 255),    -- ネオンパープル
            Glow = Color3.fromRGB(0, 255, 230)        -- グロー用
        }
    },
    SelectedTheme = "Default",
    Folder = nil,
    SaveCfg = false
}

-- Feather Icons 読み込み（そのまま流用）
local Icons = {}
local Success, Response = pcall(function()
    Icons = HttpService:JSONDecode(
        game:HttpGetAsync("https://raw.githubusercontent.com/evoincorp/lucideblox/master/src/modules/util/icons.json")
    ).icons
end)
if not Success then
    warn("\nFuturistic Orion - Failed to load Feather Icons: " .. tostring(Response) .. "\n")
end

local function GetIcon(IconName)
    return Icons[IconName] or nil
end

-- ★ ScreenGui にグローバルなネオングロー背景を追加
local Orion = Instance.new("ScreenGui")
Orion.Name = "FuturisticOrion"
Orion.IgnoreGuiInset = true
if syn and syn.protect_gui then
    syn.protect_gui(Orion)
    Orion.Parent = game.CoreGui
else
    Orion.Parent = gethui() or game.CoreGui
end

-- 既存UIの掃除
local parentGui = gethui and gethui() or game:GetService("CoreGui")
for _, Interface in ipairs(parentGui:GetChildren()) do
    if Interface.Name == Orion.Name and Interface ~= Orion then
        Interface:Destroy()
    end
end

function OrionLib:IsRunning()
    return Orion.Parent == (gethui and gethui() or game:GetService("CoreGui"))
end

local function AddConnection(Signal, Function)
    if not OrionLib:IsRunning() then return end
    local c = Signal:Connect(Function)
    table.insert(OrionLib.Connections, c)
    return c
end

task.spawn(function()
    while OrionLib:IsRunning() do task.wait() end
    for _, c in next, OrionLib.Connections do c:Disconnect() end
end)

-- ドラッグ機能（そのまま）
local function AddDraggingFunctionality(DragPoint, Main)
    pcall(function()
        local Dragging, DragInput, MousePos, FramePos = false
        DragPoint.InputBegan:Connect(function(Input)
            if Input.UserInputType == Enum.UserInputType.MouseButton1 or Input.UserInputType == Enum.UserInputType.Touch then
                Dragging = true
                MousePos = Input.Position
                FramePos = Main.Position
                Input.Changed:Connect(function()
                    if Input.UserInputState == Enum.UserInputState.End then
                        Dragging = false
                    end
                end)
            end
        end)
        DragPoint.InputChanged:Connect(function(Input)
            if Input.UserInputType == Enum.UserInputType.MouseMovement or Input.UserInputType == Enum.UserInputType.Touch then
                DragInput = Input
            end
        end)
        UserInputService.InputChanged:Connect(function(Input)
            if Input == DragInput and Dragging then
                local Delta = Input.Position - MousePos
                TweenService:Create(Main,
                    TweenInfo.new(0.35, Enum.EasingStyle.Quint, Enum.EasingDirection.Out),
                    {Position = UDim2.new(FramePos.X.Scale, FramePos.X.Offset + Delta.X, FramePos.Y.Scale, FramePos.Y.Offset + Delta.Y)}
                ):Play()
            end
        end)
    end)
end

local function Create(Name, Properties, Children)
    local Object = Instance.new(Name)
    for i, v in next, Properties or {} do Object[i] = v end
    for i, v in next, Children or {} do v.Parent = Object end
    return Object
end

local function CreateElement(ElementName, ElementFunction)
    OrionLib.Elements[ElementName] = function(...) return ElementFunction(...) end
end

local function MakeElement(ElementName, ...)
    return OrionLib.Elements[ElementName](...)
end

local function SetProps(Element, Props)
    table.foreach(Props, function(P, V) Element[P] = V end)
    return Element
end

local function SetChildren(Element, Children)
    table.foreach(Children, function(_, C) C.Parent = Element end)
    return Element
end

local function Round(Number, Factor)
    local Result = math.floor(Number / Factor + (math.sign(Number) * 0.5)) * Factor
    if Result < 0 then Result = Result + Factor end
    return Result
end

local function ReturnProperty(Object)
    if Object:IsA("Frame") or Object:IsA("TextButton") then return "BackgroundColor3" end
    if Object:IsA("ScrollingFrame") then return "ScrollBarImageColor3" end
    if Object:IsA("UIStroke") then return "Color" end
    if Object:IsA("TextLabel") or Object:IsA("TextBox") then return "TextColor3" end
    if Object:IsA("ImageLabel") or Object:IsA("ImageButton") then return "ImageColor3" end
end

local function AddThemeObject(Object, Type)
    if not OrionLib.ThemeObjects[Type] then OrionLib.ThemeObjects[Type] = {} end
    table.insert(OrionLib.ThemeObjects[Type], Object)
    Object[ReturnProperty(Object)] = OrionLib.Themes[OrionLib.SelectedTheme][Type]
    return Object
end

local function SetTheme()
    for Name, Type in pairs(OrionLib.ThemeObjects) do
        for _, Object in pairs(Type) do
            Object[ReturnProperty(Object)] = OrionLib.Themes[OrionLib.SelectedTheme][Name]
        end
    end
end

-- 設定保存系（そのまま）
local function PackColor(Color) return {R = Color.R * 255, G = Color.G * 255, B = Color.B * 255} end
local function UnpackColor(Color) return Color3.fromRGB(Color.R, Color.G, Color.B) end

local function LoadCfg(Config)
    local Data = HttpService:JSONDecode(Config)
    table.foreach(Data, function(a, b)
        if OrionLib.Flags[a] then
            spawn(function()
                if OrionLib.Flags[a].Type == "Colorpicker" then
                    OrionLib.Flags[a]:Set(UnpackColor(b))
                else
                    OrionLib.Flags[a]:Set(b)
                end
            end)
        end
    end)
end

local function SaveCfg(Name)
    local Data = {}
    for i, v in pairs(OrionLib.Flags) do
        if v.Save then
            if v.Type == "Colorpicker" then Data[i] = PackColor(v.Value) else Data[i] = v.Value end
        end
    end
    if writefile and OrionLib.Folder then
        writefile(OrionLib.Folder .. "/" .. Name .. ".txt", tostring(HttpService:JSONEncode(Data)))
    end
end

-- ★ 未来風：ネオンUIStrokeを生成するヘルパー
local function AddNeonStroke(parent, color, thickness)
    local stroke = Instance.new("UIStroke")
    stroke.Color = color or Color3.fromRGB(0, 220, 255)
    stroke.Thickness = thickness or 1.5
    stroke.Transparency = 0.15
    stroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    stroke.Parent = parent
    return stroke
end

-- ★ 未来風：グラデーション背景を追加
local function AddGradient(parent, c1, c2)
    local grad = Instance.new("UIGradient")
    grad.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, c1 or Color3.fromRGB(20, 30, 60)),
        ColorSequenceKeypoint.new(1, c2 or Color3.fromRGB(8, 10, 20))
    })
    grad.Rotation = 135
    grad.Parent = parent
    return grad
end

-- ★ 未来風：パルス（脈打つ）グローを生成
local function AddPulseGlow(obj, baseColor, speed)
    speed = speed or 1.5
    spawn(function()
        while obj.Parent do
            TweenService:Create(obj, TweenInfo.new(speed, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {
                Transparency = 0.6
            }):Play()
            task.wait(speed)
            if not obj.Parent then break end
            TweenService:Create(obj, TweenInfo.new(speed, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {
                Transparency = 0.1
            }):Play()
            task.wait(speed)
        end
    end)
end

-- 基本要素（Corner/Stroke/List/Padding/TFrame/Frame...）
CreateElement("Corner", function(Scale, Offset)
    return Create("UICorner", {CornerRadius = UDim.new(Scale or 0, Offset or 10)})
end)

CreateElement("Stroke", function(Color, Thickness)
    return Create("UIStroke", {
        Color = Color or Color3.fromRGB(0, 220, 255),
        Thickness = Thickness or 1.5,
        Transparency = 0.2
    })
end)

CreateElement("List", function(Scale, Offset)
    return Create("UIListLayout", {
        SortOrder = Enum.SortOrder.LayoutOrder,
        Padding = UDim.new(Scale or 0, Offset or 0)
    })
end)

CreateElement("Padding", function(Bottom, Left, Right, Top)
    return Create("UIPadding", {
        PaddingBottom = UDim.new(0, Bottom or 4),
        PaddingLeft = UDim.new(0, Left or 4),
        PaddingRight = UDim.new(0, Right or 4),
        PaddingTop = UDim.new(0, Top or 4)
    })
end)

CreateElement("TFrame", function()
    return Create("Frame", {BackgroundTransparency = 1})
end)

CreateElement("Frame", function(Color)
    return Create("Frame", {BackgroundColor3 = Color or Color3.fromRGB(255, 255, 255), BorderSizePixel = 0})
end)

CreateElement("RoundFrame", function(Color, Scale, Offset)
    return Create("Frame",
        {BackgroundColor3 = Color or Color3.fromRGB(255, 255, 255), BorderSizePixel = 0},
        {Create("UICorner", {CornerRadius = UDim.new(Scale, Offset)})}
    )
end)

CreateElement("Button", function()
    return Create("TextButton", {
        Text = "", AutoButtonColor = false, BackgroundTransparency = 1, BorderSizePixel = 0
    })
end)

CreateElement("ScrollFrame", function(Color, Width)
    return Create("ScrollingFrame", {
        BackgroundTransparency = 1,
        MidImage = "rbxassetid://7445543667",
        BottomImage = "rbxassetid://7445543667",
        TopImage = "rbxassetid://7445543667",
        ScrollBarImageColor3 = Color or Color3.fromRGB(0, 220, 255),
        BorderSizePixel = 0,
        ScrollBarThickness = Width,
        CanvasSize = UDim2.new(0, 0, 0, 0)
    })
end)

CreateElement("Image", function(ImageID)
    local img = Create("ImageLabel", {Image = ImageID, BackgroundTransparency = 1})
    if GetIcon(ImageID) then img.Image = GetIcon(ImageID) end
    return img
end)

CreateElement("ImageButton", function(ImageID)
    return Create("ImageButton", {Image = ImageID, BackgroundTransparency = 1})
end)

CreateElement("Label", function(Text, TextSize, Transparency)
    return Create("TextLabel", {
        Text = Text or "", TextColor3 = Color3.fromRGB(220, 245, 255),
        TextTransparency = Transparency or 0, TextSize = TextSize or 15,
        Font = Enum.Font.Gotham, RichText = true, BackgroundTransparency = 1,
        TextXAlignment = Enum.TextXAlignment.Left
    })
end)

-- 通知ホルダー
local NotificationHolder = SetProps(SetChildren(MakeElement("TFrame"), {
    SetProps(MakeElement("List"), {
        HorizontalAlignment = Enum.HorizontalAlignment.Center,
        SortOrder = Enum.SortOrder.LayoutOrder,
        VerticalAlignment = Enum.VerticalAlignment.Bottom,
        Padding = UDim.new(0, 5)
    })
}), {
    Position = UDim2.new(1, -25, 1, -25),
    Size = UDim2.new(0, 300, 1, -25),
    AnchorPoint = Vector2.new(1, 1),
    Parent = Orion
})

-- ★ 未来風通知：グラデ＋ネオン枠
function OrionLib:MakeNotification(NotificationConfig)
    spawn(function()
        NotificationConfig.Name = NotificationConfig.Name or "Notification"
        NotificationConfig.Content = NotificationConfig.Content or "Test"
        NotificationConfig.Image = NotificationConfig.Image or "rbxassetid://4384403532"
        NotificationConfig.Time = NotificationConfig.Time or 15

        local NotificationParent = SetProps(MakeElement("TFrame"), {
            Size = UDim2.new(1, 0, 0, 0), AutomaticSize = Enum.AutomaticSize.Y, Parent = NotificationHolder
        })

        local NotificationFrame = SetChildren(SetProps(
            MakeElement("RoundFrame", OrionLib.Themes.Default.Second, 0, 12),
            {Parent = NotificationParent, Size = UDim2.new(1, 0, 0, 0),
             Position = UDim2.new(1, -55, 0, 0), BackgroundTransparency = 0,
             AutomaticSize = Enum.AutomaticSize.Y}
        ), {
            AddNeonStroke(nil, Color3.fromRGB(0, 220, 255), 1.6), -- ネオン枠（UIStrokeを後で親付け）
            AddGradient(nil, Color3.fromRGB(18, 24, 40), Color3.fromRGB(8, 12, 24)), -- グラデ
            MakeElement("Padding", 12, 12, 12, 12),
            SetProps(MakeElement("Image", NotificationConfig.Image), {
                Size = UDim2.new(0, 20, 0, 20),
                ImageColor3 = Color3.fromRGB(0, 255, 230), -- ネオンカラー
                Name = "Icon"
            }),
            SetProps(MakeElement("Label", NotificationConfig.Name, 15), {
                Size = UDim2.new(1, -30, 0, 20), Position = UDim2.new(0, 30, 0, 0),
                Font = Enum.Font.GothamBold, Name = "Title",
                TextColor3 = Color3.fromRGB(220, 245, 255)
            }),
            SetProps(MakeElement("Label", NotificationConfig.Content, 14), {
                Size = UDim2.new(1, 0, 0, 0), Position = UDim2.new(0, 0, 0, 25),
                Font = Enum.Font.GothamSemibold, Name = "Content",
                AutomaticSize = Enum.AutomaticSize.Y, TextColor3 = Color3.fromRGB(160, 220, 245), TextWrapped = true
            })
        })

        -- UIStrokeとGradientを正しい親に付け替え
        local stroke = NotificationFrame:FindFirstChildOfClass("UIStroke")
        if stroke then stroke.Parent = NotificationFrame end
        local grad = NotificationFrame:FindFirstChildOfClass("UIGradient")
        if grad then grad.Parent = NotificationFrame end
        if stroke then AddPulseGlow(stroke, Color3.fromRGB(0, 220, 255), 1.2) end

        TweenService:Create(NotificationFrame, TweenInfo.new(0.5, Enum.EasingStyle.Quint),
            {Position = UDim2.new(0, 0, 0, 0)}):Play()
        task.wait(NotificationConfig.Time - 0.9)
        TweenService:Create(NotificationFrame, TweenInfo.new(0.6, Enum.EasingStyle.Quint),
            {BackgroundTransparency = 0.7}):Play()
        TweenService:Create(NotificationFrame.Title, TweenInfo.new(0.6, Enum.EasingStyle.Quint),
            {TextTransparency = 0.6}):Play()
        TweenService:Create(NotificationFrame.Content, TweenInfo.new(0.6, Enum.EasingStyle.Quint),
            {TextTransparency = 0.6}):Play()
        task.wait(0.6)
        NotificationFrame:TweenPosition(UDim2.new(1, 20, 0, 0), "In", "Quint", 0.8, true)
        task.wait(1.2)
        NotificationFrame:Destroy()
    end)
end

function OrionLib:Init()
    if OrionLib.SaveCfg then
        pcall(function()
            if isfile and OrionLib.Folder and isfile(OrionLib.Folder .. "/" .. game.GameId .. ".txt") then
                LoadCfg(readfile(OrionLib.Folder .. "/" .. game.GameId .. ".txt"))
                OrionLib:MakeNotification({
                    Name = "Configuration",
                    Content = "Auto-loaded configuration for " .. game.GameId,
                    Time = 5
                })
            end
        end)
    end
end

-- ★ 未来風メインウィンドウ
function OrionLib:MakeWindow(WindowConfig)
    local FirstTab = true
    local Minimized = false
    local Loaded = false
    local UIHidden = false

    WindowConfig = WindowConfig or {}
    WindowConfig.Name = WindowConfig.Name or "FUTURISTIC UI"
    WindowConfig.ConfigFolder = WindowConfig.ConfigFolder or WindowConfig.Name
    WindowConfig.SaveConfig = WindowConfig.SaveConfig or false
    WindowConfig.HidePremium = WindowConfig.HidePremium or false
    if WindowConfig.IntroEnabled == nil then WindowConfig.IntroEnabled = true end
    WindowConfig.IntroText = WindowConfig.IntroText or "◈ FUTURE ◈"
    WindowConfig.CloseCallback = WindowConfig.CloseCallback or function() end
    WindowConfig.ShowIcon = WindowConfig.ShowIcon or false
    WindowConfig.Icon = WindowConfig.Icon or "rbxassetid://8834748103"
    WindowConfig.IntroIcon = WindowConfig.IntroIcon or "rbxassetid://8834748103"
    OrionLib.Folder = WindowConfig.ConfigFolder
    OrionLib.SaveCfg = WindowConfig.SaveConfig

    if WindowConfig.SaveConfig and makefolder and not isfolder(WindowConfig.ConfigFolder) then
        makefolder(WindowConfig.ConfigFolder)
    end

    -- ★ タブホルダー（未来風スクロール）
    local TabHolder = AddThemeObject(
        SetChildren(SetProps(MakeElement("ScrollFrame", Color3.fromRGB(0, 220, 255), 4), {
            Size = UDim2.new(1, 0, 1, -50)
        }), {
            MakeElement("List"),
            MakeElement("Padding", 8, 0, 0, 8)
        }), "Divider"
    )

    AddConnection(TabHolder.UIListLayout:GetPropertyChangedSignal("AbsoluteContentSize"), function()
        TabHolder.CanvasSize = UDim2.new(0, 0, 0, TabHolder.UIListLayout.AbsoluteContentSize.Y + 16)
    end)

    local CloseBtn = SetChildren(SetProps(MakeElement("Button"), {
        Size = UDim2.new(0.5, 0, 1, 0), Position = UDim2.new(0.5, 0, 0, 0), BackgroundTransparency = 1
    }), {
        AddThemeObject(SetProps(MakeElement("Image", "rbxassetid://7072725342"), {
            Position = UDim2.new(0, 9, 0, 6), Size = UDim2.new(0, 18, 0, 18)
        }), "Text")
    })

    local MinimizeBtn = SetChildren(SetProps(MakeElement("Button"), {
        Size = UDim2.new(0.5, 0, 1, 0), BackgroundTransparency = 1
    }), {
        AddThemeObject(SetProps(MakeElement("Image", "rbxassetid://7072719338"), {
            Position = UDim2.new(0, 9, 0, 6), Size = UDim2.new(0, 18, 0, 18), Name = "Ico"
        }), "Text")
    })

    local DragPoint = SetProps(MakeElement("TFrame"), {Size = UDim2.new(1, 0, 0, 50)})

    -- ★ 未来風サイドバー：グラデ＋ネオンボーダー
    local WindowStuff = AddThemeObject(
        SetChildren(SetProps(MakeElement("RoundFrame", Color3.fromRGB(255, 255, 255), 0, 12), {
            Size = UDim2.new(0, 150, 1, -50), Position = UDim2.new(0, 0, 0, 50)
        }), {
            -- サイドバー背景グラデ
            AddGradient(nil, Color3.fromRGB(18, 24, 40), Color3.fromRGB(8, 12, 24)),
            -- 右端のネオンライン
            AddThemeObject(SetProps(MakeElement("Frame"), {
                Size = UDim2.new(0, 2, 1, 0), Position = UDim2.new(1, -2, 0, 0),
                BackgroundColor3 = Color3.fromRGB(0, 220, 255)
            }), "Stroke"),
            TabHolder,
            -- ユーザー情報フッター
            SetChildren(SetProps(MakeElement("TFrame"), {
                Size = UDim2.new(1, 0, 0, 50), Position = UDim2.new(0, 0, 1, -50)
            }), {
                AddThemeObject(SetProps(MakeElement("Frame"), {
                    Size = UDim2.new(1, 0, 0, 1), BackgroundColor3 = Color3.fromRGB(0, 220, 255), BackgroundTransparency = 0.5
                }), "Stroke"),
                SetChildren(SetProps(MakeElement("Frame"), {
                    AnchorPoint = Vector2.new(0, 0.5), Size = UDim2.new(0, 32, 0, 32),
                    Position = UDim2.new(0, 10, 0.5, 0), BackgroundColor3 = Color3.fromRGB(18, 24, 40)
                }), {
                    SetProps(MakeElement("Image",
                        "https://www.roblox.com/headshot-thumbnail/image?userId=" .. LocalPlayer.UserId .. "&width=420&height=420&format=png"),
                        {Size = UDim2.new(1, 0, 1, 0)}),
                    MakeElement("Corner", 1),
                    AddNeonStroke(nil, Color3.fromRGB(0, 220, 255), 1.5)
                }),
                AddThemeObject(SetProps(MakeElement("Label", LocalPlayer.DisplayName, 14), {
                    Size = UDim2.new(1, -60, 0, 14), Position = UDim2.new(0, 50, 0, 12),
                    Font = Enum.Font.GothamBold, ClipsDescendants = true
                }), "Text"),
                AddThemeObject(SetProps(MakeElement("Label", "@" .. LocalPlayer.Name, 12), {
                    Size = UDim2.new(1, -60, 0, 12), Position = UDim2.new(0, 50, 1, -26)
                }), "TextDark")
            })
        }), "Second"
    )

    -- グラデをサイドバーに適用
    local sidebarGrad = WindowStuff:FindFirstChildOfClass("UIGradient")
    if sidebarGrad then sidebarGrad.Parent = WindowStuff end

    -- ウィンドウタイトル（未来風ネオン）
    local WindowName = AddThemeObject(SetProps(MakeElement("Label", WindowConfig.Name, 14), {
        Size = UDim2.new(1, -30, 2, 0), Position = UDim2.new(0, 25, 0, -24),
        Font = Enum.Font.GothamBlack, TextSize = 20,
        TextColor3 = Color3.fromRGB(0, 255, 230) -- ネオンシアン
    }), "Text")

    local WindowTopBarLine = AddThemeObject(SetProps(MakeElement("Frame"), {
        Size = UDim2.new(1, 0, 0, 1), Position = UDim2.new(0, 0, 1, -1),
        BackgroundColor3 = Color3.fromRGB(0, 220, 255), BackgroundTransparency = 0.4
    }), "Stroke")

    -- ★ メインウィンドウ：グラデ背景＋ネオン枠＋パルスグロー
    local MainWindow = AddThemeObject(
        SetChildren(SetProps(MakeElement("RoundFrame", Color3.fromRGB(255, 255, 255), 0, 14), {
            Parent = Orion, Position = UDim2.new(0.5, -307, 0.5, -172),
            Size = UDim2.new(0, 615, 0, 344), ClipsDescendants = true
        }), {
            -- メイン背景グラデ
            AddGradient(nil, Color3.fromRGB(18, 24, 40), Color3.fromRGB(6, 8, 16)),
            -- ネオン枠（後で親付け）
            AddNeonStroke(nil, Color3.fromRGB(0, 220, 255), 2),
            SetChildren(SetProps(MakeElement("TFrame"), {
                Size = UDim2.new(1, 0, 0, 50), Name = "TopBar"
            }), {
                WindowName, WindowTopBarLine,
                AddThemeObject(SetChildren(SetProps(MakeElement("RoundFrame", Color3.fromRGB(255, 255, 255), 0, 8), {
                    Size = UDim2.new(0, 70, 0, 30), Position = UDim2.new(1, -90, 0, 10)
                }), {
                    AddNeonStroke(nil, Color3.fromRGB(0, 220, 255), 1.4),
                    AddThemeObject(SetProps(MakeElement("Frame"), {
                        Size = UDim2.new(0, 1, 1, 0), Position = UDim2.new(0.5, 0, 0, 0)
                    }), "Stroke"),
                    CloseBtn, MinimizeBtn
                }), "Second")
            }),
            DragPoint, WindowStuff
        }), "Main"
    )

    -- グラデとストロークを正しい親に付け替え
    local mainGrad = MainWindow:FindFirstChildOfClass("UIGradient")
    if mainGrad then mainGrad.Parent = MainWindow end
    local mainStroke = MainWindow:FindFirstChildOfClass("UIStroke")
    if mainStroke then mainStroke.Parent = MainWindow; AddPulseGlow(mainStroke, nil, 1.5) end

    if WindowConfig.ShowIcon then
        WindowName.Position = UDim2.new(0, 50, 0, -24)
        local WindowIcon = SetProps(MakeElement("Image", WindowConfig.Icon), {
            Size = UDim2.new(0, 20, 0, 20), Position = UDim2.new(0, 25, 0, 15),
            ImageColor3 = Color3.fromRGB(0, 255, 230)
        })
        WindowIcon.Parent = MainWindow.TopBar
    end

    AddDraggingFunctionality(DragPoint, MainWindow)

    AddConnection(CloseBtn.MouseButton1Up, function()
        MainWindow.Visible = false
        UIHidden = true
        OrionLib:MakeNotification({Name = "Interface Hidden", Content = "Tap RightShift to reopen", Time = 5})
        WindowConfig.CloseCallback()
    end)

    AddConnection(UserInputService.InputBegan, function(Input)
        if Input.KeyCode == Enum.KeyCode.RightShift and UIHidden then
            MainWindow.Visible = true
        end
    end)

    AddConnection(MinimizeBtn.MouseButton1Up, function()
        if Minimized then
            TweenService:Create(MainWindow,
                TweenInfo.new(0.5, Enum.EasingStyle.Quint, Enum.EasingDirection.Out),
                {Size = UDim2.new(0, 615, 0, 344)}):Play()
            MinimizeBtn.Ico.Image = "rbxassetid://7072719338"
            task.wait(0.02)
            MainWindow.ClipsDescendants = false
            WindowStuff.Visible = true
            WindowTopBarLine.Visible = true
        else
            MainWindow.ClipsDescendants = true
            WindowTopBarLine.Visible = false
            MinimizeBtn.Ico.Image = "rbxassetid://7072720870"
            TweenService:Create(MainWindow,
                TweenInfo.new(0.5, Enum.EasingStyle.Quint, Enum.EasingDirection.Out),
                {Size = UDim2.new(0, WindowName.TextBounds.X + 140, 0, 50)}):Play()
            task.wait(0.1)
            WindowStuff.Visible = false
        end
        Minimized = not Minimized
    end)

    -- ★ 未来風イントロ：ネオングロー付きロゴ
    local function LoadSequence()
        MainWindow.Visible = false
        local LoadSequenceLogo = SetProps(MakeElement("Image", WindowConfig.IntroIcon), {
            Parent = Orion, AnchorPoint = Vector2.new(0.5, 0.5),
            Position = UDim2.new(0.5, 0, 0.4, 0), Size = UDim2.new(0, 28, 0, 28),
            ImageColor3 = Color3.fromRGB(0, 255, 230), ImageTransparency = 1
        })
        local LoadSequenceText = SetProps(MakeElement("Label", WindowConfig.IntroText, 14), {
            Parent = Orion, Size = UDim2.new(1, 0, 1, 0), AnchorPoint = Vector2.new(0.5, 0.5),
            Position = UDim2.new(0.5, 19, 0.5, 0), TextXAlignment = Enum.TextXAlignment.Center,
            Font = Enum.Font.GothamBlack, TextTransparency = 1, TextColor3 = Color3.fromRGB(0, 255, 230)
        })
        TweenService:Create(LoadSequenceLogo,
            TweenInfo.new(0.3, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
            {ImageTransparency = 0, Position = UDim2.new(0.5, 0, 0.5, 0)}):Play()
        task.wait(0.8)
        TweenService:Create(LoadSequenceLogo,
            TweenInfo.new(0.3, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
            {Position = UDim2.new(0.5, -(LoadSequenceText.TextBounds.X / 2), 0.5, 0)}):Play()
        task.wait(0.3)
        TweenService:Create(LoadSequenceText,
            TweenInfo.new(0.3, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
            {TextTransparency = 0}):Play()
        task.wait(2)
        TweenService:Create(LoadSequenceText,
            TweenInfo.new(0.3, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
            {TextTransparency = 1}):Play()
        MainWindow.Visible = true
        LoadSequenceLogo:Destroy()
        LoadSequenceText:Destroy()
    end
    if WindowConfig.IntroEnabled then LoadSequence() end

    -- ===== タブ＆要素 =====
    local TabFunction = {}
    function TabFunction:MakeTab(TabConfig)
        TabConfig = TabConfig or {}
        TabConfig.Name = TabConfig.Name or "Tab"
        TabConfig.Icon = TabConfig.Icon or ""
        TabConfig.PremiumOnly = TabConfig.PremiumOnly or false

        local TabFrame = SetChildren(SetProps(MakeElement("Button"), {
            Size = UDim2.new(1, 0, 0, 30), Parent = TabHolder
        }), {
            AddThemeObject(SetProps(MakeElement("Image", TabConfig.Icon), {
                AnchorPoint = Vector2.new(0, 0.5), Size = UDim2.new(0, 18, 0, 18),
                Position = UDim2.new(0, 10, 0.5, 0), ImageTransparency = 0.4, Name = "Ico"
            }), "Text"),
            AddThemeObject(SetProps(MakeElement("Label", TabConfig.Name, 14), {
                Size = UDim2.new(1, -35, 1, 0), Position = UDim2.new(0, 35, 0, 0),
                Font = Enum.Font.GothamSemibold, TextTransparency = 0.4, Name = "Title"
            }), "Text")
        })
        if GetIcon(TabConfig.Icon) then TabFrame.Ico.Image = GetIcon(TabConfig.Icon) end

        local Container = AddThemeObject(
            SetChildren(SetProps(MakeElement("ScrollFrame", Color3.fromRGB(0, 220, 255), 5), {
                Size = UDim2.new(1, -150, 1, -50), Position = UDim2.new(0, 150, 0, 50),
                Parent = MainWindow, Visible = false, Name = "ItemContainer"
            }), {
                MakeElement("List", 0, 6),
                MakeElement("Padding", 15, 10, 10, 15)
            }), "Divider"
        )
        AddConnection(Container.UIListLayout:GetPropertyChangedSignal("AbsoluteContentSize"), function()
            Container.CanvasSize = UDim2.new(0, 0, 0, Container.UIListLayout.AbsoluteContentSize.Y + 30)
        end)

        if FirstTab then
            FirstTab = false
            TabFrame.Ico.ImageTransparency = 0
            TabFrame.Title.TextTransparency = 0
            TabFrame.Title.Font = Enum.Font.GothamBlack
            TabFrame.Title.TextColor3 = Color3.fromRGB(0, 255, 230) -- 選択タブはネオン
            Container.Visible = true
        end

        AddConnection(TabFrame.MouseButton1Click, function()
            for _, Tab in next, TabHolder:GetChildren() do
                if Tab:IsA("TextButton") then
                    Tab.Title.Font = Enum.Font.GothamSemibold
                    TweenService:Create(Tab.Ico, TweenInfo.new(0.25, Enum.EasingStyle.Quint, Enum.EasingDirection.Out),
                        {ImageTransparency = 0.4}):Play()
                    TweenService:Create(Tab.Title, TweenInfo.new(0.25, Enum.EasingStyle.Quint, Enum.EasingDirection.Out),
                        {TextTransparency = 0.4, TextColor3 = Color3.fromRGB(220, 245, 255)}):Play()
                end
            end
            for _, IC in next, MainWindow:GetChildren() do
                if IC.Name == "ItemContainer" then IC.Visible = false end
            end
            TweenService:Create(TabFrame.Ico, TweenInfo.new(0.25, Enum.EasingStyle.Quint, Enum.EasingDirection.Out),
                {ImageTransparency = 0}):Play()
            TweenService:Create(TabFrame.Title, TweenInfo.new(0.25, Enum.EasingStyle.Quint, Enum.EasingDirection.Out),
                {TextTransparency = 0, TextColor3 = Color3.fromRGB(0, 255, 230)}):Play()
            TabFrame.Title.Font = Enum.Font.GothamBlack
            Container.Visible = true
        end)

        -- 要素生成関数（未来風スタイル）
        local function GetElements(ItemParent)
            local ElementFunction = {}

            function ElementFunction:AddLabel(Text)
                local LabelFrame = AddThemeObject(
                    SetChildren(SetProps(MakeElement("RoundFrame", Color3.fromRGB(255, 255, 255), 0, 6), {
                        Size = UDim2.new(1, 0, 0, 30), BackgroundTransparency = 0.75, Parent = ItemParent
                    }), {
                        AddThemeObject(SetProps(MakeElement("Label", Text, 15), {
                            Size = UDim2.new(1, -12, 1, 0), Position = UDim2.new(0, 12, 0, 0),
                            Font = Enum.Font.GothamBold, Name = "Content"
                        }), "Text"),
                        AddNeonStroke(nil, Color3.fromRGB(0, 220, 255), 1.2)
                    }), "Second"
                )
                local f = LabelFrame:FindFirstChildOfClass("UIStroke"); if f then f.Parent = LabelFrame end
                local LF = {}
                function LF:Set(t) LabelFrame.Content.Text = t end
                return LF
            end

            function ElementFunction:AddParagraph(Text, Content)
                Text = Text or "Text"; Content = Content or "Content"
                local ParagraphFrame = AddThemeObject(
                    SetChildren(SetProps(MakeElement("RoundFrame", Color3.fromRGB(255, 255, 255), 0, 6), {
                        Size = UDim2.new(1, 0, 0, 30), BackgroundTransparency = 0.75, Parent = ItemParent
                    }), {
                        AddThemeObject(SetProps(MakeElement("Label", Text, 15), {
                            Size = UDim2.new(1, -12, 0, 14), Position = UDim2.new(0, 12, 0, 10),
                            Font = Enum.Font.GothamBold, Name = "Title"
                        }), "Text"),
                        AddThemeObject(SetProps(MakeElement("Label", "", 13), {
                            Size = UDim2.new(1, -24, 0, 0), Position = UDim2.new(0, 12, 0, 26),
                            Font = Enum.Font.GothamSemibold, Name = "Content", TextWrapped = true
                        }), "TextDark"),
                        AddNeonStroke(nil, Color3.fromRGB(0, 220, 255), 1.2)
                    }), "Second"
                )
                local s = ParagraphFrame:FindFirstChildOfClass("UIStroke"); if s then s.Parent = ParagraphFrame end
                AddConnection(ParagraphFrame.Content:GetPropertyChangedSignal("Text"), function()
                    ParagraphFrame.Content.Size = UDim2.new(1, -24, 0, ParagraphFrame.Content.TextBounds.Y)
                    ParagraphFrame.Size = UDim2.new(1, 0, 0, ParagraphFrame.Content.TextBounds.Y + 35)
                end)
                ParagraphFrame.Content.Text = Content
                local PF = {}
                function PF:Set(t) ParagraphFrame.Content.Text = t end
                return PF
            end

            -- ボタン（ホバーでネオン）
            function ElementFunction:AddButton(ButtonConfig)
                ButtonConfig = ButtonConfig or {}
                ButtonConfig.Name = ButtonConfig.Name or "Button"
                ButtonConfig.Callback = ButtonConfig.Callback or function() end
                ButtonConfig.Icon = ButtonConfig.Icon or "rbxassetid://3944703587"

                local Button = {}
                local Click = SetProps(MakeElement("Button"), {Size = UDim2.new(1, 0, 1, 0)})
                local ButtonFrame = AddThemeObject(
                    SetChildren(SetProps(MakeElement("RoundFrame", Color3.fromRGB(255, 255, 255), 0, 6), {
                        Size = UDim2.new(1, 0, 0, 33), Parent = ItemParent
                    }), {
                        AddThemeObject(SetProps(MakeElement("Label", ButtonConfig.Name, 15), {
                            Size = UDim2.new(1, -12, 1, 0), Position = UDim2.new(0, 12, 0, 0),
                            Font = Enum.Font.GothamBold, Name = "Content"
                        }), "Text"),
                        AddThemeObject(SetProps(MakeElement("Image", ButtonConfig.Icon), {
                            Size = UDim2.new(0, 20, 0, 20), Position = UDim2.new(1, -30, 0, 7)
                        }), "TextDark"),
                        AddNeonStroke(nil, Color3.fromRGB(0, 220, 255), 1.2),
                        Click
                    }), "Second"
                )
                local s = ButtonFrame:FindFirstChildOfClass("UIStroke"); if s then s.Parent = ButtonFrame end

                AddConnection(Click.MouseEnter, function()
                    TweenService:Create(ButtonFrame, TweenInfo.new(0.2, Enum.EasingStyle.Quint, Enum.EasingDirection.Out),
                        {BackgroundColor3 = Color3.fromRGB(24, 34, 56)}):Play()
                    if s then TweenService:Create(s, TweenInfo.new(0.2), {Transparency = 0}):Play() end
                end)
                AddConnection(Click.MouseLeave, function()
                    TweenService:Create(ButtonFrame, TweenInfo.new(0.2, Enum.EasingStyle.Quint, Enum.EasingDirection.Out),
                        {BackgroundColor3 = OrionLib.Themes.Default.Second}):Play()
                    if s then TweenService:Create(s, TweenInfo.new(0.2), {Transparency = 0.2}):Play() end
                end)
                AddConnection(Click.MouseButton1Up, function()
                    spawn(function() ButtonConfig.Callback() end)
                end)
                function Button:Set(t) ButtonFrame.Content.Text = t end
                return Button
            end

            -- トグル（ネオングロー）
            function ElementFunction:AddToggle(ToggleConfig)
                ToggleConfig = ToggleConfig or {}
                ToggleConfig.Name = ToggleConfig.Name or "Toggle"
                ToggleConfig.Default = ToggleConfig.Default or false
                ToggleConfig.Callback = ToggleConfig.Callback or function() end
                ToggleConfig.Color = ToggleConfig.Color or Color3.fromRGB(0, 220, 255)
                ToggleConfig.Flag = ToggleConfig.Flag or nil
                ToggleConfig.Save = ToggleConfig.Save or false

                local Toggle = {Value = ToggleConfig.Default, Save = ToggleConfig.Save}
                local Click = SetProps(MakeElement("Button"), {Size = UDim2.new(1, 0, 1, 0)})
                local ToggleBox = SetChildren(SetProps(MakeElement("RoundFrame", ToggleConfig.Color, 0, 5), {
                    Size = UDim2.new(0, 24, 0, 24), Position = UDim2.new(1, -24, 0.5, 0),
                    AnchorPoint = Vector2.new(0.5, 0.5)
                }), {
                    SetProps(MakeElement("Stroke"), {Color = ToggleConfig.Color, Name = "Stroke", Transparency = 0.5}),
                    SetProps(MakeElement("Image", "rbxassetid://3944680095"), {
                        Size = UDim2.new(0, 20, 0, 20), AnchorPoint = Vector2.new(0.5, 0.5),
                        Position = UDim2.new(0.5, 0, 0.5, 0),
                        ImageColor3 = Color3.fromRGB(255, 255, 255), Name = "Ico"
                    })
                })
                local ToggleFrame = AddThemeObject(
                    SetChildren(SetProps(MakeElement("RoundFrame", Color3.fromRGB(255, 255, 255), 0, 6), {
                        Size = UDim2.new(1, 0, 0, 38), Parent = ItemParent
                    }), {
                        AddThemeObject(SetProps(MakeElement("Label", ToggleConfig.Name, 15), {
                            Size = UDim2.new(1, -12, 1, 0), Position = UDim2.new(0, 12, 0, 0),
                            Font = Enum.Font.GothamBold, Name = "Content"
                        }), "Text"),
                        AddNeonStroke(nil, ToggleConfig.Color, 1.2),
                        ToggleBox, Click
                    }), "Second"
                )
                local s = ToggleFrame:FindFirstChildOfClass("UIStroke"); if s then s.Parent = ToggleFrame end

                function Toggle:Set(Value)
                    Toggle.Value = Value
                    TweenService:Create(ToggleBox, TweenInfo.new(0.3, Enum.EasingStyle.Quint, Enum.EasingDirection.Out),
                        {BackgroundColor3 = Toggle.Value and ToggleConfig.Color or Color3.fromRGB(30, 40, 60)}):Play()
                    TweenService:Create(ToggleBox.Stroke, TweenInfo.new(0.3, Enum.EasingStyle.Quint, Enum.EasingDirection.Out),
                        {Color = Toggle.Value and ToggleConfig.Color or Color3.fromRGB(80, 100, 130), Transparency = Toggle.Value and 0 or 0.5}):Play()
                    TweenService:Create(ToggleBox.Ico, TweenInfo.new(0.3, Enum.EasingStyle.Quint, Enum.EasingDirection.Out), {
                        ImageTransparency = Toggle.Value and 0 or 1,
                        Size = Toggle.Value and UDim2.new(0, 20, 0, 20) or UDim2.new(0, 8, 0, 8)
                    }):Play()
                    ToggleConfig.Callback(Toggle.Value)
                end
                Toggle:Set(Toggle.Value)

                AddConnection(Click.MouseButton1Up, function()
                    SaveCfg(game.GameId)
                    Toggle:Set(not Toggle.Value)
                end)
                if ToggleConfig.Flag then OrionLib.Flags[ToggleConfig.Flag] = Toggle end
                return Toggle
            end

            -- スライダー（未来風：バーがグラデ＆ネオン）
            function ElementFunction:AddSlider(SliderConfig)
                SliderConfig = SliderConfig or {}
                SliderConfig.Name = SliderConfig.Name or "Slider"
                SliderConfig.Min = SliderConfig.Min or 0
                SliderConfig.Max = SliderConfig.Max or 100
                SliderConfig.Increment = SliderConfig.Increment or 1
                SliderConfig.Default = SliderConfig.Default or 50
                SliderConfig.Callback = SliderConfig.Callback or function() end
                SliderConfig.ValueName = SliderConfig.ValueName or ""
                SliderConfig.Color = SliderConfig.Color or Color3.fromRGB(0, 220, 255)
                SliderConfig.Flag = SliderConfig.Flag or nil
                SliderConfig.Save = SliderConfig.Save or false

                local Slider = {Value = SliderConfig.Default, Save = SliderConfig.Save}
                local Dragging = false

                local SliderDrag = SetChildren(SetProps(MakeElement("RoundFrame", SliderConfig.Color, 0, 5), {
                    Size = UDim2.new(0, 0, 1, 0), BackgroundTransparency = 0.2, ClipsDescendants = true
                }), {
                    AddThemeObject(SetProps(MakeElement("Label", "value", 13), {
                        Size = UDim2.new(1, -12, 0, 14), Position = UDim2.new(0, 12, 0, 6),
                        Font = Enum.Font.GothamBold, Name = "Value", TextTransparency = 0
                    }), "Text")
                })

                local SliderBar = SetChildren(SetProps(MakeElement("RoundFrame", SliderConfig.Color, 0, 5), {
                    Size = UDim2.new(1, -24, 0, 26), Position = UDim2.new(0, 12, 0, 30), BackgroundTransparency = 0.85
                }), {
                    SetProps(MakeElement("Stroke"), {Color = SliderConfig.Color, Transparency = 0.4}),
                    AddThemeObject(SetProps(MakeElement("Label", "value", 13), {
                        Size = UDim2.new(1, -12, 0, 14), Position = UDim2.new(0, 12, 0, 6),
                        Font = Enum.Font.GothamBold, Name = "Value", TextTransparency = 0.8
                    }), "Text"),
                    SliderDrag
                })

                local SliderFrame = AddThemeObject(
                    SetChildren(SetProps(MakeElement("RoundFrame", Color3.fromRGB(255, 255, 255), 0, 6), {
                        Size = UDim2.new(1, 0, 0, 65), Parent = ItemParent
                    }), {
                        AddThemeObject(SetProps(MakeElement("Label", SliderConfig.Name, 15), {
                            Size = UDim2.new(1, -12, 0, 14), Position = UDim2.new(0, 12, 0, 10),
                            Font = Enum.Font.GothamBold, Name = "Content"
                        }), "Text"),
                        AddNeonStroke(nil, SliderConfig.Color, 1.2),
                        SliderBar
                    }), "Second"
                )
                local s = SliderFrame:FindFirstChildOfClass("UIStroke"); if s then s.Parent = SliderFrame end

                SliderBar.InputBegan:Connect(function(Input)
                    if Input.UserInputType == Enum.UserInputType.MouseButton1 or Input.UserInputType == Enum.UserInputType.Touch then
                        Dragging = true
                    end
                end)
                SliderBar.InputEnded:Connect(function(Input)
                    if Input.UserInputType == Enum.UserInputType.MouseButton1 or Input.UserInputType == Enum.UserInputType.Touch then
                        Dragging = false
                    end
                end)
                UserInputService.InputChanged:Connect(function(Input)
                    if Dragging and (Input.UserInputType == Enum.UserInputType.MouseMovement or Input.UserInputType == Enum.UserInputType.Touch) then
                        local Scale = math.clamp((Input.Position.X - SliderBar.AbsolutePosition.X) / SliderBar.AbsoluteSize.X, 0, 1)
                        Slider:Set(SliderConfig.Min + ((SliderConfig.Max - SliderConfig.Min) * Scale))
                        SaveCfg(game.GameId)
                    end
                end)

                function Slider:Set(Value)
                    self.Value = math.clamp(Round(Value, SliderConfig.Increment), SliderConfig.Min, SliderConfig.Max)
                    TweenService:Create(SliderDrag, TweenInfo.new(0.15, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
                        Size = UDim2.fromScale((self.Value - SliderConfig.Min) / (SliderConfig.Max - SliderConfig.Min), 1)
                    }):Play()
                    SliderBar.Value.Text = tostring(self.Value) .. " " .. SliderConfig.ValueName
                    SliderDrag.Value.Text = tostring(self.Value) .. " " .. SliderConfig.ValueName
                    SliderConfig.Callback(self.Value)
                end
                Slider:Set(Slider.Value)
                if SliderConfig.Flag then OrionLib.Flags[SliderConfig.Flag] = Slider end
                return Slider
            end

            -- ドロップダウン（未来風：ネオン枠）
            function ElementFunction:AddDropdown(DropdownConfig)
                DropdownConfig = DropdownConfig or {}
                DropdownConfig.Name = DropdownConfig.Name or "Dropdown"
                DropdownConfig.Options = DropdownConfig.Options or {}
                DropdownConfig.Default = DropdownConfig.Default or ""
                DropdownConfig.Callback = DropdownConfig.Callback or function() end
                DropdownConfig.Flag = DropdownConfig.Flag or nil
                DropdownConfig.Save = DropdownConfig.Save or false

                local Dropdown = {
                    Value = DropdownConfig.Default, Options = DropdownConfig.Options,
                    Buttons = {}, Toggled = false, Type = "Dropdown", Save = DropdownConfig.Save
                }
                local MaxElements = 5
                if not table.find(Dropdown.Options, Dropdown.Value) then Dropdown.Value = "..." end

                local DropdownList = MakeElement("List")
                local DropdownContainer = AddThemeObject(
                    SetProps(SetChildren(MakeElement("ScrollFrame", Color3.fromRGB(40, 40, 40), 4), {DropdownList}), {
                        Parent = ItemParent, Position = UDim2.new(0, 0, 0, 38),
                        Size = UDim2.new(1, 0, 1, -38), ClipsDescendants = true
                    }), "Divider"
                )
                local Click = SetProps(MakeElement("Button"), {Size = UDim2.new(1, 0, 1, 0)})
                local DropdownFrame = AddThemeObject(
                    SetChildren(SetProps(MakeElement("RoundFrame", Color3.fromRGB(255, 255, 255), 0, 6), {
                        Size = UDim2.new(1, 0, 0, 38), Parent = ItemParent, ClipsDescendants = true
                    }), {
                        DropdownContainer,
                        SetProps(SetChildren(MakeElement("TFrame"), {
                            AddThemeObject(SetProps(MakeElement("Label", DropdownConfig.Name, 15), {
                                Size = UDim2.new(1, -12, 1, 0), Position = UDim2.new(0, 12, 0, 0),
                                Font = Enum.Font.GothamBold, Name = "Content"
                            }), "Text"),
                            AddThemeObject(SetProps(MakeElement("Image", "rbxassetid://7072706796"), {
                                Size = UDim2.new(0, 20, 0, 20), AnchorPoint = Vector2.new(0, 0.5),
                                Position = UDim2.new(1, -30, 0.5, 0), ImageColor3 = Color3.fromRGB(0, 220, 255), Name = "Ico"
                            }), "TextDark"),
                            AddThemeObject(SetProps(MakeElement("Label", "Selected", 13), {
                                Size = UDim2.new(1, -40, 1, 0), Font = Enum.Font.Gotham,
                                Name = "Selected", TextXAlignment = Enum.TextXAlignment.Right
                            }), "TextDark"),
                            AddThemeObject(SetProps(MakeElement("Frame"), {
                                Size = UDim2.new(1, 0, 0, 1), Position = UDim2.new(0, 0, 1, -1), Name = "Line", Visible = false
                            }), "Stroke"),
                            Click
                        }), {Size = UDim2.new(1, 0, 0, 38), ClipsDescendants = true, Name = "F"}),
                        AddNeonStroke(nil, Color3.fromRGB(0, 220, 255), 1.2),
                        MakeElement("Corner")
                    }), "Second"
                )
                local s = DropdownFrame:FindFirstChildOfClass("UIStroke"); if s then s.Parent = DropdownFrame end

                AddConnection(DropdownList:GetPropertyChangedSignal("AbsoluteContentSize"), function()
                    DropdownContainer.CanvasSize = UDim2.new(0, 0, 0, DropdownList.AbsoluteContentSize.Y)
                end)

                local function AddOptions(Options)
                    for _, Option in pairs(Options) do
                        local OptionBtn = AddThemeObject(
                            SetProps(SetChildren(MakeElement("Button", Color3.fromRGB(40, 40, 40)), {
                                MakeElement("Corner", 0, 6),
                                AddThemeObject(SetProps(MakeElement("Label", Option, 13, 0.4), {
                                    Position = UDim2.new(0, 8, 0, 0), Size = UDim2.new(1, -8, 1, 0), Name = "Title"
                                }), "Text")
                            }), {
                                Parent = DropdownContainer, Size = UDim2.new(1, 0, 0, 28),
                                BackgroundTransparency = 1, ClipsDescendants = true
                            }), "Divider"
                        )
                        AddConnection(OptionBtn.MouseButton1Click, function()
                            Dropdown:Set(Option); SaveCfg(game.GameId)
                        end)
                        Dropdown.Buttons[Option] = OptionBtn
                    end
                end
                function Dropdown:Refresh(Options, Delete)
                    if Delete then
                        for _, v in pairs(Dropdown.Buttons) do v:Destroy() end
                        table.clear(Dropdown.Options); table.clear(Dropdown.Buttons)
                    end
                    Dropdown.Options = Options
                    AddOptions(Dropdown.Options)
                end
                function Dropdown:Set(Value)
                    if not table.find(Dropdown.Options, Value) then
                        Dropdown.Value = "..."; DropdownFrame.F.Selected.Text = Dropdown.Value
                        return
                    end
                    Dropdown.Value = Value
                    DropdownFrame.F.Selected.Text = Dropdown.Value
                    for _, v in pairs(Dropdown.Buttons) do
                        TweenService:Create(v, TweenInfo.new(0.15, Enum.EasingStyle.Quad), {BackgroundTransparency = 1}):Play()
                        TweenService:Create(v.Title, TweenInfo.new(0.15, Enum.EasingStyle.Quad), {TextTransparency = 0.4}):Play()
                    end
                    TweenService:Create(Dropdown.Buttons[Value], TweenInfo.new(0.15, Enum.EasingStyle.Quad), {BackgroundTransparency = 0}):Play()
                    TweenService:Create(Dropdown.Buttons[Value].Title, TweenInfo.new(0.15, Enum.EasingStyle.Quad), {TextTransparency = 0}):Play()
                    return DropdownConfig.Callback(Dropdown.Value)
                end
                AddConnection(Click.MouseButton1Click, function()
                    Dropdown.Toggled = not Dropdown.Toggled
                    DropdownFrame.F.Line.Visible = Dropdown.Toggled
                    TweenService:Create(DropdownFrame.F.Ico, TweenInfo.new(0.15, Enum.EasingStyle.Quad),
                        {Rotation = Dropdown.Toggled and 180 or 0}):Play()
                    local targetSize = Dropdown.Toggled and
                        UDim2.new(1, 0, 0, (#Dropdown.Options > MaxElements and (38 + MaxElements * 28) or DropdownList.AbsoluteContentSize.Y + 38))
                        or UDim2.new(1, 0, 0, 38)
                    TweenService:Create(DropdownFrame, TweenInfo.new(0.15, Enum.EasingStyle.Quad), {Size = targetSize}):Play()
                end)
                Dropdown:Refresh(Dropdown.Options, false)
                Dropdown:Set(Dropdown.Value)
                if DropdownConfig.Flag then OrionLib.Flags[DropdownConfig.Flag] = Dropdown end
                return Dropdown
            end

            -- バインド（未来風）
            function ElementFunction:AddBind(BindConfig)
                BindConfig.Name = BindConfig.Name or "Bind"
                BindConfig.Default = BindConfig.Default or Enum.KeyCode.Unknown
                BindConfig.Hold = BindConfig.Hold or false
                BindConfig.Callback = BindConfig.Callback or function() end
                BindConfig.Flag = BindConfig.Flag or nil
                BindConfig.Save = BindConfig.Save or false

                local Bind = {Value = BindConfig.Default, Binding = false, Type = "Bind", Save = BindConfig.Save}
                local Holding = false
                local Click = SetProps(MakeElement("Button"), {Size = UDim2.new(1, 0, 1, 0)})
                local BindBox = AddThemeObject(SetChildren(SetProps(MakeElement("RoundFrame", Color3.fromRGB(255, 255, 255), 0, 5), {
                    Size = UDim2.new(0, 24, 0, 24), Position = UDim2.new(1, -12, 0.5, 0), AnchorPoint = Vector2.new(1, 0.5)
                }), {
                    AddNeonStroke(nil, Color3.fromRGB(0, 220, 255), 1.2),
                    AddThemeObject(SetProps(MakeElement("Label", BindConfig.Name, 14), {
                        Size = UDim2.new(1, 0, 1, 0), Font = Enum.Font.GothamBold,
                        TextXAlignment = Enum.TextXAlignment.Center, Name = "Value"
                    }), "Text")
                }), "Main")
                local bs = BindBox:FindFirstChildOfClass("UIStroke"); if bs then bs.Parent = BindBox end

                local BindFrame = AddThemeObject(SetChildren(SetProps(MakeElement("RoundFrame", Color3.fromRGB(255, 255, 255), 0, 6), {
                    Size = UDim2.new(1, 0, 0, 38), Parent = ItemParent
                }), {
                    AddThemeObject(SetProps(MakeElement("Label", BindConfig.Name, 15), {
                        Size = UDim2.new(1, -12, 1, 0), Position = UDim2.new(0, 12, 0, 0),
                        Font = Enum.Font.GothamBold, Name = "Content"
                    }), "Text"),
                    AddNeonStroke(nil, Color3.fromRGB(0, 220, 255), 1.2),
                    BindBox, Click
                }), "Second")
                local fs = BindFrame:FindFirstChildOfClass("UIStroke"); if fs then fs.Parent = BindFrame end

                AddConnection(BindBox.Value:GetPropertyChangedSignal("Text"), function()
                    TweenService:Create(BindBox, TweenInfo.new(0.25, Enum.EasingStyle.Quint, Enum.EasingDirection.Out),
                        {Size = UDim2.new(0, BindBox.Value.TextBounds.X + 16, 0, 24)}):Play()
                end)
                AddConnection(Click.InputEnded, function(Input)
                    if Input.UserInputType == Enum.UserInputType.MouseButton1 and not Bind.Binding then
                        Bind.Binding = true
                        BindBox.Value.Text = ""
                    end
                end)
                AddConnection(UserInputService.InputBegan, function(Input)
                    if UserInputService:GetFocusedTextBox() then return end
                    if (Input.KeyCode.Name == Bind.Value or Input.UserInputType.Name == Bind.Value) and not Bind.Binding then
                        if BindConfig.Hold then Holding = true; BindConfig.Callback(Holding) else BindConfig.Callback() end
                    elseif Bind.Binding then
                        local Key
                        pcall(function() if not CheckKey(BlacklistedKeys, Input.KeyCode) then Key = Input.KeyCode end end)
                        pcall(function() if CheckKey(WhitelistedMouse, Input.UserInputType) and not Key then Key = Input.UserInputType end end)
                        Bind:Set(Key or Bind.Value)
                        SaveCfg(game.GameId)
                    end
                end)
                AddConnection(UserInputService.InputEnded, function(Input)
                    if Input.KeyCode.Name == Bind.Value or Input.UserInputType.Name == Bind.Value then
                        if BindConfig.Hold and Holding then Holding = false; BindConfig.Callback(Holding) end
                    end
                end)
                function Bind:Set(Key)
                    Bind.Binding = false
                    Bind.Value = Key or Bind.Value
                    Bind.Value = Bind.Value.Name or Bind.Value
                    BindBox.Value.Text = Bind.Value
                end
                Bind:Set(BindConfig.Default)
                if BindConfig.Flag then OrionLib.Flags[BindConfig.Flag] = Bind end
                return Bind
            end

            -- テキストボックス
            function ElementFunction:AddTextbox(TextboxConfig)
                TextboxConfig = TextboxConfig or {}
                TextboxConfig.Name = TextboxConfig.Name or "Textbox"
                TextboxConfig.Default = TextboxConfig.Default or ""
                TextboxConfig.TextDisappear = TextboxConfig.TextDisappear or false
                TextboxConfig.Callback = TextboxConfig.Callback or function() end

                local Click = SetProps(MakeElement("Button"), {Size = UDim2.new(1, 0, 1, 0)})
                local TextboxActual = AddThemeObject(Create("TextBox", {
                    Size = UDim2.new(1, 0, 1, 0), BackgroundTransparency = 1,
                    TextColor3 = Color3.fromRGB(0, 255, 230), PlaceholderColor3 = Color3.fromRGB(120, 180, 210),
                    PlaceholderText = "Input", Font = Enum.Font.GothamSemibold,
                    TextXAlignment = Enum.TextXAlignment.Center, TextSize = 14, ClearTextOnFocus = false
                }), "Text")
                local TextContainer = AddThemeObject(SetChildren(SetProps(MakeElement("RoundFrame", Color3.fromRGB(255, 255, 255), 0, 5), {
                    Size = UDim2.new(0, 24, 0, 24), Position = UDim2.new(1, -12, 0.5, 0), AnchorPoint = Vector2.new(1, 0.5)
                }), {
                    AddNeonStroke(nil, Color3.fromRGB(0, 220, 255), 1.2),
                    TextboxActual
                }), "Main")
                local cs = TextContainer:FindFirstChildOfClass("UIStroke"); if cs then cs.Parent = TextContainer end

                local TextboxFrame = AddThemeObject(SetChildren(SetProps(MakeElement("RoundFrame", Color3.fromRGB(255, 255, 255), 0, 6), {
                    Size = UDim2.new(1, 0, 0, 38), Parent = ItemParent
                }), {
                    AddThemeObject(SetProps(MakeElement("Label", TextboxConfig.Name, 15), {
                        Size = UDim2.new(1, -12, 1, 0), Position = UDim2.new(0, 12, 0, 0),
                        Font = Enum.Font.GothamBold, Name = "Content"
                    }), "Text"),
                    AddNeonStroke(nil, Color3.fromRGB(0, 220, 255), 1.2),
                    TextContainer, Click
                }), "Second")
                local ts = TextboxFrame:FindFirstChildOfClass("UIStroke"); if ts then ts.Parent = TextboxFrame end

                AddConnection(TextboxActual:GetPropertyChangedSignal("Text"), function()
                    TweenService:Create(TextContainer, TweenInfo.new(0.35, Enum.EasingStyle.Quint, Enum.EasingDirection.Out),
                        {Size = UDim2.new(0, TextboxActual.TextBounds.X + 16, 0, 24)}):Play()
                end)
                AddConnection(TextboxActual.FocusLost, function()
                    TextboxConfig.Callback(TextboxActual.Text)
                    if TextboxConfig.TextDisappear then TextboxActual.Text = "" end
                end)
                TextboxActual.Text = TextboxConfig.Default
                AddConnection(Click.MouseButton1Up, function() TextboxActual:CaptureFocus() end)
            end

            -- カラーピッカー（未来風：色相バーをネオンに）
            function ElementFunction:AddColorpicker(ColorpickerConfig)
                ColorpickerConfig = ColorpickerConfig or {}
                ColorpickerConfig.Name = ColorpickerConfig.Name or "Colorpicker"
                ColorpickerConfig.Default = ColorpickerConfig.Default or Color3.fromRGB(255, 255, 255)
                ColorpickerConfig.Callback = ColorpickerConfig.Callback or function() end
                ColorpickerConfig.Flag = ColorpickerConfig.Flag or nil
                ColorpickerConfig.Save = ColorpickerConfig.Save or false

                local ColorH, ColorS, ColorV = 1, 1, 1
                local Colorpicker = {Value = ColorpickerConfig.Default, Toggled = false, Type = "Colorpicker", Save = ColorpickerConfig.Save}

                local ColorSelection = Create("ImageLabel", {
                    Size = UDim2.new(0, 18, 0, 18),
                    Position = UDim2.new(select(3, Color3.toHSV(Colorpicker.Value))),
                    ScaleType = Enum.ScaleType.Fit, AnchorPoint = Vector2.new(0.5, 0.5),
                    BackgroundTransparency = 1, Image = "http://www.roblox.com/asset/?id=4805639000"
                })
                local HueSelection = Create("ImageLabel", {
                    Size = UDim2.new(0, 18, 0, 18),
                    Position = UDim2.new(0.5, 0, 1 - select(1, Color3.toHSV(Colorpicker.Value))),
                    ScaleType = Enum.ScaleType.Fit, AnchorPoint = Vector2.new(0.5, 0.5),
                    BackgroundTransparency = 1, Image = "http://www.roblox.com/asset/?id=4805639000"
                })
                local Color = Create("ImageLabel", {
                    Size = UDim2.new(1, -25, 1, 0), Visible = false, Image = "rbxassetid://4155801252"
                }, {
                    Create("UICorner", {CornerRadius = UDim.new(0, 5)}),
                    ColorSelection
                })
                local Hue = Create("Frame", {
                    Size = UDim2.new(0, 20, 1, 0), Position = UDim2.new(1, -20, 0, 0), Visible = false
                }, {
                    Create("UIGradient", {
                        Rotation = 270,
                        Color = ColorSequence.new {
                            ColorSequenceKeypoint.new(0.00, Color3.fromRGB(255, 0, 4)),
                            ColorSequenceKeypoint.new(0.20, Color3.fromRGB(234, 255, 0)),
                            ColorSequenceKeypoint.new(0.40, Color3.fromRGB(21, 255, 0)),
                            ColorSequenceKeypoint.new(0.60, Color3.fromRGB(0, 255, 255)),
                            ColorSequenceKeypoint.new(0.80, Color3.fromRGB(0, 17, 255)),
                            ColorSequenceKeypoint.new(0.90, Color3.fromRGB(255, 0, 251)),
                            ColorSequenceKeypoint.new(1.00, Color3.fromRGB(255, 0, 4))
                        }
                    }),
                    Create("UICorner", {CornerRadius = UDim.new(0, 5)}),
                    HueSelection
                })
                local ColorpickerContainer = Create("Frame", {
                    Position = UDim2.new(0, 0, 0, 32), Size = UDim2.new(1, 0, 1, -32),
                    BackgroundTransparency = 1, ClipsDescendants = true
                }, {
                    Hue, Color,
                    Create("UIPadding", {
                        PaddingLeft = UDim.new(0, 35), PaddingRight = UDim.new(0, 35),
                        PaddingBottom = UDim.new(0, 10), PaddingTop = UDim.new(0, 17)
                    })
                })
                local Click = SetProps(MakeElement("Button"), {Size = UDim2.new(1, 0, 1, 0)})
                local ColorpickerBox = AddThemeObject(SetChildren(SetProps(MakeElement("RoundFrame", Color3.fromRGB(255, 255, 255), 0, 5), {
                    Size = UDim2.new(0, 24, 0, 24), Position = UDim2.new(1, -12, 0.5, 0), AnchorPoint = Vector2.new(1, 0.5)
                }), {
                    AddNeonStroke(nil, Color3.fromRGB(0, 220, 255), 1.2)
                }), "Main")
                local cbs = ColorpickerBox:FindFirstChildOfClass("UIStroke"); if cbs then cbs.Parent = ColorpickerBox end

                local ColorpickerFrame = AddThemeObject(SetChildren(SetProps(MakeElement("RoundFrame", Color3.fromRGB(255, 255, 255), 0, 6), {
                    Size = UDim2.new(1, 0, 0, 38), Parent = ItemParent
                }), {
                    SetProps(SetChildren(MakeElement("TFrame"), {
                        AddThemeObject(SetProps(MakeElement("Label", ColorpickerConfig.Name, 15), {
                            Size = UDim2.new(1, -12, 1, 0), Position = UDim2.new(0, 12, 0, 0),
                            Font = Enum.Font.GothamBold, Name = "Content"
                        }), "Text"),
                        ColorpickerBox, Click,
                        AddThemeObject(SetProps(MakeElement("Frame"), {
                            Size = UDim2.new(1, 0, 0, 1), Position = UDim2.new(0, 0, 1, -1), Name = "Line", Visible = false
                        }), "Stroke")
                    }), {Size = UDim2.new(1, 0, 0, 38), ClipsDescendants = true, Name = "F"}),
                    ColorpickerContainer,
                    AddNeonStroke(nil, Color3.fromRGB(0, 220, 255), 1.2)
                }), "Second")
                local cfs = ColorpickerFrame:FindFirstChildOfClass("UIStroke"); if cfs then cfs.Parent = ColorpickerFrame end

                AddConnection(Click.MouseButton1Click, function()
                    Colorpicker.Toggled = not Colorpicker.Toggled
                    TweenService:Create(ColorpickerFrame, TweenInfo.new(0.15, Enum.EasingStyle.Quad),
                        {Size = Colorpicker.Toggled and UDim2.new(1, 0, 0, 148) or UDim2.new(1, 0, 0, 38)}):Play()
                    Color.Visible = Colorpicker.Toggled
                    Hue.Visible = Colorpicker.Toggled
                    ColorpickerFrame.F.Line.Visible = Colorpicker.Toggled
                end)
                local function UpdateColorPicker()
                    ColorpickerBox.BackgroundColor3 = Color3.fromHSV(ColorH, ColorS, ColorV)
                    Color.BackgroundColor3 = Color3.fromHSV(ColorH, 1, 1)
                    Colorpicker:Set(ColorpickerBox.BackgroundColor3)
                    ColorpickerConfig.Callback(ColorpickerBox.BackgroundColor3)
                    SaveCfg(game.GameId)
                end
                Color.InputBegan:Connect(function(input)
                    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                        if ColorInput then ColorInput:Disconnect() end
                        ColorInput = RunService.RenderStepped:Connect(function()
                            local ip = UserInputService:GetLastInputType() == Enum.UserInputType.Touch and input.Position or UserInputService:GetMouseLocation()
                            local X = math.clamp((ip.X - Color.AbsolutePosition.X) / Color.AbsoluteSize.X, 0, 1)
                            local Y = math.clamp((ip.Y - Color.AbsolutePosition.Y) / Color.AbsoluteSize.Y, 0, 1)
                            ColorSelection.Position = UDim2.new(X, 0, Y, 0)
                            ColorS, ColorV = X, 1 - Y
                            UpdateColorPicker()
                        end)
                    end
                end)
                Color.InputEnded:Connect(function(input)
                    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                        if ColorInput then ColorInput:Disconnect(); ColorInput = nil end
                    end
                end)
                Hue.InputBegan:Connect(function(input)
                    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                        if HueInput then HueInput:Disconnect() end
                        HueInput = AddConnection(RunService.RenderStepped, function()
                            local Y = (math.clamp(Mouse.Y - Hue.AbsolutePosition.Y, 0, Hue.AbsoluteSize.Y) / Hue.AbsoluteSize.Y)
                            HueSelection.Position = UDim2.new(0.5, 0, Y, 0)
                            ColorH = 1 - Y
                            UpdateColorPicker()
                        end)
                    end
                end)
                Hue.InputEnded:Connect(function(input)
                    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                        if HueInput then HueInput:Disconnect() end
                    end
                end)
                function Colorpicker:Set(Value)
                    Colorpicker.Value = Value
                    ColorpickerBox.BackgroundColor3 = Colorpicker.Value
                    ColorpickerConfig.Callback(Colorpicker.Value)
                end
                Colorpicker:Set(Colorpicker.Value)
                if ColorpickerConfig.Flag then OrionLib.Flags[ColorpickerConfig.Flag] = Colorpicker end
                return Colorpicker
            end
            return ElementFunction
        end

        local ElementFunction = {}
        function ElementFunction:AddSection(SectionConfig)
            SectionConfig.Name = SectionConfig.Name or "Section"
            local SectionFrame = SetChildren(SetProps(MakeElement("TFrame"), {
                Size = UDim2.new(1, 0, 0, 26), Parent = Container
            }), {
                AddThemeObject(SetProps(MakeElement("Label", SectionConfig.Name, 14), {
                    Size = UDim2.new(1, -12, 0, 16), Position = UDim2.new(0, 0, 0, 3),
                    Font = Enum.Font.GothamSemibold, TextColor3 = Color3.fromRGB(0, 255, 230)
                }), "TextDark"),
                SetChildren(SetProps(MakeElement("TFrame"), {
                    AnchorPoint = Vector2.new(0, 0), Size = UDim2.new(1, 0, 1, -24),
                    Position = UDim2.new(0, 0, 0, 23), Name = "Holder"
                }), {MakeElement("List", 0, 6)})
            })
            AddConnection(SectionFrame.Holder.UIListLayout:GetPropertyChangedSignal("AbsoluteContentSize"), function()
                SectionFrame.Size = UDim2.new(1, 0, 0, SectionFrame.Holder.UIListLayout.AbsoluteContentSize.Y + 31)
                SectionFrame.Holder.Size = UDim2.new(1, 0, 0, SectionFrame.Holder.UIListLayout.AbsoluteContentSize.Y)
            end)
            local SectionFunction = {}
            for i, v in next, GetElements(SectionFrame.Holder) do SectionFunction[i] = v end
            return SectionFunction
        end
        for i, v in next, GetElements(Container) do ElementFunction[i] = v end

        if TabConfig.PremiumOnly then
            for i, v in next, ElementFunction do ElementFunction[i] = function() end end
            Container:FindFirstChild("UIListLayout"):Destroy()
            Container:FindFirstChild("UIPadding"):Destroy()
            SetChildren(SetProps(MakeElement("TFrame"), {Size = UDim2.new(1, 0, 1, 0), Parent = Container}), {
                AddThemeObject(SetProps(MakeElement("Image", "rbxassetid://3610239960"), {
                    Size = UDim2.new(0, 18, 0, 18), Position = UDim2.new(0, 15, 0, 15), ImageTransparency = 0.4
                }), "Text"),
                AddThemeObject(SetProps(MakeElement("Label", "Unauthorised Access", 14), {
                    Size = UDim2.new(1, -38, 0, 14), Position = UDim2.new(0, 38, 0, 18), TextTransparency = 0.4
                }), "Text"),
                AddThemeObject(SetProps(MakeElement("Image", "rbxassetid://4483345875"), {
                    Size = UDim2.new(0, 56, 0, 56), Position = UDim2.new(0, 84, 0, 110)
                }), "Text"),
                AddThemeObject(SetProps(MakeElement("Label", "Premium Features", 14), {
                    Size = UDim2.new(1, -150, 0, 14), Position = UDim2.new(0, 150, 0, 112),
                    Font = Enum.Font.GothamBold
                }), "Text"),
                AddThemeObject(SetProps(MakeElement("Label",
                    "This part is locked to Premium users.", 12), {
                    Size = UDim2.new(1, -200, 0, 14), Position = UDim2.new(0, 150, 0, 138),
                    TextWrapped = true, TextTransparency = 0.4
                }), "Text")
            })
        end
        return ElementFunction
    end

    OrionLib:MakeNotification({
        Name = "◈ FUTURE UI ◈",
        Content = "Futuristic Orion loaded successfully.",
        Time = 5
    })
    return TabFunction
end

function OrionLib:Destroy()
    Orion:Destroy()
end

return OrionLib

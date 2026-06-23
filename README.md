if _G.LoadedTracer then
    if _G.MenuGUI then pcall(function() _G.MenuGUI:Destroy() end) end
    if _G.ClearDrawings then pcall(function() _G.ClearDrawings() end) end
end
_G.LoadedTracer = true

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local CoreGui = game:GetService("CoreGui")
local HttpService = game:GetService("HttpService")
local TweenService = game:GetService("TweenService")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

local Vector2_new = Vector2.new
local Vector3_new = Vector3.new
local Color3_fromRGB = Color3.fromRGB
local Color3_fromHSV = Color3.fromHSV
local math_clamp = math.clamp
local math_floor = math.floor
local math_huge = math.huge

local CustomFont = Font.new("rbxassetid://12187607493")

local TargetGui = CoreGui
if not pcall(function() local a = CoreGui.Name end) then
    TargetGui = LocalPlayer:FindFirstChildOfClass("PlayerGui")
end

local Config = {
    Aim = true,
    ESP = true,
    Tracers = true,
    Skeletons = true,
    Chams = true,
    Hitboxes = false,
    HitboxSize = 2,
    FOV = 150,
    Speed = 0.3,
    Smooth = 0.1,
    Part = "Head",
    MaxDist = 1000,
    Running = true,
    Language = "RU"
}

local DrawingStore = {}
local SkeletonStore = {}
local OriginalSizes = {}
local CustomFriends = {}
local FOV_Circle = nil
local RenderConnection = nil

local hasDrawing = pcall(function()
    local c = Drawing.new("Circle")
    c:Remove()
end)

if hasDrawing then
    FOV_Circle = Drawing.new("Circle")
end

local function SaveConfig()
    if writefile and HttpService then
        local data = {Config = Config, Friends = CustomFriends}
        pcall(function() writefile("Vortex_Config.json", HttpService:JSONEncode(data)) end)
    end
end

local function LoadConfig()
    if readfile and isfile and isfile("Vortex_Config.json") then
        pcall(function()
            local data = HttpService:JSONDecode(readfile("Vortex_Config.json"))
            if data then
                if data.Config then for k, v in pairs(data.Config) do Config[k] = v end end
                if data.Friends then for k, v in pairs(data.Friends) do CustomFriends[k] = v end end
            end
        end)
    end
end
LoadConfig()

-- [[ ЛОКАЛИЗАЦИЯ И СЛОВАРЬ ПОДСКАЗОК ]]
local Locale = {
    RU = {
        Combat = "Бой", Visuals = "Визуалы", Friends = "Друзья", Settings = "Настройки",
        AimbotMaster = "Аимбот Мастер", TargetPart = "Цель Аима", AimbotSpeed = "Скорость Аима",
        Smoothness = "Плавность", HitboxExpander = "Расширитель Хитбоксов", HitboxSize = "Размер Хитбокса",
        Boxes = "2D Боксы", Tracers = "Трейсера", Skeletons = "Скелеты", ChamsGlow = "Чамсы (Подсветка)",
        FieldOfView = "Радиус FOV", SaveSettings = "Сохранить Настройки", UnloadScript = "Выгрузить Скрипт",
        Remove = "Удалить", AddFriend = "Добавить", Loading = "Загрузка proj.Vortex...",
        Close = "Закрыть", InfoTitle = "Описание функции",
        Help_AimbotMaster = "Автоматически наводит камеру на выбранную часть тела противника при стрельбе или зажатии.",
        Help_TargetPart = "Выбирает конкретную часть тела (Голова, Торс), на которую будет наводиться аимбот.",
        Help_AimbotSpeed = "Регулирует скорость доводки прицела до цели. Выше значение — быстрее наводка.",
        Help_Smoothness = "Сглаживает движение камеры при наводке, делая её более легитной и незаметной.",
        Help_HitboxExpander = "Увеличивает хитбоксы (размер частей тела) врагов, позволяя легче попадать по ним.",
        Help_HitboxSize = "Задает итоговый размер увеличенных хитбоксов противников.",
        Help_Boxes = "Отрисовывает квадратные рамки вокруг врагов, позволяя видеть их сквозь стены.",
        Help_Tracers = "Проводит линии от центра экрана к каждому противнику для быстрого поиска целей.",
        Help_Skeletons = "Отображает линии костей персонажей игроков, показывая их точные позы за препятствиями.",
        Help_ChamsGlow = "Полностью подсвечивает модельки игроков ярким силуэтом, видимым сквозь любые текстуры.",
        Help_FieldOfView = "Регулирует радиус круга наведения аима. Скрипт игнорирует цели вне этого круга."
    },
    EN = {
        Combat = "Combat", Visuals = "Visuals", Friends = "Friends", Settings = "Settings",
        AimbotMaster = "Aimbot Master", TargetPart = "Target Part", AimbotSpeed = "Aimbot Speed",
        Smoothness = "Smoothness", HitboxExpander = "Hitbox Expander", HitboxSize = "Hitbox Size",
        Boxes = "2D Boxes", Tracers = "Tracers", Skeletons = "Skeletons", ChamsGlow = "Chams Glow",
        FieldOfView = "Field of View", SaveSettings = "Save Settings", UnloadScript = "Unload Script",
        Remove = "Remove", AddFriend = "Add Friend", Loading = "Loading proj.Vortex...",
        Close = "Close", InfoTitle = "Function Info",
        Help_AimbotMaster = "Automatically locks your camera onto the selected target body part when aiming.",
        Help_TargetPart = "Selects the specific body part (Head, Torso) that the aimbot will lock onto.",
        Help_AimbotSpeed = "Adjusts the aiming response speed. Higher value means faster lock-on.",
        Help_Smoothness = "Smooths out camera movements while aiming, making tracking look legit and natural.",
        Help_HitboxExpander = "Increases enemy hitboxes (body part sizes), making them significantly easier to hit.",
        Help_HitboxSize = "Sets the total size scale for the expanded enemy hitboxes.",
        Help_Boxes = "Draws 2D boxes around players, letting you locate them through walls.",
        Help_Tracers = "Draws lines from the center of your screen to every enemy player.",
        Help_Skeletons = "Renders skeleton bone lines on players, showing their exact stance behind cover.",
        Help_ChamsGlow = "Fills character models with a bright silhouette highlight visible through walls.",
        Help_FieldOfView = "Adjusts the aimbot targeting radius. Targets outside this circle are ignored."
    }
}

local function GetText(key)
    return Locale[Config.Language][key] or key
end

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "Vortex_Menu"
ScreenGui.ResetOnSpawn = false
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ScreenGui.Parent = TargetGui
_G.MenuGUI = ScreenGui

-- [[ ИНТРО С ОБНОВЛЕННЫМ РАЗМЕРОМ ХОЛСТА (742x287) ]]
local IntroFrame = Instance.new("Frame")
IntroFrame.Name = "IntroFrame"
IntroFrame.Size = UDim2.new(0, 440, 0, 170)
IntroFrame.Position = UDim2.new(0.5, -220, 0.5, -85)
IntroFrame.BackgroundColor3 = Color3_fromRGB(10, 12, 15)
IntroFrame.BorderSizePixel = 0
IntroFrame.BackgroundTransparency = 1
IntroFrame.Visible = false
IntroFrame.Parent = ScreenGui

local IntroCorner = Instance.new("UICorner")
IntroCorner.CornerRadius = UDim.new(0, 12)
IntroCorner.Parent = IntroFrame

local IntroStroke = Instance.new("UIStroke")
IntroStroke.Color = Color3_fromRGB(0, 180, 255)
IntroStroke.Thickness = 1.5
IntroStroke.Transparency = 1
IntroStroke.Parent = IntroFrame

local IntroImage = Instance.new("ImageLabel")
IntroImage.Name = "IntroImage"
IntroImage.Size = UDim2.new(1, -20, 1, -40)
IntroImage.Position = UDim2.new(0, 10, 0, 10)
IntroImage.BackgroundTransparency = 1
IntroImage.Image = "rbxassetid://113758115501265"
IntroImage.ScaleType = Enum.ScaleType.Fit
IntroImage.ImageTransparency = 1
IntroImage.Parent = IntroFrame

local IntroText = Instance.new("TextLabel")
IntroText.Name = "IntroText"
IntroText.Size = UDim2.new(1, 0, 0, 25)
IntroText.Position = UDim2.new(0, 0, 1, -30)
IntroText.BackgroundTransparency = 1
IntroText.Text = GetText("Loading")
IntroText.FontFace = CustomFont
IntroText.TextSize = 13
IntroText.TextColor3 = Color3_fromRGB(0, 180, 255)
IntroText.TextTransparency = 1
IntroText.Parent = IntroFrame

-- [[ ГЛАВНОЕ МЕНЮ ]]
local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Size = UDim2.new(0, 520, 0, 340)
MainFrame.Position = UDim2.new(0.5, -260, 0.5, -170)
MainFrame.BackgroundColor3 = Color3_fromRGB(10, 11, 14)
MainFrame.BorderSizePixel = 0
MainFrame.Active = true
MainFrame.Draggable = true
MainFrame.Visible = false
MainFrame.BackgroundTransparency = 1
MainFrame.Parent = ScreenGui

local MainCorner = Instance.new("UICorner")
MainCorner.CornerRadius = UDim.new(0, 10)
MainCorner.Parent = MainFrame

local MainStroke = Instance.new("UIStroke")
MainStroke.Color = Color3_fromRGB(0, 130, 200)
MainStroke.Thickness = 1.2
MainStroke.Parent = MainFrame

-- [[ КНОПКА Х ]]
local CollapseBtn = Instance.new("TextButton")
CollapseBtn.Name = "CollapseBtn"
CollapseBtn.Size = UDim2.new(0, 26, 0, 26)
CollapseBtn.Position = UDim2.new(1, -34, 0, 10)
CollapseBtn.BackgroundColor3 = Color3_fromRGB(18, 20, 26)
CollapseBtn.Text = "X"
CollapseBtn.FontFace = CustomFont
CollapseBtn.TextSize = 12
CollapseBtn.TextColor3 = Color3_fromRGB(240, 70, 70)
CollapseBtn.ZIndex = 10
CollapseBtn.Parent = MainFrame

local CollapseCorner = Instance.new("UICorner")
CollapseCorner.CornerRadius = UDim.new(0, 6)
CollapseCorner.Parent = CollapseBtn

local CollapseStroke = Instance.new("UIStroke")
CollapseStroke.Color = Color3_fromRGB(40, 45, 55)
CollapseStroke.Thickness = 1
CollapseStroke.Parent = CollapseBtn

-- [[ КНОПКА СМЕНЫ ЯЗЫКА (RU / EN) ]]
local LangBtn = Instance.new("TextButton")
LangBtn.Name = "LangBtn"
LangBtn.Size = UDim2.new(0, 34, 0, 26)
LangBtn.Position = UDim2.new(1, -74, 0, 10)
LangBtn.BackgroundColor3 = Color3_fromRGB(18, 20, 26)
LangBtn.Text = Config.Language
LangBtn.FontFace = CustomFont
LangBtn.TextSize = 11
LangBtn.TextColor3 = Color3_fromRGB(0, 180, 255)
LangBtn.ZIndex = 10
LangBtn.Parent = MainFrame

local LangCorner = Instance.new("UICorner")
LangCorner.CornerRadius = UDim.new(0, 6)
LangCorner.Parent = LangBtn

local LangStroke = Instance.new("UIStroke")
LangStroke.Color = Color3_fromRGB(40, 45, 55)
LangStroke.Thickness = 1
LangStroke.Parent = LangBtn

-- [[ ОКНО ДЛЯ МИНИ-ПОДСКАЗОК "?" ]]
local InfoFrame = Instance.new("Frame")
InfoFrame.Name = "InfoFrame"
InfoFrame.Size = UDim2.new(0, 280, 0, 160)
InfoFrame.Position = UDim2.new(0.5, -140, 0.5, -80)
InfoFrame.BackgroundColor3 = Color3_fromRGB(12, 14, 18)
InfoFrame.BorderSizePixel = 0
InfoFrame.ZIndex = 20
InfoFrame.Visible = false
InfoFrame.Parent = MainFrame

local InfoCorner = Instance.new("UICorner")
InfoCorner.CornerRadius = UDim.new(0, 8)
InfoCorner.Parent = InfoFrame

local InfoStroke = Instance.new("UIStroke")
InfoStroke.Color = Color3_fromRGB(0, 150, 255)
InfoStroke.Thickness = 1.2
InfoStroke.Parent = InfoFrame

local InfoTitle = Instance.new("TextLabel")
InfoTitle.Size = UDim2.new(1, -20, 0, 30)
InfoTitle.Position = UDim2.new(0, 12, 0, 5)
InfoTitle.Text = GetText("InfoTitle")
InfoTitle.FontFace = CustomFont
InfoTitle.TextSize = 13
InfoTitle.TextColor3 = Color3_fromRGB(0, 180, 255)
InfoTitle.TextXAlignment = Enum.TextXAlignment.Left
InfoTitle.BackgroundTransparency = 1
InfoTitle.ZIndex = 21
InfoTitle.Parent = InfoFrame

local InfoText = Instance.new("TextLabel")
InfoText.Size = UDim2.new(1, -24, 1, -75)
InfoText.Position = UDim2.new(0, 12, 0, 35)
InfoText.Text = ""
InfoText.FontFace = CustomFont
InfoText.TextSize = 12
InfoText.TextColor3 = Color3_fromRGB(180, 195, 205)
InfoText.TextWrapped = true
InfoText.TextXAlignment = Enum.TextXAlignment.Left
InfoText.TextYAlignment = Enum.TextYAlignment.Top
InfoText.BackgroundTransparency = 1
InfoText.ZIndex = 21
InfoText.Parent = InfoFrame

local InfoCloseBtn = Instance.new("TextButton")
InfoCloseBtn.Size = UDim2.new(1, -24, 0, 28)
InfoCloseBtn.Position = UDim2.new(0, 12, 1, -34)
InfoCloseBtn.BackgroundColor3 = Color3_fromRGB(22, 26, 35)
InfoCloseBtn.Text = GetText("Close")
InfoCloseBtn.FontFace = CustomFont
InfoCloseBtn.TextSize = 12
InfoCloseBtn.TextColor3 = Color3_fromRGB(255, 255, 255)
InfoCloseBtn.ZIndex = 21
InfoCloseBtn.Parent = InfoFrame

local InfoCloseCorner = Instance.new("UICorner")
InfoCloseCorner.CornerRadius = UDim.new(0, 5)
InfoCloseCorner.Parent = InfoCloseBtn

local InfoCloseStroke = Instance.new("UIStroke")
InfoCloseStroke.Color = Color3_fromRGB(45, 50, 65)
InfoCloseStroke.Thickness = 1
InfoCloseStroke.Parent = InfoCloseBtn

InfoCloseBtn.MouseButton1Click:Connect(function()
    InfoFrame.Visible = false
end)

local function ShowHelp(helpKey)
    InfoTitle.Text = GetText("InfoTitle")
    InfoCloseBtn.Text = GetText("Close")
    InfoText.Text = GetText(helpKey)
    InfoFrame.Visible = true
end

-- [[ ПЛАВАЮЩАЯ КНОПКА ОТКРЫТИЯ ]]
local ToggleBtn = Instance.new("TextButton")
ToggleBtn.Name = "Vortex_Toggle"
ToggleBtn.Size = UDim2.new(0, 46, 0, 46)
ToggleBtn.Position = UDim2.new(0, 15, 0, 15)
ToggleBtn.BackgroundColor3 = Color3_fromRGB(10, 12, 16)
ToggleBtn.Text = "V"
ToggleBtn.FontFace = CustomFont
ToggleBtn.TextSize = 18
ToggleBtn.TextColor3 = Color3_fromRGB(0, 180, 255)
ToggleBtn.Visible = false
ToggleBtn.Active = true
ToggleBtn.Draggable = true
ToggleBtn.Parent = ScreenGui

local ToggleCorner = Instance.new("UICorner")
ToggleCorner.CornerRadius = UDim.new(0, 23)
ToggleCorner.Parent = ToggleBtn

local ToggleStroke = Instance.new("UIStroke")
ToggleStroke.Color = Color3_fromRGB(0, 130, 200)
ToggleStroke.Thickness = 1.5
ToggleStroke.Parent = ToggleBtn

CollapseBtn.MouseButton1Click:Connect(function()
    MainFrame.Visible = false
    InfoFrame.Visible = false
    ToggleBtn.Visible = true
end)

ToggleBtn.MouseButton1Click:Connect(function()
    ToggleBtn.Visible = false
    MainFrame.Visible = true
end)

local Sidebar = Instance.new("Frame")
Sidebar.Name = "Sidebar"
Sidebar.Size = UDim2.new(0, 140, 1, 0)
Sidebar.BackgroundColor3 = Color3_fromRGB(7, 8, 10)
Sidebar.BorderSizePixel = 0
Sidebar.Parent = MainFrame

local SidebarCorner = Instance.new("UICorner")
SidebarCorner.CornerRadius = UDim.new(0, 10)
SidebarCorner.Parent = Sidebar

local SidebarFix = Instance.new("Frame")
SidebarFix.Size = UDim2.new(0, 20, 1, 0)
SidebarFix.Position = UDim2.new(1, -20, 0, 0)
SidebarFix.BackgroundColor3 = Color3_fromRGB(7, 8, 10)
SidebarFix.BorderSizePixel = 0
SidebarFix.Parent = Sidebar

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, 0, 0, 45)
Title.Position = UDim2.new(0, 15, 0, 0)
Title.Text = "proj.Vortex"
Title.FontFace = CustomFont
Title.TextSize = 16
Title.TextColor3 = Color3_fromRGB(0, 200, 255)
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.BackgroundTransparency = 1
Title.Parent = Sidebar

local TabContainer = Instance.new("Frame")
TabContainer.Size = UDim2.new(1, -10, 1, -60)
TabContainer.Position = UDim2.new(0, 5, 0, 55)
TabContainer.BackgroundTransparency = 1
TabContainer.Parent = Sidebar

local TabListLayout = Instance.new("UIListLayout")
TabListLayout.SortOrder = Enum.SortOrder.LayoutOrder
TabListLayout.Padding = UDim.new(0, 4)
TabListLayout.Parent = TabContainer

local ContainerFolder = Instance.new("Folder")
ContainerFolder.Name = "Containers"
ContainerFolder.Parent = MainFrame

local currentTab = nil
local updateListElements = {}

local function CreateTab(name, localeKey)
    local TabButton = Instance.new("TextButton")
    TabButton.Size = UDim2.new(1, 0, 0, 34)
    TabButton.BackgroundColor3 = Color3_fromRGB(0, 100, 150)
    TabButton.BackgroundTransparency = 1
    TabButton.Text = "  " .. GetText(localeKey)
    TabButton.FontFace = CustomFont
    TabButton.TextSize = 13
    TabButton.TextColor3 = Color3_fromRGB(120, 140, 150)
    TabButton.TextXAlignment = Enum.TextXAlignment.Left
    TabButton.Parent = TabContainer
    
    table.insert(updateListElements, function()
        TabButton.Text = "  " .. GetText(localeKey)
    end)
    
    local ButtonCorner = Instance.new("UICorner")
    ButtonCorner.CornerRadius = UDim.new(0, 6)
    ButtonCorner.Parent = TabButton

    local Container = Instance.new("ScrollingFrame")
    Container.Size = UDim2.new(1, -165, 1, -20)
    Container.Position = UDim2.new(0, 150, 0, 10)
    Container.BackgroundTransparency = 1
    Container.BorderSizePixel = 0
    Container.CanvasSize = UDim2.new(0, 0, 0, 0)
    Container.ScrollBarThickness = 2
    Container.ScrollBarImageColor3 = Color3_fromRGB(0, 130, 200)
    Container.Visible = false
    Container.Parent = ContainerFolder

    local ContainerLayout = Instance.new("UIListLayout")
    ContainerLayout.SortOrder = Enum.SortOrder.LayoutOrder
    ContainerLayout.Padding = UDim.new(0, 6)
    ContainerLayout.Parent = Container
    
    ContainerLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
        Container.CanvasSize = UDim2.new(0, 0, 0, ContainerLayout.AbsoluteContentSize.Y + 10)
    end)

    TabButton.MouseButton1Click:Connect(function()
        if currentTab then
            currentTab.Button.BackgroundTransparency = 1
            currentTab.Button.TextColor3 = Color3_fromRGB(120, 140, 150)
            currentTab.Container.Visible = false
        end
        TabButton.BackgroundTransparency = 0.8
        TabButton.TextColor3 = Color3_fromRGB(0, 220, 255)
        Container.Visible = true
        currentTab = {Button = TabButton, Container = Container}
    end)

    return Container
end

-- [[ ОБНОВЛЕННЫЕ ЭЛЕМЕНТЫ УПРАВЛЕНИЯ С КНОПКОЙ "?" ]]
local function AddToggle(container, localeKey, default, callback)
    local ToggleFrame = Instance.new("Frame")
    ToggleFrame.Size = UDim2.new(1, -45, 0, 36)
    ToggleFrame.BackgroundColor3 = Color3_fromRGB(14, 16, 22)
    ToggleFrame.BorderSizePixel = 0
    ToggleFrame.Parent = container

    local TFCorner = Instance.new("UICorner")
    TFCorner.CornerRadius = UDim.new(0, 6)
    TFCorner.Parent = ToggleFrame
    
    local TFStroke = Instance.new("UIStroke")
    TFStroke.Color = Color3_fromRGB(25, 30, 40)
    TFStroke.Thickness = 1
    TFStroke.Parent = ToggleFrame

    local Label = Instance.new("TextLabel")
    Label.Size = UDim2.new(1, -100, 1, 0)
    Label.Position = UDim2.new(0, 12)
    Label.Text = GetText(localeKey)
    Label.FontFace = CustomFont
    Label.TextSize = 13
    Label.TextColor3 = Color3_fromRGB(200, 220, 230)
    Label.TextXAlignment = Enum.TextXAlignment.Left
    Label.BackgroundTransparency = 1
    Label.Parent = ToggleFrame

    table.insert(updateListElements, function()
        Label.Text = GetText(localeKey)
    end)

    local HelpBtn = Instance.new("TextButton")
    HelpBtn.Size = UDim2.new(0, 20, 0, 20)
    HelpBtn.Position = UDim2.new(1, -72, 0.5, -10)
    HelpBtn.BackgroundColor3 = Color3_fromRGB(24, 28, 38)
    HelpBtn.Text = "?"
    HelpBtn.FontFace = CustomFont
    HelpBtn.TextSize = 11
    HelpBtn.TextColor3 = Color3_fromRGB(0, 180, 255)
    HelpBtn.Parent = ToggleFrame

    local HelpCorner = Instance.new("UICorner")
    HelpCorner.CornerRadius = UDim.new(0, 10)
    HelpCorner.Parent = HelpBtn
    
    HelpBtn.MouseButton1Click:Connect(function()
        ShowHelp("Help_" .. localeKey)
    end)

    local Button = Instance.new("TextButton")
    Button.Size = UDim2.new(0, 34, 0, 20)
    Button.Position = UDim2.new(1, -44, 0.5, -10)
    Button.BackgroundColor3 = default and Color3_fromRGB(0, 150, 255) or Color3_fromRGB(30, 35, 45)
    Button.Text = ""
    Button.Parent = ToggleFrame

    local BCorner = Instance.new("UICorner")
    BCorner.CornerRadius = UDim.new(0, 10)
    BCorner.Parent = Button

    local Dot = Instance.new("Frame")
    Dot.Size = UDim2.new(0, 14, 0, 14)
    Dot.Position = default and UDim2.new(1, -17, 0.5, -7) or UDim2.new(0, 3, 0.5, -7)
    Dot.BackgroundColor3 = Color3_fromRGB(255, 255, 255)
    Dot.BorderSizePixel = 0
    Dot.Parent = Button
    
    local DotCorner = Instance.new("UICorner")
    DotCorner.CornerRadius = UDim.new(0, 7)
    DotCorner.Parent = Dot

    local state = default
    Button.MouseButton1Click:Connect(function()
        state = not state
        TweenService:Create(Button, TweenInfo.new(0.2), {BackgroundColor3 = state and Color3_fromRGB(0, 150, 255) or Color3_fromRGB(30, 35, 45)}):Play()
        TweenService:Create(Dot, TweenInfo.new(0.2), {Position = state and UDim2.new(1, -17, 0.5, -7) or UDim2.new(0, 3, 0.5, -7)}):Play()
        callback(state)
    end)
end

local function AddSlider(container, localeKey, min, max, default, callback)
    local SliderFrame = Instance.new("Frame")
    SliderFrame.Size = UDim2.new(1, -45, 0, 46)
    SliderFrame.BackgroundColor3 = Color3_fromRGB(14, 16, 22)
    SliderFrame.BorderSizePixel = 0
    SliderFrame.Parent = container

    local SFCorner = Instance.new("UICorner")
    SFCorner.CornerRadius = UDim.new(0, 6)
    SFCorner.Parent = SliderFrame
    
    local SFStroke = Instance.new("UIStroke")
    SFStroke.Color = Color3_fromRGB(25, 30, 40)
    SFStroke.Thickness = 1
    SFStroke.Parent = SliderFrame

    local Label = Instance.new("TextLabel")
    Label.Size = UDim2.new(1, -140, 0, 24)
    Label.Position = UDim2.new(0, 12, 0, 2)
    Label.Text = GetText(localeKey)
    Label.FontFace = CustomFont
    Label.TextSize = 13
    Label.TextColor3 = Color3_fromRGB(200, 220, 230)
    Label.TextXAlignment = Enum.TextXAlignment.Left
    Label.BackgroundTransparency = 1
    Label.Parent = SliderFrame

    table.insert(updateListElements, function()
        Label.Text = GetText(localeKey)
    end)

    local HelpBtn = Instance.new("TextButton")
    HelpBtn.Size = UDim2.new(0, 20, 0, 20)
    HelpBtn.Position = UDim2.new(1, -102, 0, 4)
    HelpBtn.BackgroundColor3 = Color3_fromRGB(24, 28, 38)
    HelpBtn.Text = "?"
    HelpBtn.FontFace = CustomFont
    HelpBtn.TextSize = 11
    HelpBtn.TextColor3 = Color3_fromRGB(0, 180, 255)
    HelpBtn.Parent = SliderFrame

    local HelpCorner = Instance.new("UICorner")
    HelpCorner.CornerRadius = UDim.new(0, 10)
    HelpCorner.Parent = HelpBtn
    
    HelpBtn.MouseButton1Click:Connect(function()
        ShowHelp("Help_" .. localeKey)
    end)

    local ValueLabel = Instance.new("TextLabel")
    ValueLabel.Size = UDim2.new(0, 60, 0, 24)
    ValueLabel.Position = UDim2.new(1, -72, 0, 2)
    ValueLabel.Text = tostring(default)
    ValueLabel.FontFace = CustomFont
    ValueLabel.TextSize = 12
    ValueLabel.TextColor3 = Color3_fromRGB(0, 180, 255)
    ValueLabel.TextXAlignment = Enum.TextXAlignment.Right
    ValueLabel.BackgroundTransparency = 1
    ValueLabel.Parent = SliderFrame

    local SlideBar = Instance.new("TextButton")
    SlideBar.Size = UDim2.new(1, -24, 0, 4)
    SlideBar.Position = UDim2.new(0, 12, 0, 32)
    SlideBar.BackgroundColor3 = Color3_fromRGB(30, 35, 45)
    SlideBar.Text = ""
    SlideBar.Parent = SliderFrame

    local SBCorner = Instance.new("UICorner")
    SBCorner.CornerRadius = UDim.new(0, 2)
    SBCorner.Parent = SlideBar

    local Fill = Instance.new("Frame")
    local pct = (default - min) / (max - min)
    Fill.Size = UDim2.new(pct, 0, 1, 0)
    Fill.BackgroundColor3 = Color3_fromRGB(0, 150, 255)
    Fill.BorderSizePixel = 0
    Fill.Parent = SlideBar
    
    local FillCorner = Instance.new("UICorner")
    FillCorner.CornerRadius = UDim.new(0, 2)
    FillCorner.Parent = Fill

    local function update(input)
        local sizeX = math_clamp((input.Position.X - SlideBar.AbsolutePosition.X) / SlideBar.AbsoluteSize.X, 0, 1)
        Fill.Size = UDim2.new(sizeX, 0, 1, 0)
        local val = min + (max - min) * sizeX
        if max - min > 50 then val = math_floor(val) else val = math_floor(val * 100) / 100 end
        ValueLabel.Text = tostring(val)
        callback(val)
    end

    local dragging = false
    SlideBar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            update(input)
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            update(input)
        end
    end)
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = false
        end
    end)
end

local function AddDropdown(container, localeKey, options, default, callback)
    local DropFrame = Instance.new("Frame")
    DropFrame.Size = UDim2.new(1, -45, 0, 36)
    DropFrame.BackgroundColor3 = Color3_fromRGB(14, 16, 22)
    DropFrame.BorderSizePixel = 0
    DropFrame.ClipsDescendants = true
    DropFrame.Parent = container

    local DFCorner = Instance.new("UICorner")
    DFCorner.CornerRadius = UDim.new(0, 6)
    DFCorner.Parent = DropFrame
    
    local DFStroke = Instance.new("UIStroke")
    DFStroke.Color = Color3_fromRGB(25, 30, 40)
    DFStroke.Thickness = 1
    DFStroke.Parent = DropFrame

    local Label = Instance.new("TextLabel")
    Label.Size = UDim2.new(0, 130, 0, 36)
    Label.Position = UDim2.new(0, 12)
    Label.Text = GetText(localeKey)
    Label.FontFace = CustomFont
    Label.TextSize = 13
    Label.TextColor3 = Color3_fromRGB(200, 220, 230)
    Label.TextXAlignment = Enum.TextXAlignment.Left
    Label.BackgroundTransparency = 1
    Label.Parent = DropFrame

    table.insert(updateListElements, function()
        Label.Text = GetText(localeKey)
    end)

    local HelpBtn = Instance.new("TextButton")
    HelpBtn.Size = UDim2.new(0, 20, 0, 20)
    HelpBtn.Position = UDim2.new(0, 115, 0.5, -10)
    HelpBtn.BackgroundColor3 = Color3_fromRGB(24, 28, 38)
    HelpBtn.Text = "?"
    HelpBtn.FontFace = CustomFont
    HelpBtn.TextSize = 11
    HelpBtn.TextColor3 = Color3_fromRGB(0, 180, 255)
    HelpBtn.Parent = DropFrame

    local HelpCorner = Instance.new("UICorner")
    HelpCorner.CornerRadius = UDim.new(0, 10)
    HelpCorner.Parent = HelpBtn
    
    HelpBtn.MouseButton1Click:Connect(function()
        ShowHelp("Help_" .. localeKey)
    end)

    local MainBtn = Instance.new("TextButton")
    MainBtn.Size = UDim2.new(0, 130, 0, 24)
    MainBtn.Position = UDim2.new(1, -142, 0, 6)
    MainBtn.BackgroundColor3 = Color3_fromRGB(22, 26, 35)
    MainBtn.Text = default .. "  ▼"
    MainBtn.FontFace = CustomFont
    MainBtn.TextSize = 12
    MainBtn.TextColor3 = Color3_fromRGB(0, 200, 255)
    MainBtn.Parent = DropFrame

    local MBCorner = Instance.new("UICorner")
    MBCorner.CornerRadius = UDim.new(0, 4)
    MBCorner.Parent = MainBtn

    local ListFrame = Instance.new("Frame")
    ListFrame.Size = UDim2.new(0, 130, 0, #options * 26)
    ListFrame.Position = UDim2.new(1, -142, 0, 36)
    ListFrame.BackgroundTransparency = 1
    ListFrame.Parent = DropFrame

    for idx, opt in ipairs(options) do
        local OptBtn = Instance.new("TextButton")
        OptBtn.Size = UDim2.new(1, 0, 0, 24)
        OptBtn.Position = UDim2.new(0, 0, 0, (idx - 1) * 26)
        OptBtn.BackgroundColor3 = Color3_fromRGB(18, 22, 30)
        OptBtn.Text = opt
        OptBtn.FontFace = CustomFont
        OptBtn.TextSize = 12
        OptBtn.TextColor3 = Color3_fromRGB(170, 190, 200)
        OptBtn.BorderSizePixel = 0
        OptBtn.Parent = ListFrame

        OptBtn.MouseButton1Click:Connect(function()
            MainBtn.Text = opt .. "  ▼"
            DropFrame.Size = UDim2.new(1, -45, 0, 36)
            callback(opt)
        end)
    end

    local open = false
    MainBtn.MouseButton1Click:Connect(function()
        open = not open
        DropFrame.Size = open and UDim2.new(1, -45, 0, 42 + (#options * 26)) or UDim2.new(1, -45, 0, 36)
    end)
end

local function AddButton(container, localeKey, callback)
    local Btn = Instance.new("TextButton")
    Btn.Size = UDim2.new(1, -45, 0, 34)
    Btn.BackgroundColor3 = Color3_fromRGB(0, 120, 200)
    Btn.Text = GetText(localeKey)
    Btn.FontFace = CustomFont
    Btn.TextSize = 13
    Btn.TextColor3 = Color3_fromRGB(255, 255, 255)
    Btn.Parent = container

    table.insert(updateListElements, function()
        Btn.Text = GetText(localeKey)
    end)

    local BCorner = Instance.new("UICorner")
    BCorner.CornerRadius = UDim.new(0, 6)
    BCorner.Parent = Btn

    Btn.MouseButton1Click:Connect(callback)
end

-- [[ ИНИЦИАЛИЗАЦИЯ И СМЕНА ЯЗЫКОВЫХ СТРУКТУР ]]
LangBtn.MouseButton1Click:Connect(function()
    Config.Language = (Config.Language == "RU") and "EN" or "RU"
    LangBtn.Text = Config.Language
    for i = 1, #updateListElements do
        updateListElements[i]()
    end
end)

-- [[ ВКЛАДКИ ]]
local CombatTab = CreateTab("Combat", "Combat")
local VisualsTab = CreateTab("Visuals", "Visuals")
local FriendsTab = CreateTab("Friends", "Friends")
local SettingsTab = CreateTab("Settings", "Settings")

AddToggle(CombatTab, "AimbotMaster", Config.Aim, function(v) Config.Aim = v end)
AddDropdown(CombatTab, "TargetPart", {"Head", "HumanoidRootPart", "LowerTorso"}, Config.Part, function(v) Config.Part = v end)
AddSlider(CombatTab, "AimbotSpeed", 0.05, 1, Config.Speed, function(v) Config.Speed = v end)
AddSlider(CombatTab, "Smoothness", 0, 0.9, Config.Smooth, function(v) Config.Smooth = v end)
AddToggle(CombatTab, "HitboxExpander", Config.Hitboxes, function(v) Config.Hitboxes = v end)
AddSlider(CombatTab, "HitboxSize", 2, 30, Config.HitboxSize, function(v) Config.HitboxSize = v end)

AddToggle(VisualsTab, "Boxes", Config.ESP, function(v) Config.ESP = v end)
AddToggle(VisualsTab, "Tracers", Config.Tracers, function(v) Config.Tracers = v end)
AddToggle(VisualsTab, "Skeletons", Config.Skeletons, function(v) Config.Skeletons = v end)
AddToggle(VisualsTab, "ChamsGlow", Config.Chams, function(v) Config.Chams = v end)
AddSlider(VisualsTab, "FieldOfView", 30, 600, Config.FOV, function(v) Config.FOV = v end)

local function RefreshFriendsList()
    for _, child in ipairs(FriendsTab:GetChildren()) do
        if not child:IsA("UIListLayout") then child:Destroy() end
    end
    
    local players = Players:GetPlayers()
    for i = 1, #players do
        local p = players[i]
        if p ~= LocalPlayer then
            local pFrame = Instance.new("Frame")
            pFrame.Size = UDim2.new(1, -45, 0, 36)
            pFrame.BackgroundColor3 = Color3_fromRGB(14, 16, 22)
            pFrame.Parent = FriendsTab
            
            local pfCorner = Instance.new("UICorner")
            pfCorner.CornerRadius = UDim.new(0, 6)
            pfCorner.Parent = pFrame

            local pLabel = Instance.new("TextLabel")
            pLabel.Size = UDim2.new(1, -90, 1, 0)
            pLabel.Position = UDim2.new(0, 12)
            pLabel.Text = p.Name
            pLabel.FontFace = CustomFont
            pLabel.TextSize = 13
            pLabel.TextColor3 = CustomFriends[p.UserId] and Color3_fromRGB(0, 220, 255) or Color3_fromRGB(200, 220, 230)
            pLabel.TextXAlignment = Enum.TextXAlignment.Left
            pLabel.BackgroundTransparency = 1
            pLabel.Parent = pFrame

            local fBtn = Instance.new("TextButton")
            fBtn.Size = UDim2.new(0, 70, 0, 22)
            fBtn.Position = UDim2.new(1, -82, 0.5, -11)
            fBtn.BackgroundColor3 = CustomFriends[p.UserId] and Color3_fromRGB(200, 50, 50) or Color3_fromRGB(0, 130, 200)
            fBtn.Text = CustomFriends[p.UserId] and GetText("Remove") or GetText("AddFriend")
            fBtn.FontFace = CustomFont
            fBtn.TextSize = 11
            fBtn.TextColor3 = Color3_fromRGB(255, 255, 255)
            fBtn.Parent = pFrame
            
            local fCorner = Instance.new("UICorner")
            fCorner.CornerRadius = UDim.new(0, 4)
            fCorner.Parent = fBtn

            fBtn.MouseButton1Click:Connect(function()
                if CustomFriends[p.UserId] then
                    CustomFriends[p.UserId] = nil
                    fBtn.Text = GetText("AddFriend")
                    fBtn.BackgroundColor3 = Color3_fromRGB(0, 130, 200)
                    pLabel.TextColor3 = Color3_fromRGB(200, 220, 230)
                else
                    CustomFriends[p.UserId] = true
                    fBtn.Text = GetText("Remove")
                    fBtn.BackgroundColor3 = Color3_fromRGB(200, 50, 50)
                    pLabel.TextColor3 = Color3_fromRGB(0, 220, 255)
                end
            end)
        end
    end
end
Players.PlayerAdded:Connect(RefreshFriendsList)
Players.PlayerRemoving:Connect(RefreshFriendsList)

AddButton(SettingsTab, "SaveSettings", function() SaveConfig() end)
AddButton(SettingsTab, "UnloadScript", function()
    _G.ClearDrawings()
    ScreenGui:Destroy()
    _G.LoadedTracer = nil
end)

local children = TabContainer:GetChildren()
local firstTabButton = nil
for i = 1, #children do
    if children[i]:IsA("TextButton") then
        firstTabButton = children[i]
        break
    end
end

if firstTabButton then
    firstTabButton.BackgroundTransparency = 0.8
    firstTabButton.TextColor3 = Color3_fromRGB(0, 220, 255)
    local containers = ContainerFolder:GetChildren()
    if containers[1] then containers[1].Visible = true end
    currentTab = {Button = firstTabButton, Container = containers[1]}
end

_G.ClearDrawings = function()
    Config.Running = false
    if RenderConnection then 
        RenderConnection:Disconnect() 
        RenderConnection = nil
    end
    
    for _, obj in pairs(DrawingStore) do
        if obj.Line then pcall(function() obj.Line:Remove() end) end
        if obj.Box then pcall(function() obj.Box:Remove() end) end
        if obj.NameText then pcall(function() obj.NameText:Remove() end) end
        if obj.DistText then pcall(function() obj.DistText:Remove() end) end
        if obj.HpText then pcall(function() obj.HpText:Remove() end) end
    end
    DrawingStore = {}
    
    for _, skeleton in pairs(SkeletonStore) do
        for _, line in pairs(skeleton) do pcall(function() line:Remove() end) end
    end
    SkeletonStore = {}

    if FOV_Circle then pcall(function() FOV_Circle:Remove() end) end

    for p, parts in pairs(OriginalSizes) do
        if p and p.Parent and p:FindFirstChild("Character") then
            for name, size in pairs(parts) do
                local part = p.Character:FindFirstChild(name)
                if part then part.Size = size; part.CanCollide = true end
            end
        end
    end
    OriginalSizes = {}

    local currentPlayers = Players:GetPlayers()
    for i = 1, #currentPlayers do
        local p = currentPlayers[i]
        if p.Character and p.Character:FindFirstChild("vortex_Cham") then 
            pcall(function() p.Character["vortex_Cham"]:Destroy() end) 
        end
    end
end

local function ManageVisualStore(player)
    if not hasDrawing then return end
    if not DrawingStore[player] then
        DrawingStore[player] = {
            Line = Drawing.new("Line"), Box = Drawing.new("Square"),
            NameText = Drawing.new("Text"), DistText = Drawing.new("Text"), HpText = Drawing.new("Text")
        }
        local d = DrawingStore[player]
        d.Line.Thickness = 1.2; d.Box.Thickness = 1.2; d.Box.Filled = false
        d.NameText.Size = 13; d.NameText.Center = true; d.NameText.Outline = true; d.NameText.Font = 3
        d.DistText.Size = 11; d.DistText.Center = true; d.DistText.Outline = true; d.DistText.Font = 3
        d.HpText.Size = 12; d.HpText.Outline = true; d.HpText.Font = 3
    end

    if not SkeletonStore[player] then
        local function cl() local l = Drawing.new("Line"); l.Thickness = 1.5; return l end
        SkeletonStore[player] = {
            HeadToTorso = cl(), TorsoToLShoulder = cl(), LShoulderToLElbow = cl(), LElbowToLWrist = cl(),
            TorsoToRShoulder = cl(), RShoulderToRElbow = cl(), RElbowToRWrist = cl(), TorsoToLHip = cl(),
            LHipToLKnee = cl(), LKneeToLAnkle = cl(), TorsoRHip = cl(), RHipToRKnee = cl(), RKneeToRAnkle = cl()
        }
    end
end

local function GetScreenPos(pos)
    local screen, vis = Camera:WorldToViewportPoint(pos)
    if vis then return Vector2_new(screen.X, screen.Y), true end
    local center = Vector2_new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    local rel = (Vector2_new(screen.X, screen.Y) - center)
    if rel.Magnitude == 0 then return center, false end
    local k = math.min((center.X - 20) / math.abs(rel.X), (center.Y - 20) / math.abs(rel.Y))
    return (screen.Z < 0) and (center - (rel * k)) or (center + (rel * k)), false
end

local function DrawBone(line, p1, p2, col, vis)
    if p1 and p2 then
        local s1, v1 = Camera:WorldToViewportPoint(p1.Position)
        local s2, v2 = Camera:WorldToViewportPoint(p2.Position)
        if v1 and v2 and vis then
            line.From = Vector2_new(s1.X, s1.Y)
            line.To = Vector2_new(s2.X, s2.Y)
            line.Color = col; line.Visible = true
            return
        end
    end
    line.Visible = false
end

local function GetClosestTarget()
    local target, min3DDist = nil, math_huge
    local mouse = Vector2_new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    local playersArray = Players:GetPlayers()

    for i = 1, #playersArray do
        local p = playersArray[i]
        if p ~= LocalPlayer and p.Character and not CustomFriends[p.UserId] then
            local hum = p.Character:FindFirstChild("Humanoid")
            local part = p.Character:FindFirstChild(Config.Part)
            if hum and part and hum.Health > 0 then
                local screen, vis = Camera:WorldToViewportPoint(part.Position)
                if vis then
                    local screenMag = (Vector2_new(screen.X, screen.Y) - mouse).Magnitude
                    if screenMag < Config.FOV then
                        local current3DDist = (Camera.CFrame.Position - part.Position).Magnitude
                        if current3DDist < min3DDist then
                            min3DDist = current3DDist; target = part
                        end
                    end
                end
            end
        end
    end
    return target
end

local function InitMainScript()
    if hasDrawing and FOV_Circle then
        FOV_Circle.Visible = true
        FOV_Circle.Thickness = 1.2
        FOV_Circle.Color = Color3_fromRGB(0, 180, 255)
        FOV_Circle.Filled = false
    end

    RenderConnection = RunService.RenderStepped:Connect(function()
        if not Config.Running then 
            if RenderConnection then RenderConnection:Disconnect() end
            return 
        end

        local center = Vector2_new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
        
        if hasDrawing and FOV_Circle then
            FOV_Circle.Position = center
            FOV_Circle.Radius = Config.FOV
            FOV_Circle.Visible = Config.Aim
        end

        if Config.Aim then
            local t = GetClosestTarget()
            if t then 
                local lookVector = (t.Position - Camera.CFrame.Position).Unit
                local rightVector = lookVector:Cross(Vector3_new(0, 1, 0)).Unit
                local upVector = rightVector:Cross(lookVector).Unit
                local targetCFrame = CFrame.fromMatrix(Camera.CFrame.Position, rightVector, upVector, -lookVector)
                Camera.CFrame = Camera.CFrame:Lerp(targetCFrame, math_clamp(Config.Speed * (1 - Config.Smooth), 0.005, 1))
            end
        end

        local playersList = Players:GetPlayers()
        for i = 1, #playersList do
            local p = playersList[i]
            if p ~= LocalPlayer then
                local char = p.Character
                if char and char:FindFirstChild("HumanoidRootPart") and char:FindFirstChild("Humanoid") and char.Humanoid.Health > 0 then
                    
                    if hasDrawing then ManageVisualStore(p) end
                    
                    local v = DrawingStore[p]
                    local sk = SkeletonStore[p]
                    local root = char.HumanoidRootPart
                    local head = char:FindFirstChild("Head")
                    local hum = char.Humanoid
                    local screenPos, onScreen = GetScreenPos(root.Position)
                    local dist = (Camera.CFrame.Position - root.Position).Magnitude
                    
                    local col = CustomFriends[p.UserId] and Color3_fromRGB(0, 220, 255) or Color3_fromHSV(math_clamp(dist / Config.MaxDist, 0, 0.33), 1, 1)

                    if Config.Chams then
                        local cham = char:FindFirstChild("vortex_Cham")
                        if not cham then
                            cham = Instance.new("Highlight")
                            cham.Name = "vortex_Cham"; cham.Parent = char
                        end
                        cham.FillColor = CustomFriends[p.UserId] and Color3_fromRGB(0, 220, 255) or Color3_fromRGB(0, 140, 255)
                        cham.FillTransparency = 0.5; cham.Enabled = true
                    else
                        if char:FindFirstChild("vortex_Cham") then char["vortex_Cham"].Enabled = false end
                    end

                    if Config.Hitboxes then
                        if not OriginalSizes[p] then
                            OriginalSizes[p] = {}
                            if head then OriginalSizes[p]["Head"] = head.Size end
                            OriginalSizes[p]["HumanoidRootPart"] = root.Size
                        end
                        if head then head.Size = Vector3_new(Config.HitboxSize, Config.HitboxSize, Config.HitboxSize); head.CanCollide = false end
                        root.Size = Vector3_new(Config.HitboxSize, Config.HitboxSize, Config.HitboxSize); root.CanCollide = false
                    else
                        if OriginalSizes[p] then
                            if head and OriginalSizes[p]["Head"] then head.Size = OriginalSizes[p]["Head"] end
                            root.Size = OriginalSizes[p]["HumanoidRootPart"]
                            OriginalSizes[p] = nil
                        end
                    end

                    if hasDrawing and v then
                        v.Line.From = center; v.Line.To = screenPos; v.Line.Color = col; v.Line.Visible = Config.Tracers
                        
                        if Config.ESP and onScreen then
                            local s = math_clamp(2500 / dist, 5, 300)
                            local boxW, boxH = s, s * 1.5
                            local boxX, boxY = screenPos.X - boxW / 2, screenPos.Y - boxH / 2
                            
                            v.Box.Size = Vector2_new(boxW, boxH); v.Box.Position = Vector2_new(boxX, boxY); v.Box.Color = col; v.Box.Visible = true
                            v.NameText.Position = Vector2_new(boxX + boxW / 2, boxY - 16); v.NameText.Text = p.Name; v.NameText.Color = Color3_fromRGB(255, 255, 255); v.NameText.Visible = true
                            v.DistText.Position = Vector2_new(boxX + boxW / 2, boxY + boxH + 2); v.DistText.Text = math_floor(dist) .. " studs"; v.DistText.Color = Color3_fromRGB(180, 200, 210); v.DistText.Visible = true
                            v.HpText.Position = Vector2_new(boxX - 45, boxY + 2); v.HpText.Text = "HP: " .. math_floor(hum.Health); v.HpText.Color = Color3_fromRGB(0, 255, 150):Lerp(Color3_fromRGB(255, 50, 50), 1 - (hum.Health / hum.MaxHealth)); v.HpText.Visible = true
                        else
                            v.Box.Visible = false; v.NameText.Visible = false; v.DistText.Visible = false; v.HpText.Visible = false
                        end
                    end

                    if hasDrawing and sk then
                        local isR15 = (hum.RigType == Enum.HumanoidRigType.R15)
                        local torso = char:FindFirstChild("UpperTorso") or char:FindFirstChild("Torso")
                        local lShoulder = char:FindFirstChild("LeftUpperArm") or char:FindFirstChild("Left Arm")
                        local lElbow = isR15 and char:FindFirstChild("LeftLowerArm") or nil
                        local lWrist = char:FindFirstChild("LeftHand") or nil
                        local rShoulder = char:FindFirstChild("RightUpperArm") or char:FindFirstChild("Right Arm")
                        local rElbow = isR15 and char:FindFirstChild("RightLowerArm") or nil
                        local rWrist = char:FindFirstChild("RightHand") or nil
                        local lHip = char:FindFirstChild("LeftUpperLeg") or char:FindFirstChild("Left Leg")
                        local lKnee = isR15 and char:FindFirstChild("LeftLowerLeg") or nil
                        local lAnkle = char:FindFirstChild("LeftFoot") or nil
                        local rHip = char:FindFirstChild("RightUpperLeg") or char:FindFirstChild("Right Leg")
                        local rKnee = isR15 and char:FindFirstChild("RightLowerLeg") or nil
                        local rAnkle = char:FindFirstChild("RightFoot") or nil

                        local activeSkel = Config.Skeletons and onScreen
                        DrawBone(sk.HeadToTorso, head, torso, col, activeSkel)
                        DrawBone(sk.TorsoToLShoulder, torso, lShoulder, col, activeSkel)
                        DrawBone(sk.LShoulderToLElbow, lShoulder, lElbow or lShoulder, col, isR15 and activeSkel or false)
                        DrawBone(sk.LElbowToLWrist, lElbow or lShoulder, lWrist or lShoulder, col, activeSkel)
                        DrawBone(sk.TorsoToRShoulder, torso, rShoulder, col, activeSkel)
                        DrawBone(sk.RShoulderToRElbow, rShoulder, rElbow or rShoulder, col, isR15 and activeSkel or false)
                        DrawBone(sk.RElbowToRWrist, rElbow or rShoulder, rWrist or rShoulder, col, activeSkel)
                        DrawBone(sk.TorsoToLHip, torso, lHip, col, activeSkel)
                        DrawBone(sk.LHipToLKnee, lHip, lKnee or lHip, col, isR15 and activeSkel or false)
                        DrawBone(sk.LKneeToLAnkle, lKnee or lHip, lAnkle or lHip, col, activeSkel)
                        DrawBone(sk.TorsoRHip, torso, rHip, col, activeSkel)
                        DrawBone(sk.RHipToRKnee, rHip, rKnee or rHip, col, isR15 and activeSkel or false)
                        DrawBone(sk.RKneeToRAnkle, rKnee or rHip, rAnkle or rHip, col, activeSkel)
                    end
                else
                    if hasDrawing and DrawingStore[p] then
                        local v = DrawingStore[p]
                        v.Line.Visible = false; v.Box.Visible = false; v.NameText.Visible = false; v.DistText.Visible = false; v.HpText.Visible = false
                    end
                    if hasDrawing and SkeletonStore[p] then
                        for _, line in pairs(SkeletonStore[p]) do line.Visible = false end
                    end
                    if char and char:FindFirstChild("vortex_Cham") then char["vortex_Cham"].Enabled = false end
                end
            end
        end
    end)
end

task.spawn(function()
    task.wait(4.5)
    
    IntroFrame.Visible = true
    IntroFrame.BackgroundTransparency = 0.2
    TweenService:Create(IntroStroke, TweenInfo.new(0.4), {Transparency = 0}):Play()
    TweenService:Create(IntroImage, TweenInfo.new(0.4), {ImageTransparency = 0}):Play()
    TweenService:Create(IntroText, TweenInfo.new(0.4), {TextTransparency = 0}):Play()
    
    task.wait(2.0)
    
    TweenService:Create(IntroFrame, TweenInfo.new(0.4), {BackgroundTransparency = 1}):Play()
    TweenService:Create(IntroStroke, TweenInfo.new(0.4), {Transparency = 1}):Play()
    TweenService:Create(IntroImage, TweenInfo.new(0.4), {ImageTransparency = 1}):Play()
    TweenService:Create(IntroText, TweenInfo.new(0.4), {TextTransparency = 1}):Play()
    
    task.wait(0.4)
    IntroFrame:Destroy()
    
    MainFrame.Visible = true
    RefreshFriendsList()
    TweenService:Create(MainFrame, TweenInfo.new(0.4), {BackgroundTransparency = 0}):Play()
    
    InitMainScript()
end)

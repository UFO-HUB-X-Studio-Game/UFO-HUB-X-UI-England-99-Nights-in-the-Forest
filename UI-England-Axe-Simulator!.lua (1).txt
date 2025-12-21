-- ==== UFO HUB X • One-shot Boot Guard (PER SESSION; no cooldown reopen) ====
-- วางบนสุดของไฟล์ก่อนโค้ดทั้งหมด
do
    local BOOT = getgenv().UFO_BOOT or { status = "idle" }  -- status: idle|running|done
    -- ถ้ากำลังบูต หรือเคยบูตเสร็จแล้ว → ไม่ให้รันอีก
    if BOOT.status == "running" or BOOT.status == "done" then
        return
    end
    BOOT.status = "running"
    getgenv().UFO_BOOT = BOOT
end
-- ===== UFO HUB X • Local Save (executor filesystem) — per map (PlaceId) =====
do
    local HttpService = game:GetService("HttpService")
    local MarketplaceService = game:GetService("MarketplaceService")

    local FS = {
        isfolder   = (typeof(isfolder)=="function") and isfolder   or function() return false end,
        makefolder = (typeof(makefolder)=="function") and makefolder or function() end,
        isfile     = (typeof(isfile)=="function") and isfile       or function() return false end,
        readfile   = (typeof(readfile)=="function") and readfile   or function() return nil end,
        writefile  = (typeof(writefile)=="function") and writefile or function() end,
    }

    local ROOT = "UFO HUB X"  -- โฟลเดอร์หลักในตัวรัน
    local function safeMakeRoot() pcall(function() if not FS.isfolder(ROOT) then FS.makefolder(ROOT) end end) end
    safeMakeRoot()

    local placeId  = tostring(game.PlaceId)
    local gameId   = tostring(game.GameId)
    local mapName  = "Unknown"
    pcall(function()
        local inf = MarketplaceService:GetProductInfo(game.PlaceId)
        if inf and inf.Name then mapName = inf.Name end
    end)

    local FILE = string.format("%s/%s.json", ROOT, placeId)
    local _cache = nil
    local _dirty = false
    local _debounce = false

    local function _load()
        if _cache then return _cache end
        local ok, txt = pcall(function()
            if FS.isfile(FILE) then return FS.readfile(FILE) end
            return nil
        end)
        local data = nil
        if ok and txt and #txt > 0 then
            local ok2, t = pcall(function() return HttpService:JSONDecode(txt) end)
            data = ok2 and t or nil
        end
        if not data or type(data)~="table" then
            data = { __meta = { placeId = placeId, gameId = gameId, mapName = mapName, savedAt = os.time() } }
        end
        _cache = data
        return _cache
    end

    local function _flushNow()
        if not _cache then return end
        _cache.__meta = _cache.__meta or {}
        _cache.__meta.placeId = placeId
        _cache.__meta.gameId  = gameId
        _cache.__meta.mapName = mapName
        _cache.__meta.savedAt = os.time()
        local ok, json = pcall(function() return HttpService:JSONEncode(_cache) end)
        if ok and json then
            pcall(function()
                safeMakeRoot()
                FS.writefile(FILE, json)
            end)
        end
        _dirty = false
    end

    local function _scheduleFlush()
        if _debounce then return end
        _debounce = true
        task.delay(0.25, function()
            _debounce = false
            if _dirty then _flushNow() end
        end)
    end

    local Save = {}

    -- อ่านค่า: key = "Tab.Key" เช่น "RJ.enabled" / "A1.Reduce" / "AFK.Black"
    function Save.get(key, defaultValue)
        local db = _load()
        local v = db[key]
        if v == nil then return defaultValue end
        return v
    end

    -- เซ็ตค่า + เขียนไฟล์แบบดีบาวซ์
    function Save.set(key, value)
        local db = _load()
        db[key] = value
        _dirty = true
        _scheduleFlush()
    end

    -- ตัวช่วย: apply ค่าเซฟถ้ามี ไม่งั้นใช้ default แล้วเซฟกลับ
    function Save.apply(key, defaultValue, applyFn)
        local v = Save.get(key, defaultValue)
        if applyFn then
            local ok = pcall(applyFn, v)
            if ok and v ~= nil then Save.set(key, v) end
        end
        return v
    end

    -- ให้เรียกใช้ที่อื่นได้
    getgenv().UFOX_SAVE = Save
end
-- ===== [/Local Save] =====
--[[
UFO HUB X • One-shot = Toast(2-step) + Main UI (100%)
- Step1: Toast โหลด + แถบเปอร์เซ็นต์
- Step2: Toast "ดาวน์โหลดเสร็จ" โผล่ "พร้อมกับ" UI หลัก แล้วเลือนหายเอง
]]

------------------------------------------------------------
-- 1) ห่อ "UI หลักของคุณ (เดิม 100%)" ไว้ในฟังก์ชัน _G.UFO_ShowMainUI()
------------------------------------------------------------
_G.UFO_ShowMainUI = function()

--[[
UFO HUB X • Main UI + Safe Toggle (one-shot paste)
- ไม่ลบปุ่ม Toggle อีกต่อไป (ลบเฉพาะ UI หลัก)
- Toggle อยู่ของตัวเอง, มีขอบเขียว, ลากได้, บล็อกกล้องตอนลาก
- ซิงก์สถานะกับ UI หลักอัตโนมัติ และรีบอินด์ทุกครั้งที่ UI ถูกสร้างใหม่
]]

local Players  = game:GetService("Players")
local CoreGui  = game:GetService("CoreGui")
local UIS      = game:GetService("UserInputService")
local CAS      = game:GetService("ContextActionService")
local TS       = game:GetService("TweenService")
local RunS     = game:GetService("RunService")

-- ===== Theme / Size =====
local THEME = {
    GREEN=Color3.fromRGB(0,255,140),
    MINT=Color3.fromRGB(120,255,220),
    BG_WIN=Color3.fromRGB(16,16,16),
    BG_HEAD=Color3.fromRGB(6,6,6),
    BG_PANEL=Color3.fromRGB(22,22,22),
    BG_INNER=Color3.fromRGB(18,18,18),
    TEXT=Color3.fromRGB(235,235,235),
    RED=Color3.fromRGB(200,40,40),
    HILITE=Color3.fromRGB(22,30,24),
}
local SIZE={WIN_W=640,WIN_H=360,RADIUS=12,BORDER=3,HEAD_H=46,GAP_OUT=14,GAP_IN=8,BETWEEN=12,LEFT_RATIO=0.22}
local IMG_UFO="rbxassetid://100650447103028"
local ICON_HOME   = 134323882016779
local ICON_QUEST   = 72473476254744
local ICON_SHOP     = 139824330037901
local ICON_UPDATE   = 134419329246667
local ICON_SETTINGS = 72289858646360
local TOGGLE_ICON = "rbxassetid://117052960049460"

local function corner(p,r) local u=Instance.new("UICorner",p) u.CornerRadius=UDim.new(0,r or 10) return u end
local function stroke(p,th,col,tr) local s=Instance.new("UIStroke",p) s.Thickness=th or 1 s.Color=col or THEME.MINT s.Transparency=tr or 0.35 s.ApplyStrokeMode=Enum.ApplyStrokeMode.Border s.LineJoinMode=Enum.LineJoinMode.Round return s end

-- ===== Utilities: find main UI + sync =====
local function findMain()
    local root = CoreGui:FindFirstChild("UFO_HUB_X_UI")
    if not root then
        local pg = Players.LocalPlayer and Players.LocalPlayer:FindFirstChild("PlayerGui")
        if pg then root = pg:FindFirstChild("UFO_HUB_X_UI") end
    end
    local win = root and (root:FindFirstChild("Win") or root:FindFirstChildWhichIsA("Frame")) or nil
    return root, win
end

local function setOpen(open)
    local gui, win = findMain()
    if gui then gui.Enabled = open end
    if win then win.Visible = open end
    getgenv().UFO_ISOPEN = not not open
end

-- ====== SAFE TOGGLE (สร้าง/รีใช้, ไม่โดนลบ) ======
local ToggleGui = CoreGui:FindFirstChild("UFO_HUB_X_Toggle") :: ScreenGui
if not ToggleGui then
    ToggleGui = Instance.new("ScreenGui")
    ToggleGui.Name = "UFO_HUB_X_Toggle"
    ToggleGui.IgnoreGuiInset = true
    ToggleGui.DisplayOrder = 100001
    ToggleGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    ToggleGui.ResetOnSpawn = false
    ToggleGui.Parent = CoreGui

    local Btn = Instance.new("ImageButton", ToggleGui)
    Btn.Name = "Button"
    Btn.Size = UDim2.fromOffset(64,64)
    Btn.Position = UDim2.fromOffset(90,220)
    Btn.Image = TOGGLE_ICON
    Btn.BackgroundColor3 = Color3.fromRGB(0,0,0)
    Btn.BorderSizePixel = 0
    corner(Btn,8); stroke(Btn,2,THEME.GREEN,0)

    -- drag + block camera
    local function block(on)
        local name="UFO_BlockLook_Toggle"
        if on then
            CAS:BindActionAtPriority(name,function() return Enum.ContextActionResult.Sink end,false,9000,
                Enum.UserInputType.MouseMovement,Enum.UserInputType.Touch,Enum.UserInputType.MouseButton1)
        else pcall(function() CAS:UnbindAction(name) end) end
    end
    local dragging=false; local start; local startPos
    Btn.InputBegan:Connect(function(i)
        if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch then
            dragging=true; start=i.Position; startPos=Vector2.new(Btn.Position.X.Offset, Btn.Position.Y.Offset); block(true)
            i.Changed:Connect(function() if i.UserInputState==Enum.UserInputState.End then dragging=false; block(false) end end)
        end
    end)
    UIS.InputChanged:Connect(function(i)
        if dragging and (i.UserInputType==Enum.UserInputType.MouseMovement or i.UserInputType==Enum.UserInputType.Touch) then
            local d=i.Position-start; Btn.Position=UDim2.fromOffset(startPos.X+d.X,startPos.Y+d.Y)
        end
    end)
end

-- (Re)bind toggle actions (กันผูกซ้ำ)
do
    local Btn = ToggleGui:FindFirstChild("Button")
    if getgenv().UFO_ToggleClick then pcall(function() getgenv().UFO_ToggleClick:Disconnect() end) end
    if getgenv().UFO_ToggleKey   then pcall(function() getgenv().UFO_ToggleKey:Disconnect() end) end
    getgenv().UFO_ToggleClick = Btn.MouseButton1Click:Connect(function() setOpen(not getgenv().UFO_ISOPEN) end)
    getgenv().UFO_ToggleKey   = UIS.InputBegan:Connect(function(i,gp) if gp then return end if i.KeyCode==Enum.KeyCode.RightShift then setOpen(not getgenv().UFO_ISOPEN) end end)
end

-- ====== ลบ "เฉพาะ" UI หลักเก่าก่อนสร้างใหม่ (ไม่ยุ่ง Toggle) ======
pcall(function() local old = CoreGui:FindFirstChild("UFO_HUB_X_UI"); if old then old:Destroy() end end)

-- ====== MAIN UI (เหมือนเดิม) ======
local GUI=Instance.new("ScreenGui")
GUI.Name="UFO_HUB_X_UI"
GUI.IgnoreGuiInset=true
GUI.ResetOnSpawn=false
GUI.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
GUI.DisplayOrder = 100000
GUI.Parent = CoreGui

local Win=Instance.new("Frame",GUI) Win.Name="Win"
Win.Size=UDim2.fromOffset(SIZE.WIN_W,SIZE.WIN_H)
Win.AnchorPoint=Vector2.new(0.5,0.5); Win.Position=UDim2.new(0.5,0,0.5,0)
Win.BackgroundColor3=THEME.BG_WIN; Win.BorderSizePixel=0
corner(Win,SIZE.RADIUS); stroke(Win,3,THEME.GREEN,0)

do local sc=Instance.new("UIScale",Win)
   local function fit() local v=workspace.CurrentCamera and workspace.CurrentCamera.ViewportSize or Vector2.new(1280,720)
       sc.Scale=math.clamp(math.min(v.X/860,v.Y/540),0.72,1.0) end
   fit(); RunS.RenderStepped:Connect(fit)
end

local Header=Instance.new("Frame",Win)
Header.Size=UDim2.new(1,0,0,SIZE.HEAD_H)
Header.BackgroundColor3=THEME.BG_HEAD; Header.BorderSizePixel=0
corner(Header,SIZE.RADIUS)
local Accent=Instance.new("Frame",Header)
Accent.AnchorPoint=Vector2.new(0.5,1); Accent.Position=UDim2.new(0.5,0,1,0)
Accent.Size=UDim2.new(1,-20,0,1); Accent.BackgroundColor3=THEME.MINT; Accent.BackgroundTransparency=0.35
local Title=Instance.new("TextLabel",Header)
Title.BackgroundTransparency=1; Title.AnchorPoint=Vector2.new(0.5,0)
Title.Position=UDim2.new(0.5,0,0,6); Title.Size=UDim2.new(0.8,0,0,36)
Title.Font=Enum.Font.GothamBold; Title.TextScaled=true; Title.RichText=true
Title.Text='<font color="#FFFFFF">UFO</font> <font color="#00FF8C">HUB X</font>'
Title.TextColor3=THEME.TEXT

local BtnClose=Instance.new("TextButton",Header)
BtnClose.AutoButtonColor=false; BtnClose.Size=UDim2.fromOffset(24,24)
BtnClose.Position=UDim2.new(1,-34,0.5,-12); BtnClose.BackgroundColor3=THEME.RED
BtnClose.Text="X"; BtnClose.Font=Enum.Font.GothamBold; BtnClose.TextSize=13
BtnClose.TextColor3=Color3.new(1,1,1); BtnClose.BorderSizePixel=0
corner(BtnClose,6); stroke(BtnClose,1,Color3.fromRGB(255,0,0),0.1)
BtnClose.MouseButton1Click:Connect(function() setOpen(false) end)

-- UFO icon
local UFO=Instance.new("ImageLabel",Win)
UFO.BackgroundTransparency=1; UFO.Image=IMG_UFO
UFO.Size=UDim2.fromOffset(168,168); UFO.AnchorPoint=Vector2.new(0.5,1)
UFO.Position=UDim2.new(0.5,0,0,84); UFO.ZIndex=4

-- === DRAG MAIN ONLY (ลากได้เฉพาะ UI หลักที่ Header; บล็อกกล้องระหว่างลาก) ===
do
    local dragging = false
    local startInputPos: Vector2
    local startWinOffset: Vector2
    local blockDrag = false

    -- กันเผลอลากตอนกดปุ่ม X
    BtnClose.MouseButton1Down:Connect(function() blockDrag = true end)
    BtnClose.MouseButton1Up:Connect(function() blockDrag = false end)

    local function blockCamera(on: boolean)
        local name = "UFO_BlockLook_MainDrag"
        if on then
            CAS:BindActionAtPriority(name, function()
                return Enum.ContextActionResult.Sink
            end, false, 9000,
            Enum.UserInputType.MouseMovement,
            Enum.UserInputType.Touch,
            Enum.UserInputType.MouseButton1)
        else
            pcall(function() CAS:UnbindAction(name) end)
        end
    end

    Header.InputBegan:Connect(function(input)
        if blockDrag then return end
        if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            startInputPos  = input.Position
            startWinOffset = Vector2.new(Win.Position.X.Offset, Win.Position.Y.Offset)
            blockCamera(true)
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                    blockCamera(false)
                end
            end)
        end
    end)

    UIS.InputChanged:Connect(function(input)
        if not dragging then return end
        if input.UserInputType ~= Enum.UserInputType.MouseMovement
        and input.UserInputType ~= Enum.UserInputType.Touch then return end
        local delta = input.Position - startInputPos
        Win.Position = UDim2.new(0.5, startWinOffset.X + delta.X, 0.5, startWinOffset.Y + delta.Y)
    end)
end
-- === END DRAG MAIN ONLY ===

-- BODY
local Body=Instance.new("Frame",Win)
Body.BackgroundColor3=THEME.BG_INNER; Body.BorderSizePixel=0
Body.Position=UDim2.new(0,SIZE.GAP_OUT,0,SIZE.HEAD_H+SIZE.GAP_OUT)
Body.Size=UDim2.new(1,-SIZE.GAP_OUT*2,1,-(SIZE.HEAD_H+SIZE.GAP_OUT*2))
corner(Body,12); stroke(Body,0.5,THEME.MINT,0.35)

-- === LEFT (แทนที่บล็อกก่อนหน้าได้เลย) ================================
local LeftShell = Instance.new("Frame", Body)
LeftShell.BackgroundColor3 = THEME.BG_PANEL
LeftShell.BorderSizePixel  = 0
LeftShell.Position         = UDim2.new(0, SIZE.GAP_IN, 0, SIZE.GAP_IN)
LeftShell.Size             = UDim2.new(SIZE.LEFT_RATIO, -(SIZE.BETWEEN/2), 1, -SIZE.GAP_IN*2)
LeftShell.ClipsDescendants = true
corner(LeftShell, 10)
stroke(LeftShell, 1.2, THEME.GREEN, 0)
stroke(LeftShell, 0.45, THEME.MINT, 0.35)

local LeftScroll = Instance.new("ScrollingFrame", LeftShell)
LeftScroll.BackgroundTransparency = 1
LeftScroll.Size                   = UDim2.fromScale(1,1)
LeftScroll.ScrollBarThickness     = 0
LeftScroll.ScrollingDirection     = Enum.ScrollingDirection.Y
LeftScroll.AutomaticCanvasSize    = Enum.AutomaticSize.None
LeftScroll.ElasticBehavior        = Enum.ElasticBehavior.Never
LeftScroll.ScrollingEnabled       = true
LeftScroll.ClipsDescendants       = true

local padL = Instance.new("UIPadding", LeftScroll)
padL.PaddingTop    = UDim.new(0, 8)
padL.PaddingLeft   = UDim.new(0, 8)
padL.PaddingRight  = UDim.new(0, 8)
padL.PaddingBottom = UDim.new(0, 8)

local LeftList = Instance.new("UIListLayout", LeftScroll)
LeftList.Padding   = UDim.new(0, 8)
LeftList.SortOrder = Enum.SortOrder.LayoutOrder

-- ===== คุม Canvas + กันเด้งกลับตอนคลิกแท็บ =====
local function refreshLeftCanvas()
    local contentH = LeftList.AbsoluteContentSize.Y + padL.PaddingTop.Offset + padL.PaddingBottom.Offset
    LeftScroll.CanvasSize = UDim2.new(0, 0, 0, contentH)
end

local function clampTo(yTarget)
    local contentH = LeftList.AbsoluteContentSize.Y + padL.PaddingTop.Offset + padL.PaddingBottom.Offset
    local viewH    = LeftScroll.AbsoluteSize.Y
    local maxY     = math.max(0, contentH - viewH)
    LeftScroll.CanvasPosition = Vector2.new(0, math.clamp(yTarget or 0, 0, maxY))
end

-- ✨ จำตำแหน่งล่าสุดไว้ใช้ “ทุกครั้ง” ที่มีการจัดเลย์เอาต์ใหม่
local lastY = 0

LeftList:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
    refreshLeftCanvas()
    clampTo(lastY) -- ใช้ค่าเดิมที่จำไว้ ไม่อ่านจาก CanvasPosition ที่อาจโดนรีเซ็ต
end)

task.defer(refreshLeftCanvas)

-- name/icon = ชื่อ/ไอคอนฝั่งขวา, setFns = ฟังก์ชันเซ็ต active, btn = ปุ่มที่ถูกกด
local function onTabClick(name, icon, setFns, btn)
    -- บันทึกตำแหน่งปัจจุบัน “ไว้ก่อน” ที่เลย์เอาต์จะขยับ
    lastY = LeftScroll.CanvasPosition.Y

    setFns()
    showRight(name, icon)

    task.defer(function()
        refreshLeftCanvas()
        clampTo(lastY) -- คืนตำแหน่งเดิมเสมอ

        -- ถ้าปุ่มอยู่นอกจอ ค่อยเลื่อนเข้าเฟรมอย่างพอดี (จะปรับ lastY ด้วย)
        if btn and btn.Parent then
            local viewH   = LeftScroll.AbsoluteSize.Y
            local btnTop  = btn.AbsolutePosition.Y - LeftScroll.AbsolutePosition.Y
            local btnBot  = btnTop + btn.AbsoluteSize.Y
            local pad     = 8
            local y = LeftScroll.CanvasPosition.Y
            if btnTop < 0 then
                y = y + (btnTop - pad)
            elseif btnBot > viewH then
                y = y + (btnBot - viewH) + pad
            end
            lastY = y
            clampTo(lastY)
        end
    end)
end

-- === ผูกคลิกแท็บทั้ง 7 (เหมือนเดิม) ================================
task.defer(function()
    repeat task.wait() until
        btnHome and btnQuest and btnShop and btnSettings
  

   btnHome.MouseButton1Click:Connect(function()
        onTabClick("Home", ICON_HOME, function()
            setHomeActive(true); setQuestActive(false)
            setShopActive(false); setSettingsActive(false)
        end, btnHome)
    end)

    btnQuest.MouseButton1Click:Connect(function()
        onTabClick("Quest", ICON_QUEST, function()
            setHomeActive(false); setQuestActive(true)
            setShopActive(false); setSettingsActive(false)
        end, btnQuest)
    end)

    btnShop.MouseButton1Click:Connect(function()
        onTabClick("Shop", ICON_SHOP, function()
            setHomeActive(false); setQuestActive(false)
            setShopActive(true); setSettingsActive(false)
        end, btnShop)
    end) 

    btnSettings.MouseButton1Click:Connect(function()
        onTabClick("Settings", ICON_SETTINGS, function()
            setHomeActive(false); setQuestActive(false)
            setShopActive(false); setSettingsActive(true)
        end, btnSettings)
    end)
end)
-- ===================================================================

----------------------------------------------------------------
-- LEFT (ปุ่มแท็บ) + RIGHT (คอนเทนต์) — เวอร์ชันครบ + แก้บัคสกอร์ลแยกแท็บ
----------------------------------------------------------------

-- ========== LEFT ==========
local LeftShell=Instance.new("Frame",Body)
LeftShell.BackgroundColor3=THEME.BG_PANEL; LeftShell.BorderSizePixel=0
LeftShell.Position=UDim2.new(0,SIZE.GAP_IN,0,SIZE.GAP_IN)
LeftShell.Size=UDim2.new(SIZE.LEFT_RATIO,-(SIZE.BETWEEN/2),1,-SIZE.GAP_IN*2)
LeftShell.ClipsDescendants=true
corner(LeftShell,10); stroke(LeftShell,1.2,THEME.GREEN,0); stroke(LeftShell,0.45,THEME.MINT,0.35)

local LeftScroll=Instance.new("ScrollingFrame",LeftShell)
LeftScroll.BackgroundTransparency=1
LeftScroll.Size=UDim2.fromScale(1,1)
LeftScroll.ScrollBarThickness=0
LeftScroll.ScrollingDirection=Enum.ScrollingDirection.Y
LeftScroll.AutomaticCanvasSize=Enum.AutomaticSize.None
LeftScroll.ElasticBehavior=Enum.ElasticBehavior.Never
LeftScroll.ScrollingEnabled=true
LeftScroll.ClipsDescendants=true

local padL=Instance.new("UIPadding",LeftScroll)
padL.PaddingTop=UDim.new(0,8); padL.PaddingLeft=UDim.new(0,8); padL.PaddingRight=UDim.new(0,8); padL.PaddingBottom=UDim.new(0,8)
local LeftList=Instance.new("UIListLayout",LeftScroll); LeftList.Padding=UDim.new(0,8); LeftList.SortOrder=Enum.SortOrder.LayoutOrder

local function refreshLeftCanvas()
    local contentH = LeftList.AbsoluteContentSize.Y + padL.PaddingTop.Offset + padL.PaddingBottom.Offset
    LeftScroll.CanvasSize = UDim2.new(0,0,0,contentH)
end
local lastLeftY = 0
LeftList:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
    refreshLeftCanvas()
    local viewH = LeftScroll.AbsoluteSize.Y
    local maxY  = math.max(0, LeftScroll.CanvasSize.Y.Offset - viewH)
    LeftScroll.CanvasPosition = Vector2.new(0, math.clamp(lastLeftY,0,maxY))
end)
task.defer(refreshLeftCanvas)

-- สร้างปุ่มแท็บ
local function makeTabButton(parent, label, iconId)
    local holder = Instance.new("Frame", parent) holder.BackgroundTransparency=1 holder.Size = UDim2.new(1,0,0,38)
    local b = Instance.new("TextButton", holder) b.AutoButtonColor=false b.Text="" b.Size=UDim2.new(1,0,1,0) b.BackgroundColor3=THEME.BG_INNER corner(b,8)
    local st = stroke(b,1,THEME.MINT,0.35)
    local ic = Instance.new("ImageLabel", b) ic.BackgroundTransparency=1 ic.Image="rbxassetid://"..tostring(iconId) ic.Size=UDim2.fromOffset(22,22) ic.Position=UDim2.new(0,10,0.5,-11)
    local tx = Instance.new("TextLabel", b) tx.BackgroundTransparency=1 tx.TextColor3=THEME.TEXT tx.Font=Enum.Font.GothamMedium tx.TextSize=15 tx.TextXAlignment=Enum.TextXAlignment.Left tx.Position=UDim2.new(0,38,0,0) tx.Size=UDim2.new(1,-46,1,0) tx.Text = label
    local flash=Instance.new("Frame",b) flash.BackgroundColor3=THEME.GREEN flash.BackgroundTransparency=1 flash.BorderSizePixel=0 flash.AnchorPoint=Vector2.new(0.5,0.5) flash.Position=UDim2.new(0.5,0,0.5,0) flash.Size=UDim2.new(0,0,0,0) corner(flash,12)
    b.MouseButton1Down:Connect(function() TS:Create(b, TweenInfo.new(0.08, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {Size = UDim2.new(1,0,1,-2)}):Play() end)
    b.MouseButton1Up:Connect(function() TS:Create(b, TweenInfo.new(0.10, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Size = UDim2.new(1,0,1,0)}):Play() end)
    local function setActive(on)
        if on then
            b.BackgroundColor3=THEME.HILITE; st.Color=THEME.GREEN; st.Transparency=0; st.Thickness=2
            flash.BackgroundTransparency=0.35; flash.Size=UDim2.new(0,0,0,0)
            TS:Create(flash, TweenInfo.new(0.18, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {Size=UDim2.new(1,0,1,0), BackgroundTransparency=1}):Play()
        else
            b.BackgroundColor3=THEME.BG_INNER; st.Color=THEME.MINT; st.Transparency=0.35; st.Thickness=1
        end
    end
    return b, setActive
end

local btnHome,    setHomeActive     = makeTabButton(LeftScroll, "Home",    ICON_HOME)
local btnQuest,   setQuestActive    = makeTabButton(LeftScroll, "Event",   ICON_QUEST)
local btnShop,    setShopActive     = makeTabButton(LeftScroll, "Shop",    ICON_SHOP)
local btnSettings,setSettingsActive = makeTabButton(LeftScroll, "Settings",ICON_SETTINGS)

-- ========== RIGHT ==========
local RightShell=Instance.new("Frame",Body)
RightShell.BackgroundColor3=THEME.BG_PANEL; RightShell.BorderSizePixel=0
RightShell.Position=UDim2.new(SIZE.LEFT_RATIO,SIZE.BETWEEN,0,SIZE.GAP_IN)
RightShell.Size=UDim2.new(1-SIZE.LEFT_RATIO,-SIZE.GAP_IN-SIZE.BETWEEN,1,-SIZE.GAP_IN*2)
corner(RightShell,10); stroke(RightShell,1.2,THEME.GREEN,0); stroke(RightShell,0.45,THEME.MINT,0.35)

local RightScroll=Instance.new("ScrollingFrame",RightShell)
RightScroll.BackgroundTransparency=1; RightScroll.Size=UDim2.fromScale(1,1)
RightScroll.ScrollBarThickness=0; RightScroll.ScrollingDirection=Enum.ScrollingDirection.Y
RightScroll.AutomaticCanvasSize=Enum.AutomaticSize.None   -- คุมเองเพื่อกันเด้ง/จำ Y ได้
RightScroll.ElasticBehavior=Enum.ElasticBehavior.Never

local padR=Instance.new("UIPadding",RightScroll)
padR.PaddingTop=UDim.new(0,12); padR.PaddingLeft=UDim.new(0,12); padR.PaddingRight=UDim.new(0,12); padR.PaddingBottom=UDim.new(0,12)
local RightList=Instance.new("UIListLayout",RightScroll); RightList.Padding=UDim.new(0,10); RightList.SortOrder = Enum.SortOrder.LayoutOrder

local function refreshRightCanvas()
    local contentH = RightList.AbsoluteContentSize.Y + padR.PaddingTop.Offset + padR.PaddingBottom.Offset
    RightScroll.CanvasSize = UDim2.new(0,0,0,contentH)
end
RightList:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
    local yBefore = RightScroll.CanvasPosition.Y
    refreshRightCanvas()
    local viewH = RightScroll.AbsoluteSize.Y
    local maxY  = math.max(0, RightScroll.CanvasSize.Y.Offset - viewH)
    RightScroll.CanvasPosition = Vector2.new(0, math.clamp(yBefore,0,maxY))
end)
-- ================= RIGHT: Modular per-tab (drop-in) =================
-- ใส่หลังจากสร้าง RightShell เสร็จ (และก่อนผูกปุ่มกด)

-- 1) เก็บ/ใช้ state กลาง
if not getgenv().UFO_RIGHT then getgenv().UFO_RIGHT = {} end
local RSTATE = getgenv().UFO_RIGHT
RSTATE.frames   = RSTATE.frames   or {}
RSTATE.builders = RSTATE.builders or {}
RSTATE.scrollY  = RSTATE.scrollY  or {}
RSTATE.current  = RSTATE.current

-- 2) ถ้ามี RightScroll เก่าอยู่ ให้ลบทิ้ง
pcall(function()
    local old = RightShell:FindFirstChildWhichIsA("ScrollingFrame")
    if old then old:Destroy() end
end)

-- 3) สร้าง ScrollingFrame ต่อแท็บ
local function makeTabFrame(tabName)
    local root = Instance.new("Frame")
    root.Name = "RightTab_"..tabName
    root.BackgroundTransparency = 1
    root.Size = UDim2.fromScale(1,1)
    root.Visible = false
    root.Parent = RightShell

    local sf = Instance.new("ScrollingFrame", root)
    sf.Name = "Scroll"
    sf.BackgroundTransparency = 1
    sf.Size = UDim2.fromScale(1,1)
    sf.ScrollBarThickness = 0      -- ← ซ่อนสกรอลล์บาร์ (เดิม 4)
    sf.ScrollingDirection = Enum.ScrollingDirection.Y
    sf.AutomaticCanvasSize = Enum.AutomaticSize.None
    sf.ElasticBehavior = Enum.ElasticBehavior.Never
    sf.CanvasSize = UDim2.new(0,0,0,600)  -- เลื่อนได้ตั้งแต่เริ่ม

    local pad = Instance.new("UIPadding", sf)
    pad.PaddingTop    = UDim.new(0,12)
    pad.PaddingLeft   = UDim.new(0,12)
    pad.PaddingRight  = UDim.new(0,12)
    pad.PaddingBottom = UDim.new(0,12)

    local list = Instance.new("UIListLayout", sf)
    list.Padding = UDim.new(0,10)
    list.SortOrder = Enum.SortOrder.LayoutOrder
    list.VerticalAlignment = Enum.VerticalAlignment.Top

    local function refreshCanvas()
        local h = list.AbsoluteContentSize.Y + pad.PaddingTop.Offset + pad.PaddingBottom.Offset
        sf.CanvasSize = UDim2.new(0,0,0, math.max(h,600))
    end

    list:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
        local yBefore = sf.CanvasPosition.Y
        refreshCanvas()
        local viewH = sf.AbsoluteSize.Y
        local maxY  = math.max(0, sf.CanvasSize.Y.Offset - viewH)
        sf.CanvasPosition = Vector2.new(0, math.clamp(yBefore, 0, maxY))
    end)

    task.defer(refreshCanvas)

    RSTATE.frames[tabName] = {root=root, scroll=sf, list=list, built=false}
    return RSTATE.frames[tabName]
end

-- 4) ลงทะเบียนฟังก์ชันสร้างคอนเทนต์ต่อแท็บ (รองรับหลายตัว)
local function registerRight(tabName, builderFn)
    RSTATE.builders[tabName] = RSTATE.builders[tabName] or {}
    table.insert(RSTATE.builders[tabName], builderFn)
end

-- 5) หัวเรื่อง
local function addHeader(parentScroll, titleText, iconId)
    local row = Instance.new("Frame")
    row.BackgroundTransparency = 1
    row.Size = UDim2.new(1,0,0,28)
    row.Parent = parentScroll

    local icon = Instance.new("ImageLabel", row)
    icon.BackgroundTransparency = 1
    icon.Image = "rbxassetid://"..tostring(iconId or "")
    icon.Size = UDim2.fromOffset(20,20)
    icon.Position = UDim2.new(0,0,0.5,-10)

    local head = Instance.new("TextLabel", row)
    head.BackgroundTransparency = 1
    head.Font = Enum.Font.GothamBold
    head.TextSize = 18
    head.TextXAlignment = Enum.TextXAlignment.Left
    head.TextColor3 = THEME.TEXT
    head.Position = UDim2.new(0,26,0,0)
    head.Size = UDim2.new(1,-26,1,0)
    head.Text = titleText
end

------------------------------------------------------------
-- 6) API หลัก + แปลชื่อหัวข้อเป็นภาษาไทย
------------------------------------------------------------

-- map ชื่อแท็บ (key ภาษาอังกฤษด้านใน) -> หัวข้อภาษาไทยที่โชว์
local TAB_TITLE_TH = {
    Quest    = "Event",
    
}

function showRight(tabKey, iconId)
    -- tabKey = key ภาษาอังกฤษ ("Player","Home","Settings",...)
    local tab = tabKey
    -- ข้อความที่โชว์บนหัวข้อ ใช้ภาษาไทย ถ้ามีในตาราง ไม่มีก็ใช้อังกฤษเดิม
    local titleText = TAB_TITLE_TH[tabKey] or tabKey

    if RSTATE.current and RSTATE.frames[RSTATE.current] then
        RSTATE.scrollY[RSTATE.current] = RSTATE.frames[RSTATE.current].scroll.CanvasPosition.Y
        RSTATE.frames[RSTATE.current].root.Visible = false
    end

    local f = RSTATE.frames[tab] or makeTabFrame(tab)
    f.root.Visible = true

    if not f.built then
        -- ตรงนี้ใช้ titleText (ไทย) สำหรับหัวข้อ
        addHeader(f.scroll, titleText, iconId)

        local list = RSTATE.builders[tab] or {}
        for _, builder in ipairs(list) do
            pcall(builder, f.scroll)
        end
        f.built = true
    end

    task.defer(function()
        local y = RSTATE.scrollY[tab] or 0
        local viewH = f.scroll.AbsoluteSize.Y
        local maxY  = math.max(0, f.scroll.CanvasSize.Y.Offset - viewH)
        f.scroll.CanvasPosition = Vector2.new(0, math.clamp(y, 0, maxY))
    end)

    RSTATE.current = tab
end
    
-- 7) ตัวอย่างแท็บ (ลบเดโมรายการออกแล้ว)
registerRight("Home", function(scroll)
    -- วาง UI ของ Player ที่นี่ (ตอนนี้ปล่อยว่าง ไม่มี Item#)
end)

registerRight("Home", function(scroll) end)
registerRight("Quest", function(scroll) end)
registerRight("Shop", function(scroll) end)
registerRight("Settings", function(scroll) end)
 --===== UFO HUB X • Home – Auto Farm (Model A V1 + AA1 + Global Auto-Run + SYNC DETECT) [HARD FIX DESYNC v2] =====
-- Row1: Auto Mine  -> FireServer("Toggle Setting","AutoMine")
-- Row2: Auto Train -> FireServer("Toggle Setting","AutoTrain")
-- ✅ เปิดได้ทีละอัน + เซฟ AA1 + Auto-Run ตอนโหลดสคริปต์
-- ✅ SYNC ตรวจจับ “คนอื่นยิง Toggle Setting” แล้วอัปเดตสวิตช์เราให้ตรงกัน
-- ✅ FIX v2: กันอาการ “ระบบทำงานแต่สวิตช์โชว์ปิด / เปิดกลายเป็นปิด” (multi-hook / double flip / stale-local-on)

---------------------------------------------------------------------
-- 1) AA1 GLOBAL + SYNC DETECT (รันทันทีที่โหลดสคริปต์)
---------------------------------------------------------------------
do
    local ReplicatedStorage = game:GetService("ReplicatedStorage")

    -- BUS กลาง: กัน hook เก่า/หลายตัว “ตีความ FireServer ของเราเป็น external”
    _G.UFOX_BUS = _G.UFOX_BUS or {}
    _G.UFOX_BUS.InternalToggle = _G.UFOX_BUS.InternalToggle or false

    local SAVE = (getgenv and getgenv().UFOX_SAVE) or {
        get = function(_, _, d) return d end,
        set = function() end,
    }

    local GAME_ID  = tonumber(game.GameId)  or 0
    local PLACE_ID = tonumber(game.PlaceId) or 0
    local BASE_SCOPE = ("AA1/HomeAutoFarm/%d/%d"):format(GAME_ID, PLACE_ID)

    local function K(field) return BASE_SCOPE .. "/" .. field end

    local function SaveGet(field, default)
        local ok, v = pcall(function()
            return SAVE.get(K(field), default)
        end)
        return ok and v or default
    end

    local function SaveSet(field, value)
        pcall(function()
            SAVE.set(K(field), value)
        end)
    end

    local function getEvt()
        local paper   = ReplicatedStorage:WaitForChild("Paper")
        local remotes = paper:WaitForChild("Remotes")
        return remotes:WaitForChild("__remoteevent")
    end

    local EVT = nil
    local function ensureEvt()
        if EVT and EVT.Parent then return EVT end
        local ok, ev = pcall(getEvt)
        if ok then EVT = ev end
        return EVT
    end

    -- โหลด/ใช้ instance เดิมถ้ามี (กันซ้อนหลายสคริปต์)
    _G.UFOX_AA1 = _G.UFOX_AA1 or {}
    _G.UFOX_AA1["HomeAutoFarm"] = _G.UFOX_AA1["HomeAutoFarm"] or {}
    local SYS = _G.UFOX_AA1["HomeAutoFarm"]

    SYS._VERSION = "HARD_FIX_V2"

    SYS.state = SYS.state or {
        AutoMine  = (SaveGet("AutoMine",  false) == true),
        AutoTrain = (SaveGet("AutoTrain", false) == true),
    }

    local STATE = SYS.state

    -- กันเซฟหลุดเป็น true ทั้งคู่
    if STATE.AutoMine and STATE.AutoTrain then
        STATE.AutoTrain = false
        SaveSet("AutoTrain", false)
    end

    -- UI sync hook (ถูก bind จากฝั่ง UI)
    SYS._UI_SYNC = SYS._UI_SYNC or {
        setMine  = function(_) end,
        setTrain = function(_) end,
    }

    -- ยิง toggle แบบ “global-safe”
    local function fireToggle(settingName)
        local ok, err = pcall(function()
            local ev = ensureEvt()
            if not ev then return end

            _G.UFOX_BUS.InternalToggle = true
            ev:FireServer("Toggle Setting", settingName)

            -- หน่วงนิดเดียว กัน hook หลายตัวจับไม่ทัน
            task.delay(0.06, function()
                _G.UFOX_BUS.InternalToggle = false
            end)
        end)
        if not ok then
            _G.UFOX_BUS.InternalToggle = false
            warn("[UFO HUB X • HomeAutoFarm] Toggle error:", settingName, err)
        end
    end

    -- ตั้งค่าให้เป็น ON/OFF แน่นอน (ถ้าต่างจริงค่อยยิง Toggle)
    local function setOne(name, wantOn, doFire)
        wantOn = (wantOn == true)
        if STATE[name] == wantOn then return end
        STATE[name] = wantOn
        SaveSet(name, wantOn)
        if doFire ~= false then
            fireToggle(name)
        end
    end

    local function setMine(on, doFire)
        on = (on == true)
        if on then
            if STATE.AutoTrain then
                setOne("AutoTrain", false, doFire)
                SYS._UI_SYNC.setTrain(false)
            end
            setOne("AutoMine", true, doFire)
            SYS._UI_SYNC.setMine(true)
        else
            setOne("AutoMine", false, doFire)
            SYS._UI_SYNC.setMine(false)
        end
    end

    local function setTrain(on, doFire)
        on = (on == true)
        if on then
            if STATE.AutoMine then
                setOne("AutoMine", false, doFire)
                SYS._UI_SYNC.setMine(false)
            end
            setOne("AutoTrain", true, doFire)
            SYS._UI_SYNC.setTrain(true)
        else
            setOne("AutoTrain", false, doFire)
            SYS._UI_SYNC.setTrain(false)
        end
    end

    ---------------------------------------------------------------------
    -- DETECT: external toggle -> flip จาก STATE ที่เรารู้ล่าสุด (ข้ามถ้าเป็น internal ของเรา)
    ---------------------------------------------------------------------
    SYS._DETECT_READY = (SYS._DETECT_READY == true)

    local function onExternalToggle(settingName)
        if _G.UFOX_BUS.InternalToggle then return end
        if settingName ~= "AutoMine" and settingName ~= "AutoTrain" then return end

        local newOn = not (STATE[settingName] == true)

        if settingName == "AutoMine" then
            if newOn and STATE.AutoTrain then
                STATE.AutoTrain = false
                SaveSet("AutoTrain", false)
                SYS._UI_SYNC.setTrain(false)
            end
            STATE.AutoMine = newOn
            SaveSet("AutoMine", newOn)
            SYS._UI_SYNC.setMine(newOn)

        elseif settingName == "AutoTrain" then
            if newOn and STATE.AutoMine then
                STATE.AutoMine = false
                SaveSet("AutoMine", false)
                SYS._UI_SYNC.setMine(false)
            end
            STATE.AutoTrain = newOn
            SaveSet("AutoTrain", newOn)
            SYS._UI_SYNC.setTrain(newOn)
        end
    end

    local function setupDetect()
        if SYS._DETECT_READY then return end
        local ev = ensureEvt()
        if not ev then return end

        local ok = pcall(function()
            if typeof(hookmetamethod) ~= "function" then return end

            local old
            old = hookmetamethod(game, "__namecall", function(self, ...)
                local method = getnamecallmethod and getnamecallmethod() or ""
                if self == ev and method == "FireServer" then
                    local a1, a2 = ...
                    if a1 == "Toggle Setting" and (a2 == "AutoMine" or a2 == "AutoTrain") then
                        onExternalToggle(a2)
                    end
                end
                return old(self, ...)
            end)

            SYS._DETECT_READY = true
        end)

        if not ok then
            -- no warn
        end
    end

    -- export API
    SYS.setMine  = function(on) setMine(on, true) end
    SYS.setTrain = function(on) setTrain(on, true) end
    SYS.getMine  = function() return STATE.AutoMine  == true end
    SYS.getTrain = function() return STATE.AutoTrain == true end
    SYS._uiBind  = function(bind)
        if type(bind) == "table" then
            if type(bind.setMine)  == "function" then SYS._UI_SYNC.setMine  = bind.setMine end
            if type(bind.setTrain) == "function" then SYS._UI_SYNC.setTrain = bind.setTrain end
        end
    end
    SYS._setupDetect = setupDetect

    -- AUTO-RUN ตามค่าเซฟทันที (ไม่ต้องกด Home)
    task.defer(function()
        setupDetect()
        if STATE.AutoMine then
            fireToggle("AutoMine")
        elseif STATE.AutoTrain then
            fireToggle("AutoTrain")
        end
    end)
end

---------------------------------------------------------------------
-- 2) UI ฝั่งขวา (Model A V1) สำหรับแท็บ Home
---------------------------------------------------------------------
registerRight("Home", function(scroll)
    local TweenService = game:GetService("TweenService")

    local THEME = {
        GREEN = Color3.fromRGB(25,255,125),
        RED   = Color3.fromRGB(255,40,40),
        WHITE = Color3.fromRGB(255,255,255),
        BLACK = Color3.fromRGB(0,0,0),
    }

    local function corner(ui, r)
        local c = Instance.new("UICorner")
        c.CornerRadius = UDim.new(0, r or 12)
        c.Parent = ui
        return c
    end

    local function stroke(ui, th, col)
        local s = Instance.new("UIStroke")
        s.Thickness = th or 2.2
        s.Color = col or THEME.GREEN
        s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
        s.Parent = ui
        return s
    end

    local function tween(o, p, d)
        TweenService:Create(
            o,
            TweenInfo.new(d or 0.08, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
            p
        ):Play()
    end

    local AA1 = _G.UFOX_AA1 and _G.UFOX_AA1["HomeAutoFarm"]
    local STATE = (AA1 and AA1.state) or { AutoMine=false, AutoTrain=false }

    ------------------------------------------------------------------------
    -- CLEANUP (กันซ้อน)
    ------------------------------------------------------------------------
    for _, name in ipairs({ "HAF_Header", "HAF_Row1", "HAF_Row2" }) do
        local o = scroll:FindFirstChild(name)
            or scroll.Parent:FindFirstChild(name)
            or (scroll:FindFirstAncestorOfClass("ScreenGui")
                and scroll:FindFirstAncestorOfClass("ScreenGui"):FindFirstChild(name))
        if o then o:Destroy() end
    end

    ------------------------------------------------------------------------
    -- UIListLayout (A V1: 1 layout + dynamic base)
    ------------------------------------------------------------------------
    local vlist = scroll:FindFirstChildOfClass("UIListLayout")
    if not vlist then
        vlist = Instance.new("UIListLayout")
        vlist.Parent = scroll
        vlist.Padding   = UDim.new(0, 12)
        vlist.SortOrder = Enum.SortOrder.LayoutOrder
    end
    scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y

    local base = 0
    for _, ch in ipairs(scroll:GetChildren()) do
        if ch:IsA("GuiObject") and ch ~= vlist then
            base = math.max(base, ch.LayoutOrder or 0)
        end
    end

    ------------------------------------------------------------------------
    -- HEADER
    ------------------------------------------------------------------------
    local header = Instance.new("TextLabel")
    header.Name = "HAF_Header"
    header.Parent = scroll
    header.BackgroundTransparency = 1
    header.Size = UDim2.new(1, 0, 0, 36)
    header.Font = Enum.Font.GothamBold
    header.TextSize = 16
    header.TextColor3 = THEME.WHITE
    header.TextXAlignment = Enum.TextXAlignment.Left
    header.Text = "》》》Auto Farm 🦾《《《"
    header.LayoutOrder = base + 1

    ------------------------------------------------------------------------
    -- Base Row (A V1 card)
    ------------------------------------------------------------------------
    local function makeRow(name, order, labelText)
        local row = Instance.new("Frame")
        row.Name = name
        row.Parent = scroll
        row.Size = UDim2.new(1, -6, 0, 46)
        row.BackgroundColor3 = THEME.BLACK
        corner(row, 12)
        stroke(row, 2.2, THEME.GREEN)
        row.LayoutOrder = order

        local lab = Instance.new("TextLabel")
        lab.Parent = row
        lab.BackgroundTransparency = 1
        lab.Size = UDim2.new(1, -160, 1, 0)
        lab.Position = UDim2.new(0, 16, 0, 0)
        lab.Font = Enum.Font.GothamBold
        lab.TextSize = 13
        lab.TextColor3 = THEME.WHITE
        lab.TextXAlignment = Enum.TextXAlignment.Left
        lab.Text = labelText

        return row
    end

    ------------------------------------------------------------------------
    -- A V1 Switch helper (TRUE SOURCE = AA1.STATE) + cooldown กัน double click
    ------------------------------------------------------------------------
    local function makeAV1Switch(parentRow, initialOn, onRequest)
        local sw = Instance.new("Frame")
        sw.Parent = parentRow
        sw.AnchorPoint = Vector2.new(1,0.5)
        sw.Position = UDim2.new(1, -12, 0.5, 0)
        sw.Size = UDim2.fromOffset(52,26)
        sw.BackgroundColor3 = THEME.BLACK
        corner(sw, 13)

        local swStroke = Instance.new("UIStroke")
        swStroke.Parent = sw
        swStroke.Thickness = 1.8

        local knob = Instance.new("Frame")
        knob.Parent = sw
        knob.Size = UDim2.fromOffset(22,22)
        knob.BackgroundColor3 = THEME.WHITE
        corner(knob, 11)

        local btn = Instance.new("TextButton")
        btn.Parent = sw
        btn.BackgroundTransparency = 1
        btn.Size = UDim2.fromScale(1,1)
        btn.Text = ""
        btn.AutoButtonColor = false

        local on = (initialOn == true)
        local busy = false

        local function update()
            swStroke.Color = on and THEME.GREEN or THEME.RED
            tween(knob, {Position = UDim2.new(on and 1 or 0, on and -24 or 2, 0.5, -11)}, 0.08)
        end

        btn.MouseButton1Click:Connect(function()
            if busy then return end
            busy = true

            local wantOn = not on
            -- โชว์ทันที กัน “กดแล้วไม่เปลี่ยน”
            on = wantOn
            update()

            if onRequest then onRequest(wantOn) end

            task.delay(0.12, function()
                busy = false
            end)
        end)

        update()

        return {
            set = function(v) on = (v == true); update() end,
            get = function() return on end,
        }
    end

    local row1 = makeRow("HAF_Row1", base + 2, "Auto Mine")
    local row2 = makeRow("HAF_Row2", base + 3, "Auto Train")

    local swMine, swTrain

    swMine = makeAV1Switch(row1, (AA1 and AA1.getMine and AA1.getMine()) or (STATE.AutoMine == true), function(wantOn)
        if AA1 and AA1.setMine then
            AA1.setMine(wantOn)
        else
            STATE.AutoMine = (wantOn == true)
        end
        -- ซิงค์กลับ “ย้ำ” กันเพี้ยน
        task.defer(function()
            local real = (AA1 and AA1.getMine and AA1.getMine()) or (STATE.AutoMine == true)
            if swMine then swMine.set(real) end
        end)
    end)

    swTrain = makeAV1Switch(row2, (AA1 and AA1.getTrain and AA1.getTrain()) or (STATE.AutoTrain == true), function(wantOn)
        if AA1 and AA1.setTrain then
            AA1.setTrain(wantOn)
        else
            STATE.AutoTrain = (wantOn == true)
        end
        task.defer(function()
            local real = (AA1 and AA1.getTrain and AA1.getTrain()) or (STATE.AutoTrain == true)
            if swTrain then swTrain.set(real) end
        end)
    end)

    -- bind ให้ AA1 ดันสวิตช์เราได้ (ทั้งจาก setMine/setTrain และ detect)
    if AA1 and AA1._uiBind then
        AA1._uiBind({
            setMine  = function(v)
                if swMine then swMine.set(v) end
                if v and swTrain then swTrain.set(false) end
            end,
            setTrain = function(v)
                if swTrain then swTrain.set(v) end
                if v and swMine then swMine.set(false) end
            end,
        })
    end

    if AA1 and AA1._setupDetect then
        task.defer(function()
            AA1._setupDetect()
        end)
    end

    -- INIT SYNC UI จาก STATE (ย้ำให้ตรงจริง)
    task.defer(function()
        local mineOn  = (AA1 and AA1.getMine  and AA1.getMine())  or (STATE.AutoMine  == true)
        local trainOn = (AA1 and AA1.getTrain and AA1.getTrain()) or (STATE.AutoTrain == true)
        if mineOn and trainOn then trainOn = false end
        if swMine then swMine.set(mineOn) end
        if swTrain then swTrain.set(trainOn) end
    end)
end)
--===== UFO HUB X • Home – Auto Rebirth (AA1 Runner + Model A V1 + A V2) =====
-- Logic main:
--   • ส่วน AA1 (ด้านบน) รันทันทีตอนโหลดสคริปต์ (ไม่ต้องกด Home)
--   • ส่วน UI (registerRight("Home")) แค่ sync ปุ่มกับ STATE ของ AA1

----------------------------------------------------------------------
-- AA1 RUNNER (ไม่มี UI, ทำงานทันทีตอนรันสคริปต์)
----------------------------------------------------------------------
do
    local ReplicatedStorage = game:GetService("ReplicatedStorage")

    ------------------------------------------------------------------
    -- SAVE (AA1) ใช้ getgenv().UFOX_SAVE
    ------------------------------------------------------------------
    local SAVE = (getgenv and getgenv().UFOX_SAVE) or {
        get = function(_, _, d) return d end,
        set = function() end,
    }

    local GAME_ID  = tonumber(game.GameId)  or 0
    local PLACE_ID = tonumber(game.PlaceId) or 0

    -- AA1/HomeAutoRebirth/<GAME>/<PLACE>/(Enabled|Mode|Amount)
    local BASE_SCOPE = ("AA1/HomeAutoRebirth/%d/%d"):format(GAME_ID, PLACE_ID)

    local function K(field)
        return BASE_SCOPE .. "/" .. field
    end

    local function SaveGet(field, default)
        local ok, v = pcall(function()
            return SAVE.get(K(field), default)
        end)
        return ok and v or default
    end

    local function SaveSet(field, value)
        pcall(function()
            SAVE.set(K(field), value)
        end)
    end

    ------------------------------------------------------------------
    -- STATE จาก AA1
    ------------------------------------------------------------------
    local STATE = {
        Enabled = SaveGet("Enabled", false),       -- เปิด Auto Rebirth อยู่ไหม
        Mode    = SaveGet("Mode", "SEQUENCE"),     -- "SEQUENCE" หรือ "FIXED"
        Amount  = SaveGet("Amount", 1),            -- 1–36
    }

    if type(STATE.Amount) ~= "number" or STATE.Amount < 1 or STATE.Amount > 36 then
        STATE.Amount = 1
        SaveSet("Amount", STATE.Amount)
    end

    if STATE.Mode ~= "FIXED" and STATE.Mode ~= "SEQUENCE" then
        STATE.Mode = "SEQUENCE"
        SaveSet("Mode", STATE.Mode)
    end

    ------------------------------------------------------------------
    -- REMOTE: Rebirth
    ------------------------------------------------------------------
    local function getRebirthRemote()
        local ok, rf = pcall(function()
            local paper   = ReplicatedStorage:WaitForChild("Paper")
            local remotes = paper:WaitForChild("Remotes")
            return remotes:WaitForChild("__remotefunction")
        end)
        if not ok then
            warn("[UFO HUB X • Auto Rebirth AA1] cannot get __remotefunction")
            return nil
        end
        return rf
    end

    local function doRebirth(amount)
        amount = math.clamp(math.floor(tonumber(amount) or 1), 1, 36)
        local rf = getRebirthRemote()
        if not rf then return end

        local args = { "Rebirth", amount }
        local ok, err = pcall(function()
            rf:InvokeServer(unpack(args))
        end)
        if not ok then
            warn("[UFO HUB X • Auto Rebirth AA1] Rebirth(",amount,") error:", err)
        end
    end

    ------------------------------------------------------------------
    -- LOOP AUTO REBIRTH (วิ่งจริงจาก STATE)
    ------------------------------------------------------------------
    local AUTO_INTERVAL = 0.03   -- เร็ว
    local loopRunning   = false

    local function startAutoLoop()
        if loopRunning then return end
        loopRunning = true

        task.spawn(function()
            while STATE.Enabled do
                if STATE.Mode == "FIXED" then
                    doRebirth(STATE.Amount)
                    task.wait(AUTO_INTERVAL)
                else
                    for amt = 36, 1, -1 do
                        if not STATE.Enabled then break end
                        doRebirth(amt)
                        task.wait(AUTO_INTERVAL)
                    end
                end
            end
            loopRunning = false
        end)
    end

    local function applyFromState()
        if STATE.Enabled then
            startAutoLoop()
        end
    end

    ------------------------------------------------------------------
    -- EXPORT AA1 + AUTO-RUN ทันทีตอนโหลดสคริปต์หลัก
    ------------------------------------------------------------------
    _G.UFOX_AA1 = _G.UFOX_AA1 or {}
    _G.UFOX_AA1["HomeAutoRebirth"] = {
        state      = STATE,
        apply      = applyFromState,
        setEnabled = function(v)
            STATE.Enabled = v and true or false
            SaveSet("Enabled", STATE.Enabled)
            applyFromState()
        end,
        setMode    = function(mode)
            if mode ~= "FIXED" and mode ~= "SEQUENCE" then return end
            STATE.Mode = mode
            SaveSet("Mode", STATE.Mode)
            applyFromState()
        end,
        setAmount  = function(amount)
            STATE.Amount = math.clamp(math.floor(tonumber(amount) or 1), 1, 36)
            SaveSet("Amount", STATE.Amount)
        end,
        saveGet    = SaveGet,
        saveSet    = SaveSet,
    }

    -- AA1: ถ้าเคยเปิดไว้ → รันเลย โดยไม่ต้องกด Home
    task.defer(function()
        applyFromState()
    end)
end

----------------------------------------------------------------------
-- UI PART: Model A V1 + Model A V2 ใน Tab Home (Sync กับ AA1 ตัวบน)
----------------------------------------------------------------------

registerRight("Home", function(scroll)
    local TweenService     = game:GetService("TweenService")
    local UserInputService = game:GetService("UserInputService")

    ------------------------------------------------------------------------
    -- THEME + HELPERS
    ------------------------------------------------------------------------
    local THEME = {
        GREEN       = Color3.fromRGB(25,255,125),
        GREEN_DARK  = Color3.fromRGB(0,120,60),
        WHITE       = Color3.fromRGB(255,255,255),
        BLACK       = Color3.fromRGB(0,0,0),
        RED         = Color3.fromRGB(255,40,40),
    }

    local function corner(ui, r)
        local c = Instance.new("UICorner")
        c.CornerRadius = UDim.new(0, r or 12)
        c.Parent = ui
        return c
    end

    local function stroke(ui, th, col)
        local s = Instance.new("UIStroke")
        s.Thickness = th or 2.2
        s.Color = col or THEME.GREEN
        s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
        s.Parent = ui
        return s
    end

    local function tween(o, p, d)
        TweenService:Create(
            o,
            TweenInfo.new(d or 0.08, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
            p
        ):Play()
    end

    local function trim(s)
        return (s:gsub("^%s*(.-)%s*$", "%1"))
    end

    ------------------------------------------------------------------------
    -- CONFIG: ปรับชื่อปุ่ม Rebirth 1–36 ได้จากตรงนี้
    ------------------------------------------------------------------------
    local REBIRTH_LABELS = {
        [1] = "1 Rebirth",
        [2] = "5 Rebirth",
        [3] = "20 Rebirth",
        [4] = "50 Rebirth",
        [5] = "100 Rebirth",
        [6] = "250 Rebirth",
        [7] = "500 Rebirth",
        [8] = "1K Rebirth",
        [9] = "2.5K Rebirth",
        [10] = "5K Rebirth",
        [11] = "10K Rebirth",
        [12] = "25K Rebirth",
        [13] = "50K Rebirth",
        [14] = "100K Rebirth",
        [15] = "250K Rebirth",
        [16] = "500K Rebirth",
        [17] = "1M Rebirth",
        [18] = "2.5M Rebirth",
        [19] = "10M Rebirth",
        [20] = "25M Rebirth",
        [21] = "100M Rebirth",
        [22] = "1B Rebirth",
        [23] = "50B Rebirth",
        [24] = "500B Rebirth",
        [25] = "5T Rebirth",
        [26] = "100T Rebirth",
        [27] = "1Qd Rebirth",
        [28] = "50Qd Rebirth",
        [29] = "500Qd Rebirth",
        [30] = "2.5Qn Rebirth",
        [31] = "50Qn Rebirth",
        [32] = "500Qn Rebirth",
        [33] = "5Sx Rebirth",
        [34] = "100Sx Rebirth",
        [35] = "1Sp Rebirth",
        [36] = "50Sp Rebirth",
    }

    local function getRebirthLabel(amount)
        return REBIRTH_LABELS[amount] or (tostring(amount) .. " Rebirth")
    end

    ------------------------------------------------------------------------
    -- ดึง AA1 STATE (จากบล็อกด้านบน)
    ------------------------------------------------------------------------
    local AA1  = _G.UFOX_AA1 and _G.UFOX_AA1["HomeAutoRebirth"]
    local STATE = (AA1 and AA1.state) or {
        Enabled = false,
        Mode    = "SEQUENCE",
        Amount  = 1,
    }

    ------------------------------------------------------------------------
    -- UIListLayout (Model A V1 Rule)
    ------------------------------------------------------------------------
    local vlist = scroll:FindFirstChildOfClass("UIListLayout")
    if not vlist then
        vlist = Instance.new("UIListLayout")
        vlist.Parent = scroll
        vlist.Padding   = UDim.new(0, 12)
        vlist.SortOrder = Enum.SortOrder.LayoutOrder
    end
    scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y

    local base = 0
    for _, ch in ipairs(scroll:GetChildren()) do
        if ch:IsA("GuiObject") and ch ~= vlist then
            base = math.max(base, ch.LayoutOrder or 0)
        end
    end

    ------------------------------------------------------------------------
    -- HEADER: Auto Rebirth 🔁
    ------------------------------------------------------------------------
    local header = Instance.new("TextLabel")
    header.Name = "A1_Home_AutoRebirth_Header"
    header.Parent = scroll
    header.BackgroundTransparency = 1
    header.Size = UDim2.new(1, 0, 0, 36)
    header.Font = Enum.Font.GothamBold
    header.TextSize = 16
    header.TextColor3 = THEME.WHITE
    header.TextXAlignment = Enum.TextXAlignment.Left
    header.Text = "》》》Auto Rebirth 🔁《《《"
    header.LayoutOrder = base + 1

    ------------------------------------------------------------------------
    -- HELPERS: แถวสวิตช์ (Model A V1)
    ------------------------------------------------------------------------
    local function makeRowSwitch(name, order, labelText, initialOn, onToggle)
        local row = Instance.new("Frame")
        row.Name = name
        row.Parent = scroll
        row.Size = UDim2.new(1, -6, 0, 46)
        row.BackgroundColor3 = THEME.BLACK
        corner(row, 12)
        stroke(row, 2.2, THEME.GREEN)
        row.LayoutOrder = order

        local lab = Instance.new("TextLabel")
        lab.Parent = row
        lab.BackgroundTransparency = 1
        lab.Size = UDim2.new(1, -160, 1, 0)
        lab.Position = UDim2.new(0, 16, 0, 0)
        lab.Font = Enum.Font.GothamBold
        lab.TextSize = 13
        lab.TextColor3 = THEME.WHITE
        lab.TextXAlignment = Enum.TextXAlignment.Left
        lab.Text = labelText

        local sw = Instance.new("Frame")
        sw.Parent = row
        sw.AnchorPoint = Vector2.new(1,0.5)
        sw.Position = UDim2.new(1, -12, 0.5, 0)
        sw.Size = UDim2.fromOffset(52,26)
        sw.BackgroundColor3 = THEME.BLACK
        corner(sw, 13)

        local swStroke = Instance.new("UIStroke")
        swStroke.Parent = sw
        swStroke.Thickness = 1.8

        local knob = Instance.new("Frame")
        knob.Parent = sw
        knob.Size = UDim2.fromOffset(22,22)
        knob.BackgroundColor3 = THEME.WHITE
        knob.Position = UDim2.new(0,2,0.5,-11)
        corner(knob,11)

        local currentOn = initialOn and true or false

        local function updateVisual(on)
            currentOn = on
            swStroke.Color = on and THEME.GREEN or THEME.RED
            tween(knob, { Position = UDim2.new(on and 1 or 0, on and -24 or 2, 0.5, -11) }, 0.08)
        end

        local function setState(on, fireCallback)
            fireCallback = (fireCallback ~= false)
            if currentOn == on then return end
            updateVisual(on)
            if fireCallback and onToggle then onToggle(on) end
        end

        local btn = Instance.new("TextButton")
        btn.Parent = sw
        btn.BackgroundTransparency = 1
        btn.Size = UDim2.fromScale(1,1)
        btn.Text = ""
        btn.AutoButtonColor = false
        btn.MouseButton1Click:Connect(function()
            setState(not currentOn, true)
        end)

        updateVisual(currentOn)

        return { row = row, setState = setState, getState = function() return currentOn end }
    end

    ------------------------------------------------------------------------
    -- Row1: Auto Rebirth
    ------------------------------------------------------------------------
    local autoRebirthRow = makeRowSwitch(
        "A1_Home_AutoRebirth",
        base + 2,
        "Auto Rebirth",
        STATE.Enabled,
        function(state)
            if AA1 and AA1.setEnabled then
                AA1.setEnabled(state)
            end
        end
    )

    ------------------------------------------------------------------------
    -- Model A V2 PART: Row + Select Options + Panel
    ------------------------------------------------------------------------
    local panelParent = scroll.Parent
    local amountPanel
    local inputConn
    local opened = false

    local amountButtons = {}
    local allButtons    = {}

    -- ✅ เก็บ ref ของปุ่ม Select เพื่อให้ closeAmountPanel() ดับไฟได้เสมอ
    local selectBtnRef

    local function disconnectInput()
        if inputConn then
            inputConn:Disconnect()
            inputConn = nil
        end
    end

    -- ✅ Visual ของปุ่ม Select (เหมือน V A2)
    local selectStrokeRef
    local function updateSelectVisual(isOpen)
        if not selectStrokeRef then return end
        if isOpen then
            selectStrokeRef.Color        = THEME.GREEN
            selectStrokeRef.Thickness    = 2.4
            selectStrokeRef.Transparency = 0
        else
            selectStrokeRef.Color        = THEME.GREEN_DARK
            selectStrokeRef.Thickness    = 1.8
            selectStrokeRef.Transparency = 0.4
        end
    end

    -- ✅ ปิดแบบศูนย์กลาง: ปิด panel + ดับไฟ + opened=false (แก้บั๊กค้างไฟเขียว)
    local function closeAmountPanel()
        if amountPanel then
            amountPanel:Destroy()
            amountPanel = nil
        end
        disconnectInput()
        amountButtons = {}
        allButtons    = {}
        opened = false

        updateSelectVisual(false)
    end

    local function destroyAmountPanel()
        closeAmountPanel()
    end

    local function updateAmountHighlight()
        for amt, info in pairs(amountButtons) do
            local on = (STATE.Mode == "FIXED" and STATE.Amount == amt)
            if on then
                info.stroke.Color        = THEME.GREEN
                info.stroke.Thickness    = 2.4
                info.stroke.Transparency = 0
                info.glow.Visible        = true
            else
                info.stroke.Color        = THEME.GREEN_DARK
                info.stroke.Thickness    = 1.6
                info.stroke.Transparency = 0.4
                info.glow.Visible        = false
            end
        end
    end

    local function openAmountPanel()
        destroyAmountPanel()
        if not panelParent or not panelParent.AbsoluteSize then return end

        local pw, ph = panelParent.AbsoluteSize.X, panelParent.AbsoluteSize.Y
        local leftRatio   = 0.645
        local topRatio    = 0.02
        local bottomRatio = 0.02
        local rightMargin = 8

        local leftX   = math.floor(pw * leftRatio)
        local topY    = math.floor(ph * topRatio)
        local bottomM = math.floor(ph * bottomRatio)

        local w = pw - leftX - rightMargin
        local h = ph - topY - bottomM

        amountPanel = Instance.new("Frame")
        amountPanel.Name = "VA2_RebirthPanel"
        amountPanel.Parent = panelParent
        amountPanel.BackgroundColor3 = THEME.BLACK
        amountPanel.ClipsDescendants = true
        amountPanel.AnchorPoint = Vector2.new(0, 0)
        amountPanel.Position    = UDim2.new(0, leftX, 0, topY)
        amountPanel.Size        = UDim2.new(0, w, 0, h)
        amountPanel.ZIndex      = 50

        corner(amountPanel, 12)
        stroke(amountPanel, 2.4, THEME.GREEN)

        local body = Instance.new("Frame")
        body.Name = "Body"
        body.Parent = amountPanel
        body.BackgroundTransparency = 1
        body.BorderSizePixel = 0
        body.Position = UDim2.new(0, 4, 0, 4)
        body.Size     = UDim2.new(1, -8, 1, -8)
        body.ZIndex   = amountPanel.ZIndex + 1

        local searchBox = Instance.new("TextBox")
        searchBox.Name = "SearchBox"
        searchBox.Parent = body
        searchBox.BackgroundColor3 = THEME.BLACK
        searchBox.ClearTextOnFocus = false
        searchBox.Font = Enum.Font.GothamBold
        searchBox.TextSize = 14
        searchBox.TextColor3 = THEME.WHITE
        searchBox.PlaceholderText = "🔍 Search"
        searchBox.TextXAlignment = Enum.TextXAlignment.Center
        searchBox.Text = ""
        searchBox.ZIndex = body.ZIndex + 1
        searchBox.Size = UDim2.new(1, 0, 0, 32)
        searchBox.Position = UDim2.new(0, 0, 0, 0)
        corner(searchBox, 8)

        local sbStroke = stroke(searchBox, 1.8, THEME.GREEN)
        sbStroke.ZIndex = searchBox.ZIndex + 1

        local listHolder = Instance.new("ScrollingFrame")
        listHolder.Name = "AmountList"
        listHolder.Parent = body
        listHolder.BackgroundColor3 = THEME.BLACK
        listHolder.BorderSizePixel = 0
        listHolder.ScrollBarThickness = 0
        listHolder.AutomaticCanvasSize = Enum.AutomaticSize.Y
        listHolder.CanvasSize = UDim2.new(0,0,0,0)
        listHolder.ZIndex = body.ZIndex + 1
        listHolder.ScrollingDirection = Enum.ScrollingDirection.Y
        listHolder.ClipsDescendants = true

        local listTopOffset = 32 + 10
        listHolder.Position = UDim2.new(0, 0, 0, listTopOffset)
        listHolder.Size     = UDim2.new(1, 0, 1, -(listTopOffset + 4))

        local listLayout = Instance.new("UIListLayout")
        listLayout.Parent = listHolder
        listLayout.Padding = UDim.new(0, 8)
        listLayout.SortOrder = Enum.SortOrder.LayoutOrder
        listLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center

        local listPadding = Instance.new("UIPadding")
        listPadding.Parent = listHolder
        listPadding.PaddingTop = UDim.new(0, 6)
        listPadding.PaddingBottom = UDim.new(0, 6)
        listPadding.PaddingLeft = UDim.new(0, 4)
        listPadding.PaddingRight = UDim.new(0, 4)

        local locking = false
        listHolder:GetPropertyChangedSignal("CanvasPosition"):Connect(function()
            if locking then return end
            locking = true
            local pos = listHolder.CanvasPosition
            if pos.X ~= 0 then
                listHolder.CanvasPosition = Vector2.new(0, pos.Y)
            end
            locking = false
        end)

        amountButtons = {}
        allButtons    = {}

        local function makeGlowButton(amount)
            local label = getRebirthLabel(amount)

            local btn = Instance.new("TextButton")
            btn.Name = "Btn_Rebirth_" .. tostring(amount)
            btn.Parent = listHolder
            btn.Size = UDim2.new(1, 0, 0, 28)
            btn.BackgroundColor3 = THEME.BLACK
            btn.AutoButtonColor = false
            btn.Font = Enum.Font.GothamBold
            btn.TextSize = 14
            btn.TextColor3 = THEME.WHITE
            btn.Text = label
            btn.ZIndex = listHolder.ZIndex + 1
            btn.TextXAlignment = Enum.TextXAlignment.Center
            btn.TextYAlignment = Enum.TextYAlignment.Center
            corner(btn, 6)

            local st = stroke(btn, 1.6, THEME.GREEN_DARK)
            st.Transparency = 0.4

            local glowBar = Instance.new("Frame")
            glowBar.Name = "GlowBar"
            glowBar.Parent = btn
            glowBar.BackgroundColor3 = THEME.GREEN
            glowBar.BorderSizePixel = 0
            glowBar.Size = UDim2.new(0, 3, 1, 0)
            glowBar.Position = UDim2.new(0, 0, 0, 0)
            glowBar.ZIndex = btn.ZIndex + 1
            glowBar.Visible = false

            amountButtons[amount] = { button = btn, stroke = st, glow = glowBar }
            table.insert(allButtons, btn)

            btn.MouseButton1Click:Connect(function()
                if not AA1 then return end

                if STATE.Mode == "FIXED" and STATE.Amount == amount then
                    AA1.setMode("SEQUENCE")
                    updateAmountHighlight()
                    AA1.apply()
                    return
                end

                AA1.setAmount(amount)
                AA1.setMode("FIXED")
                updateAmountHighlight()
                AA1.apply()
            end)

            return btn
        end

        for amt = 1, 36 do
            local b = makeGlowButton(amt)
            b.LayoutOrder = amt
        end

        updateAmountHighlight()

        local function applySearch()
            local q = trim(searchBox.Text or "")
            q = string.lower(q)

            if q == "" then
                for _, btn in ipairs(allButtons) do btn.Visible = true end
            else
                for _, btn in ipairs(allButtons) do
                    local text = string.lower(btn.Text or "")
                    btn.Visible = string.find(text, q, 1, true) ~= nil
                end
            end

            listHolder.CanvasPosition = Vector2.new(0, 0)
        end

        searchBox:GetPropertyChangedSignal("Text"):Connect(applySearch)
        searchBox.Focused:Connect(function() sbStroke.Color = THEME.GREEN end)
        searchBox.FocusLost:Connect(function() sbStroke.Color = THEME.GREEN end)

        -- ✅ กดนอกจอ = ปิด + ดับไฟปุ่ม (ไม่ค้างแล้ว)
        inputConn = UserInputService.InputBegan:Connect(function(input)
            if not amountPanel then return end
            if input.UserInputType ~= Enum.UserInputType.MouseButton1
               and input.UserInputType ~= Enum.UserInputType.Touch then
                return
            end

            local pos = input.Position
            local op  = amountPanel.AbsolutePosition
            local os  = amountPanel.AbsoluteSize

            local inside =
                pos.X >= op.X and pos.X <= op.X + os.X and
                pos.Y >= op.Y and pos.Y <= op.Y + os.Y

            if not inside then
                closeAmountPanel()
            end
        end)
    end

    ------------------------------------------------------------------------
    -- Row2: แถว + ปุ่ม Select Options (โมเดล A V2 เป๊ะ)
    ------------------------------------------------------------------------
    local row2 = Instance.new("Frame")
    row2.Name = "VA2_Rebirth_Row"
    row2.Parent = scroll
    row2.Size = UDim2.new(1, -6, 0, 46)
    row2.BackgroundColor3 = THEME.BLACK
    corner(row2, 12)
    stroke(row2, 2.2, THEME.GREEN)
    row2.LayoutOrder = base + 3

    local lab2 = Instance.new("TextLabel")
    lab2.Parent = row2
    lab2.BackgroundTransparency = 1
    lab2.Size = UDim2.new(0, 180, 1, 0)
    lab2.Position = UDim2.new(0, 16, 0, 0)
    lab2.Font = Enum.Font.GothamBold
    lab2.TextSize = 13
    lab2.TextColor3 = THEME.WHITE
    lab2.TextXAlignment = Enum.TextXAlignment.Left
    lab2.Text = "Select Rebirth Amount"

    local selectBtn = Instance.new("TextButton")
    selectBtnRef = selectBtn

    selectBtn.Name = "VA2_Rebirth_Select"
    selectBtn.Parent = row2
    selectBtn.AnchorPoint = Vector2.new(1, 0.5)
    selectBtn.Position = UDim2.new(1, -16, 0.5, 0)
    selectBtn.Size = UDim2.new(0, 220, 0, 28)
    selectBtn.BackgroundColor3 = THEME.BLACK
    selectBtn.AutoButtonColor = false
    selectBtn.Text = "🔍 Select Options"
    selectBtn.Font = Enum.Font.GothamBold
    selectBtn.TextSize = 13
    selectBtn.TextColor3 = THEME.WHITE
    selectBtn.TextXAlignment = Enum.TextXAlignment.Center
    selectBtn.TextYAlignment = Enum.TextYAlignment.Center
    corner(selectBtn, 8)

    local selectStroke = stroke(selectBtn, 1.8, THEME.GREEN_DARK)
    selectStroke.Transparency = 0.4
    selectStrokeRef = selectStroke

    updateSelectVisual(false)

    local padding = Instance.new("UIPadding")
    padding.Parent = selectBtn
    padding.PaddingLeft  = UDim.new(0, 8)
    padding.PaddingRight = UDim.new(0, 26)

    local arrow = Instance.new("TextLabel")
    arrow.Parent = selectBtn
    arrow.AnchorPoint = Vector2.new(1,0.5)
    arrow.Position = UDim2.new(1, -6, 0.5, 0)
    arrow.Size = UDim2.new(0, 18, 0, 18)
    arrow.BackgroundTransparency = 1
    arrow.Font = Enum.Font.GothamBold
    arrow.TextSize = 18
    arrow.TextColor3 = THEME.WHITE
    arrow.Text = "▼"

    selectBtn.MouseButton1Click:Connect(function()
        if opened then
            closeAmountPanel() -- ✅ ปิดแบบดับไฟ
        else
            openAmountPanel()
            opened = true
            updateSelectVisual(true)
        end
        print("[V A2 • Rebirth] Select Options clicked, opened =", opened)
    end)

    ------------------------------------------------------------------------
    -- Sync UI จาก STATE ที่เซฟไว้ (ตอนเปิด Tab Home)
    ------------------------------------------------------------------------
    task.defer(function()
        autoRebirthRow.setState(STATE.Enabled, false)
    end)
end) 
--===== UFO HUB X • Home – Auto Claim Rewards 🎁 (Model A V1 + AA1 • PERMA LOOPS) =====
-- Tab: Home
-- Header: Auto Claim Rewards 🎁
-- Row1: Auto Claim Aura Egg (SPAM LOOP)      -> Claim Time Reward + Use Aura Egg
-- Row2: Auto Claim Daily Chest (PERMA LOOP)  -> Claim Chest "DailyChest"
-- Row3: Auto Claim Group Chest (PERMA LOOP)  -> Claim Chest "GroupChest"
-- Row4: Auto Claim Daily Reward              -> Claim Daily
-- Row5: Auto Claim Index Reward              -> Claim Index Reward
-- + AA1: จำสถานะสวิตช์ และ Auto-run ตั้งแต่โหลด UI ไม่ต้องกดปุ่ม Home

local TweenService      = game:GetService("TweenService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

------------------------------------------------------------------------
-- THEME + HELPERS (Model A V1)
------------------------------------------------------------------------
local THEME = {
    GREEN = Color3.fromRGB(25,255,125),
    RED   = Color3.fromRGB(255,40,40),
    WHITE = Color3.fromRGB(255,255,255),
    BLACK = Color3.fromRGB(0,0,0),
}

local function corner(ui, r)
    local c = Instance.new("UICorner")
    c.CornerRadius = UDim.new(0, r or 12)
    c.Parent = ui
end

local function stroke(ui, th, col)
    local s = Instance.new("UIStroke")
    s.Thickness = th or 2.2
    s.Color = col or THEME.GREEN
    s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    s.Parent = ui
end

local function tween(o, p, d)
    TweenService:Create(
        o,
        TweenInfo.new(d or 0.08, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
        p
    ):Play()
end

------------------------------------------------------------------------
-- AA1 SAVE (HomeAutoClaim) • ใช้ getgenv().UFOX_SAVE
------------------------------------------------------------------------
local SAVE = (getgenv and getgenv().UFOX_SAVE) or {
    get = function(_, _, d) return d end,
    set = function() end,
}

local GAME_ID  = tonumber(game.GameId)  or 0
local PLACE_ID = tonumber(game.PlaceId) or 0

local BASE_SCOPE = ("AA1/HomeAutoClaim/%d/%d"):format(GAME_ID, PLACE_ID)

local function K(field)
    return BASE_SCOPE .. "/" .. field
end

local function SaveGet(field, default)
    local ok, v = pcall(function()
        return SAVE.get(K(field), default)
    end)
    return ok and v or default
end

local function SaveSet(field, value)
    pcall(function()
        SAVE.set(K(field), value)
    end)
end

local STATE = {
    AutoEgg       = SaveGet("AutoEgg",       false),
    AutoDaily     = SaveGet("AutoDaily",     false),
    AutoGroup     = SaveGet("AutoGroup",     false),
    AutoDailyRw   = SaveGet("AutoDailyRw",   false), -- Row4
    AutoIndexRw   = SaveGet("AutoIndexRw",   false), -- Row5
}

------------------------------------------------------------------------
-- REMOTES
------------------------------------------------------------------------
local function getRemoteFunction()
    local ok, rf = pcall(function()
        local paper   = ReplicatedStorage:WaitForChild("Paper")
        local remotes = paper:WaitForChild("Remotes")
        return remotes:WaitForChild("__remotefunction")
    end)
    if not ok then
        warn("[UFO HUB X • HomeAutoClaim] cannot get __remotefunction:", rf)
        return nil
    end
    return rf
end

local function claimAuraEggOnce()
    local rf = getRemoteFunction()
    if not rf then return end

    local ok1, err1 = pcall(function()
        rf:InvokeServer("Claim Time Reward")
    end)
    if not ok1 then
        warn("[UFO HUB X • HomeAutoClaim] Claim Time Reward error:", err1)
    end

    task.wait(0.25)

    local ok2, err2 = pcall(function()
        rf:InvokeServer("Use Item", "Aura Egg", 1)
    end)
    if not ok2 then
        warn("[UFO HUB X • HomeAutoClaim] Use Aura Egg error:", err2)
    end
end

local function claimDailyChestOnce()
    local rf = getRemoteFunction()
    if not rf then return end
    local ok, err = pcall(function()
        rf:InvokeServer("Claim Chest", "DailyChest")
    end)
    if not ok then
        warn("[UFO HUB X • HomeAutoClaim] Claim DailyChest error:", err)
    end
end

local function claimGroupChestOnce()
    local rf = getRemoteFunction()
    if not rf then return end
    local ok, err = pcall(function()
        rf:InvokeServer("Claim Chest", "GroupChest")
    end)
    if not ok then
        warn("[UFO HUB X • HomeAutoClaim] Claim GroupChest error:", err)
    end
end

-- Row4
local function claimDailyRewardOnce()
    local rf = getRemoteFunction()
    if not rf then return end
    local ok, err = pcall(function()
        rf:InvokeServer("Claim Daily")
    end)
    if not ok then
        warn("[UFO HUB X • HomeAutoClaim] Claim Daily error:", err)
    end
end

-- Row5
local function claimIndexRewardOnce()
    local rf = getRemoteFunction()
    if not rf then return end
    local ok, err = pcall(function()
        rf:InvokeServer("Claim Index Reward")
    end)
    if not ok then
        warn("[UFO HUB X • HomeAutoClaim] Claim Index Reward error:", err)
    end
end

------------------------------------------------------------------------
-- LOOP FLAGS + PERMA LOOPS
------------------------------------------------------------------------
local EGG_SPAM_DELAY        = 0.8
local DAILY_CHEST_SPAM      = 1.2
local GROUP_CHEST_SPAM      = 1.2
local DAILY_REWARD_SPAM     = 1.2
local INDEX_REWARD_SPAM     = 1.2

local eggOn       = STATE.AutoEgg
local dailyOn     = STATE.AutoDaily
local groupOn     = STATE.AutoGroup
local dailyRwOn   = STATE.AutoDailyRw
local indexRwOn   = STATE.AutoIndexRw

-- Row1: Aura Egg (วนเรื่อยๆ)
task.spawn(function()
    while true do
        if eggOn then
            claimAuraEggOnce()
            task.wait(EGG_SPAM_DELAY)
        else
            task.wait(0.5)
        end
    end
end)

-- Row2: Daily Chest (วนเรื่อยๆ)
task.spawn(function()
    while true do
        if dailyOn then
            claimDailyChestOnce()
            task.wait(DAILY_CHEST_SPAM)
        else
            task.wait(0.5)
        end
    end
end)

-- Row3: Group Chest (วนเรื่อยๆ)
task.spawn(function()
    while true do
        if groupOn then
            claimGroupChestOnce()
            task.wait(GROUP_CHEST_SPAM)
        else
            task.wait(0.5)
        end
    end
end)

-- Row4: Claim Daily (วนเรื่อยๆ)
task.spawn(function()
    while true do
        if dailyRwOn then
            claimDailyRewardOnce()
            task.wait(DAILY_REWARD_SPAM)
        else
            task.wait(0.5)
        end
    end
end)

-- Row5: Claim Index Reward (วนเรื่อยๆ)
task.spawn(function()
    while true do
        if indexRwOn then
            claimIndexRewardOnce()
            task.wait(INDEX_REWARD_SPAM)
        else
            task.wait(0.5)
        end
    end
end)

------------------------------------------------------------------------
-- UI ฝั่งขวา (Model A V1) • Tab: Home
------------------------------------------------------------------------
registerRight("Home", function(scroll)
    local vlist = scroll:FindFirstChildOfClass("UIListLayout")
    if not vlist then
        vlist = Instance.new("UIListLayout")
        vlist.Parent = scroll
        vlist.Padding   = UDim.new(0, 12)
        vlist.SortOrder = Enum.SortOrder.LayoutOrder
    end
    scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y

    local base = 0
    for _, ch in ipairs(scroll:GetChildren()) do
        if ch:IsA("GuiObject") and ch ~= vlist then
            base = math.max(base, ch.LayoutOrder or 0)
        end
    end

    local header = Instance.new("TextLabel")
    header.Name = "A1_Home_AutoClaim_Header"
    header.Parent = scroll
    header.BackgroundTransparency = 1
    header.Size = UDim2.new(1, 0, 0, 36)
    header.Font = Enum.Font.GothamBold
    header.TextSize = 16
    header.TextColor3 = THEME.WHITE
    header.TextXAlignment = Enum.TextXAlignment.Left
    header.Text = "》》》Auto Claim Rewards 🎁《《《"
    header.LayoutOrder = base + 1

    local function makeRowSwitch(name, order, labelText, onToggle)
        local row = Instance.new("Frame")
        row.Name = name
        row.Parent = scroll
        row.Size = UDim2.new(1, -6, 0, 46)
        row.BackgroundColor3 = THEME.BLACK
        corner(row, 12)
        stroke(row, 2.2, THEME.GREEN)
        row.LayoutOrder = order

        local lab = Instance.new("TextLabel")
        lab.Parent = row
        lab.BackgroundTransparency = 1
        lab.Size = UDim2.new(1, -160, 1, 0)
        lab.Position = UDim2.new(0, 16, 0, 0)
        lab.Font = Enum.Font.GothamBold
        lab.TextSize = 13
        lab.TextColor3 = THEME.WHITE
        lab.TextXAlignment = Enum.TextXAlignment.Left
        lab.Text = labelText

        local sw = Instance.new("Frame")
        sw.Parent = row
        sw.AnchorPoint = Vector2.new(1,0.5)
        sw.Position = UDim2.new(1, -12, 0.5, 0)
        sw.Size = UDim2.fromOffset(52,26)
        sw.BackgroundColor3 = THEME.BLACK
        corner(sw, 13)

        local swStroke = Instance.new("UIStroke")
        swStroke.Parent = sw
        swStroke.Thickness = 1.8

        local knob = Instance.new("Frame")
        knob.Parent = sw
        knob.Size = UDim2.fromOffset(22,22)
        knob.BackgroundColor3 = THEME.WHITE
        knob.Position = UDim2.new(0,2,0.5,-11)
        corner(knob,11)

        local currentOn = false

        local function updateVisual(on)
            currentOn = on
            swStroke.Color = on and THEME.GREEN or THEME.RED
            tween(knob, {Position = UDim2.new(on and 1 or 0, on and -24 or 2, 0.5, -11)}, 0.08)
        end

        local function setState(on, fireCallback)
            fireCallback = (fireCallback ~= false)
            if currentOn == on then return end
            updateVisual(on)
            if fireCallback and onToggle then
                onToggle(on)
            end
        end

        local btn = Instance.new("TextButton")
        btn.Parent = sw
        btn.BackgroundTransparency = 1
        btn.Size = UDim2.fromScale(1,1)
        btn.Text = ""
        btn.AutoButtonColor = false
        btn.MouseButton1Click:Connect(function()
            setState(not currentOn, true)
        end)

        updateVisual(false)

        return { setState = setState }
    end

    local row1 = makeRowSwitch(
        "A1_Home_AutoClaim_AuraEgg",
        base + 2,
        "Auto Claim Aura Egg (non-stop loop)",
        function(state)
            eggOn = state
            SaveSet("AutoEgg", state)
        end
    )

    local row2 = makeRowSwitch(
        "A1_Home_AutoClaim_DailyChest",
        base + 3,
        "Auto Claim Daily Chest (non-stop loop)",
        function(state)
            dailyOn = state
            SaveSet("AutoDaily", state)
        end
    )

    local row3 = makeRowSwitch(
        "A1_Home_AutoClaim_GroupChest",
        base + 4,
        "Auto Claim Group Chest (non-stop loop)",
        function(state)
            groupOn = state
            SaveSet("AutoGroup", state)
        end
    )

    local row4 = makeRowSwitch(
        "A1_Home_AutoClaim_DailyReward",
        base + 5,
        "Auto Claim Daily Reward",
        function(state)
            dailyRwOn = state
            SaveSet("AutoDailyRw", state)
        end
    )

    local row5 = makeRowSwitch(
        "A1_Home_AutoClaim_IndexReward",
        base + 6,
        "Auto Claim Index Reward",
        function(state)
            indexRwOn = state
            SaveSet("AutoIndexRw", state)
        end
    )

    task.defer(function()
        if eggOn     then row1.setState(true, false) end
        if dailyOn   then row2.setState(true, false) end
        if groupOn   then row3.setState(true, false) end
        if dailyRwOn then row4.setState(true, false) end
        if indexRwOn then row5.setState(true, false) end
    end)
end)
--===== UFO HUB X • Home – Auto Potion 🧪 (AA1 + Model A V1 + V A2 Overlay) =====
-- Tab: Home
-- Row1 (A V1 Switch): Auto Potion (AA1)
-- Row2 (V A2 Overlay 100%): Select Potions (4 buttons, multi-select, click again = cancel)
-- Remote:
-- local args = {"Use Item","Luck Potion",1}
-- ReplicatedStorage.Paper.Remotes.__remotefunction:InvokeServer(unpack(args))

----------------------------------------------------------------------
-- AA1 RUNNER (ทำงานทันทีตอนรันสคริปต์หลัก)
----------------------------------------------------------------------
do
    local ReplicatedStorage = game:GetService("ReplicatedStorage")

    local SAVE = (getgenv and getgenv().UFOX_SAVE) or {
        get = function(_, _, d) return d end,
        set = function() end,
    }

    local GAME_ID  = tonumber(game.GameId)  or 0
    local PLACE_ID = tonumber(game.PlaceId) or 0
    local BASE     = ("AA1/HomeAutoPotion/%d/%d"):format(GAME_ID, PLACE_ID)

    local function K(field) return BASE .. "/" .. field end

    local function SaveGet(field, default)
        local ok, v = pcall(function()
            return SAVE.get(K(field), default)
        end)
        return ok and v or default
    end

    local function SaveSet(field, value)
        pcall(function()
            SAVE.set(K(field), value)
        end)
    end

    local POTION_LIST = {
        "Luck Potion",
        "Speed Potion",
        "Damage Potion",
        "Coin Potion",
    }

    _G.UFOX_AA1 = _G.UFOX_AA1 or {}
    _G.UFOX_AA1["HomeAutoPotion"] = _G.UFOX_AA1["HomeAutoPotion"] or {}

    local SYS = _G.UFOX_AA1["HomeAutoPotion"]

    SYS.STATE = SYS.STATE or {
        Enabled  = SaveGet("Enabled", false),
        Selected = SaveGet("Selected", {}), -- {["Luck Potion"]=true, ...}
    }

    local STATE = SYS.STATE
    if type(STATE.Selected) ~= "table" then STATE.Selected = {} end
    for k,v in pairs(STATE.Selected) do
        if v ~= true then STATE.Selected[k] = nil end
    end

    local function getRF()
        local ok, rf = pcall(function()
            return ReplicatedStorage:WaitForChild("Paper")
                :WaitForChild("Remotes")
                :WaitForChild("__remotefunction")
        end)
        if not ok then return nil end
        return rf
    end

    local function usePotion(itemName)
        local rf = getRF()
        if not rf then return end
        local args = { "Use Item", itemName, 1 }
        pcall(function()
            rf:InvokeServer(unpack(args))
        end)
    end

    local runnerStarted = false
    local function ensureRunner()
        if runnerStarted then return end
        runnerStarted = true

        task.spawn(function()
            while true do
                if STATE.Enabled then
                    local did = false
                    for _, name in ipairs(POTION_LIST) do
                        if not STATE.Enabled then break end
                        if STATE.Selected[name] == true then
                            did = true
                            usePotion(name)
                            task.wait(0.25)
                        end
                    end
                    task.wait(did and 0.10 or 0.25)
                else
                    task.wait(0.25)
                end
            end
        end)
    end

    local function setEnabled(v)
        v = v and true or false
        STATE.Enabled = v
        SaveSet("Enabled", v)
        ensureRunner()
    end

    local function setSelected(name, v)
        if v then
            STATE.Selected[name] = true
        else
            STATE.Selected[name] = nil
        end
        SaveSet("Selected", STATE.Selected)
    end

    SYS.setEnabled  = setEnabled
    SYS.setSelected = setSelected
    SYS.getEnabled  = function() return STATE.Enabled end
    SYS.getSelected = function(name) return STATE.Selected[name] == true end
    SYS.ensureRunner = ensureRunner

    -- AA1: ถ้าเคยเปิดไว้ → รันเลย (ไม่ต้องกด Home)
    task.defer(function()
        ensureRunner()
    end)
end

----------------------------------------------------------------------
-- UI PART: Model A V1 + V A2 Overlay ใน Tab Home (Sync กับ AA1)
----------------------------------------------------------------------
registerRight("Home", function(scroll)
    local TweenService      = game:GetService("TweenService")
    local UserInputService  = game:GetService("UserInputService")

    local THEME = {
        GREEN       = Color3.fromRGB(25,255,125),
        GREEN_DARK  = Color3.fromRGB(0,120,60),
        WHITE       = Color3.fromRGB(255,255,255),
        BLACK       = Color3.fromRGB(0,0,0),
        RED         = Color3.fromRGB(255,40,40),
    }

    local function corner(ui, r)
        local c = Instance.new("UICorner")
        c.CornerRadius = UDim.new(0, r or 12)
        c.Parent = ui
        return c
    end

    local function stroke(ui, th, col)
        local s = Instance.new("UIStroke")
        s.Thickness = th or 2.2
        s.Color = col or THEME.GREEN
        s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
        s.Parent = ui
        return s
    end

    local function tween(o, p, d)
        TweenService:Create(
            o,
            TweenInfo.new(d or 0.08, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
            p
        ):Play()
    end

    local function trim(s)
        return (s:gsub("^%s*(.-)%s*$", "%1"))
    end

    local AA1  = _G.UFOX_AA1 and _G.UFOX_AA1["HomeAutoPotion"]
    local STATE = (AA1 and AA1.STATE) or { Enabled=false, Selected={} }

    local POTION_LIST = {
        "Luck Potion",
        "Speed Potion",
        "Damage Potion",
        "Coin Potion",
    }

    ------------------------------------------------------------------------
    -- CLEANUP (กันซ้อน)
    ------------------------------------------------------------------------
    for _, name in ipairs({
        "HPOT_Header",
        "HPOT_Row1",
        "HPOT_Row2",
        "HPOT_OptionsPanel",
    }) do
        local o = scroll:FindFirstChild(name)
            or scroll.Parent:FindFirstChild(name)
            or (scroll:FindFirstAncestorOfClass("ScreenGui")
                and scroll:FindFirstAncestorOfClass("ScreenGui"):FindFirstChild(name))
        if o then o:Destroy() end
    end

    ------------------------------------------------------------------------
    -- UIListLayout (A V1: 1 layout + dynamic base)
    ------------------------------------------------------------------------
    local vlist = scroll:FindFirstChildOfClass("UIListLayout")
    if not vlist then
        vlist = Instance.new("UIListLayout")
        vlist.Parent = scroll
        vlist.Padding   = UDim.new(0, 12)
        vlist.SortOrder = Enum.SortOrder.LayoutOrder
    end
    scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y

    local base = 0
    for _, ch in ipairs(scroll:GetChildren()) do
        if ch:IsA("GuiObject") and ch ~= vlist then
            base = math.max(base, ch.LayoutOrder or 0)
        end
    end

    ------------------------------------------------------------------------
    -- HEADER
    ------------------------------------------------------------------------
    local header = Instance.new("TextLabel")
    header.Name = "HPOT_Header"
    header.Parent = scroll
    header.BackgroundTransparency = 1
    header.Size = UDim2.new(1, 0, 0, 36)
    header.Font = Enum.Font.GothamBold
    header.TextSize = 16
    header.TextColor3 = THEME.WHITE
    header.TextXAlignment = Enum.TextXAlignment.Left
    header.Text = "》》》Auto Potion 🧪《《《"
    header.LayoutOrder = base + 1

    ------------------------------------------------------------------------
    -- Base Row (A V1 card)
    ------------------------------------------------------------------------
    local function makeRow(name, order, labelText)
        local row = Instance.new("Frame")
        row.Name = name
        row.Parent = scroll
        row.Size = UDim2.new(1, -6, 0, 46)
        row.BackgroundColor3 = THEME.BLACK
        corner(row, 12)
        stroke(row, 2.2, THEME.GREEN)
        row.LayoutOrder = order

        local lab = Instance.new("TextLabel")
        lab.Parent = row
        lab.BackgroundTransparency = 1
        lab.Size = UDim2.new(0, 220, 1, 0)
        lab.Position = UDim2.new(0, 16, 0, 0)
        lab.Font = Enum.Font.GothamBold
        lab.TextSize = 13
        lab.TextColor3 = THEME.WHITE
        lab.TextXAlignment = Enum.TextXAlignment.Left
        lab.Text = labelText

        return row, lab
    end

    ------------------------------------------------------------------------
    -- Row1: A V1 Switch (AA1)
    ------------------------------------------------------------------------
    local row1 = makeRow("HPOT_Row1", base + 2, "Auto Potion")

    local function makeAV1Switch(parentRow, initialOn, onToggle)
        local sw = Instance.new("Frame")
        sw.Parent = parentRow
        sw.AnchorPoint = Vector2.new(1,0.5)
        sw.Position = UDim2.new(1, -16, 0.5, 0)
        sw.Size = UDim2.fromOffset(52,26)
        sw.BackgroundColor3 = THEME.BLACK
        corner(sw, 13)

        local swStroke = Instance.new("UIStroke")
        swStroke.Parent = sw
        swStroke.Thickness = 1.8

        local knob = Instance.new("Frame")
        knob.Parent = sw
        knob.Size = UDim2.fromOffset(22,22)
        knob.BackgroundColor3 = THEME.WHITE
        corner(knob, 11)

        local btn = Instance.new("TextButton")
        btn.Parent = sw
        btn.BackgroundTransparency = 1
        btn.Size = UDim2.fromScale(1,1)
        btn.Text = ""
        btn.AutoButtonColor = false

        local on = initialOn and true or false

        local function update()
            swStroke.Color = on and THEME.GREEN or THEME.RED
            tween(knob, {Position = UDim2.new(on and 1 or 0, on and -24 or 2, 0.5, -11)}, 0.08)
        end

        btn.MouseButton1Click:Connect(function()
            on = not on
            update()
            if onToggle then onToggle(on) end
        end)

        update()
        return {
            set = function(v) on = v and true or false; update() end,
            get = function() return on end,
        }
    end

    local sw1 = makeAV1Switch(row1, (AA1 and AA1.getEnabled and AA1.getEnabled()) or (STATE.Enabled == true), function(on)
        if AA1 and AA1.setEnabled then
            AA1.setEnabled(on)
        else
            STATE.Enabled = on and true or false
        end
    end)

    ------------------------------------------------------------------------
    -- Row2: V A2 Overlay (เปิดดูได้เลย ไม่ต้องเปิดสวิตช์)
    ------------------------------------------------------------------------
    local row2 = makeRow("HPOT_Row2", base + 3, "Select Potions")
    local panelParent = scroll.Parent

    local selectBtn = Instance.new("TextButton")
    selectBtn.Name = "HPOT_Select"
    selectBtn.Parent = row2
    selectBtn.AnchorPoint = Vector2.new(1, 0.5)
    selectBtn.Position = UDim2.new(1, -16, 0.5, 0)
    selectBtn.Size = UDim2.new(0, 220, 0, 28)
    selectBtn.BackgroundColor3 = THEME.BLACK
    selectBtn.AutoButtonColor = false
    selectBtn.Text = "🔍 Select Options"
    selectBtn.Font = Enum.Font.GothamBold
    selectBtn.TextSize = 13
    selectBtn.TextColor3 = THEME.WHITE
    selectBtn.TextXAlignment = Enum.TextXAlignment.Center
    selectBtn.TextYAlignment = Enum.TextYAlignment.Center
    corner(selectBtn, 8)

    local selectStroke = stroke(selectBtn, 1.8, THEME.GREEN_DARK)
    selectStroke.Transparency = 0.4

    local function updateSelectVisual(isOpen)
        if isOpen then
            selectStroke.Color        = THEME.GREEN
            selectStroke.Thickness    = 2.4
            selectStroke.Transparency = 0
        else
            selectStroke.Color        = THEME.GREEN_DARK
            selectStroke.Thickness    = 1.8
            selectStroke.Transparency = 0.4
        end
    end
    updateSelectVisual(false)

    local padding = Instance.new("UIPadding")
    padding.Parent = selectBtn
    padding.PaddingLeft  = UDim.new(0, 8)
    padding.PaddingRight = UDim.new(0, 26)

    local arrow = Instance.new("TextLabel")
    arrow.Parent = selectBtn
    arrow.AnchorPoint = Vector2.new(1,0.5)
    arrow.Position = UDim2.new(1, -6, 0.5, 0)
    arrow.Size = UDim2.new(0, 18, 0, 18)
    arrow.BackgroundTransparency = 1
    arrow.Font = Enum.Font.GothamBold
    arrow.TextSize = 18
    arrow.TextColor3 = THEME.WHITE
    arrow.Text = "▼"

    ------------------------------------------------------------------------
    -- V A2 Popup Panel + CLOSE BOTH SCREEN (ยกเว้น panel / selectBtn / search)
    ------------------------------------------------------------------------
    local optionsPanel
    local inputConn
    local opened = false
    local searchBox
    local allButtons = {}

    local function isInside(gui, pos)
        if not gui or not gui.Parent then return false end
        local ap = gui.AbsolutePosition
        local as = gui.AbsoluteSize
        return pos.X >= ap.X and pos.X <= ap.X + as.X and pos.Y >= ap.Y and pos.Y <= ap.Y + as.Y
    end

    local function disconnectInput()
        if inputConn then
            inputConn:Disconnect()
            inputConn = nil
        end
    end

    local function closePanel()
        if optionsPanel then
            optionsPanel:Destroy()
            optionsPanel = nil
        end
        searchBox = nil
        allButtons = {}
        disconnectInput()
        opened = false
        updateSelectVisual(false)
    end

    local function openPanel()
        closePanel()

        local pw, ph = panelParent.AbsoluteSize.X, panelParent.AbsoluteSize.Y
        local leftRatio   = 0.645
        local topRatio    = 0.02
        local bottomRatio = 0.02
        local rightMargin = 8

        local leftX   = math.floor(pw * leftRatio)
        local topY    = math.floor(ph * topRatio)
        local bottomM = math.floor(ph * bottomRatio)

        local w = pw - leftX - rightMargin
        local h = ph - topY - bottomM

        optionsPanel = Instance.new("Frame")
        optionsPanel.Name = "HPOT_OptionsPanel"
        optionsPanel.Parent = panelParent
        optionsPanel.BackgroundColor3 = THEME.BLACK
        optionsPanel.ClipsDescendants = true
        optionsPanel.AnchorPoint = Vector2.new(0, 0)
        optionsPanel.Position    = UDim2.new(0, leftX, 0, topY)
        optionsPanel.Size        = UDim2.new(0, w, 0, h)
        optionsPanel.ZIndex      = 50

        corner(optionsPanel, 12)
        stroke(optionsPanel, 2.4, THEME.GREEN)

        local body = Instance.new("Frame")
        body.Name = "Body"
        body.Parent = optionsPanel
        body.BackgroundTransparency = 1
        body.BorderSizePixel = 0
        body.Position = UDim2.new(0, 4, 0, 4)
        body.Size     = UDim2.new(1, -8, 1, -8)
        body.ZIndex   = optionsPanel.ZIndex + 1

        -- Search Box
        searchBox = Instance.new("TextBox")
        searchBox.Name = "SearchBox"
        searchBox.Parent = body
        searchBox.BackgroundColor3 = THEME.BLACK
        searchBox.ClearTextOnFocus = false
        searchBox.Font = Enum.Font.GothamBold
        searchBox.TextSize = 14
        searchBox.TextColor3 = THEME.WHITE
        searchBox.PlaceholderText = "🔍 Search"
        searchBox.TextXAlignment = Enum.TextXAlignment.Center
        searchBox.Text = ""
        searchBox.ZIndex = body.ZIndex + 1
        searchBox.Size = UDim2.new(1, 0, 0, 32)
        searchBox.Position = UDim2.new(0, 0, 0, 0)
        corner(searchBox, 8)

        local sbStroke = stroke(searchBox, 1.8, THEME.GREEN)
        sbStroke.ZIndex = searchBox.ZIndex + 1

        -- List
        local listHolder = Instance.new("ScrollingFrame")
        listHolder.Name = "PList"
        listHolder.Parent = body
        listHolder.BackgroundColor3 = THEME.BLACK
        listHolder.BorderSizePixel = 0
        listHolder.ScrollBarThickness = 0
        listHolder.AutomaticCanvasSize = Enum.AutomaticSize.Y
        listHolder.CanvasSize = UDim2.new(0,0,0,0)
        listHolder.ZIndex = body.ZIndex + 1
        listHolder.ScrollingDirection = Enum.ScrollingDirection.Y
        listHolder.ClipsDescendants = true

        local listTopOffset = 32 + 10
        listHolder.Position = UDim2.new(0, 0, 0, listTopOffset)
        listHolder.Size     = UDim2.new(1, 0, 1, -(listTopOffset + 4))

        local listLayout = Instance.new("UIListLayout")
        listLayout.Parent = listHolder
        listLayout.Padding = UDim.new(0, 8)
        listLayout.SortOrder = Enum.SortOrder.LayoutOrder
        listLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center

        local listPadding = Instance.new("UIPadding")
        listPadding.Parent = listHolder
        listPadding.PaddingTop = UDim.new(0, 6)
        listPadding.PaddingBottom = UDim.new(0, 6)
        listPadding.PaddingLeft = UDim.new(0, 4)
        listPadding.PaddingRight = UDim.new(0, 4)

        allButtons = {}

        local function makeGlowButton(label)
            local btn = Instance.new("TextButton")
            btn.Name = "Btn_" .. label
            btn.Parent = listHolder
            btn.Size = UDim2.new(1, 0, 0, 28)

            btn.BackgroundColor3 = THEME.BLACK
            btn.AutoButtonColor = false
            btn.Font = Enum.Font.GothamBold
            btn.TextSize = 14
            btn.TextColor3 = THEME.WHITE
            btn.Text = label
            btn.ZIndex = listHolder.ZIndex + 1
            btn.TextXAlignment = Enum.TextXAlignment.Center
            btn.TextYAlignment = Enum.TextYAlignment.Center
            corner(btn, 6)

            local st = stroke(btn, 1.6, THEME.GREEN_DARK)
            st.Transparency = 0.4

            local glowBar = Instance.new("Frame")
            glowBar.Name = "GlowBar"
            glowBar.Parent = btn
            glowBar.BackgroundColor3 = THEME.GREEN
            glowBar.BorderSizePixel = 0
            glowBar.Size = UDim2.new(0, 3, 1, 0)
            glowBar.Position = UDim2.new(0, 0, 0, 0)
            glowBar.ZIndex = btn.ZIndex + 1
            glowBar.Visible = false

            local function update()
                local on = (AA1 and AA1.getSelected and AA1.getSelected(label)) or (STATE.Selected and STATE.Selected[label] == true)
                if on then
                    st.Color        = THEME.GREEN
                    st.Thickness    = 2.4
                    st.Transparency = 0
                    glowBar.Visible = true
                else
                    st.Color        = THEME.GREEN_DARK
                    st.Thickness    = 1.6
                    st.Transparency = 0.4
                    glowBar.Visible = false
                end
            end
            update()

            btn.MouseButton1Click:Connect(function()
                local cur = (AA1 and AA1.getSelected and AA1.getSelected(label)) or (STATE.Selected and STATE.Selected[label] == true)
                local newv = not cur
                if AA1 and AA1.setSelected then
                    AA1.setSelected(label, newv)
                else
                    STATE.Selected = STATE.Selected or {}
                    if newv then STATE.Selected[label] = true else STATE.Selected[label] = nil end
                end
                update()
            end)

            table.insert(allButtons, btn)
            return btn
        end

        for i, name in ipairs(POTION_LIST) do
            local b = makeGlowButton(name)
            b.LayoutOrder = i
        end

        -- Lock CanvasPosition.X
        local locking = false
        listHolder:GetPropertyChangedSignal("CanvasPosition"):Connect(function()
            if locking then return end
            locking = true
            local pos = listHolder.CanvasPosition
            if pos.X ~= 0 then
                listHolder.CanvasPosition = Vector2.new(0, pos.Y)
            end
            locking = false
        end)

        -- Search filter
        local function applySearch()
            local q = trim(searchBox.Text or "")
            q = string.lower(q)

            if q == "" then
                for _, btn in ipairs(allButtons) do
                    btn.Visible = true
                end
            else
                for _, btn in ipairs(allButtons) do
                    local text = string.lower(btn.Text or "")
                    btn.Visible = string.find(text, q, 1, true) ~= nil
                end
            end

            listHolder.CanvasPosition = Vector2.new(0, 0)
        end

        searchBox:GetPropertyChangedSignal("Text"):Connect(applySearch)
        searchBox.Focused:Connect(function() sbStroke.Color = THEME.GREEN end)
        searchBox.FocusLost:Connect(function() sbStroke.Color = THEME.GREEN end)

        -- CLOSE ทั้งหน้าจอแบบ “ปิดแน่นอน”
        -- NOTE: ไม่สน gp แล้ว เพื่อให้กดตรงไหนก็ปิดได้จริง
        inputConn = UserInputService.InputBegan:Connect(function(input)
            if not optionsPanel then return end

            local t = input.UserInputType
            if t ~= Enum.UserInputType.MouseButton1 and t ~= Enum.UserInputType.Touch then
                return
            end

            local pos = input.Position
            local keep =
                isInside(optionsPanel, pos)
                or isInside(selectBtn, pos)
                or (searchBox and isInside(searchBox, pos))

            if not keep then
                closePanel()
            end
        end)
    end

    ------------------------------------------------------------------------
    -- Toggle Select Options (เปิดได้เลยตลอด)
    ------------------------------------------------------------------------
    selectBtn.MouseButton1Click:Connect(function()
        if opened then
            closePanel()
        else
            openPanel()
            opened = true
            updateSelectVisual(true)
        end
    end)

    ------------------------------------------------------------------------
    -- INIT SYNC (AA1)
    ------------------------------------------------------------------------
    task.defer(function()
        if AA1 and AA1.ensureRunner then
            AA1.ensureRunner()
        end
        if AA1 and AA1.getEnabled then
            sw1.set(AA1.getEnabled())
        else
            sw1.set(STATE.Enabled == true)
        end
    end)
end)
--===== UFO HUB X • Quest – Buy Event Pickaxe 🎄 (Model A V1 + AA1) =====
-- Tab: Quest
-- Row1 (A V1 Switch): Auto Buy Event Pickaxe
-- AA1: Auto-run from SaveState on UI load (no need to click Quest)
-- Remote:
-- local args = {"Buy Christmas Pickaxe"}
-- ReplicatedStorage.Paper.Remotes.__remotefunction:InvokeServer(unpack(args))

----------------------------------------------------------------------
-- AA1 RUNNER (ไม่มี UI, ทำงานทันทีตอนรันสคริปต์)
----------------------------------------------------------------------
do
    local ReplicatedStorage = game:GetService("ReplicatedStorage")

    -- SAVE (AA1) ใช้ getgenv().UFOX_SAVE
    local SAVE = (getgenv and getgenv().UFOX_SAVE) or {
        get = function(_, _, d) return d end,
        set = function() end,
    }

    local GAME_ID  = tonumber(game.GameId)  or 0
    local PLACE_ID = tonumber(game.PlaceId) or 0
    local BASE_SCOPE = ("AA1/QuestBuyEventPickaxe/%d/%d"):format(GAME_ID, PLACE_ID)

    local function K(field) return BASE_SCOPE .. "/" .. field end

    local function SaveGet(field, default)
        local ok, v = pcall(function()
            return SAVE.get(K(field), default)
        end)
        return ok and v or default
    end

    local function SaveSet(field, value)
        pcall(function()
            SAVE.set(K(field), value)
        end)
    end

    -- STATE
    local STATE = {
        Enabled = SaveGet("Enabled", false),
    }

    -- Remote
    local function getRF()
        local ok, rf = pcall(function()
            return ReplicatedStorage:WaitForChild("Paper")
                :WaitForChild("Remotes")
                :WaitForChild("__remotefunction")
        end)
        if not ok then
            warn("[UFO HUB X • QuestBuyEventPickaxe AA1] cannot get __remotefunction")
            return nil
        end
        return rf
    end

    local function buyOnce()
        local rf = getRF()
        if not rf then return end
        local args = { "Buy Christmas Pickaxe" }
        pcall(function()
            rf:InvokeServer(unpack(args))
        end)
    end

    -- LOOP
    local LOOP_DELAY = 0.35
    local loopRunning = false

    local function startLoop()
        if loopRunning then return end
        loopRunning = true
        task.spawn(function()
            while STATE.Enabled do
                buyOnce()
                task.wait(LOOP_DELAY)
            end
            loopRunning = false
        end)
    end

    local function applyFromState()
        if STATE.Enabled then
            startLoop()
        end
    end

    -- EXPORT AA1
    _G.UFOX_AA1 = _G.UFOX_AA1 or {}
    _G.UFOX_AA1["QuestBuyEventPickaxe"] = {
        state = STATE,
        apply = applyFromState,
        setEnabled = function(v)
            STATE.Enabled = v and true or false
            SaveSet("Enabled", STATE.Enabled)
            applyFromState()
        end,
        saveGet = SaveGet,
        saveSet = SaveSet,
    }

    -- AUTO-RUN: ถ้าเคยเปิดไว้ -> ทำงานทันทีตอนรัน UI หลัก
    task.defer(function()
        applyFromState()
    end)
end

----------------------------------------------------------------------
-- UI PART: Model A V1 ใน Tab Quest (Sync กับ AA1 ด้านบน)
----------------------------------------------------------------------

registerRight("Quest", function(scroll)
    local TweenService = game:GetService("TweenService")

    ------------------------------------------------------------------------
    -- THEME + HELPERS (Model A V1)
    ------------------------------------------------------------------------
    local THEME = {
        GREEN       = Color3.fromRGB(25,255,125),
        GREEN_DARK  = Color3.fromRGB(0,120,60),
        WHITE       = Color3.fromRGB(255,255,255),
        BLACK       = Color3.fromRGB(0,0,0),
        RED         = Color3.fromRGB(255,40,40),
    }

    local function corner(ui, r)
        local c = Instance.new("UICorner")
        c.CornerRadius = UDim.new(0, r or 12)
        c.Parent = ui
        return c
    end

    local function stroke(ui, th, col)
        local s = Instance.new("UIStroke")
        s.Thickness = th or 2.2
        s.Color = col or THEME.GREEN
        s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
        s.Parent = ui
        return s
    end

    local function tween(o, p, d)
        TweenService:Create(
            o,
            TweenInfo.new(d or 0.08, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
            p
        ):Play()
    end

    ------------------------------------------------------------------------
    -- CLEANUP (กันซ้อน)
    ------------------------------------------------------------------------
    for _, name in ipairs({
        "QEV_Header",
        "QEV_Row1",
    }) do
        local o = scroll:FindFirstChild(name)
            or scroll.Parent:FindFirstChild(name)
            or (scroll:FindFirstAncestorOfClass("ScreenGui")
                and scroll:FindFirstAncestorOfClass("ScreenGui"):FindFirstChild(name))
        if o then o:Destroy() end
    end

    ------------------------------------------------------------------------
    -- UIListLayout (A V1 rule: 1 layout + dynamic base)
    ------------------------------------------------------------------------
    local vlist = scroll:FindFirstChildOfClass("UIListLayout")
    if not vlist then
        vlist = Instance.new("UIListLayout")
        vlist.Parent = scroll
        vlist.Padding   = UDim.new(0, 12)
        vlist.SortOrder = Enum.SortOrder.LayoutOrder
    end
    scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y

    local base = 0
    for _, ch in ipairs(scroll:GetChildren()) do
        if ch:IsA("GuiObject") and ch ~= vlist then
            base = math.max(base, ch.LayoutOrder or 0)
        end
    end

    ------------------------------------------------------------------------
    -- HEADER (English + emoji)
    ------------------------------------------------------------------------
    local header = Instance.new("TextLabel")
    header.Name = "QEV_Header"
    header.Parent = scroll
    header.BackgroundTransparency = 1
    header.Size = UDim2.new(1, 0, 0, 36)
    header.Font = Enum.Font.GothamBold
    header.TextSize = 16
    header.TextColor3 = THEME.WHITE
    header.TextXAlignment = Enum.TextXAlignment.Left
    header.Text = "》》》⛏️ Buy Event Pickaxe 🎄《《《"
    header.LayoutOrder = base + 1

    ------------------------------------------------------------------------
    -- Base Row (A V1 card)
    ------------------------------------------------------------------------
    local function makeRow(name, order, labelText)
        local row = Instance.new("Frame")
        row.Name = name
        row.Parent = scroll
        row.Size = UDim2.new(1, -6, 0, 46)
        row.BackgroundColor3 = THEME.BLACK
        corner(row, 12)
        stroke(row, 2.2, THEME.GREEN)
        row.LayoutOrder = order

        local lab = Instance.new("TextLabel")
        lab.Parent = row
        lab.BackgroundTransparency = 1
        lab.Size = UDim2.new(1, -160, 1, 0)
        lab.Position = UDim2.new(0, 16, 0, 0)
        lab.Font = Enum.Font.GothamBold
        lab.TextSize = 13
        lab.TextColor3 = THEME.WHITE
        lab.TextXAlignment = Enum.TextXAlignment.Left
        lab.Text = labelText

        return row, lab
    end

    ------------------------------------------------------------------------
    -- A V1 Switch
    ------------------------------------------------------------------------
    local function makeAV1Switch(parentRow, initialOn, onToggle)
        local sw = Instance.new("Frame")
        sw.Parent = parentRow
        sw.AnchorPoint = Vector2.new(1,0.5)
        sw.Position = UDim2.new(1, -12, 0.5, 0)
        sw.Size = UDim2.fromOffset(52,26)
        sw.BackgroundColor3 = THEME.BLACK
        corner(sw, 13)

        local swStroke = Instance.new("UIStroke")
        swStroke.Parent = sw
        swStroke.Thickness = 1.8

        local knob = Instance.new("Frame")
        knob.Parent = sw
        knob.Size = UDim2.fromOffset(22,22)
        knob.BackgroundColor3 = THEME.WHITE
        knob.Position = UDim2.new(0,2,0.5,-11)
        corner(knob, 11)

        local btn = Instance.new("TextButton")
        btn.Parent = sw
        btn.BackgroundTransparency = 1
        btn.Size = UDim2.fromScale(1,1)
        btn.Text = ""
        btn.AutoButtonColor = false

        local on = initialOn and true or false

        local function update()
            swStroke.Color = on and THEME.GREEN or THEME.RED
            tween(knob, {Position = UDim2.new(on and 1 or 0, on and -24 or 2, 0.5, -11)}, 0.08)
        end

        btn.MouseButton1Click:Connect(function()
            on = not on
            update()
            if onToggle then onToggle(on) end
        end)

        update()
        return function(v)
            on = v and true or false
            update()
        end
    end

    ------------------------------------------------------------------------
    -- Wire to AA1
    ------------------------------------------------------------------------
    local AA1 = _G.UFOX_AA1 and _G.UFOX_AA1["QuestBuyEventPickaxe"]
    local STATE = (AA1 and AA1.state) or { Enabled = false }

    local row1 = makeRow("QEV_Row1", base + 2, "Auto Buy Event Pickaxe")

    local setSwitchVisual = makeAV1Switch(row1, STATE.Enabled, function(on)
        if AA1 and AA1.setEnabled then
            AA1.setEnabled(on)
        end
    end)

    -- Sync visual + ensure AA1 apply (เผื่อ UI เปิดทีหลัง)
    task.defer(function()
        setSwitchVisual(STATE.Enabled)
        if AA1 and AA1.apply then AA1.apply() end
    end)
end)
--===== UFO HUB X • Quest – Christmas Tree 🎄 (Model A V1 + AA1) =====
-- Tab: Quest
-- Row1 (A V1 Switch): Auto Buy & Unlock Christmas Tree
-- AA1: Auto-run from SaveState on UI load (no need to click Quest)
-- Remote sequence each loop:
-- 1) InvokeServer("Buy Christmas Rank")
-- 2) InvokeServer("Claim Christmas Tree", false)

----------------------------------------------------------------------
-- AA1 RUNNER (ไม่มี UI, ทำงานทันทีตอนรันสคริปต์)
----------------------------------------------------------------------
do
    local ReplicatedStorage = game:GetService("ReplicatedStorage")

    -- SAVE (AA1) ใช้ getgenv().UFOX_SAVE
    local SAVE = (getgenv and getgenv().UFOX_SAVE) or {
        get = function(_, _, d) return d end,
        set = function() end,
    }

    local GAME_ID  = tonumber(game.GameId)  or 0
    local PLACE_ID = tonumber(game.PlaceId) or 0
    local BASE_SCOPE = ("AA1/QuestChristmasTree/%d/%d"):format(GAME_ID, PLACE_ID)

    local function K(field) return BASE_SCOPE .. "/" .. field end

    local function SaveGet(field, default)
        local ok, v = pcall(function()
            return SAVE.get(K(field), default)
        end)
        return ok and v or default
    end

    local function SaveSet(field, value)
        pcall(function()
            SAVE.set(K(field), value)
        end)
    end

    -- STATE
    local STATE = {
        Enabled = SaveGet("Enabled", false),
    }

    -- Remote
    local function getRF()
        local ok, rf = pcall(function()
            return ReplicatedStorage:WaitForChild("Paper")
                :WaitForChild("Remotes")
                :WaitForChild("__remotefunction")
        end)
        if not ok then
            warn("[UFO HUB X • QuestChristmasTree AA1] cannot get __remotefunction")
            return nil
        end
        return rf
    end

    local function buyRank()
        local rf = getRF()
        if not rf then return end
        local args = { "Buy Christmas Rank" }
        pcall(function()
            rf:InvokeServer(unpack(args))
        end)
    end

    local function claimTree()
        local rf = getRF()
        if not rf then return end
        local args = { "Claim Christmas Tree", false }
        pcall(function()
            rf:InvokeServer(unpack(args))
        end)
    end

    -- LOOP
    local LOOP_DELAY = 0.45
    local loopRunning = false

    local function startLoop()
        if loopRunning then return end
        loopRunning = true
        task.spawn(function()
            while STATE.Enabled do
                buyRank()
                task.wait(0.12)
                claimTree()
                task.wait(LOOP_DELAY)
            end
            loopRunning = false
        end)
    end

    local function applyFromState()
        if STATE.Enabled then
            startLoop()
        end
    end

    -- EXPORT AA1
    _G.UFOX_AA1 = _G.UFOX_AA1 or {}
    _G.UFOX_AA1["QuestChristmasTree"] = {
        state = STATE,
        apply = applyFromState,
        setEnabled = function(v)
            STATE.Enabled = v and true or false
            SaveSet("Enabled", STATE.Enabled)
            applyFromState()
        end,
        saveGet = SaveGet,
        saveSet = SaveSet,
    }

    -- AUTO-RUN: ถ้าเคยเปิดไว้ -> ทำงานทันทีตอนรัน UI หลัก
    task.defer(function()
        applyFromState()
    end)
end

----------------------------------------------------------------------
-- UI PART: Model A V1 ใน Tab Quest (Sync กับ AA1 ด้านบน)
----------------------------------------------------------------------

registerRight("Quest", function(scroll)
    local TweenService = game:GetService("TweenService")

    ------------------------------------------------------------------------
    -- THEME + HELPERS (Model A V1)
    ------------------------------------------------------------------------
    local THEME = {
        GREEN       = Color3.fromRGB(25,255,125),
        GREEN_DARK  = Color3.fromRGB(0,120,60),
        WHITE       = Color3.fromRGB(255,255,255),
        BLACK       = Color3.fromRGB(0,0,0),
        RED         = Color3.fromRGB(255,40,40),
    }

    local function corner(ui, r)
        local c = Instance.new("UICorner")
        c.CornerRadius = UDim.new(0, r or 12)
        c.Parent = ui
        return c
    end

    local function stroke(ui, th, col)
        local s = Instance.new("UIStroke")
        s.Thickness = th or 2.2
        s.Color = col or THEME.GREEN
        s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
        s.Parent = ui
        return s
    end

    local function tween(o, p, d)
        TweenService:Create(
            o,
            TweenInfo.new(d or 0.08, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
            p
        ):Play()
    end

    ------------------------------------------------------------------------
    -- CLEANUP (กันซ้อน)
    ------------------------------------------------------------------------
    for _, name in ipairs({
        "QCT_Header",
        "QCT_Row1",
    }) do
        local o = scroll:FindFirstChild(name)
            or scroll.Parent:FindFirstChild(name)
            or (scroll:FindFirstAncestorOfClass("ScreenGui")
                and scroll:FindFirstAncestorOfClass("ScreenGui"):FindFirstChild(name))
        if o then o:Destroy() end
    end

    ------------------------------------------------------------------------
    -- UIListLayout (A V1 rule: 1 layout + dynamic base)
    ------------------------------------------------------------------------
    local vlist = scroll:FindFirstChildOfClass("UIListLayout")
    if not vlist then
        vlist = Instance.new("UIListLayout")
        vlist.Parent = scroll
        vlist.Padding   = UDim.new(0, 12)
        vlist.SortOrder = Enum.SortOrder.LayoutOrder
    end
    scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y

    local base = 0
    for _, ch in ipairs(scroll:GetChildren()) do
        if ch:IsA("GuiObject") and ch ~= vlist then
            base = math.max(base, ch.LayoutOrder or 0)
        end
    end

    ------------------------------------------------------------------------
    -- HEADER (English + emoji)
    ------------------------------------------------------------------------
    local header = Instance.new("TextLabel")
    header.Name = "QCT_Header"
    header.Parent = scroll
    header.BackgroundTransparency = 1
    header.Size = UDim2.new(1, 0, 0, 36)
    header.Font = Enum.Font.GothamBold
    header.TextSize = 16
    header.TextColor3 = THEME.WHITE
    header.TextXAlignment = Enum.TextXAlignment.Left
    header.Text = "》》》🎁 Unlock Christmas Tree 🎄《《《"
    header.LayoutOrder = base + 1

    ------------------------------------------------------------------------
    -- Base Row (A V1 card)
    ------------------------------------------------------------------------
    local function makeRow(name, order, labelText)
        local row = Instance.new("Frame")
        row.Name = name
        row.Parent = scroll
        row.Size = UDim2.new(1, -6, 0, 46)
        row.BackgroundColor3 = THEME.BLACK
        corner(row, 12)
        stroke(row, 2.2, THEME.GREEN)
        row.LayoutOrder = order

        local lab = Instance.new("TextLabel")
        lab.Parent = row
        lab.BackgroundTransparency = 1
        lab.Size = UDim2.new(1, -160, 1, 0)
        lab.Position = UDim2.new(0, 16, 0, 0)
        lab.Font = Enum.Font.GothamBold
        lab.TextSize = 13
        lab.TextColor3 = THEME.WHITE
        lab.TextXAlignment = Enum.TextXAlignment.Left
        lab.Text = labelText

        return row, lab
    end

    ------------------------------------------------------------------------
    -- A V1 Switch
    ------------------------------------------------------------------------
    local function makeAV1Switch(parentRow, initialOn, onToggle)
        local sw = Instance.new("Frame")
        sw.Parent = parentRow
        sw.AnchorPoint = Vector2.new(1,0.5)
        sw.Position = UDim2.new(1, -12, 0.5, 0)
        sw.Size = UDim2.fromOffset(52,26)
        sw.BackgroundColor3 = THEME.BLACK
        corner(sw, 13)

        local swStroke = Instance.new("UIStroke")
        swStroke.Parent = sw
        swStroke.Thickness = 1.8

        local knob = Instance.new("Frame")
        knob.Parent = sw
        knob.Size = UDim2.fromOffset(22,22)
        knob.BackgroundColor3 = THEME.WHITE
        knob.Position = UDim2.new(0,2,0.5,-11)
        corner(knob, 11)

        local btn = Instance.new("TextButton")
        btn.Parent = sw
        btn.BackgroundTransparency = 1
        btn.Size = UDim2.fromScale(1,1)
        btn.Text = ""
        btn.AutoButtonColor = false

        local on = initialOn and true or false

        local function update()
            swStroke.Color = on and THEME.GREEN or THEME.RED
            tween(knob, {Position = UDim2.new(on and 1 or 0, on and -24 or 2, 0.5, -11)}, 0.08)
        end

        btn.MouseButton1Click:Connect(function()
            on = not on
            update()
            if onToggle then onToggle(on) end
        end)

        update()
        return function(v)
            on = v and true or false
            update()
        end
    end

    ------------------------------------------------------------------------
    -- Wire to AA1
    ------------------------------------------------------------------------
    local AA1 = _G.UFOX_AA1 and _G.UFOX_AA1["QuestChristmasTree"]
    local STATE = (AA1 and AA1.state) or { Enabled = false }

    local row1 = makeRow("QCT_Row1", base + 2, "Auto Unlock Christmas Tree")

    local setSwitchVisual = makeAV1Switch(row1, STATE.Enabled, function(on)
        if AA1 and AA1.setEnabled then
            AA1.setEnabled(on)
        end
    end)

    -- Sync visual + ensure AA1 apply
    task.defer(function()
        setSwitchVisual(STATE.Enabled)
        if AA1 and AA1.apply then AA1.apply() end
    end)
end)
--===== UFO HUB X • Quest – Auto Event Upgrades 🎁 (AA1 + Model A V1 + V A2 Overlay) =====
-- Row1 (A V1 Switch): Auto Buy Event Upgrades
-- Row2 (A V2 Overlay 100%): Select Event Upgrades (4 buttons, multi-select, click again = cancel)

----------------------------------------------------------------------
-- AA1 RUNNER (ไม่มี UI, ทำงานทันทีตอนรันสคริปต์หลัก)
----------------------------------------------------------------------
do
    local ReplicatedStorage = game:GetService("ReplicatedStorage")
    local HttpService = game:GetService("HttpService")

    local SAVE = (getgenv and getgenv().UFOX_SAVE) or {
        get = function(_, _, d) return d end,
        set = function() end,
    }

    local GAME_ID  = tonumber(game.GameId)  or 0
    local PLACE_ID = tonumber(game.PlaceId) or 0

    -- AA1/QuestEventUpgrades/<GAME>/<PLACE>/(Enabled|SelectedJson)
    local BASE_SCOPE = ("AA1/QuestEventUpgrades/%d/%d"):format(GAME_ID, PLACE_ID)
    local function K(field) return BASE_SCOPE .. "/" .. field end

    local function SaveGet(field, default)
        local ok, v = pcall(function()
            return SAVE.get(K(field), default)
        end)
        return ok and v or default
    end

    local function SaveSet(field, value)
        pcall(function()
            SAVE.set(K(field), value)
        end)
    end

    local UPGRADES = {
        "More Event Damage",
        "More Candy Canes",
        "More Snowflakes",
        "Present Luck",
    }

    local function emptySelected()
        local t = {}
        for _, n in ipairs(UPGRADES) do t[n] = false end
        return t
    end

    local function decodeSelected(json)
        local base = emptySelected()
        if type(json) ~= "string" or json == "" then return base end
        local ok, data = pcall(function() return HttpService:JSONDecode(json) end)
        if not ok or type(data) ~= "table" then return base end
        for k, v in pairs(data) do
            if base[k] ~= nil then base[k] = (v == true) end
        end
        return base
    end

    local function encodeSelected(tbl)
        local out = {}
        if type(tbl) == "table" then
            for _, n in ipairs(UPGRADES) do
                out[n] = (tbl[n] == true)
            end
        end
        local ok, json = pcall(function() return HttpService:JSONEncode(out) end)
        return ok and json or "{}"
    end

    local STATE = {
        Enabled  = (SaveGet("Enabled", false) == true),
        Selected = decodeSelected(SaveGet("SelectedJson", "")),
    }

    local function getRF()
        local ok, rf = pcall(function()
            return ReplicatedStorage:WaitForChild("Paper")
                :WaitForChild("Remotes")
                :WaitForChild("__remotefunction")
        end)
        return (ok and rf) or nil
    end

    local function doUpgrade(name)
        local rf = getRF()
        if not rf then return end
        local args = { "Event Upgrade", tostring(name) }
        pcall(function()
            rf:InvokeServer(unpack(args))
        end)
    end

    local LOOP_SEC = 0.35
    local loopRunning = false

    local function startLoop()
        if loopRunning then return end
        loopRunning = true
        task.spawn(function()
            while STATE.Enabled do
                local did = false
                for _, name in ipairs(UPGRADES) do
                    if not STATE.Enabled then break end
                    if STATE.Selected[name] == true then
                        did = true
                        doUpgrade(name)
                        task.wait(LOOP_SEC)
                    end
                end
                if not did then
                    task.wait(0.30)
                else
                    task.wait(0.05)
                end
            end
            loopRunning = false
        end)
    end

    local function applyFromState()
        if STATE.Enabled then
            startLoop()
        end
    end

    _G.UFOX_AA1 = _G.UFOX_AA1 or {}
    _G.UFOX_AA1["QuestEventUpgrades"] = {
        state = STATE,
        apply = applyFromState,

        setEnabled = function(v)
            STATE.Enabled = (v == true)
            SaveSet("Enabled", STATE.Enabled)
            applyFromState()
        end,

        setSelected = function(name, on)
            if STATE.Selected[name] == nil then return end
            STATE.Selected[name] = (on == true)
            SaveSet("SelectedJson", encodeSelected(STATE.Selected))
        end,

        setSelectedTable = function(tbl)
            if type(tbl) ~= "table" then return end
            for k, _ in pairs(STATE.Selected) do
                STATE.Selected[k] = (tbl[k] == true)
            end
            SaveSet("SelectedJson", encodeSelected(STATE.Selected))
        end,

        clearSelected = function()
            STATE.Selected = emptySelected()
            SaveSet("SelectedJson", encodeSelected(STATE.Selected))
        end,
    }

    task.defer(function()
        applyFromState()
    end)
end

----------------------------------------------------------------------
-- UI PART: Quest (Model A V1 + Model A V2 Overlay) Sync กับ AA1
----------------------------------------------------------------------
registerRight("Quest", function(scroll)
    local TweenService      = game:GetService("TweenService")
    local UserInputService  = game:GetService("UserInputService")
    local HttpService       = game:GetService("HttpService")

    local AA1   = _G.UFOX_AA1 and _G.UFOX_AA1["QuestEventUpgrades"]
    local STATE = (AA1 and AA1.state) or { Enabled=false, Selected={} }

    ------------------------------------------------------------------------
    -- THEME + HELPERS
    ------------------------------------------------------------------------
    local THEME = {
        GREEN       = Color3.fromRGB(25,255,125),
        GREEN_DARK  = Color3.fromRGB(0,120,60),
        WHITE       = Color3.fromRGB(255,255,255),
        BLACK       = Color3.fromRGB(0,0,0),
        RED         = Color3.fromRGB(255,40,40),
    }

    local function corner(ui, r)
        local c = Instance.new("UICorner")
        c.CornerRadius = UDim.new(0, r or 12)
        c.Parent = ui
        return c
    end

    local function stroke(ui, th, col)
        local s = Instance.new("UIStroke")
        s.Thickness = th or 2.2
        s.Color = col or THEME.GREEN
        s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
        s.Parent = ui
        return s
    end

    local function tween(o, p, d)
        TweenService:Create(
            o,
            TweenInfo.new(d or 0.08, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
            p
        ):Play()
    end

    local function trim(s)
        return (tostring(s or ""):gsub("^%s*(.-)%s*$", "%1"))
    end

    ------------------------------------------------------------------------
    -- CLEANUP (กันซ้อน)
    ------------------------------------------------------------------------
    for _, name in ipairs({
        "QEU_Header",
        "QEU_Row1",
        "QEU_Row2",
        "QEU_OptionsPanel",
    }) do
        local o = scroll:FindFirstChild(name)
            or scroll.Parent:FindFirstChild(name)
            or (scroll:FindFirstAncestorOfClass("ScreenGui")
                and scroll:FindFirstAncestorOfClass("ScreenGui"):FindFirstChild(name))
        if o then o:Destroy() end
    end

    ------------------------------------------------------------------------
    -- UIListLayout (Model A V1 Rule)
    ------------------------------------------------------------------------
    local vlist = scroll:FindFirstChildOfClass("UIListLayout")
    if not vlist then
        vlist = Instance.new("UIListLayout")
        vlist.Parent = scroll
        vlist.Padding   = UDim.new(0, 12)
        vlist.SortOrder = Enum.SortOrder.LayoutOrder
    end
    scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y

    local base = 0
    for _, ch in ipairs(scroll:GetChildren()) do
        if ch:IsA("GuiObject") and ch ~= vlist then
            base = math.max(base, ch.LayoutOrder or 0)
        end
    end

    ------------------------------------------------------------------------
    -- HEADER
    ------------------------------------------------------------------------
    local header = Instance.new("TextLabel")
    header.Name = "QEU_Header"
    header.Parent = scroll
    header.BackgroundTransparency = 1
    header.Size = UDim2.new(1, 0, 0, 36)
    header.Font = Enum.Font.GothamBold
    header.TextSize = 16
    header.TextColor3 = THEME.WHITE
    header.TextXAlignment = Enum.TextXAlignment.Left
    header.Text = "》》》🎁 Auto Event Upgrades 🎄《《《"
    header.LayoutOrder = base + 1

    ------------------------------------------------------------------------
    -- Base Row (A V1 card)
    ------------------------------------------------------------------------
    local function makeRow(name, order, labelText)
        local row = Instance.new("Frame")
        row.Name = name
        row.Parent = scroll
        row.Size = UDim2.new(1, -6, 0, 46)
        row.BackgroundColor3 = THEME.BLACK
        corner(row, 12)
        stroke(row, 2.2, THEME.GREEN)
        row.LayoutOrder = order

        local lab = Instance.new("TextLabel")
        lab.Parent = row
        lab.BackgroundTransparency = 1
        lab.Size = UDim2.new(0, 240, 1, 0)
        lab.Position = UDim2.new(0, 16, 0, 0)
        lab.Font = Enum.Font.GothamBold
        lab.TextSize = 13
        lab.TextColor3 = THEME.WHITE
        lab.TextXAlignment = Enum.TextXAlignment.Left
        lab.Text = labelText

        return row, lab
    end

    ------------------------------------------------------------------------
    -- A V1 Switch helper
    ------------------------------------------------------------------------
    local function makeAV1Switch(parentRow, initialOn, onToggle)
        local sw = Instance.new("Frame")
        sw.Parent = parentRow
        sw.AnchorPoint = Vector2.new(1,0.5)
        sw.Position = UDim2.new(1, -16, 0.5, 0)
        sw.Size = UDim2.fromOffset(52,26)
        sw.BackgroundColor3 = THEME.BLACK
        corner(sw, 13)

        local swStroke = Instance.new("UIStroke")
        swStroke.Parent = sw
        swStroke.Thickness = 1.8

        local knob = Instance.new("Frame")
        knob.Parent = sw
        knob.Size = UDim2.fromOffset(22,22)
        knob.BackgroundColor3 = THEME.WHITE
        corner(knob, 11)

        local btn = Instance.new("TextButton")
        btn.Parent = sw
        btn.BackgroundTransparency = 1
        btn.Size = UDim2.fromScale(1,1)
        btn.Text = ""
        btn.AutoButtonColor = false

        local on = initialOn and true or false

        local function update()
            swStroke.Color = on and THEME.GREEN or THEME.RED
            tween(knob, {Position = UDim2.new(on and 1 or 0, on and -24 or 2, 0.5, -11)}, 0.08)
        end

        btn.MouseButton1Click:Connect(function()
            on = not on
            update()
            if onToggle then onToggle(on) end
        end)

        update()
        return {
            set = function(v) on = v and true or false; update() end,
            get = function() return on end,
        }
    end

    ------------------------------------------------------------------------
    -- UPGRADE LIST
    ------------------------------------------------------------------------
    local UPGRADE_LIST = {
        "More Event Damage",
        "More Candy Canes",
        "More Snowflakes",
        "Present Luck",
    }

    STATE.Selected = STATE.Selected or {}
    for _, n in ipairs(UPGRADE_LIST) do
        if STATE.Selected[n] == nil then
            STATE.Selected[n] = false
        end
    end

    ------------------------------------------------------------------------
    -- Row1: Switch (คุม AA1 อย่างเดียว)
    ------------------------------------------------------------------------
    local row1 = makeRow("QEU_Row1", base + 2, "Auto Buy Event Upgrades")

    ------------------------------------------------------------------------
    -- Row2: Overlay (เปิดได้เลย ไม่ต้องพึ่ง Row1)
    ------------------------------------------------------------------------
    local row2 = makeRow("QEU_Row2", base + 3, "Select Event Upgrades")
    local panelParent = scroll.Parent

    local selectBtn = Instance.new("TextButton")
    selectBtn.Name = "QEU_Select"
    selectBtn.Parent = row2
    selectBtn.AnchorPoint = Vector2.new(1, 0.5)
    selectBtn.Position = UDim2.new(1, -16, 0.5, 0)
    selectBtn.Size = UDim2.new(0, 220, 0, 28)
    selectBtn.BackgroundColor3 = THEME.BLACK
    selectBtn.AutoButtonColor = false
    selectBtn.Text = "🔍 Select Options"
    selectBtn.Font = Enum.Font.GothamBold
    selectBtn.TextSize = 13
    selectBtn.TextColor3 = THEME.WHITE
    selectBtn.TextXAlignment = Enum.TextXAlignment.Center
    selectBtn.TextYAlignment = Enum.TextYAlignment.Center
    corner(selectBtn, 8)

    local selectStroke = stroke(selectBtn, 1.8, THEME.GREEN_DARK)
    selectStroke.Transparency = 0.4

    local function updateSelectVisual(isOpen)
        if isOpen then
            selectStroke.Color        = THEME.GREEN
            selectStroke.Thickness    = 2.4
            selectStroke.Transparency = 0
        else
            selectStroke.Color        = THEME.GREEN_DARK
            selectStroke.Thickness    = 1.8
            selectStroke.Transparency = 0.4
        end
    end
    updateSelectVisual(false)

    local padding = Instance.new("UIPadding")
    padding.Parent = selectBtn
    padding.PaddingLeft  = UDim.new(0, 8)
    padding.PaddingRight = UDim.new(0, 26)

    local arrow = Instance.new("TextLabel")
    arrow.Parent = selectBtn
    arrow.AnchorPoint = Vector2.new(1,0.5)
    arrow.Position = UDim2.new(1, -6, 0.5, 0)
    arrow.Size = UDim2.new(0, 18, 0, 18)
    arrow.BackgroundTransparency = 1
    arrow.Font = Enum.Font.GothamBold
    arrow.TextSize = 18
    arrow.TextColor3 = THEME.WHITE
    arrow.Text = "▼"

    ------------------------------------------------------------------------
    -- V A2 Popup Panel (Search + Glow Buttons + CLOSE FULL SCREEN) [MATCH HOME]
    -- ปิดเมื่อแตะ/คลิก/สกอลล์ "ทั้งหน้าจอ" จริงๆ
    -- ยกเว้น: แตะใน optionsPanel / แตะ selectBtn / แตะ searchBox
    ------------------------------------------------------------------------
    local optionsPanel
    local tapConn
    local wheelConn
    local removedConn
    local opened = false
    local searchBox
    local allButtons = {}

    local function isInside(gui, pos)
        if not gui or not gui.Parent then return false end
        local ap = gui.AbsolutePosition
        local as = gui.AbsoluteSize
        return pos.X >= ap.X and pos.X <= ap.X + as.X and pos.Y >= ap.Y and pos.Y <= ap.Y + as.Y
    end

    local function disconnectAll()
        if tapConn then tapConn:Disconnect() tapConn = nil end
        if wheelConn then wheelConn:Disconnect() wheelConn = nil end
        if removedConn then removedConn:Disconnect() removedConn = nil end
    end

    local function closePanel()
        disconnectAll()

        if optionsPanel then
            optionsPanel:Destroy()
            optionsPanel = nil
        end

        searchBox = nil
        allButtons = {}

        opened = false
        updateSelectVisual(false)
    end

    local function bindLife(panel)
        panel.AncestryChanged:Connect(function(_, parent)
            if not parent then
                optionsPanel = nil
                closePanel()
            end
        end)

        removedConn = panelParent.ChildRemoved:Connect(function(ch)
            if ch == panel then
                optionsPanel = nil
                closePanel()
            end
        end)
    end

    local function openPanel()
        closePanel()

        local pw, ph = panelParent.AbsoluteSize.X, panelParent.AbsoluteSize.Y
        local leftRatio   = 0.645
        local topRatio    = 0.02
        local bottomRatio = 0.02
        local rightMargin = 8

        local leftX   = math.floor(pw * leftRatio)
        local topY    = math.floor(ph * topRatio)
        local bottomM = math.floor(ph * bottomRatio)

        local w = pw - leftX - rightMargin
        local h = ph - topY - bottomM

        optionsPanel = Instance.new("Frame")
        optionsPanel.Name = "QEU_OptionsPanel"
        optionsPanel.Parent = panelParent
        optionsPanel.BackgroundColor3 = THEME.BLACK
        optionsPanel.ClipsDescendants = true
        optionsPanel.AnchorPoint = Vector2.new(0, 0)
        optionsPanel.Position    = UDim2.new(0, leftX, 0, topY)
        optionsPanel.Size        = UDim2.new(0, w, 0, h)
        optionsPanel.ZIndex      = 50

        corner(optionsPanel, 12)
        stroke(optionsPanel, 2.4, THEME.GREEN)

        bindLife(optionsPanel)

        local body = Instance.new("Frame")
        body.Name = "Body"
        body.Parent = optionsPanel
        body.BackgroundTransparency = 1
        body.BorderSizePixel = 0
        body.Position = UDim2.new(0, 4, 0, 4)
        body.Size     = UDim2.new(1, -8, 1, -8)
        body.ZIndex   = optionsPanel.ZIndex + 1

        -- Search Box
        searchBox = Instance.new("TextBox")
        searchBox.Name = "SearchBox"
        searchBox.Parent = body
        searchBox.BackgroundColor3 = THEME.BLACK
        searchBox.ClearTextOnFocus = false
        searchBox.Font = Enum.Font.GothamBold
        searchBox.TextSize = 14
        searchBox.TextColor3 = THEME.WHITE
        searchBox.PlaceholderText = "🔍 Search"
        searchBox.TextXAlignment = Enum.TextXAlignment.Center
        searchBox.Text = ""
        searchBox.ZIndex = body.ZIndex + 1
        searchBox.Size = UDim2.new(1, 0, 0, 32)
        searchBox.Position = UDim2.new(0, 0, 0, 0)
        corner(searchBox, 8)

        local sbStroke = stroke(searchBox, 1.8, THEME.GREEN)
        sbStroke.ZIndex = searchBox.ZIndex + 1

        -- List
        local listHolder = Instance.new("ScrollingFrame")
        listHolder.Name = "UList"
        listHolder.Parent = body
        listHolder.BackgroundColor3 = THEME.BLACK
        listHolder.BorderSizePixel = 0
        listHolder.ScrollBarThickness = 0
        listHolder.AutomaticCanvasSize = Enum.AutomaticSize.Y
        listHolder.CanvasSize = UDim2.new(0,0,0,0)
        listHolder.ZIndex = body.ZIndex + 1
        listHolder.ScrollingDirection = Enum.ScrollingDirection.Y
        listHolder.ClipsDescendants = true

        local listTopOffset = 32 + 10
        listHolder.Position = UDim2.new(0, 0, 0, listTopOffset)
        listHolder.Size     = UDim2.new(1, 0, 1, -(listTopOffset + 4))

        local listLayout = Instance.new("UIListLayout")
        listLayout.Parent = listHolder
        listLayout.Padding = UDim.new(0, 8)
        listLayout.SortOrder = Enum.SortOrder.LayoutOrder
        listLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center

        local listPadding = Instance.new("UIPadding")
        listPadding.Parent = listHolder
        listPadding.PaddingTop = UDim.new(0, 6)
        listPadding.PaddingBottom = UDim.new(0, 6)
        listPadding.PaddingLeft = UDim.new(0, 4)
        listPadding.PaddingRight = UDim.new(0, 4)

        -- Glow Button (multi-select + sync AA1)
        allButtons = {}

        local function makeGlowButton(label)
            local btn = Instance.new("TextButton")
            btn.Name = "Btn_" .. label
            btn.Parent = listHolder
            btn.Size = UDim2.new(1, 0, 0, 28)

            btn.BackgroundColor3 = THEME.BLACK
            btn.AutoButtonColor = false
            btn.Font = Enum.Font.GothamBold
            btn.TextSize = 14
            btn.TextColor3 = THEME.WHITE
            btn.Text = label
            btn.ZIndex = listHolder.ZIndex + 1
            btn.TextXAlignment = Enum.TextXAlignment.Center
            btn.TextYAlignment = Enum.TextYAlignment.Center
            corner(btn, 6)

            local st = stroke(btn, 1.6, THEME.GREEN_DARK)
            st.Transparency = 0.4
            st.ZIndex = btn.ZIndex + 1

            local glowBar = Instance.new("Frame")
            glowBar.Name = "GlowBar"
            glowBar.Parent = btn
            glowBar.BackgroundColor3 = THEME.GREEN
            glowBar.BorderSizePixel = 0
            glowBar.Size = UDim2.new(0, 3, 1, 0)
            glowBar.Position = UDim2.new(0, 0, 0, 0)
            glowBar.ZIndex = btn.ZIndex + 2
            glowBar.Visible = false

            local function update()
                local on = (STATE.Selected[label] == true)
                if on then
                    st.Color        = THEME.GREEN
                    st.Thickness    = 2.4
                    st.Transparency = 0
                    glowBar.Visible = true
                else
                    st.Color        = THEME.GREEN_DARK
                    st.Thickness    = 1.6
                    st.Transparency = 0.4
                    glowBar.Visible = false
                end
            end
            update()

            btn.MouseButton1Click:Connect(function()
                local newOn = not (STATE.Selected[label] == true)
                STATE.Selected[label] = newOn
                if AA1 and AA1.setSelected then
                    AA1.setSelected(label, newOn)
                end
                update()
            end)

            table.insert(allButtons, btn)
            return btn
        end

        for i, name in ipairs(UPGRADE_LIST) do
            local b = makeGlowButton(name)
            b.LayoutOrder = i
        end

        -- Lock CanvasPosition.X
        local locking = false
        listHolder:GetPropertyChangedSignal("CanvasPosition"):Connect(function()
            if locking then return end
            locking = true
            local pos = listHolder.CanvasPosition
            if pos.X ~= 0 then
                listHolder.CanvasPosition = Vector2.new(0, pos.Y)
            end
            locking = false
        end)

        -- Search filter
        local function applySearch()
            local q = string.lower(trim(searchBox.Text))
            if q == "" then
                for _, btn in ipairs(allButtons) do btn.Visible = true end
            else
                for _, btn in ipairs(allButtons) do
                    local text = string.lower(btn.Text or "")
                    btn.Visible = (string.find(text, q, 1, true) ~= nil)
                end
            end
            listHolder.CanvasPosition = Vector2.new(0, 0)
        end

        searchBox:GetPropertyChangedSignal("Text"):Connect(applySearch)
        searchBox.Focused:Connect(function() sbStroke.Color = THEME.GREEN end)
        searchBox.FocusLost:Connect(function() sbStroke.Color = THEME.GREEN end)

        -- ✅ CLOSE ทั้งหน้าจอแบบ Home: ไม่สน gp → กดตรงไหนก็ปิดได้จริง
        tapConn = UserInputService.InputBegan:Connect(function(input)
            if not optionsPanel then return end

            local t = input.UserInputType
            if t ~= Enum.UserInputType.MouseButton1 and t ~= Enum.UserInputType.Touch then
                return
            end

            local pos = input.Position
            local keep =
                isInside(optionsPanel, pos)
                or isInside(selectBtn, pos)
                or (searchBox and isInside(searchBox, pos))

            if not keep then
                closePanel()
            end
        end)

        wheelConn = UserInputService.InputChanged:Connect(function(input)
            if not optionsPanel then return end
            if input.UserInputType ~= Enum.UserInputType.MouseWheel then return end

            local pos = UserInputService:GetMouseLocation()
            local keep =
                isInside(optionsPanel, pos)
                or isInside(selectBtn, pos)
                or (searchBox and isInside(searchBox, pos))

            if not keep then
                closePanel()
            end
        end)

        opened = true
        updateSelectVisual(true)
    end

    ------------------------------------------------------------------------
    -- Wire Row1 switch -> AA1 (คุมระบบวิ่งอย่างเดียว)
    ------------------------------------------------------------------------
    local sw1 = makeAV1Switch(row1, STATE.Enabled, function(on)
        if AA1 and AA1.setEnabled then
            AA1.setEnabled(on)
        else
            STATE.Enabled = (on == true)
        end
    end)

    task.defer(function()
        sw1.set(STATE.Enabled)
    end)

    ------------------------------------------------------------------------
    -- Select Options toggle (เปิด/ปิด แผงขวา)  [Row2 เปิดได้เลย]
    ------------------------------------------------------------------------
    selectBtn.MouseButton1Click:Connect(function()
        if opened then
            closePanel()
        else
            openPanel()
        end
    end)
end)
--===== UFO HUB X • Shop – Auto Sell (Model A V1 + AA1) =====
-- Tab: Shop
-- Header: Auto Sell 💰
-- Row1: Auto Sell Ores (สวิตช์เปิด/ปิด)
-- ใช้ Remote:
--   local args = { "Sell All Ores" }
--   __remotefunction:InvokeServer(unpack(args))
-- มีระบบเซฟ AA1 + Auto-Run จาก SaveState

---------------------------------------------------------------------
-- 1) AA1 • ShopAutoSell (Global Auto-Run)
---------------------------------------------------------------------
do
    local ReplicatedStorage = game:GetService("ReplicatedStorage")

    -----------------------------------------------------------------
    -- SAVE (UFOX_SAVE)
    -----------------------------------------------------------------
    local SAVE = (getgenv and getgenv().UFOX_SAVE) or {
        get = function(_, _, d) return d end,
        set = function() end,
    }

    local GAME_ID  = tonumber(game.GameId)  or 0
    local PLACE_ID = tonumber(game.PlaceId) or 0

    -- AA1/ShopAutoSell/<GAME>/<PLACE>/Enabled
    local SYSTEM_NAME = "ShopAutoSell"
    local BASE_SCOPE  = ("AA1/%s/%d/%d"):format(SYSTEM_NAME, GAME_ID, PLACE_ID)

    local function K(field)
        return BASE_SCOPE .. "/" .. field
    end

    local function SaveGet(field, default)
        local ok, v = pcall(function()
            return SAVE.get(K(field), default)
        end)
        return ok and v or default
    end

    local function SaveSet(field, value)
        pcall(function()
            SAVE.set(K(field), value)
        end)
    end

    -----------------------------------------------------------------
    -- STATE + CONFIG
    -----------------------------------------------------------------
    local STATE = {
        Enabled = SaveGet("Enabled", false),
    }

    -- ระยะเวลาขายออโต้ (วินาทีต่อครั้ง)
    local SELL_INTERVAL = 5

    -----------------------------------------------------------------
    -- ฟังก์ชันขาย 1 ครั้ง
    -----------------------------------------------------------------
    local function sellOnce()
        local ok, err = pcall(function()
            local paper   = ReplicatedStorage:WaitForChild("Paper")
            local remotes = paper:WaitForChild("Remotes")
            local rf      = remotes:WaitForChild("__remotefunction")

            local args = { "Sell All Ores" }
            rf:InvokeServer(unpack(args))
        end)

        if not ok then
            warn("[UFO HUB X • ShopAutoSell] sellOnce error:", err)
        end
    end

    -----------------------------------------------------------------
    -- applyFromState + loop
    -----------------------------------------------------------------
    local running = false

    local function applyFromState()
        if not STATE.Enabled then
            -- ปิดระบบ → ปล่อยให้ loop จบเอง
            return
        end

        -- ถ้ามี loop อยู่แล้ว ไม่ต้องสร้างซ้ำ
        if running then return end
        running = true

        task.spawn(function()
            while STATE.Enabled do
                sellOnce()
                task.wait(SELL_INTERVAL)
            end
            running = false
        end)
    end

    local function SetEnabled(v)
        STATE.Enabled = v and true or false
        SaveSet("Enabled", STATE.Enabled)
        task.defer(applyFromState)
    end

    -----------------------------------------------------------------
    -- AA1 Auto-Run ตอนโหลดสคริปต์
    -----------------------------------------------------------------
    task.defer(function()
        applyFromState()
    end)

    -----------------------------------------------------------------
    -- export ให้ UI เรียกใช้
    -----------------------------------------------------------------
    _G.UFOX_AA1 = _G.UFOX_AA1 or {}
    _G.UFOX_AA1[SYSTEM_NAME] = {
        state      = STATE,
        apply      = applyFromState,
        setEnabled = SetEnabled,
        saveGet    = SaveGet,
        saveSet    = SaveSet,
    }
end

---------------------------------------------------------------------
-- 2) UI ฝั่งขวา • Shop (Model A V1)
---------------------------------------------------------------------
registerRight("Shop", function(scroll)
    local TweenService      = game:GetService("TweenService")
    local ReplicatedStorage = game:GetService("ReplicatedStorage")

    -----------------------------------------------------------------
    -- THEME + HELPERS (Model A V1)
    -----------------------------------------------------------------
    local THEME = {
        GREEN = Color3.fromRGB(25,255,125),
        RED   = Color3.fromRGB(255,40,40),
        WHITE = Color3.fromRGB(255,255,255),
        BLACK = Color3.fromRGB(0,0,0),
    }

    local function corner(ui, r)
        local c = Instance.new("UICorner")
        c.CornerRadius = UDim.new(0, r or 12)
        c.Parent = ui
    end

    local function stroke(ui, th, col)
        local s = Instance.new("UIStroke")
        s.Thickness = th or 2.2
        s.Color = col or THEME.GREEN
        s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
        s.Parent = ui
    end

    local function tween(o, p, d)
        TweenService:Create(
            o,
            TweenInfo.new(d or 0.08, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
            p
        ):Play()
    end

    -----------------------------------------------------------------
    -- ดึง AA1 ของ ShopAutoSell (ถ้ามี)
    -----------------------------------------------------------------
    local AA1 = _G.UFOX_AA1 and _G.UFOX_AA1["ShopAutoSell"]
    local savedOn = false
    if AA1 and AA1.state then
        savedOn = AA1.state.Enabled and true or false
    end

    -----------------------------------------------------------------
    -- UIListLayout (Model A V1 rule)
    -----------------------------------------------------------------
    local vlist = scroll:FindFirstChildOfClass("UIListLayout")
    if not vlist then
        vlist = Instance.new("UIListLayout")
        vlist.Parent = scroll
        vlist.Padding   = UDim.new(0, 12)
        vlist.SortOrder = Enum.SortOrder.LayoutOrder
    end
    scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y

    local base = 0
    for _, ch in ipairs(scroll:GetChildren()) do
        if ch:IsA("GuiObject") and ch ~= vlist then
            base = math.max(base, ch.LayoutOrder or 0)
        end
    end

    -----------------------------------------------------------------
    -- HEADER: Auto Sell 💰
    -----------------------------------------------------------------
    local header = Instance.new("TextLabel")
    header.Name = "A1_Shop_AutoSell_Header"
    header.Parent = scroll
    header.BackgroundTransparency = 1
    header.Size = UDim2.new(1, 0, 0, 36)
    header.Font = Enum.Font.GothamBold
    header.TextSize = 16
    header.TextColor3 = THEME.WHITE
    header.TextXAlignment = Enum.TextXAlignment.Left
    header.Text = "》》》Auto Sell 💰《《《"
    header.LayoutOrder = base + 1

    -----------------------------------------------------------------
    -- แถวสวิตช์ Model A V1
    -----------------------------------------------------------------
    local function makeRowSwitch(name, order, labelText, onToggle)
        local row = Instance.new("Frame")
        row.Name = name
        row.Parent = scroll
        row.Size = UDim2.new(1, -6, 0, 46)
        row.BackgroundColor3 = THEME.BLACK
        corner(row, 12)
        stroke(row, 2.2, THEME.GREEN)
        row.LayoutOrder = order

        local lab = Instance.new("TextLabel")
        lab.Parent = row
        lab.BackgroundTransparency = 1
        lab.Size = UDim2.new(1, -160, 1, 0)
        lab.Position = UDim2.new(0, 16, 0, 0)
        lab.Font = Enum.Font.GothamBold
        lab.TextSize = 13
        lab.TextColor3 = THEME.WHITE
        lab.TextXAlignment = Enum.TextXAlignment.Left
        lab.Text = labelText

        local sw = Instance.new("Frame")
        sw.Parent = row
        sw.AnchorPoint = Vector2.new(1,0.5)
        sw.Position = UDim2.new(1, -12, 0.5, 0)
        sw.Size = UDim2.fromOffset(52,26)
        sw.BackgroundColor3 = THEME.BLACK
        corner(sw, 13)

        local swStroke = Instance.new("UIStroke")
        swStroke.Parent = sw
        swStroke.Thickness = 1.8

        local knob = Instance.new("Frame")
        knob.Parent = sw
        knob.Size = UDim2.fromOffset(22,22)
        knob.BackgroundColor3 = THEME.WHITE
        knob.Position = UDim2.new(0,2,0.5,-11)
        corner(knob,11)

        local currentOn = false

        local function updateVisual(on)
            currentOn = on
            swStroke.Color = on and THEME.GREEN or THEME.RED
            tween(knob, {
                Position = UDim2.new(on and 1 or 0, on and -24 or 2, 0.5, -11)
            }, 0.08)
        end

        local function setState(on, fireCallback)
            fireCallback = (fireCallback ~= false)
            if currentOn == on then return end
            updateVisual(on)
            if fireCallback and onToggle then
                onToggle(on)
            end
        end

        local btn = Instance.new("TextButton")
        btn.Parent = sw
        btn.BackgroundTransparency = 1
        btn.Size = UDim2.fromScale(1,1)
        btn.Text = ""
        btn.AutoButtonColor = false
        btn.MouseButton1Click:Connect(function()
            setState(not currentOn, true)
        end)

        updateVisual(false)

        return {
            row      = row,
            setState = setState,
            getState = function() return currentOn end,
        }
    end

    -----------------------------------------------------------------
    -- Row1: Auto Sell Ores (เชื่อมกับ AA1 ShopAutoSell)
    -----------------------------------------------------------------
    local autoSellRow

    autoSellRow = makeRowSwitch("A1_Shop_AutoSell", base + 2, "Auto Sell Ores", function(state)
        if AA1 and AA1.setEnabled then
            AA1.setEnabled(state)
        else
            -- fallback แบบตรง ๆ (เผื่อ AA1 ไม่มี)
            local SAVE = (getgenv and getgenv().UFOX_SAVE) or {
                get = function(_, _, d) return d end,
                set = function() end,
            }
            local GAME_ID  = tonumber(game.GameId)  or 0
            local PLACE_ID = tonumber(game.PlaceId) or 0
            local BASE_SCOPE  = ("AA1/%s/%d/%d"):format("ShopAutoSell", GAME_ID, PLACE_ID)
            local function K(field) return BASE_SCOPE .. "/" .. field end
            pcall(function()
                SAVE.set(K("Enabled"), state and true or false)
            end)
        end
    end)

    -----------------------------------------------------------------
    -- Sync UI จาก STATE เซฟ (เปิดแท็บ Shop ครั้งแรก)
    -----------------------------------------------------------------
    task.defer(function()
        if savedOn and autoSellRow then
            autoSellRow.setState(true, false) -- อัปเดต UI เฉย ๆ ไม่ยิง onToggle ซ้ำ
        end
    end)
end)
--===== UFO HUB X • Shop – Auto Buy Pickaxe & Miners + Auto Buy Auras + Auto Buy Map (Model A V1 + AA1) =====
-- Tab: Shop
-- Row1: Auto Buy Pickaxe -> "Buy Pickaxe"
-- Row2: Auto Buy Miners  -> "Buy Miner"
-- Row3: Auto Buy Auras   -> "Buy Aura" (วน 10 ชื่อ)
-- Row4: Auto Buy Map     -> "Unlock Next World"
-- AA1: Auto-run from SaveState (ไม่ต้องกด Shop)

----------------------------------------------------------------------
-- AA1 RUNNER (ไม่มี UI, ทำงานทันทีตอนรันสคริปต์)
----------------------------------------------------------------------
do
    local ReplicatedStorage = game:GetService("ReplicatedStorage")

    ------------------------------------------------------------------
    -- SAVE (AA1) ใช้ getgenv().UFOX_SAVE
    ------------------------------------------------------------------
    local SAVE = (getgenv and getgenv().UFOX_SAVE) or {
        get = function(_, _, d) return d end,
        set = function() end,
    }

    local GAME_ID  = tonumber(game.GameId)  or 0
    local PLACE_ID = tonumber(game.PlaceId) or 0

    -- AA1/ShopAutoBuy/<GAME>/<PLACE>/AutoPickaxe / AutoMiners
    local BASE_SCOPE = ("AA1/ShopAutoBuy/%d/%d"):format(GAME_ID, PLACE_ID)
    -- AA1/ShopAutoAura/<GAME>/<PLACE>/AutoAura
    local BASE_AURA  = ("AA1/ShopAutoAura/%d/%d"):format(GAME_ID, PLACE_ID)
    -- AA1/ShopAutoMap/<GAME>/<PLACE>/AutoMap
    local BASE_MAP   = ("AA1/ShopAutoMap/%d/%d"):format(GAME_ID, PLACE_ID)

    local function K(scope, field)
        return scope .. "/" .. field
    end

    local function SaveGet(scope, field, default)
        local ok, v = pcall(function()
            return SAVE.get(K(scope, field), default)
        end)
        return ok and v or default
    end

    local function SaveSet(scope, field, value)
        pcall(function()
            SAVE.set(K(scope, field), value)
        end)
    end

    ------------------------------------------------------------------
    -- STATE จาก AA1
    ------------------------------------------------------------------
    local STATE_BUY = {
        AutoPickaxe = SaveGet(BASE_SCOPE, "AutoPickaxe", false),
        AutoMiners  = SaveGet(BASE_SCOPE, "AutoMiners",  false),
    }

    local STATE_AURA = {
        AutoAura = SaveGet(BASE_AURA, "AutoAura", false),
    }

    local STATE_MAP = {
        AutoMap = SaveGet(BASE_MAP, "AutoMap", false),
    }

    ------------------------------------------------------------------
    -- REMOTES: __remotefunction
    ------------------------------------------------------------------
    local function getRemoteFunction()
        local ok, rf = pcall(function()
            local paper   = ReplicatedStorage:WaitForChild("Paper")
            local remotes = paper:WaitForChild("Remotes")
            return remotes:WaitForChild("__remotefunction")
        end)
        if not ok then
            warn("[UFO HUB X • AA1] cannot get __remotefunction")
            return nil
        end
        return rf
    end

    ------------------------------------------------------------------
    -- Buy Pickaxe / Miner
    ------------------------------------------------------------------
    local function buyPickaxeOnce()
        local rf = getRemoteFunction()
        if not rf then return end
        local args = { "Buy Pickaxe" }
        local ok, err = pcall(function()
            rf:InvokeServer(unpack(args))
        end)
        if not ok then
            warn("[UFO HUB X • AutoBuy AA1] Buy Pickaxe error:", err)
        end
    end

    local function buyMinerOnce()
        local rf = getRemoteFunction()
        if not rf then return end
        local args = { "Buy Miner" }
        local ok, err = pcall(function()
            rf:InvokeServer(unpack(args))
        end)
        if not ok then
            warn("[UFO HUB X • AutoBuy AA1] Buy Miner error:", err)
        end
    end

    ------------------------------------------------------------------
    -- Buy Aura (วน 10 ชื่อ)
    ------------------------------------------------------------------
    local AURAS = {
        "Plasma",
        "Toxic Flame",
        "Fire",
        "Water",
        "Shine",
        "Electric",
        "Red",
        "Wind",
        "Rage",
        "Inferno",
    }

    local auraIndex = 1
    local function buyAuraOnce()
        local rf = getRemoteFunction()
        if not rf then return end

        local auraName = AURAS[auraIndex] or "Plasma"
        local args = { "Buy Aura", auraName }

        local ok, err = pcall(function()
            rf:InvokeServer(unpack(args))
        end)
        if not ok then
            warn("[UFO HUB X • AutoAura AA1] Buy Aura error:", err)
        end

        auraIndex += 1
        if auraIndex > #AURAS then
            auraIndex = 1
        end
    end

    ------------------------------------------------------------------
    -- Auto Buy Map (Unlock Next World)
    ------------------------------------------------------------------
    local function buyMapOnce()
        local rf = getRemoteFunction()
        if not rf then return end

        -- ใช้แบบเป๊ะๆ ตามที่ให้มา
        local args = { "Unlock Next World" }
        local ok, err = pcall(function()
            rf:InvokeServer(unpack(args))
        end)
        if not ok then
            warn("[UFO HUB X • AutoMap AA1] Unlock Next World error:", err)
        end
    end

    ------------------------------------------------------------------
    -- LOOP FLAGS (ฝั่ง AA1)
    ------------------------------------------------------------------
    local AUTO_INTERVAL = 5      -- pickaxe/miner ทุก 5 วิ
    local AURA_DELAY    = 1.2    -- ซื้อ aura ทีละชื่อหน่วง 1.2 วิ
    local MAP_DELAY     = 2.0    -- ปลดล็อกแมพหน่วง 2 วิ (กันเด้ง)

    local pickaxeLoopRunning = false
    local minerLoopRunning   = false
    local auraLoopRunning    = false
    local mapLoopRunning     = false

    local function ensurePickaxeLoop()
        if pickaxeLoopRunning then return end
        pickaxeLoopRunning = true
        task.spawn(function()
            while STATE_BUY.AutoPickaxe do
                buyPickaxeOnce()
                for i = 1, AUTO_INTERVAL * 10 do
                    if not STATE_BUY.AutoPickaxe then break end
                    task.wait(0.1)
                end
            end
            pickaxeLoopRunning = false
        end)
    end

    local function ensureMinerLoop()
        if minerLoopRunning then return end
        minerLoopRunning = true
        task.spawn(function()
            while STATE_BUY.AutoMiners do
                buyMinerOnce()
                for i = 1, AUTO_INTERVAL * 10 do
                    if not STATE_BUY.AutoMiners then break end
                    task.wait(0.1)
                end
            end
            minerLoopRunning = false
        end)
    end

    local function ensureAuraLoop()
        if auraLoopRunning then return end
        auraLoopRunning = true
        task.spawn(function()
            while STATE_AURA.AutoAura do
                buyAuraOnce()
                for i = 1, math.floor(AURA_DELAY * 10) do
                    if not STATE_AURA.AutoAura then break end
                    task.wait(0.1)
                end
            end
            auraLoopRunning = false
        end)
    end

    local function ensureMapLoop()
        if mapLoopRunning then return end
        mapLoopRunning = true
        task.spawn(function()
            while STATE_MAP.AutoMap do
                buyMapOnce()
                for i = 1, math.floor(MAP_DELAY * 10) do
                    if not STATE_MAP.AutoMap then break end
                    task.wait(0.1)
                end
            end
            mapLoopRunning = false
        end)
    end

    local function applyFromState()
        if STATE_BUY.AutoPickaxe then ensurePickaxeLoop() end
        if STATE_BUY.AutoMiners  then ensureMinerLoop()   end
        if STATE_AURA.AutoAura   then ensureAuraLoop()    end
        if STATE_MAP.AutoMap     then ensureMapLoop()     end
    end

    ------------------------------------------------------------------
    -- EXPORT AA1 + AUTO-RUN ตอนโหลดสคริปต์หลัก
    ------------------------------------------------------------------
    _G.UFOX_AA1 = _G.UFOX_AA1 or {}

    _G.UFOX_AA1["ShopAutoBuy"] = {
        state = STATE_BUY,
        apply = applyFromState,

        setPickaxe = function(on)
            on = on and true or false
            STATE_BUY.AutoPickaxe = on
            SaveSet(BASE_SCOPE, "AutoPickaxe", on)
            if on then ensurePickaxeLoop() end
        end,

        setMiners = function(on)
            on = on and true or false
            STATE_BUY.AutoMiners = on
            SaveSet(BASE_SCOPE, "AutoMiners", on)
            if on then ensureMinerLoop() end
        end,
    }

    _G.UFOX_AA1["ShopAutoAura"] = {
        state = STATE_AURA,
        apply = applyFromState,

        setAura = function(on)
            on = on and true or false
            STATE_AURA.AutoAura = on
            SaveSet(BASE_AURA, "AutoAura", on)
            if on then ensureAuraLoop() end
        end,
    }

    _G.UFOX_AA1["ShopAutoMap"] = {
        state = STATE_MAP,
        apply = applyFromState,

        setMap = function(on)
            on = on and true or false
            STATE_MAP.AutoMap = on
            SaveSet(BASE_MAP, "AutoMap", on)
            if on then ensureMapLoop() end
        end,
    }

    -- ถ้าเคยเปิด Auto ไว้ → รันเลย (ไม่ต้องกด Shop)
    task.defer(function()
        applyFromState()
    end)
end

----------------------------------------------------------------------
-- UI PART: Model A V1 ใน Tab Shop (คุม STATE ของ AA1)
----------------------------------------------------------------------
registerRight("Shop", function(scroll)
    local TweenService = game:GetService("TweenService")

    ------------------------------------------------------------------------
    -- THEME + HELPERS (Model A V1)
    ------------------------------------------------------------------------
    local THEME = {
        GREEN = Color3.fromRGB(25,255,125),
        RED   = Color3.fromRGB(255,40,40),
        WHITE = Color3.fromRGB(255,255,255),
        BLACK = Color3.fromRGB(0,0,0),
    }

    local function corner(ui, r)
        local c = Instance.new("UICorner")
        c.CornerRadius = UDim.new(0, r or 12)
        c.Parent = ui
    end

    local function stroke(ui, th, col)
        local s = Instance.new("UIStroke")
        s.Thickness = th or 2.2
        s.Color = col or THEME.GREEN
        s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
        s.Parent = ui
    end

    local function tween(o, p, d)
        TweenService:Create(
            o,
            TweenInfo.new(d or 0.08, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
            p
        ):Play()
    end

    ------------------------------------------------------------------------
    -- ดึง STATE จาก AA1
    ------------------------------------------------------------------------
    local AA1_BUY   = _G.UFOX_AA1 and _G.UFOX_AA1["ShopAutoBuy"]
    local AA1_AURA  = _G.UFOX_AA1 and _G.UFOX_AA1["ShopAutoAura"]
    local AA1_MAP   = _G.UFOX_AA1 and _G.UFOX_AA1["ShopAutoMap"]

    local STATE_BUY = (AA1_BUY and AA1_BUY.state) or { AutoPickaxe=false, AutoMiners=false }
    local STATE_AUR = (AA1_AURA and AA1_AURA.state) or { AutoAura=false }
    local STATE_MAP = (AA1_MAP and AA1_MAP.state) or { AutoMap=false }

    ------------------------------------------------------------------------
    -- UIListLayout (Model A V1 rule: 1 layout + base จากของเดิม)
    ------------------------------------------------------------------------
    local vlist = scroll:FindFirstChildOfClass("UIListLayout")
    if not vlist then
        vlist = Instance.new("UIListLayout")
        vlist.Parent = scroll
        vlist.Padding   = UDim.new(0, 12)
        vlist.SortOrder = Enum.SortOrder.LayoutOrder
    end
    scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y

    local base = 0
    for _, ch in ipairs(scroll:GetChildren()) do
        if ch:IsA("GuiObject") and ch ~= vlist then
            base = math.max(base, ch.LayoutOrder or 0)
        end
    end

    ------------------------------------------------------------------------
    -- HEADER
    ------------------------------------------------------------------------
    local header = Instance.new("TextLabel")
    header.Name = "A1_Shop_AutoBuy_Header"
    header.Parent = scroll
    header.BackgroundTransparency = 1
    header.Size = UDim2.new(1, 0, 0, 36)
    header.Font = Enum.Font.GothamBold
    header.TextSize = 16
    header.TextColor3 = THEME.WHITE
    header.TextXAlignment = Enum.TextXAlignment.Left
    header.Text = "》》》Auto Buy 💸《《《"
    header.LayoutOrder = base + 1

    ------------------------------------------------------------------------
    -- Switch Row (Model A V1)
    ------------------------------------------------------------------------
    local function makeRowSwitch(name, order, labelText, initialOn, onToggle)
        local row = Instance.new("Frame")
        row.Name = name
        row.Parent = scroll
        row.Size = UDim2.new(1, -6, 0, 46)
        row.BackgroundColor3 = THEME.BLACK
        corner(row, 12)
        stroke(row, 2.2, THEME.GREEN)
        row.LayoutOrder = order

        local lab = Instance.new("TextLabel")
        lab.Parent = row
        lab.BackgroundTransparency = 1
        lab.Size = UDim2.new(1, -160, 1, 0)
        lab.Position = UDim2.new(0, 16, 0, 0)
        lab.Font = Enum.Font.GothamBold
        lab.TextSize = 13
        lab.TextColor3 = THEME.WHITE
        lab.TextXAlignment = Enum.TextXAlignment.Left
        lab.Text = labelText

        local sw = Instance.new("Frame")
        sw.Parent = row
        sw.AnchorPoint = Vector2.new(1,0.5)
        sw.Position = UDim2.new(1, -12, 0.5, 0)
        sw.Size = UDim2.fromOffset(52,26)
        sw.BackgroundColor3 = THEME.BLACK
        corner(sw, 13)

        local swStroke = Instance.new("UIStroke")
        swStroke.Parent = sw
        swStroke.Thickness = 1.8

        local knob = Instance.new("Frame")
        knob.Parent = sw
        knob.Size = UDim2.fromOffset(22,22)
        knob.BackgroundColor3 = THEME.WHITE
        knob.Position = UDim2.new(0,2,0.5,-11)
        corner(knob,11)

        local currentOn = initialOn and true or false

        local function updateVisual(on)
            currentOn = on
            swStroke.Color = on and THEME.GREEN or THEME.RED
            tween(knob, { Position = UDim2.new(on and 1 or 0, on and -24 or 2, 0.5, -11) }, 0.08)
        end

        local function setState(on, fireCallback)
            fireCallback = (fireCallback ~= false)
            if currentOn == on then return end
            updateVisual(on)
            if fireCallback and onToggle then onToggle(on) end
        end

        local btn = Instance.new("TextButton")
        btn.Parent = sw
        btn.BackgroundTransparency = 1
        btn.Size = UDim2.fromScale(1,1)
        btn.Text = ""
        btn.AutoButtonColor = false
        btn.MouseButton1Click:Connect(function()
            setState(not currentOn, true)
        end)

        updateVisual(currentOn)
        return { setState = setState }
    end

    -- Row1: Auto Buy Pickaxe
    local rowPickaxe = makeRowSwitch(
        "A1_Shop_AutoBuy_Pickaxe",
        base + 2,
        "Auto Buy Pickaxe",
        STATE_BUY.AutoPickaxe,
        function(state)
            if AA1_BUY and AA1_BUY.setPickaxe then AA1_BUY.setPickaxe(state) end
        end
    )

    -- Row2: Auto Buy Miners
    local rowMiner = makeRowSwitch(
        "A1_Shop_AutoBuy_Miners",
        base + 3,
        "Auto Buy Miners",
        STATE_BUY.AutoMiners,
        function(state)
            if AA1_BUY and AA1_BUY.setMiners then AA1_BUY.setMiners(state) end
        end
    )

    -- Row3: Auto Buy Auras
    local rowAura = makeRowSwitch(
        "A1_Shop_AutoBuy_Auras",
        base + 4,
        "Auto Buy Auras",
        STATE_AUR.AutoAura,
        function(state)
            if AA1_AURA and AA1_AURA.setAura then AA1_AURA.setAura(state) end
        end
    )

    -- Row4: Auto Buy Map
    local rowMap = makeRowSwitch(
        "A1_Shop_AutoBuy_Map",
        base + 5,
        "Auto Buy Map",
        STATE_MAP.AutoMap,
        function(state)
            if AA1_MAP and AA1_MAP.setMap then AA1_MAP.setMap(state) end
        end
    )

    -- Sync UI จาก STATE ที่เซฟไว้ (ตอนเปิด Tab Shop)
    task.defer(function()
        rowPickaxe.setState(STATE_BUY.AutoPickaxe, false)
        rowMiner.setState(STATE_BUY.AutoMiners,   false)
        rowAura.setState(STATE_AUR.AutoAura,      false)
        rowMap.setState(STATE_MAP.AutoMap,        false)
    end)
end)
--===== UFO HUB X • Shop – Upgrades Auto ⚡ (Model A V1 + V A2 Overlay + AA1) =====
-- Tab: Shop
-- Row1 (A V1 Switch): Enable Upgrades Auto
-- Row2 (V A2 Overlay): 🔍 Select Options (11 buttons, multi-select, click again = cancel)
-- AA1: Auto-run from SaveState on UI load (no need to click Shop)

----------------------------------------------------------------------
-- SERVICES
----------------------------------------------------------------------
local TweenService      = game:GetService("TweenService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local UserInputService  = game:GetService("UserInputService")

----------------------------------------------------------------------
-- AA1 SAVE (uses getgenv().UFOX_SAVE)
----------------------------------------------------------------------
local SAVE = (getgenv and getgenv().UFOX_SAVE) or {
    get = function(_, _, d) return d end,
    set = function() end,
}

local GAME_ID  = tonumber(game.GameId)  or 0
local PLACE_ID = tonumber(game.PlaceId) or 0
local BASE     = ("AA1/ShopUpgradesAuto/%d/%d"):format(GAME_ID, PLACE_ID)

local function K(field) return BASE .. "/" .. field end

local function SaveGet(field, default)
    local ok, v = pcall(function()
        return SAVE.get(K(field), default)
    end)
    return ok and v or default
end

local function SaveSet(field, value)
    pcall(function()
        SAVE.set(K(field), value)
    end)
end

----------------------------------------------------------------------
-- REMOTE
----------------------------------------------------------------------
local function getRF()
    local ok, rf = pcall(function()
        local paper = ReplicatedStorage:WaitForChild("Paper")
        local rem   = paper:WaitForChild("Remotes")
        return rem:WaitForChild("__remotefunction")
    end)
    if not ok then
        warn("[UFO HUB X • ShopUpgradesAuto] cannot get __remotefunction:", rf)
        return nil
    end
    return rf
end

----------------------------------------------------------------------
-- UPGRADE LIST (11)
----------------------------------------------------------------------
local UPGRADE_NAMES = {
    "More Gems",
    "More Rebirths",
    "More Coins",
    "More Damage",
    "Egg Luck",
    "Hatch Speed",
    "Pets Equipped",
    "Inventory Space",
    "Rainbow Chance",
    "Golden Chance",
    "Walk Speed",
}

----------------------------------------------------------------------
-- STATE + EXPORT (AA1)
----------------------------------------------------------------------
_G.UFOX_AA1 = _G.UFOX_AA1 or {}
_G.UFOX_AA1["ShopUpgradesAuto"] = _G.UFOX_AA1["ShopUpgradesAuto"] or {}

local SYS = _G.UFOX_AA1["ShopUpgradesAuto"]

SYS.STATE = SYS.STATE or {
    Enabled  = SaveGet("Enabled", false),
    Selected = SaveGet("Selected", {}), -- table: {["More Gems"]=true, ...}
}

local STATE = SYS.STATE

if type(STATE.Selected) ~= "table" then STATE.Selected = {} end
for k,v in pairs(STATE.Selected) do
    if v ~= true then STATE.Selected[k] = nil end
end

----------------------------------------------------------------------
-- RUNNER (PERMA LOOP)
----------------------------------------------------------------------
local UPGRADE_DELAY = 0.35

local function doUpgradeOnce(name)
    local rf = getRF()
    if not rf then return end
    local args = { "Upgrade", name }
    local ok, err = pcall(function()
        rf:InvokeServer(unpack(args))
    end)
    if not ok then
        warn("[UFO HUB X • ShopUpgradesAuto] Upgrade error ("..tostring(name).."):", err)
    end
end

local runnerStarted = false
local function ensureRunner()
    if runnerStarted then return end
    runnerStarted = true
    task.spawn(function()
        while true do
            if STATE.Enabled then
                for _, name in ipairs(UPGRADE_NAMES) do
                    if not STATE.Enabled then break end
                    if STATE.Selected[name] == true then
                        doUpgradeOnce(name)
                        task.wait(UPGRADE_DELAY)
                    end
                end
                task.wait(0.15)
            else
                task.wait(0.25)
            end
        end
    end)
end

local function setEnabled(v)
    v = v and true or false
    STATE.Enabled = v
    SaveSet("Enabled", v)
end

local function setSelected(name, v)
    if v then
        STATE.Selected[name] = true
    else
        STATE.Selected[name] = nil
    end
    SaveSet("Selected", STATE.Selected)
end

SYS.setEnabled  = setEnabled
SYS.setSelected = setSelected
SYS.getEnabled  = function() return STATE.Enabled end
SYS.getSelected = function(name) return STATE.Selected[name] == true end

-- AUTO-RUN from AA1
task.defer(function()
    ensureRunner()
end)

----------------------------------------------------------------------
-- UI (Shop) — Model A V1 + V A2 Overlay (ของจริง)
----------------------------------------------------------------------
registerRight("Shop", function(scroll)

    ------------------------------------------------------------------------
    -- THEME + HELPERS (Model A V1 / V A2)
    ------------------------------------------------------------------------
    local THEME = {
        GREEN       = Color3.fromRGB(25,255,125),
        GREEN_DARK  = Color3.fromRGB(0,120,60),
        WHITE       = Color3.fromRGB(255,255,255),
        BLACK       = Color3.fromRGB(0,0,0),
    }

    local function corner(ui, r)
        local c = Instance.new("UICorner")
        c.CornerRadius = UDim.new(0, r or 12)
        c.Parent = ui
    end

    local function stroke(ui, th, col)
        local s = Instance.new("UIStroke")
        s.Thickness = th or 2.2
        s.Color = col or THEME.GREEN
        s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
        s.Parent = ui
        return s
    end

    local function tween(o, p, d)
        TweenService:Create(
            o,
            TweenInfo.new(d or 0.08, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
            p
        ):Play()
    end

    ------------------------------------------------------------------------
    -- CLEANUP เฉพาะของ V A2 เดิม (กันซ้อน)
    ------------------------------------------------------------------------
    for _, name in ipairs({"SHVA2_Header","SHVA2_Row1","SHVA2_OptionsPanel","SHVA2_Row_Enable"}) do
        local o = scroll:FindFirstChild(name)
            or scroll.Parent:FindFirstChild(name)
            or (scroll:FindFirstAncestorOfClass("ScreenGui")
                and scroll:FindFirstAncestorOfClass("ScreenGui"):FindFirstChild(name))
        if o then o:Destroy() end
    end

    ------------------------------------------------------------------------
    -- UIListLayout (A V1 rule: 1 layout + dynamic base)
    ------------------------------------------------------------------------
    local vlist = scroll:FindFirstChildOfClass("UIListLayout")
    if not vlist then
        vlist = Instance.new("UIListLayout")
        vlist.Parent = scroll
        vlist.Padding   = UDim.new(0, 12)
        vlist.SortOrder = Enum.SortOrder.LayoutOrder
    end
    scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y

    local base = 0
    for _, ch in ipairs(scroll:GetChildren()) do
        if ch:IsA("GuiObject") and ch ~= vlist then
            base = math.max(base, ch.LayoutOrder or 0)
        end
    end

    ------------------------------------------------------------------------
    -- HEADER
    ------------------------------------------------------------------------
    local header = Instance.new("TextLabel")
    header.Name = "SHVA2_Header"
    header.Parent = scroll
    header.BackgroundTransparency = 1
    header.Size = UDim2.new(1, 0, 0, 36)
    header.Font = Enum.Font.GothamBold
    header.TextSize = 16
    header.TextColor3 = THEME.WHITE
    header.TextXAlignment = Enum.TextXAlignment.Left
    header.Text = "》》》Upgrades Auto ⚡《《《"
    header.LayoutOrder = base + 1

    ------------------------------------------------------------------------
    -- Base Row (A V1 card)
    ------------------------------------------------------------------------
    local function makeRow(name, order, labelText)
        local row = Instance.new("Frame")
        row.Name = name
        row.Parent = scroll
        row.Size = UDim2.new(1, -6, 0, 46)
        row.BackgroundColor3 = THEME.BLACK
        corner(row, 12)
        stroke(row, 2.2, THEME.GREEN)
        row.LayoutOrder = order

        local lab = Instance.new("TextLabel")
        lab.Parent = row
        lab.BackgroundTransparency = 1
        lab.Size = UDim2.new(0, 220, 1, 0)
        lab.Position = UDim2.new(0, 16, 0, 0)
        lab.Font = Enum.Font.GothamBold
        lab.TextSize = 13
        lab.TextColor3 = THEME.WHITE
        lab.TextXAlignment = Enum.TextXAlignment.Left
        lab.Text = labelText

        return row, lab
    end

    ------------------------------------------------------------------------
    -- Row1: A V1 Switch (Enable)
    ------------------------------------------------------------------------
    local function makeAV1Switch(parentRow)
        local sw = Instance.new("Frame")
        sw.Parent = parentRow
        sw.AnchorPoint = Vector2.new(1,0.5)
        sw.Position = UDim2.new(1, -16, 0.5, 0)
        sw.Size = UDim2.fromOffset(52,26)
        sw.BackgroundColor3 = THEME.BLACK
        corner(sw, 13)

        local swStroke = Instance.new("UIStroke")
        swStroke.Parent = sw
        swStroke.Thickness = 1.8

        local knob = Instance.new("Frame")
        knob.Parent = sw
        knob.Size = UDim2.fromOffset(22,22)
        knob.BackgroundColor3 = THEME.WHITE
        corner(knob, 11)

        local btn = Instance.new("TextButton")
        btn.Parent = sw
        btn.BackgroundTransparency = 1
        btn.Size = UDim2.fromScale(1,1)
        btn.Text = ""
        btn.AutoButtonColor = false

        local on = STATE.Enabled and true or false

        local function update()
            swStroke.Color = on and THEME.GREEN or Color3.fromRGB(255,40,40)
            tween(knob, {Position = UDim2.new(on and 1 or 0, on and -24 or 2, 0.5, -11)}, 0.08)
        end

        btn.MouseButton1Click:Connect(function()
            on = not on
            SYS.setEnabled(on)
            ensureRunner()
            update()
        end)

        update()
        return function(v)
            on = v and true or false
            update()
        end
    end

    local rowEnable = makeRow("SHVA2_Row_Enable", base + 2, "Enable Upgrades Auto")
    local setEnableVisual = makeAV1Switch(rowEnable)

    ------------------------------------------------------------------------
    -- Row2: Select Options button (V A2 ของจริง)
    ------------------------------------------------------------------------
    local rowSelect = makeRow("SHVA2_Row1", base + 3, "Select Upgrades")
    local panelParent = scroll.Parent -- กรอบขวาของ Shop

    local selectBtn = Instance.new("TextButton")
    selectBtn.Name = "SHVA2_Select"
    selectBtn.Parent = rowSelect
    selectBtn.AnchorPoint = Vector2.new(1, 0.5)
    selectBtn.Position = UDim2.new(1, -16, 0.5, 0)
    selectBtn.Size = UDim2.new(0, 220, 0, 28)
    selectBtn.BackgroundColor3 = THEME.BLACK
    selectBtn.AutoButtonColor = false
    selectBtn.Text = "🔍 Select Options"
    selectBtn.Font = Enum.Font.GothamBold
    selectBtn.TextSize = 13
    selectBtn.TextColor3 = THEME.WHITE
    selectBtn.TextXAlignment = Enum.TextXAlignment.Center
    selectBtn.TextYAlignment = Enum.TextYAlignment.Center
    corner(selectBtn, 8)

    local selectStroke = stroke(selectBtn, 1.8, THEME.GREEN_DARK)
    selectStroke.Transparency = 0.4

    local function updateSelectVisual(isOpen)
        if isOpen then
            selectStroke.Color        = THEME.GREEN
            selectStroke.Thickness    = 2.4
            selectStroke.Transparency = 0
        else
            selectStroke.Color        = THEME.GREEN_DARK
            selectStroke.Thickness    = 1.8
            selectStroke.Transparency = 0.4
        end
    end
    updateSelectVisual(false)

    local padding = Instance.new("UIPadding")
    padding.Parent = selectBtn
    padding.PaddingLeft  = UDim.new(0, 8)
    padding.PaddingRight = UDim.new(0, 26)

    local arrow = Instance.new("TextLabel")
    arrow.Parent = selectBtn
    arrow.AnchorPoint = Vector2.new(1,0.5)
    arrow.Position = UDim2.new(1, -6, 0.5, 0)
    arrow.Size = UDim2.new(0, 18, 0, 18)
    arrow.BackgroundTransparency = 1
    arrow.Font = Enum.Font.GothamBold
    arrow.TextSize = 18
    arrow.TextColor3 = THEME.WHITE
    arrow.Text = "▼"

    ------------------------------------------------------------------------
    -- Popup Panel + GLOBAL CLICK CLOSE (ทั้งหน้าจอจริง)
    -- กดตรงไหนก็ปิด ยกเว้น: optionsPanel / selectBtn / searchBox
    ------------------------------------------------------------------------
    local optionsPanel
    local inputConn
    local opened = false
    local searchBox -- keep ref for exception

    local function isInside(gui, pos)
        if not gui or not gui.Parent then return false end
        local ap = gui.AbsolutePosition
        local as = gui.AbsoluteSize
        return pos.X >= ap.X and pos.X <= ap.X + as.X and pos.Y >= ap.Y and pos.Y <= ap.Y + as.Y
    end

    local function disconnectInput()
        if inputConn then
            inputConn:Disconnect()
            inputConn = nil
        end
    end

    local function closePanel()
        if optionsPanel then
            optionsPanel:Destroy()
            optionsPanel = nil
        end
        searchBox = nil
        disconnectInput()
        opened = false
        updateSelectVisual(false)
    end

    local function openPanel()
        closePanel()

        local pw, ph = panelParent.AbsoluteSize.X, panelParent.AbsoluteSize.Y
        local leftRatio   = 0.645
        local topRatio    = 0.02
        local bottomRatio = 0.02
        local rightMargin = 8

        local leftX   = math.floor(pw * leftRatio)
        local topY    = math.floor(ph * topRatio)
        local bottomM = math.floor(ph * bottomRatio)

        local w = pw - leftX - rightMargin
        local h = ph - topY - bottomM

        optionsPanel = Instance.new("Frame")
        optionsPanel.Name = "SHVA2_OptionsPanel"
        optionsPanel.Parent = panelParent
        optionsPanel.BackgroundColor3 = THEME.BLACK
        optionsPanel.ClipsDescendants = true
        optionsPanel.AnchorPoint = Vector2.new(0, 0)
        optionsPanel.Position    = UDim2.new(0, leftX, 0, topY)
        optionsPanel.Size        = UDim2.new(0, w, 0, h)
        optionsPanel.ZIndex      = 50

        corner(optionsPanel, 12)
        stroke(optionsPanel, 2.4, THEME.GREEN)

        local body = Instance.new("Frame")
        body.Name = "Body"
        body.Parent = optionsPanel
        body.BackgroundTransparency = 1
        body.BorderSizePixel = 0
        body.Position = UDim2.new(0, 4, 0, 4)
        body.Size     = UDim2.new(1, -8, 1, -8)
        body.ZIndex   = optionsPanel.ZIndex + 1

        -- Search Box
        searchBox = Instance.new("TextBox")
        searchBox.Name = "SearchBox"
        searchBox.Parent = body
        searchBox.BackgroundColor3 = THEME.BLACK
        searchBox.ClearTextOnFocus = false
        searchBox.Font = Enum.Font.GothamBold
        searchBox.TextSize = 14
        searchBox.TextColor3 = THEME.WHITE
        searchBox.PlaceholderText = "🔍 Search"
        searchBox.TextXAlignment = Enum.TextXAlignment.Center
        searchBox.Text = ""
        searchBox.ZIndex = body.ZIndex + 1
        searchBox.Size = UDim2.new(1, 0, 0, 32)
        searchBox.Position = UDim2.new(0, 0, 0, 0)
        corner(searchBox, 8)

        local sbStroke = stroke(searchBox, 1.8, THEME.GREEN)
        sbStroke.ZIndex = searchBox.ZIndex + 1

        -- List
        local listHolder = Instance.new("ScrollingFrame")
        listHolder.Name = "UList"
        listHolder.Parent = body
        listHolder.BackgroundColor3 = THEME.BLACK
        listHolder.BorderSizePixel = 0
        listHolder.ScrollBarThickness = 0
        listHolder.AutomaticCanvasSize = Enum.AutomaticSize.Y
        listHolder.CanvasSize = UDim2.new(0,0,0,0)
        listHolder.ZIndex = body.ZIndex + 1
        listHolder.ScrollingDirection = Enum.ScrollingDirection.Y
        listHolder.ClipsDescendants = true

        local listTopOffset = 32 + 10
        listHolder.Position = UDim2.new(0, 0, 0, listTopOffset)
        listHolder.Size     = UDim2.new(1, 0, 1, -(listTopOffset + 4))

        local listLayout = Instance.new("UIListLayout")
        listLayout.Parent = listHolder
        listLayout.Padding = UDim.new(0, 8)
        listLayout.SortOrder = Enum.SortOrder.LayoutOrder
        listLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center

        local listPadding = Instance.new("UIPadding")
        listPadding.Parent = listHolder
        listPadding.PaddingTop = UDim.new(0, 6)
        listPadding.PaddingBottom = UDim.new(0, 6)
        listPadding.PaddingLeft = UDim.new(0, 4)
        listPadding.PaddingRight = UDim.new(0, 4)

        local allButtons = {}

        local function makeGlowButton(label, initialOn)
            local btn = Instance.new("TextButton")
            btn.Name = "Btn_" .. label
            btn.Parent = listHolder
            btn.Size = UDim2.new(1, 0, 0, 28)

            btn.BackgroundColor3 = THEME.BLACK
            btn.AutoButtonColor = false
            btn.Font = Enum.Font.GothamBold
            btn.TextSize = 14
            btn.TextColor3 = THEME.WHITE
            btn.Text = label
            btn.ZIndex = listHolder.ZIndex + 1
            btn.TextXAlignment = Enum.TextXAlignment.Center
            btn.TextYAlignment = Enum.TextYAlignment.Center
            corner(btn, 6)

            local st = stroke(btn, 1.6, THEME.GREEN_DARK)
            st.Transparency = 0.4

            local glowBar = Instance.new("Frame")
            glowBar.Name = "GlowBar"
            glowBar.Parent = btn
            glowBar.BackgroundColor3 = THEME.GREEN
            glowBar.BorderSizePixel = 0
            glowBar.Size = UDim2.new(0, 3, 1, 0)
            glowBar.Position = UDim2.new(0, 0, 0, 0)
            glowBar.ZIndex = btn.ZIndex + 1
            glowBar.Visible = false

            local on = initialOn and true or false
            local function update()
                if on then
                    st.Color        = THEME.GREEN
                    st.Thickness    = 2.4
                    st.Transparency = 0
                    glowBar.Visible = true
                else
                    st.Color        = THEME.GREEN_DARK
                    st.Thickness    = 1.6
                    st.Transparency = 0.4
                    glowBar.Visible = false
                end
            end
            update()

            btn.MouseButton1Click:Connect(function()
                on = not on
                SYS.setSelected(label, on)
                update()
            end)

            table.insert(allButtons, btn)
            return btn
        end

        for i, name in ipairs(UPGRADE_NAMES) do
            local b = makeGlowButton(name, STATE.Selected[name] == true)
            b.LayoutOrder = i
        end

        -- Lock CanvasPosition.X
        local locking = false
        listHolder:GetPropertyChangedSignal("CanvasPosition"):Connect(function()
            if locking then return end
            locking = true
            local pos = listHolder.CanvasPosition
            if pos.X ~= 0 then
                listHolder.CanvasPosition = Vector2.new(0, pos.Y)
            end
            locking = false
        end)

        -- Search
        local function trim(s)
            return (s:gsub("^%s*(.-)%s*$", "%1"))
        end

        local function applySearch()
            local q = trim(searchBox.Text or "")
            q = string.lower(q)

            if q == "" then
                for _, btn in ipairs(allButtons) do
                    btn.Visible = true
                end
            else
                for _, btn in ipairs(allButtons) do
                    local text = string.lower(btn.Text or "")
                    btn.Visible = string.find(text, q, 1, true) ~= nil
                end
            end

            listHolder.CanvasPosition = Vector2.new(0, 0)
        end

        searchBox:GetPropertyChangedSignal("Text"):Connect(applySearch)
        searchBox.Focused:Connect(function() sbStroke.Color = THEME.GREEN end)
        searchBox.FocusLost:Connect(function() sbStroke.Color = THEME.GREEN end)

        --------------------------------------------------------------------
        -- GLOBAL CLICK CLOSE (ทั้งหน้าจอจริง)
        -- กด/แตะตรงไหนก็ปิด ยกเว้น: panel / selectBtn / searchBox
        --------------------------------------------------------------------
        inputConn = UserInputService.InputBegan:Connect(function(input, gp)
            if not optionsPanel then return end

            local t = input.UserInputType
            if t ~= Enum.UserInputType.MouseButton1 and t ~= Enum.UserInputType.Touch then
                return
            end

            -- สำคัญ: ไม่เช็ค gp (เพราะคลิก UI อื่นๆ gp จะเป็น true แล้วมันไม่ปิด)
            local pos
            if t == Enum.UserInputType.Touch then
                pos = input.Position
            else
                pos = UserInputService:GetMouseLocation()
            end

            local keep =
                isInside(optionsPanel, pos)
                or isInside(selectBtn, pos)
                or (searchBox and isInside(searchBox, pos))

            if not keep then
                closePanel()
            end
        end)
    end

    -- Toggle Select Options
    selectBtn.MouseButton1Click:Connect(function()
        if opened then
            closePanel()
        else
            openPanel()
            opened = true
            updateSelectVisual(true)
        end
    end)

    -- INIT SYNC (AA1)
    task.defer(function()
        setEnableVisual(STATE.Enabled)
        ensureRunner()
    end)
end)
--===== UFO HUB X • SETTINGS — Smoother 🚀 (A V1 • fixed 3 rows) + Runner Save (per-map) + AA1 =====
registerRight("Settings", function(scroll)
    local TweenService = game:GetService("TweenService")
    local Lighting     = game:GetService("Lighting")
    local Players      = game:GetService("Players")
    local Http         = game:GetService("HttpService")
    local MPS          = game:GetService("MarketplaceService")
    local lp           = Players.LocalPlayer

    --=================== PER-MAP SAVE (file: UFO HUB X/<PlaceId - Name>.json; fallback RAM) ===================
    local function safePlaceName()
        local ok,info = pcall(function() return MPS:GetProductInfo(game.PlaceId) end)
        local n = (ok and info and info.Name) or ("Place_"..tostring(game.PlaceId))
        return n:gsub("[^%w%-%._ ]","_")
    end
    local SAVE_DIR  = "UFO HUB X"
    local SAVE_FILE = SAVE_DIR .. "/" .. tostring(game.PlaceId) .. " - " .. safePlaceName() .. ".json"
    local hasFS = (typeof(isfolder)=="function" and typeof(makefolder)=="function"
                and typeof(readfile)=="function" and typeof(writefile)=="function")
    if hasFS and not isfolder(SAVE_DIR) then pcall(makefolder, SAVE_DIR) end
    getgenv().UFOX_RAM = getgenv().UFOX_RAM or {}
    local RAM = getgenv().UFOX_RAM

    local function loadSave()
        if hasFS and pcall(function() return readfile(SAVE_FILE) end) then
            local ok, data = pcall(function() return Http:JSONDecode(readfile(SAVE_FILE)) end)
            if ok and type(data)=="table" then return data end
        end
        return RAM[SAVE_FILE] or {}
    end
    local function writeSave(t)
        t = t or {}
        if hasFS then pcall(function() writefile(SAVE_FILE, Http:JSONEncode(t)) end) end
        RAM[SAVE_FILE] = t
    end
    local function getSave(path, default)
        local cur = loadSave()
        for seg in string.gmatch(path, "[^%.]+") do cur = (type(cur)=="table") and cur[seg] or nil end
        return (cur==nil) and default or cur
    end
    local function setSave(path, value)
        local data, p, keys = loadSave(), nil, {}
        for seg in string.gmatch(path, "[^%.]+") do table.insert(keys, seg) end
        p = data
        for i=1,#keys-1 do local k=keys[i]; if type(p[k])~="table" then p[k] = {} end; p = p[k] end
        p[keys[#keys]] = value
        writeSave(data)
    end
    --==========================================================================================================

    -- THEME (A V1)
    local THEME = {
        GREEN = Color3.fromRGB(25,255,125),
        WHITE = Color3.fromRGB(255,255,255),
        BLACK = Color3.fromRGB(0,0,0),
        TEXT  = Color3.fromRGB(255,255,255),
        RED   = Color3.fromRGB(255,40,40),
    }
    local function corner(ui,r) local c=Instance.new("UICorner") c.CornerRadius=UDim.new(0,r or 12) c.Parent=ui end
    local function stroke(ui,th,col) local s=Instance.new("UIStroke") s.Thickness=th or 2.2 s.Color=col or THEME.GREEN s.ApplyStrokeMode=Enum.ApplyStrokeMode.Border s.Parent=ui end
    local function tween(o,p) TweenService:Create(o,TweenInfo.new(0.1,Enum.EasingStyle.Quad,Enum.EasingDirection.Out),p):Play() end

    -- Ensure ListLayout
    local list = scroll:FindFirstChildOfClass("UIListLayout") or Instance.new("UIListLayout", scroll)
    list.Padding = UDim.new(0,12); list.SortOrder = Enum.SortOrder.LayoutOrder
    scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y

    -- STATE
    _G.UFOX_SMOOTH = _G.UFOX_SMOOTH or { mode=0, plastic=false, _snap={}, _pp={} }
    local S = _G.UFOX_SMOOTH

    -- ===== restore from SAVE =====
    S.mode    = getSave("Settings.Smoother.Mode",    S.mode)      -- 0/1/2
    S.plastic = getSave("Settings.Smoother.Plastic", S.plastic)   -- boolean

    -- Header
    local head = scroll:FindFirstChild("A1_Header") or Instance.new("TextLabel", scroll)
    head.Name="A1_Header"; head.BackgroundTransparency=1; head.Size=UDim2.new(1,0,0,36)
    head.Font=Enum.Font.GothamBold; head.TextSize=16; head.TextColor3=THEME.TEXT
    head.TextXAlignment=Enum.TextXAlignment.Left; head.Text="》》》Smoothness Settings 🚀《《《"; head.LayoutOrder = 10

    -- Remove any old rows
    for _,n in ipairs({"A1_Reduce","A1_Remove","A1_Plastic"}) do local old=scroll:FindFirstChild(n); if old then old:Destroy() end end

    -- Row factory
    local function makeRow(name, label, order, onToggle)
        local row = Instance.new("Frame", scroll)
        row.Name=name; row.Size=UDim2.new(1,-6,0,46); row.BackgroundColor3=THEME.BLACK
        row.LayoutOrder=order; corner(row,12); stroke(row,2.2,THEME.GREEN)

        local lab=Instance.new("TextLabel", row)
        lab.BackgroundTransparency=1; lab.Size=UDim2.new(1,-160,1,0); lab.Position=UDim2.new(0,16,0,0)
        lab.Font=Enum.Font.GothamBold; lab.TextSize=13; lab.TextColor3=THEME.WHITE
        lab.TextXAlignment=Enum.TextXAlignment.Left; lab.Text=label

        local sw=Instance.new("Frame", row)
        sw.AnchorPoint=Vector2.new(1,0.5); sw.Position=UDim2.new(1,-12,0.5,0)
        sw.Size=UDim2.fromOffset(52,26); sw.BackgroundColor3=THEME.BLACK
        corner(sw,13)
        local swStroke=Instance.new("UIStroke", sw); swStroke.Thickness=1.8; swStroke.Color=THEME.RED

        local knob=Instance.new("Frame", sw)
        knob.Size=UDim2.fromOffset(22,22); knob.BackgroundColor3=THEME.WHITE
        knob.Position=UDim2.new(0,2,0.5,-11); corner(knob,11)

        local state=false
        local function setState(v)
            state=v
            swStroke.Color = v and THEME.GREEN or THEME.RED
            tween(knob, {Position=UDim2.new(v and 1 or 0, v and -24 or 2, 0.5, -11)})
            if onToggle then onToggle(v) end
        end
        local btn=Instance.new("TextButton", sw)
        btn.BackgroundTransparency=1; btn.Size=UDim2.fromScale(1,1); btn.Text=""
        btn.MouseButton1Click:Connect(function() setState(not state) end)

        return setState
    end

    -- ===== FX helpers (same as before) =====
    local FX = {ParticleEmitter=true, Trail=true, Beam=true, Smoke=true, Fire=true, Sparkles=true}
    local PP = {BloomEffect=true, ColorCorrectionEffect=true, DepthOfFieldEffect=true, SunRaysEffect=true, BlurEffect=true}

    local function capture(inst)
        if S._snap[inst] then return end
        local t={}; pcall(function()
            if inst:IsA("ParticleEmitter") then t.Rate=inst.Rate; t.Enabled=inst.Enabled
            elseif inst:IsA("Trail") then t.Enabled=inst.Enabled; t.Brightness=inst.Brightness
            elseif inst:IsA("Beam") then t.Enabled=inst.Enabled; t.Brightness=inst.Brightness
            elseif inst:IsA("Smoke") then t.Enabled=inst.Enabled; t.Opacity=inst.Opacity
            elseif inst:IsA("Fire") then t.Enabled=inst.Enabled; t.Heat=inst.Heat; t.Size=inst.Size
            elseif inst:IsA("Sparkles") then t.Enabled=inst.Enabled end
        end)
        S._snap[inst]=t
    end
    for _,d in ipairs(workspace:GetDescendants()) do if FX[d.ClassName] then capture(d) end end

    local function applyHalf()
        for i,t in pairs(S._snap) do if i.Parent then pcall(function()
            if i:IsA("ParticleEmitter") then i.Rate=(t.Rate or 10)*0.5
            elseif i:IsA("Trail") or i:IsA("Beam") then i.Brightness=(t.Brightness or 1)*0.5
            elseif i:IsA("Smoke") then i.Opacity=(t.Opacity or 1)*0.5
            elseif i:IsA("Fire") then i.Heat=(t.Heat or 5)*0.5; i.Size=(t.Size or 5)*0.7
            elseif i:IsA("Sparkles") then i.Enabled=false end
        end) end end
        for _,obj in ipairs(Lighting:GetChildren()) do
            if PP[obj.ClassName] then
                S._pp[obj]={Enabled=obj.Enabled, Intensity=obj.Intensity, Size=obj.Size}
                obj.Enabled=true; if obj.Intensity then obj.Intensity=(obj.Intensity or 1)*0.5 end
                if obj.ClassName=="BlurEffect" and obj.Size then obj.Size=math.floor((obj.Size or 0)*0.5) end
            end
        end
    end
    local function applyOff()
        for i,_ in pairs(S._snap) do if i.Parent then pcall(function() i.Enabled=false end) end end
        for _,obj in ipairs(Lighting:GetChildren()) do if PP[obj.ClassName] then obj.Enabled=false end end
    end
    local function restoreAll()
        for i,t in pairs(S._snap) do if i.Parent then for k,v in pairs(t) do pcall(function() i[k]=v end) end end end
        for obj,t in pairs(S._pp)   do if obj.Parent then for k,v in pairs(t) do pcall(function() obj[k]=v end) end end end
    end

    local function plasticMode(on)
        for _,p in ipairs(workspace:GetDescendants()) do
            if p:IsA("BasePart") and not p:IsDescendantOf(lp.Character) then
                if on then
                    if not p:GetAttribute("Mat0") then p:SetAttribute("Mat0",p.Material.Name); p:SetAttribute("Refl0",p.Reflectance) end
                    p.Material=Enum.Material.SmoothPlastic; p.Reflectance=0
                else
                    local m=p:GetAttribute("Mat0"); local r=p:GetAttribute("Refl0")
                    if m then pcall(function() p.Material=Enum.Material[m] end) p:SetAttribute("Mat0",nil) end
                    if r~=nil then p.Reflectance=r; p:SetAttribute("Refl0",nil) end
                end
            end
        end
    end

    -- ===== 3 switches (fixed orders 11/12/13) + SAVE =====
    local set50, set100, setPl

    set50  = makeRow("A1_Reduce", "Reduce Effects 50%", 11, function(v)
        if v then
            S.mode=1; applyHalf()
            if set100 then set100(false) end
        else
            if S.mode==1 then S.mode=0; restoreAll() end
        end
        setSave("Settings.Smoother.Mode", S.mode)
    end)

    set100 = makeRow("A1_Remove", "Remove Effects 100%", 12, function(v)
        if v then
            S.mode=2; applyOff()
            if set50 then set50(false) end
        else
            if S.mode==2 then S.mode=0; restoreAll() end
        end
        setSave("Settings.Smoother.Mode", S.mode)
    end)

    setPl   = makeRow("A1_Plastic","Change Map to Plastic)", 13, function(v)
        S.plastic=v; plasticMode(v)
        setSave("Settings.Smoother.Plastic", v)
    end)

    -- ===== Apply restored saved state to UI/World =====
    if S.mode==1 then
        set50(true)
    elseif S.mode==2 then
        set100(true)
    else
        set50(false); set100(false); restoreAll()
    end
    setPl(S.plastic)
end)

-- ########## AA1 — Auto-run Smoother from SaveState (ไม่ต้องกดปุ่ม UI) ##########
task.defer(function()
    local TweenService = game:GetService("TweenService")
    local Lighting     = game:GetService("Lighting")
    local Players      = game:GetService("Players")
    local Http         = game:GetService("HttpService")
    local MPS          = game:GetService("MarketplaceService")
    local lp           = Players.LocalPlayer

    -- ใช้ SAVE เดิมแบบเดียวกับด้านบน
    local function safePlaceName()
        local ok,info = pcall(function() return MPS:GetProductInfo(game.PlaceId) end)
        local n = (ok and info and info.Name) or ("Place_"..tostring(game.PlaceId))
        return n:gsub("[^%w%-%._ ]","_")
    end
    local SAVE_DIR  = "UFO HUB X"
    local SAVE_FILE = SAVE_DIR .. "/" .. tostring(game.PlaceId) .. " - " .. safePlaceName() .. ".json"
    local hasFS = (typeof(isfolder)=="function" and typeof(makefolder)=="function"
                and typeof(readfile)=="function" and typeof(writefile)=="function")
    if hasFS and not isfolder(SAVE_DIR) then pcall(makefolder, SAVE_DIR) end
    getgenv().UFOX_RAM = getgenv().UFOX_RAM or {}
    local RAM = getgenv().UFOX_RAM

    local function loadSave()
        if hasFS and pcall(function() return readfile(SAVE_FILE) end) then
            local ok, data = pcall(function() return Http:JSONDecode(readfile(SAVE_FILE)) end)
            if ok and type(data)=="table" then return data end
        end
        return RAM[SAVE_FILE] or {}
    end
    local function getSave(path, default)
        local cur = loadSave()
        for seg in string.gmatch(path, "[^%.]+") do cur = (type(cur)=="table") and cur[seg] or nil end
        return (cur==nil) and default or cur
    end

    -- ใช้ state เดียวกับ UI
    _G.UFOX_SMOOTH = _G.UFOX_SMOOTH or { mode=0, plastic=false, _snap={}, _pp={} }
    local S = _G.UFOX_SMOOTH

    local FX = {ParticleEmitter=true, Trail=true, Beam=true, Smoke=true, Fire=true, Sparkles=true}
    local PP = {BloomEffect=true, ColorCorrectionEffect=true, DepthOfFieldEffect=true, SunRaysEffect=true, BlurEffect=true}

    local function capture(inst)
        if S._snap[inst] then return end
        local t={}; pcall(function()
            if inst:IsA("ParticleEmitter") then t.Rate=inst.Rate; t.Enabled=inst.Enabled
            elseif inst:IsA("Trail") then t.Enabled=inst.Enabled; t.Brightness=inst.Brightness
            elseif inst:IsA("Beam") then t.Enabled=inst.Enabled; t.Brightness=inst.Brightness
            elseif inst:IsA("Smoke") then t.Enabled=inst.Enabled; t.Opacity=inst.Opacity
            elseif inst:IsA("Fire") then t.Enabled=inst.Enabled; t.Heat=inst.Heat; t.Size=inst.Size
            elseif inst:IsA("Sparkles") then t.Enabled=inst.Enabled end
        end)
        S._snap[inst]=t
    end
    for _,d in ipairs(workspace:GetDescendants()) do
        if FX[d.ClassName] then capture(d) end
    end

    local function applyHalf()
        for i,t in pairs(S._snap) do
            if i.Parent then pcall(function()
                if i:IsA("ParticleEmitter") then i.Rate=(t.Rate or 10)*0.5
                elseif i:IsA("Trail") or i:IsA("Beam") then i.Brightness=(t.Brightness or 1)*0.5
                elseif i:IsA("Smoke") then i.Opacity=(t.Opacity or 1)*0.5
                elseif i:IsA("Fire") then i.Heat=(t.Heat or 5)*0.5; i.Size=(t.Size or 5)*0.7
                elseif i:IsA("Sparkles") then i.Enabled=false end
            end) end
        end
        for _,obj in ipairs(Lighting:GetChildren()) do
            if PP[obj.ClassName] then
                S._pp[obj] = S._pp[obj] or {}
                local snap = S._pp[obj]
                if snap.Enabled == nil then
                    snap.Enabled = obj.Enabled
                    if obj.Intensity ~= nil then snap.Intensity = obj.Intensity end
                    if obj.ClassName=="BlurEffect" and obj.Size then snap.Size = obj.Size end
                end
                obj.Enabled = true
                if obj.Intensity and snap.Intensity ~= nil then
                    obj.Intensity = (snap.Intensity or obj.Intensity or 1)*0.5
                end
                if obj.ClassName=="BlurEffect" and obj.Size and snap.Size ~= nil then
                    obj.Size = math.floor((snap.Size or obj.Size or 0)*0.5)
                end
            end
        end
    end

    local function applyOff()
        for i,_ in pairs(S._snap) do
            if i.Parent then pcall(function() i.Enabled=false end) end
        end
        for _,obj in ipairs(Lighting:GetChildren()) do
            if PP[obj.ClassName] then obj.Enabled=false end
        end
    end

    local function restoreAll()
        for i,t in pairs(S._snap) do
            if i.Parent then
                for k,v in pairs(t) do pcall(function() i[k]=v end) end
            end
        end
        for obj,t in pairs(S._pp) do
            if obj.Parent then
                for k,v in pairs(t) do pcall(function() obj[k]=v end) end
            end
        end
    end

    local function plasticMode(on)
        for _,p in ipairs(workspace:GetDescendants()) do
            if p:IsA("BasePart") and not p:IsDescendantOf(lp.Character) then
                if on then
                    if not p:GetAttribute("Mat0") then
                        p:SetAttribute("Mat0", p.Material.Name)
                        p:SetAttribute("Refl0", p.Reflectance)
                    end
                    p.Material = Enum.Material.SmoothPlastic
                    p.Reflectance = 0
                else
                    local m = p:GetAttribute("Mat0")
                    local r = p:GetAttribute("Refl0")
                    if m then pcall(function() p.Material = Enum.Material[m] end); p:SetAttribute("Mat0", nil) end
                    if r ~= nil then p.Reflectance = r; p:SetAttribute("Refl0", nil) end
                end
            end
        end
    end

    -- อ่าน SaveState แล้ว apply อัตโนมัติ (AA1)
    local mode    = getSave("Settings.Smoother.Mode",    S.mode or 0)
    local plastic = getSave("Settings.Smoother.Plastic", S.plastic or false)
    S.mode    = mode
    S.plastic = plastic

    if mode == 1 then
        applyHalf()
    elseif mode == 2 then
        applyOff()
    else
        restoreAll()
    end
    plasticMode(plastic)
end)
-- ===== UFO HUB X • Settings — AFK 💤 (MODEL A LEGACY, full systems) + Runner Save + AA1 =====
-- 1) Black Screen (Performance AFK)  [toggle]
-- 2) White Screen (Performance AFK)  [toggle]
-- 3) AFK Anti-Kick (20 min)          [toggle default ON]
-- 4) Activity Watcher (5 min → enable #3) [toggle default ON]
-- + AA1: Auto-run จาก SaveState โดยตรง ไม่ต้องแตะ UI

-- ########## SERVICES ##########
local Players       = game:GetService("Players")
local TweenService  = game:GetService("TweenService")
local UIS           = game:GetService("UserInputService")
local RunService    = game:GetService("RunService")
local VirtualUser   = game:GetService("VirtualUser")
local Http          = game:GetService("HttpService")
local MPS           = game:GetService("MarketplaceService")
local lp            = Players.LocalPlayer

-- ########## PER-MAP SAVE (file + RAM fallback) ##########
local function safePlaceName()
    local ok,info = pcall(function() return MPS:GetProductInfo(game.PlaceId) end)
    local n = (ok and info and info.Name) or ("Place_"..tostring(game.PlaceId))
    return n:gsub("[^%w%-%._ ]","_")
end

local SAVE_DIR  = "UFO HUB X"
local SAVE_FILE = SAVE_DIR.."/"..tostring(game.PlaceId).." - "..safePlaceName()..".json"

local hasFS = (typeof(isfolder)=="function" and typeof(makefolder)=="function"
            and typeof(writefile)=="function" and typeof(readfile)=="function")

if hasFS and not isfolder(SAVE_DIR) then pcall(makefolder, SAVE_DIR) end

getgenv().UFOX_RAM = getgenv().UFOX_RAM or {}
local RAM = getgenv().UFOX_RAM

local function loadSave()
    if hasFS and pcall(function() return readfile(SAVE_FILE) end) then
        local ok,dec = pcall(function() return Http:JSONDecode(readfile(SAVE_FILE)) end)
        if ok and type(dec)=="table" then return dec end
    end
    return RAM[SAVE_FILE] or {}
end

local function writeSave(t)
    t = t or {}
    if hasFS then
        pcall(function()
            writefile(SAVE_FILE, Http:JSONEncode(t))
        end)
    end
    RAM[SAVE_FILE] = t
end

local function getSave(path, default)
    local data = loadSave()
    local cur  = data
    for seg in string.gmatch(path,"[^%.]+") do
        cur = (type(cur)=="table") and cur[seg] or nil
    end
    return (cur==nil) and default or cur
end

local function setSave(path, value)
    local data = loadSave()
    local keys = {}
    for seg in string.gmatch(path,"[^%.]+") do table.insert(keys, seg) end
    local p = data
    for i=1,#keys-1 do
        local k = keys[i]
        if type(p[k])~="table" then p[k] = {} end
        p = p[k]
    end
    p[keys[#keys]] = value
    writeSave(data)
end

-- ########## THEME / HELPERS ##########
local THEME = {
    GREEN = Color3.fromRGB(25,255,125),
    RED   = Color3.fromRGB(255,40,40),
    WHITE = Color3.fromRGB(255,255,255),
    BLACK = Color3.fromRGB(0,0,0),
    TEXT  = Color3.fromRGB(255,255,255),
}

local function corner(ui,r)
    local c = Instance.new("UICorner")
    c.CornerRadius = UDim.new(0,r or 12)
    c.Parent = ui
end

local function stroke(ui,th,col)
    local s = Instance.new("UIStroke")
    s.Thickness = th or 2.2
    s.Color = col or THEME.GREEN
    s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    s.Parent = ui
end

local function tween(o,p)
    TweenService:Create(o, TweenInfo.new(0.1, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), p):Play()
end

-- ########## GLOBAL AFK STATE ##########
_G.UFOX_AFK = _G.UFOX_AFK or {
    blackOn    = false,
    whiteOn    = false,
    antiIdleOn = true,   -- default ON
    watcherOn  = true,   -- default ON
    lastInput  = tick(),
    antiIdleLoop = nil,
    idleHooked   = false,
    gui          = nil,
    watcherConn  = nil,
    inputConns   = {},
}

local S = _G.UFOX_AFK

-- ===== restore from SAVE → override defaults =====
S.blackOn    = getSave("Settings.AFK.Black",    S.blackOn)
S.whiteOn    = getSave("Settings.AFK.White",    S.whiteOn)
S.antiIdleOn = getSave("Settings.AFK.AntiKick", S.antiIdleOn)
S.watcherOn  = getSave("Settings.AFK.Watcher",  S.watcherOn)

-- ########## CORE: OVERLAY (Black / White) ##########
local function ensureGui()
    if S.gui and S.gui.Parent then return S.gui end
    local gui = Instance.new("ScreenGui")
    gui.Name="UFOX_AFK_GUI"
    gui.IgnoreGuiInset = true
    gui.ResetOnSpawn   = false
    gui.DisplayOrder   = 999999
    gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    gui.Parent = lp:WaitForChild("PlayerGui")
    S.gui = gui
    return gui
end

local function clearOverlay(name)
    if S.gui then
        local f = S.gui:FindFirstChild(name)
        if f then f:Destroy() end
    end
end

local function showBlack(v)
    clearOverlay("WhiteOverlay")
    clearOverlay("BlackOverlay")
    if not v then return end
    local gui = ensureGui()
    local black = Instance.new("Frame", gui)
    black.Name = "BlackOverlay"
    black.BackgroundColor3 = Color3.new(0,0,0)
    black.Size = UDim2.fromScale(1,1)
    black.ZIndex = 200
    black.Active = true
end

local function showWhite(v)
    clearOverlay("BlackOverlay")
    clearOverlay("WhiteOverlay")
    if not v then return end
    local gui = ensureGui()
    local white = Instance.new("Frame", gui)
    white.Name = "WhiteOverlay"
    white.BackgroundColor3 = Color3.new(1,1,1)
    white.Size = UDim2.fromScale(1,1)
    white.ZIndex = 200
    white.Active = true
end

local function syncOverlays()
    if S.blackOn then
        S.whiteOn = false
        showWhite(false)
        showBlack(true)
    elseif S.whiteOn then
        S.blackOn = false
        showBlack(false)
        showWhite(true)
    else
        showBlack(false)
        showWhite(false)
    end
end

-- ########## CORE: Anti-Kick / Activity ##########
local function pulseOnce()
    local cam = workspace.CurrentCamera
    local cf  = cam and cam.CFrame or CFrame.new()
    pcall(function()
        VirtualUser:CaptureController()
        VirtualUser:ClickButton2(Vector2.new(0,0), cf)
    end)
end

local function startAntiIdle()
    if S.antiIdleLoop then return end
    S.antiIdleLoop = task.spawn(function()
        while S.antiIdleOn do
            pulseOnce()
            for i=1,540 do  -- ~9 นาที (ตรงกับค่าเดิม)
                if not S.antiIdleOn then break end
                task.wait(1)
            end
        end
        S.antiIdleLoop = nil
    end)
end

-- hook Roblox Idle แค่ครั้งเดียว (เหมือนเดิม แต่ global)
if not S.idleHooked then
    S.idleHooked = true
    lp.Idled:Connect(function()
        if S.antiIdleOn then
            pulseOnce()
        end
    end)
end

-- input watcher (mouse/keyboard/touch) → update lastInput
local function ensureInputHooks()
    if S.inputConns and #S.inputConns > 0 then return end
    local function markInput() S.lastInput = tick() end
    table.insert(S.inputConns, UIS.InputBegan:Connect(markInput))
    table.insert(S.inputConns, UIS.InputChanged:Connect(function(io)
        if io.UserInputType ~= Enum.UserInputType.MouseWheel then
            markInput()
        end
    end))
end

local INACTIVE = 5*60 -- 5 นาที
local function startWatcher()
    if S.watcherConn then return end
    S.watcherConn = RunService.Heartbeat:Connect(function()
        if not S.watcherOn then return end
        if tick() - S.lastInput >= INACTIVE then
            -- เปิด Anti-Kick อัตโนมัติ (เหมือนเดิม)
            S.antiIdleOn = true
            setSave("Settings.AFK.AntiKick", true)
            if not S.antiIdleLoop then startAntiIdle() end
            pulseOnce()
            S.lastInput = tick()
        end
    end)
end

-- ########## AA1: AUTO-RUN จาก SaveState (ไม่ต้องแตะ UI) ##########
task.defer(function()
    -- sync หน้าจอ AFK (black/white) ตามค่าที่เซฟไว้
    syncOverlays()

    -- ถ้า Anti-Kick ON → start loop ให้เลย
    if S.antiIdleOn then
        startAntiIdle()
    end

    -- watcher & input hooks (ดูการขยับทุก 5 นาทีเหมือนเดิม)
    ensureInputHooks()
    startWatcher()
end)

-- ########## UI ฝั่งขวา (MODEL A LEGACY • เหมือนเดิม) ##########
registerRight("Settings", function(scroll)
    -- ลบ section เก่า (ถ้ามี)
    local old = scroll:FindFirstChild("Section_AFK_Preview"); if old then old:Destroy() end
    local old2 = scroll:FindFirstChild("Section_AFK_Full");  if old2 then old2:Destroy() end

    -- layout เดิม
    local vlist = scroll:FindFirstChildOfClass("UIListLayout") or Instance.new("UIListLayout", scroll)
    vlist.Padding = UDim.new(0,12)
    vlist.SortOrder = Enum.SortOrder.LayoutOrder
    scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y

    local nextOrder = 10
    for _,ch in ipairs(scroll:GetChildren()) do
        if ch:IsA("GuiObject") and ch ~= vlist then
            nextOrder = math.max(nextOrder, (ch.LayoutOrder or 0)+1)
        end
    end

    -- Header
    local header = Instance.new("TextLabel", scroll)
    header.Name = "Section_AFK_Full"
    header.BackgroundTransparency = 1
    header.Size = UDim2.new(1,0,0,36)
    header.Font = Enum.Font.GothamBold
    header.TextSize = 16
    header.TextColor3 = THEME.TEXT
    header.TextXAlignment = Enum.TextXAlignment.Left
    header.Text = "AFK 💤"
    header.LayoutOrder = nextOrder

    -- Row helper (เหมือนโค้ดเดิม)
    local function makeRow(textLabel, defaultOn, onToggle)
        local row = Instance.new("Frame", scroll)
        row.Size = UDim2.new(1,-6,0,46)
        row.BackgroundColor3 = THEME.BLACK
        corner(row,12)
        stroke(row,2.2,THEME.GREEN)
        row.LayoutOrder = header.LayoutOrder + 1

        local lab = Instance.new("TextLabel", row)
        lab.BackgroundTransparency = 1
        lab.Size = UDim2.new(1,-160,1,0)
        lab.Position = UDim2.new(0,16,0,0)
        lab.Font = Enum.Font.GothamBold
        lab.TextSize = 13
        lab.TextColor3 = THEME.WHITE
        lab.TextXAlignment = Enum.TextXAlignment.Left
        lab.Text = textLabel

        local sw = Instance.new("Frame", row)
        sw.AnchorPoint = Vector2.new(1,0.5)
        sw.Position = UDim2.new(1,-12,0.5,0)
        sw.Size = UDim2.fromOffset(52,26)
        sw.BackgroundColor3 = THEME.BLACK
        corner(sw,13)

        local swStroke = Instance.new("UIStroke", sw)
        swStroke.Thickness = 1.8
        swStroke.Color = defaultOn and THEME.GREEN or THEME.RED

        local knob = Instance.new("Frame", sw)
        knob.Size = UDim2.fromOffset(22,22)
        knob.Position = UDim2.new(defaultOn and 1 or 0, defaultOn and -24 or 2, 0.5, -11)
        knob.BackgroundColor3 = THEME.WHITE
        corner(knob,11)

        local state = defaultOn
        local function setState(v)
            state = v
            swStroke.Color = v and THEME.GREEN or THEME.RED
            tween(knob, {Position = UDim2.new(v and 1 or 0, v and -24 or 2, 0.5, -11)})
            if onToggle then onToggle(v) end
        end

        local btn = Instance.new("TextButton", sw)
        btn.BackgroundTransparency = 1
        btn.Size = UDim2.fromScale(1,1)
        btn.Text = ""
        btn.AutoButtonColor = false
        btn.MouseButton1Click:Connect(function()
            setState(not state)
        end)

        return setState
    end

    -- ===== Rows + bindings (ใช้ STATE เดิม + SAVE + CORE) =====
    local setBlack = makeRow("Black Screen (Performance AFK)", S.blackOn, function(v)
        S.blackOn = v
        if v then S.whiteOn = false end
        syncOverlays()
        setSave("Settings.AFK.Black", v)
        if v == true then
            setSave("Settings.AFK.White", false)
        end
    end)

    local setWhite = makeRow("White Screen (Performance AFK)", S.whiteOn, function(v)
        S.whiteOn = v
        if v then S.blackOn = false end
        syncOverlays()
        setSave("Settings.AFK.White", v)
        if v == true then
            setSave("Settings.AFK.Black", false)
        end
    end)

    local setAnti  = makeRow("AFK Anti-Kick (20 min)", S.antiIdleOn, function(v)
        S.antiIdleOn = v
        setSave("Settings.AFK.AntiKick", v)
        if v then
            startAntiIdle()
        end
    end)

    local setWatch = makeRow("Activity Watcher (5 min → enable #3)", S.watcherOn, function(v)
        S.watcherOn = v
        setSave("Settings.AFK.Watcher", v)
        -- watcher loop จะเช็ค S.watcherOn อยู่แล้ว
    end)

    -- ===== Init เมื่อเปิดแท็บ Settings (ให้ตรงกับสถานะจริง) =====
    syncOverlays()
    if S.antiIdleOn then
        startAntiIdle()
    end
    ensureInputHooks()
    startWatcher()
end)
---- ========== ผูกปุ่มแท็บ + เปิดแท็บแรก ==========
local tabs = {
    {btn = btnHome,     set = setHomeActive,     name = "Home",     icon = ICON_HOME},
    {btn = btnQuest,    set = setQuestActive,    name = "Quest",    icon = ICON_QUEST},
    {btn = btnShop,     set = setShopActive,     name = "Shop",     icon = ICON_SHOP},
    {btn = btnSettings, set = setSettingsActive, name = "Settings", icon = ICON_SETTINGS},
}

local function activateTab(t)
    -- จดตำแหน่งสกอร์ลซ้ายไว้ก่อน (กันเด้ง)
    lastLeftY = LeftScroll.CanvasPosition.Y
    for _,x in ipairs(tabs) do x.set(x == t) end
    showRight(t.name, t.icon)
    task.defer(function()
        refreshLeftCanvas()
        local viewH = LeftScroll.AbsoluteSize.Y
        local maxY  = math.max(0, LeftScroll.CanvasSize.Y.Offset - viewH)
        LeftScroll.CanvasPosition = Vector2.new(0, math.clamp(lastLeftY,0,maxY))
        -- ถ้าปุ่มอยู่นอกเฟรม ค่อยเลื่อนให้อยู่พอดี
        local btn = t.btn
        if btn and btn.Parent then
            local top = btn.AbsolutePosition.Y - LeftScroll.AbsolutePosition.Y
            local bot = top + btn.AbsoluteSize.Y
            local pad = 8
            if top < 0 then
                LeftScroll.CanvasPosition = LeftScroll.CanvasPosition + Vector2.new(0, top - pad)
            elseif bot > viewH then
                LeftScroll.CanvasPosition = LeftScroll.CanvasPosition + Vector2.new(0, (bot - viewH) + pad)
            end
            lastLeftY = LeftScroll.CanvasPosition.Y
        end
    end)
end

for _,t in ipairs(tabs) do
    t.btn.MouseButton1Click:Connect(function() activateTab(t) end)
end

-- เปิดด้วยแท็บแรก
activateTab(tabs[1])

-- ===== Start visible & sync toggle to this UI =====
setOpen(true)

-- ===== Rebind close buttons inside this UI (กันกรณีชื่อ X หลายตัว) =====
for _,o in ipairs(GUI:GetDescendants()) do
    if o:IsA("TextButton") and (o.Text or ""):upper()=="X" then
        o.MouseButton1Click:Connect(function() setOpen(false) end)
    end
end

-- ===== Auto-rebind ถ้า UI หลักถูกสร้างใหม่ภายหลัง =====
local function hookContainer(container)
    if not container then return end
    container.ChildAdded:Connect(function(child)
        if child.Name=="UFO_HUB_X_UI" then
            task.wait() -- ให้ลูกพร้อม
            for _,o in ipairs(child:GetDescendants()) do
                if o:IsA("TextButton") and (o.Text or ""):upper()=="X" then
                    o.MouseButton1Click:Connect(function() setOpen(false) end)
                end
            end
        end
    end)
end
hookContainer(CoreGui)
local pg = Players.LocalPlayer and Players.LocalPlayer:FindFirstChild("PlayerGui")
hookContainer(pg)

end -- <<== จบ _G.UFO_ShowMainUI() (โค้ด UI หลักของคุณแบบ 100%)

------------------------------------------------------------
-- 2) Toast chain (2-step) • โผล่ Step2 พร้อมกับ UI หลัก แล้วเลือนหาย
------------------------------------------------------------
do
    -- ล้าง Toast เก่า (ถ้ามี)
    pcall(function()
        local pg = game:GetService("Players").LocalPlayer:WaitForChild("PlayerGui")
        for _,n in ipairs({"UFO_Toast_Test","UFO_Toast_Test_2"}) do
            local g = pg:FindFirstChild(n); if g then g:Destroy() end
        end
    end)

    -- CONFIG
    local EDGE_RIGHT_PAD, EDGE_BOTTOM_PAD = 2, 2
    local TOAST_W, TOAST_H = 320, 86
    local RADIUS, STROKE_TH = 10, 2
    local GREEN = Color3.fromRGB(0,255,140)
    local BLACK = Color3.fromRGB(10,10,10)
    local LOGO_STEP1 = "rbxassetid://89004973470552"
    local LOGO_STEP2 = "rbxassetid://83753985156201"
    local TITLE_TOP, MSG_TOP = 12, 34
    local BAR_LEFT, BAR_RIGHT_PAD, BAR_H = 68, 12, 10
    local LOAD_TIME = 2.0

    local TS = game:GetService("TweenService")
    local RunS = game:GetService("RunService")
    local PG = game:GetService("Players").LocalPlayer:WaitForChild("PlayerGui")

    local function tween(inst, ti, ease, dir, props)
        return TS:Create(inst, TweenInfo.new(ti, ease or Enum.EasingStyle.Quad, dir or Enum.EasingDirection.Out), props)
    end
    local function makeToastGui(name)
        local gui = Instance.new("ScreenGui")
        gui.Name = name
        gui.ResetOnSpawn = false
        gui.IgnoreGuiInset = true
        gui.DisplayOrder = 999999
        gui.Parent = PG
        return gui
    end
    local function buildBox(parent)
        local box = Instance.new("Frame")
        box.Name = "Toast"
        box.AnchorPoint = Vector2.new(1,1)
        box.Position = UDim2.new(1, -EDGE_RIGHT_PAD, 1, -(EDGE_BOTTOM_PAD - 24))
        box.Size = UDim2.fromOffset(TOAST_W, TOAST_H)
        box.BackgroundColor3 = BLACK
        box.BorderSizePixel = 0
        box.Parent = parent
        Instance.new("UICorner", box).CornerRadius = UDim.new(0, RADIUS)
        local stroke = Instance.new("UIStroke", box)
        stroke.Thickness = STROKE_TH
        stroke.Color = GREEN
        stroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
        stroke.LineJoinMode = Enum.LineJoinMode.Round
        return box
    end
    local function buildTitle(box)
        local title = Instance.new("TextLabel")
        title.BackgroundTransparency = 1
        title.Font = Enum.Font.GothamBold
        title.RichText = true
        title.Text = '<font color="#FFFFFF">UFO</font> <font color="#00FF8C">HUB X</font>'
        title.TextSize = 18
        title.TextColor3 = Color3.fromRGB(235,235,235)
        title.TextXAlignment = Enum.TextXAlignment.Left
        title.Position = UDim2.fromOffset(68, TITLE_TOP)
        title.Size = UDim2.fromOffset(TOAST_W - 78, 20)
        title.Parent = box
        return title
    end
    local function buildMsg(box, text)
        local msg = Instance.new("TextLabel")
        msg.BackgroundTransparency = 1
        msg.Font = Enum.Font.Gotham
        msg.Text = text
        msg.TextSize = 13
        msg.TextColor3 = Color3.fromRGB(200,200,200)
        msg.TextXAlignment = Enum.TextXAlignment.Left
        msg.Position = UDim2.fromOffset(68, MSG_TOP)
        msg.Size = UDim2.fromOffset(TOAST_W - 78, 18)
        msg.Parent = box
        return msg
    end
    local function buildLogo(box, imageId)
        local logo = Instance.new("ImageLabel")
        logo.BackgroundTransparency = 1
        logo.Image = imageId
        logo.Size = UDim2.fromOffset(54, 54)
        logo.AnchorPoint = Vector2.new(0, 0.5)
        logo.Position = UDim2.new(0, 8, 0.5, -2)
        logo.Parent = box
        return logo
    end

    -- Step 1 (progress)
    local gui1 = makeToastGui("UFO_Toast_Test")
    local box1 = buildBox(gui1)
    buildLogo(box1, LOGO_STEP1)
    buildTitle(box1)
    local msg1 = buildMsg(box1, "Initializing... please wait")

    local barWidth = TOAST_W - BAR_LEFT - BAR_RIGHT_PAD
    local track = Instance.new("Frame"); track.BackgroundColor3 = Color3.fromRGB(25,25,25); track.BorderSizePixel = 0
    track.Position = UDim2.fromOffset(BAR_LEFT, TOAST_H - (BAR_H + 12))
    track.Size = UDim2.fromOffset(barWidth, BAR_H); track.Parent = box1
    Instance.new("UICorner", track).CornerRadius = UDim.new(0, BAR_H // 2)

    local fill = Instance.new("Frame"); fill.BackgroundColor3 = GREEN; fill.BorderSizePixel = 0
    fill.Size = UDim2.fromOffset(0, BAR_H); fill.Parent = track
    Instance.new("UICorner", fill).CornerRadius = UDim.new(0, BAR_H // 2)

    local pct = Instance.new("TextLabel")
    pct.BackgroundTransparency = 1; pct.Font = Enum.Font.GothamBold; pct.TextSize = 12
    pct.TextColor3 = Color3.new(1,1,1); pct.TextStrokeTransparency = 0.15; pct.TextStrokeColor3 = Color3.new(0,0,0)
    pct.TextXAlignment = Enum.TextXAlignment.Center; pct.TextYAlignment = Enum.TextYAlignment.Center
    pct.AnchorPoint = Vector2.new(0.5,0.5); pct.Position = UDim2.fromScale(0.5,0.5); pct.Size = UDim2.fromScale(1,1)
    pct.Text = "0%"; pct.ZIndex = 20; pct.Parent = track

    tween(box1, 0.22, Enum.EasingStyle.Quart, Enum.EasingDirection.Out,
        {Position = UDim2.new(1, -EDGE_RIGHT_PAD, 1, -EDGE_BOTTOM_PAD)}):Play()

    task.spawn(function()
        local t0 = time()
        local progress = 0
        while progress < 100 do
            progress = math.clamp(math.floor(((time() - t0)/LOAD_TIME)*100 + 0.5), 0, 100)
            fill.Size = UDim2.fromOffset(math.floor(barWidth*(progress/100)), BAR_H)
            pct.Text = progress .. "%"
            RunS.Heartbeat:Wait()
        end
        msg1.Text = "Loaded successfully."
        task.wait(0.25)
        local out1 = tween(box1, 0.32, Enum.EasingStyle.Quint, Enum.EasingDirection.InOut,
            {Position = UDim2.new(1, -EDGE_RIGHT_PAD, 1, -(EDGE_BOTTOM_PAD - 24))})
        out1:Play(); out1.Completed:Wait(); gui1:Destroy()

        -- Step 2 (no progress) + เปิด UI หลักพร้อมกัน
        local gui2 = makeToastGui("UFO_Toast_Test_2")
        local box2 = buildBox(gui2)
        buildLogo(box2, LOGO_STEP2)
        buildTitle(box2)
        buildMsg(box2, "Download UI completed. ✅")
        tween(box2, 0.22, Enum.EasingStyle.Quart, Enum.EasingDirection.Out,
            {Position = UDim2.new(1, -EDGE_RIGHT_PAD, 1, -EDGE_BOTTOM_PAD)}):Play()

        -- เปิด UI หลัก "พร้อมกัน" กับ Toast ขั้นที่ 2
        if _G.UFO_ShowMainUI then pcall(_G.UFO_ShowMainUI) end

        -- ให้ผู้ใช้เห็นข้อความครบ แล้วค่อยเลือนลง (ปรับเวลาได้ตามใจ)
        task.wait(1.2)
        local out2 = tween(box2, 0.34, Enum.EasingStyle.Quint, Enum.EasingDirection.InOut,
            {Position = UDim2.new(1, -EDGE_RIGHT_PAD, 1, -(EDGE_BOTTOM_PAD - 24))})
        out2:Play(); out2.Completed:Wait(); gui2:Destroy()
    end)
end
-- ==== mark boot done (lock forever until reset) ====
do
    local B = getgenv().UFO_BOOT or {}
    B.status = "done"
    getgenv().UFO_BOOT = B
end

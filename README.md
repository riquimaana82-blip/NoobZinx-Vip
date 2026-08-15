-- @NoobZinx - PARTE 1/5 (INTERFACE + LICENÇA ISOLADA)
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Camera = workspace.CurrentCamera
local LocalPlayer = Players.LocalPlayer
local UserInput = game:GetService("UserInputService")
local HttpService = game:GetService("HttpService")
local Stats = game:GetService("Stats")

-- ═══════════════════════════════════════════════════════
-- 🔐 CONFIGURAÇÕES
-- ═══════════════════════════════════════════════════════
local Config = {
    Aimbot = false, MostrarFOV = false, ESP = false,
    ESP_Caixa = false, ESP_Nome = false, ESP_Vida = false,
    ESP_Linha = false, ESP_Distancia = false, Speed = false,
    Rainbow = false, SuperJump = false, AtravessarParede = false,
    HitboxAmpliada = false, FPSBooster = false, KillAll = false,
    MostrarPing = false, MostrarPlayers = false, MostrarFPS = false,
    Fly = false,
    TargetPart = "Head", FOV = 150, Smooth = 0.2,
    SpeedMul = 50, AimType = "Ao Olhar"
}

-- ═══════════════════════════════════════════════════════
-- 🔐 SISTEMA DE KEY PERSISTENTE (ISOLADO)
-- ═══════════════════════════════════════════════════════
local USER_ID = LocalPlayer.UserId
local ARQUIVO_LICENCA = "NoobZinx_licenca.txt"

-- Banco de dados global
if not _G.KeysDatabase then
    _G.KeysDatabase = {}
end
if not _G.Licencas then
    _G.Licencas = {}
end
if not _G.KeysUtilizadas then
    _G.KeysUtilizadas = {}
end

-- ═══════════════════════════════════════════════════════
-- 🛡️ PROTEÇÃO ANTI-DUPLICAÇÃO
-- ═══════════════════════════════════════════════════════
if _G.NoobZinxCarregado then
    print("⚠️ @NoobZinx já está carregado! Ignorando execução duplicada.")
    return
end
_G.NoobZinxCarregado = true

-- ═══════════════════════════════════════════════════════
-- 🔐 SISTEMA DE VALIDAÇÃO DE KEYS (ALGORITMO COMPARTILHADO)
-- ═══════════════════════════════════════════════════════
local function calcularHash(key)
    local hash = 0
    for i = 1, #key do
        hash = (hash * 31 + string.byte(key, i)) % 1000000007
    end
    return hash
end

local function extrairInfoKey(key)
    local partes = {}
    for parte in string.gmatch(key, "[^-]+") do
        table.insert(partes, parte)
    end
    
    if #partes ~= 3 then return nil end
    if partes[1] ~= "NOOBZINX" then return nil end
    
    local tipo = partes[2]
    local codigo = partes[3]
    
    if #codigo ~= 8 then return nil end
    if not string.match(codigo, "^[A-Z0-9]+$") then return nil end
    
    local tipoKey = nil
    local duracao = nil
    
    if tipo == "VIP" then
        tipoKey = "VIP Permanente"
        duracao = -1
    else
        local dias = string.match(tipo, "^(%d+)D$")
        if not dias then return nil end
        dias = tonumber(dias)
        if dias < 1 or dias > 7 then return nil end
        tipoKey = dias .. " Dias"
        duracao = dias * 86400
    end
    
    local hashEsperado = calcularHash("NOOBZINX-" .. tipo .. "-" .. codigo)
    local digitoVerificador = hashEsperado % 10
    
    if tipo == "VIP" then
        if digitoVerificador % 2 ~= 0 then return nil end
    else
        if digitoVerificador % 2 == 0 then return nil end
    end
    
    return {
        tipo = tipoKey,
        duracao = duracao,
        codigo = codigo,
        hash = hashEsperado
    }
end

local function validarKeySistema(key)
    if not key or key == "" then return false, nil end
    
    local infoKey = extrairInfoKey(key)
    if not infoKey then return false, nil end
    
    if _G.KeysUtilizadas and _G.KeysUtilizadas[key] then
        return false, "utilizada"
    end
    
    return true, infoKey
end

-- ⬅️ FUNÇÕES DE ARQUIVO (COM TRATAMENTO DE ERRO)
local function salvarLicenca(licenca)
    if not licenca then return false end
    local ok, dados = pcall(function() return HttpService:JSONEncode(licenca) end)
    if not ok or not dados then return false end
    local ok2 = pcall(function() writefile(ARQUIVO_LICENCA, dados) end)
    return ok2
end

local function carregarLicenca()
    local ok1, existe = pcall(function() return isfile(ARQUIVO_LICENCA) end)
    if not ok1 or not existe then return nil end
    local ok2, dados = pcall(function() return readfile(ARQUIVO_LICENCA) end)
    if not ok2 or not dados or dados == "" then return nil end
    local ok3, licenca = pcall(function() return HttpService:JSONDecode(dados) end)
    if not ok3 or not licenca then return nil end
    return licenca
end

local function verificarLicenca()
    local licenca = _G.Licencas[USER_ID]
    if not licenca then
        licenca = carregarLicenca()
        if licenca and licenca.userId == USER_ID then
            _G.Licencas[USER_ID] = licenca
        end
    end
    if not licenca then return false, nil end
    if licenca.tipo == "VIP Permanente" then return true, licenca end
    if licenca.dataExpiracao and licenca.dataExpiracao > os.time() then
        return true, licenca
    end
    _G.Licencas[USER_ID] = nil
    pcall(function() writefile(ARQUIVO_LICENCA, "") end)
    return false, nil
end

local function ativarKey(key, keyData)
    if not keyData then return false end
    
    local dataExpiracao = nil
    if keyData.tipo ~= "VIP Permanente" then
        dataExpiracao = os.time() + keyData.duracao
    end
    
    local licenca = {
        userId = USER_ID,
        key = key,
        tipo = keyData.tipo,
        dataAtivacao = os.time(),
        dataExpiracao = dataExpiracao,
        ativa = true
    }
    _G.Licencas[USER_ID] = licenca
    salvarLicenca(licenca)
    return true
end

-- ═══════════════════════════════════════════════════════
-- 🌈 EFEITO RAINBOW (MAIS LENTO - 2X MAIS SUAVE)
-- ═══════════════════════════════════════════════════════
local tempoRainbow = 0

local function atualizarCorRainbow()
    tempoRainbow = tempoRainbow + 0.005
    return Color3.fromHSV(tempoRainbow % 1, 1, 1)
end

-- ═══════════════════════════════════════════════════════
-- 🚀 VARIÁVEIS GLOBAIS
-- ═══════════════════════════════════════════════════════
local ativarHitbox, desativarHitbox, ativarFPSBooster, desativarFPSBooster, puxarPlayers, lblFPS
local ativarESPCompleto, desativarESPCompleto
local ativarFly, desativarFly
local menuCriado = false
local SG, MF, T
local TelaKey = nil
local InputKey = nil
local MensagemErro = nil

-- Variáveis para novos recursos
local notificacoes = {}
local mostradorPing = nil
local mostradorPlayers = nil
local mostradorFPS = nil

-- ═══════════════════════════════════════════════════════
-- 🔔 FUNÇÃO PARA MOSTRAR NOTIFICAÇÃO
-- ═══════════════════════════════════════════════════════
local function mostrarNotificacao(texto)
    if not SG then return end
    
    local notif = Instance.new("Frame")
    notif.Parent = SG
    notif.BackgroundColor3 = Color3.fromRGB(20,20,20)
    notif.BorderColor3 = atualizarCorRainbow()
    notif.BorderSizePixel = 1
    notif.Size = UDim2.new(0, 200, 0, 35)
    notif.Position = UDim2.new(1, -210, 0, 10)
    notif.ZIndex = 100
    Instance.new("UICorner",notif).CornerRadius = UDim.new(0,8)
    
    local textoNotif = Instance.new("TextLabel")
    textoNotif.Parent = notif
    textoNotif.BackgroundTransparency = 1
    textoNotif.Size = UDim2.new(1,0,1,0)
    textoNotif.Text = texto
    textoNotif.TextColor3 = Color3.fromRGB(255,255,255)
    textoNotif.TextScaled = true
    textoNotif.Font = Enum.Font.GothamBold
    textoNotif.ZIndex = 101
    
    table.insert(notificacoes, notif)
    
    task.spawn(function()
        task.wait(3)
        pcall(function() notif:Destroy() end)
        for i, n in pairs(notificacoes) do
            if n == notif then
                table.remove(notificacoes, i)
                break
            end
        end
    end)
end

-- ═══════════════════════════════════════════════════════
-- 🔥 FUNÇÃO PARA ATUALIZAR O TÍTULO
-- ═══════════════════════════════════════════════════════
local function atualizarTitulo()
    if not T then return end
    local licenca = _G.Licencas[USER_ID]
    local textoNome = "@NoobZinx"
    if licenca then
        if licenca.tipo == "VIP Permanente" then
            textoNome = textoNome .. " • VIP PERMANENTE"
        elseif licenca.dataExpiracao then
            local tempo = math.max(licenca.dataExpiracao - os.time(), 0)
            local h = math.floor(tempo / 3600)
            local m = math.floor((tempo % 3600) / 60)
            local s = math.floor(tempo % 60)
            if h > 0 then
                textoNome = textoNome .. " • " .. string.format("%02dh %02dm", h, m)
            else
                textoNome = textoNome .. " • " .. string.format("%02dm %02ds", m, s)
            end
        end
    end
    T.Text = textoNome
end

-- ═══════════════════════════════════════════════════════
-- 🔥 FUNÇÃO PARA VERIFICAR KEY
-- ═══════════════════════════════════════════════════════
local function verificarKey()
    if not InputKey then return end
    local key = string.upper(string.gsub(InputKey.Text, "^%s*(.-)%s*$", "%1"))
    if key == "" then
        if MensagemErro then MensagemErro.Text = "❌ Digite uma Key!"; MensagemErro.TextColor3 = Color3.fromRGB(255,255,0) end
        return
    end
    
    local keyValida, keyData = validarKeySistema(key)
    
    if not keyValida then
        if keyData == "utilizada" then
            if MensagemErro then MensagemErro.Text = "❌ KEY JÁ UTILIZADA!"; MensagemErro.TextColor3 = Color3.fromRGB(255,0,0) end
        else
            if MensagemErro then MensagemErro.Text = "❌ KEY INVÁLIDA!"; MensagemErro.TextColor3 = Color3.fromRGB(255,0,0) end
        end
        InputKey.Text = ""
        return
    end
    
    if ativarKey(key, keyData) then
        _G.KeysUtilizadas[key] = {
            userId = USER_ID,
            dataUso = os.time()
        }
        
        if MensagemErro then MensagemErro.Text = "✅ KEY VÁLIDA! Carregando..."; MensagemErro.TextColor3 = Color3.fromRGB(0,255,0) end
        task.wait(0.5)
        if TelaKey then TelaKey:Destroy(); TelaKey = nil end
        IniciarMenu()
    else
        if MensagemErro then MensagemErro.Text = "❌ ERRO AO ATIVAR KEY!"; MensagemErro.TextColor3 = Color3.fromRGB(255,0,0) end
        InputKey.Text = ""
    end
end

-- ═══════════════════════════════════════════════════════
-- 🔥 FUNÇÃO PARA CRIAR TELA DE KEY (COM RAINBOW MAIS LENTO)
-- ═══════════════════════════════════════════════════════
local function criarTelaKey()
    if TelaKey then return end
    
    if LocalPlayer.PlayerGui:FindFirstChild("TelaKeyNoobZinx") then
        LocalPlayer.PlayerGui:FindFirstChild("TelaKeyNoobZinx"):Destroy()
    end
    
    TelaKey = Instance.new("ScreenGui")
    TelaKey.Name = "TelaKeyNoobZinx"
    TelaKey.Parent = LocalPlayer.PlayerGui
    TelaKey.ResetOnSpawn = false
    TelaKey.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

    local PainelKey = Instance.new("Frame")
    PainelKey.Parent = TelaKey
    PainelKey.BackgroundColor3 = atualizarCorRainbow()
    PainelKey.Size = UDim2.new(0,350,0,280)
    PainelKey.Position = UDim2.new(0.5,-175,0.5,-140)
    PainelKey.BorderSizePixel = 1
    PainelKey.BorderColor3 = Color3.fromRGB(255,255,255)
    Instance.new("UICorner",PainelKey).CornerRadius = UDim.new(0,12)

    local TituloKey = Instance.new("TextLabel")
    TituloKey.Parent = PainelKey
    TituloKey.BackgroundTransparency = 1
    TituloKey.Size = UDim2.new(1,0,0,50)
    TituloKey.Position = UDim2.new(0,0,0,15)
    TituloKey.Text = "🔐 VERIFICAÇÃO"
    TituloKey.TextColor3 = Color3.fromRGB(255,255,255)
    TituloKey.TextScaled = true
    TituloKey.Font = Enum.Font.GothamBold

    local SubTitulo = Instance.new("TextLabel")
    SubTitulo.Parent = PainelKey
    SubTitulo.BackgroundTransparency = 1
    SubTitulo.Size = UDim2.new(1,0,0,30)
    SubTitulo.Position = UDim2.new(0,0,0,65)
    SubTitulo.Text = "Digite a Key para acessar"
    SubTitulo.TextColor3 = Color3.fromRGB(255,255,255)
    SubTitulo.TextScaled = true
    SubTitulo.Font = Enum.Font.Gotham

    InputKey = Instance.new("TextBox")
    InputKey.Parent = PainelKey
    InputKey.BackgroundColor3 = Color3.fromRGB(40,40,40)
    InputKey.Size = UDim2.new(0.8,0,0,40)
    InputKey.Position = UDim2.new(0.1,0,0,110)
    InputKey.PlaceholderText = "Digite a Key aqui..."
    InputKey.PlaceholderColor3 = Color3.fromRGB(150,150,150)
    InputKey.Text = ""
    InputKey.TextColor3 = Color3.fromRGB(255,255,255)
    InputKey.TextScaled = true
    InputKey.Font = Enum.Font.Gotham
    Instance.new("UICorner",InputKey).CornerRadius = UDim.new(0,8)

    local BotaoVerificar = Instance.new("TextButton")
    BotaoVerificar.Parent = PainelKey
    BotaoVerificar.BackgroundColor3 = Color3.fromRGB(255,255,255)
    BotaoVerificar.Size = UDim2.new(0,140,0,40)
    BotaoVerificar.Position = UDim2.new(0.5,-70,0,170)
    BotaoVerificar.Text = "VERIFICAR"
    BotaoVerificar.TextColor3 = Color3.fromRGB(30,30,30)
    BotaoVerificar.TextScaled = true
    BotaoVerificar.Font = Enum.Font.GothamBold
    Instance.new("UICorner",BotaoVerificar).CornerRadius = UDim.new(0,8)
    BotaoVerificar.MouseButton1Click:Connect(verificarKey)

    MensagemErro = Instance.new("TextLabel")
    MensagemErro.Parent = PainelKey
    MensagemErro.BackgroundTransparency = 1
    MensagemErro.Size = UDim2.new(1,0,0,25)
    MensagemErro.Position = UDim2.new(0,0,0,220)
    MensagemErro.Text = ""
    MensagemErro.TextColor3 = Color3.fromRGB(255,255,0)
    MensagemErro.TextScaled = true
    MensagemErro.Font = Enum.Font.GothamBold

    local BotaoDiscord = Instance.new("TextButton")
    BotaoDiscord.Parent = PainelKey
    BotaoDiscord.BackgroundColor3 = Color3.fromRGB(88,101,242)
    BotaoDiscord.Size = UDim2.new(0,140,0,35)
    BotaoDiscord.Position = UDim2.new(0.5,-70,0,250)
    BotaoDiscord.Text = "💬 Discord"
    BotaoDiscord.TextColor3 = Color3.fromRGB(255,255,255)
    BotaoDiscord.TextScaled = true
    BotaoDiscord.Font = Enum.Font.GothamBold
    Instance.new("UICorner",BotaoDiscord).CornerRadius = UDim.new(0,8)
    BotaoDiscord.MouseButton1Click:Connect(function()
        setclipboard("https://discord.gg/VNjfq35gPQ")
        MensagemErro.Text = "📋 Link copiado!"
        MensagemErro.TextColor3 = Color3.fromRGB(0,255,0)
        task.wait(2)
        MensagemErro.Text = ""
    end)

    InputKey.FocusLost:Connect(function(enter)
        if enter then verificarKey() end
    end)
    
    -- Loop para atualizar a cor rainbow da tela de key (mais lento)
    coroutine.wrap(function()
        while TelaKey and PainelKey and PainelKey.Parent do
            task.wait(0.1)
            PainelKey.BackgroundColor3 = atualizarCorRainbow()
        end
    end)()
end

-- ═══════════════════════════════════════════════════════
-- 🚀 INICIALIZAÇÃO PRINCIPAL (SIMPLES E SEGURA)
-- ═══════════════════════════════════════════════════════
print("🚀 Inicializando @NoobZinx...")

local painelExistente = nil
for _, obj in pairs(LocalPlayer.PlayerGui:GetChildren()) do
    if obj:IsA("ScreenGui") and obj.Name == "NoobZinxPainel" then
        painelExistente = obj
        break
    end
end

if painelExistente then
    print("✅ Painel @NoobZinx já existe! Não criando outro.")
    return
end

local temLicenca = false
local ok, err = pcall(function()
    temLicenca = verificarLicenca()
end)
if not ok then
    print("⚠️ Erro ao verificar licença:", err)
    temLicenca = false
end

if temLicenca then
    print("✅ Licença ativa para usuário " .. USER_ID)
    task.spawn(function()
        task.wait(1)
        IniciarMenu()
    end)
else
    print("🔐 Nenhuma licença ativa. Mostrando tela de Key.")
    criarTelaKey()
end

coroutine.wrap(function()
    while true do
        task.wait(1)
        pcall(atualizarTitulo)
        local licenca = _G.Licencas[USER_ID]
        if licenca and licenca.dataExpiracao then
            local tempo = licenca.dataExpiracao - os.time()
            if tempo <= 0 and licenca.tipo ~= "VIP Permanente" then
                _G.Licencas[USER_ID] = nil
                pcall(function() writefile(ARQUIVO_LICENCA, "") end)
                print("⏰ Licença expirada!")
                
                pcall(function()
                    if desativarHitbox then desativarHitbox() end
                    if desativarFPSBooster then desativarFPSBooster() end
                    if desativarESPCompleto then desativarESPCompleto() end
                    if desativarFly then desativarFly() end
                end)
                
                Config.Aimbot = false
                Config.MostrarFOV = false
                Config.ESP = false
                Config.ESP_Caixa = false
                Config.ESP_Nome = false
                Config.ESP_Vida = false
                Config.ESP_Linha = false
                Config.ESP_Distancia = false
                Config.Speed = false
                Config.Rainbow = false
                Config.SuperJump = false
                Config.AtravessarParede = false
                Config.HitboxAmpliada = false
                Config.FPSBooster = false
                Config.KillAll = false
                Config.MostrarPing = false
                Config.MostrarPlayers = false
                Config.MostrarFPS = false
                Config.Fly = false
                
                if SG then pcall(function() SG:Destroy() end); SG = nil end
                menuCriado = false
                _G.NoobZinxCarregado = false
                criarTelaKey()
            end
        end
    end
end)()
-- @NoobZinx - PARTE 2/5 (PAINEL + BOTÃO MENU CORRIGIDO)
function IniciarMenu()
    if menuCriado then 
        print("⚠️ Menu já criado! Ignorando...")
        return 
    end
    
    if LocalPlayer.PlayerGui:FindFirstChild("NoobZinxPainel") then
        print("⚠️ Painel já existe! Ignorando criação...")
        return
    end
    
    menuCriado = true

SG = Instance.new("ScreenGui")
SG.Name = "NoobZinxPainel"
SG.Parent = LocalPlayer.PlayerGui
SG.ResetOnSpawn = false
SG.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

local OB = Instance.new("TextButton")
OB.Parent = SG
OB.BackgroundColor3 = atualizarCorRainbow()
OB.Size = UDim2.new(0,45,0,45)
OB.Position = UDim2.new(0,10,0,10)
OB.Text = "MENU"
OB.TextColor3 = Color3.fromRGB(255,255,255)
OB.Font = Enum.Font.GothamBold
OB.TextScaled = true
OB.Visible = false
OB.ZIndex = 10
Instance.new("UICorner",OB).CornerRadius = UDim.new(0,10)

-- Sistema de arraste corrigido para o botão MENU
local arrastandoOB = false
local inicioArrasteOB = nil
local posicaoInicialOB = nil
local moveuOB = false

OB.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
        arrastandoOB = true
        moveuOB = false
        inicioArrasteOB = input.Position
        posicaoInicialOB = OB.Position
    end
end)

UserInput.InputChanged:Connect(function(input)
    if arrastandoOB and (input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseMovement) then
        local delta = input.Position - inicioArrasteOB
        if math.abs(delta.X) > 5 or math.abs(delta.Y) > 5 then
            moveuOB = true
        end
        OB.Position = UDim2.new(0, posicaoInicialOB.X.Offset + delta.X, 0, posicaoInicialOB.Y.Offset + delta.Y)
    end
end)

UserInput.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
        arrastandoOB = false
    end
end)

-- Clique normal para abrir/fechar o painel
OB.MouseButton1Click:Connect(function()
    if not moveuOB then
        MF.Visible = true
        OB.Visible = false
    end
end)

MF = Instance.new("Frame")
MF.Parent = SG
MF.BackgroundColor3 = Color3.fromRGB(15,15,15)
MF.Size = UDim2.new(0,340,0,420)
MF.Position = UDim2.new(0.5,-170,0.5,-210)
MF.Active = true
MF.Draggable = true
MF.BorderSizePixel = 1
MF.BorderColor3 = atualizarCorRainbow()
MF.ZIndex = 5
Instance.new("UICorner",MF).CornerRadius = UDim.new(0,10)

local TB = Instance.new("Frame")
TB.Parent = MF
TB.BackgroundColor3 = atualizarCorRainbow()
TB.Size = UDim2.new(1,0,0,36)
TB.ZIndex = 6
Instance.new("UICorner",TB).CornerRadius = UDim.new(0,10)

T = Instance.new("TextLabel")
T.Parent = TB
T.BackgroundTransparency = 1
T.Size = UDim2.new(0.8,0,1,0)
T.Position = UDim2.new(0.1,0,0,0)
atualizarTitulo()
T.TextColor3 = Color3.fromRGB(255,255,255)
T.TextScaled = true
T.Font = Enum.Font.GothamBold
T.ZIndex = 7

local X = Instance.new("TextButton")
X.Parent = TB
X.BackgroundColor3 = Color3.fromRGB(200,10,120)
X.Size = UDim2.new(0,36,1,0)
X.Position = UDim2.new(1,-36,0,0)
X.Text = "X"
X.TextColor3 = Color3.fromRGB(255,255,255)
X.TextScaled = true
X.Font = Enum.Font.GothamBold
X.ZIndex = 7
Instance.new("UICorner",X).CornerRadius = UDim.new(0,10)
X.MouseButton1Click:Connect(function() MF.Visible=false; OB.Visible=true end)

local Lateral = Instance.new("Frame")
Lateral.Parent = MF
Lateral.BackgroundColor3 = Color3.fromRGB(25,25,25)
Lateral.Size = UDim2.new(0,55,1,-36)
Lateral.Position = UDim2.new(0,0,0,36)
Lateral.ZIndex = 6

local Conteudo = Instance.new("Frame")
Conteudo.Parent = MF
Conteudo.BackgroundTransparency = 1
Conteudo.Size = UDim2.new(1,-55,1,-36)
Conteudo.Position = UDim2.new(0,55,0,36)
Conteudo.ZIndex = 6

local sep = function(pai, y)
    local s = Instance.new("Frame")
    s.Parent = pai
    s.BackgroundColor3 = atualizarCorRainbow()
    s.Size = UDim2.new(0.92,0,0,1)
    s.Position = UDim2.new(0.04,0,0,y)
    s.ZIndex = 6
end

local function abaLateral(icone, indice)
    local btn = Instance.new("TextButton")
    btn.Parent = Lateral
    btn.BackgroundColor3 = indice == 1 and atualizarCorRainbow() or Color3.fromRGB(35,35,35)
    btn.Size = UDim2.new(1,0,0,55)
    btn.Position = UDim2.new(0,0,0,(indice-1)*55)
    btn.Text = icone
    btn.TextColor3 = Color3.fromRGB(255,255,255)
    btn.Font = Enum.Font.GothamBold
    btn.TextScaled = true
    btn.ZIndex = 6
    Instance.new("UICorner",btn).CornerRadius = UDim.new(0,8)
    return btn
end

local bAim = abaLateral("🎯",1)
local bEsp = abaLateral("👁",2)
local bConf = abaLateral("⚙",3)
local bInfo = abaLateral("📄",4)

local function criarBotaoSwitch(pai, posY, texto, valorAtual, funcaoMudar)
    local lblTexto = Instance.new("TextLabel")
    lblTexto.Parent = pai
    lblTexto.BackgroundTransparency = 1
    lblTexto.Size = UDim2.new(0.55,0,0,28)
    lblTexto.Position = UDim2.new(0.05,0,0,posY)
    lblTexto.Text = texto
    lblTexto.TextColor3 = Color3.fromRGB(255,255,255)
    lblTexto.TextScaled = true
    lblTexto.Font = Enum.Font.Gotham
    lblTexto.ZIndex = 7

    local fundo = Instance.new("Frame")
    fundo.Parent = pai
    fundo.Size = UDim2.new(0,48,0,24)
    fundo.Position = UDim2.new(0.75,0,0,posY+2)
    fundo.BackgroundColor3 = Color3.fromRGB(60,60,60)
    fundo.ZIndex = 7
    Instance.new("UICorner",fundo).CornerRadius = UDim.new(0,12)

    local botao = Instance.new("Frame")
    botao.Parent = fundo
    botao.Size = UDim2.new(0,18,0,18)
    botao.Position = UDim2.new(0,3,0.5,-9)
    botao.BackgroundColor3 = Color3.fromRGB(255,255,255)
    botao.ZIndex = 8
    Instance.new("UICorner",botao).CornerRadius = UDim.new(1,0)

    local clique = Instance.new("TextButton")
    clique.Parent = fundo
    clique.Size = UDim2.new(1,0,1,0)
    clique.BackgroundTransparency = 1
    clique.Text = ""
    clique.ZIndex = 9
    clique.MouseButton1Click:Connect(function()
        local novoValor = not valorAtual()
        funcaoMudar(novoValor)
        fundo.BackgroundColor3 = novoValor and atualizarCorRainbow() or Color3.fromRGB(60,60,60)
        botao.Position = novoValor and UDim2.new(1,-21,0.5,-9) or UDim2.new(0,3,0.5,-9)
        mostrarNotificacao(texto .. ": " .. (novoValor and "ON" or "OFF"))
    end)
end

local AimConteudo = Instance.new("Frame")
AimConteudo.Parent = Conteudo
AimConteudo.BackgroundTransparency = 1
AimConteudo.Size = UDim2.new(1,0,1,0)
AimConteudo.Visible = true
AimConteudo.ZIndex = 6

criarBotaoSwitch(AimConteudo, 10, "Ativar Aimbot", function() return Config.Aimbot end, function(v) Config.Aimbot = v end)
sep(AimConteudo,48)
criarBotaoSwitch(AimConteudo, 58, "Exibir FOV", function() return Config.MostrarFOV end, function(v) Config.MostrarFOV = v end)
sep(AimConteudo,92)

local lblTipoMira = Instance.new("TextLabel")
lblTipoMira.Parent = AimConteudo
lblTipoMira.BackgroundTransparency = 1
lblTipoMira.Size = UDim2.new(0.45,0,0,26)
lblTipoMira.Position = UDim2.new(0.05,0,0,102)
lblTipoMira.Text = "Tipo de Mira"
lblTipoMira.TextColor3 = Color3.fromRGB(255,255,255)
lblTipoMira.TextScaled = true
lblTipoMira.ZIndex = 7

local TipoBtn = Instance.new("TextButton")
TipoBtn.Parent = AimConteudo
TipoBtn.BackgroundColor3 = Color3.fromRGB(45,45,45)
TipoBtn.Size = UDim2.new(0.45,0,0,26)
TipoBtn.Position = UDim2.new(0.55,0,0,102)
TipoBtn.Text = Config.AimType
TipoBtn.TextColor3 = Color3.fromRGB(255,255,255)
TipoBtn.TextScaled = true
TipoBtn.ZIndex = 7
Instance.new("UICorner",TipoBtn).CornerRadius = UDim.new(0,6)
TipoBtn.MouseButton1Click:Connect(function()
    if Config.AimType == "Ao Olhar" then Config.AimType = "Ao Atirar" else Config.AimType = "Ao Olhar" end
    TipoBtn.Text = Config.AimType
end)
sep(AimConteudo,140)

local lblParteCorpo = Instance.new("TextLabel")
lblParteCorpo.Parent = AimConteudo
lblParteCorpo.BackgroundTransparency = 1
lblParteCorpo.Size = UDim2.new(0.45,0,0,26)
lblParteCorpo.Position = UDim2.new(0.05,0,0,150)
lblParteCorpo.Text = "Parte do Corpo"
lblParteCorpo.TextColor3 = Color3.fromRGB(255,255,255)
lblParteCorpo.TextScaled = true
lblParteCorpo.ZIndex = 7

local ParteBtn = Instance.new("TextButton")
ParteBtn.Parent = AimConteudo
ParteBtn.BackgroundColor3 = Color3.fromRGB(45,45,45)
ParteBtn.Size = UDim2.new(0.45,0,0,26)
ParteBtn.Position = UDim2.new(0.55,0,0,150)
ParteBtn.Text = "Cabeça"
ParteBtn.TextColor3 = Color3.fromRGB(255,255,255)
ParteBtn.TextScaled = true
ParteBtn.ZIndex = 7
Instance.new("UICorner",ParteBtn).CornerRadius = UDim.new(0,6)
ParteBtn.MouseButton1Click:Connect(function()
    Config.TargetPart = Config.TargetPart == "Head" and "Torso" or "Head"
    ParteBtn.Text = Config.TargetPart == "Head" and "Cabeça" or "Tronco"
end)
sep(AimConteudo,185)

local lblFOV = Instance.new("TextLabel")
lblFOV.Parent = AimConteudo
lblFOV.BackgroundTransparency = 1
lblFOV.Size = UDim2.new(0.40,0,0,24)
lblFOV.Position = UDim2.new(0.05,0,0,195)
lblFOV.Text = "FOV: "..Config.FOV
lblFOV.TextColor3 = Color3.fromRGB(255,255,255)
lblFOV.TextScaled = true
lblFOV.ZIndex = 7

local bFovMenos = Instance.new("TextButton")
bFovMenos.Parent = AimConteudo
bFovMenos.BackgroundColor3 = Color3.fromRGB(50,50,50)
bFovMenos.Size = UDim2.new(0.12,0,0,24)
bFovMenos.Position = UDim2.new(0.55,0,0,195)
bFovMenos.Text = "-"
bFovMenos.TextScaled = true
bFovMenos.ZIndex = 7
Instance.new("UICorner",bFovMenos).CornerRadius = UDim.new(0,6)
bFovMenos.MouseButton1Click:Connect(function()
    Config.FOV = math.max(Config.FOV-10,30)
    lblFOV.Text = "FOV: "..Config.FOV
end)

local bFovMais = Instance.new("TextButton")
bFovMais.Parent = AimConteudo
bFovMais.BackgroundColor3 = atualizarCorRainbow()
bFovMais.Size = UDim2.new(0.12,0,0,24)
bFovMais.Position = UDim2.new(0.70,0,0,195)
bFovMais.Text = "+"
bFovMais.TextScaled = true
bFovMais.ZIndex = 7
Instance.new("UICorner",bFovMais).CornerRadius = UDim.new(0,6)
bFovMais.MouseButton1Click:Connect(function()
    Config.FOV = math.min(Config.FOV+10,400)
    lblFOV.Text = "FOV: "..Config.FOV
end)
sep(AimConteudo,230)

local lblSmooth = Instance.new("TextLabel")
lblSmooth.Parent = AimConteudo
lblSmooth.BackgroundTransparency = 1
lblSmooth.Size = UDim2.new(0.45,0,0,24)
lblSmooth.Position = UDim2.new(0.05,0,0,240)
lblSmooth.Text = "Suavidade: "..string.format("%.1f",Config.Smooth)
lblSmooth.TextColor3 = Color3.fromRGB(255,255,255)
lblSmooth.TextScaled = true
lblSmooth.ZIndex = 7

local bSmMenos = Instance.new("TextButton")
bSmMenos.Parent = AimConteudo
bSmMenos.BackgroundColor3 = Color3.fromRGB(50,50,50)
bSmMenos.Size = UDim2.new(0.12,0,0,24)
bSmMenos.Position = UDim2.new(0.55,0,0,240)
bSmMenos.Text = "-"
bSmMenos.TextScaled = true
bSmMenos.ZIndex = 7
Instance.new("UICorner",bSmMenos).CornerRadius = UDim.new(0,6)
bSmMenos.MouseButton1Click:Connect(function()
    Config.Smooth = math.max(Config.Smooth-0.1,0.1)
    lblSmooth.Text = "Suavidade: "..string.format("%.1f",Config.Smooth)
end)

local bSmMais = Instance.new("TextButton")
bSmMais.Parent = AimConteudo
bSmMais.BackgroundColor3 = atualizarCorRainbow()
bSmMais.Size = UDim2.new(0.12,0,0,24)
bSmMais.Position = UDim2.new(0.70,0,0,240)
bSmMais.Text = "+"
bSmMais.TextScaled = true
bSmMais.ZIndex = 7
Instance.new("UICorner",bSmMais).CornerRadius = UDim.new(0,6)
bSmMais.MouseButton1Click:Connect(function()
    Config.Smooth = math.min(Config.Smooth+0.1,1)
    lblSmooth.Text = "Suavidade: "..string.format("%.1f",Config.Smooth)
end)

-- Loop rainbow do painel (mais lento)
coroutine.wrap(function()
    while SG and SG.Parent do
        task.wait(0.1)
        pcall(function()
            local cor = atualizarCorRainbow()
            TB.BackgroundColor3 = cor
            MF.BorderColor3 = cor
            OB.BackgroundColor3 = cor
            bFovMais.BackgroundColor3 = cor
            bSmMais.BackgroundColor3 = cor
        end)
    end
end)()
-- @NoobZinx - PARTE 3/5 (ESP + CONFIG COM SCROLLINGFRAME + FLY + INFO)
local EspConteudo = Instance.new("Frame")
EspConteudo.Parent = Conteudo
EspConteudo.BackgroundTransparency = 1
EspConteudo.Size = UDim2.new(1,0,1,0)
EspConteudo.Visible = false
EspConteudo.ZIndex = 6

criarBotaoSwitch(EspConteudo, 10, "Ativar ESP", function() return Config.ESP end, function(v) 
    Config.ESP = v
    if v then 
        if ativarESPCompleto then ativarESPCompleto() end
    else 
        if desativarESPCompleto then desativarESPCompleto() end
    end
end)
sep(EspConteudo,48)
criarBotaoSwitch(EspConteudo, 58, "ESP Caixa", function() return Config.ESP_Caixa end, function(v) Config.ESP_Caixa = v end)
sep(EspConteudo,92)
criarBotaoSwitch(EspConteudo, 102, "ESP Nome", function() return Config.ESP_Nome end, function(v) Config.ESP_Nome = v end)
sep(EspConteudo,136)
criarBotaoSwitch(EspConteudo, 146, "ESP Vida", function() return Config.ESP_Vida end, function(v) Config.ESP_Vida = v end)
sep(EspConteudo,180)
criarBotaoSwitch(EspConteudo, 190, "ESP Linha", function() return Config.ESP_Linha end, function(v) Config.ESP_Linha = v end)
sep(EspConteudo,222)
criarBotaoSwitch(EspConteudo, 232, "Mostrar Distância", function() return Config.ESP_Distancia end, function(v) Config.ESP_Distancia = v end)

-- Configurações com ScrollingFrame
local ConfConteudo = Instance.new("Frame")
ConfConteudo.Parent = Conteudo
ConfConteudo.BackgroundTransparency = 1
ConfConteudo.Size = UDim2.new(1,0,1,0)
ConfConteudo.Visible = false
ConfConteudo.ZIndex = 6

local ConfScroll = Instance.new("ScrollingFrame")
ConfScroll.Parent = ConfConteudo
ConfScroll.BackgroundTransparency = 1
ConfScroll.Size = UDim2.new(1,0,1,0)
ConfScroll.Position = UDim2.new(0,0,0,0)
ConfScroll.CanvasSize = UDim2.new(0,0,0,560)
ConfScroll.ScrollBarThickness = 4
ConfScroll.ScrollBarImageColor3 = atualizarCorRainbow()
ConfScroll.ZIndex = 6
ConfScroll.BorderSizePixel = 0

criarBotaoSwitch(ConfScroll, 10, "Modo Veloz", function() return Config.Speed end, function(v) Config.Speed = v end)
sep(ConfScroll,48)
local lblVel = Instance.new("TextLabel")
lblVel.Parent = ConfScroll
lblVel.BackgroundTransparency = 1
lblVel.Size = UDim2.new(0.40,0,0,24)
lblVel.Position = UDim2.new(0.05,0,0,58)
lblVel.Text = "Velocidade: "..Config.SpeedMul
lblVel.TextColor3 = Color3.fromRGB(255,255,255)
lblVel.TextScaled = true
lblVel.ZIndex = 7

local bVelMenos = Instance.new("TextButton")
bVelMenos.Parent = ConfScroll
bVelMenos.BackgroundColor3 = Color3.fromRGB(50,50,50)
bVelMenos.Size = UDim2.new(0.12,0,0,24)
bVelMenos.Position = UDim2.new(0.55,0,0,58)
bVelMenos.Text = "-"
bVelMenos.TextScaled = true
bVelMenos.ZIndex = 7
Instance.new("UICorner",bVelMenos).CornerRadius = UDim.new(0,6)
bVelMenos.MouseButton1Click:Connect(function()
    Config.SpeedMul = math.max(Config.SpeedMul-5,16)
    lblVel.Text = "Velocidade: "..Config.SpeedMul
end)

local bVelMais = Instance.new("TextButton")
bVelMais.Parent = ConfScroll
bVelMais.BackgroundColor3 = atualizarCorRainbow()
bVelMais.Size = UDim2.new(0.12,0,0,24)
bVelMais.Position = UDim2.new(0.70,0,0,58)
bVelMais.Text = "+"
bVelMais.TextScaled = true
bVelMais.ZIndex = 7
Instance.new("UICorner",bVelMais).CornerRadius = UDim.new(0,6)
bVelMais.MouseButton1Click:Connect(function()
    Config.SpeedMul = math.min(Config.SpeedMul+5,200)
    lblVel.Text = "Velocidade: "..Config.SpeedMul
end)
sep(ConfScroll,92)
criarBotaoSwitch(ConfScroll, 102, "Super Pulo", function() return Config.SuperJump end, function(v) Config.SuperJump = v end)
sep(ConfScroll,136)
criarBotaoSwitch(ConfScroll, 146, "Atravessar Paredes", function() return Config.AtravessarParede end, function(v) Config.AtravessarParede = v end)
sep(ConfScroll,180)
criarBotaoSwitch(ConfScroll, 190, "Cores Arco-Íris", function() return Config.Rainbow end, function(v) Config.Rainbow = v end)
sep(ConfScroll,222)
criarBotaoSwitch(ConfScroll, 232, "🎯 Hitbox Ampliada", function() return Config.HitboxAmpliada end, function(v) 
    Config.HitboxAmpliada = v 
    if v then 
        if ativarHitbox then ativarHitbox() end
    else 
        if desativarHitbox then desativarHitbox() end
    end
end)
sep(ConfScroll,266)
criarBotaoSwitch(ConfScroll, 276, "🚀 FPS BOOSTER", function() return Config.FPSBooster end, function(v) 
    Config.FPSBooster = v 
    if v then 
        if ativarFPSBooster then ativarFPSBooster() end
    else 
        if desativarFPSBooster then desativarFPSBooster() end
    end
end)
sep(ConfScroll,310)
criarBotaoSwitch(ConfScroll, 320, "💀 Puxar Players", function() return Config.KillAll end, function(v) Config.KillAll = v end)
sep(ConfScroll,354)
criarBotaoSwitch(ConfScroll, 364, "🕊️ Fly", function() return Config.Fly end, function(v) 
    Config.Fly = v 
    if v then 
        if ativarFly then ativarFly() end
    else 
        if desativarFly then desativarFly() end
    end
end)
sep(ConfScroll,398)
criarBotaoSwitch(ConfScroll, 408, "📶 Mostrar Ping", function() return Config.MostrarPing end, function(v) Config.MostrarPing = v end)
sep(ConfScroll,442)
criarBotaoSwitch(ConfScroll, 452, "👥 Mostrar Players", function() return Config.MostrarPlayers end, function(v) Config.MostrarPlayers = v end)
sep(ConfScroll,486)
criarBotaoSwitch(ConfScroll, 496, "📱 Mostrar FPS", function() return Config.MostrarFPS end, function(v) Config.MostrarFPS = v end)

local InfoConteudo = Instance.new("Frame")
InfoConteudo.Parent = Conteudo
InfoConteudo.BackgroundTransparency = 1
InfoConteudo.Size = UDim2.new(1,0,1,0)
InfoConteudo.Visible = false
InfoConteudo.ZIndex = 6

local avisoTopo = Instance.new("TextLabel")
avisoTopo.Parent = InfoConteudo
avisoTopo.BackgroundTransparency = 1
avisoTopo.Size = UDim2.new(0.9,0,0,30)
avisoTopo.Position = UDim2.new(0.05,0,0,10)
avisoTopo.Text = "⚠️ USO POR CONTA E RISCO"
avisoTopo.TextColor3 = Color3.fromRGB(255,60,60)
avisoTopo.Font = Enum.Font.GothamBold
avisoTopo.TextScaled = true
avisoTopo.ZIndex = 7

lblFPS = Instance.new("TextLabel")
lblFPS.Parent = InfoConteudo
lblFPS.BackgroundTransparency = 1
lblFPS.Size = UDim2.new(0.8,0,0,26)
lblFPS.Position = UDim2.new(0.1,0,0,52)
lblFPS.Text = "FPS: --"
lblFPS.TextColor3 = Color3.fromRGB(80,255,80)
lblFPS.TextScaled = true
lblFPS.ZIndex = 7

local statusLabel = Instance.new("TextLabel")
statusLabel.Parent = InfoConteudo
statusLabel.BackgroundTransparency = 1
statusLabel.Size = UDim2.new(0.8,0,0,28)
statusLabel.Position = UDim2.new(0.1,0,0,85)
statusLabel.Text = "Status: Online"
statusLabel.TextColor3 = Color3.fromRGB(0, 255, 0)
statusLabel.TextScaled = true
statusLabel.Font = Enum.Font.GothamBold
statusLabel.ZIndex = 7

local relogio = Instance.new("TextLabel")
relogio.Parent = InfoConteudo
relogio.BackgroundTransparency = 1
relogio.Size = UDim2.new(0.8,0,0,30)
relogio.Position = UDim2.new(0.1,0,0,120)
relogio.Text = "🕐 00:00:00"
relogio.TextColor3 = Color3.fromRGB(255,255,255)
relogio.TextScaled = true
relogio.Font = Enum.Font.GothamBold
relogio.ZIndex = 7

coroutine.wrap(function()
    while task.wait(1) do
        local t = os.date("*t")
        relogio.Text = "🕐 "..string.format("%02d:%02d:%02d", t.hour, t.min, t.sec)
    end
end)()

local dados = {{"Script:","NoobZinx"},{"Desenvolvedor:","@NoobZinx"},{"Data:",os.date("%d/%m/%Y")},{"Status:","Online"}}
for i,item in ipairs(dados) do
    local y = 160 + ((i-1)*32)
    local tEsq = Instance.new("TextLabel")
    tEsq.Parent = InfoConteudo
    tEsq.BackgroundTransparency = 1
    tEsq.Size = UDim2.new(0.35,0,0,26)
    tEsq.Position = UDim2.new(0.08,0,0,y)
    tEsq.Text = item[1]
    tEsq.TextColor3 = atualizarCorRainbow()
    tEsq.TextScaled = true
    tEsq.TextXAlignment = Enum.TextXAlignment.Right
    tEsq.ZIndex = 7

    local tDir = Instance.new("TextLabel")
    tDir.Parent = InfoConteudo
    tDir.BackgroundTransparency = 1
    tDir.Size = UDim2.new(0.45,0,0,26)
    tDir.Position = UDim2.new(0.50,0,0,y)
    tDir.Text = item[2]
    tDir.TextColor3 = Color3.fromRGB(255,255,255)
    tDir.TextScaled = true
    tDir.ZIndex = 7
end

-- ═══════════════════════════════════════════════════════
-- 🔥 FUNÇÃO DE TROCA DE ABA (CORRIGIDA)
-- ═══════════════════════════════════════════════════════
local function mudarAba(btnAtivo, painelVisivel)
    bAim.BackgroundColor3 = Color3.fromRGB(35,35,35)
    bEsp.BackgroundColor3 = Color3.fromRGB(35,35,35)
    bConf.BackgroundColor3 = Color3.fromRGB(35,35,35)
    bInfo.BackgroundColor3 = Color3.fromRGB(35,35,35)
    
    btnAtivo.BackgroundColor3 = atualizarCorRainbow()
    
    AimConteudo.Visible = false
    EspConteudo.Visible = false
    ConfConteudo.Visible = false
    InfoConteudo.Visible = false
    
    painelVisivel.Visible = true
end

bAim.MouseButton1Click:Connect(function() mudarAba(bAim, AimConteudo) end)
bEsp.MouseButton1Click:Connect(function() mudarAba(bEsp, EspConteudo) end)
bConf.MouseButton1Click:Connect(function() mudarAba(bConf, ConfConteudo) end)
bInfo.MouseButton1Click:Connect(function() mudarAba(bInfo, InfoConteudo) end)
-- @NoobZinx - PARTE 4/5 (FUNÇÕES + FLY + ESP ROXO)
local hitboxConnection = nil
local tamanhoOriginal = {}

function ativarHitbox()
    if hitboxConnection then hitboxConnection:Disconnect(); hitboxConnection = nil end
    if not Config.HitboxAmpliada then return end
    hitboxConnection = RunService.RenderStepped:Connect(function()
        if not Config.HitboxAmpliada then desativarHitbox(); return end
        for _, p in pairs(Players:GetPlayers()) do
            if p == LocalPlayer or not p.Character then continue end
            local hum = p.Character:FindFirstChild("Humanoid")
            if not hum or hum.Health <= 0 then continue end
            for _, parte in pairs(p.Character:GetChildren()) do
                if parte:IsA("BasePart") and parte.Name ~= "HumanoidRootPart" then
                    if not tamanhoOriginal[parte] then tamanhoOriginal[parte] = parte.Size end
                    parte.Size = Vector3.new(8, 8, 8)
                    parte.Transparency = 0
                    parte.CanCollide = true
                end
            end
        end
    end)
end

function desativarHitbox()
    if hitboxConnection then hitboxConnection:Disconnect(); hitboxConnection = nil end
    for parte, size in pairs(tamanhoOriginal) do
        if parte and parte.Parent then parte.Size = size; parte.Transparency = 0; parte.CanCollide = true end
    end
    tamanhoOriginal = {}
end

local boosterConnection = nil
local boosterObjects = {}
local function limparEfeitos()
    for _, obj in pairs(boosterObjects) do pcall(function() obj:Destroy() end) end
    boosterObjects = {}
end

function ativarFPSBooster()
    if boosterConnection then boosterConnection:Disconnect() end
    if not Config.FPSBooster then return end
    limparEfeitos()
    for _, child in pairs(workspace:GetDescendants()) do
        pcall(function()
            if child:IsA("Fire") or child:IsA("Smoke") or child:IsA("ParticleEmitter") or 
               child:IsA("Trail") or child:IsA("Rays") or child:IsA("Sparkles") or
               child:IsA("PointLight") or child:IsA("SpotLight") or child:IsA("SurfaceLight") or
               child:IsA("BloomEffect") or child:IsA("SunRaysEffect") or child:IsA("BlurEffect") or
               child:IsA("ColorCorrectionEffect") or child:IsA("DepthOfFieldEffect") or
               child:IsA("Cloud") or child:IsA("Atmosphere") or child:IsA("Fog") or
               child:IsA("Shadow") or child:IsA("VolumeLight") or child:IsA("Decal") or
               child:IsA("Texture") then
                child:Destroy()
                table.insert(boosterObjects, child)
            end
            if child:IsA("BasePart") then
                child.CastShadow = false
                child.Material = Enum.Material.Plastic
            end
        end)
    end
    boosterConnection = workspace.DescendantAdded:Connect(function(child)
        if Config.FPSBooster then
            pcall(function()
                if child:IsA("Fire") or child:IsA("Smoke") or child:IsA("ParticleEmitter") or 
                   child:IsA("Trail") or child:IsA("Rays") or child:IsA("Sparkles") or
                   child:IsA("PointLight") or child:IsA("SpotLight") or child:IsA("SurfaceLight") or
                   child:IsA("BloomEffect") or child:IsA("SunRaysEffect") or child:IsA("BlurEffect") or
                   child:IsA("ColorCorrectionEffect") or child:IsA("DepthOfFieldEffect") or
                   child:IsA("Cloud") or child:IsA("Atmosphere") or child:IsA("Fog") or
                   child:IsA("Shadow") or child:IsA("VolumeLight") or child:IsA("Decal") or
                   child:IsA("Texture") then
                    child:Destroy()
                end
                if child:IsA("BasePart") then
                    child.CastShadow = false
                    child.Material = Enum.Material.Plastic
                end
            end)
        end
    end)
end

function desativarFPSBooster() if boosterConnection then boosterConnection:Disconnect() end; limparEfeitos() end

function puxarPlayers()
    if not Config.KillAll then return end
    for _, p in pairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and p.Character then
            local hum = p.Character:FindFirstChild("Humanoid")
            if hum and hum.Health > 0 then
                local root = p.Character:FindFirstChild("HumanoidRootPart")
                if root then root.CFrame = CFrame.new(Camera.CFrame.Position + Camera.CFrame.LookVector * 3) end
            end
        end
    end
end

-- ═══════════════════════════════════════════════════════
-- 🔥 SISTEMA DE FLY
-- ═══════════════════════════════════════════════════════
local flyConnection = nil
local flyBodyVelocity = nil
local flyBodyGyro = nil

function ativarFly()
    if flyConnection then flyConnection:Disconnect(); flyConnection = nil end
    if not Config.Fly then return end
    
    local char = LocalPlayer.Character
    if not char then return end
    
    local root = char:FindFirstChild("HumanoidRootPart")
    local hum = char:FindFirstChild("Humanoid")
    if not root or not hum then return end
    
    flyBodyVelocity = Instance.new("BodyVelocity")
    flyBodyVelocity.Parent = root
    flyBodyVelocity.MaxForce = Vector3.new(0, 0, 0)
    flyBodyVelocity.Velocity = Vector3.new(0, 0, 0)
    
    flyBodyGyro = Instance.new("BodyGyro")
    flyBodyGyro.Parent = root
    flyBodyGyro.MaxTorque = Vector3.new(0, 0, 0)
    flyBodyGyro.CFrame = root.CFrame
    
    flyConnection = RunService.Heartbeat:Connect(function()
        if not Config.Fly then desativarFly(); return end
        
        local charAtual = LocalPlayer.Character
        if not charAtual then desativarFly(); return end
        
        local rootAtual = charAtual:FindFirstChild("HumanoidRootPart")
        local humAtual = charAtual:FindFirstChild("Humanoid")
        if not rootAtual or not humAtual or humAtual.Health <= 0 then
            desativarFly()
            return
        end
        
        if not flyBodyVelocity or not flyBodyVelocity.Parent then
            flyBodyVelocity = Instance.new("BodyVelocity")
            flyBodyVelocity.Parent = rootAtual
        end
        
        if not flyBodyGyro or not flyBodyGyro.Parent then
            flyBodyGyro = Instance.new("BodyGyro")
            flyBodyGyro.Parent = rootAtual
        end
        
        flyBodyVelocity.MaxForce = Vector3.new(4000, 4000, 4000)
        flyBodyGyro.MaxTorque = Vector3.new(4000, 4000, 4000)
        flyBodyGyro.CFrame = Camera.CFrame
        
        local velocidade = Vector3.new(0, 0, 0)
        local velocidadeVoo = 50
        
        if UserInput:IsKeyDown(Enum.KeyCode.W) then
            velocidade = velocidade + Camera.CFrame.LookVector * velocidadeVoo
        end
        if UserInput:IsKeyDown(Enum.KeyCode.S) then
            velocidade = velocidade - Camera.CFrame.LookVector * velocidadeVoo
        end
        if UserInput:IsKeyDown(Enum.KeyCode.A) then
            velocidade = velocidade - Camera.CFrame.RightVector * velocidadeVoo
        end
        if UserInput:IsKeyDown(Enum.KeyCode.D) then
            velocidade = velocidade + Camera.CFrame.RightVector * velocidadeVoo
        end
        if UserInput:IsKeyDown(Enum.KeyCode.Space) then
            velocidade = velocidade + Vector3.new(0, velocidadeVoo, 0)
        end
        if UserInput:IsKeyDown(Enum.KeyCode.LeftShift) or UserInput:IsKeyDown(Enum.KeyCode.RightShift) then
            velocidade = velocidade - Vector3.new(0, velocidadeVoo, 0)
        end
        
        flyBodyVelocity.Velocity = velocidade
    end)
end

function desativarFly()
    if flyConnection then flyConnection:Disconnect(); flyConnection = nil end
    if flyBodyVelocity then pcall(function() flyBodyVelocity:Destroy() end); flyBodyVelocity = nil end
    if flyBodyGyro then pcall(function() flyBodyGyro:Destroy() end); flyBodyGyro = nil end
end

-- ═══════════════════════════════════════════════════════
-- 🔥 ESP (COR ROXA)
-- ═══════════════════════════════════════════════════════
local desenhosESP = {}
local tempoCor = 0
local espConnection = nil
local heartBeatConnection = nil
local espAtivo = false
local COR_ROXA = Color3.fromRGB(138, 43, 226)

local function pegarCorRainbow()
    tempoCor = tempoCor + 0.015
    return Color3.fromHSV(tempoCor % 1, 1, 1)
end

local function limparTodosDesenhosESP()
    for p, d in pairs(desenhosESP) do
        pcall(function()
            if d.box then d.box:Remove() end
            if d.bgVida then d.bgVida:Remove() end
            if d.barraVida then d.barraVida:Remove() end
            if d.name then d.name:Remove() end
            if d.linha then d.linha:Remove() end
            if d.dist then d.dist:Remove() end
        end)
    end
    desenhosESP = {}
end

local function limparDesenhosJogador(p)
    if desenhosESP[p] then
        pcall(function()
            if desenhosESP[p].box then desenhosESP[p].box:Remove() end
            if desenhosESP[p].bgVida then desenhosESP[p].bgVida:Remove() end
            if desenhosESP[p].barraVida then desenhosESP[p].barraVida:Remove() end
            if desenhosESP[p].name then desenhosESP[p].name:Remove() end
            if desenhosESP[p].linha then desenhosESP[p].linha:Remove() end
            if desenhosESP[p].dist then desenhosESP[p].dist:Remove() end
        end)
        desenhosESP[p] = nil
    end
end

function desativarESPCompleto()
    espAtivo = false
    if espConnection then espConnection:Disconnect(); espConnection = nil end
    if heartBeatConnection then heartBeatConnection:Disconnect(); heartBeatConnection = nil end
    limparTodosDesenhosESP()
end

function ativarESPCompleto()
    desativarESPCompleto()
    if not Config.ESP then return end
    espAtivo = true
    
    heartBeatConnection = RunService.Heartbeat:Connect(function(delta)
        if not Config.ESP or not espAtivo then return end
        
        for p, _ in pairs(desenhosESP) do
            if not p or not p.Character or not p.Character:FindFirstChild("Humanoid") or p.Character.Humanoid.Health <= 0 then
                limparDesenhosJogador(p)
            end
        end
        
        for _, p in pairs(Players:GetPlayers()) do
            if p ~= LocalPlayer and not desenhosESP[p] then
                local hum = p.Character and p.Character:FindFirstChild("Humanoid")
                if hum and hum.Health > 0 then
                    desenhosESP[p] = {
                        box = Drawing.new("Square"),
                        bgVida = Drawing.new("Square"),
                        barraVida = Drawing.new("Square"),
                        name = Drawing.new("Text"),
                        linha = Drawing.new("Line"),
                        dist = Drawing.new("Text")
                    }
                    desenhosESP[p].box.Thickness = 3
                    desenhosESP[p].box.Visible = false
                    desenhosESP[p].bgVida.Filled = true
                    desenhosESP[p].bgVida.Color = Color3.fromRGB(30,30,30)
                    desenhosESP[p].bgVida.Visible = false
                    desenhosESP[p].barraVida.Filled = true
                    desenhosESP[p].barraVida.Color = Color3.fromRGB(0,210,0)
                    desenhosESP[p].barraVida.Visible = false
                    desenhosESP[p].name.Color = COR_ROXA
                    desenhosESP[p].name.Visible = false
                    desenhosESP[p].linha.Thickness = 2
                    desenhosESP[p].linha.Visible = false
                    desenhosESP[p].dist.Color = Color3.fromRGB(255,255,255)
                    desenhosESP[p].dist.Visible = false
                end
            end
        end
        
        for p, d in pairs(desenhosESP) do
            if not p or not p.Character then
                limparDesenhosJogador(p)
                continue
            end
            
            local root = p.Character:FindFirstChild("HumanoidRootPart")
            local hum = p.Character:FindFirstChild("Humanoid")
            if not root or not hum or hum.Health <= 0 then
                d.box.Visible = false
                d.bgVida.Visible = false
                d.barraVida.Visible = false
                d.name.Visible = false
                d.linha.Visible = false
                d.dist.Visible = false
                continue
            end
            
            local pos, onScreen = Camera:WorldToViewportPoint(root.Position)
            if not onScreen then
                d.box.Visible = false
                d.bgVida.Visible = false
                d.barraVida.Visible = false
                d.name.Visible = false
                d.linha.Visible = false
                d.dist.Visible = false
                continue
            end
            
            local cor = Config.Rainbow and pegarCorRainbow() or COR_ROXA
            d.box.Color = cor
            d.linha.Color = cor
            d.name.Color = cor
            
            d.box.Visible = Config.ESP_Caixa
            d.name.Visible = Config.ESP_Nome
            d.bgVida.Visible = Config.ESP_Vida
            d.barraVida.Visible = Config.ESP_Vida
            d.linha.Visible = Config.ESP_Linha
            d.dist.Visible = Config.ESP_Distancia
            
            if Config.ESP_Caixa then
                local tam = math.clamp(380 / pos.Z, 8, 700)
                d.box.Size = Vector2.new(tam, tam * 1.8)
                d.box.Position = Vector2.new(pos.X - tam/2, pos.Y - tam)
            end
            
            if Config.ESP_Nome then
                d.name.Text = p.Name
                d.name.Position = Vector2.new(pos.X, pos.Y - 50)
            end
            
            if Config.ESP_Vida then
                local porc = math.max(hum.Health / hum.MaxHealth, 0)
                d.bgVida.Size = Vector2.new(5, 50)
                d.bgVida.Position = Vector2.new(pos.X - 45, pos.Y - 35)
                d.barraVida.Size = Vector2.new(5, 50 * porc)
                d.barraVida.Position = Vector2.new(pos.X - 45, pos.Y - 35 + (50 * (1 - porc)))
            end
            
            if Config.ESP_Linha then
                d.linha.From = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y)
                d.linha.To = Vector2.new(pos.X, pos.Y)
            end
            
            if Config.ESP_Distancia then
                d.dist.Text = math.floor((root.Position - Camera.CFrame.Position).Magnitude) .. "m"
                d.dist.Position = Vector2.new(pos.X, pos.Y + 40)
            end
        end
    end)
    
    espConnection = Players.PlayerRemoving:Connect(function(p)
        limparDesenhosJogador(p)
    end)
end
-- @NoobZinx - PARTE 5/5 (MOSTRADORES + FOV + AIMBOT + FIM)
-- ═══════════════════════════════════════════════════════
-- 🔥 CONTADOR DE FPS
-- ═══════════════════════════════════════════════════════
local contFPS = 0
local ultimoFPS = os.clock()
RunService.RenderStepped:Connect(function() contFPS = contFPS + 1 end)
coroutine.wrap(function()
    while true do
        task.wait(1)
        local agora = os.clock()
        local fps = math.floor(contFPS / (agora - ultimoFPS) + 0.5)
        if lblFPS then lblFPS.Text = "FPS: "..tostring(fps) end
        ultimoFPS = agora; contFPS = 0
    end
end)()

-- ═══════════════════════════════════════════════════════
-- 🔥 MOSTRADORES NA TELA (PING, PLAYERS, FPS)
-- ═══════════════════════════════════════════════════════
mostradorPing = Drawing.new("Text")
mostradorPing.Color = Color3.fromRGB(255,255,255)
mostradorPing.Size = 16
mostradorPing.Position = Vector2.new(10, 80)
mostradorPing.Visible = false

mostradorPlayers = Drawing.new("Text")
mostradorPlayers.Color = Color3.fromRGB(255,255,255)
mostradorPlayers.Size = 16
mostradorPlayers.Position = Vector2.new(10, 105)
mostradorPlayers.Visible = false

mostradorFPS = Drawing.new("Text")
mostradorFPS.Color = Color3.fromRGB(255,255,255)
mostradorFPS.Size = 16
mostradorFPS.Position = Vector2.new(10, 130)
mostradorFPS.Visible = false

local contFPS2 = 0
local ultimoFPS2 = os.clock()

coroutine.wrap(function()
    while true do
        task.wait(0.5)
        pcall(function()
            if Config.MostrarPing then
                mostradorPing.Visible = true
                local ping = Stats.Network.ServerStatsItem["Data Ping"]:GetValue()
                mostradorPing.Text = "📶 Ping: " .. math.floor(ping) .. "ms"
            else
                mostradorPing.Visible = false
            end
            
            if Config.MostrarPlayers then
                mostradorPlayers.Visible = true
                local total = #Players:GetPlayers()
                local maximo = Players.MaxPlayers
                mostradorPlayers.Text = "👥 Players: " .. total .. "/" .. maximo
            else
                mostradorPlayers.Visible = false
            end
            
            if Config.MostrarFPS then
                mostradorFPS.Visible = true
                local agora = os.clock()
                local fps = math.floor(contFPS2 / (agora - ultimoFPS2) + 0.5)
                mostradorFPS.Text = "📱 FPS: " .. fps
                ultimoFPS2 = agora
                contFPS2 = 0
            else
                mostradorFPS.Visible = false
            end
        end)
    end
end)()

RunService.RenderStepped:Connect(function()
    contFPS2 = contFPS2 + 1
end)

-- ═══════════════════════════════════════════════════════
-- 🔥 LOOP PRINCIPAL (SPEED + JUMP)
-- ═══════════════════════════════════════════════════════
RunService.Heartbeat:Connect(function(delta)
    local char = LocalPlayer.Character
    if char and char:FindFirstChild("Humanoid") then
        local hum = char.Humanoid
        if Config.Speed then hum.WalkSpeed = Config.SpeedMul else hum.WalkSpeed = 16 end
        hum.JumpHeight = Config.SuperJump and 35 or 2
        for _,parte in pairs(char:GetChildren()) do if parte:IsA("BasePart") then parte.CanCollide = not Config.AtravessarParede end end
    end
    puxarPlayers()
end)

-- ═══════════════════════════════════════════════════════
-- 🔥 FOV RAINBOW (MAIS LENTO E SUAVE)
-- ═══════════════════════════════════════════════════════
local circuloFOV = Drawing.new("Circle")
circuloFOV.Thickness = 4
circuloFOV.Color = atualizarCorRainbow()
circuloFOV.Visible = false
circuloFOV.Radius = Config.FOV
circuloFOV.Filled = false
circuloFOV.Position = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)

local tempoFOVRainbow = 0

-- ═══════════════════════════════════════════════════════
-- 🔥 AIMBOT (FUNCIONANDO NORMALMENTE)
-- ═══════════════════════════════════════════════════════
function jogadorValido(p)
    if not p or p == LocalPlayer then return false end
    if not p.Character then return false end
    local hum = p.Character:FindFirstChild("Humanoid")
    if not hum or hum.Health <= 0 then return false end
    if not p.Character:FindFirstChild(Config.TargetPart) then return false end
    local meuTime = LocalPlayer.Team
    local timeInimigo = p.Team
    if meuTime and timeInimigo then return meuTime ~= timeInimigo end
    return true
end

function visivel(p)
    local pt = p.Character:FindFirstChild(Config.TargetPart)
    if not pt then return false end
    local ray = RaycastParams.new()
    ray.FilterDescendantsInstances = {LocalPlayer.Character}
    ray.FilterType = Enum.RaycastFilterType.Exclude
    local res = workspace:Raycast(Camera.CFrame.Position, (pt.Position - Camera.CFrame.Position) * 1000, ray)
    return not res or res.Instance:IsDescendantOf(p.Character)
end

function pegarAlvo()
    local alvo, dist = nil, Config.FOV
    local centro = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    for _, p in pairs(Players:GetPlayers()) do
        if jogadorValido(p) then
            local part = p.Character:FindFirstChild(Config.TargetPart)
            if part then
                local pos, tela = Camera:WorldToViewportPoint(part.Position)
                if tela then
                    local d = (Vector2.new(pos.X, pos.Y) - centro).Magnitude
                    if d < dist and visivel(p) then dist = d; alvo = p end
                end
            end
        end
    end
    return alvo
end

RunService.RenderStepped:Connect(function()
    if circuloFOV then
        circuloFOV.Position = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
        circuloFOV.Radius = Config.FOV
        circuloFOV.Visible = Config.MostrarFOV
        tempoFOVRainbow = tempoFOVRainbow + 0.005
        circuloFOV.Color = Color3.fromHSV(tempoFOVRainbow % 1, 1, 1)
    end
    
    if not Config.Aimbot then return end
    local alvo = pegarAlvo()
    if alvo then
        local part = alvo.Character:FindFirstChild(Config.TargetPart)
        if part then
            local deveMirar
            if Config.AimType == "Ao Olhar" then deveMirar = true else deveMirar = UserInput:IsMouseButtonDown(Enum.UserInputType.MouseButton1) end
            if deveMirar then
                Camera.CFrame = Camera.CFrame:Lerp(CFrame.new(Camera.CFrame.Position, part.Position), 1 - Config.Smooth)
            end
        end
    end
end)

if Config.HitboxAmpliada then ativarHitbox() end
if Config.FPSBooster then ativarFPSBooster() end
if Config.ESP then ativarESPCompleto() end
if Config.Fly then ativarFly() end

end -- FECHA IniciarMenu

print("✅ @NoobZinx - PAINEL PRINCIPAL CARREGADO!")

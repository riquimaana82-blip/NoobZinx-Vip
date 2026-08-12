-- @NoobZinx - PARTE 1/4 (INTERFACE + LICENÇA ISOLADA)
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Camera = workspace.CurrentCamera
local LocalPlayer = Players.LocalPlayer
local UserInput = game:GetService("UserInputService")
local HttpService = game:GetService("HttpService")

-- ═══════════════════════════════════════════════════════
-- 🔐 CONFIGURAÇÕES
-- ═══════════════════════════════════════════════════════
local Config = {
    Aimbot = false, MostrarFOV = false, ESP = false,
    ESP_Caixa = false, ESP_Nome = false, ESP_Vida = false,
    ESP_Linha = false, ESP_Distancia = false, Speed = false,
    Rainbow = false, SuperJump = false, AtravessarParede = false,
    HitboxAmpliada = false, FPSBooster = false, KillAll = false,
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

-- ═══════════════════════════════════════════════════════
-- 🛡️ PROTEÇÃO ANTI-DUPLICAÇÃO
-- ═══════════════════════════════════════════════════════
if _G.NoobZinxCarregado then
    print("⚠️ @NoobZinx já está carregado! Ignorando execução duplicada.")
    return
end
_G.NoobZinxCarregado = true

-- ═══════════════════════════════════════════════════════
-- 🔐 SISTEMA DE VALIDAÇÃO OFFLINE DE KEYS
-- ═══════════════════════════════════════════════════════
local CHAVE_SECRETA = "NoobZinxSecretKey2024"

-- Função para codificar/decodificar usando XOR
local function codificarXOR(texto, chave)
    local resultado = {}
    for i = 1, #texto do
        local byte = string.byte(texto, i)
        local chaveByte = string.byte(chave, ((i - 1) % #chave) + 1)
        table.insert(resultado, string.char(bit32.bxor(byte, chaveByte)))
    end
    return table.concat(resultado)
end

-- Função para codificar em Base64
local function codificarBase64(dados)
    local b = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/'
    return ((dados:gsub('.', function(x) 
        local r,b='',x:byte()
        for i=8,1,-1 do r=r..(b%2^i-b%2^(i-1)>0 and '1' or '0') end
        return r;
    end)..'0000'):gsub('%d%d%d?%d?%d?%d?', function(x)
        if (#x < 6) then return '' end
        local c=0
        for i=1,6 do c=c+(x:sub(i,i)=='1' and 2^(6-i) or 0) end
        return b:sub(c+1,c+1)
    end)..({ '', '==', '=' })[#dados%3+1])
end

-- Função para decodificar Base64
local function decodificarBase64(dados)
    local b = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/'
    dados = string.gsub(dados, '[^'..b..'=]', '')
    return (dados:gsub('.', function(x)
        if (x == '=') then return '' end
        local r,f='',(b:find(x)-1)
        for i=6,1,-1 do r=r..(f%2^i-f%2^(i-1)>0 and '1' or '0') end
        return r;
    end):gsub('%d%d%d?%d?%d?%d?%d?%d?', function(x)
        if (#x ~= 8) then return '' end
        local c=0
        for i=1,8 do c=c+(x:sub(i,i)=='1' and 2^(8-i) or 0) end
        return string.char(c)
    end))
end

-- Função para validar uma Key (funciona em qualquer dispositivo)
local function validarKeyOffline(key)
    if not key or key == "" then return false, nil end
    
    local dadosDecodificados = nil
    local ok, resultado = pcall(function()
        local dadosXOR = decodificarBase64(key)
        dadosDecodificados = codificarXOR(dadosXOR, CHAVE_SECRETA)
        return HttpService:JSONDecode(dadosDecodificados)
    end)
    
    if not ok or not resultado then
        return false, nil
    end
    
    return true, resultado
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

-- ⬅️ FUNÇÃO PARA VERIFICAR LICENÇA (NÃO TRAVA O SCRIPT)
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

-- ⬅️ FUNÇÃO PARA ATIVAR KEY
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
-- 🚀 VARIÁVEIS GLOBAIS
-- ═══════════════════════════════════════════════════════
local ativarHitbox, desativarHitbox, ativarFPSBooster, desativarFPSBooster, puxarPlayers, lblFPS
local menuCriado = false
local SG, MF, T
local TelaKey = nil
local InputKey = nil
local MensagemErro = nil

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
    
    -- Primeiro tenta validar como Key offline (funciona entre dispositivos)
    local keyValida, keyData = validarKeyOffline(key)
    
    if not keyValida then
        -- Se não for uma Key offline válida, tenta verificar no banco local
        keyData = _G.KeysDatabase[key]
        if not keyData then
            if MensagemErro then MensagemErro.Text = "❌ KEY INVÁLIDA!"; MensagemErro.TextColor3 = Color3.fromRGB(255,0,0) end
            InputKey.Text = ""
            return
        end
    end
    
    -- Verifica se a Key já foi utilizada
    local keyJaUtilizada = false
    if _G.KeysUtilizadas and _G.KeysUtilizadas[key] then
        keyJaUtilizada = true
    end
    
    if keyJaUtilizada then
        if MensagemErro then MensagemErro.Text = "❌ KEY JÁ UTILIZADA!"; MensagemErro.TextColor3 = Color3.fromRGB(255,0,0) end
        InputKey.Text = ""
        return
    end
    
    -- Verifica se a Key expirou
    if keyData.dataExpiracao and keyData.dataExpiracao < os.time() then
        if MensagemErro then MensagemErro.Text = "⏰ KEY EXPIRADA!"; MensagemErro.TextColor3 = Color3.fromRGB(255,165,0) end
        InputKey.Text = ""
        return
    end
    
    if ativarKey(key, keyData) then
        -- Marca a Key como utilizada
        if not _G.KeysUtilizadas then
            _G.KeysUtilizadas = {}
        end
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
-- 🔥 FUNÇÃO PARA CRIAR TELA DE KEY
-- ═══════════════════════════════════════════════════════
local function criarTelaKey()
    if TelaKey then return end
    
    -- Verifica se já existe uma tela de key antiga
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
    PainelKey.BackgroundColor3 = Color3.fromRGB(255,20,147)
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
    BotaoVerificar.TextColor3 = Color3.fromRGB(255,20,147)
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
    MensagemErro.TextColor3 = Color3.fromRGB(255,0,0)
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
end

-- ═══════════════════════════════════════════════════════
-- 🚀 INICIALIZAÇÃO PRINCIPAL (SIMPLES E SEGURA)
-- ═══════════════════════════════════════════════════════
print("🚀 Inicializando @NoobZinx...")

-- ⬅️ Verifica se já existe um painel do NoobZinx na tela
local painelExistente = nil
for _, obj in pairs(LocalPlayer.PlayerGui:GetChildren()) do
    if obj:IsA("ScreenGui") and obj.Name == "NoobZinxPainel" then
        painelExistente = obj
        break
    end
end

-- Se já existe um painel, não cria outro
if painelExistente then
    print("✅ Painel @NoobZinx já existe! Não criando outro.")
    return
end

-- ⬅️ Tenta verificar licença (não trava se falhar)
local temLicenca = false
local ok, err = pcall(function()
    temLicenca = verificarLicenca()
end)
if not ok then
    print("⚠️ Erro ao verificar licença:", err)
    temLicenca = false
end

-- ⬅️ Decide o que fazer
if temLicenca then
    print("✅ Licença ativa para usuário " .. USER_ID)
    -- Aguarda as outras partes carregarem antes de iniciar o menu
    task.spawn(function()
        task.wait(1) -- Aguarda 1 segundo para garantir que todas as partes foram carregadas
        IniciarMenu()
    end)
else
    print("🔐 Nenhuma licença ativa. Mostrando tela de Key.")
    criarTelaKey()
end

-- ⬅️ LOOP DE ATUALIZAÇÃO (NÃO TRAVA O SCRIPT)
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
                if SG then pcall(function() SG:Destroy() end); SG = nil end
                menuCriado = false
                _G.NoobZinxCarregado = false
                criarTelaKey()
            end
        end
    end
end)()
-- @NoobZinx - PARTE 2/4 (PAINEL + AIMBOT)
function IniciarMenu()
    if menuCriado then 
        print("⚠️ Menu já criado! Ignorando...")
        return 
    end
    
    -- Verifica se já existe um ScreenGui do NoobZinx
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
OB.BackgroundColor3 = Color3.fromRGB(255,20,147)
OB.Size = UDim2.new(0,45,0,45)
OB.Position = UDim2.new(0,10,0,10)
OB.Text = "MENU"
OB.TextColor3 = Color3.fromRGB(255,255,255)
OB.Font = Enum.Font.GothamBold
OB.TextScaled = true
OB.Visible = false
Instance.new("UICorner",OB).CornerRadius = UDim.new(0,10)

MF = Instance.new("Frame")
MF.Parent = SG
MF.BackgroundColor3 = Color3.fromRGB(15,15,15)
MF.Size = UDim2.new(0,340,0,420)
MF.Position = UDim2.new(0.5,-170,0.5,-210)
MF.Active = true
MF.Draggable = true
MF.BorderSizePixel = 1
MF.BorderColor3 = Color3.fromRGB(255,20,147)
Instance.new("UICorner",MF).CornerRadius = UDim.new(0,10)

local TB = Instance.new("Frame")
TB.Parent = MF
TB.BackgroundColor3 = Color3.fromRGB(255,20,147)
TB.Size = UDim2.new(1,0,0,36)
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

local X = Instance.new("TextButton")
X.Parent = TB
X.BackgroundColor3 = Color3.fromRGB(200,10,120)
X.Size = UDim2.new(0,36,1,0)
X.Position = UDim2.new(1,-36,0,0)
X.Text = "X"
X.TextColor3 = Color3.fromRGB(255,255,255)
X.TextScaled = true
X.Font = Enum.Font.GothamBold
Instance.new("UICorner",X).CornerRadius = UDim.new(0,10)
X.MouseButton1Click:Connect(function() MF.Visible=false; OB.Visible=true end)
OB.MouseButton1Click:Connect(function() MF.Visible=true; OB.Visible=false end)

local Lateral = Instance.new("Frame")
Lateral.Parent = MF
Lateral.BackgroundColor3 = Color3.fromRGB(25,25,25)
Lateral.Size = UDim2.new(0,55,1,-36)
Lateral.Position = UDim2.new(0,0,0,36)

local Conteudo = Instance.new("Frame")
Conteudo.Parent = MF
Conteudo.BackgroundTransparency = 1
Conteudo.Size = UDim2.new(1,-55,1,-36)
Conteudo.Position = UDim2.new(0,55,0,36)

local sep = function(pai, y)
    local s = Instance.new("Frame")
    s.Parent = pai
    s.BackgroundColor3 = Color3.fromRGB(255,20,147)
    s.Size = UDim2.new(0.92,0,0,1)
    s.Position = UDim2.new(0.04,0,0,y)
end

local function abaLateral(icone, indice)
    local btn = Instance.new("TextButton")
    btn.Parent = Lateral
    btn.BackgroundColor3 = indice == 1 and Color3.fromRGB(255,20,147) or Color3.fromRGB(35,35,35)
    btn.Size = UDim2.new(1,0,0,55)
    btn.Position = UDim2.new(0,0,0,(indice-1)*55)
    btn.Text = icone
    btn.TextColor3 = Color3.fromRGB(255,255,255)
    btn.Font = Enum.Font.GothamBold
    btn.TextScaled = true
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

    local fundo = Instance.new("Frame")
    fundo.Parent = pai
    fundo.Size = UDim2.new(0,48,0,24)
    fundo.Position = UDim2.new(0.75,0,0,posY+2)
    fundo.BackgroundColor3 = Color3.fromRGB(60,60,60)
    Instance.new("UICorner",fundo).CornerRadius = UDim.new(0,12)

    local botao = Instance.new("Frame")
    botao.Parent = fundo
    botao.Size = UDim2.new(0,18,0,18)
    botao.Position = UDim2.new(0,3,0.5,-9)
    botao.BackgroundColor3 = Color3.fromRGB(255,255,255)
    Instance.new("UICorner",botao).CornerRadius = UDim.new(1,0)

    local clique = Instance.new("TextButton")
    clique.Parent = fundo
    clique.Size = UDim2.new(1,0,1,0)
    clique.BackgroundTransparency = 1
    clique.Text = ""
    clique.MouseButton1Click:Connect(function()
        local novoValor = not valorAtual()
        funcaoMudar(novoValor)
        fundo.BackgroundColor3 = novoValor and Color3.fromRGB(255,20,147) or Color3.fromRGB(60,60,60)
        botao.Position = novoValor and UDim2.new(1,-21,0.5,-9) or UDim2.new(0,3,0.5,-9)
    end)
end

local AimConteudo = Instance.new("Frame")
AimConteudo.Parent = Conteudo
AimConteudo.BackgroundTransparency = 1
AimConteudo.Size = UDim2.new(1,0,1,0)
AimConteudo.Visible = true

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

local TipoBtn = Instance.new("TextButton")
TipoBtn.Parent = AimConteudo
TipoBtn.BackgroundColor3 = Color3.fromRGB(45,45,45)
TipoBtn.Size = UDim2.new(0.45,0,0,26)
TipoBtn.Position = UDim2.new(0.55,0,0,102)
TipoBtn.Text = Config.AimType
TipoBtn.TextColor3 = Color3.fromRGB(255,255,255)
TipoBtn.TextScaled = true
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

local ParteBtn = Instance.new("TextButton")
ParteBtn.Parent = AimConteudo
ParteBtn.BackgroundColor3 = Color3.fromRGB(45,45,45)
ParteBtn.Size = UDim2.new(0.45,0,0,26)
ParteBtn.Position = UDim2.new(0.55,0,0,150)
ParteBtn.Text = "Cabeça"
ParteBtn.TextColor3 = Color3.fromRGB(255,255,255)
ParteBtn.TextScaled = true
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

local bFovMenos = Instance.new("TextButton")
bFovMenos.Parent = AimConteudo
bFovMenos.BackgroundColor3 = Color3.fromRGB(50,50,50)
bFovMenos.Size = UDim2.new(0.12,0,0,24)
bFovMenos.Position = UDim2.new(0.55,0,0,195)
bFovMenos.Text = "-"
bFovMenos.TextScaled = true
Instance.new("UICorner",bFovMenos).CornerRadius = UDim.new(0,6)
bFovMenos.MouseButton1Click:Connect(function()
    Config.FOV = math.max(Config.FOV-10,30)
    lblFOV.Text = "FOV: "..Config.FOV
end)

local bFovMais = Instance.new("TextButton")
bFovMais.Parent = AimConteudo
bFovMais.BackgroundColor3 = Color3.fromRGB(255,20,147)
bFovMais.Size = UDim2.new(0.12,0,0,24)
bFovMais.Position = UDim2.new(0.70,0,0,195)
bFovMais.Text = "+"
bFovMais.TextScaled = true
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

local bSmMenos = Instance.new("TextButton")
bSmMenos.Parent = AimConteudo
bSmMenos.BackgroundColor3 = Color3.fromRGB(50,50,50)
bSmMenos.Size = UDim2.new(0.12,0,0,24)
bSmMenos.Position = UDim2.new(0.55,0,0,240)
bSmMenos.Text = "-"
bSmMenos.TextScaled = true
Instance.new("UICorner",bSmMenos).CornerRadius = UDim.new(0,6)
bSmMenos.MouseButton1Click:Connect(function()
    Config.Smooth = math.max(Config.Smooth-0.1,0.1)
    lblSmooth.Text = "Suavidade: "..string.format("%.1f",Config.Smooth)
end)

local bSmMais = Instance.new("TextButton")
bSmMais.Parent = AimConteudo
bSmMais.BackgroundColor3 = Color3.fromRGB(255,20,147)
bSmMais.Size = UDim2.new(0.12,0,0,24)
bSmMais.Position = UDim2.new(0.70,0,0,240)
bSmMais.Text = "+"
bSmMais.TextScaled = true
Instance.new("UICorner",bSmMais).CornerRadius = UDim.new(0,6)
bSmMais.MouseButton1Click:Connect(function()
    Config.Smooth = math.min(Config.Smooth+0.1,1)
    lblSmooth.Text = "Suavidade: "..string.format("%.1f",Config.Smooth)
end)
-- @NoobZinx - PARTE 3/4 (ESP + CONFIG + INFO)
local EspConteudo = Instance.new("Frame")
EspConteudo.Parent = Conteudo
EspConteudo.BackgroundTransparency = 1
EspConteudo.Size = UDim2.new(1,0,1,0)
EspConteudo.Visible = false

criarBotaoSwitch(EspConteudo, 10, "Ativar ESP", function() return Config.ESP end, function(v) 
    Config.ESP = v
    if v then ativarESPCompleto() else desativarESPCompleto() end
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

local ConfConteudo = Instance.new("Frame")
ConfConteudo.Parent = Conteudo
ConfConteudo.BackgroundTransparency = 1
ConfConteudo.Size = UDim2.new(1,0,1,0)
ConfConteudo.Visible = false

criarBotaoSwitch(ConfConteudo, 10, "Modo Veloz", function() return Config.Speed end, function(v) Config.Speed = v end)
sep(ConfConteudo,48)
local lblVel = Instance.new("TextLabel")
lblVel.Parent = ConfConteudo
lblVel.BackgroundTransparency = 1
lblVel.Size = UDim2.new(0.40,0,0,24)
lblVel.Position = UDim2.new(0.05,0,0,58)
lblVel.Text = "Velocidade: "..Config.SpeedMul
lblVel.TextColor3 = Color3.fromRGB(255,255,255)
lblVel.TextScaled = true

local bVelMenos = Instance.new("TextButton")
bVelMenos.Parent = ConfConteudo
bVelMenos.BackgroundColor3 = Color3.fromRGB(50,50,50)
bVelMenos.Size = UDim2.new(0.12,0,0,24)
bVelMenos.Position = UDim2.new(0.55,0,0,58)
bVelMenos.Text = "-"
bVelMenos.TextScaled = true
Instance.new("UICorner",bVelMenos).CornerRadius = UDim.new(0,6)
bVelMenos.MouseButton1Click:Connect(function()
    Config.SpeedMul = math.max(Config.SpeedMul-5,16)
    lblVel.Text = "Velocidade: "..Config.SpeedMul
end)

local bVelMais = Instance.new("TextButton")
bVelMais.Parent = ConfConteudo
bVelMais.BackgroundColor3 = Color3.fromRGB(255,20,147)
bVelMais.Size = UDim2.new(0.12,0,0,24)
bVelMais.Position = UDim2.new(0.70,0,0,58)
bVelMais.Text = "+"
bVelMais.TextScaled = true
Instance.new("UICorner",bVelMais).CornerRadius = UDim.new(0,6)
bVelMais.MouseButton1Click:Connect(function()
    Config.SpeedMul = math.min(Config.SpeedMul+5,200)
    lblVel.Text = "Velocidade: "..Config.SpeedMul
end)
sep(ConfConteudo,92)
criarBotaoSwitch(ConfConteudo, 102, "Super Pulo", function() return Config.SuperJump end, function(v) Config.SuperJump = v end)
sep(ConfConteudo,136)
criarBotaoSwitch(ConfConteudo, 146, "Atravessar Paredes", function() return Config.AtravessarParede end, function(v) Config.AtravessarParede = v end)
sep(ConfConteudo,180)
criarBotaoSwitch(ConfConteudo, 190, "Cores Arco-Íris", function() return Config.Rainbow end, function(v) Config.Rainbow = v end)
sep(ConfConteudo,222)
criarBotaoSwitch(ConfConteudo, 232, "🎯 Hitbox Ampliada", function() return Config.HitboxAmpliada end, function(v) 
    Config.HitboxAmpliada = v 
    if v then ativarHitbox() else desativarHitbox() end
end)
sep(ConfConteudo,266)
criarBotaoSwitch(ConfConteudo, 276, "🚀 FPS BOOSTER", function() return Config.FPSBooster end, function(v) 
    Config.FPSBooster = v 
    if v then ativarFPSBooster() else desativarFPSBooster() end
end)
sep(ConfConteudo,310)
criarBotaoSwitch(ConfConteudo, 320, "💀 Puxar Players", function() return Config.KillAll end, function(v) Config.KillAll = v end)

local InfoConteudo = Instance.new("Frame")
InfoConteudo.Parent = Conteudo
InfoConteudo.BackgroundTransparency = 1
InfoConteudo.Size = UDim2.new(1,0,1,0)
InfoConteudo.Visible = false

local avisoTopo = Instance.new("TextLabel")
avisoTopo.Parent = InfoConteudo
avisoTopo.BackgroundTransparency = 1
avisoTopo.Size = UDim2.new(0.9,0,0,30)
avisoTopo.Position = UDim2.new(0.05,0,0,10)
avisoTopo.Text = "⚠️ USO POR CONTA E RISCO"
avisoTopo.TextColor3 = Color3.fromRGB(255,60,60)
avisoTopo.Font = Enum.Font.GothamBold
avisoTopo.TextScaled = true

lblFPS = Instance.new("TextLabel")
lblFPS.Parent = InfoConteudo
lblFPS.BackgroundTransparency = 1
lblFPS.Size = UDim2.new(0.8,0,0,26)
lblFPS.Position = UDim2.new(0.1,0,0,52)
lblFPS.Text = "FPS: --"
lblFPS.TextColor3 = Color3.fromRGB(80,255,80)
lblFPS.TextScaled = true

local statusLabel = Instance.new("TextLabel")
statusLabel.Parent = InfoConteudo
statusLabel.BackgroundTransparency = 1
statusLabel.Size = UDim2.new(0.8,0,0,28)
statusLabel.Position = UDim2.new(0.1,0,0,85)
statusLabel.Text = "Status: Online"
statusLabel.TextColor3 = Color3.fromRGB(0, 255, 0)
statusLabel.TextScaled = true
statusLabel.Font = Enum.Font.GothamBold

local relogio = Instance.new("TextLabel")
relogio.Parent = InfoConteudo
relogio.BackgroundTransparency = 1
relogio.Size = UDim2.new(0.8,0,0,30)
relogio.Position = UDim2.new(0.1,0,0,120)
relogio.Text = "🕐 00:00:00"
relogio.TextColor3 = Color3.fromRGB(255,255,255)
relogio.TextScaled = true
relogio.Font = Enum.Font.GothamBold

coroutine.wrap(function()
    while task.wait(1) do
        local t = os.date("*t")
        relogio.Text = "🕐 "..string.format("%02d:%02d:%02d", t.hour, t.min, t.sec)
    end
end)()

local dados = {{"Script:","NoobZinx"},{"Desenvolvedor:","@NoobZinx"},{"Data:",os.date("%d/%m/%Y")},{"Status:","Operacional"}}
for i,item in ipairs(dados) do
    local y = 160 + ((i-1)*32)
    local tEsq = Instance.new("TextLabel")
    tEsq.Parent = InfoConteudo
    tEsq.BackgroundTransparency = 1
    tEsq.Size = UDim2.new(0.35,0,0,26)
    tEsq.Position = UDim2.new(0.08,0,0,y)
    tEsq.Text = item[1]
    tEsq.TextColor3 = Color3.fromRGB(255,20,147)
    tEsq.TextScaled = true
    tEsq.TextXAlignment = Enum.TextXAlignment.Right

    local tDir = Instance.new("TextLabel")
    tDir.Parent = InfoConteudo
    tDir.BackgroundTransparency = 1
    tDir.Size = UDim2.new(0.45,0,0,26)
    tDir.Position = UDim2.new(0.50,0,0,y)
    tDir.Text = item[2]
    tDir.TextColor3 = Color3.fromRGB(255,255,255)
    tDir.TextScaled = true
end

local function mudarAba(btnAtivo, painelVisivel)
    bAim.BackgroundColor3 = Color3.fromRGB(35,35,35)
    bEsp.BackgroundColor3 = Color3.fromRGB(35,35,35)
    bConf.BackgroundColor3 = Color3.fromRGB(35,35,35)
    bInfo.BackgroundColor3 = Color3.fromRGB(35,35,35)
    btnAtivo.BackgroundColor3 = Color3.fromRGB(255,20,147)
    AimConteudo.Visible = false
    EspConteudo.Visible = false
    ConfConteudo.Visible = false
    InfoConteudo.Visible = false
    painelVisivel.Visible = true
end

bAim.MouseButton1Click:Connect(function() mudarAba(bAim,AimConteudo) end)
bEsp.MouseButton1Click:Connect(function() mudarAba(bEsp,EspConteudo) end)
bConf.MouseButton1Click:Connect(function() mudarAba(bConf,ConfConteudo) end)
bInfo.MouseButton1Click:Connect(function() mudarAba(bInfo,InfoConteudo) end)
-- @NoobZinx - PARTE 4/4 (FUNÇÕES + ESP)
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
-- 🔥 ESP
-- ═══════════════════════════════════════════════════════
local desenhosESP = {}
local tempoCor = 0
local espConnection = nil
local heartBeatConnection = nil
local espAtivo = false

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
                    desenhosESP[p].bgVida.Filled = true
                    desenhosESP[p].bgVida.Color = Color3.fromRGB(30,30,30)
                    desenhosESP[p].barraVida.Filled = true
                    desenhosESP[p].barraVida.Color = Color3.fromRGB(0,210,0)
                    desenhosESP[p].name.Color = Color3.fromRGB(255,20,147)
                    desenhosESP[p].linha.Thickness = 2
                    desenhosESP[p].dist.Color = Color3.fromRGB(255,255,255)
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
            
            local cor = Config.Rainbow and pegarCorRainbow() or Color3.fromRGB(255,20,147)
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
-- 🔥 AIMBOT
-- ═══════════════════════════════════════════════════════
local circuloFOV = Drawing.new("Circle")
circuloFOV.Thickness = 4
circuloFOV.Color = Color3.fromRGB(255, 20, 147)

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
    circuloFOV.Position = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    circuloFOV.Radius = Config.FOV
    circuloFOV.Visible = Config.MostrarFOV
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

end -- FECHA IniciarMenu

print("✅ @NoobZinx - PAINEL PRINCIPAL CARREGADO!")

-- NEVER web client. Controls live on the website; no in-game panel.
  local WEB_URL = "https://nevertown-production.up.railway.app"
  -- KEY belongs to your account and must match your web sign-in key.
  local KEY = "523b76f28d4a4175507eae05251cdb64013b0f97af42bb42d8e6b7d707dc1c7c"
  local Players = game:GetService("Players")
  local HttpService = game:GetService("HttpService")
  local player = Players.LocalPlayer
  assert(player, "Run Never on the client")
  local requestFn = (syn and syn.request) or (http and http.request) or http_request or request
  assert(type(requestFn) == "function", "Executor HTTP request API is required")

  if _G.NeverWebRuntime and _G.NeverWebRuntime.stop then _G.NeverWebRuntime.stop() end
  local playerGui = player:WaitForChild("PlayerGui")
  local legacy = playerGui:FindFirstChild("NeverAngleUI")
  if legacy then legacy:Destroy() end
  local legacyHook = _G.NeverAngleCodeHook
  if type(legacyHook) == "table" then legacyHook.gui = nil end
  local runtime = {alive = true, connections = {}, workers = {}, targets = {}, connected = false}
  _G.NeverWebRuntime = runtime
  local config = {angle = 59, headEnabled = true, auraEnabled = false, radius = 20,
      barrierVisible = true, transparency = 0.9, interval = 0.2, debug = false}
  local status = {found = 0, inRange = 0, message = "กำลังเชื่อมต่อ", hookAvailable = false}
  local lastSync = 0
  local deviceId = HttpService:GenerateGUID(false)
  local barrier = Instance.new("SphereHandleAdornment")
  barrier.Name = "NeverWebBarrier"
  barrier.Color3 = Color3.fromRGB(180, 240, 220)
  barrier.Transparency = config.transparency
  barrier.Radius = config.radius
  barrier.AlwaysOnTop = false
  barrier.Visible = false
  barrier.Parent = playerGui

  local function connect(signal, callback)
      local connection = signal:Connect(callback)
      table.insert(runtime.connections, connection)
  end
  local function ownCharacter()
      local character = player.Character
      if character and character:IsDescendantOf(workspace) then return character end
      local named = workspace:FindFirstChild(player.Name)
      if named and named:IsA("Model") then return named end
      return nil
  end
  local function rootOf(model)
      if not model or not model:IsA("Model") then return nil end
      local root = model:FindFirstChild("HumanoidRootPart") or model.PrimaryPart
          or model:FindFirstChild("Torso") or model:FindFirstChild("UpperTorso") or model:FindFirstChild("Head")
      if root and root:IsA("BasePart") then return root end
      return nil
  end
  local function ownerOf(model)
      local owner = Players:GetPlayerFromCharacter(model)
      if owner then return owner end
      local named = Players:FindFirstChild(model.Name)
      if named and named:IsA("Player") then return named end
      return nil
  end
  local function addTarget(instance)
      if instance:IsA("Model") then runtime.targets[instance] = true end
  end
  connect(workspace.DescendantAdded, addTarget)
  connect(workspace.DescendantRemoving, function(instance) runtime.targets[instance] = nil end)
  for _, instance in ipairs(workspace:GetDescendants()) do addTarget(instance) end

  local function refreshBarrier()
      local character = ownCharacter()
      local root = rootOf(character)
      local humanoid = character and character:FindFirstChildOfClass("Humanoid")
      barrier.Adornee = root
      if barrier.Radius ~= config.radius then barrier.Radius = config.radius end
      if barrier.Transparency ~= config.transparency then barrier.Transparency = config.transparency end
      local visible = runtime.connected and config.auraEnabled and config.barrierVisible
          and config.radius > 0 and root ~= nil and (not humanoid or humanoid.Health > 0)
      if barrier.Visible ~= visible then barrier.Visible = visible end
  end

  local function resolveRoot(target)
      if typeof(target) == "Instance" then
          if target:IsA("Player") then return rootOf(target.Character) end
          if target:IsA("Model") then return rootOf(target) end
          if target:IsA("BasePart") then return rootOf(target:FindFirstAncestorOfClass("Model")) or target end
      elseif type(target) == "string" then
          local named = workspace:FindFirstChild(target)
          if named and named:IsA("Model") then return rootOf(named) end
      end
  end
  local hitParts = {Head = true, HumanoidRootPart = true, Torso = true, UpperTorso = true, LowerTorso = true,
      LeftUpperArm = true, LeftLowerArm = true, LeftHand = true, RightUpperArm = true, RightLowerArm = true, RightHand = true,
      LeftUpperLeg = true, LeftLowerLeg = true, LeftFoot = true, RightUpperLeg = true, RightLowerLeg = true, RightFoot = true,
      ["Left Arm"] = true, ["Right Arm"] = true, ["Left Leg"] = true, ["Right Leg"] = true}
  local function shouldUseHead(target, part)
      if not runtime.connected or not config.headEnabled or config.angle * 2 <= 10 then return false end
      if type(part) ~= "string" or not hitParts[part] then return false end
      local targetRoot = resolveRoot(target)
      local ownRoot = rootOf(ownCharacter())
      if not targetRoot or not ownRoot or targetRoot == ownRoot then return false end
      local delta = ownRoot.Position - targetRoot.Position
      local look = targetRoot.CFrame.LookVector
      local dm = math.sqrt(delta.X * delta.X + delta.Z * delta.Z)
      local fm = math.sqrt(look.X * look.X + look.Z * look.Z)
      if dm < 0.001 or fm < 0.001 then return false end
      local dx, dz, fx, fz = delta.X / dm, delta.Z / dm, look.X / fm, look.Z / fm
      local angle = math.deg(math.atan2(dx * -fz + dz * fx, dx * fx + dz * fz))
      return math.abs(angle) <= config.angle + 0.0001
  end

  local hookState
  if type(hookmetamethod) == "function" and type(newcclosure) == "function"
      and type(getnamecallmethod) == "function" and type(setnamecallmethod) == "function" then
      hookState = _G.NeverWebHook
      if type(hookState) ~= "table" then hookState = {}; _G.NeverWebHook = hookState end
      hookState.runtime = runtime
      hookState.shouldUseHead = shouldUseHead
      hookState.config = config
      hookState.auraSending = false
      if not hookState.installed then
          local state = hookState
          local old
          old = hookmetamethod(game, "__namecall", newcclosure(function(self, ...)
              local method = getnamecallmethod()
              if state.runtime and state.runtime.alive and method == "FireServer" and not state.auraSending then
                  local target, part = ...
                  local ok, useHead = false, false
                  if target ~= "Use" then ok, useHead = pcall(state.shouldUseHead, target, part) end
                  if ok and useHead then
                      local Args = table.pack(...)
                      Args[2] = "Head"
                      if state.config.debug then print("Arg 1:", Args[1]); print("Arg 2:", Args[2]) end
                      setnamecallmethod(method)
                      return old(self, table.unpack(Args, 1, Args.n))
                  end
                  setnamecallmethod(method)
              end
              return old(self, ...)
          end))
          hookState.installed = true
      end
      status.hookAvailable = true
  end

  local function sync()
      local response = requestFn({Url = WEB_URL .. "/api/device/sync", Method = "POST",
          Headers = {["Content-Type"] = "application/json", Authorization = "Bearer " .. KEY},
          Body = HttpService:JSONEncode({deviceId = deviceId, player = player.Name, status = status})})
      if response and (response.StatusCode == 401 or response.StatusCode == 403) then
          runtime.keyRejected = true
          error("Account key rejected", 0)
      end
      assert(response and response.StatusCode == 200, "Web connection failed")
      local body = HttpService:JSONDecode(response.Body)
      local nextConfig = body.config
      assert(type(nextConfig) == "table", "Invalid config")
      for key, value in pairs(nextConfig) do
          if typeof(value) == typeof(config[key]) then config[key] = value end
      end
      config.angle = math.clamp(math.floor(config.angle), 0, 180)
      config.radius = math.clamp(math.floor(config.radius), 0, 5000)
      config.interval = math.clamp(config.interval, 0.2, 2)
      config.transparency = math.clamp(config.transparency, 0.5, 0.98)
      _G.Left, _G.right = config.angle, config.angle
      _G.Kuy = config.headEnabled
      lastSync = os.clock()
      runtime.connected = true
      refreshBarrier()
  end

  local function auraTick()
      refreshBarrier()
      if not runtime.connected or not config.auraEnabled then status.message = "ปิด Kill Aura"; return end
      if config.radius == 0 then status.message = "ระยะ 0"; status.inRange = 0; return end
      local character = ownCharacter()
      local ownRoot = rootOf(character)
      local ownHumanoid = character and character:FindFirstChildOfClass("Humanoid")
      if not ownRoot or (ownHumanoid and ownHumanoid.Health <= 0) then status.message = "รอตัวละคร"; return end
      local bat = character:FindFirstChild("BaseballBat")
      local batScript = bat and bat:FindFirstChild("LocalScript")
      local Event = batScript and batScript:FindFirstChild("Damage")
      if not Event or not Event:IsA("RemoteEvent") then status.message = "ถือ BaseballBat"; return end
      local origin, radiusSquared = ownRoot.Position, config.radius * config.radius
      local batch, owners, found = {}, {}, {}
      for target in pairs(runtime.targets) do
          if target:IsDescendantOf(workspace) then
              local owner = ownerOf(target)
              if owner and owner ~= player and target ~= character then
                  local root = rootOf(target)
                  local head = target:FindFirstChild("Head")
                  local humanoid = target:FindFirstChildOfClass("Humanoid")
                  if root and head and head:IsA("BasePart") then
                      found[owner] = true
                      local delta = root.Position - origin
                      local distance = delta.X * delta.X + delta.Y * delta.Y + delta.Z * delta.Z
                      if (not humanoid or humanoid.Health > 0) and distance <= radiusSquared and not owners[owner] then
                          owners[owner] = true
                          table.insert(batch, {target = target, root = root, humanoid = humanoid, owner = owner})
                      end
                  end
              end
          else runtime.targets[target] = nil end
      end
      local count = 0
      for _ in pairs(found) do count = count + 1 end
      status.found, status.inRange, status.message = count, #batch, "กำลังทำงาน"
      for _, entry in ipairs(batch) do
          task.defer(function(target, root, humanoid, owner)
              if not runtime.alive or not runtime.connected or not config.auraEnabled or config.radius <= 0
                  or ownCharacter() ~= character or not target:IsDescendantOf(workspace)
                  or owner.Parent ~= Players or not Event:IsDescendantOf(character)
                  or (humanoid and humanoid.Health <= 0) then return end
              local delta = root.Position - ownRoot.Position
              if delta.X * delta.X + delta.Y * delta.Y + delta.Z * delta.Z > config.radius * config.radius then return end
              if hookState then hookState.auraSending = true end
              -- Adapted from the supplied Cobalt call: one Head request for each selected player.
              local ok, message = pcall(function() Event:FireServer(target, "Head") end)
              if hookState then hookState.auraSending = false end
              if not ok then status.message = "ส่งไม่สำเร็จ"; if config.debug then warn(message) end end
          end, entry.target, entry.root, entry.humanoid, entry.owner)
      end
  end

  function runtime.stop()
      if not runtime.alive then return end
      runtime.alive = false
      runtime.connected = false
      if hookState and hookState.runtime == runtime then hookState.runtime = nil end
      for _, thread in ipairs(runtime.workers) do pcall(task.cancel, thread) end
      for _, connection in ipairs(runtime.connections) do connection:Disconnect() end
      barrier:Destroy()
  end
  table.insert(runtime.workers, task.spawn(function()
      local failures = 0
      while runtime.alive do
          local ok = pcall(sync)
          if ok then failures = 0 else
              failures = failures + 1
              if runtime.keyRejected then
                  runtime.connected = false
                  barrier.Visible = false
                  warn("Never: KEY ไม่ถูกต้องหรือถูกปิด กรุณาตรวจสอบกับผู้ดูแล")
                  task.defer(runtime.stop)
                  break
              end
              status.message = "เชื่อมเว็บไม่สำเร็จ"
              if os.clock() - lastSync > 10 then runtime.connected = false; barrier.Visible = false end
          end
          task.wait(math.min(5, failures + 1))
      end
  end))
  table.insert(runtime.workers, task.spawn(function()
      while task.wait(config.interval) do
          if not runtime.alive then break end
          if os.clock() - lastSync > 10 then runtime.connected = false; barrier.Visible = false end
          if runtime.connected then
              local ok, message = xpcall(auraTick, tostring)
              if hookState then hookState.auraSending = false end
              if not ok then status.message = "เกิดข้อผิดพลาด"; if config.debug then warn(message) end; task.wait(1) end
          end
      end
  end))
  

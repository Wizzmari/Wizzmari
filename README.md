@@ -2,10 +2,15 @@ Config = Config or {}
Config.framework = 'qbcore' --(qbcore/esx/custom)
Config.useItem = true
Config.itemName = 'speaker' --You need to had this item created in your config or database
Config.KeyAccessUi = 38
Config.KeyDeleteSpeaker = 194
Config.KeyToMove = 311
Config.KeyToPlaceSpeaker = 191
Config.KeyToChangeAnim = 311

Config.Translations = {
    notEnoughDistance = 'You should reserve a little more distance from the other nearby speaker.',
    helpNotify = 'Press ~INPUT_CONTEXT~ to access into the speaker or ~INPUT_FRONTEND_RRIGHT~ to delete it',
    helpNotify = 'Press ~INPUT_CONTEXT~ to access into the speaker, ~INPUT_FRONTEND_RRIGHT~ to delete it or ~INPUT_REPLAY_SHOWHOTKEY~ to hold boombox',
    libraryLabel = 'Your library',
    newPlaylistLabel = 'Create new Playlist',
    importPlaylistLabel = 'Import a existing Playlist',
@@ -15,5 +20,6 @@ Config.Translations = {
    deletePlaylist = 'Delete Playlist',
    unkown = 'Unkown',
    titleFirstMessage = "Don't have a playlist yet?",
    secondFirstMessage = "Create a Playlist"
    secondFirstMessage = "Create a Playlist",
    holdingBoombox = 'Press ~INPUT_FRONTEND_RDOWN~ to place the speaker or ~INPUT_REPLAY_SHOWHOTKEY~ to change anim'
}
  132 changes: 126 additions & 6 deletions132  
client/client.lua
@@ -2,6 +2,10 @@ local speakers = {}
local isInUI = false
local playlistsLoaded = false
local reproInUi = -1
local movingASpeaker = false
local movingObject = 0
local movingSpeakerId = -1
local gangAnim = true

function SendReactMessage(action, data)
    SendNUIMessage({
@@ -28,13 +32,16 @@ function LoadSpeakers()
    TriggerCallback('gacha_boombox:callback:getBoomboxs', function(result)
        for k,v in pairs(result) do
            SendReactMessage('createRepro', '')
            Wait(100)
        end
        Wait(200)
        speakers = result
    end)
end

RegisterNUICallback('webLoaded', function()
    print('Loading speakers')
    Wait(100)
    LoadSpeakers()
end)

@@ -87,6 +94,19 @@ RegisterNetEvent('gacha_boombox:client:updateDist', function(id, dist)
    speakers[id].maxDistance = dist
end)

RegisterNetEvent('gacha_boombox:client:doAnim', function()
    if not HasAnimDictLoaded('anim@heists@money_grab@briefcase') then
        RequestAnimDict('anim@heists@money_grab@briefcase')

        while not HasAnimDictLoaded('anim@heists@money_grab@briefcase') do
            Citizen.Wait(1)
        end
    end
    TaskPlayAnim(PlayerPedId(), 'anim@heists@money_grab@briefcase', 'put_down_case', 8.0, -8.0, -1, 1, 0, false, false, false)
    Citizen.Wait(1000)
    ClearPedTasks(PlayerPedId())
end)

RegisterNetEvent('gacha_boombox:client:insertSpeaker', function(data)
    table.insert(speakers, data)
    SendReactMessage('createRepro', data.url)
@@ -120,6 +140,16 @@ RegisterNUICallback('syncNewTime', function(data)
    end
end)

function LoadAnima()
    if not HasAnimDictLoaded('missfinale_c2mcs_1') then
        RequestAnimDict('missfinale_c2mcs_1')

        while not HasAnimDictLoaded('missfinale_c2mcs_1') do
            Citizen.Wait(1)
        end
    end
end

Citizen.CreateThread(function()
    while true do
        local sleep = 500
@@ -141,42 +171,132 @@ Citizen.CreateThread(function()
                else
                    SendReactMessage('changeVolume', {repro = tonumber(k - 1), volume = 0})
                end
                DrawMarker(20, v.coords.x, v.coords.y, v.coords.z, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.3, 0.3, 0.3, 0, 0, 0, 255, false, false, false, true, false, false, false)
                if distance < 1.5 then
                    ShowHelpNotification(Config.Translations.helpNotify)
                    if IsControlJustPressed(1, 38) then
                if distance < 1.5 and not v.isMoving then
                    if not movingASpeaker then
                        ShowHelpNotification(Config.Translations.helpNotify)
                    end
                    if IsControlJustPressed(1, Config.KeyAccessUi)  and not movingASpeaker then
                        SendReactMessage('setRepro', tonumber(k - 1))
                        if not playlistsLoaded then
                            SendReactMessage('getPlaylists')
                        end
                        if v.playlistPLaying.songs and v.playlistPLaying.songs[v.songId] then
                            SendReactMessage('sendSongInfo', {author= v.playlistPLaying.songs[v.songId].author, name = v.playlistPLaying.songs[v.songId].name, url = v.url, volume = v.volume, dist = v.maxDistance, maxDuration = v.maxDuration, paused = v.paused, pausedTime = v.pausedTime})
                            SendReactMessage('sendSongInfo', {author = v.playlistPLaying.songs[v.songId].author, name = v.playlistPLaying.songs[v.songId].name, url = v.url, volume = v.volume, dist = v.maxDistance, maxDuration = v.maxDuration, paused = v.paused, pausedTime = v.pausedTime})
                        end
                        toggleNuiFrame(true)
                        reproInUi = k - 1
                    end
                    if IsControlJustPressed(0, 194) and not isInUI then
                    if IsControlJustPressed(1, Config.KeyDeleteSpeaker) and not isInUI and not movingASpeaker then
                        TriggerServerEvent('gacha_boombox:server:deleteBoombox', k, v.coords.x)
                        Wait(200)
                    end
                    if IsControlJustPressed(1, Config.KeyToMove) and not movingASpeaker then
                        TriggerCallback('gacha_boombox:callback:canMove', function(canMove)
                            if canMove then
                                movingASpeaker = true
                                local ped = PlayerPedId()
                                local x, y, z = table.unpack(GetOffsetFromEntityInWorldCoords(ped, 0.0, 3.0, 0.5))
                                local obj = CreateObjectNoOffset('prop_boombox_01', x, y, z, true, false)
                                SetModelAsNoLongerNeeded('prop_boombox_01')
                                movingObject = obj
                                movingSpeakerId = k
                                if gangAnim then
                                    LoadAnima()
                                    TaskPlayAnim(PlayerPedId(), 'missfinale_c2mcs_1', 'fin_c2_mcs_1_camman', 8.0, -8.0, -1, 51, 0, false, false, false)
                                    AttachEntityToEntity(obj, ped, GetPedBoneIndex(ped, 28422),0.0, 0.0, 0.1, 0.0, 0.0, 0.0, true, true, false, true, 1, true)
                                else
                                    AttachEntityToEntity(obj, ped, GetPedBoneIndex(ped, 57005), 0.32, 0, -0.05, 0.10, 270.0, 60.0, true, true, false, true, 1, true)
                                end
                            end
                        end, k)
                    end
                end
            else
                if v.isPlaying then
                    SendReactMessage('stopSong', tonumber(k - 1))
                    v.isPlaying = false
                end
            end
            if v.isMoving then
                if v.playerMoving >= 1 then
                    local playerIdx = GetPlayerFromServerId(v.playerMoving)
                    local ped = GetPlayerPed(playerIdx)
                    local coords = GetEntityCoords(ped)
                    if coords == GetEntityCoords(PlayerPedId()) then
                        if movingASpeaker and k == movingSpeakerId then
                            v.coords = coords
                        end
                    else
                        v.coords = coords
                    end
                end
            end
        end

        if movingASpeaker then
            sleep = 5
            ShowHelpNotification(Config.Translations.holdingBoombox)
            if IsControlJustPressed(1, Config.KeyToPlaceSpeaker) then
                if not HasAnimDictLoaded('anim@heists@money_grab@briefcase') then
                    RequestAnimDict('anim@heists@money_grab@briefcase')

                    while not HasAnimDictLoaded('anim@heists@money_grab@briefcase') do
                        Citizen.Wait(1)
                    end
                end
                TaskPlayAnim(PlayerPedId(), 'anim@heists@money_grab@briefcase', 'put_down_case', 8.0, -8.0, -1, 1, 0, false, false, false)
                Citizen.Wait(1000)
                ClearPedTasks(PlayerPedId())
                DeleteEntity(movingObject)
                TriggerServerEvent('gacha_boombox:server:updateObjectCoords', movingSpeakerId)
                movingSpeakerId = 0
                movingASpeaker = false
            end
            if IsControlJustPressed(1, Config.KeyToChangeAnim) then
                gangAnim = not gangAnim
                if gangAnim then
                    LoadAnima()
                    AttachEntityToEntity(movingObject, PlayerPedId(), GetPedBoneIndex(PlayerPedId(), 28422),0.0, 0.0, 0.1, 0.0, 0.0, 0.0, true, true, false, true, 1, true)
                    TaskPlayAnim(PlayerPedId(), 'missfinale_c2mcs_1', 'fin_c2_mcs_1_camman', 8.0, -8.0, -1, 51, 0, false, false, false)
                else
                    ClearPedTasks(PlayerPedId())
                    AttachEntityToEntity(movingObject, PlayerPedId(), GetPedBoneIndex(PlayerPedId(), 57005), 0.32, 0, -0.05, 0.10, 270.0, 60.0, true, true, false, true, 1, true)
                end
            end
        end

        Wait(sleep)
    end
end)

RegisterNetEvent('gacha_boombox:client:updatePlayerMoving', function(id, src)
    speakers[id].isMoving = true
    speakers[id].playerMoving = src
end)

AddEventHandler('onResourceStop', function(resourceName)
    if (GetCurrentResourceName() ~= resourceName) then
        return
    end
    DeleteEntity(movingObject)
end)

RegisterNUICallback('getPlaylists', function(_, cb)
    TriggerCallback('gacha_boombox:callback:getPlaylists', function(result)
        cb(result)
    end)
end)

RegisterNetEvent('gacha_boombox:client:syncLastCoords', function(id, coords)
    speakers[id].coords = coords
    speakers[id].isMoving = false
    speakers[id].playerMoving = -2
end)

RegisterNetEvent('gacha_boombox:client:syncLastCoordsSync', function(id, coords)
    speakers[id].coords = coords
end)

RegisterNUICallback('addSong', function(data)
    TriggerServerEvent('gacha_boombox:server:addSong', data)
end)
  54 changes: 52 additions & 2 deletions54  
server/server.lua
@@ -1,4 +1,5 @@
Speakers = {}
local objects = {}

function SufficientDistance(coords)
    local minDistance = true
@@ -54,6 +55,8 @@ RegisterNetEvent('gacha_boombox:server:deleteBoombox', function(id, x)
        Speakers[id].permaDisabled = true
        Speakers[id].playlistPLaying = {}
        Speakers[id].url = ''
        DeleteEntity(objects[id])
        objects[id] = -1
        TriggerClientEvent('gacha_boombox:client:deleteBoombox', -1, id)
        if Config.useItem then
            AddItem(src)
@@ -91,6 +94,13 @@ Citizen.CreateThread(function()
                        v.songId = v.songId + 1
                        v.maxDuration = v.playlistPLaying.songs[v.songId].maxDuration
                        TriggerClientEvent('gacha_boombox:client:updateBoombox', -1, k, v)
                    else
                        v.url = v.playlistPLaying.songs[1].url
                        v.time = os.time() * 1000
                        v.isPlaying = false
                        v.songId = 1
                        v.maxDuration = v.playlistPLaying.songs[1].maxDuration
                        TriggerClientEvent('gacha_boombox:client:updateBoombox', -1, k, v)
                    end
                end
            end
@@ -289,13 +299,53 @@ end)
function CreateSpeaker(src)
    local enoughDistance = SufficientDistance(GetEntityCoords(GetPlayerPed(src)))
    if enoughDistance then
        local data = {volume = 50, url = '', coords = GetEntityCoords(GetPlayerPed(src)), playlistPLaying = {}, time = 0, maxDistance = 15, isPlaying = false, maxDuration = 5000000, songId = -2, permaDisabled = false, paused = false, pausedTime = 0}
        local data = {volume = 50, url = '', coords = GetEntityCoords(GetPlayerPed(src)), playlistPLaying = {}, time = 0, maxDistance = 15, isPlaying = false, maxDuration = 5000000, songId = -2, permaDisabled = false, paused = false, pausedTime = 0, isMoving = false, playerMoving = -2}
        table.insert(Speakers, data)
        if Config.useItem then
            DeleteItem(src)
        end
        TriggerClientEvent('gacha_boombox:client:doAnim', src)
        Citizen.Wait(1000)
        local obj = CreateObject('prop_boombox_01', data.coords - vector3(0.0, 0.0, 1.0), true, false, true)
        table.insert(objects, obj)
        TriggerClientEvent('gacha_boombox:client:insertSpeaker', -1, data)
    else
        TriggerClientEvent('gacha_boombox:client:notify', src, Config.Translations.notEnoughDistance)
    end
end
end

AddEventHandler('onResourceStop', function(resourceName)
    if (GetCurrentResourceName() ~= resourceName) then
        return
    end
    for k,v in pairs(objects) do
        if v ~= -1 then
            DeleteEntity(v)
        end
    end
end)

CreateCallback("gacha_boombox:callback:canMove", function(source, cb, id)
    local src = source
    if not Speakers[id].isMoving then
        Speakers[id].isMoving = true
        Speakers[id].playerMoving = src
        DeleteEntity(objects[id])
        objects[id] = -1
        TriggerClientEvent('gacha_boombox:client:updatePlayerMoving', -1, id, src)
    end
    cb(Speakers[id].isMoving)
end)

RegisterNetEvent('gacha_boombox:server:updateObjectCoords', function(id)
    local src = source
    if Speakers[id].isMoving and Speakers[id].playerMoving == src then
        local coords = GetEntityCoords(GetPlayerPed(src))
        local obj = CreateObject('prop_boombox_01', coords - vector3(0.0, 0.0, 1.0), true, false, true)
        objects[id] = obj
        Speakers[id].isMoving = false
        Speakers[id].coords = coords
        Speakers[id].playerMoving = -1
        TriggerClientEvent('gacha_boombox:client:syncLastCoords', -1, id, coords)
    end
end)- 
--->

---
title: OC(1)- 让无人机和机器人聪明一点
published: 2026-01-29
description: '用开放式电脑征服世界'
image: 'covers/opencomputerslua.png'
tags: [Minecraft, Lua]
category: 'Minecraft'
draft: false 
lang: 'zh_CN'
---

# 初步介绍
OpenComputers是Minecraft 1.12.2最大的可编程化MOD，这个Mod添加了电脑，机器人，无人机，服务器，单片机等概念，它们的运行逻辑就是Lua，因此不掌握Lua的编程技巧和OC的各种API，那这个Mod就白装了。

如果没有Lua编程技术，也没关系，OC的API才是核心，我们还可以向AI求助。

这是OC的官方API文档 [[点我访问](https://ocdoc.cil.li/ "点我访问")]。

本文讲机器人和无人机的自动化逻辑，而且会不定扩充内容。

冷知识：鼠标中键可以粘贴文本
# 节点一——机器人
机器人说白了就是一个可移动大型电脑，最大的作用就是执行Lua脚本自动干活（自动采集成熟的小麦，甘蔗等作物），同时还要确定好自己需要什么类型的升级，如果是收集成熟的小麦，需要安装地质探测仪；如果是为了简单的路径巡航，需要安装导航升级；如果需要续航，可以安装电池升级和太阳能发电升级...
## 举个例子
我习惯给机器人安装（3x4）：
|  T3电池升级 | 导航升级  |  太阳能发电升级 |
| ------------ | ------------ | ------------ |
| 物品栏升级  |  空 | 空  |
| 键盘  | 地质探测仪  |  T1显示器 |
| 软盘驱动器  |  T2扩展卡升级 |  T2扩展升级容器 |

CPU为T2 APU，
内存为两个T1.5内存，
扩展卡为T2无线网卡，英特网卡，
以上配置组装大概需要6分钟

## 跟随玩家
这个方面对我而言确实很难，但是在AI的帮助下确实硬生生地整出来了一个相对完整的代码：
```lua
local component = require("component")
local nav = component.navigation
local event = require("event")
local shell = require("shell")
local robot = require("robot")
local modem = component.modem
modem.open(111)
local facingMap = { [2] = 0, [5] = 1, [3] = 2, [4] = 3 } 
local facing = facingMap[nav.getFacing()] or 0 
-- 朝向定义：0=北,1=东,2=南,3=西
-- 转向函数
local function turnTo(dir)
    local diff = (dir - facing) % 4
    if diff == 1 then
        robot.turnRight()
    elseif diff == 2 then
        robot.turnRight(); robot.turnRight()
    elseif diff == 3 then
        robot.turnLeft()
    end
    facing = dir
end
-- 尝试前进，遇到障碍时尝试绕过
local function safeForward()
    if not robot.detect() then
        return robot.forward()
    else
        -- 尝试左绕
        robot.turnLeft()
        if not robot.detect() and robot.forward() then
            robot.turnRight()
            return true
        else
            -- 左绕失败，恢复方向
            robot.turnRight()
            -- 尝试右绕
            robot.turnRight()
            if not robot.detect() and robot.forward() then
                robot.turnLeft()
                return true
            else
                -- 右绕也失败，恢复方向
                robot.turnLeft()
                return false
            end
        end
    end
end
-- 前往指定坐标
local function goTo(tx, ty, tz)
    local x, y, z = nav.getPosition()
    local dx, dy, dz = tx - math.floor(x), ty - math.floor(y), tz - math.floor(z)
    -- 垂直移动
    if dy > 0 then
        for i=1,dy do
            while not robot.up() do robot.swingUp() end
        end
    elseif dy < 0 then
        for i=1,-dy do
            while not robot.down() do robot.swingDown() end
        end
    end
    -- X方向移动
    if dx > 0 then
        turnTo(1) -- 东
        for i=1,dx do
            while not safeForward() do
              os.sleep(0.5) -- 等待一会再尝试
            end
        end
    elseif dx < 0 then
        turnTo(3) -- 西
        for i=1,-dx do
            while not safeForward() do
              os.sleep(0.5) -- 等待一会再尝试
            end
        end
    end
    -- Z方向移动
    if dz > 0 then
        turnTo(2) -- 南
        for i=1,dz do
            while not safeForward() do
              os.sleep(0.5) -- 等待一会再尝试
            end
        end
    elseif dz < 0 then
        turnTo(0) -- 北
        for i=1,-dz do
            while not safeForward() do
              os.sleep(0.5) -- 等待一会再尝试
            end
        end
    end
end
while true do
    local _, _, _, _, _, msg = event.pull("modem_message")
    local tx, ty, tz = msg:match("(-?%d+),%s*(-?%d+),%s*(-?%d+)")
    if tx and ty and tz then
        goTo(tonumber(tx), tonumber(ty), tonumber(tz))
        break
    end
end
```
这个代码的原理是，持续监控来自控制端的坐标指令，其中("(-?%d+),%s*(-?%d+),%s*(-?%d+)")是接收端解释数据的方式，因此控制端的发送格式就得是(x, y, z) (-?%d+代表一个整数，%s代表一个空格)，而且可以在需要时转向，躲避/清除障碍。

因为在被控制端中，开放了一个端口为111，控制端可以向111端口广播自己的位置或者任意坐标。

任意位置：
`component.modem.broadcast(111, "x, y, z")`
自己的位置：
```lua
x, y, z=component.navigation.getPosition()
tx, ty, tz=math.floor(x), math.floor(y), math.floor(z)
component.modem.broadcast(111, tx ... ", " ... ty ... ", " ... tz)
```
### 注意事项
控制端和被控制端最好安装同一个导航升级（只需要注意做两个一模一样的定位器地图就行了），这样一个能确定自己的位置，一个能确定自己需要去的位置，而且相对坐标一致（导航升级的坐标指数和地图相关，**和玩家的坐标没有关系**）
# 节点二——无人机
无人机就更冷门了，我目前只研究出了持续跟随玩家的功能，考虑到无人机比较节能，我就安装了这些东西：
| 主选  | 导航升级  |  物品栏升级 |  物品栏升级 |
| ------------ | ------------ | ------------ | ------------ |
| 备选  | 导航升级  |  太阳能发电升级 | 物品栏升级  |

CPU：T1 CPU，
内存：T1.5 内存（T1内存就够了），
扩展卡：T2 无线网卡，
## 持续跟随玩家/上下左右移动
这部分也是最核心的部分，在测试代码之前建议在开一个创造模式世界，不然生存模式出错了拆解重新安装要废很多时间
```lua
local drone = component.proxy(component.list("drone")())
local nav   = component.proxy(component.list("navigation")())
local modem = component.proxy(component.list("modem")()) --类似于Python的import
modem.open(111) --利用无线网卡监听111端口
drone.setStatusText("Listening on port 111")
while true do
    local signal, _, _, port, _, msg = computer.pullSignal() --只监听port端口和msg消息，_代表忽略
    if signal == "modem_message" and port == 111 then
        msg = tostring(msg)
        local tx, ty, tz = msg:match("(-?%d+)%s+(-?%d+)%s+(-?%d+)") --转换接收到的数值
        if tx and ty and tz then
            tx, ty, tz = tonumber(tx), tonumber(ty), tonumber(tz)
            local x, y, z = nav.getPosition() --获取当前位置
            if x then
                local dx, dy, dz = tx - x, ty - y, tz - z --减去绝对位置的值，确定位移所需的值
                drone.move(dx, dy, dz) --开始位移
                drone.setStatusText("Moving") --文本会输出在无人机的小屏幕上
            else
                drone.setStatusText("failed") --算是异常处理了，可以避免无人机因意外关机
            end
        elseif msg == "up" then --这些是上下左右移动指令，从控制端发出
            drone.move(0, 1, 0)
            drone.setStatusText("up")
        elseif msg == "down" then
            drone.move(0, -1, 0)
            drone.setStatusText("down")
        elseif msg == "forward" then
            drone.move(0, 0, -1)
            drone.setStatusText("forward")
        elseif msg == "back" then
            drone.move(0, 0, 1)
            drone.setStatusText("back")
        elseif msg == "left" then
            drone.move(-1, 0, 0)
            drone.setStatusText("left")
        elseif msg == "right" then
            drone.move(1, 0, 0)
            drone.setStatusText("right")
        elseif msg == "shutdown" then --关机指令
            computer.shutdown()
        else
            drone.setStatusText("err: " .. msg) --异常处理
        end
    end
end  --lua没有进位，只是用end代表结束，我之所以进位是为了好看
```
*（从最后一行注释能看出来，其实编程Lua只需要像编程HTML就行了）*原理我在注释都写明白了
## 控制端代码
因为考虑到灵活性，我还特意为控制无人机做了一个控制器软件：
```lua
local component = require("component")
local event = require("event")
local shell = require("shell")
local modem = component.modem
local nav   = component.navigation
local sendPort = 111
print("command (exit):")
while true do
    io.write("> ")
    local cmd = io.read()
    if cmd == "exit" then
        break
    elseif cmd == "return" then
        local x, y, z = nav.getPosition()
        local pos = math.floor(x) .. " " .. math.floor(y) .. " " .. math.floor(z)
        modem.broadcast(sendPort, pos)
        print("Returning " .. pos)
    elseif cmd == "home" then
        modem.broadcast(sendPort, "900 70 -900") --这只是个样例
    elseif cmd == "follow" then
        shell.execute("follow.lua")
    else
        modem.broadcast(sendPort, cmd)
        print("Sent " .. cmd)
    end
end
```
稍微改一改，也能用在机器人身上，因为我设定无人机接收坐标的格式是'x y z'
# 吐槽
~~***不是兄弟，你知道熬夜改代码改到凌晨3点是什么感觉吗？***~~

但是这个教程也会扩建，不能吃灰了，这次我是赶工写的；赶谁的工？；赶自己的工！

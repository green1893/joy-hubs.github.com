---
layout: post
title:  "APISIX 日志插件大响应数据日志乱码处理方案"
date:   2025-04-12 10:06:00 +0800
category: programming
tags: 编程实践 APISIX
---
# 一、软件版本
- APISIX 3.11.0

> 说明：APISIX 3.11.0 版本环境中，Lua版本为 5.1

# 二、问题现象
在 APISIX 网关中，可以给服务（Service）、路由（Route）配置日志插件，日志插件可以将请求（Request）和响应（Response）的原始日志信息记录下来，不同的日志插件可以将日志输出到不同的地方，比如文件、消息队列（Kafka、RocketMQ）、云厂商日志服务等。

默认情况下，APISIX 日志插件不会记录响应体（Response Body），如果需要记录响应体，需要在日志插件中配置 `include_resp_body` 参数为 `true`。

但是在实际使用中，如果调用一个结果比较大的数据接口，可能会遇到日志乱码（包含二进制字节数据）的问题，通过`vim ping.log`可以看到日志因为包含了字节数据而无法正常显示。
![vim-unread-log](/assets/vim_unread_log.png)

通过`cat`命令查看日志文件内容，也可以看到响应体`body`中出现了乱码内容：
```text
"body": "{\"code\":0,\"data\":{\"code\":200,\"msg\":\"请求成功\",\"data\":{\"content\":\"中文内容�\"}}}"
```

**日志插件配置**

一个开启了响应结果日志记录的日志插件配置如下：
```json5
{
  "_meta": {
    "disable": false
  },
  "include_resp_body": true,
  "path": "logs/ping.log"
}
```
> 参考：[apisix file-logger](https://apisix.apache.org/docs/apisix/3.11/plugins/file-logger/) 日志插件参数说明文档

# 三、问题分析
APISIX 中内置插件都是使用了 Lua 进行开发实现，以`file-logger`日志插件为例，通过阅读代码可以发现所有日志插件都是通过`log-util`模块进行请求日志收集。
```lua
-- 引入log-util模块
local log_util     =   require("apisix.utils.log-util")

-- 根据插件配置的参数，收集响应体（response body）部分
function _M.body_filter(conf, ctx)
    log_util.collect_body(conf, ctx)
end

```

进一步阅读`collect_body`函数的实现，可以发现`collect_body`对收集的响应体进行最大长度的限制，默认最大长度为 512KB，如果响应体超过了这个长度，则会被截断。
```lua
-- 默认响应体的最大长度为 512KB
local MAX_RESP_BODY     = 524288

-- 如果开启了响应体日志收集
if log_response_body then
    -- 响应体的最大字节数
    local max_resp_body_bytes = conf.max_resp_body_bytes or MAX_RESP_BODY
    
    if ctx._resp_body_bytes and ctx._resp_body_bytes >= max_resp_body_bytes then
        return
    end
    -- 收集指定最大长度的响应体
    local final_body = core.response.hold_body_chunk(ctx, true, max_resp_body_bytes)
    if not final_body then
        return
    end
    -- 后续代码省略...
end 
```

**发现原因**

在`hold_body_chunk`函数中，对响应体数据（字节流）进行截断，<u>请求结果通常是 UTF-8 编码格式的字符串，当发生响应体数据截断时，可能会截断在一个多字节字符的中间位置，导致字符无法正常解析，从而出现乱码。</u>这也就是为什么在日志中通常看到响应结果最后一个字符是乱码的根本原因。

# 四、解决方案
针对**响应结果是 UTF-8 编码字符串**这种场景，可以在日志输出前对响应体数据进行检查，**去除掉尾部不完整的 UTF8 字符字节数据**，保证输出完整的 UTF8 字符串数据，从而避免乱码问题。
```lua
function _M.log(conf, ctx)
    local entry = log_util.get_log_entry(plugin_name, conf, ctx)
    if entry == nil then
        return
    end
    
    -- if response body is too huge, fast truncate string in utf8 encoding
    if ctx.resp_body and entry.response.body then
        entry.response.body = safe_truncate_utf8(entry.response.body);
    end
    
    -- 日志输出
    write_file_data(conf, entry)
end
```

**safe_truncate_utf8** 函数的完整实现如下：
```lua
-- 从指定位置反向找到合法首字节位置
local function find_valid_start_byte(str, pos)
    while pos > 0 do
        local b = str:byte(pos)
        -- 首字节需满足 UTF-8 规则
        if (b >= 0x00 and b <= 0x7F) or          -- 1字节
                (b >= 0xC2 and b <= 0xDF) or     -- 2字节
                (b >= 0xE0 and b <= 0xEF) or     -- 3字节
                (b >= 0xF0 and b <= 0xF4) then   -- 4字节
            return pos
        end
        pos = pos - 1
    end
    return 0
end

local function validate_sequence(str, start_pos)
    local b = str:byte(start_pos)
    local required_follow = 0

    -- 严格遵循 UTF-8 首字节规范
    if b >= 0xF0 and b <= 0xF4 then       -- 4字节字符（修正范围）
        required_follow = 3
    elseif b >= 0xE0 and b <= 0xEF then   -- 3字节字符
        required_follow = 2
    elseif b >= 0xC2 and b <= 0xDF then   -- 2字节字符（修正范围）
        required_follow = 1
    elseif b <= 0x7F then                 -- 1字节 ASCII
        required_follow = 0
    else                                  -- 非法首字节（如 0x80-0xBF）
        return false, 0
    end

    -- 检查后续字节合法性
    for i = 1, required_follow do
        local pos = start_pos + i
        if pos > #str then return false, 0 end
        local fb = str:byte(pos)
        if fb < 0x80 or fb > 0xBF then   -- 必须为 10xxxxxx
            return false, 0
        end
    end
    return true, (required_follow + 1)  -- 总字节数 = 首字节 + 后续字节数
end

---
--- 截除尾部不完整的 UTF-8 字符字节数据（兼容Lua 5.1版本）
---@since Lua 5.1
---@overload fun(s:string):string
---@param str string
---@return string
local function safe_truncate_utf8(str)
    if not str or str == "" then return str end
    local max_pos = #str

    -- 反向查找合法字符边界
    while max_pos > 0 do
        -- 找到有效首字节位置（避免从中间字节开始）
        local start_pos = find_valid_start_byte(str, max_pos)
        if start_pos == 0 then break end

        -- 验证字符完整性并获取总字节数
        local is_valid, char_length = validate_sequence(str, start_pos)
        if is_valid then
            return str:sub(1, start_pos + char_length - 1)  -- 截取完整字符
        else
            max_pos = start_pos - 1  -- 向前回溯
        end
    end
    return ""
end
```

Lua 5.3 中增强了对字符串的处理能力，提供了`utf8`模块，可以通过`utf8.len`结合`string.sub`完成类似的截取操作。
```lua
---
--- 截除尾部不完整的 UTF-8 字符字节数据（通过Lua 5.3版本utf8库实现）
---@overload fun(s:string):string
---@param s string
---@return string
function fast_truncate_utf8(s)
    if not s or s == "" then
        return s
    end
    -- utf8.len 返回 nil 和出错位置（第一个非法字节的位置）
    -- 如果返回数字，则说明整个字符串都是合法的 UTF-8。
    local n, pos = utf8.len(s)
    if n then
        -- 完全合法，无需修剪
        return s
    else
        -- pos 为第一个检测到错误的字节下标，trim 掉从该位置开始的内容
        return s:sub(1, pos - 1)
    end
end
```
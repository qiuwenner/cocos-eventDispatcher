#### 简介
1. cocos的事件分发机制，先创建事件，然后注册到事件管理中心_eventDispatcher，通过发布事件得到响应进行回调，完成事件流。
2. _eventDispatcher是Node的属性，通过它管理当前节点（场景、层、精灵等）的所有事件的分发。但它本身是一个单例模式值的引用，在Node的构造函数中，通过**Director::getInstance()->getEventDispatcher()** 获取，有了这个属性，就能方便的处理事件。

#### 事件相关的类
```css
Event （基类） 
EventCustom （自定义事件） 
EventTouch （触摸事件）
EventMouse （鼠标事件） 
EventKeyboard （键盘事件） 
EventFocus （控件获取焦点事件） 
EventAcceleration （加速计事件）
```

#### 事件监听器
```css
EventListener, EventListenerCustom, EventListenerFocus, EventListenerMouse, EventListenerTouch, EventListenerKeyboard, EventListenerAcceleration
```

#### 事件分发器
```css
EventDispatcher （事件分发机制逻辑集合体）
```

#### 事件监听器的创建与监听
*创建*：直接new一个Event的子类即可。

*监听*：添加事件监听器有三个方法，都在 EventDispatch 类中，分别是:
```cpp
void addEventListenerWithSceneGraphPriority(EventListener* listener, Node* node);
void addEventListenerWithFixedPriority(EventListener* listener, int fixedPriority);
EventListenerCustom* addCustomEventListener(const std::string &eventName, const std::function<void(EventCustom*)>& callback);
```

#### 事件的优先级
1. 优先级越低，越先响应事件
2. 如果优先级相同，则上层的（z轴）先接收触摸事件。
```cpp
void EventDispatcher::dispatchEvent(Event* event)
{
    ...
    // 先通过event获取到事件的标志ListenerID
    auto listenerID = __getListenerID(event);
    // 排序此事件的所有的监听器
    sortEventListeners(listenerID);
    // 分发事件逻辑的函数指针
    auto pfnDispatchEventToListeners = &EventDispatcher::dispatchEventToListeners;
    if (event->getType() == Event::Type::MOUSE) {
        // 如果是鼠标事件重新赋值分发事件的函数指针
        pfnDispatchEventToListeners = &EventDispatcher::dispatchTouchEventToListeners;
    }
    // 获取改事件的所有的监听器
    auto iter = _listenerMap.find(listenerID);
    if (iter != _listenerMap.end())
    {
        // 如果有，取出里面监听器的Vector
        auto listeners = iter->second;
        // 找到对应的监听器的时候会触发的回调函数
        auto onEvent = [&event](EventListener* listener) -> bool{
            event->setCurrentTarget(listener->getAssociatedNode());
            // 触发onEvent回调
            listener->_onEvent(event);
            return event->isStopped();
        };
        // 调用函数指针分发事件
        (this->*pfnDispatchEventToListeners)(listeners, onEvent);
    }
 
    ...
}

void EventDispatcher::dispatchEventToListeners(EventListenerVector* listeners, const std::function<bool(EventListener*)>& onEvent)
{
    bool shouldStopPropagation = false;
    auto fixedPriorityListeners = listeners->getFixedPriorityListeners();
    auto sceneGraphPriorityListeners = listeners->getSceneGraphPriorityListeners();
 
    ssize_t i = 0;
    // priority < 0 优先处理priority小于0的时候的事件监听器
    if (fixedPriorityListeners)
    {
        CCASSERT(listeners->getGt0Index() <= static_cast<ssize_t>(fixedPriorityListeners->size()), "Out of range exception!");
 
        if (!fixedPriorityListeners->empty())
        {
            for (; i < listeners->getGt0Index(); ++i)
            {
                auto l = fixedPriorityListeners->at(i);
                // 判断是否可以执行事件，如果可以最后调用onEvent执行，如果onEvent返回true，说明吞噬事件，结束分发。
                if (l->isEnabled() && !l->isPaused() && l->isRegistered() && onEvent(l))
                {
                    shouldStopPropagation = true;
                    break;
                }
            }
        }
    }
    // 接下来分发SceneGraphPriority的事件
    if (sceneGraphPriorityListeners)
    {
        // 判断事件是否已经终止发送
        if (!shouldStopPropagation)
        {
            // priority == 0, scene graph priority
            for (auto& l : *sceneGraphPriorityListeners)
            {
                if (l->isEnabled() && !l->isPaused() && l->isRegistered() && onEvent(l))
                {
                    shouldStopPropagation = true;
                    break;
                }
            }
        }
    }
    // 最后分发到fixedPriority > 0 的监听器
    if (fixedPriorityListeners)
    {
        if (!shouldStopPropagation)
        {
            // priority > 0
            ssize_t size = fixedPriorityListeners->size();
            for (; i < size; ++i)
            {
                auto l = fixedPriorityListeners->at(i);
 
                if (l->isEnabled() && !l->isPaused() && l->isRegistered() && onEvent(l))
                {
                    shouldStopPropagation = true;
                    break;
                }
            }
        }
    }
}
```
分析 dispatchEvent 和 dispatchEventToListeners 方法可以得出：
两种优先级， SceneGraphPriority 和 FixedPriority ，先分发事件到 fixedPriority < 0 的监听器中，然后再分发到 = 0 的监听器 (SceneGraphPriority) 中，最后在分发到 > 0 的监听器中，如果中途出现 onEvent 返回为 true 的结果，则终止分发。
```css
ps:
1. 这里的 onEvent 调用的其实就是上面 dispatchEvent 代码中的 lambda 表达式。如果想要创建一个触摸事件的优先级比当前所有的触摸事件优先级都高话，只需要把 fixedPriority 的数值设为 < 0 即可。
2. addEventListenerWithSceneGraphPriority 的事件监听器优先级是0，而且在 addEventListenerWithFixedPriority 中的事件监听器的优先级不可以设置为 0，因为这个是保留给 SceneGraphPriority 使用的。
有一点非常重要，FixedPriority listener添加完之后需要手动remove，而SceneGraphPriority listener是跟node绑定的，在node的析构函数中会被移除。移除方法：dispatcher->removeEventListener(listener)
```

#### 实例
***下面是我封装的用户自定义事件的lua层代码，可以直接使用***

```lua
-- @ FileName:   GameEventDispatcher.lua
-- @ Describe:   游戏自定义事件

local GameEvent = {}
GameEvent.eventDispatcher = cc.Director:getInstance():getEventDispatcher()

-- 注册监听对象事件
function GameEvent:RegisterObjEvent(eventName, handler, obj)
    local customListener = cc.EventListenerCustom:create(eventName, handler)
    self.eventDispatcher:addEventListenerWithSceneGraphPriority(customListener, obj)
end

-- 移除对象事件
function GameEvent:Remove(obj)
    self.eventDispatcher:removeEventListenersForTarget(obj)
end

-- 注册监听事件
function GameEvent:RegisterEvent(eventName, handler, fixedPriority)
    fixedPriority = fixedPriority or 1
    local customListener = cc.EventListenerCustom:create(eventName, handler)
    self.eventDispatcher:addEventListenerWithFixedPriority(customListener, fixedPriority)
end

-- 移除事件
function GameEvent:RemoveCustomEvent(eventName)
    self.eventDispatcher:removeCustomEventListeners(eventName)
end

-- 派发事件  
function GameEvent:PushEvent(eventName, userData)
    userData = userData or ""
    local newEvent = cc.EventCustom:new(eventName)
    newEvent._usedata = userData
    self.eventDispatcher:dispatchEvent(newEvent)
end

return GameEvent
```

调用
```lua
-- @ FileName:   testClass.lua
-- @ Describe:   调用定义事件

local GameEvent = import(".GameEvent")
local testClass = class("testClass", function() end)

-- main
function testClass:ctor()
    GameEvent:RegisterObjEvent("test_eventName", handle(self, self.testfunc), self) -- 注册事件
    self:send()
end 

-- 事件回调
function testClass:testfunc = function(data)
    print("userdata is ----", data._usedata)
end 

-- 派发事件
function testClass:send = function()
    GameEvent:PushCustomEvent("test_eventName", "I am userdata")
end 

return testClass
```
ps:
需要注意的一点是，我在查看**EventDispatcher**类时，发现是有直接派发用户自定义事件的函数**void dispatchCustomEvent(const std::string &eventName, void *optionalUserData = nullptr)**，但是在C++导出给lua的API中，将参数2拦截了，调用会直接报错

C++ EventDispatcher 类中
```cpp
void EventDispatcher::dispatchCustomEvent(const std::string &eventName, void *optionalUserData)
{
    EventCustom ev(eventName);
    ev.setUserData(optionalUserData);
    dispatchEvent(&ev);
}
```

C++导出给lua的API中
```cpp
    ...
    argc = lua_gettop(tolua_S)-1;
    if (argc == 1) 
    {
        std::string arg0;

        ok &= luaval_to_std_string(tolua_S, 2,&arg0, "cc.EventDispatcher:dispatchCustomEvent");
        if(!ok)
        {
            tolua_error(tolua_S,"invalid arguments in function 'lua_cocos2dx_EventDispatcher_dispatchCustomEvent'", nullptr);
            return 0;
        }
        cobj->dispatchCustomEvent(arg0);
        lua_settop(tolua_S, 1);
        return 1;
    }
    if (argc == 2) 
    {
        std::string arg0;
        void* arg1;

        ok &= luaval_to_std_string(tolua_S, 2,&arg0, "cc.EventDispatcher:dispatchCustomEvent");

        #pragma warning NO CONVERSION TO NATIVE FOR void*
		ok = false;
        if(!ok)
        {
            tolua_error(tolua_S,"invalid arguments in function 'lua_cocos2dx_EventDispatcher_dispatchCustomEvent'", nullptr);
            return 0;
        }
        cobj->dispatchCustomEvent(arg0, arg1);
        lua_settop(tolua_S, 1);
        return 1;
    }
    luaL_error(tolua_S, "%s has wrong number of arguments: %d, was expecting %d \n", "cc.EventDispatcher:dispatchCustomEvent",argc, 1);
    return 0;
    ...
```
可以看到，当参数为2时，ok = false 直接拦截了，大概就是不让你使用。其实dispatchCustomEvent里面也是调用dispatchEvent，所以也没什么问题。

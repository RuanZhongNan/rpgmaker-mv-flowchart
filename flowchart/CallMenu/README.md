CallMenu分三部分 onKeyDown、SceneMangager.update、onKeyUp
# onKeyDown 
任何一个键位被按下就会触发Input._onKeyDown函数
这个函数在游戏初始化时就被注册到事件监听上
```
Input._onKeyDown = function(event) {
    if (this._shouldPreventDefault(event.keyCode)) {
        event.preventDefault();
    if (event.keyCode === 144) {    // Numlock
        this.clear();
    }
    
    var buttonName = this.keyMapper[event.keyCode];
    if (ResourceHandler.exists() && buttonName === 'ok') 
        ResourceHandler.retry();
    } else if (buttonName) {
        this._currentState[buttonName] = true;
    }
};
```
在唤出菜单只需关注三个键位"escape","X","numpad 0"
在Input.keyMapper中这三个键被映射为"escape"

如果按下其中任何一个键
则this(Input)._currentState[buttonName] = true //buttonName = "escape"

# SceneManager.update

每帧更新都会调用这个函数。
在唤出菜单中只有this.updateMain函数比较重要。

## 进入到循环while (this._accumulator >= this._deltaTime),

第一次循环进行场景的改变，后面的循环进行新场景的开始
### 第一次循环

#### 1)首先调用this.updateInputData更新按键数据
此时，this._latestButton 可能为null,也可能为"escape"映射之外的键位
在这一步，将 this._latestButton 更新为了"escape"
同时 this._pressedTime = 0 
同时 this._previousState[name] = this._currentState[name] 将"escape“更新为上一个按键状态
按键数据更新完毕

#### 2)this.changeScene()

if (this.isSceneChanging() && !this.isCurrentSceneBusy()) //false
下面看看调用的这两个函数的具体返回值

```
SceneManager.isSceneChanging = function() {
	return this._exiting || !!this._nextScene;
};
```

this._exiting 退出游戏时才会为true,一般情况下都为false
this._nextScene 此时为null
这个函数返回false

```
SceneManager.isCurrentSceneBusy = function() {
    return this._scene && this._scene.isBusy();
};

Scene_Base.prototype.isBusy = function() {
    return this._fadeDuration > 0;
};
```
this._fadeDuration 用于控制淡入淡出的持续时间,一般情况下为0
this.isCurrentSceneBusy()函数返回false
但是由于是 !this.isCurrentSceneBusy() 故&&右边的表达式为true

**第一次循环不进入this.changeScene()**

#### 3)this.updateScene()
```
SceneManager.updateScene = function() {
    if (this._scene) {
        if (!this._sceneStarted && this._scene.isReady()) {
            this._scene.start();
            this._sceneStarted = true;
            this.onSceneStart();
        }
        if (this.isCurrentSceneStarted()) {
            this._scene.update();
        }
    }
};
```
在第一次循环中,this._scene = Scene_Map
this_sceneStarted 记录当前场景是否已经开始,显然,Scene_Map已经开始,this._sceneStarted = true。
!this._sceneStarted && this._scene.isReady() 为false

```
SceneManager.isCurrentSceneStarted = function() {
    return this._scene && this._sceneStarted;
};
```
this.isCurrentSceneStarted() 为true

##### 进入到Scene_Map.update
```
Scene_Map.prototype.update = function() {
    this.updateDestination();
    this.updateMainMultiply();
    if (this.isSceneChangeOk()) {
        this.updateScene();
    } else if (SceneManager.isNextScene(Scene_Battle)) {
        this.updateEncounterEffect();
    }
    this.updateWaitCount();
    Scene_Base.prototype.update.call(this);
};
```

```
Scene_Map.prototype.isSceneChangeOk = function() {
    return this.isActive() && !$gameMessage.isBusy();
};

Scene_Base.prototype.isActive = function() {
    return this._active;
};
```
 显而易见，此时Scene_Map正在运行，且没有消息窗口。

##### 进入到this.updateScene()

```
Scene_Map.prototype.updateScene = function() {
    this.checkGameover();
    if (!SceneManager.isSceneChanging()) {
        this.updateTransferPlayer();
    }
    if (!SceneManager.isSceneChanging()) {
        this.updateEncounter();
    }
    if (!SceneManager.isSceneChanging()) {
        this.updateCallMenu();
    }
    if (!SceneManager.isSceneChanging()) {
        this.updateCallDebug();
    }
};
```
此时SceneManager.isSceneChanging()为false
可以看到this.updateCallMenu()才是我们需要研究的函数，在这个函数内判断是否唤出菜单。

##### 进入到this.updateCallMenu()

```
Scene_Map.prototype.updateCallMenu = function() {
    if (this.isMenuEnabled()) {
        if (this.isMenuCalled()) {
            this.menuCalling = true;
        }
        if (this.menuCalling && !$gamePlayer.isMoving()) {
            this.callMenu();
        }
    } else {
        this.menuCalling = false;
    }
};
```
首先判断了菜单是否允许开始，显然是允许的this.isMenuEnabled()返回true

接下来要重点关注this.isMenuCalled函数，这个函数判断菜单是否要被唤出
```
Scene_Map.prototype.isMenuCalled = function() {
    return Input.isTriggered('menu') || TouchInput.isCancelled();
};
//判断按键输入 或 触摸输入
```

```
Input.isTriggered = function(keyName) {
    if (this._isEscapeCompatible(keyName) && this.isTriggered('escape')) {
        return true;
    } else {
        return this._latestButton === keyName && this._pressedTime === 0;
    }
};

Input._isEscapeCompatible = function(keyName) {
    return keyName === 'cancel' || keyName === 'menu';
};//true
```
根据前文所述this._latestButton = "escape"
this._isEscapeCompatible(keyName) 也返回true
this.isTriggered('escape')返回true
this.isMenuCalled()返回true

##### 进入到this.callMenu()
在这一步执行切换场景的准备工作
将Scene_Map运行状态设置为停止
将下一个场景设置为Scene_Menu

```
Scene_Map.prototype.callMenu = function() {
    SoundManager.playOk();
    SceneManager.push(Scene_Menu); //场景预替换
    Window_MenuCommand.initCommandPosition(); 
    $gameTemp.clearDestination();
    this._mapNameWindow.hide();
    this._waitCount = 2;
};
```
在SceneManager.push(Scene_Menu)执行主要的场景预替换工作

```
SceneManager.goto = function(sceneClass) {
    if (sceneClass) {
        this._nextScene = new sceneClass();
    }
    if (this._scene) {
        this._scene.stop();
    }
};
```
this._nextScene 初始化为Scene_Menu实例  

this._scene.stop()将Scene_Map运行状态设置为停止
主要关注一点Scene_Map._active = false

**运行到这一步，updateScene()结束了。**

### 第二次循环
#### 1)this.updateInputData() 
将Input._pressedTime加1

#### 2)this.changeScene()
```
SceneManager.changeScene = function() {
    if (this.isSceneChanging() && !this.isCurrentSceneBusy()) {
        if (this._scene) {
            this._scene.terminate();
            this._scene.detachReservation();
            this._previousClass = this._scene.constructor;
        }

        this._scene = this._nextScene;

        if (this._scene) {
            this._scene.attachReservation();
            this._scene.create();
            this._nextScene = null;
            this._sceneStarted = false;
            this.onSceneCreate();
        }
        if (this._exiting) {
            this.terminate();
        }
    }
};
```
据前文所述this.isCurrentSceneBusy()一般情况下为false
而this.isSceneChanging()由于_nextscene = Scene_Menu 返回值为true
此时会进入changeScene函数执行场景的替换

一开始this._scene = Scene_Map
this._scene.terminate() 执行Scene_Map的停止

**this._scene = this._nextScene 更换当前场景**
this._scene.create() 创建Scene_Menu
具体怎么创建在此就不展开了，想知道可以去研究Scene_Menu.prototype.create函数
```
Scene_Menu.prototype.create = function() {
    Scene_MenuBase.prototype.create.call(this) //背景
    this.createCommandWindow(); //左上角选项
    this.createGoldWindow(); //金币窗口
    this.createStatusWindow() //右侧状态窗口
};
```
**this._sceneStarted = false 标识当前场景未开始**

#### 3)this.updateScene
```
SceneManager.updateScene = function() {
    if (this._scene) {
        if (!this._sceneStarted && this._scene.isReady()) {
            this._scene.start();
            this._sceneStarted = true;
            this.onSceneStart();
        }
        if (this.isCurrentSceneStarted()) {
            this._scene.update();
        }
    }
};
```

!this._sceneStarted = true
this._scene.isReady() 当Scene_Menu准备好，返回true
主要就是确认图片资源是否缓存好，这里假定已经缓存好

this._scene.start()
```
Scene_Menu.prototype.start = function() 
    Scene_MenuBase.prototype.start.call(this); //Scene_Base.prototype.start
    this._statusWindow.refresh(); //刷新右侧状态窗口
};
```

至此，唤出菜单就结束了。
this._scene.update() 调用Scene_Base.prototype.update更新children
接下来的循环就是窗口状态的更新

# onKeyUp
这一步比较简单
```
Input._onKeyUp = function(event) {
    var buttonName = this.keyMapper[event.keyCode];
    if (buttonName) {
        this._currentState[buttonName] = false;
    }
    if (event.keyCode === 0) {  // For QtWebEngine on OS X
        this.clear();
    }
};
```
就是将按键的状态设置为false
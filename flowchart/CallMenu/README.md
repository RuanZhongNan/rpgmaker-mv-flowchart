CallMenu分三部分 onKeyDown、SceneMangager.update、onKeyUp

# onKeyDown 
任何一个键位被按下就会触发Input._onKeyDown函数

在唤出菜单只需关注三个键位"enter","space","Z"
在Input.keyMapper中这三个键被映射为"ok"

如果按下其中任何一个键
则this(Input)._currentState[buttonName] = true //buttonName = "ok"

# SceneManager.update

每帧更新都会调用这个函数。
在唤出菜单中只有this.updateMain函数比较重要。

## 进入到循环while (this._accumulator >= this._deltaTime),

第一次循环进行场景的改变，第二次循环进行新场景的开始
### 第一次循环

#### 1)首先调用this.updateInputData更新按键数据
此时，this._latestButton 可能为null,也可能为"ok"映射之外的键位
在这一步，将 this._latestButton 更新为了"ok"
同时 this._pressedTime = 0 初始化该键位的按下次数为0
同时 this._previousState[name] = this._currentState[name] 将"ok“更新为上一个按键状态
按键数据更新完毕

#### 2)this.changeScene()

if (this.isSceneChanging() && !this.isCurrentSceneBusy()) //false

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
this._fadeDuration 应该是用于控制淡入淡出,一般情况下为0
这个函数也返回false
但是由于是 !this.isCurrentSceneBusy() 故&&右边的表达式一般为true

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

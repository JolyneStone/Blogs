## Windows Terminal 安装及美化

windows terminal 是今年微软Build大会上推出的一款的全新终端，用来代替cmder之类的第三方终端。具有亚克力透明、多标签、Unicode支持(中文,Emoji)、自带等宽字体等这些特性。
现在可以在微软商店里面进行安装，系统要求是win10 1903及以上，不过目前还是preview的，可能有些bug，但是我用着还没遇到过。

![](https://oos-cn-kirayoshikage.oss-cn-hangzhou.aliyuncs.com/images/20190901171122.png)
点击直接安装即可，先打开来瞅瞅长啥样。
![](https://oos-cn-kirayoshikage.oss-cn-hangzhou.aliyuncs.com/images/20190901171658.png)
说实话看着有点丑，接下来就让我们来美化它吧~

### 自定义配置
Windows Terminal提供了许多设置和配置选项，可以对Terminal的外观自定义设置。配置文件格式为json，对于程序员来说很友好。
打开设置

```json
{
    "globals" : 
    {
        "alwaysShowTabs" : true,
        "copyOnSelect" : false,
        "defaultProfile" : "{61c54bbd-c2c6-5271-96e7-009a87ff44bf}",
        "initialCols" : 120,
        "initialRows" : 30,
        "keybindings" : 
        [
            ...
        ],
        "requestedTheme" : "system",
        "showTabsInTitlebar" : true,
        "showTerminalTitleInTitlebar" : true,
        "wordDelimiters" : " ./\\()\"'-:,.;<>~!@#$%^&*|+=[]{}~?\u2502"
    },
    "profiles" : 
    [
        {
            "acrylicOpacity" : 0.5,
            "background" : "#012456",
            "closeOnExit" : true,
            "colorScheme" : "Campbell",
            "commandline" : "powershell.exe",
            "cursorColor" : "#FFFFFF",
            "cursorShape" : "bar",
            "fontFace" : "Consolas",
            "fontSize" : 10,
            "guid" : "{61c54bbd-c2c6-5271-96e7-009a87ff44bf}",
            "historySize" : 9001,
            "icon" : "ms-appx:///ProfileIcons/{61c54bbd-c2c6-5271-96e7-009a87ff44bf}.png",
            "name" : "Windows PowerShell",
            "padding" : "0, 0, 0, 0",
            "snapOnInput" : true,
            "startingDirectory" : "%USERPROFILE%",
            "useAcrylic" : false
        },
        ...
    ],
    "schemes" : 
    [
        ...
    ]
}
```

结构很简单，globals是全局设置，profiles是各个不同的shell的配置，schemes是配色的方案， ms-appx:///xxx 指的是应用程序包的xxx文件，在我电脑上对应的路径是C:\\Program Files\\WindowsApps\\Microsoft.WindowsTerminal_0.4.2382.0_x64__8wekyb3d8bbwe

### 设置默认shell

```json
"defaultProfile": "{c6eaf9f4-32a7-5fdc-b5cf-066e8a4b1e40}"
```

将defaultProfile设置成profile的guid值即可，我设置为了wsl的guid

### 设置字体

```json
"fontFace": "DejaVu Sans Mono"
```

### 设置背景图

```json
   "acrylicOpacity": 0.5,
   "useAcrylic": true,
   "backgroundImage": "C:\\Users\\Kira Yoshikage\\Pictures\\wallhaven-36230.png",
   "backgroundImageOpacity": 0.75,
   "backgroundImageStretchMode": "fill"
```

### 设置schemes

```json
{
 "schemes": [
    {
      "name": "Campbell",
      "foreground": "#F2F2F2",
      "background": "#0C0C0C",
      "colors": [
        "#0C0C0C",
        "#C50F1F",
        "#13A10E",
        "#C19C00",
        "#0037DA",
        "#881798",
        "#3A96DD",
        "#CCCCCC",
        "#767676",
        "#E74856",
        "#16C60C",
        "#F9F1A5",
        "#3B78FF",
        "#B4009E",
        "#61D6D6",
        "#F2F2F2"
      ]
    },
    {
      "name": "Solarized Dark",
      "foreground": "#FDF6E3",
      "background": "#073642",
      "colors": [
        "#073642",
        "#D30102",
        "#859900",
        "#B58900",
        "#268BD2",
        "#D33682",
        "#2AA198",
        "#EEE8D5",
        "#002B36",
        "#CB4B16",
        "#586E75",
        "#657B83",
        "#839496",
        "#6C71C4",
        "#93A1A1",
        "#FDF6E3"
      ]
    },
    {
      "name": "Solarized Light",
      "foreground": "#073642",
      "background": "#FDF6E3",
      "colors": [
        "#073642",
        "#D30102",
        "#859900",
        "#B58900",
        "#268BD2",
        "#D33682",
        "#2AA198",
        "#EEE8D5",
        "#002B36",
        "#CB4B16",
        "#586E75",
        "#657B83",
        "#839496",
        "#6C71C4",
        "#93A1A1",
        "#FDF6E3"
      ]
    },
    {
      "name": "Ubuntu",
      "foreground": "#EEEEEC",
      "background": "#2C001E",
      "colors": [
        "#EEEEEC",
        "#16C60C",
        "#729FCF",
        "#B58900",
        "#268BD2",
        "#D33682",
        "#2AA198",
        "#EEE8D5",
        "#002B36",
        "#CB4B16",
        "#586E75",
        "#657B83",
        "#839496",
        "#6C71C4",
        "#93A1A1",
        "#FDF6E3"
      ]
    },
    {
      "name": "UbuntuLegit",
      "foreground": "#EEEEEE",
      "background": "#2C001E",
      "colors": [
        "#4E9A06",
        "#CC0000",
        "#300A24",
        "#C4A000",
        "#3465A4",
        "#75507B",
        "#06989A",
        "#D3D7CF",
        "#555753",
        "#EF2929",
        "#8AE234",
        "#FCE94F",
        "#729FCF",
        "#AD7FA8",
        "#34E2E2",
        "#EEEEEE"
      ]
    }
}
```

### 最终结果
![](https://oos-cn-kirayoshikage.oss-cn-hangzhou.aliyuncs.com/images/20190901205148.png)
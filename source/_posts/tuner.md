---
title: 实现一个在线调音器
date: 2023-06-28 16:46:21
tags: 小工具
---

<small>原文发布于 [CSDN](https://blog.csdn.net/RaeZhang/article/details/113075391?spm=1001.2014.3001.5502)，本文系近期迁移，想看更多欢迎访问 [我的 CSDN 主页](https://blog.csdn.net/RaeZhang?type=blog)</small>

本来想做一个网页版的调音器，在网上搜了一下，发现已经有人实现过了，但是自己对这个原理还是很感兴趣，所以参照人家的[博客](https://www.jianshu.com/p/4fd8779943c3)和[源码](https://github.com/qiuxiang/tuner)，用 svelte 重新实现了,最终效果请看 [演示](https://raezhang0822.github.io/codes/tuner/app/index.html)

一般用的调音器都是表盘式的，工作原理就是设备听到声音，分析声音的频率，判断出音高之后在屏幕上显示出来，在这个过程中我们可能需要解决这些问题：

1. 获取设备麦克风权限
2. 获取声音频率
3. 将声音频率换算成音高
4. 将当前声音最接近的音名显示到表盘中间，并用指针显示当前声音与该音名对应的声音之间的偏差。偏低则向左指，偏低则向右指。

看下这个过程的代码实现。

## 获取麦克风权限

获取麦克风权限使用 [navigator.mediaDevices.getUserMedia()](https://developer.mozilla.org/zh-CN/docs/Web/API/MediaDevices/getUserMedia) 这个 api,
它接受一个[MediaStreamConstraints (en-US) ](https://developer.mozilla.org/en-US/docs/Web/API/MediaStreamConstraints)对象，指定了请求的媒体类型和相对应的参数。返回对象是一个 Promise.

这里只需要处理音频，所以参数传入`{audio:true}` 就可以

```javascript
navigator.mediaDevices
  .getUserMedia({ audio: true })
  .then(handleStream)
  .catch(function (error) {
    alert(error.name + ": " + error.message);
  });
```

获取到媒体流之后，要从中分析得到声音频率。

这里有一个坑，我把代码部署到服务器之后，发现无法获取到麦克风权限，控制台报错
![image](./img/tuner/error.png)

![image](./img/tuner/note.png)
发现必须使用 HTTPS 加密通信才能获取`getUserMedia()`，这个问题需要注意下。

## 获取声音数据

用来处理音频的 API 是[AudioContext](https://developer.mozilla.org/zh-CN/docs/Web/API/AudioContext)

> AudioContext 接口表示由链接在一起的音频模块构建的音频处理图，每个模块由一个 AudioNode 表示。音频上下文控制它包含的节点的创建和音频处理或解码的执行。在做任何其他操作之前，您需要创建一个 AudioContext 对象，因为所有事情都是在上下文中发生的。建议创建一个 AudioContext 对象并复用它，而不是每次初始化一个新的 AudioContext 对象，并且可以对多个不同的音频源和管道同时使用一个 AudioContext 对象。

分析音频流需要使用 [AudioContext.createAnalyser()](https://developer.mozilla.org/zh-CN/docs/Web/API/BaseAudioContext/createAnalyser)

> AudioContext 的 createAnalyser()方法能创建一个 AnalyserNode，可以用来获取音频时间和频率数据，以及实现数据可视化。

用 js 处理音频流需要用到
[AudioContext.createScriptProcessor()](https://developer.mozilla.org/zh-CN/docs/Web/API/BaseAudioContext/createScriptProcessor)

> AudioContext 接口的 createScriptProcessor() 方法创建一个 ScriptProcessorNode 用于通过 JavaScript 直接处理音频.

`scriptProcessor`是一个[ScriptProcessorNode](https://developer.mozilla.org/zh-CN/docs/Web/API/ScriptProcessorNode)

> ScriptProcessorNode 接口允许使用 JavaScript 生成、处理、分析音频. 它是一个 AudioNode， 连接着两个缓冲区音频处理模块, 其中一个缓冲区包含输入音频数据，另外一个包含处理后的输出音频数据. 实现了 AudioProcessingEvent (en-US) 接口的一个事件，每当输入缓冲区有新的数据时，事件将被发送到该对象，并且事件将在数据填充到输出缓冲区后结束.

监听它的`audioprocess`事件就可以获得输入音频数据。

音频通路需要连接最终的目标节点
[AudioContext.destination](https://developer.mozilla.org/zh-CN/docs/Web/API/BaseAudioContext/destination)

> AudioContext 的 destination 属性返回一个 AudioDestinationNode 表示 context 中所有音频（节点）的最终目标节点，一般是音频渲染设备，比如扬声器。

将以上几个节点依次连接起来

```javascript
// 获取AudioContext实例
const audioContext = new window.AudioContext();
// 创建 analyser用来获取音频频率数据
const analyser = audioContext.createAnalyser();
//
const bufferSize = 4096;
const scriptProcessor = audioContext.createScriptProcessor(bufferSize, 1, 1);

const handleStream = (stream) => {
  // 创建MediaStreamAudioSourceNode
  // 把AudioBufferSourceNode连接到analyser
  audioContext.createMediaStreamSource(stream).connect(analyser);
  // 将analyser连接到 scriptProcessor,用js处理
  analyser.connect(scriptProcessor);
  //将scriptProcessor与destination连接，这样才能形成到达扬声器的通路
  scriptProcessor.connect(audioContext.destination);
  scriptProcessor.addEventListener("audioprocess", function (event) {
    const data = event.inputBuffer.getChannelData(0); // 获取到音频数据
  });
};
```

## 计算声音频率

> 音高检测算法已经很成熟了，不乏论文和资料，诸如 java、c/c++ 都有现成的库可用，而 js 在这方面显然是有缺失的。因为我的目的是快速实现一个调音器，所以我是基于一个现有的 c 音频库 aubio 用 emscripten 编译成 js，以便在浏览器里运行。

这里将 c 语言的音频库转译成 js 涉及到[Web Assembluy](https://www.wasm.com.cn/)相关的知识。关于 Web Assembly,我也不是特别了解，以后有机会做一下实践。这里只需要了解下 Web Assembly 是做什么的就好。

简单来说，WebAssembly 是一种新的字节码格式，目前可以使用 C、C++、Rust、Go、Java、C#等编译器（未来还有更多）来创建 wasm 模块（见下图）。该模块以二进制的格式发送到浏览器，浏览器提供了专门的虚拟机环境来运行 wasm 的代码，与 JavaScript 虚拟机共享内存和线程等资源。

编译之后的 audio 库有两个文件，一个 audio.js 文件，一个 audio.wasm 文件。

通过这个库提供的方法，可以计算得到音频的频率

```javascript
  let pitchDetector = null;
  Aubio().then(function (aubio) {
  	// 确保Aubio模块加载完毕之后使用
    pitchDetector = new aubio.Pitch('default', bufferSize, 1, audioContext.sampleRate);
    ...
  });


  const handleStream = (stream) => {
    audioContext.createMediaStreamSource(stream).connect(analyser);
    analyser.connect(scriptProcessor);
    scriptProcessor.connect(audioContext.destination);
    scriptProcessor.addEventListener('audioprocess', function (event) {
      // 计算获取频率
      const frequency = pitchDetector.do(event.inputBuffer.getChannelData(0));
    });
  };
```

## 计算音名

### 关于音名

传统音乐理论中使用前七个拉丁字母：A、B、C、D、E、F、G（按此顺序则音高循序而上）以及一些变化（升音和降音符号）来标示不同的音符。两个音符间若相差一倍的频率，则称两者之间相差一个八度。为了标示同名但不同高度的音符，科学音调记号法利用字母及一个用来表示所在八度的阿拉伯数字，明确指出音符的位置。比如说，现在的标准调音音高 440 赫兹名为 A4，往上高八度则为 A5，继续向上可无限延伸；至于 A4 往下，则为 A3、A2…等。

把音高频率转成对应的 MIDI 编号，对照即可找到音名，比如 82.41 Hz 计算得到 MIDI 编号 38，即数字法音名 E2。[维基百科-音符](https://zh.wikipedia.org/wiki/%E9%9F%B3%E7%AC%A6)里有详细的描述、公式及八度命名系统对照表格。
![image](./img/tuner/p.png)
![image](./img/tuner/table.png)

写成代码

```javascript
const middleA = 440;
const semitone = 69;
const getNote = function (frequency) {
  const note = 12 * (Math.log(frequency / middleA) / Math.log(2));
  return Math.round(note) + semitone;
};

scriptProcessor.addEventListener("audioprocess", function (event) {
  const frequency = pitchDetector.do(event.inputBuffer.getChannelData(0));
  if (frequency) {
    const note = getNote(frequency);
  }
});
```

## 计算指针偏转

并不是每个声音的频率都恰好准确地等于音名的频率，大部分声音的频率会相对于音名有一定偏移量，**音分**是用来衡量音高偏移的单位。

在十二平均律中，将一个八度音程分为 12 个半音。每一个半音的音程（相当于相邻钢琴键间的音程）等于 100 音分。相差一个八度的两个音之间的音程总共是 1200 音分。

12 个音名分别是

```javascript
const noteStrings = [
  "C",
  "C♯",
  "D",
  "D♯",
  "E",
  "F",
  "F♯",
  "G",
  "G♯",
  "A",
  "A♯",
  "B",
];
```

每两个音名之间的差距为 100 音分，用音分来计算声音相对最近的音名的偏移量，并用来计算指针旋转角度。

下面是计算音分的方法，其计算公式详细可以去看[维基百科-音分](https://zh.wikipedia.org/wiki/%E9%9F%B3%E5%88%86)

```javascript
const getStandardFrequency = function (note) {
  return middleA * Math.pow(2, (note - semitone) / 12);
};

const getCents = function (frequency, note) {
  return Math.floor(
    (1200 * Math.log(frequency / getStandardFrequency(note))) / Math.log(2)
  );
};
```

完整的代码

```javascript
const onNoteDetected = function (note) {
  const { value, cents, frequency } = note;
  curValue = value;
  curDeg = (cents / 50) * 45; // 计算得到旋转角度
  curFrq = frequency;
};

scriptProcessor.addEventListener("audioprocess", function (event) {
  const frequency = pitchDetector.do(event.inputBuffer.getChannelData(0));
  if (frequency && onNoteDetected) {
    const note = getNote(frequency);
    onNoteDetected({
      name: noteStrings[note % 12],
      value: note,
      cents: getCents(frequency, note),
      octave: parseInt(note / 12) - 1,
      frequency: frequency,
    });
  }
});
```

## 表盘实现

通过上面的计算我们已经获得了当前频率、当前音名与指针需要偏转的角度，表盘的实现部分就比较简单了，直接贴代码看一下。

```html
// 刻度盘与指针
<script>
    export let deg = 0;
    const arr = Array.from(new Array(11).keys());
</script>

<div class="meter">
    <div class="meter-dot"></div>
    <div class="meter-pointer" style={`transform:rotate(${deg}deg)`}></div>
    {#each arr as i}
        <div class:meter-scale={true} class:meter-scale-strong = {i % 5 === 0 } style={`transform:rotate(${i * 9 - 45}deg)`}>
        </div>
    {/each}
</div>


<style>
.meter {
  position: fixed;
  left: 0;
  right: 0;
  bottom: 50%;
  width: 400px;
  height: 33%;
  margin: 0 auto 5vh auto;
}

.meter-pointer {
  width: 2px;
  height: 100%;
  background: #2c3e50;
  transform: rotate(45deg);
  transform-origin: bottom;
  transition: transform 0.5s;
  position: absolute;
  right: 50%;
}

.meter-dot {
  width: 10px;
  height: 10px;
  background: #2c3e50;
  border-radius: 50%;
  position: absolute;
  bottom: -5px;
  right: 50%;
  margin-right: -4px;
}

.meter-scale {
  width: 1px;
  height: 100%;
  transform-origin: bottom;
  transition: transform 0.2s;
  box-sizing: border-box;
  border-top: 10px solid;
  position: absolute;
  right: 50%;
}

.meter-scale-strong {
  width: 2px;
  border-top-width: 20px;
}
</style>
```

```html
<script>
  import Note from "./Note.svelte";
  export let value; //当前检测到的MIDI编号
  export let frequency; //当前频率
  export let nodeList; // node列表的dom节点
  const octaveList = [2, 3, 4, 5];
  const noteStrings = [
    "C",
    "C♯",
    "D",
    "D♯",
    "E",
    "F",
    "F♯",
    "G",
    "G♯",
    "A",
    "A♯",
    "B",
  ];

  const handleActive = (e) => {
    const dom = e.detail;
    nodeList.scrollLeft =
      dom.offsetLeft - (nodeList.clientWidth - dom.clientWidth) / 2;
  };
</script>

<style>
  .notes {
    margin: auto;
    width: 400px;
    position: fixed;
    top: 50%;
    left: 0;
    right: 0;
    text-align: center;
  }
  .notes-list {
    overflow: auto;
    overflow: -moz-scrollbars-none;
    white-space: nowrap;
    -ms-overflow-style: none;
    -webkit-mask-image: -webkit-linear-gradient(
      left,
      rgba(255, 255, 255, 0),
      #fff,
      rgba(255, 255, 255, 0)
    );
  }

  .frequency {
    font-size: 32px;
  }

  .frequency span {
    font-size: 50%;
    margin-left: 0.25em;
  }

  .notes-list::-webkit-scrollbar {
    display: none;
  }

  @media (max-width: 768px) {
    .notes {
      width: 100%;
    }
  }
</style>

<div class="notes">
  <div class="notes-list" bind:this="{nodeList}">
    {#each octaveList as octave} {#each noteStrings as note, index} <Note
    note={note} octave={octave} isActive={value === 12*(octave+1)+index}
    listDom={nodeList} on:onActive={handleActive}/> {/each} {/each}
  </div>
  <div class="frequency">{frequency.toFixed(2)}<span>Hz</span></div>
</div>
```

以上就是代码的核心部分。

<script>
  import Panel from "./Panel.svelte";
  import {
    isValid,
    setCookie,
    getCookie,
    notification,
    sendIsStarted,
    chooseFile,
    getDefaultAudios,
  } from "./utils.js";
  import { onMount } from "svelte";
  let tomatoes = "1";
  let rests = "0";
  let cycles = "1";
  let curCycles = 1;
  let minutes = 0;
  let seconds = 0;
  let isStarted = 1;
  let myWorker;
  let runningTitle = "";
  let audio = new Audio();
  let cur_audio = {};
  let audios = [];
  const IS_RUNNING = 0;
  const IS_TERMINATE = 1;
  const NOTIFICATION = 2;
  const RUNNING_STATUS = {
    TOMATO: 1,
    REST: 2,
  };
  $: btn_name = isStarted == 1 ? "开始" : "停止";
  function startOrStop() {
    if (isStarted == 1) {
      isStarted = 0;
      sendIsStarted(isStarted);
      myWorker = new Worker(new URL("./worker.js", import.meta.url));
      myWorker.postMessage({
        tomatoes,
        rests,
        cycles,
        status: 0,
      });
      myWorker.onmessage = (e) => {
        if (e.data.isPlayed != undefined) {
          if (e.data.isPlayed) {
            if (cur_audio.path != "") {
              audio = new Audio(new URL(cur_audio.path));
              audio.onerror = function () {
                notification("音乐文件播放失败，检查路径以及文件是否损坏。");
              };
              audio.play();
            }
          } else {
            if (audio.paused == false) {
              audio.pause();
              audio.currentTime = 0;
            }
          }
        }
        if (e.data.status == IS_TERMINATE) {
          isStarted = 1;
          sendIsStarted(isStarted);
          minutes = 0;
          seconds = 0;
          myWorker.terminate();
          if (audio.paused == false) {
            audio.pause();
          }
          audio.currentTime = 0;
          runningTitle = "";
        } else if (e.data.status == IS_RUNNING) {
          let remain_seconds = e.data.remain_seconds;
          if (e.data.running_status == RUNNING_STATUS.TOMATO) {
            runningTitle = "正在专注";
          } else if (e.data.running_status == RUNNING_STATUS.REST) {
            runningTitle = "正在休息";
          }
          minutes = Math.floor(remain_seconds / 60);
          seconds = Math.floor(remain_seconds % 60);
          curCycles = e.data.curCycles;
        } else if (e.data.status == NOTIFICATION) {
          notification(e.data.notification.message);
        }
      };
    } else {
      isStarted = 1;
      sendIsStarted(isStarted);
      myWorker.terminate();
      minutes = 0;
      seconds = 0;
      runningTitle = "";
      if (audio.paused == false) {
        audio.pause();
      }
      audio.currentTime = 0;
    }
  }

  async function checkAndSave() {
    if (
      !isValid(tomatoes, 1, 240) ||
      !isValid(rests, 0, 240) ||
      !isValid(cycles, 1, 100)
    ) {
      let obj = await getCookie();
      obj = JSON.parse(obj[0].value);
      tomatoes = obj.tomatoes;
      cycles = obj.cycles;
      rests = obj.rests;
      audios = obj.audios;
      cur_audio = obj.cur_audio;
    } else {
      tomatoes = parseInt(tomatoes).toString();
      rests = parseInt(rests).toString();
      cycles = parseInt(cycles).toString();
      setCookie({ tomatoes, rests, cycles, audios, cur_audio });
    }
  }
  function handleSelect(event) {
    let path = event.target.value;
    let result = audios.find((e) => {
      return e.path == path;
    });
    cur_audio = result;
    setCookie({
      tomatoes,
      rests,
      cycles,
      audios,
      cur_audio,
    });
  }
  async function addLocalMusic() {
    let file = await chooseFile();
    if (file != null) {
      audios.push({ name: file.name, path: file.path });
    }
    audios = audios.slice();
    cur_audio = file;
    setCookie({ tomatoes, rests, cycles, audios, cur_audio });
  }

  async function cleanMusic() {
    audios = await getDefaultAudios();
    cur_audio = audios[0];
    setCookie({ tomatoes, rests, cycles, audios, cur_audio });
  }
  onMount(async () => {
    let obj = await getCookie();
    obj = JSON.parse(obj[0].value);
    tomatoes = obj.tomatoes;
    rests = obj.rests;
    cycles = obj.cycles;
    cur_audio = obj.cur_audio;
    audios = obj.audios;
  });
</script>

<main>
  <div style="text-align:center">
    <Panel {minutes} {seconds} {runningTitle} />
  </div>
  <div style="display: flex; justify-content: center;">
    <table>
      <tbody>
        <tr style="text-align:center">
          <button on:click={startOrStop}>{btn_name}</button>
        </tr>
        {#if isStarted == 0}
          <tr>
            <div style="font-size: 20px; text-align:center">
              当前第{curCycles}个/总共{cycles}个
            </div>
          </tr>
        {/if}
        <tr>
          <div style="display: inline;flex:auto;margin:10px">
            <span style="margin-right:2px">番茄</span>
            <input
              bind:value={tomatoes}
              on:change={checkAndSave}
              type="text"
              min="0"
              max="10"
              placeholder="x"
            /><span>分钟</span>
          </div>
          <div style="display: inline; flex:auto;margin:10px">
            <span style="margin-right:2px">休息</span>
            <input
              bind:value={rests}
              on:change={checkAndSave}
              type="text"
              placeholder="x"
            /><span>分钟</span>
          </div>
          <div style="display: inline; flex:auto; margin:10px">
            <span>循环</span>
            <input
              bind:value={cycles}
              on:change={checkAndSave}
              type="text"
              placeholder="x"
            /><span>次</span>
          </div>
        </tr>
        <tr>
          <select
            style="white-space: nowrap;
            overflow: hidden;
						text-overflow: ellipsis;
						width: 100px;
						margin-left:10px"
            value={cur_audio.path}
            on:change={handleSelect}
          >
            {#each audios as audio}
              <option
                style="white-space: nowrap;
						overflow: hidden;
						text-overflow: ellipsis;
					"
                value={audio.path}
              >
                {audio.name}
              </option>
            {/each}
          </select>
          <div style="margin-left:10px">
            <a href="javascript:void(0)" on:click={addLocalMusic}
              >点击选择提示音</a
            >
          </div>
          <div style="margin-left:10px">
            <a href="javascript:void(0)" on:click={cleanMusic}>
              清理自定义音乐
            </a>
          </div>
        </tr>
      </tbody>
    </table>
  </div>
</main>

<style>
  span {
    font-size: 18px;
  }
  input {
    text-align: center;
    max-width: 30px;
    margin-right: 10px;
  }
</style>

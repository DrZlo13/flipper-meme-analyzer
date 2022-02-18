<script>
  import GenericSerial from "./generic-serial.js";
  import TerminalEmu from "./TerminalEmu.svelte";
  import Popup from "./Popup.svelte";
  import { onMount } from "svelte";

  let terminal;
  let popup_message;
  let popup_message_data = {};

  let serial = new GenericSerial();
  serial.connect_callback = connect_callback;
  serial.disconnect_callback = disconnect_callback;
  serial.send_callback = send_callback;
  serial.receive_callback = receive_callback;

  let connected = false;
  let info = 0;

  let canvas;

  function connect_callback() {
    connected = true;
    info = 0;

    serial.read_loop();
  }

  function disconnect_callback() {
    connected = false;
  }

  function decode(data) {
    return new TextDecoder().decode(data);
  }

  function send_callback(data) {}

  const colors = [
    [255, 0, 0],
    [0, 255, 0],
    [0, 0, 255],
    [255, 255, 0],
    [255, 0, 255],
    [0, 255, 255],
    [128, 0, 0],
    [0, 128, 0],
    [0, 0, 128],
    [128, 0, 128],
    [128, 128, 0],
    [0, 128, 128],
    [127, 255, 0],
    [210, 105, 30],
    [100, 149, 237],
    [255, 140, 0],
    [255, 20, 147],
    [218, 165, 32],
    [75, 0, 130],
    [176, 196, 222],
    [147, 112, 219],
    [255, 222, 173],
    [152, 251, 152],
    [250, 128, 114],
  ];

  let color_counter = 0;
  let thread_list;

  function thread_list_reset() {
    color_counter = 0;
    thread_list = {
      ".bss": { color: [128, 128, 128], size: 0 },
      main: { color: [255, 255, 255], size: 0 },
    };
  }
  thread_list_reset();

  function thread_list_insert_thread(thread) {
    if (!(thread in thread_list)) {
      thread_list[thread] = { color: colors[color_counter], size: 0 };
      color_counter++;
    }
  }

  const SRAM_BASE = 0x20000000;
  const SRAM_SIZE = (192 * 1024) / 4;
  const SRAM_W = 256;
  const SRAM_H = 192;

  if (SRAM_W * SRAM_H != SRAM_SIZE) {
    console.error("Wrong SRAM canvas size");
  }

  let memory = new Array(SRAM_SIZE);

  function memory_reset() {
    for (let index = 0; index <= SRAM_SIZE; index++) {
      memory[index] = { thread: "", size: 0, alloc: false, history: [] };
    }
  }

  function memory_insert_static(addr, size) {
    for (let index = addr; index < addr + size; index++) {
      memory[index] = { thread: ".bss", size: 0, alloc: true, history: [] };
    }

    if (size > 0) {
      memory[addr] = { thread: ".bss", size: size, alloc: true, history: [] };
    }
  }

  memory_reset();

  console.log("SRAM size: %d", SRAM_SIZE);

  function mem_set_block(addr, alloc, thread, size) {
    for (let i = 0; i < size / 4; i++) {
      if (memory[addr / 4 + i].alloc == alloc) {
        console.error(
          "Block crashed",
          addr / 4 + i,
          memory[addr / 4 + i],
          addr,
          {
            thread: thread,
            size: size,
            alloc: alloc,
          }
        );
      }

      memory[addr / 4 + i].alloc = alloc;

      if (alloc) {
        memory[addr / 4 + i].thread = thread;
        memory[addr / 4 + i].history.push("+" + thread + "[" + size + "]");
      } else {
        memory[addr / 4 + i].thread = "";
        memory[addr / 4 + i].history.push("-" + thread + "[" + size + "]");
      }
    }
  }

  function memory_get_size() {
    for (let thread in thread_list) {
      thread_list[thread].size = 0;
    }

    let size = 0;
    for (let index = 0; index <= SRAM_SIZE; index++) {
      size += memory[index].size;
      if (memory[index].thread != "" && memory[index].size > 0) {
        thread_list[memory[index].thread].size += memory[index].size;
      }
    }

    return size;
  }

  function memory_malloc(addr, thread, size) {
    thread_list_insert_thread(thread);

    mem_set_block(addr, true, thread, size);
    memory[addr / 4].thread = thread;
    memory[addr / 4].size = size;
    memory[addr / 4].alloc = true;
  }

  function memory_free(addr, thread) {
    if (memory[addr / 4].size == 0 || !memory[addr / 4].alloc) {
      console.error(
        'Mem at 0x%s[%d] not allocated but freed by "%s"',
        addr.toString(16),
        memory[addr / 4].size,
        thread
      );
    } else {
      if (memory[addr / 4].thread != thread) {
        console.error(
          'Mem at 0x%s[%d] allocated by "%s" but freed by "%s"',
          addr.toString(16),
          memory[addr / 4].size,
          memory[addr / 4].thread,
          thread
        );
      }
    }

    mem_set_block(addr, false, thread, memory[addr / 4].size);
    memory[addr / 4].thread = "";
    memory[addr / 4].size = 0;
    memory[addr / 4].alloc = false;
  }

  function parse_data(data) {
    let split = data.split("|");
    let name = split[0];
    let type = split[1];
    let addr = parseInt(split[2], 16) - SRAM_BASE;
    let size = 0;

    if (name == "") name = "main";

    if (name == "PHStart") {
      let heap_start = parseInt(split[1], 16) - SRAM_BASE;
      let heap_end = parseInt(split[2], 16) - SRAM_BASE;

      console.log("--- HEAP INIT ---");

      memory_reset();
      thread_list_reset();
      memory_insert_static(0, heap_start / 4);
      memory_insert_static(heap_end / 4, Math.max(0, SRAM_SIZE - heap_end / 4));
    } else {
      if (addr < 0) {
        console.error("Address < RAM start:", data);
      } else if (addr > SRAM_BASE) {
        console.error("Address > RAM end:", data);
      } else if (addr % 4 != 0) {
        console.error("Address not aligned: %s, %d", data, addr % 4);
      } else {
        if (type == "m") {
          size = parseInt(split[3]);
          memory_malloc(addr, name, size);
        } else if (type == "f") {
          memory_free(addr, name);
        }
      }
    }

    info = memory_get_size();
  }

  const data_start = new TextEncoder("utf-8").encode("{")[0];
  const data_end = new TextEncoder("utf-8").encode("}")[0];
  let skip = 0;
  let data_accumulate = false;
  let bytes = "";

  function receive_callback(data) {
    data.forEach((element, index, array) => {
      if (element == data_start && data_accumulate == false) {
        skip = 1;
        data_accumulate = true;
      }

      if (element == data_end && data_accumulate == true) {
        skip = 3;
        data_accumulate = false;
        parse_data(bytes);
        bytes = "";
      }

      if (data_accumulate) {
        if (skip == 0) {
          bytes += String.fromCharCode(element);
        } else {
          skip--;
        }
      } else {
        if (skip == 0) {
          terminal.add_line([element]);
        } else {
          skip--;
        }
      }
    });
  }

  async function connect() {
    await serial.try_to_connect();
  }

  async function may_be_connected() {
    await serial.on_connect();
  }

  may_be_connected();

  let highlight = {
    active: false,
    start: 0,
    end: 0,
  };

  onMount(() => {
    // if (typeof canvas !== "undefined") {
    const ctx = canvas.getContext("2d");
    // ctx.translate(0.5, 0.5);
    ctx.imageSmoothingEnabled = false;
    let frame = requestAnimationFrame(loop);

    function loop(t) {
      frame = requestAnimationFrame(loop);

      const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);

      for (let p = 0; p < imageData.data.length; p += 4) {
        const i = p / 4;
        const x = i % canvas.width;
        const y = (i / canvas.width) >>> 0;

        let c = [0, 0, 0];
        if (memory[i].alloc) {
          c = thread_list[memory[i].thread].color.slice();
        }

        if (c[0] > 200) c[0] -= 55;
        if (c[1] > 200) c[1] -= 55;
        if (c[2] > 200) c[2] -= 55;

        c[4] = 255;
        if (memory[i].size > 0) {
          c[0] += 55;
          c[1] += 55;
          c[2] += 55;
        }

        if (highlight.active) {
          if (i >= highlight.start && i < highlight.end) {
            c[0] = 0;
            c[1] = 255;
            c[2] = 0;
          }
        }

        imageData.data[p + 0] = c[0];
        imageData.data[p + 1] = c[1];
        imageData.data[p + 2] = c[2];
        imageData.data[p + 3] = c[4];
      }
      ctx.putImageData(imageData, 0, 0);
    }

    return () => {
      cancelAnimationFrame(frame);
    };
  });

  let popit = {
    show: false,
    x: 0,
    y: 0,
    address: "",
    size: 0,
    thread: "",
    history: [],
  };

  function canvas_click(e) {
    let rect = this.getBoundingClientRect();
    let pos = { x: e.clientX - rect.left, y: e.clientY - rect.top };
    let mem_pos = {
      x: Math.floor((canvas.width / rect.width) * pos.x),
      y: Math.floor((canvas.height / rect.height) * pos.y),
    };
    let mem_addr = mem_pos.y * canvas.width + mem_pos.x;

    if (mem_addr > 0) {
      let start = mem_addr;
      while (start >= 0) {
        if (memory[start].size > 0 || memory[start].alloc == false) break;
        start--;
      }

      let end = mem_addr + 1;
      while (end < SRAM_SIZE) {
        if (memory[end].size != 0 || memory[end].alloc == false) break;
        end++;
      }

      popup_message_data.show = true;
      popup_message_data.thread = memory[mem_addr].thread;
      popup_message_data.address =
        "0x" + (mem_addr * 4).toString(16) + " 0x" + (start * 4).toString(16);
      popup_message_data.size = memory[start].size;
      popup_message_data.history = memory[mem_addr].history;

      popup_message.show();
    }
  }

  function mouse_enter() {
    popit.show = true;
  }

  function mouse_leave() {
    popit.show = false;
    highlight.active = false;
  }

  function mouse_move(e) {
    popit.x = e.clientX + 20;
    popit.y = e.clientY + 20;

    let rect = this.getBoundingClientRect();
    let pos = { x: e.clientX - rect.left, y: e.clientY - rect.top };
    let mem_pos = {
      x: Math.floor((canvas.width / rect.width) * pos.x),
      y: Math.floor((canvas.height / rect.height) * pos.y),
    };
    let mem_addr = mem_pos.y * canvas.width + mem_pos.x;

    if (mem_addr > 0) {
      let start = mem_addr;
      while (start >= 0) {
        if (memory[start].size > 0 || memory[start].alloc == false) break;
        start--;
      }

      let end = mem_addr + 1;
      while (end < SRAM_SIZE) {
        if (memory[end].size != 0 || memory[end].alloc == false) break;
        end++;
      }

      popit.show = true;
      popit.thread = memory[mem_addr].thread;
      popit.address =
        "0x" + (mem_addr * 4).toString(16) + " 0x" + (start * 4).toString(16);
      popit.size = memory[start].size;
      popit.history = memory[mem_addr].history;

      highlight.active = true;
      highlight.start = start;
      highlight.end = end;
      // console.log(start, end);
    }
  }
</script>

<div class="screen">
  {#if popit.show}
    <div class="popit" style="left: {popit.x}px; top: {popit.y}px;">
      {popit.address}[{popit.size}]<br />
      {popit.thread}
      <div>
        {#each Object.entries(popit.history) as [key, data]}
          {data}<br />
        {/each}
      </div>
    </div>
  {/if}
  <div class="wrapper">
    {#if connected == false}
      <div class="wrapper-button">
        <button on:click={connect}>Connect</button>
      </div>
    {/if}
    <div class="wrapper-list">
      <!-- <div class="border" /> -->
      <div class="info">{info} - {SRAM_SIZE * 4}</div>
      <canvas
        class="sram_canvas"
        bind:this={canvas}
        width={SRAM_W}
        height={SRAM_H}
        on:mouseenter={mouse_enter}
        on:mousemove={mouse_move}
        on:mouseleave={mouse_leave}
        on:mousedown={canvas_click}
      />

      <div class="threads">
        {#each Object.entries(thread_list) as [thread, data]}
          <div
            class="thread"
            style="color: rgb({data.color[0]}, {data.color[1]}, {data
              .color[2]});"
          >
            {thread}[{data.size}]
          </div>
        {/each}
      </div>
    </div>
  </div>
  <div class="wrapper"><TerminalEmu bind:this={terminal} /></div>
  <Popup bind:this={popup_message}>
    {popup_message_data.address}[{popup_message_data.size}]<br />
    {popup_message_data.thread}
    <div>
      {#each Object.entries(popup_message_data.history) as [key, data]}
        {data}&nbsp;
      {/each}
    </div>
  </Popup>
</div>

<style>
  * {
    box-sizing: border-box;
  }

  button {
    padding: 10px 10px;
    color: #000000;
    background: #bbbbbb;
    text-align: center;
    vertical-align: middle;
    -webkit-box-shadow: 10px 10px 0 #000000;
    -moz-box-shadow: 10px 10px 0 #000000;
    box-shadow: 10px 10px 0 #000000;
    border: 0;
    border-radius: 0;
    margin-bottom: auto;
  }

  button:hover,
  button:focus {
    color: #ffffff;
    text-decoration: none;
    background: #555555;
    outline: 0;
  }

  button:active {
    transform: translate(10px, 10px);
    box-shadow: 0px 0px 0 #000000;
  }

  button:before {
    content: "< ";
  }
  button:after {
    content: " >";
  }

  .screen {
    display: flex;
    height: 100%;
  }

  .wrapper {
    height: 100%;
    position: relative;
    width: 50%;
  }

  .wrapper-button {
    display: flex;
    align-content: center;
    flex-wrap: wrap;
    justify-content: center;
    align-items: center;
    height: 50%;
  }

  .wrapper-list {
    display: flex;
    flex-direction: column;
    align-content: center;
    flex-wrap: wrap;
    align-items: center;
    height: 100%;
  }

  .info {
    background-color: #007f80;
    color: #000000;
    text-align: center;
    padding: 0 24px;
  }

  .sram_canvas {
    /* margin-top: 24px; */
    width: 100%;
  }

  .threads {
    display: flex;
    flex-wrap: wrap;
    flex-direction: row;
    justify-content: space-between;
    padding: 10px;
    background-color: black;
    border-top: 4px solid #000084;
  }

  .thread {
    padding: 5px;
  }

  .popit {
    background-color: #000000;
    padding: 5px 10px;
    pointer-events: none;
    position: absolute;
    z-index: 6969;
    max-height: 500px;
    overflow: auto;
  }

  .sram_canvas {
    image-rendering: -moz-crisp-edges;
    image-rendering: -webkit-crisp-edges;
    image-rendering: pixelated;
    image-rendering: crisp-edges;
  }
</style>

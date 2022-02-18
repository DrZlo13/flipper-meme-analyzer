<script>
  import { onMount } from "svelte";
  let xterm;
  let terminal;

  export async function add_line(text) {
    terminal.write(text);
  }

  onMount(async () => {
    let fit = await import("xterm-addon-fit");
    let FitAddon = fit.FitAddon;

    xterm = await import("xterm");
    let Terminal = xterm.Terminal;

    terminal = new Terminal({
      convertEol: true,
      fontFamily: `"Consolas", monospace`,
      fontSize: 20,
      rendererType: "canvas", // default is canvas
    });
    const fitAddon = new FitAddon();
    terminal.loadAddon(fitAddon);

    terminal.open(document.getElementById("terminal"));
    fitAddon.fit();
  });
</script>

<div id="terminal" />

<style>
  #terminal {
    width: 100%;
    height: 100%;
    background-color: black;
  }
</style>
